---
layout: post
title: "OPNsense as a Datacenter Firewall on Proxmox"
date: 2026-03-29 17:10:00 +0200
categories: setup
tags: opnsense firewall proxmox networking security
---

Running a dedicated firewall VM is essential when your datacenter hosts production services on the public internet. This post details how I deploy OPNsense as the sole network gateway on a Proxmox VE node at Hetzner - handling VLAN segmentation, WireGuard tunnels, multi-WAN routing, and strict per-interface firewall rules.

## Why OPNsense on Proxmox?

The datacenter node sits on a Hetzner dedicated server with multiple public IPs and private subnets. Rather than relying on the host firewall (iptables on the hypervisor), I run OPNsense as a VM that owns all routing and security policy. This gives me:

- **Full GUI firewall management** with audit logging
- **VLAN-aware interface separation** without touching the hypervisor
- **WireGuard integration** natively within the firewall
- **Multi-WAN** support for dedicated IP assignment per service
- **HAProxy/NAT/port forwarding** all in one place

OPNsense is the first VM to boot at the DC - every other workload depends on it for network connectivity. If OPNsense is down, the entire site is offline.

---

## Architecture

The DC Proxmox host creates multiple Linux bridges, each mapped to an OPNsense interface. This creates clean network segmentation at the hypervisor level - VMs and LXCs attach to the bridge for their intended VLAN, and OPNsense handles all inter-VLAN routing and firewall policy.

### Interface Map

| OPNsense Interface | Bridge Purpose | Traffic Type |
|---|---|---|
| WAN | Public internet uplink | Inbound/outbound internet |
| LAN (Management) | Core infrastructure | Hypervisor, PBS, admin tools, monitoring |
| Clients | Web hosting and email | HestiaCP, Mailcow, client-facing services |
| Backup | Backup replication traffic | PBS replication, Synology NFS |
| WireGuard Backup (wg0) | Tunnel interface | PBS-to-PBS cross-site replication |
| WireGuard Management (wg1) | Tunnel interface | Cross-site admin and management access |
| Dedicated Mail WAN | Separate public IP | 1:1 NAT for email server |

That's seven interfaces on a single firewall VM. Each has its own subnet, its own firewall ruleset, and its own purpose. No traffic crosses zones without an explicit rule.

### VM Resources

| Resource | Allocation |
|---|---|
| vCPUs | 4 |
| Memory | 8 GB |
| Disk | 50 GB |
| NICs | 5 virtual NICs + 2 WireGuard interfaces |

8 GB might sound generous for a firewall, but OPNsense handles WireGuard encryption, stateful packet inspection across seven interfaces, and DNS resolution - the headroom is warranted.

---

## Multi-WAN Design

The DC has access to multiple public IPs. Rather than routing everything through a single address, I assign dedicated public IPs to specific services:

| Public IP Purpose | Mapped To |
|---|---|
| Shared WAN | OPNsense primary - web hosting port forwards, WireGuard endpoints, default outbound |
| Dedicated Mail WAN | 1:1 NAT to Mailcow - completely separate IP and ruleset |
| Host Management | Proxmox host public IP - not routed through OPNsense |

### Why Mailcow Gets Its Own IP

Email reputation depends heavily on the sending IP's history. If Mailcow shared an IP with web hosting, any abuse complaint against a hosted website could tank email deliverability. The dedicated mail WAN interface gives Mailcow:

- **Complete IP isolation** - its own public IP with no shared traffic
- **Independent firewall rules** - only mail ports and HTTPS for the admin UI
- **Clean reverse DNS** - the PTR record points exclusively to the mail domain
- **No interference** - web hosting issues cannot affect email routing

This is achieved via a 1:1 NAT rule on the dedicated mail WAN interface - all traffic to that public IP forwards directly to the Mailcow internal IP, and all outbound traffic from Mailcow exits via that same dedicated IP.

---

## Firewall Rule Philosophy

I follow a zero-trust, deny-by-default approach on every interface. Each interface has its own ruleset with only the minimum necessary rules:

### WAN Inbound Rules

| Rule | Protocol | Port | Purpose |
|---|---|---|---|
| Web to hosting | TCP | 80, 443 | Port forward to HestiaCP |
| SSH to hosting | TCP | Non-standard | Port forward to HestiaCP SSH |
| WireGuard | UDP | 51820 | Backup tunnel endpoint |

Everything else is dropped. No ICMP from the internet, no unexpected ports.

### Dedicated Mail WAN Inbound Rules

| Rule | Protocol | Ports | Purpose |
|---|---|---|---|
| Mail traffic | TCP | 25, 465, 587, 993, 4190 | SMTP, SMTPS, submission, IMAP, ManageSieve |
| Web UI | TCP | 443 | Mailcow admin interface |

Port 80 is deliberately disabled on the mail WAN - all web traffic to Mailcow must use HTTPS.

### Management LAN Rules

| Direction | Rule | Purpose |
|---|---|---|
| Outbound | Allow all | Management hosts need full internet access for updates |
| To Clients | Block | No lateral movement from management to client VLAN |

### Clients VLAN Rules

| Direction | Rule | Purpose |
|---|---|---|
| Outbound | Allow internet | Web hosting and email need public connectivity |
| To Docker subnet | Allow | Mailcow internal Docker network access |
| To Management | Block | No lateral movement to management VLAN |

### Backup VLAN Rules

