---
layout: post
title: "Passbolt - Self-Hosted Password Manager"
date: 2026-03-29 18:50:00 +0200
categories: setup
tags: passbolt password-manager security self-hosted
---

Every infrastructure credential in my homelab lives in Passbolt. SMTP passwords, API tokens, encryption keys, tunnel secrets, cloud provider credentials - all of it. Passbolt is a self-hosted, open-source password manager designed for teams, and it's the single most critical service in my entire setup. Lose access to Passbolt, and I lose access to everything.

## Why Passbolt?

I evaluated several self-hosted password managers before choosing Passbolt:

- **Vaultwarden** - excellent personal password manager, but limited team and organizational features
- **HashiCorp Vault** - powerful secrets engine, but overkill for a homelab and operationally complex
- **KeePass/KeeWeb** - file-based, no native multi-user support, difficult to integrate with automation
- **Passbolt** - purpose-built for team credential sharing, browser extension, CLI, API access, and self-hosted

The deciding factors were:

1. **Team-oriented design** - even though it's currently just me, the infrastructure is documented and designed for potential handoff. Passbolt's sharing model supports this
2. **API access** - credentials can be retrieved programmatically, which is essential for automation workflows
3. **Browser extension** - daily use is seamless with the browser extension for web-based admin panels
4. **Open source** - the community edition provides all the features I need

---

## Architecture

Passbolt runs as an LXC container on the home cluster's primary node.

| Detail | Value |
|---|---|
| Type | LXC Container |
| Host | Primary home node |
| Network | Home management VLAN |
| Access | Via Pangolin + Newt tunnel |

### Why Home, Not DC?

This was a deliberate placement decision. Passbolt is on the home cluster because:

1. **Physical access** - in a catastrophic failure scenario, I have physical access to the home cluster. If both Pangolin and the site-to-site VPN go down, I can still reach Passbolt locally
2. **Bootstrap dependency** - when rebuilding DC infrastructure from scratch, I need credentials before the DC is operational. Having Passbolt at home means I can access secrets during DC recovery
3. **Network proximity** - day-to-day, I'm physically at home. Accessing Passbolt through a local network path is faster and doesn't depend on WAN connectivity

Public access to Passbolt goes through Pangolin and Newt, following the zero-trust model. When I'm away from home, I access it through the Pangolin-proxied URL just like any other service.

---

## What Passbolt Stores

Passbolt is the single source of truth for every credential in the infrastructure. Here's what lives in it:

### Infrastructure Credentials

| Category | Examples |
|---|---|
| Service admin passwords | Root and admin passwords for all services |
| SMTP credentials | The sysadmin SMTP password used by all Postfix relay hosts |
| Proxmox API tokens | Tokens used by Terraform and Ansible for infrastructure automation |
| PBS encryption keys | Datastore encryption keys for Proxmox Backup Server |

### Tunnel and Proxy Credentials

| Category | Examples |
|---|---|
| Pangolin/Newt secrets | Agent IDs and secrets for all four Newt tunnel agents |
| WireGuard keys | Pre-shared keys for site-to-site tunnels |

### Cloud Provider Credentials

| Category | Examples |
|---|---|
| Wasabi S3 | Access key and secret key for offsite backup storage |
| Historical credentials | Backblaze B2 credentials retained for reference (service decommissioned) |

### Application Credentials

| Category | Examples |
|---|---|
| Database passwords | MariaDB, PostgreSQL credentials for hosted applications |
| OIDC client secrets | Client IDs and secrets for PocketID integrations |
| Mail domain credentials | Per-domain mailbox passwords |

The organization in Passbolt follows the infrastructure structure - credentials are grouped by service and environment, making them easy to find during incident response when time is critical.

---

## Browser Extension Workflow

For daily use, Passbolt's browser extension is how I interact with credentials most often. The workflow is:

1. Navigate to a service's admin panel
2. The browser extension detects the URL and offers matching credentials
3. Click to autofill username and password
4. Service is authenticated

For services behind Pangolin, the extension matches on the public hostname (e.g., `service.biggie.be`). For internal services accessed directly, it matches on the internal hostname.

The extension requires a passphrase to unlock on each browser session start. This passphrase is not stored in Passbolt itself (for obvious reasons) - it exists only in my memory and in a physical backup stored securely offsite.

---

## API Access for Automation

One of Passbolt's strengths is its API. This enables automated credential retrieval in infrastructure workflows:

### Use Cases

- **Ansible playbooks** can retrieve SMTP credentials when configuring Postfix relay hosts
- **Terraform** can pull API tokens when provisioning Proxmox resources
- **Backup scripts** can retrieve encryption keys without hardcoding them
- **CI/CD pipelines** can access credentials during deployment

The API uses GPG-based authentication, which means API access requires both the API key and the associated GPG private key. This is more secure than simple token-based authentication - even if the API key is compromised, it's useless without the GPG key.

---

## Criticality and Risk

Passbolt is the most critical service in my infrastructure. Its failure modes and their impacts:

| Failure Mode | Impact | Mitigation |
|---|---|---|
| Service unavailable | Cannot retrieve credentials for any service | PBS backups, physical access to home cluster |
| Data corruption | Credential loss across all services | Nightly PBS backups with verification |
| Credential compromise | Attacker gains access to all infrastructure | Pangolin-only access, Wazuh monitoring, GPG encryption |
| Encryption key loss | Cannot decrypt credential database | Physical backup of master key stored offsite |

### Why This Is Different from Other Services

If Mailcow goes down, email queues and I fix it. If HestiaCP goes down, websites are offline and I restore from backup. If Passbolt goes down, I potentially cannot authenticate to anything - including the tools I need to fix Passbolt. This circular dependency is why:

