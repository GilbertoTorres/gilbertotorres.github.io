---
layout: post
title: "Proxmox Backup Server - Multi-Site Replication"
date: 2026-03-29 17:30:00 +0200
categories: setup
tags: proxmox-backup-server backup replication disaster-recovery zfs
---

A backup strategy is only as good as the number of independent copies you maintain and how quickly you can restore from them. This post covers my multi-site Proxmox Backup Server deployment - two PBS instances, cross-site replication over WireGuard, cloud offloading to Wasabi S3, and the lessons learned building a 3-2-1+ backup architecture.

## Strategy - 3-2-1 Plus Cloud

The classic 3-2-1 rule says: three copies of your data, on two different storage technologies, with one copy offsite. I extend this with a cloud tier for geographic redundancy beyond my own infrastructure:

| Copy | Location | Technology | Purpose |
|---|---|---|---|
| Primary | Home PBS (local ZFS) | PBS + ZFS | Fast local backups and restores |
| Secondary | Home PBS (Synology iSCSI LUN) | PBS + iSCSI | Overflow storage on NAS hardware |
| Offsite replica | DC PBS | PBS pull sync over WireGuard | Geographic separation |
| Cloud | Wasabi S3 | PBS native S3 sync | Cloud-tier redundancy |
| Contingency | Both Synology NAS devices | Raw vzdump archives | Technology diversity (non-PBS format) |

Every VM and LXC across both sites is protected by at least four independent copies. The primary and secondary copies are at home, the offsite replica is at the DC, and the cloud copy is at Wasabi. Raw vzdump archives on Synology NAS devices provide an additional technology-diverse contingency layer.

---

## Architecture

### PBS Home - The Source of Truth

The home PBS instance runs as a VM on the primary Proxmox node. It serves as the central collection point for all home cluster backups and the source for cross-site replication.

| Item | Detail |
|---|---|
| Type | VM on primary home node |
| NICs | Dual-homed - management VLAN + storage VLAN |
| Datastores | ZFS-backed primary + Synology iSCSI secondary |
| Role | Layer 1 - primary backup target for all home workloads |

#### Dual Datastores

| Datastore | Backing Storage | Purpose |
|---|---|---|
| Primary (Backups) | ZFS pool on host NVMe | Fast daily backups - shared with VMs on the host |
| Secondary (Synology) | iSCSI LUN from Synology NAS | Overflow and redundancy - dedicated block device |

The primary datastore shares the host's ZFS pool with running VMs and LXCs. This is not ideal - a backup surge can compete with VM I/O. But with only a 1 TB NVMe, a dedicated backup disk was not an option after decommissioning the third node. I monitor ZFS pool usage closely and keep it below 80%.

The Synology datastore uses an iSCSI LUN presented as a block device. This gives PBS native filesystem access (ext4 on the LUN) rather than the overhead of NFS. The iSCSI target is bound to the storage VLAN interface on the Synology - not the management interface - to keep backup traffic off the management network.

### PBS DC - The Offsite Replica

The DC PBS instance runs at Hetzner on the standalone Proxmox node. It has two roles: receiving replicated home backups and backing up local DC workloads.

| Item | Detail |
|---|---|
| Type | VM on DC Proxmox node |
| NICs | Dual-homed - management VLAN + backup VLAN |
| Datastores | Local disk + Wasabi S3 (cloud tier) |
| Role | Layer 2 (offsite replica) + Layer 3 (cloud sync) |

#### Dual-Homed Networking

The DC PBS has two network interfaces for a reason:

| Interface | Subnet | Purpose |
|---|---|---|
| Management NIC | DC management VLAN | Default gateway, web UI, admin access |
| Backup NIC | DC backup VLAN | PBS replication traffic, Synology NFS |

Backup traffic must stay on the backup NIC. A static route ensures traffic destined for the home storage subnet routes via the backup interface and through the WireGuard backup tunnel - not the default management gateway. Without this static route, replication silently fails because traffic exits via the wrong interface.

---

## Backup Flows

I maintain nine active backup flows. Here are the key ones:

### Flow 1 - Home PVE to Home PBS (Daily 01:00)

All home VMs and LXCs push to the home PBS primary datastore daily at 01:00. Retention: keep-daily 7, keep-hourly 24.

### Flow 2 - Home PBS to DC PBS - Pull Sync (Daily 05:00)

The DC PBS pulls a replica of all home backups over the WireGuard backup tunnel. This is the core offsite replication flow.

**Why pull instead of push?** Pull sync means the DC side initiates the connection. The home PBS only needs to expose its API - it does not need outbound access to the DC. This matches the security model: the DC pulls from home, home does not push to DC.

