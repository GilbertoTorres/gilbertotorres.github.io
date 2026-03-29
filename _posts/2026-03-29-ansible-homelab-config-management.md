---
layout: post
title: "Ansible for Homelab Configuration Management"
date: 2026-03-29 19:20:00 +0200
categories: setup
tags: ansible configuration-management automation iac
---

Terraform provisions the infrastructure. Ansible configures what runs inside it. This post covers how I use Ansible for configuration management across all DC guests - from Wazuh agent deployment to base system hardening - and how it fits into the broader GitLab CI pipeline.

## The Division of Labor

The boundary between Terraform and Ansible is deliberate and strict:

| Layer | Tool | Manages |
|---|---|---|
| Infrastructure | Terraform | VMs, LXCs, disks, networks, cloud-init |
| Configuration | Ansible | Packages, services, files, users, agents |

Terraform creates a VM with the right CPU, memory, disk, and network settings. Cloud-init provides SSH keys and a basic network configuration so the host is reachable. Then Ansible takes over - it installs packages, deploys configuration files, starts services, and ensures ongoing compliance.

This separation means each tool does what it's best at. Terraform is declarative and idempotent for infrastructure resources. Ansible is procedural enough to handle the nuances of software configuration while still being idempotent when written correctly.

---

## Repository and Inventory

The Ansible codebase lives in `infra/dc-pve-ansible` in GitLab, alongside the Terraform repo.

### Inventory Structure

The inventory maps directly to the Proxmox environment. Hosts are organized into groups that correspond to their function:

```ini
[security_stack]
wazuh
thehive
misp

[client_services]
mailcow
hestiacp
dns-authoritative

[iac_tools]
gitlab
gitlab-runner
rackpeek

[infrastructure]
opnsense
tunnel-agent
```

Each host entry includes connection details (IP address, SSH user, Python interpreter path). The grouping enables targeted playbook runs - deploying the Wazuh agent to all hosts, or updating only the security stack, or rolling out a base configuration change everywhere.

### Group and Host Variables

Variables are layered using Ansible's standard hierarchy:

- `group_vars/all.yml` - settings that apply to every host (DNS servers, NTP servers, common packages, mail relay configuration)
- `group_vars/<group>.yml` - group-specific settings (security stack memory tuning, service-specific package lists)
- `host_vars/<host>.yml` - per-host overrides when needed

This hierarchy keeps the configuration DRY. When I change the DNS server address, I change it in one place and every host picks it up.

---

## Key Roles

Ansible roles are the building blocks. Each role encapsulates a specific configuration concern and can be applied independently or composed into playbooks.

### wazuh_agent - Security Agent Deployment

This is the most critical role in the repository. Every DC guest must run a Wazuh agent for security monitoring. The role handles the full lifecycle:

| Aspect | Configuration |
|---|---|
| Agent version | Pinned to a specific release |
| Enrollment | Auto-enrollment enabled |
| Manager communication | TCP-based agent communication |
| Monitored logs | syslog, auth.log, dpkg.log |
| FIM directories | /etc, /usr/bin, /usr/sbin, /bin, /sbin, /boot |
| FIM frequency | Every 12 hours (full scan), real-time for critical paths |
| Syscollector | Hardware, OS, network, packages, ports, processes - hourly |

The role templates the `ossec.conf` configuration file using Jinja2, which means the manager address, monitored log paths, and FIM settings are all driven by variables. If the Wazuh manager moves or the monitoring requirements change, I update the variables and re-run the playbook.

#### Deployment

```bash
# Dry run - verify what would change without applying
ansible-playbook playbooks/wazuh.yml --check

# Deploy agents to all hosts
ansible-playbook playbooks/wazuh.yml

# Deploy to a specific group only
ansible-playbook playbooks/wazuh.yml --limit security_stack
```

The `--check` flag is a habit I never skip. Ansible's check mode simulates the run and reports what would change without actually changing it. This is the Ansible equivalent of `terraform plan` - always review before applying.

### common - Base System Configuration

The common role establishes a consistent baseline across all hosts:

- **Package management** - installs a standard set of utilities (curl, wget, htop, vim, unzip, and others) and ensures unattended-upgrades is configured for security patches
- **SSH hardening** - disables password authentication, enforces key-based auth, restricts root login
- **Timezone and NTP** - sets the timezone and configures time synchronization
- **User management** - ensures the automation user exists with the correct SSH keys and sudo access
- **MOTD** - sets a login banner identifying the host and its Terraform-managed VMID

Every new host gets this role as part of the initial provisioning workflow. It's the foundation that makes the environment consistent and predictable.

### postfix_relay - Mail Relay Configuration

