---
layout: post
title: "Backup & Disaster Recovery — The 3-2-1+ Strategy"
date: 2026-03-29 14:00:00 +0200
categories: homelab
tags: backup disaster-recovery proxmox-backup-server wasabi synology
---

Backups are the part of homelab infrastructure nobody finds exciting — until they need one. This post covers my multi-site, multi-technology backup strategy and the disaster recovery plan that gives me confidence to experiment freely.

## The 3-2-1+ Model

The classic 3-2-1 rule says: 3 copies of data, on 2 different media types, with 1 copy offsite. I extend this with a **+** for cloud redundancy:

| Copy | Location | Technology |
|---|---|---|
| 1 | Home PBS (local) | Proxmox Backup Server on ZFS + Synology iSCSI |
| 2 | DC PBS (offsite) | Proxmox Backup Server — pull sync from home |
| 3 | Wasabi S3 (cloud) | PBS native S3 sync |
| 4 | Synology NAS + Wasabi (contingency) | Raw vzdump archives via NFS + rclone |

Every VM and LXC across both sites is protected by at least three independent backup copies.

---

## Proxmox Backup Server — The Core

PBS is the workhorse. Two instances — one at home, one in the DC — handle all VM and container backups with deduplication, encryption, and verification.

### PBS Home

Runs as a VM on the primary home node. It has two datastores:

- **Local ZFS** — fast backups on the same NVMe pool as the VMs
- **Synology iSCSI LUN** — additional capacity from the NAS over the storage VLAN

Home PVE pushes all VM/LXC backups to PBS Home daily. This is the fastest recovery path — restoring from local PBS takes minutes.

### PBS DC

Runs as a VM on the Hetzner node. It serves two purposes:

1. **Local DC backups** — DC PVE pushes all workloads to PBS DC daily
2. **Home replica** — PBS DC pulls a sync from PBS Home daily over the WireGuard backup tunnel

The pull sync means the DC always has a recent copy of every home workload, stored in a separate namespace. If the home site burns down, I can restore everything from DC.

---

## Backup Flow Summary

Nine active backup flows keep data moving across four locations:

| Flow | Path | Method | Schedule |
|---|---|---|---|
| F1 | Home PVE → Home PBS | PBS push | Daily 01:00 |
| F2 | Home PBS → DC PBS | PBS pull sync | Daily 05:00 |
| F3 | DC PBS → Wasabi S3 | PBS native S3 sync | Daily |
| F4 | DC PVE → DC PBS | PBS push | Daily 02:00 |
| F5 | Home PVE → Home Synology | vzdump NFS | Weekly + Monthly |
| F6 | Home Synology → DC Synology | rsync over SSH | Daily 03:20 |
| F7 | DC PVE → DC Synology | vzdump NFS | Daily 04:00 |
| F8 | DC Synology → Wasabi S3 | rclone sync | Daily |
| F9 | Home PBS ↔ Synology | iSCSI LUN | Always-on |

The schedule is staggered so flows don't compete for bandwidth — home backups finish before the cross-site sync begins.

---

## Synology NAS — iSCSI and NFS Target

The Synology DS423+ serves multiple backup roles:

- **iSCSI LUN** for PBS Home — presented as a block device over the storage VLAN
- **NFS export** for vzdump — weekly and monthly raw backup archives
- **QDevice host** — also runs the corosync QDevice for cluster quorum
- **rsync source** — pushes vzdump archives to the DC Synology daily

The Synology is approaching 80% capacity, so I monitor it closely. Raw vzdump files are larger than deduplicated PBS backups, but they're valuable as a technology-independent contingency — I can restore a vzdump archive on any Proxmox node without needing PBS.

---

## Wasabi S3 — Cloud Cold Storage

Wasabi S3 is the final tier — geographically independent cloud storage in EU-Central.

Two buckets:

| Bucket | Source | Content |
|---|---|---|
| PBS backups | DC PBS | Deduplicated PBS datastore (~768 GB and growing) |
| vzdump archives | DC Synology | Raw vzdump files with smart recycle retention |

PBS has **native S3 datastore support**, which means the sync is built into PBS itself — no external scripts or cron jobs needed. The DC Synology uses rclone for its vzdump sync.

### Why Wasabi?

During initial cloud backup evaluation, I trialed both Wasabi and Backblaze B2. Backblaze had a 1 GB/day free-tier upload cap that made the initial seed painfully slow. Wasabi's flat-rate pricing with no egress fees won out.

---

## The Borg Decommission

The original cloud backup chain was: PBS → Borg → Hetzner Storage Box → rclone → Wasabi. This worked, but it was fragile — four tools in a chain, each with its own failure modes.

When PBS added native S3 datastore support, I replaced the entire chain:

**Before:** PBS → Borg export → Hetzner Storage Box → rclone → Wasabi
**After:** PBS → Wasabi S3 (native sync)

Borg was uninstalled from PBS DC, all systemd timers and scripts cleaned up, and the Hetzner Storage Box decommissioned. The credentials are retained in Passbolt for reference, but the service is gone.

---

## Disaster Recovery Plan

Backups are useless without tested recovery procedures. I maintain a DR test plan with three tiers:

### Type 1 — Tabletop (After Major Changes)

Walk through recovery steps without executing. Scenarios: VM deletion, data corruption, PBS failure, site outage.

### Type 2 — Partial Restore (Quarterly)

Actually restore something:
- File-level restore from PBS
- Full VM restore to an isolated network
- LXC restore and verification

### Type 3 — Full DR (Annually)

Complete rebuild: reinstall Proxmox → reconfigure ZFS → deploy PBS → restore all workloads.

### Recovery Time Targets

| Scenario | Severity | RTO | Source |
|---|---|---|---|
| Accidental VM deletion | Low | ~15 min | PBS Home |
| PBS datastore corruption | Medium | ~1 hour | DC PBS |
| Home site failure | High | 2–4 hours | DC PBS |
| Ransomware / mass deletion | High | 4–8 hours | Wasabi S3 |
| Full site disaster | Critical | 8–24 hours | Wasabi S3 |

The key insight: PBS local restore is measured in minutes. Cross-site restore is hours. Cloud restore is a day. Design your backup tiers accordingly.

---

## Lessons Learned

- **Deduplicated backups (PBS) for speed, raw vzdump for independence** — having both gives you options
- **Pull sync > push sync for cross-site** — the DC pulling from home means home doesn't need DC credentials
- **Test your restores** — a backup you haven't tested is a hope, not a plan
- **Simplify the chain** — Borg → Hetzner → rclone → Wasabi was clever but fragile. PBS native S3 is one step
- **Monitor capacity** — NAS storage fills up faster than you think, especially with raw vzdump retention
- **Stagger your schedules** — backup jobs competing for bandwidth and I/O cause failures

Next: the self-hosted services that all this infrastructure exists to support.