The sync uses a dedicated service account on the home PBS with read-only permissions scoped to the backup datastore. The DC stores replicated data in a separate namespace to avoid mixing home replicas with local DC backups.

### Flow 3 - DC PBS to Wasabi S3 (Daily)

The DC PBS pushes its entire datastore (including the home replica namespace) to Wasabi S3 via PBS's native S3 datastore support. This replaced an earlier Borg + Hetzner Storage Box setup that was decommissioned in favor of the simpler native integration.

| S3 Detail | Value |
|---|---|
| Provider | Wasabi |
| Region | EU Central |
| Bucket | Dedicated PBS bucket |
| Access | IAM user with S3 full access policy |
| Cache | Local cache directory on DC PBS |

### Flow 4 - DC PVE to DC PBS (Daily 02:00)

All DC VMs and LXCs push to the DC PBS daily at 02:00. Retention: keep-daily 7, keep-weekly 4.

### Flow 5 - Raw vzdump to Synology NAS (Weekly/Monthly)

Independent of PBS, raw vzdump archives go to the home Synology via NFS. These are technology-diverse backups - if PBS has a catastrophic bug, vzdump archives are a separate recovery path.

| Schedule | Retention |
|---|---|
| Weekly (Sunday) | Keep last 4 |
| Monthly (1st) | Keep last 6 |

### Flow 6 - Synology-to-Synology rsync (Daily 03:20)

The home Synology pushes vzdump archives to the DC Synology via rsync over SSH through the WireGuard backup tunnel. The rsync runs without `--delete` - both sites' vzdump files coexist in the same directory on the DC Synology. This provides geographic redundancy for the raw vzdump contingency layer.

### Flow 7 - DC Synology to Wasabi S3 (Daily)

The DC Synology pushes its vzdump archive directory to a dedicated Wasabi bucket via rclone. This is the cloud copy of the raw vzdump contingency layer.

---

## Full Backup Schedule

| Time | Flow | Direction |
|---|---|---|
| 01:00 | Home PVE to Home PBS | Local push |
| 02:00 | DC PVE to DC PBS | Local push |
| Sunday 02:00 | Home PVE to Synology (vzdump) | Weekly NFS push |
| 1st of month | Home PVE to Synology (vzdump) | Monthly NFS push |
| 03:20 | Synology to Synology rsync | Cross-site push via WireGuard |
| 04:00 | DC PVE to DC Synology (vzdump) | Local NFS push |
| 05:00 | Home PBS to DC PBS pull sync | Cross-site pull via WireGuard |
| Daily | DC PBS to Wasabi S3 | Cloud push |
| Daily | DC Synology to Wasabi S3 | Cloud push via rclone |

The schedule is staggered to avoid overlapping I/O windows. Home backups complete by 02:00, DC backups by 03:00, cross-site replication by 06:00, and cloud sync fills in the remaining hours.

---

## User and Service Account Model

Both PBS instances use separate user accounts for each purpose:

### Home PBS Accounts

| Account | Role | Used By |
|---|---|---|
| root | Superuser | Admin access |
| pve user | Backup operator | Home Proxmox backup jobs |
| DC sync user | Read-only sync | DC PBS pull sync credential |
| Monitor user | Read-only monitor | Pulse uptime dashboard |

### DC PBS Accounts

| Account | Role | Used By |
|---|---|---|
| root | Superuser | Admin access |
| backup user | Backup operator | DC Proxmox backup jobs + pull sync |

Principle of least privilege - the DC sync account on the home PBS can only read backup data, not delete it. This means a compromised DC cannot wipe home backups.

---

## Cloud Migration Story - From Borg to Native S3

When I first set up offsite cloud backups, I used a Borg + Hetzner Storage Box stack. The flow was: DC PBS exports to Borg, Borg pushes to Hetzner Storage Box, and rclone mirrors to Wasabi as a secondary cloud copy.

This stack worked but was complex - four moving parts (PBS export, Borg, Hetzner, rclone) with systemd timers, custom scripts, and multiple credential stores. When Proxmox added native S3 datastore support, I migrated the entire cloud tier to a single flow: DC PBS syncs directly to Wasabi S3.

The decommission involved removing:

- Borg packages and scripts
- All systemd timers and services for the Borg pipeline
- Hetzner Storage Box credentials and targets
- Backblaze B2 bucket (trialled and rejected due to upload caps)
- rclone remotes for the old workflow

The result is a single, clean, PBS-native sync job replacing an entire pipeline.

---

## iSCSI Persistence - The Silent Killer

The Synology iSCSI LUN backing the home PBS secondary datastore is rock-solid when configured correctly. When it's not, you get silent data loss.

### Key Configuration Points

