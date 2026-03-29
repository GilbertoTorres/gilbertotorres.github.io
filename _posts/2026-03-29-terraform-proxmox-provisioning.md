---
layout: post
title: "Managing Proxmox with Terraform"
date: 2026-03-29 19:10:00 +0200
categories: setup
tags: terraform proxmox iac provisioning infrastructure-as-code
---

Every VM and LXC in my homelab is defined in Terraform. If a resource isn't in the Terraform state, it either doesn't exist or it's a documented exception. This post walks through how I use Terraform with the bpg/proxmox provider to manage all Proxmox resources, the CI/CD pipeline that executes it, and the patterns I've developed for a reliable provisioning workflow.

## Why Terraform for Proxmox?

Proxmox has a perfectly functional web UI. You can click through VM creation in a few minutes. But click-ops has problems that compound over time:

- **No audit trail** - who created this VM and when? What changed last Tuesday?
- **Drift** - manual changes accumulate and nobody knows the actual state
- **Reproducibility** - rebuilding after a failure means remembering (or guessing) the original configuration
- **Scaling** - provisioning one VM is fast. Provisioning twenty is tedious. Ensuring all twenty are consistent is impossible

Terraform solves all of these. The configuration is version-controlled in GitLab, changes go through merge requests with plan output, and the state file is the single source of truth for what exists.

---

## The bpg/proxmox Provider

