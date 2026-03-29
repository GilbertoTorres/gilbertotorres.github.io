---
layout: post
title: "Self-Hosting GitLab CE as an IaC Control Plane"
date: 2026-03-29 19:00:00 +0200
categories: setup
tags: gitlab iac ci-cd git self-hosted
---

Every infrastructure-as-code workflow needs a control plane - a single source of truth for code, state, and execution. This post covers why I chose self-hosted GitLab CE for that role, how it's configured, and the lessons learned running it as the backbone of my homelab IaC stack.

## Why GitLab CE?

I evaluated several options before settling on GitLab CE: GitHub (hosted), Gitea, Forgejo, and self-hosted GitLab. The deciding factor was a single feature - GitLab's **built-in Terraform HTTP state backend**.

Terraform needs somewhere to store its state file. Most setups use S3 + DynamoDB for locking, or Consul, or Terraform Cloud. All of these add operational overhead - another service to maintain, another thing to back up, another failure point. GitLab CE includes a Terraform state backend out of the box. You point your Terraform backend configuration at GitLab's API, and it handles state storage and locking with zero additional infrastructure.

That alone justified the heavier footprint compared to Gitea or Forgejo. The runner, CI/CD pipelines, and container registry are bonuses.

---

## Architecture

GitLab CE runs as a VM on the DC Proxmox node. It's an Omnibus installation on Ubuntu 24.04 LTS - the all-in-one package that includes PostgreSQL, Redis, Puma, Sidekiq, and the web frontend.

| Component | Detail |
|---|---|
| Installation method | Omnibus (all-in-one) |
| OS | Ubuntu 24.04 LTS |
| Resources | 4 vCPU, 8 GB RAM, 80 GB disk |
| Access | Internal URL + external URL via reverse proxy |
| Version | GitLab CE 18.x (Omnibus, regularly updated) |

### Why a VM Instead of an LXC?

GitLab is one of the few workloads I run as a full VM rather than an LXC container. The reasoning:

- GitLab's Omnibus package bundles its own PostgreSQL, Redis, and multiple Ruby processes - it manages its own service lifecycle in ways that can conflict with LXC's shared kernel
- Memory usage is more predictable in a VM with dedicated resources
- Backup and restore workflows are simpler when the entire system is a self-contained VM disk

The 8 GB RAM allocation is the minimum I'd recommend. Puma (the web server) and Sidekiq (the background job processor) are the primary consumers. I tuned both down to fit within the allocation.

---

## Key Configuration Decisions

### Puma Worker Tuning

GitLab's default Puma configuration assumes more resources than a homelab typically provides. I reduced the worker count to 2, which is sufficient for a single-user environment and keeps memory usage reasonable.

With 2 Puma workers, the web UI remains responsive for normal operations - browsing repos, reviewing merge requests, and checking pipeline status. If you're running GitLab for a team, you'd want more workers, but for a homelab IaC control plane where I'm the only user, 2 is plenty.

### Sidekiq Concurrency

Similarly, Sidekiq concurrency is set to 5 (down from the default 20). Sidekiq handles background jobs like sending emails, processing webhooks, and updating CI pipeline statuses. With lower concurrency, jobs queue slightly longer during peak activity, but the trade-off in memory savings is worth it.

### Signup Disabled

Public signup is disabled. This is a private IaC control plane - accounts are created manually by the admin. Even though the instance is only accessible through the network perimeter, defense in depth means not relying solely on network controls.

### Mail via Internal Relay

GitLab sends notifications through an internal SMTP relay. Pipeline failures, merge request comments, and system alerts all flow through email. This is especially useful for the apply-stage notifications - I want to know when a Terraform apply succeeds or fails, even if I'm not watching the UI.

---

## Repository Structure

All IaC repositories live under a structured group hierarchy in GitLab:

| Group / Repo | Purpose |
|---|---|
| `infra/dc-pve-terraform` | Proxmox VM/LXC provisioning via Terraform |
| `infra/dc-pve-ansible` | Guest configuration management via Ansible |
| `infra/rackpeek-config` | Physical and logical inventory configuration |

The `infra/` group acts as the namespace for all infrastructure-related repos. This keeps them organized and makes it easy to set group-level CI/CD variables (like Proxmox API tokens) that apply to all repos underneath.

### Branch Protection

The `main` branch on all IaC repos is protected:

- No direct pushes - all changes go through merge requests
- Pipeline must pass before merging
- Force-push is disabled

This means every infrastructure change has an audit trail in the form of a merge request. When something breaks, I can trace back to the exact commit, the pipeline output, and the merge request discussion.

---

## GitLab Runner

CI/CD pipelines don't execute on the GitLab server itself. A dedicated **GitLab Runner** handles all job execution.

| Setting | Detail |
|---|---|
| Type | LXC container on DC Proxmox node |
| Executor | Shell |
| Pre-installed tools | Terraform, Ansible |
| Tags | dc, proxmox, shell |

### Why Shell Executor?

GitLab supports multiple executor types - Docker, Kubernetes, shell, and others. I use the **shell executor** for a specific reason: infrastructure automation tools need direct access to the network and SSH keys. Running Terraform inside a Docker container means dealing with Docker-in-Docker networking, mounting SSH keys into containers, and managing Proxmox API token access across container boundaries.

The shell executor runs jobs directly on the runner's filesystem. Terraform can reach the Proxmox API over the network. Ansible can SSH to hosts using keys already on the runner. It's simpler, faster, and eliminates an entire class of networking and permission issues.

### Runner Registration

The runner is registered with specific tags (`dc`, `proxmox`, `shell`) so CI jobs can target it explicitly. Pipeline definitions use `tags:` to ensure jobs land on the correct runner:

```yaml
plan:
  stage: plan
  tags:
    - dc
    - shell
  script:
    - terraform init
    - terraform plan -out=plan.tfplan
```

This becomes important when (not if) I add runners at other sites or for other purposes.

---

## The Terraform State Backend

This is the feature that makes the entire setup worthwhile. Here's how it works in practice.

### Backend Configuration

In each Terraform project, the backend block points to GitLab's API:

```hcl
terraform {
  backend "http" {
    # Address, lock, and unlock URLs are set via CI/CD variables
    # or the init command
  }
}
```

GitLab provides a dedicated API endpoint for each project's Terraform state. The state file is versioned - you can see the history of state changes in the GitLab UI, which is invaluable for debugging.

### State Locking

State locking prevents concurrent `terraform apply` operations from corrupting the state file. GitLab's backend handles this natively - when one pipeline is running, another will wait for the lock to release. No external locking mechanism needed.

This eliminated one of my biggest operational concerns. With S3 backends, I've seen state corruption from concurrent runs in professional environments. GitLab's locking is reliable and simple.

### State Visibility

The GitLab UI shows Terraform state under **Infrastructure > Terraform states**. You can see:

- Current state file contents
- State version history
- Which pipeline last modified the state
- Lock status

This visibility means I don't need to run `terraform state list` on the command line to check what's managed. The UI gives a quick overview.

---

## Backup Strategy

GitLab hosts the state of my entire infrastructure - losing it would mean losing the source of truth for every VM, container, and configuration. Backup is critical.

### Automated Daily Backups

A daily cron job runs `gitlab-backup create`, which produces a tar archive containing:

- All Git repositories (including wiki pages and snippets)
- Database dump (PostgreSQL)
- Uploads and attachments
- CI/CD job artifacts
- Terraform state files

The backup archive is copied to a remote NFS mount for off-host storage.

### Secrets Backup

GitLab's backup command does **not** include secrets - specifically `gitlab-secrets.json` and the GitLab configuration file. These are backed up separately to the same NFS target. Without the secrets file, you can't decrypt CI/CD variables or two-factor authentication data from a restored backup.

This is a common gotcha. I've seen GitLab restore procedures fail because the secrets file wasn't backed up alongside the data. A separate cron job handles this daily.

### Restore Testing

Having backups is only half the story - you need to verify they work. I periodically test restores by spinning up a temporary GitLab instance, restoring from the latest backup, and verifying that repos, pipelines, and Terraform state are intact.

---

## Operational Notes

### Update Process

GitLab CE is updated through the standard Omnibus package manager:

```bash
apt-get update && apt-get install -y gitlab-ce
gitlab-ctl reconfigure
```

I run updates monthly unless a security patch demands something sooner. The `reconfigure` step is important - it migrates the database schema and restarts services.

### Monitoring

The GitLab runner's health is monitored through the Wazuh agent deployed via Ansible. Both the GitLab VM and runner LXC report to the central Wazuh SIEM, which catches:

- Unexpected service restarts
- Disk space warnings (GitLab grows - CI artifacts and state files accumulate)
- Authentication anomalies
- Package changes outside maintenance windows

### Resource Monitoring

GitLab is one of the heaviest single applications in the homelab. I keep an eye on:

- **Memory** - Puma and Sidekiq are the primary consumers. If the instance starts swapping, response times degrade significantly
- **Disk** - CI artifacts, container registry layers, and backup tars all consume space. A scheduled cleanup job prunes old artifacts
- **CPU** - generally low, but spikes during `gitlab-ctl reconfigure` and CI pipeline bursts

---

## Integration With the Broader Stack

GitLab CE doesn't exist in isolation. It's integrated with several other homelab services:

| Integration | How |
|---|---|
| **Terraform** | State backend, CI plan/apply pipelines |
| **Ansible** | Playbook execution from runner, role testing |
| **Wazuh** | Agent on both GitLab VM and runner LXC |
| **Mail relay** | Pipeline notifications via SMTP |
| **Proxmox** | Runner has API token for Terraform provider |

The runner is the integration hub - it has credentials for Proxmox (Terraform), SSH access to hosts (Ansible), and network access to internal services. Securing the runner is as important as securing GitLab itself.

---

## Lessons Learned

- **GitLab's built-in Terraform state backend is the killer feature** - it eliminates an entire category of infrastructure (S3, DynamoDB, Consul) that you'd otherwise need to manage
- **Tune for your workload** - default Puma and Sidekiq settings assume a team environment. For a single-user homelab, reduce workers and concurrency to save memory
- **Back up the secrets file separately** - `gitlab-backup create` does not include `gitlab-secrets.json`. Lose that file and your backup is incomplete
- **Shell executor over Docker for infra automation** - fighting Docker networking for Terraform and Ansible access isn't worth the isolation benefits
- **Protect main branches** - even when you're the only user, merge request workflows create an audit trail that's invaluable for debugging
- **Plan for growth** - 80 GB of disk sounds like a lot until CI artifacts and container images accumulate. Set up artifact expiration policies early

---

## What's Next

- **Forgejo migration** - I'm watching Forgejo's development closely. Once it supports a Terraform HTTP state backend (or a compatible alternative), I'll likely migrate. Forgejo is dramatically lighter than GitLab CE and starts in seconds instead of minutes
- **GitLab Pages for internal docs** - hosting project documentation alongside the code
- **Expanded CI pipelines** - adding automated testing for Ansible roles (Molecule) and Terraform configurations (tflint, checkov)

GitLab CE is the heaviest single application in my homelab, but it earns its resource allocation. As the IaC control plane, it's the foundation that makes everything else manageable, traceable, and reproducible.