Every host needs the ability to send email - for cron job output, service alerts, and system notifications. The postfix_relay role configures Postfix as a relay that forwards all outbound mail through the internal mail server.

This means I don't need to configure SMTP credentials on every individual service. Any application that sends mail to `localhost:25` has it relayed automatically.

---

## The Playbook Structure

Playbooks compose roles into targeted workflows:

```
playbooks/
  site.yml           # Full-site convergence (all roles, all hosts)
  wazuh.yml          # Wazuh agent deployment only
  common.yml         # Base configuration only
  postfix.yml        # Mail relay configuration only
```

### site.yml - Full Convergence

The `site.yml` playbook applies all roles to all hosts in the correct order:

```yaml
---
- name: Base configuration
  hosts: all
  become: true
  roles:
    - common
    - postfix_relay
    - wazuh_agent

- name: Security stack specific
  hosts: security_stack
  become: true
  roles:
    - security_tuning
```

Running `site.yml` brings the entire environment to its desired state. This is useful after major changes or as a periodic compliance check. In practice, I run targeted playbooks (just `wazuh.yml` or just `common.yml`) for day-to-day operations.

---

## CI/CD Integration

Ansible playbooks run through GitLab CI, just like Terraform. The pipeline is simpler since Ansible doesn't have a "plan" concept as formal as Terraform's, but the workflow still includes validation and gated execution.

### Pipeline Structure

```yaml
stages:
  - lint
  - dry-run
  - apply

lint:
  stage: lint
  tags:
    - dc
    - shell
  script:
    - ansible-lint playbooks/site.yml
    - yamllint .

dry-run:
  stage: dry-run
  tags:
    - dc
    - shell
  script:
    - ansible-playbook playbooks/site.yml --check --diff
  only:
    - merge_requests

apply:
  stage: apply
  tags:
    - dc
    - shell
  script:
    - ansible-playbook playbooks/site.yml
  when: manual
  only:
    - main
```

### Linting

`ansible-lint` catches common mistakes - deprecated module usage, missing `become` directives, tasks without names, and other best practice violations. `yamllint` ensures YAML syntax is clean. These run on every push, catching issues before they reach a merge request.

### Dry Run

The dry-run stage uses `--check --diff` to simulate the playbook and show what would change. The `--diff` flag is especially useful - it shows file content changes inline, so I can verify template rendering without applying.

### Manual Apply

Like Terraform, the apply stage is manual. I review the dry-run output in the merge request, merge to main, and then manually trigger the apply. For configuration management, this gated approach prevents unintended changes from rolling out automatically.

---

## Writing Idempotent Roles

Ansible's power comes from idempotency - running a playbook twice should produce the same result. But idempotency isn't automatic. It requires careful role design.

### Patterns I Follow

**Use modules, not shell commands.** Ansible modules are designed to be idempotent. The `apt` module checks if a package is installed before trying to install it. The `template` module checks if the file content has changed before writing. Shell commands don't have this intelligence - they run every time.

```yaml
# Good - idempotent
- name: Install required packages
  ansible.builtin.apt:
    name: "{{ common_packages }}"
    state: present

# Bad - runs every time
- name: Install required packages
  ansible.builtin.shell: apt-get install -y curl wget htop
```

**Use handlers for service restarts.** When a configuration file changes, the service needs to restart. But it should only restart if the file actually changed, not on every playbook run:

```yaml
- name: Deploy Wazuh agent config
  ansible.builtin.template:
    src: ossec.conf.j2
    dest: /var/ossec/etc/ossec.conf
    owner: root
    group: wazuh
    mode: '0640'
  notify: restart wazuh-agent

handlers:
  - name: restart wazuh-agent
    ansible.builtin.service:
      name: wazuh-agent
      state: restarted
```

The handler only fires if the template task reports a change. On subsequent runs where the config hasn't changed, the service isn't restarted unnecessarily.

**Use FQCN (Fully Qualified Collection Names).** Always use the full module name like `ansible.builtin.apt` instead of the short form `apt`. This avoids ambiguity when multiple collections provide modules with the same name, and it makes the code self-documenting about which collection each module comes from.

---

## Ansible Collections

Beyond the built-in modules, I use community collections for Proxmox-specific and security-specific tasks:

| Collection | Purpose |
|---|---|
| `community.proxmox` | Proxmox guest management and API interaction |
| `ansible.posix` | POSIX system management (authorized_keys, sysctl) |
| `community.general` | General-purpose modules (timezone, hostname) |
| `community.crypto` | Certificate and key management |

Collections are pinned in `requirements.yml` and installed on the GitLab Runner. Version pinning is critical - collection updates can change module behavior and break existing playbooks.