| Setting | Value | Why |
|---|---|---|
| Target binding | Storage VLAN interface only | Prevents iSCSI traffic from traversing management VLAN |
| Startup mode | Automatic | iSCSI reconnects after reboot without manual intervention |
| fstab entry | By UUID | Device names can change after reboot; UUID is stable |

### Common Gotchas

- **Ghost sessions after DSM reboot.** If the iSCSI target reboots, stale sessions can prevent reconnection. Logout all sessions, restart the iSCSI initiator, then re-login.
- **VLAN binding drift.** DSM updates can reset the iSCSI target's network portal binding. After every DSM update, verify the target is still bound to the storage VLAN interface.
- **Device name changes.** The iSCSI LUN might appear as `/dev/sdc` on one boot and `/dev/sdd` on the next. Always mount by UUID in fstab, never by device name.

---

## NFS and Unprivileged LXC Backups

When backing up unprivileged LXC containers to NFS storage (like the Synology vzdump share), you can hit permission failures due to user namespace ID mapping. The LXC runs as a non-root user on the host, and NFS doesn't understand the mapped UIDs.

The fix is straightforward - set `tmpdir: /var/tmp` in `/etc/vzdump.conf`. This tells vzdump to use a local temporary directory instead of the NFS mount for intermediate files, avoiding the permission issue entirely.

---

## Notifications

Both PBS instances send email notifications for backup job completions, failures, and warnings. They relay through the internal Mailcow instance via Postfix:

| Instance | Relay Target | Status |
|---|---|---|
| Home PBS | Mail server via submission port | Active |
| DC PBS | Mailcow internal IP via submission port | Active |

The DC PBS relays to Mailcow's internal IP on the clients VLAN - not the public domain - to avoid hairpin NAT.

---

## Capacity Planning

| Location | Storage | Used | Free | Utilization |
|---|---|---|---|---|
| Home PBS - ZFS datastore | ~1 TB (shared with VMs) | Monitor closely | Below 80% target | Shared pool risk |
| Home PBS - Synology LUN | Synology iSCSI | Managed by PBS GC | Synology capacity | ~78% on NAS |
| DC PBS - Local disk | ~500 GB | ~170 GB | ~300 GB | 36% |
| DC PBS - Wasabi S3 | Cloud (unlimited) | Growing | N/A | Pay per GB |
| DC Synology - NFS | ~14 TB | ~260 GB | ~14 TB | 2% |

The home Synology NAS at 78% capacity is the most concerning metric. If it fills up, the vzdump contingency copies and the iSCSI LUN for PBS both stop working. I have alerts set for 85% and 90% thresholds.

---

## Disaster Recovery Scenarios

| Scenario | Recovery Path |
|---|---|
| Single VM/LXC loss | Restore from Home PBS (fastest - local ZFS) |
| Home node failure | Restore from Home PBS on surviving node |
| Home site total loss | Restore from DC PBS (offsite replica) |
| DC site total loss | Home PBS retains all home backups; DC workloads restore from DC PBS Wasabi S3 |
| Both sites total loss | Restore from Wasabi S3 (cloud tier) |
| PBS software bug | Restore from raw vzdump archives on Synology NAS |

The key insight is that each layer is independent. A PBS bug does not affect vzdump archives. A WireGuard tunnel failure does not affect local backups. A site loss does not affect the cloud tier.

---

## Operational Lessons

- **Pull sync is more secure than push.** The remote site initiates replication with read-only credentials. The source site never needs outbound access to the destination.

- **Dual-homed PBS is essential.** Backup traffic on the management network competes with admin access and monitoring. Separate interfaces, separate VLANs, separate subnets.

- **Native S3 beats Borg pipelines.** Fewer moving parts means fewer failure modes. The migration from Borg to native S3 eliminated four services, three scripts, and two credential stores.

- **Raw vzdump is your insurance policy.** PBS is excellent, but technology diversity matters. If PBS has a critical bug, vzdump archives in a simple tar format can be restored with basic Linux tools.

- **iSCSI needs babysitting.** It's fast and efficient, but the connection is fragile across reboots and NAS updates. Always mount by UUID, always verify bindings after updates.

- **Wasabi's 90-day minimum billing matters.** Aggressive pruning of recently uploaded objects still incurs charges for the full 90 days. Design your retention policy with this in mind.

---

## What's Next

The backup architecture is comprehensive and well-tested. Future improvements include automating the manual health checks (ZFS pool capacity, iSCSI mount status, sync job verification) into a single dashboard, and investigating Proxmox's upcoming improvements to S3 datastore support for potentially replacing the rclone vzdump-to-Wasabi flow with another native PBS sync job.
