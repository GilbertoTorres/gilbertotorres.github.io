---
layout: post
title: "Setting Up a Proxmox VE Cluster with QDevice Quorum"
date: 2026-03-29 17:00:00 +0200
categories: setup
tags: proxmox cluster corosync qdevice quorum high-availability
---

Running a two-node Proxmox VE cluster without a quorum witness is a recipe for split-brain disaster. This post covers how I set up my home cluster with a Corosync QDevice for reliable quorum, how Proxmox Datacenter Manager ties both sites together, and the architectural decisions behind the whole setup.

## The Quorum Problem in a 2-Node Cluster

Proxmox VE uses Corosync for cluster communication. Corosync requires a majority of votes (quorum) to function - if quorum is lost, the cluster fences itself to prevent split-brain. In a standard 2-node cluster with 2 votes, losing a single node means you lose quorum - and the surviving node refuses to operate.

The solution is a **QDevice** - a lightweight quorum witness that adds a third vote without adding a third hypervisor. With 3 total votes (2 nodes + 1 QDevice), either node can go offline while the remaining node plus QDevice maintain quorum.

---

## Architecture Overview

| Component | Role | Details |
|---|---|---|
| Primary node | Main workloads | Intel i3-N305, 16 GB RAM, ZFS on 1 TB NVMe |
| Secondary node | Overflow + management | ZFS on single disk |
| Synology NAS | QDevice host | Docker container running corosync-qnetd |

The cluster is named `homelab` and uses the `ffsplit` algorithm for vote assignment. The QDevice runs as a Docker container on the Synology NAS - a device that's always on anyway for NFS and iSCSI duties.

### Why Not Just Add a Third Node?

A third Proxmox node would work, but it means maintaining another hypervisor for what is essentially a tiebreaker role. The QDevice approach is more efficient - it consumes negligible resources on the NAS and provides the same quorum guarantee. I did previously have a third node (dedicated to backup duties), but when I consolidated PBS onto the primary node, the extra hardware became unnecessary and was decommissioned.

---

## QDevice Setup

### Prerequisites

The QDevice host (Synology NAS) needs to run `corosync-qnetd`. Since the NAS already runs Docker, I deployed it as a container - no need to install Corosync packages directly on DSM.

### On the QDevice Host

The corosync-qnetd container needs to be reachable on port 5403 from both cluster nodes. The container runs in host networking mode to simplify this.

### On the Cluster Nodes

Once the QDevice host is running, you add it to the cluster from any node:

```bash
pvecm qdevice setup <qdevice-ip>
```

This command handles SSH key exchange, certificate setup, and Corosync configuration automatically. After setup, verify with:

```bash
pvecm status
```

You should see output like this:

```
Membership information
    Nodeid      Votes    Qdevice    Name
0x00000001          1    A,V,NMW    node-a
0x00000004          1    A,V,NMW    node-b
0x00000000          1               Qdevice

Total votes:   3
Expected votes: 3
Flags:         Quorate Qdevice
```

### Reading the QDevice Flags

The `A,V,NMW` flags next to each node tell you the QDevice's view of that node:

| Flag | Meaning |
|---|---|
| A | Active - the QDevice sees this node |
| V | Voting - the QDevice grants this node a vote |
| NMW | Not Master Wins - the ffsplit algorithm is active |

If you see `NA` or missing flags, the QDevice has lost contact with that node - investigate immediately.

---

## Ffsplit Algorithm

The `ffsplit` (fifty-fifty split) algorithm is the QDevice voting model for 2-node clusters. When both nodes are online, the QDevice grants both nodes votes and holds one itself - 3 total. When a node disappears, the QDevice sides with the surviving node, maintaining quorum.

This is different from the `lms` (last man standing) algorithm, which can cause issues in network partitions. Ffsplit is the recommended choice for 2-node deployments.

---

## What Runs on Each Node

### Primary Node - Workloads

| Type | Name | Purpose |
|---|---|---|
| VM | Home Assistant | Home automation hub (USB passthrough) |
| VM | peaNUT | UPS monitoring via NUT (USB passthrough) |
| VM | PBS | Proxmox Backup Server - home site primary |
| LXC | AdGuard Home | DNS filtering for all home VLANs |
| LXC | Plex | Media server |
| LXC | Newt | Pangolin tunnel agent |
| LXC | WireGuard | Dual WireGuard tunnels to DC |

### Secondary Node - Management

| Type | Name | Purpose |
|---|---|---|
| LXC | PDM | Proxmox Datacenter Manager |
| LXC | PocketID | OIDC identity provider |
| LXC | Patchmon | Patch monitoring |

