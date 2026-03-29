---
layout: post
title: "Self-Hosted Services — Mail, Web & Apps"
date: 2026-03-29 15:00:00 +0200
categories: homelab
tags: self-hosted mailcow hestiacp pangolin pocketid passbolt plex home-assistant
---

Infrastructure exists to serve applications. This post covers the self-hosted services running across my homelab — from email and web hosting to identity management and home automation.

## Mailcow — Self-Hosted Email

Mailcow is a full-featured mail server running as a Docker Compose stack inside an LXC container on the DC node. It handles email for five domains.

### Why Self-Host Email?

Control. I manage SPF, DKIM, and DMARC records myself. I control retention, filtering, and privacy. And for a business that runs client services, having email on infrastructure I own eliminates vendor lock-in.

### Architecture Decisions

- **Dedicated public IP** — Mailcow gets its own WAN interface on OPNsense with a completely separate firewall ruleset. This isolates email reputation from web hosting traffic
- **Docker-in-LXC** — Mailcow is a Docker Compose stack, but it runs inside an LXC container rather than directly on the hypervisor. This keeps the Docker footprint contained and makes backups via PBS straightforward
- **DNS coordination** — MX, SPF, DKIM, and DMARC records are managed in HestiaCP. Any domain change requires DNS-first workflow: configure records in HestiaCP, then enable the domain in Mailcow

### What It Handles

| Domain Count | Mailboxes | Total Messages |
|---|---|---|
| 5 domains | Multiple | ~7,700+ |

Mailcow includes Sieve filtering, SOGo webmail, and autodiscover for desktop and mobile clients.

---

## HestiaCP — Web & DNS Hosting

HestiaCP is a lightweight hosting control panel running on a dedicated LXC at the DC site. It manages web hosting, DNS zones, and SSL certificates.

### What HestiaCP Manages

- **Web hosting** — Apache/Nginx virtual hosts with PHP-FPM per domain
- **DNS** — authoritative DNS for hosted domains (separate from internal Technitium zones)
- **SSL** — automated Let's Encrypt certificates
- **FTP/SSH** — per-user accounts with chroot isolation

### Security Posture

HestiaCP is one of the few services directly exposed to the public internet (via OPNsense port forwards). Hardening includes:

- SSH key-only authentication — no password login
- Non-standard SSH port to reduce noise
- fail2ban actively monitoring access attempts
- Wazuh agent reporting to the SOC stack

---

## Pangolin & Newt — Zero-Trust Reverse Proxy

Pangolin is how every other service reaches the internet — without opening a single inbound firewall port.

### How It Works

1. **Pangolin server** runs on a cloud VPS and handles TLS termination
2. **Newt agents** at each site make outbound-only connections to Pangolin
3. Requests arrive at Pangolin, traverse the tunnel, and reach internal services
4. No inbound firewall rules needed at home or DC

### Active-Active HA

Each site runs **two Newt agents** on separate Proxmox nodes:

| Site | Primary | HA |
|---|---|---|
| DC | Newt on one node | Newt on another node |
| Home | Newt on primary node | Newt on secondary node |

Both agents connect simultaneously. If one agent or its host node goes down, Pangolin seamlessly routes through the survivor. This was validated during node maintenance — public services stayed up throughout.

### Health Check Gotcha

Pangolin health checks must hit a specific status endpoint — using the root path causes redirect loops that result in 503 errors even when the service is perfectly healthy. Small detail, but it cost me troubleshooting time.

---

## PocketID — OIDC Identity Provider

PocketID provides OpenID Connect (OIDC) single sign-on for internal services. It runs as an LXC on the home cluster.

### What It Enables

- SSO for services that support OIDC
- Centralized user authentication
- Email notifications for new device logins and verification codes

PocketID is a stepping stone toward a full centralized identity system. The long-term plan is to back it with LDAP (Synology Directory Server), but for now it handles OIDC independently.

---

## Passbolt — Password Manager

Passbolt is the secrets vault for the entire infrastructure. It stores:

- Service admin passwords
- SMTP credentials used by all Postfix relay hosts
- PBS encryption keys
- Proxmox API tokens
- Pangolin/Newt tunnel credentials
- Cloud provider access keys (Wasabi S3)

Passbolt runs on the home cluster and is accessible through Pangolin. It's a critical service — loss of access means loss of access to everything else. PBS backups of the Passbolt container are verified quarterly.

---

## Home Services

Not everything is enterprise-grade. Some services exist purely for quality of life.

### Plex — Media Server

Plex runs as an LXC on the home cluster, serving media to devices throughout the house. It's accessible externally through Pangolin for when I'm traveling.

### Home Assistant — Home Automation

Home Assistant runs as a full VM (not LXC) because it needs USB passthrough for the Zigbee coordinator. It manages smart home devices across the IoT VLAN — lights, sensors, climate, and automations.

The IoT VLAN is intentionally isolated — devices can reach Home Assistant but nothing else on the network.

### peaNUT — UPS Monitoring

peaNUT provides a web UI for Network UPS Tools (NUT), monitoring the UPS that protects the home lab. It runs as a VM with USB passthrough for direct UPS communication.

On power events (outage, low battery), peaNUT sends notifications via Postfix. In a sustained outage, the UPS signals Proxmox to initiate a clean shutdown.

---

## Other Services

| Service | Purpose |
|---|---|
| **Guacamole** | HTML5 remote desktop gateway — browser-based access to VMs |
| **MeshCentral** | Remote device management |
| **Pulse** | Uptime monitoring dashboard |
| **RackPeek** | Physical and logical rack inventory |
| **Umami** | Privacy-focused web analytics |

---

## The Notification Pipeline

Every service sends notifications through a unified pipeline:

```
Service → Postfix (local relay) → Mailcow → notifications inbox
```

Each LXC/VM runs Postfix configured as a relay host pointing to Mailcow. This means every service — from PBS backup completion to Wazuh alerts to UPS power events — lands in one inbox. Configuration is standardized across all hosts via Ansible.

---

## Lessons Learned

- **Docker-in-LXC works** — Mailcow's Docker stack runs perfectly inside an LXC container, and you get PBS backup support for free
- **Dedicated IPs for email** — separating mail traffic from web hosting protects your sender reputation
- **Zero-trust with Pangolin is liberating** — no more maintaining NAT rules and port forwards for every new service
- **Active-active HA for Newt** — dual tunnel agents per site was easy to set up and eliminated single points of failure
- **Centralize notifications** — one inbox for all infrastructure alerts beats checking five different dashboards

Next up: how all of this is provisioned and managed as code.
