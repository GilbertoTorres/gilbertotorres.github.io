---
layout: post
title: "Networking — OPNsense, WireGuard & DNS"
date: 2026-03-29 12:00:00 +0200
categories: homelab
tags: networking opnsense wireguard dns adguard technitium vlan
---

Networking is the backbone of any multi-site homelab. This post covers how I connect my home and datacenter sites, segment traffic with VLANs, and run a layered DNS architecture.

## OPNsense as the DC Firewall/Router

The datacenter site runs **OPNsense** as a virtual appliance on Proxmox. It's the first VM to boot — every other workload depends on it for routing and firewall policy.

OPNsense manages multiple network interfaces, each mapped to a dedicated Proxmox bridge:

| Interface | Purpose |
|---|---|
| WAN | Public internet uplink |
| Management | Core infrastructure and admin access |
| Clients | Web hosting, email, and client-facing services |
| Backup | PBS replication and NAS traffic |
| WireGuard Backup | Point-to-point tunnel for backup replication |
| WireGuard Management | Point-to-point tunnel for cross-site admin |
| Dedicated Mail WAN | Separate public IP for email (1:1 NAT) |

The dedicated mail WAN interface is key — Mailcow gets its own public IP with a completely separate firewall ruleset. This keeps email reputation isolated from everything else.

At home, **UniFi** handles switching and wireless, with DHCP served per VLAN. The UniFi gateway does not run DNS — that's delegated entirely to AdGuard Home.

---

## VLAN Segmentation Philosophy

Both sites use purpose-specific VLANs. The philosophy is simple: **no implicit cross-zone trust**. Each VLAN serves a single purpose, and traffic between VLANs must pass through a firewall with explicit allow rules.

| VLAN | Purpose | Sites |
|---|---|---|
| Management | Infrastructure admin, hypervisors, core services | Home + DC |
| WLAN | Wireless clients | Home only |
| Lab | Experimentation and dev workloads | Home + DC |
| IoT | Smart home devices (isolated) | Home only |
| Storage | NFS, iSCSI, backup traffic | Home + DC |
| Clients | Web/mail hosting | DC only |
| Backup | PBS replication | DC only |
| DMZ | Isolated services | Home + DC |

IoT devices live on their own VLAN with no route to management or storage. Smart TVs and streaming devices stay on a legacy network segment with only public internet access — they don't need internal DNS or management access, so they don't get it.

---

## Dual WireGuard Tunnels

Two WireGuard point-to-point tunnels link the home and DC sites. Each tunnel is purpose-separated with tight `AllowedIPs` scoping — this isn't a flat bridge.

| Tunnel | Purpose | Addressing |
|---|---|---|
| wg0 — Backup | PBS-to-PBS replication between storage VLANs | /30 point-to-point |
| wg1 — Management | Cross-site admin access between management VLANs | /30 point-to-point |

### Why Two Tunnels?

Separating backup and management traffic into distinct tunnels provides:

1. **Blast radius control** — a misconfiguration on one tunnel doesn't break the other
2. **Independent firewall policies** — each tunnel has its own OPNsense interface and ruleset
3. **Safe fallback** — the backup tunnel (wg0) is the safe one. I always verify wg0 is healthy before touching wg1

This design paid off during a DNS deployment that caused wg1 instability — backup replication continued unaffected because it runs on a completely separate tunnel.

### Operational Discipline

Modifying `AllowedIPs` on active WireGuard tunnels is high-risk — changes take effect immediately and can drop active sessions, including the Pangolin reverse proxy chain. The rule is: **always have backup tunnel access before modifying the management tunnel**.

---

## DNS Architecture

DNS is a three-layer system spanning both sites and the public internet.

### Layer 1 — AdGuard Home (Home)

AdGuard Home is the sole DNS resolver for all home VLANs. It provides:

- **Ad and tracker blocking** via upstream filter lists
- **Split-horizon DNS** — resolves internal hostnames to internal IPs
- **Conditional forwarding** — routes internal zone queries to Technitium

Upstream resolvers are DoT to AdGuard DNS (unfiltered — local filtering handles the rest), with Cloudflare and Quad9 as fallbacks.

### Layer 2 — Technitium DNS Cluster (Home + DC)

A two-node **Technitium DNS** cluster provides authoritative internal DNS:

| Node | Location | Role |
|---|---|---|
| Primary | DC | Authoritative for all internal zones |
| Secondary | Home | Synced replica via catalog zones |

The cluster uses catalog zones for automatic zone synchronization — when I add a zone on the primary, it automatically appears on the secondary. Communication between nodes runs over the WireGuard management tunnel.

**Resilience:** If the DC primary goes down, the home secondary continues resolving all internal zones from its synced copy. Home DNS never depends on a working WireGuard tunnel for cached zones.

### Layer 3 — Cloudflare (Public)

Cloudflare manages the external DNS zone. All public-facing services resolve to the Pangolin reverse proxy — no direct IP exposure.

### The Full Resolution Path

```
Home client
  → AdGuard Home [ad blocking + conditional forwarder]
    → Technitium Secondary [synced copy of all zones]
      → Technitium Primary (DC) via WireGuard [authoritative]
    → Cloudflare / Quad9 [public internet]

DC client
  → Technitium Primary [authoritative]
    → Cloudflare / Quad9 [public internet]
```

---

## Zero-Trust External Access — Pangolin & Newt

All public-facing services go through **Pangolin** — a self-hosted tunnelled reverse proxy. There are **no inbound firewall rules** at either site.

### How It Works

1. **Pangolin server** runs on a cloud VPS and terminates TLS
2. **Newt agents** at each site make outbound connections to Pangolin
3. Pangolin routes incoming requests through the tunnel to internal services
4. Both sites run two Newt agents each (on separate Proxmox nodes) for **active-active HA**

If one Newt agent or node goes down, Pangolin continues routing through the surviving agent. This gives me zero-trust perimeter security with built-in redundancy — and I never have to open an inbound port.

---

## Lessons Learned

- **Separate tunnels per purpose** — one flat VPN is tempting but fragile
- **DNS is infrastructure** — treat it like you'd treat power. Redundancy matters
- **Zero-trust isn't just a buzzword** — Pangolin + Newt eliminated my entire inbound firewall ruleset
- **Document your AllowedIPs** — WireGuard is stateless and unforgiving. Know exactly what each tunnel carries

Next up: the security and monitoring stack that watches over all of this.
