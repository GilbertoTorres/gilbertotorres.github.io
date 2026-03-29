---
layout: post
title: "BiGGie's Homelab — Architecture & Goals"
date: 2026-03-29 10:00:00 +0200
categories: homelab
tags: proxmox networking wireguard backup infrastructure
---

This is the first post on BiGGie's Place — a blog where I document my homelab journey. Let me walk you through what I've built, how it's connected, and where I'm headed.

## The Big Picture

My infrastructure spans two sites connected by WireGuard tunnels:

- **Home Lab** — the primary environment for experimentation, local services, and management
- **Datacenter (Hetzner)** — a resilience layer for offsite backups, security monitoring, and production-facing services

Everything is documented in my internal wiki (the *BiGGie Grimoire*), and this blog is where I share the highlights publicly.

---

## Home Cluster

A 2-node **Proxmox VE** cluster running on ZFS, with QDevice quorum hosted on my Synology NAS.

| Node | Role | Storage |
|---|---|---|
| pve0 | Primary node | ZFS on NVMe |
| pven | Secondary node | ZFS |
| Synology DS423+ | NAS + QDevice host | NFS / iSCSI |

### What's running at home

| Service | Role |
|---|---|
| AdGuard Home | DNS filtering & ad blocking |
| Home Assistant | Home automation |
| Plex | Media server |
| PBS Home | Proxmox Backup Server |
| WireGuard (WG-Dashboard) | VPN tunnels to DC |
| Pangolin & Newt | Zero-trust reverse proxy |
| Proxmox Datacenter Manager | Multi-site cluster management |
| PocketID | OIDC identity provider |

---

## Datacenter (Hetzner)

A standalone **Proxmox VE** node with **OPNsense** as the firewall/router. This is where the production-facing and security workloads live.

| Service | Role |
|---|---|
| OPNsense | Firewall & router |
| Mailcow | Self-hosted email |
| HestiaCP | Web & DNS hosting |
| Wazuh SIEM | Security monitoring |
| TheHive & MISP | Incident response & threat intel |
| GitLab CE | IaC source of truth |
| Technitium DNS | DC DNS resolver |
| Passbolt | Password manager |
| Guacamole | Remote access gateway |
| MeshCentral | Remote management |
| RackPeek | Rack inventory |

---

## Network Design

The network is segmented into purpose-specific VLANs at both sites:

| VLAN | Purpose |
|---|---|
| Management | Core infrastructure & admin access |
| WLAN | Wireless clients (home only) |
| Lab | Experimentation & dev workloads |
| IoT | Smart home devices (home only) |
| Storage | NFS / iSCSI / backup traffic |
| DMZ | Isolated services |

### Cross-site connectivity

Two WireGuard point-to-point tunnels link the sites:

- **wg0 (Backup)** — PBS replication between home storage and DC backup VLANs
- **wg1 (Management)** — cross-site admin access between management VLANs

### External access

All public-facing services go through **Pangolin + Newt** tunnel agents. There are no inbound firewall rules — zero-trust perimeter by design.

### DNS

- **Home:** AdGuard Home handles DNS filtering and resolves `h.biggie.be`
- **DC:** Technitium DNS is primary for `d.biggie.be`, forwards home zones back to AdGuard
- **Public:** Cloudflare manages the external `biggie.be` zone

---

## Backup & DR

| Layer | Solution | Target |
|---|---|---|
| VM/LXC backups | Proxmox Backup Server | Local (PBS Home + PBS DC) |
| Offsite replication | PBS replication over WireGuard | DC ↔ Home |
| File-level offsite | Borg | Hetzner Storage Box |
| Cloud cold storage | Wasabi S3 | ~768 GB and growing |

DR test scenarios are documented and scheduled quarterly.

---

## IaC & Automation

Infrastructure is managed as code wherever possible:

- **Terraform** — provisions all Proxmox resources (home + DC), state stored in GitLab
- **Ansible** — configuration management and playbooks
- **GitLab CI** — pipelines for plan/apply workflows
- **GitLab CE** (self-hosted in DC) — the single source of truth

---

## Goals & Roadmap

Here's what I'm working on and where I'm heading:

### Active

- **Wazuh + TheHive + MISP integration** — building a full SOC stack with automated alert triage
- **Internal DNS migration** — adding `h.biggie.be` / `d.biggie.be` subdomains for all Pangolin resources

### Planned

- **Local AI stack** — Ollama + llama.cpp on an RTX 5080, with agent orchestration via Paperclip
- **Honeypot infrastructure** — deployed on the DC node, alerts feeding into Wazuh
- **Quarterly DR test execution** — formalizing and automating disaster recovery drills

### Backlog

- Centralized LDAP identity (Synology-backed SSO)
- Forgejo migration from GitLab (waiting on Terraform state backend support)

---

## What's Next

This blog will be a running log of everything I build, break, and fix. Expect posts about Proxmox, networking, self-hosting, IaC, and security monitoring.

Stay tuned.
