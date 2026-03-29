---
layout: post
title: "Virtualization — Proxmox VE Cluster Setup"
date: 2026-03-29 11:00:00 +0200
categories: homelab
tags: proxmox virtualization cluster zfs ceph lxc
---

This post dives into the virtualization layer of my homelab — a multi-site Proxmox VE deployment spanning a home cluster and a standalone datacenter node.

## Why Proxmox VE?

After years on VMware's ecosystem, I wanted a hypervisor that was open-source, supported both VMs and containers natively, and had enterprise features like clustering and live migration without a license paywall. Proxmox VE checked every box — and with ZFS built in, it eliminated the need for a separate storage layer.

---

## Home Cluster — 2 Nodes + QDevice

The home site runs a 2-node Proxmox VE cluster. In a traditional cluster, you need an odd number of nodes for quorum. With only two nodes, I use a **QDevice** — a lightweight quorum witness running as a Docker container on my Synology NAS.

| Node | Role | Storage |
|---|---|---|
| Primary node | Main workloads | ZFS on NVMe (1 TB) |
| Secondary node | Overflow + management services | ZFS (single disk) |
| Synology NAS | QDevice host + NFS/iSCSI target | HDD array |

### How QDevice Works

The QDevice adds a third vote to the cluster, giving it 3 total expected votes. This means either node can go offline without losing quorum — critical for maintenance windows and unexpected reboots. The QDevice runs `corosync-qnetd` in a Docker container on the Synology, which is always on anyway.

### What Runs at Home

The home cluster hosts services that benefit from local access or need hardware passthrough:

- **AdGuard Home** — DNS filtering for all home VLANs
- **Home Assistant** — home automation hub (VM for USB passthrough)
- **Plex** — media server
- **Proxmox Backup Server** — local backup target
- **WireGuard** — VPN tunnels to the datacenter
- **Pangolin Newt agents** — zero-trust reverse proxy tunnel endpoints
- **Proxmox Datacenter Manager** — cross-site cluster management
- **PocketID** — OIDC identity provider
- **peaNUT** — UPS monitoring via NUT (VM for USB passthrough)

---

## Datacenter Node — Standalone at Hetzner

The DC runs a standalone Proxmox VE node on a Hetzner dedicated server. It's intentionally **not** part of the home cluster — different failure domain, different network, different purpose.

| Component | Detail |
|---|---|
| CPU | Intel Core i7 — 6 cores / 12 threads |
| Memory | 128 GB RAM |
| Storage | ZFS on NVMe (~950 GB) + NFS from Synology |
| Firewall | OPNsense VM as the network gateway |

The DC hosts production-facing workloads: email (Mailcow), web hosting (HestiaCP), the full SOC stack (Wazuh, TheHive, MISP), GitLab CE, and all IaC tooling.

### VMID Allocation Scheme

I keep VMID ranges organized by function:

| Range | Purpose |
|---|---|
| 100–199 | Core infrastructure (firewall, admin, NAS) |
| 1000–1099 | Infrastructure VMs and LXCs |
| 2000–2099 | Security stack (Wazuh, TheHive, MISP) |
| 3000–3099 | Client-facing services (web, mail, DNS) |
| 4000–4099 | Business services |
| 5000–5099 | Dev/misc |
| 6000–6099 | IaC stack (GitLab, Runner, RackPeek) |

---

## ZFS and Ceph RBD

**Home:** Both nodes run ZFS — the primary node on NVMe, the secondary on spinning rust. ZFS gives me snapshots, compression, and checksumming without extra software. The Synology provides supplementary storage over iSCSI (for PBS) and NFS (for vzdump backups).

**DC:** The Hetzner node also runs ZFS on NVMe. For my VMware-to-Proxmox migration project, I tested **Ceph RBD** as a target for `qemu-img convert` — converting VMware VMDKs directly to Ceph RBD images. Ceph isn't used for daily operations at home (yet), but it's proven useful for migration workflows.

---

## LXC vs VM — When to Use Which

One of Proxmox's strengths is first-class support for both LXC containers and full VMs. My decision framework:

**Use an LXC when:**
- The workload is Linux-only and doesn't need a custom kernel
- No hardware passthrough is required
- You want minimal overhead (LXCs share the host kernel)
- Examples: DNS servers, tunnel agents, monitoring dashboards, web apps

**Use a VM when:**
- You need hardware passthrough (USB, GPU, NIC)
- The workload requires a specific OS or kernel
- Isolation requirements are higher (e.g., firewall appliance)
- Examples: OPNsense, Home Assistant (USB Zigbee stick), peaNUT (USB UPS), PBS

Most of my workloads are LXCs — they're lighter, faster to provision, and easier to manage with Terraform.

---

## Proxmox Datacenter Manager (PDM)

PDM is a relatively new Proxmox product that provides a unified management interface across multiple Proxmox sites. I run it as an LXC on the home cluster, and it connects to both the home cluster and the DC node.

### What PDM Enables

- **Cross-site VM/LXC migration** — I used PDM extensively to migrate LXCs from home to DC (batch migration of several containers in one session)
- **Datacenter-wide resource overview** — see CPU, RAM, and storage across all nodes in one dashboard
- **Centralized task monitoring** — track running jobs across sites

PDM migrations require both source and target to be reachable, which in my case means the WireGuard management tunnel must be healthy. I always verify tunnel status before starting a cross-site migration.

---

## The pvem Decommission Story

The home cluster originally had three nodes: pve0, pven, and pvem. **pvem** was a dedicated backup node — its only job was hosting the PBS VM and a local backup disk.

When I migrated PBS to pve0 (where it could use ZFS directly and access the Synology iSCSI LUN), pvem became redundant. I decommissioned it in early 2026:

1. Migrated PBS VM to pve0
2. Verified all backup flows were healthy on the new host
3. Removed pvem from the cluster
4. Repurposed the hardware

The lesson: dedicated backup hardware sounds nice in theory, but when your primary nodes have enough headroom and better storage connectivity, consolidation wins.

---

## What's Next

The virtualization layer is stable and well-understood. Future plans include:
- **Local AI stack** — an RTX 5080 GPU in the home cluster for Ollama and llama.cpp
- **Ceph expansion** — if I add a third home node, Ceph becomes viable for replicated storage
- **Forgejo migration** — replacing GitLab CE once Terraform state backend support lands

Stay tuned for posts on each of the other homelab departments — networking, security, backup, services, and IaC.
