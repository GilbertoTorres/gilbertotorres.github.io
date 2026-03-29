---
layout: post
title: "Dual WireGuard Tunnels for Cross-Site Connectivity"
date: 2026-03-29 17:20:00 +0200
categories: setup
tags: wireguard vpn networking cross-site tunnel
---

Connecting a home lab to a remote datacenter with a single VPN tunnel is simple - but fragile. This post explains why I run two separate WireGuard tunnels between my home and DC sites, how they're architected, and the operational discipline required to keep them healthy.

## Why Two Tunnels?

The obvious approach is one WireGuard tunnel carrying all cross-site traffic. I started there and quickly ran into problems:

1. **A misconfiguration on the tunnel broke everything.** Backup replication, management access, DNS forwarding - all gone in one `AllowedIPs` change
2. **Firewall rules became a mess.** One interface carrying mixed traffic means complex rules to differentiate backup from management
3. **No safe fallback.** When the tunnel goes down, you lose all cross-site access simultaneously

Splitting into two purpose-specific tunnels solves all three problems. Each tunnel carries one type of traffic, has its own firewall interface, and can fail independently.

---

## Tunnel Architecture

| Tunnel | Interface | Port | Addressing | Purpose |
|---|---|---|---|---|
| Backup | wg0 | 51820 | /30 point-to-point | PBS-to-PBS replication between storage VLANs |
| Management | wg1 | 51821 | /30 point-to-point | Cross-site admin access between management VLANs |

Both tunnels use /30 addressing - just two IPs per tunnel. The home endpoint runs on a dedicated WireGuard LXC container, and the DC endpoint is integrated into OPNsense. This means DC-side tunnel traffic passes through the firewall engine automatically.

### Home Side

The home endpoints live in a dedicated LXC container on the primary Proxmox node. This container hosts both wg0 and wg1 interfaces and runs WG-Dashboard for web-based monitoring.

| Component | Detail |
|---|---|
| Container type | LXC on primary node |
| Interfaces | wg0 (backup) + wg1 (management) |
| Management UI | WG-Dashboard |
| Connectivity | Management VLAN - routes to storage and management subnets |

### DC Side

The DC endpoints are configured directly inside OPNsense. Each tunnel maps to a dedicated OPNsense interface with its own firewall ruleset:

| Tunnel | OPNsense Interface | Firewall Zone |
|---|---|---|
| wg0 | WireGuard Backup | Dedicated backup rules - storage VLAN traffic only |
| wg1 | WireGuard Management | Dedicated management rules - admin VLAN traffic only |

---

## AllowedIPs - The Critical Configuration

WireGuard's `AllowedIPs` setting is both the routing table and the access control list. It determines which traffic enters and exits each tunnel. Getting this wrong is the single most common cause of tunnel problems.

### Backup Tunnel (wg0) - AllowedIPs

The backup tunnel carries only storage VLAN traffic - specifically, PBS replication between the home and DC storage subnets:

| Peer | AllowedIPs Scope |
|---|---|
| Home peer | DC storage subnet |
| DC peer | Home storage subnet |

Nothing else traverses this tunnel. No management traffic, no DNS, no web browsing. If I mess up wg1, backup replication continues unaffected.

### Management Tunnel (wg1) - AllowedIPs

The management tunnel routes between the home and DC management subnets:

| Peer | AllowedIPs Scope |
|---|---|
| Home peer | DC management subnet |
| DC peer | Home management subnet |

This tunnel carries admin SSH sessions, Proxmox web UI access, DNS forwarding between Technitium nodes, and PDM cross-site management traffic.

---

## Routing and Integration

### Home Routing

The WireGuard LXC container needs routes to forward tunnel traffic to the correct local subnets. The home firewall (or the LXC itself) handles this routing:

- Traffic arriving on wg0 destined for the home storage subnet gets forwarded to the storage VLAN
- Traffic arriving on wg1 destined for the home management subnet gets forwarded to the management VLAN

Other devices on the home network that need to reach DC subnets must have static routes pointing to the WireGuard container as the next hop. This includes:

- The Synology NAS (for Synology-to-Synology rsync over the backup tunnel)
- Any management host that needs direct DC access

### DC Routing

On the DC side, OPNsense handles all routing natively since the tunnels are OPNsense interfaces. The firewall's routing table knows:

- wg0 traffic for the home storage subnet exits via the backup tunnel interface
- wg1 traffic for the home management subnet exits via the management tunnel interface

### Static Route on PBS

The DC PBS server has a dedicated backup NIC on the backup VLAN. For pull sync from the home PBS, traffic to the home storage subnet must route via the backup interface and through the WireGuard backup tunnel - not via the default management gateway.

This requires a static route on PBS itself:

```
iface ens18 inet static
    address <backup-ip>
    post-up ip route add <home-storage-subnet> via <backup-gateway> dev ens18
    pre-down ip route del <home-storage-subnet> via <backup-gateway> dev ens18
```

Without this static route, the kernel's default route (on the management NIC) silently absorbs storage VLAN traffic. PBS sync jobs fail with no obvious error - the traffic just goes to the wrong gateway and gets dropped. This is one of those gotchas that costs hours to debug if you don't know to look for it.