1. Passbolt is on the home cluster where I have physical console access
2. PBS backup of the Passbolt container is verified quarterly
3. The GPG master key and Passbolt passphrase have physical (offline) backups
4. Critical recovery credentials (Proxmox root, PBS encryption keys) are also stored in a separate offline backup

---

## Backup Strategy

Given its criticality, Passbolt's backup strategy is more thorough than most services:

### Automated Backups

- **Nightly PBS backup** - the entire LXC container is backed up as part of the standard schedule
- **Replicated to DC** - PBS backups are replicated to the DC Proxmox Backup Server via the WireGuard backup tunnel
- **Offsite copy** - backups are synced to Wasabi S3 for geographic redundancy

### Recovery Testing

Quarterly, I restore the Passbolt container to a temporary VMID and verify:

1. The container boots cleanly
2. The Passbolt web interface loads
3. I can authenticate with my credentials
4. Stored passwords are decryptable and readable
5. The browser extension connects to the restored instance

This is the only service where I perform full functional recovery testing, not just boot verification. The consequences of a failed Passbolt recovery are too severe to leave to assumption.

### Offline Backups

In addition to the automated pipeline:

- The GPG master key is backed up to physical media stored offsite
- The Passbolt passphrase is documented in a physical format stored separately from the GPG key
- Critical bootstrap credentials (enough to rebuild from scratch) are in a sealed envelope at a separate physical location

This might sound paranoid, but when every credential depends on one service, defense in depth is the only rational approach.

---

## Security Hardening

### Access Control

- **Pangolin-only access** - no direct port forwards to Passbolt. All access goes through the zero-trust tunnel
- **No public exposure** - unlike HestiaCP or Mailcow, Passbolt is never directly accessible from the internet
- **Browser extension authentication** - GPG passphrase required for each session

### Monitoring

- **Wazuh agent** - file integrity monitoring, log analysis, and syscollector running on the LXC container
- **Login notifications** - authentication events are logged and monitored
- **Postfix alerts** - system-level notifications from the container relay through Mailcow

The Postfix relay on the Passbolt container is configured to send from `passbolt@biggie.be` to the shared notifications inbox. This covers service-level notifications like password share events and user account changes.

### Encryption Model

Passbolt uses GPG (GNU Privacy Guard) for its encryption model. Each user has a GPG key pair:

- **Private key** - stored encrypted on the client side, decrypted with the user's passphrase
- **Public key** - stored on the server, used to encrypt credentials for that user

When a password is stored, it's encrypted with the recipient's public key. When it's retrieved, the client decrypts it with the private key. The server never sees the plaintext credential. This is end-to-end encryption - even if the Passbolt server is compromised, the attacker gets encrypted blobs, not passwords.

---

## Credential Rotation Workflow

When a credential needs to be rotated (e.g., an SMTP password or API token), the workflow is:

1. Generate a new credential in the target service
2. Update the entry in Passbolt with the new value
3. If the credential is used by automated systems (Ansible, Terraform), run the relevant playbook to push the update
4. Verify all consumers of the credential are working with the new value
5. Document the rotation in the wiki with the date

For SMTP credentials specifically, this affects every Postfix relay host in the infrastructure. A single `ansible-playbook` run updates the relay password on all LXC containers and VMs simultaneously. Without Passbolt centralizing the credential and Ansible distributing it, this would be a manual change on dozens of hosts.

---

## Integration Points

| System | Integration |
|---|---|
| **Ansible** | Credential retrieval for automated deployments |
| **Terraform** | API token storage and retrieval |
| **Pangolin/Newt** | Tunnel credential storage |
| **All Postfix relays** | SMTP credential source of truth |
| **PBS** | Encryption key storage |
| **Wasabi S3** | Cloud credential storage |
| **PocketID** | OIDC client secret storage |
| **Browser** | Extension for daily web-based credential access |

---

## Operational Notes

### Decommissioned Credentials

When a service or provider is decommissioned, I don't delete the credentials from Passbolt. Instead, I move them to an archive folder with a note about when and why the service was decommissioned. This has proven useful more than once - when investigating old configurations or audit trails, having the historical credentials available saves time.

For example, Backblaze B2 credentials are still in Passbolt even though the service was decommissioned in early March 2026 and replaced with Wasabi S3. They're archived, not deleted.

### Emergency Access Procedure

If Passbolt is inaccessible and I need credentials urgently:

1. Physical access to the home cluster via console
2. Start the Passbolt LXC if it's down
3. Access the web interface from a machine on the management VLAN
4. If the LXC is corrupted, restore from the latest PBS backup
5. If PBS is also unavailable, use the offline physical backup to rebuild

This procedure is documented in the wiki and tested annually.

---

## Lessons Learned

- **Your password manager is your most critical service** - treat it accordingly with extra backup layers and recovery testing
- **Physical offline backups are not optional** - when everything depends on one encrypted database, you need a recovery path that doesn't depend on any digital infrastructure
- **Quarterly recovery testing builds confidence** - knowing your backup actually works is worth the hour it takes to verify
- **Archive, don't delete** - decommissioned credentials have historical value
- **GPG-based encryption is worth the complexity** - end-to-end encryption means a server compromise doesn't expose credentials
- **Place critical services where you have physical access** - in a multi-site setup, your password manager belongs at the site you can physically reach
- **Centralize, then automate** - once credentials are in Passbolt, Ansible can distribute them consistently across all hosts

Passbolt is the keystone of my infrastructure. Every other service depends on it, directly or indirectly. Treating it with the seriousness it deserves - from backup strategy to physical offline recovery materials - is not overkill. It's the minimum responsible approach.
