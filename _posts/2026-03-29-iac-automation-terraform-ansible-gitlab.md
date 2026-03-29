---
layout: post
title: "IaC & Automation — Terraform, Ansible & GitLab CI"
date: 2026-03-29 16:00:00 +0200
categories: homelab
tags: iac terraform ansible gitlab ci-cd automation infrastructure-as-code
---

Manual infrastructure management doesn't scale — even in a homelab. This post covers how I use Terraform, Ansible, and GitLab CI to manage every VM, container, and configuration across both sites as code.

## GitLab CE — The Source of Truth

A self-hosted GitLab CE instance on the DC node serves as the IaC control plane. It provides:

- **Git hosting** for all infrastructure repos
- **Built-in Terraform state backend** with locking (no external S3 or Consul needed)
- **CI/CD pipelines** via a dedicated GitLab Runner
- **GitOps workflow** — all infrastructure changes go through merge requests

### Why Self-Hosted GitLab?

I considered GitHub, Gitea, and Forgejo. GitLab won because of the built-in Terraform HTTP state backend — it stores state files and provides locking out of the box, which eliminates the need to manage a separate state backend. The runner executes on a dedicated LXC with Terraform and Ansible pre-installed.

### Repository Structure

| Repo | Purpose |
|---|---|
| `infra/dc-pve-terraform` | Proxmox VM/LXC provisioning |
| `infra/dc-pve-ansible` | Guest configuration management |
| `infra/rackpeek-config` | Physical/logical inventory config |

---

## Terraform — Zero-Drift Provisioning

Terraform manages all Proxmox resources across both sites using the `bpg/proxmox` provider. Every VM and LXC is defined in code — if it's not in Terraform, it doesn't exist (with documented exceptions).

### What Terraform Manages

- VM and LXC creation, sizing, and network configuration
- Storage allocation and disk provisioning
- Cloud-init templates for initial guest bootstrapping
- Proxmox resource pools and permissions

### The Workflow

```
Developer pushes to feature branch
    ↓
GitLab CI runs terraform plan
    ↓
Merge request shows planned changes
    ↓
Merge to main triggers terraform apply
    ↓
Infrastructure converges to desired state
```

State is stored in GitLab's built-in HTTP backend with automatic locking — no race conditions, no stale state.

### Documented Exceptions

A few resources can't be managed by Terraform:

- **USB passthrough VMs** (Home Assistant, peaNUT) — the Proxmox API token doesn't have permission for USB device assignment; this requires root access
- **OPNsense firewall** — managed through its own UI/API, not Proxmox resources
- **Synology NAS** — no Terraform provider available

These exceptions are documented in the repo README so nobody (including future-me) wastes time trying to import them.

---

## Ansible — Configuration Management

Where Terraform provisions the infrastructure, Ansible configures what's inside it. The `dc-pve-ansible` repo handles post-provisioning setup.

### Key Roles

| Role | Purpose |
|---|---|
| `wazuh_agent` | Deploy and configure Wazuh agents on all guests |
| `postfix_relay` | Configure Postfix as a mail relay on every host |
| `common` | Base packages, SSH hardening, timezone, NTP |

### Wazuh Agent Deployment

The most critical Ansible role deploys Wazuh agents to every DC guest:

- Pinned agent version for consistency
- Auto-enrollment to the Wazuh manager
- Standardized log paths and FIM configuration
- Dry-run support via `--check` mode

When a new LXC is provisioned by Terraform, running the Ansible playbook is all it takes to bring it into the monitoring fold.

### Inventory

Ansible inventory is maintained in the repo and maps to the Proxmox VMID scheme. Groups correspond to service roles (security stack, client services, infrastructure), making it easy to target playbooks.

---

## GitLab CI — Plan/Apply Pipelines

GitLab CI ties Terraform and Ansible together into automated workflows.

### Terraform Pipeline

```yaml
stages:
  - validate
  - plan
  - apply

plan:
  stage: plan
  script:
    - terraform init
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - plan.tfplan

apply:
  stage: apply
  script:
    - terraform apply plan.tfplan
  when: manual
  only:
    - main
```

The `apply` stage requires manual approval — I review the plan output before any changes hit production. This is the safety net that makes GitOps work for infrastructure.

### Runner Setup

The GitLab Runner is a shell executor on a dedicated LXC with:

- Terraform (current stable version)
- Ansible (current stable version)
- SSH keys for accessing Proxmox hosts
- Proxmox API token for Terraform provider authentication

---

## VMware → Proxmox Migration Project

Beyond managing my own homelab, I've developed an Ansible-based framework for migrating infrastructure from VMware to Proxmox VE — published as an open-source project.

### Migration Tracks

The framework supports four approaches depending on the source environment:

| Track | Method | Best For |
|---|---|---|
| **A — Rebuild** | Provision fresh VMs via Ansible | Environments already defined as code |
| **B1 — Native Import** | Proxmox's built-in ESXi import (8.1+) | Direct network path to ESXi host |
| **B2 — qemu-img** | Convert VMDK to Ceph RBD | VMDK accessible via NFS/file transfer |
| **B3 — PXE Streaming** | Stream disk over SSH to Ceph RBD | Air-gapped or legacy environments |

### Why This Exists

The VMware ecosystem's licensing changes pushed many organizations to explore alternatives. Proxmox is the natural landing zone for those environments, but the migration tooling was scattered across blog posts and forum threads. This framework codifies the process into repeatable Ansible playbooks with proper inventory management and multi-environment support.

The project is tested against my homelab before being used in production environments.

---

## The Full Pipeline

Here's how a typical infrastructure change flows:

1. **Code** — define the change in Terraform (new LXC) or Ansible (new configuration)
2. **Push** — commit to a feature branch and push to GitLab
3. **Plan** — CI automatically runs `terraform plan` and posts the output
4. **Review** — inspect the plan in the merge request
5. **Merge** — merge to main
6. **Apply** — manually trigger `terraform apply` in CI
7. **Configure** — run the Ansible playbook to set up the new guest
8. **Monitor** — Wazuh agent auto-enrolls, Postfix relay configured, service is live

From code to monitored production service — fully traceable, version-controlled, and repeatable.

---

## Lessons Learned

- **GitLab's built-in state backend is underrated** — it eliminates the need for S3 + DynamoDB or Consul for state management
- **Manual apply gates save you** — automated apply sounds efficient until a bad plan destroys a production database
- **Document your Terraform exceptions** — not everything can or should be managed by Terraform. Document what's excluded and why
- **Ansible for day-2 operations** — Terraform provisions, Ansible configures. Don't try to make Terraform do configuration management
- **Pin your versions** — Terraform providers, Ansible collections, and agent versions should all be pinned. Drift in tooling causes drift in infrastructure
- **Shell executor over Docker executor** — for infrastructure automation, the shell executor on a dedicated LXC is simpler and faster than Docker-in-Docker

---

## What's Next

- **Forgejo migration** — GitLab CE works, but Forgejo is lighter and faster. Waiting on Terraform state backend support before migrating
- **Expanded Ansible coverage** — more roles for standardized host configuration
- **Automated DR testing** — CI pipeline that provisions a test environment, restores from backup, and validates services

This IaC stack is what makes a homelab manageable long-term. Without it, every change is a snowflake. With it, the infrastructure is a living codebase — documented, version-controlled, and reproducible.
