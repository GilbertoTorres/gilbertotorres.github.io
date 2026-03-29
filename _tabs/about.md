---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

## Hey, I'm Gilberto

Welcome to **BiGGie's Place** — my personal blog where I document everything I learn and build in my homelab.

I'm a sysadmin and self-hoster based in Belgium, passionate about infrastructure, automation, and keeping things running smoothly. This blog is where I share notes, guides, and lessons learned from running my own infrastructure.

---

## The Homelab

### Home Cluster

A 2-node **Proxmox VE** cluster (`pve0` + `pven`) with QDevice quorum, backed by ZFS storage and a **Synology DS423+** NAS for NFS/iSCSI and offsite staging.

**What's running at home:**

| Service | What it does |
|---|---|
| AdGuard Home | DNS filtering & ad blocking |
| Home Assistant | Home automation |
| Plex | Media server |
| PBS Home | Proxmox Backup Server |
| WireGuard | VPN tunnels between home & DC |
| Pangolin & Newt | Zero-trust reverse proxy & tunnel agents |

### Datacenter (Hetzner)

A standalone **Proxmox VE** node at Hetzner with **OPNsense** as the firewall/router, providing the resilience layer and hosting production-facing services.

**What's running in the DC:**

| Service | What it does |
|---|---|
| Mailcow | Self-hosted email |
| HestiaCP | Web & DNS hosting panel |
| Wazuh SIEM | Security monitoring (+ TheHive & MISP) |
| GitLab CE | IaC source of truth |
| Technitium DNS | DC DNS resolver |
| Passbolt | Password manager |
| PocketID | OIDC identity provider |
| Guacamole | Remote access gateway |
| MeshCentral | Remote management |
| RackPeek | Rack inventory |

### Offsite Backup

- **Wasabi S3** — cloud cold storage
- **Hetzner Storage Box** — offsite Borg target

---

## The Stack

- **Virtualization:** Proxmox VE, LXC, KVM
- **Networking:** OPNsense, WireGuard, UniFi, VLANs
- **Storage:** ZFS, Ceph RBD, Synology NAS
- **Backup:** Proxmox Backup Server, Borg, Wasabi S3
- **IaC:** Terraform, Ansible, GitLab CI
- **Monitoring:** Wazuh SIEM, TheHive, MISP
- **Reverse Proxy:** Pangolin + Traefik (zero-trust, no inbound firewall rules)

---

## Get in Touch

- [GitHub](https://github.com/gilbertotorres)
- [LinkedIn](https://www.linkedin.com/in/gilberto-torres)
- [Facebook](https://www.facebook.com/gilberto.j.p.torres)