| Direction | Rule | Purpose |
|---|---|---|
| To WireGuard tunnel | Allow | PBS pull sync from DC to home via wg0 |
| To backup server | Allow | Inbound backup traffic to PBS |
| To Management | Block | No lateral movement to management VLAN |

---

## VLAN Segmentation on Proxmox Bridges

Each Proxmox bridge maps to exactly one OPNsense interface and one subnet:

| Bridge | Subnet | OPNsense Interface | Used By |
|---|---|---|---|
| vmbr0 | Public | WAN | OPNsense external |
| vmbr1 | Management | LAN | Admin VMs, PBS (mgmt NIC), monitoring |
| vmbr2 | Clients | Clients | HestiaCP, Mailcow, web services |
| vmbr3 | Backup | Backup | PBS (backup NIC), Synology NFS |

When I create a new VM or LXC, I attach it to the appropriate bridge and it automatically lands in the correct security zone. No manual VLAN tagging, no trunk ports to manage - the bridge-to-interface mapping handles everything.

---

## WireGuard Integration

OPNsense has native WireGuard support, so both tunnels are configured directly within the firewall. Each tunnel gets its own OPNsense interface with dedicated firewall rules.

### Tunnel Interfaces

| Tunnel | OPNsense Interface | UDP Port | Purpose |
|---|---|---|---|
| wg0 | WireGuard Backup | 51820 | PBS replication between storage VLANs |
| wg1 | WireGuard Management | 51821 | Cross-site admin access |

Having WireGuard inside OPNsense means tunnel traffic passes through the firewall engine - I get logging, rate limiting, and rule enforcement on tunnel traffic just like any other interface. This is a significant advantage over running WireGuard on a standalone host outside the firewall path.

---

## NAT and Port Forwarding

### HestiaCP (Shared WAN)

Web hosting runs on the shared WAN IP via port forwards:

| External Port | Internal Destination | Service |
|---|---|---|
| 80, 443 | HestiaCP internal IP | Web hosting |
| Non-standard SSH port | HestiaCP internal IP | Secure shell access |

### Mailcow (Dedicated Mail WAN)

Mailcow uses 1:1 NAT - no port forwarding needed. The entire public IP maps bidirectionally to the internal Mailcow IP.

### Hairpin NAT

Internal services that need to reach Mailcow via its public domain name hit a classic hairpin NAT problem - the request goes out to the public IP and comes back in on the same interface. OPNsense handles this with NAT reflection, but I prefer split-horizon DNS as the cleaner solution - internal clients resolve the mail domain to the internal IP directly, bypassing NAT entirely.

---

## Docker Network Considerations

Mailcow uses Docker internally, which creates its own network subnet. This subnet is invisible to OPNsense unless explicitly allowed. Without an allow rule on the Clients interface for the Docker subnet, internal Mailcow components cannot communicate with each other across the Docker bridge.

This is an easy gotcha to miss - everything looks fine from outside, but Mailcow containers fail to talk to each other. The fix is a single allow rule on the Clients interface permitting traffic to the Docker subnet.

---

## Boot Order and Dependencies

OPNsense must be running before any other VM or LXC starts. Without it, nothing has a gateway, nothing has DNS, and nothing can reach the internet or cross VLANs.

The boot order on the DC Proxmox node:

1. **OPNsense** - network gateway and firewall
2. **PBS** - must be running before backup windows (01:00-05:00)
3. **Everything else** - services, monitoring, SOC stack

Proxmox's start order and startup delay settings handle this automatically, but I've been bitten by race conditions during host reboots where services started before OPNsense had fully initialized. A generous startup delay on OPNsense (30-60 seconds before other VMs start) prevents this.

---

## Operational Notes

**Notifications via Postfix relay.** All LXCs and VMs relay email notifications through the internal Mailcow instance. This requires a specific relay configuration pointing to Mailcow's internal IP on the submission port - not the public IP, to avoid hairpin NAT.

**Synology NFS mount verification.** The NFS storage from the DC Synology NAS mounts at the Proxmox host level. After a NAS reboot, verify the mount is healthy before backup windows. A stale NFS mount causes vzdump jobs to hang silently.

**Firewall rule auditing.** OPNsense logs every passed and blocked packet. I review the logs periodically for unexpected traffic patterns - especially on the WireGuard interfaces, where a misconfigured `AllowedIPs` can cause traffic to leak between zones.

---

## Lessons Learned

- **Multi-WAN is worth the complexity for email.** Dedicated IPs for mail services protect your sender reputation. It takes 10 minutes to configure and saves hours of deliverability troubleshooting.

- **Bridge-per-VLAN is cleaner than VLAN trunking.** On a single-host deployment, one bridge per subnet eliminates VLAN tagging mistakes and makes VM networking straightforward.

- **WireGuard inside the firewall beats standalone.** Having tunnel traffic pass through OPNsense means you get logging, rules, and visibility without extra hops.

- **Boot order is infrastructure.** A firewall that starts after its dependents is useless. Treat boot order as a first-class configuration item.

- **Hairpin NAT is a smell.** If you need NAT reflection, consider whether split-horizon DNS is the better answer. It usually is.

---

## What's Next

The OPNsense configuration is stable and handles daily production traffic without issues. Future plans include evaluating OPNsense's built-in IDS/IPS (Suricata) for inline threat detection on the WAN interfaces, and potentially automating firewall rule management via the OPNsense API for tighter IaC integration.