The primary node handles the heavier workloads and anything requiring hardware passthrough (USB devices for Home Assistant and UPS monitoring). The secondary node runs lightweight management services.

---

## Proxmox Datacenter Manager

PDM is the glue between sites. It runs as an LXC on the secondary home node and connects to both the home cluster and the standalone DC node over the WireGuard management tunnel.

### Key Capabilities

- **Cross-site migration** - migrate VMs and LXCs between home and DC without manual backup/restore cycles. I used this extensively when moving several containers from home to DC in a single batch operation
- **Unified dashboard** - see CPU, RAM, and storage utilization across all nodes in one view
- **Centralized task monitoring** - track running jobs regardless of which site originated them

### Migration Prerequisites

Cross-site PDM migrations require:

1. Both source and target nodes reachable (WireGuard management tunnel must be healthy)
2. Compatible storage configurations at both ends
3. Network connectivity for the migration data stream

I always verify WireGuard tunnel status before starting a cross-site migration - losing connectivity mid-migration is not a fun recovery scenario.

---

## The Datacenter Node - Standalone by Design

The DC runs a standalone Proxmox VE node at Hetzner. It is intentionally **not** part of the home cluster for several reasons:

| Reason | Detail |
|---|---|
| Failure domain isolation | A home internet outage should not affect DC operations |
| Latency | WAN latency would degrade Corosync heartbeats |
| Independence | DC workloads must survive if the home site goes dark |
| Security boundary | Different trust zones, different firewall policies |

The DC node runs production workloads - email, web hosting, the SOC stack, GitLab, and all IaC tooling. It has its own OPNsense firewall VM, its own PBS instance, and its own Synology NAS for backup redundancy.

### DC Hardware

| Component | Specification |
|---|---|
| CPU | Intel Core i7 - 6 cores / 12 threads |
| Memory | 128 GB RAM |
| Primary storage | ZFS on NVMe (~950 GB) |
| Supplementary storage | NFS from Synology (~14 TB) |
| Network | Multiple bridges for VLAN segmentation |

---

## VMID Allocation Strategy

Across both sites, I maintain a structured VMID allocation scheme:

| Range | Purpose |
|---|---|
| 100-199 | Core infrastructure (NAS, admin) |
| 1000-1099 | Infrastructure VMs and LXCs (firewall, PBS, admin) |
| 2000-2099 | Security stack (Wazuh, TheHive, MISP) |
| 3000-3099 | Client-facing services and tunnel endpoints |
| 4000-4099 | Business services |
| 5000-5099 | Development and miscellaneous |
| 6000-6099 | IaC stack (GitLab, Runner, tooling) |

This makes it easy to identify a workload's purpose at a glance - no guessing what VM 3001 does when you know the 3000 range is for client services.

---

## Operational Lessons

**QDevice on a NAS is elegant.** The Synology is always on, always accessible, and adding a Docker container to it costs nothing. No extra hardware, no extra power consumption.

**Ffsplit over LMS.** For a 2-node cluster, the ffsplit algorithm is the only sensible choice. LMS can cause unpredictable behavior during network partitions.

**Decommissioning nodes is healthy.** When I retired the third node and moved PBS to the primary, I simplified the cluster without losing capability. Fewer nodes means less to maintain, patch, and troubleshoot.

**PDM requires tunnel discipline.** Cross-site features are powerful but only work when connectivity is solid. I never touch the WireGuard management tunnel during a migration window.

**Standalone DC is the right call.** Corosync over WAN is fragile. Two independent Proxmox deployments connected by WireGuard and PDM gives me the best of both worlds - central visibility without interdependence.

---

## Monitoring Quorum Health

I check quorum health regularly as part of routine maintenance:

```bash
# Check cluster status and quorum
pvecm status

# Verify QDevice specifically
corosync-qdevice-tool -s

# Quick one-liner to confirm quorum
pvecm expected 3 && echo "Quorum OK"
```

If the QDevice goes offline (Synology reboot, Docker restart), the cluster still runs - but you lose the safety net. Both nodes need to stay up or you lose quorum. I treat QDevice downtime like running with no backup - technically fine, but get it fixed quickly.

---

## What's Next

The cluster is stable and well-tested. Future plans include adding GPU passthrough for a local AI stack and potentially expanding to a third physical node if Ceph becomes practical for replicated storage. For now, two nodes plus QDevice gives me everything I need - high availability, simple operations, and clean failure domains.

The other posts in this series cover the OPNsense firewall setup, the dual WireGuard tunnels, and the multi-site PBS replication strategy that ties it all together.