```yaml
# requirements.yml
collections:
  - name: community.proxmox
    version: ">=1.5.0,<2.0.0"
  - name: ansible.posix
    version: ">=1.5.0,<2.0.0"
  - name: community.general
    version: ">=9.0.0,<10.0.0"
```

---

## The New Host Workflow

When Terraform provisions a new LXC or VM, here's the Ansible workflow that follows:

1. **Terraform creates the host** with cloud-init (SSH keys, network, base packages)
2. **Add the host to the Ansible inventory** in the correct group
3. **Run the common playbook** to establish the baseline: `ansible-playbook playbooks/common.yml --limit new-host`
4. **Run the Wazuh playbook** to deploy the security agent: `ansible-playbook playbooks/wazuh.yml --limit new-host`
5. **Run any role-specific playbooks** for the host's function
6. **Verify in Wazuh** that the agent has enrolled and is reporting

From bare LXC to monitored, hardened, mail-enabled host - the entire workflow is codified. No SSH-ing in and running commands manually. No "I'll document this later" (you won't).

---

## Secrets Management

Ansible playbooks often need secrets - API keys, passwords, certificates. I handle these through a combination of approaches:

- **GitLab CI/CD variables** for secrets used during pipeline execution (passed as environment variables to the runner)
- **Ansible Vault** for secrets that need to live in the repository (encrypted at rest, decrypted during playbook runs)
- **Variable files excluded from Git** for local development (`.gitignore`'d)

The Vault password is stored as a CI/CD variable in GitLab and passed to `ansible-playbook` via the `--vault-password-file` flag pointing to an environment variable.

---

## Operational Patterns

### Rolling Updates

When deploying a change across all hosts, I use Ansible's `serial` directive to limit the blast radius:

```yaml
- name: Deploy configuration update
  hosts: all
  become: true
  serial: 2
  roles:
    - common
```

This applies the change to 2 hosts at a time. If something fails on the first batch, the remaining hosts are untouched. For a homelab this might seem like overkill, but it's a good habit that transfers directly to production environments.

### Tagging

Tasks and roles are tagged so I can run subsets of a playbook without executing everything:

```bash
# Only run tasks tagged with 'ssh'
ansible-playbook playbooks/common.yml --tags ssh

# Run everything except package installation
ansible-playbook playbooks/common.yml --skip-tags packages
```

Tags make iterative development faster. When working on the SSH hardening configuration, I don't need to wait for package installation and NTP configuration to run first.

### Fact Gathering

Ansible gathers system facts (OS version, IP addresses, memory, disk) at the start of each playbook run. These facts are used in templates and conditionals:

```yaml
- name: Configure based on OS family
  ansible.builtin.template:
    src: "syslog.conf.{{ ansible_os_family }}.j2"
    dest: /etc/rsyslog.d/50-custom.conf
  when: ansible_os_family == "Debian"
```

This makes roles portable across different OS versions without hardcoding assumptions.

---

## Lessons Learned

- **Idempotency is a discipline, not a feature** - Ansible doesn't guarantee idempotency. You achieve it by using modules correctly, avoiding raw shell commands, and testing thoroughly
- **Check mode before apply, always** - `--check --diff` is the safety net. Skip it once and you'll regret it
- **FQCN everywhere** - using `ansible.builtin.apt` instead of `apt` avoids subtle bugs from collection name conflicts and makes the code more readable
- **Pin collection versions** - unpinned collections will eventually break your playbooks when an upstream update changes behavior
- **One role, one concern** - resist the urge to put everything in a single role. Separate roles are easier to test, reuse, and debug
- **The common role is the most important** - it's unglamorous, but a consistent base configuration across all hosts prevents an entire category of debugging headaches
- **Ansible for day-2, Terraform for day-0** - don't blur the boundary. Terraform provisions infrastructure, Ansible configures it. Terraform provisioners are an anti-pattern for configuration management
- **Handlers prevent unnecessary restarts** - always use `notify` and handlers instead of restarting services unconditionally

---

## What's Next

- **Molecule testing** - adding automated role testing with Molecule so every merge request verifies that roles converge correctly on a fresh container
- **Expanded role coverage** - more roles for application-specific configuration (database tuning, web server hardening, log rotation policies)
- **Dynamic inventory from Proxmox** - using the `community.proxmox` inventory plugin to auto-discover hosts instead of maintaining a static inventory file
- **AWX/Semaphore** - evaluating a web UI for Ansible to provide better visibility into playbook runs and scheduling

Ansible is the glue that turns a collection of freshly provisioned VMs and LXCs into a consistent, monitored, and secure environment. Combined with Terraform for provisioning and GitLab CI for execution, it completes the IaC stack that makes the homelab manageable at scale.