I use the [`bpg/proxmox`](https://registry.terraform.io/providers/bpg/proxmox/latest) Terraform provider, which is the most actively maintained Proxmox provider available. It covers the full Proxmox API surface.

### Provider Configuration

The provider authenticates to Proxmox using an API token - not a username/password. API tokens are scoped and can be revoked independently, which is better for automation.

```hcl
provider "proxmox" {
  endpoint = var.proxmox_url
  api_token = var.proxmox_api_token
  insecure = false

  ssh {
    agent = true
  }
}
```

The API token is stored as a CI/CD variable in GitLab - never committed to the repository. The SSH configuration enables the provider to perform operations that require direct host access (like uploading ISO images or cloud-init configs).

### What the Provider Manages

| Resource Type | Terraform Resource |
|---|---|
| Virtual machines | `proxmox_virtual_environment_vm` |
| LXC containers | `proxmox_virtual_environment_container` |
| Cloud-init drives | `proxmox_virtual_environment_file` |
| Storage pools | `proxmox_virtual_environment_pool` |
| DNS records | via cloud-init integration |
| Network config | Bridge/VLAN assignment per VM/LXC |

---

## Repository Structure

The Terraform codebase lives in `infra/dc-pve-terraform` in GitLab. Here's the general layout:

```
dc-pve-terraform/
  main.tf              # Provider config, backend config
  variables.tf         # Input variables
  outputs.tf           # Useful outputs (IPs, VMIDs)
  terraform.tfvars     # Non-sensitive variable values
  versions.tf          # Provider version constraints
  modules/
    vm/                # Reusable VM module
    lxc/               # Reusable LXC module
  resources/
    security-stack.tf  # Wazuh, TheHive, MISP
    iac-stack.tf       # GitLab, Runner, RackPeek
    services.tf        # Mailcow, HestiaCP, etc.
    infrastructure.tf  # Core infra (DNS, tunnel agents)
```

### Module Pattern

I use local modules for VMs and LXCs to enforce consistency. Every VM and LXC goes through a module that sets sensible defaults:

```hcl
module "wazuh" {
  source = "./modules/lxc"

  hostname    = "wazuh"
  vmid        = 2000
  cores       = 4
  memory      = 4096
  disk_size   = 25
  target_node = var.dc_node
  vlan_tag    = var.mgmt_vlan
  template    = var.ubuntu_template
  ssh_keys    = var.ssh_public_keys
}
```

The module handles boilerplate - network configuration, cloud-init bootstrapping, start-on-boot settings, and tagging. Individual resource definitions stay clean and focused on what makes each workload unique.

---

## State Management

Terraform state is the mapping between your configuration and the real-world resources. Losing or corrupting it is one of the worst things that can happen in a Terraform workflow.

### GitLab HTTP State Backend

State is stored in GitLab's built-in Terraform HTTP state backend:

```hcl
terraform {
  backend "http" {
    # Configured via environment variables in CI
  }
}
```

This gives me:

- **Versioned state** - every `terraform apply` creates a new state version. I can see the history and roll back if needed
- **Automatic locking** - concurrent applies are blocked. No race conditions
- **No extra infrastructure** - no S3 bucket, no DynamoDB table, no Consul cluster
- **Integrated visibility** - state is viewable in the GitLab UI alongside the code

### State Hygiene

A few practices I follow to keep the state clean:

- **Never edit state manually** unless recovering from corruption. `terraform state mv` and `terraform state rm` are the safe alternatives
- **Import existing resources** before managing them. If a VM was created manually, import it into state before adding it to configuration
- **Regular state list reviews** - periodically run `terraform state list` to verify the state matches expectations

---

## The CI/CD Pipeline

All Terraform operations run through GitLab CI - never from my local machine. This ensures consistency (same Terraform version, same provider versions, same network access) and creates an audit trail.

### Pipeline Stages

```yaml
stages:
  - validate
  - plan
  - apply

validate:
  stage: validate
  tags:
    - dc
    - shell
  script:
    - terraform init
    - terraform validate
    - terraform fmt -check

plan:
  stage: plan
  tags:
    - dc
    - shell
  script:
    - terraform init
    - terraform plan -out=plan.tfplan
  artifacts:
    paths:
      - plan.tfplan
    expire_in: 1 hour

apply:
  stage: apply
  tags:
    - dc
    - shell
  script:
    - terraform apply plan.tfplan
  when: manual
  only:
    - main
```

### The Validate Stage

Before anything else, the pipeline checks that the configuration is syntactically valid and properly formatted. `terraform fmt -check` enforces consistent code style - if someone pushes unformatted HCL, the pipeline fails immediately.

### The Plan Stage

`terraform plan` is the heart of the workflow. It compares the desired state (configuration) with the actual state (state file + Proxmox API) and produces a diff. This diff is saved as an artifact (`plan.tfplan`) so the apply stage uses the exact same plan.

The plan output shows up in the pipeline job log. In a merge request workflow, I review this output before approving the merge. This is the safety net - I see exactly what will change before it happens.

### The Apply Stage

The apply stage is `when: manual` and restricted to the `main` branch. This means:

1. Changes must be merged to main (through a merge request)
2. Someone must manually click "play" on the apply job

Automatic apply sounds efficient, but for infrastructure changes, human review before execution is a safety requirement. A bad plan that destroys the wrong VM should be caught by a human, not automatically applied.

---

## Cloud-Init Integration

Most of my VMs and LXCs use cloud-init for initial bootstrapping. Terraform uploads a cloud-init configuration that handles:

- Setting the hostname
- Configuring the network (static IP, gateway, DNS)
- Adding SSH public keys
- Installing base packages
- Setting the timezone

This means a freshly provisioned VM is immediately accessible via SSH without any manual configuration. From there, Ansible takes over for application-level setup.

### The Handoff to Ansible

The provisioning workflow has a clear boundary:

```
Terraform provisions the VM/LXC
    - CPU, memory, disk
    - Network configuration
    - Cloud-init (SSH keys, base packages)
    ↓
Ansible configures the guest
    - Application installation
    - Service configuration
    - Wazuh agent deployment
    - Postfix relay setup
```

Terraform owns the "infrastructure" layer. Ansible owns the "configuration" layer. Neither crosses into the other's domain. This separation keeps both codebases clean and avoids the temptation to use Terraform provisioners for configuration management.

---

## Documented Exceptions

Not everything in the homelab is managed by Terraform. Some resources can't or shouldn't be:

| Exception | Reason |
|---|---|
| USB passthrough VMs | Proxmox API tokens can't assign USB devices - requires root-level access |
| OPNsense firewall | Managed through its own UI/API, not as a Proxmox resource |
| Synology NAS | No Terraform provider available for Synology DSM |
| Manually created test VMs | Short-lived experiments not worth codifying |

These exceptions are documented in the repository README. The documentation includes the reason each resource is excluded, so future-me doesn't waste time trying to import something that can't be managed.

---

## VMID Allocation Scheme

Proxmox assigns VMIDs automatically, but I use a structured allocation scheme so VMIDs are meaningful:

| Range | Purpose |
|---|---|
| 100-199 | Core infrastructure |
| 1000-1099 | Infrastructure VMs and LXCs |
| 2000-2099 | Security stack |
| 3000-3099 | Client-facing services |
| 4000-4099 | Business services |
| 5000-5099 | Dev/misc |
| 6000-6099 | IaC stack |

In Terraform, the VMID is explicitly set in each resource definition. This prevents Proxmox from auto-assigning IDs and keeps the numbering consistent with the documented scheme.

---

## Working With the bpg/proxmox Provider - Practical Tips

### Pin the Provider Version

Provider updates can introduce breaking changes. I pin the version in `versions.tf`:

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.78.0"
    }
  }
}
```

The `~>` constraint allows patch updates but blocks minor version bumps, which is where breaking changes tend to land.

### Handling LXC Templates

LXC containers are provisioned from templates stored on the Proxmox node. Terraform can download templates automatically, but I prefer to pre-stage them. The template download is a one-time operation, and having it in Terraform means it runs on every `init` if the state doesn't track it correctly.

### Dealing With Timeouts

VM and LXC creation in Proxmox can take longer than Terraform's default timeouts, especially for operations that involve disk cloning or template extraction. I set generous timeouts in the resource configuration to avoid false failures:

```hcl
resource "proxmox_virtual_environment_container" "example" {
  # ... configuration ...

  timeouts {
    create = "5m"
    delete = "2m"
  }
}
```

### Import Workflow

When bringing existing manually-created resources under Terraform management:

1. Write the Terraform configuration for the resource
2. Run `terraform import` to associate the configuration with the existing resource
3. Run `terraform plan` to verify there's no diff (or minimal expected diff)
4. Adjust the configuration until the plan is clean
5. Commit and merge

This is the safe path. Never destroy and recreate a production resource just to get it into Terraform state.

---

## Lessons Learned

- **Start with LXCs** - they're simpler to manage in Terraform than VMs and faster to provision. Get comfortable with the provider on LXCs before tackling VMs with disks, cloud-init, and PCI passthrough
- **One resource file per logical group** - don't put all resources in a single `main.tf`. Group by function (security stack, services, IaC tools) for readability
- **Use modules for consistency** - local modules enforce standards across all resources. When you decide to change the default disk size or network configuration, you change it in one place
- **Manual apply gates are essential** - automated `terraform apply` is a disaster waiting to happen in an infrastructure context. Always review the plan first
- **Document your exceptions** - resources outside Terraform management should be documented with the reason. This prevents wasted effort and confusion
- **Test provider upgrades carefully** - run `terraform plan` after upgrading the provider version. Breaking changes in the provider can produce unexpected diffs
- **Never store secrets in tfvars** - API tokens, passwords, and SSH keys belong in CI/CD variables, not in committed files

---

## What's Next

- **Multi-site Terraform** - extending the codebase to manage the home cluster alongside the DC node, using workspaces or a separate state per site
- **Automated drift detection** - a scheduled CI pipeline that runs `terraform plan` nightly and alerts if drift is detected
- **Policy as code** - integrating tools like OPA (Open Policy Agent) or Checkov to validate Terraform configurations against security and compliance policies before apply

Terraform turned my Proxmox infrastructure from a collection of manually-created snowflakes into a reproducible, version-controlled codebase. Every change is traceable, every resource is documented, and disaster recovery is a `terraform apply` away.