---

## WG-Dashboard

I run WG-Dashboard on the home WireGuard container to monitor both tunnels from a web UI. It shows:

- **Peer status** - last handshake time, data transferred, current endpoint
- **Interface status** - up/down, listening port, public key
- **Traffic graphs** - useful for spotting anomalies in backup replication patterns

WG-Dashboard is strictly a monitoring tool in my setup - I make all configuration changes via the CLI or OPNsense GUI. Having a dashboard that shows both tunnels side-by-side makes it easy to verify health at a glance.

---

## Operational Discipline

### The Golden Rule

**Always verify the backup tunnel (wg0) is healthy before touching the management tunnel (wg1).**

This is non-negotiable. The management tunnel carries the traffic that lets me fix things remotely - if I break wg1 and wg0 is already down, I'm locked out of the DC entirely (short of using an out-of-band console).

### AllowedIPs Changes Are Dangerous

Modifying `AllowedIPs` on an active WireGuard tunnel takes effect immediately. There is no staging, no preview, no rollback. A typo can:

- Drop all active sessions on the tunnel
- Break the Pangolin reverse proxy chain (which depends on management tunnel connectivity)
- Cause traffic to route into the wrong tunnel
- Create asymmetric routing where packets enter via one path and try to exit via another

Before any `AllowedIPs` change:

1. Verify both tunnels are healthy
2. Have an out-of-band access path ready (Hetzner rescue console for DC)
3. Document the current working configuration
4. Make the change on one side first and verify
5. Then update the other side

### Tunnel Health Checks

Quick verification commands:

```bash
# Check WireGuard interface status
wg show wg0
wg show wg1

# Verify handshake recency (should be within 2 minutes for active tunnels)
wg show wg0 latest-handshakes
wg show wg1 latest-handshakes

# Ping through each tunnel
ping -c 3 <dc-backup-tunnel-ip>    # via wg0
ping -c 3 <dc-mgmt-tunnel-ip>      # via wg1
```

A stale handshake (older than 2-3 minutes on an active tunnel) usually means a firewall rule is blocking UDP traffic on the WireGuard port, or the remote endpoint's IP has changed.

---

## Real-World Incident - DNS Deployment and Tunnel Instability

During the deployment of Technitium DNS at the DC site, I needed to modify wg1's `AllowedIPs` to include the new DNS container's subnet. The change caused wg1 instability - management tunnel traffic was intermittently dropping.

Because backup replication runs on wg0 (a completely separate tunnel), PBS sync jobs continued without interruption. I was able to diagnose and fix the wg1 issue without any impact on backup operations.

If I had been running a single tunnel, both management access and backup replication would have been affected simultaneously. The dual-tunnel design turned a potential outage into a contained management-plane issue.

---

## Performance Considerations

WireGuard is lightweight, but running two tunnels does consume some resources:

| Resource | Impact |
|---|---|
| CPU | Negligible - WireGuard uses in-kernel encryption |
| Memory | Minimal - each tunnel adds a few KB of state |
| Bandwidth | No overhead beyond WireGuard's ~60 byte header per packet |
| Latency | One extra hop vs direct routing, but WireGuard adds sub-millisecond processing |

The real cost is operational complexity - two tunnels means two sets of configurations to maintain, two sets of firewall rules, and two things to check when troubleshooting connectivity. The safety benefit far outweighs this cost.

---

## Integration Points

The dual-tunnel design touches several other systems:

| System | Tunnel Used | Purpose |
|---|---|---|
| PBS Home to DC replication | wg0 (backup) | Daily pull sync of all home VM/LXC backups |
| Synology rsync | wg0 (backup) | Daily vzdump file replication between NAS devices |
| Proxmox Datacenter Manager | wg1 (management) | Cross-site VM/LXC migration and monitoring |
| Technitium DNS | wg1 (management) | DNS zone synchronization between home and DC |
| SSH admin access | wg1 (management) | Remote management of home resources from DC and vice versa |
| Pangolin/Newt | wg1 (management) | Reverse proxy tunnel connectivity |

---

## Lessons Learned

- **Separation of concerns applies to tunnels too.** One tunnel per purpose is cleaner than one tunnel with complex routing rules.

- **AllowedIPs is your friend and your enemy.** It's the most powerful and most dangerous WireGuard setting. Treat changes with the respect you'd give a firewall rule change in production.

- **Static routes are silent killers.** A missing static route on a multi-homed host causes traffic to take the default gateway path. Everything looks fine until you check which interface the traffic actually traverses.

- **WG-Dashboard pays for itself.** A quick glance at handshake times and traffic graphs tells you more about tunnel health than any amount of log parsing.

- **Always have a fallback.** Whether it's the backup tunnel surviving a management tunnel failure, or an out-of-band console for the DC host, never put yourself in a position where a single tunnel failure locks you out.

---

## What's Next

The dual-tunnel setup has been stable since deployment. Future improvements include automated tunnel health monitoring with alerting (currently I check manually via WG-Dashboard and Pulse), and potentially adding a third tunnel for a dedicated IoT/media streaming path if the traffic patterns justify it.
