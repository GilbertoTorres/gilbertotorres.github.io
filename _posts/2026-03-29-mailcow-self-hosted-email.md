---
layout: post
title: "Self-Hosting Email with Mailcow on Proxmox"
date: 2026-03-29 18:10:00 +0200
categories: setup
tags: mailcow email self-hosted proxmox docker
---

Self-hosting email is one of those things people tell you not to do. Deliverability is hard, reputation management is a grind, and one misconfigured SPF record can land you in spam folders globally. I did it anyway - and after running Mailcow for over a year, I'm glad I made that call.

## Why Self-Host Email?

The short answer is control. I manage SPF, DKIM, and DMARC records myself. I control retention, filtering, and privacy. For a setup that includes client-facing services, having email on infrastructure I own eliminates vendor lock-in and gives me full audit visibility.

The longer answer is that email underpins everything in my homelab. Every LXC and VM sends notifications through Postfix relays that ultimately land in Mailcow. Backup completions, security alerts, UPS power events, certificate renewals - they all flow through email. Outsourcing that to a third-party provider would mean trusting critical infrastructure alerts to someone else's uptime.

---

## Architecture

Mailcow runs as a Docker Compose stack inside an LXC container on the DC Proxmox node. This might sound unusual - Docker inside LXC - but it works remarkably well and has some real advantages.

### Docker-in-LXC

Running the full Mailcow Docker stack inside an LXC container rather than directly on the hypervisor achieves two things:

1. **Containment** - the Docker footprint stays isolated. No Docker networks leaking into the host, no risk of a container escape reaching the hypervisor
2. **Backup integration** - LXC containers are backed up natively by Proxmox Backup Server. The entire Mailcow stack, including its database and mail storage, gets snapshotted and replicated without needing Docker-aware backup tooling

The Docker Compose stack runs its own internal overlay network for inter-container communication. OPNsense has an explicit allow rule for this Docker subnet on the Clients interface so that containers can reach external services when needed.

### Dedicated WAN Interface

This is the architectural decision I'm most glad I made. Mailcow has its own dedicated WAN interface on OPNsense with a completely separate public IP. Email traffic never shares a path with web hosting, Pangolin tunnels, or any other service.

Why does this matter? Email reputation is tied to IP address. If a web application on a shared IP gets flagged for spam or abuse, your mail deliverability suffers too. By giving Mailcow its own IP, its reputation is entirely self-contained.

The OPNsense configuration for this interface includes:

| Rule Set | Ports | Purpose |
|---|---|---|
| MAIL_PORTS | 25, 465, 587, 993, 4190 | Core mail protocols |
| WEB_PORTS | 443 | Admin UI and autodiscover |

Port 80 has a rule configured but deliberately disabled - all web access is HTTPS-only.

---

## Mail Protocols and Ports

Mailcow exposes a full set of mail protocols, each serving a specific purpose:

| Port | Protocol | Purpose |
|---|---|---|
| 25 | SMTP | Inbound mail delivery (MX records point here) |
| 465 | SMTPS | Mail submission with implicit SSL |
| 587 | SMTP | Mail submission with STARTTLS |
| 993 | IMAPS | IMAP over SSL for mail retrieval |
| 4190 | Sieve | ManageSieve for server-side mail filtering |

The distinction between port 25 and ports 465/587 is important. Port 25 handles server-to-server mail delivery - when someone sends me an email, their mail server connects on port 25. Ports 465 and 587 are for client submission - when I send an email from Thunderbird or a mobile client, it authenticates and submits on one of these ports.

---

## Hosted Domains

Mailcow handles email for multiple domains, each with a different purpose:

| Domain | Role | Notes |
|---|---|---|
| Primary domain | Main mailboxes | Multiple mailboxes, thousands of messages stored |
| Business domains | Alias forwarding | Alias domains forwarding to primary mailboxes |
| Client domain | Client mailbox | Dedicated mailbox for client communication |
| Personal domain | Personal use | Separate personal identity |

The alias domain setup is particularly useful. Business domains don't need their own mailboxes - they forward to the primary domain's mailboxes, keeping everything in one place while maintaining separate sender identities.

### Sender Ownership Enforcement

Mailcow enforces sender address ownership strictly. If you want an alias to send as a particular domain, that domain must be configured in Mailcow first - even if it's just an alias domain. This tripped me up initially when setting up cross-domain aliases, but it's a sensible security measure that prevents spoofing.

---

## DNS Coordination with HestiaCP

This is where email self-hosting gets operationally complex. DNS records for all mail domains are managed in HestiaCP, not in Mailcow. This means any domain change requires a coordinated workflow:

1. **Create DNS records in HestiaCP first** - MX, SPF, DKIM, and DMARC records must exist before Mailcow will properly handle the domain
2. **Generate DKIM keys in Mailcow** - Mailcow generates the DKIM key pair when you add a domain
3. **Copy the DKIM public key to HestiaCP** - the TXT record with the DKIM public key goes into the DNS zone
4. **Test deliverability** - send test emails and verify DKIM/SPF pass in the headers

The DNS-first rule is non-negotiable. I've seen what happens when you enable a domain in Mailcow before the DNS is ready - bounced messages, failed DKIM signatures, and SPF hard-fails that can take days to recover from in terms of reputation.

### Required DNS Records

For each mail domain, the following records must exist:

| Record Type | Purpose |
|---|---|
| MX | Points to the Mailcow server |
| SPF (TXT) | Declares which IPs are authorized to send for the domain |
| DKIM (TXT) | Public key for email signing verification |
| DMARC (TXT) | Policy for handling SPF/DKIM failures |
| Autodiscover (CNAME/SRV) | Automatic client configuration |

Getting these right from the start saves enormous headaches later.

---

## Sieve Filtering

Mailcow includes ManageSieve support on port 4190, enabling server-side mail filtering. This is one of those features that seems minor but becomes essential quickly.

Server-side filtering means rules execute regardless of which client I'm using. Whether I check mail from Thunderbird on my desktop, a mobile client, or SOGo webmail, the same rules apply:

- Infrastructure alerts route to a dedicated folder
- Backup completion notifications get auto-archived
- Security alerts (from Wazuh) stay in the inbox with high priority
- Known mailing lists go to categorized folders

The alternative - client-side rules - means every device needs its own rule set, and if I read mail from a new device, everything is unfiltered. Sieve eliminates that problem entirely.

---

## SOGo Webmail

Mailcow ships with SOGo as its webmail interface. It handles email, contacts, and calendars through a browser. I primarily use Thunderbird as my desktop client, but SOGo is invaluable for quick access when I'm away from my workstation or need to check mail from a device that doesn't have a configured client.

SOGo is accessible through the same HTTPS endpoint as the Mailcow admin UI, which means it benefits from the dedicated WAN interface and its own firewall rules.

---

## The Notification Pipeline

Every service in my homelab sends notifications through Mailcow. The pipeline is simple and standardized:

```
Service (Wazuh, PBS, peaNUT, etc.)
    -> Postfix (local relay on each LXC/VM)
    -> Mailcow (SMTP relay on port 587)
    -> notifications inbox
```

Each LXC and VM runs a local Postfix instance configured as a relay host. The relay configuration points to Mailcow using authenticated SMTP. This is deployed consistently via Ansible - every new container gets the same Postfix relay configuration automatically.

The SMTP credentials used by all relay hosts are stored in Passbolt. If a credential rotation is needed, Ansible pushes the update to all hosts simultaneously.

---

## DC Hairpin NAT

One operational gotcha worth mentioning: internal services at the DC site that need to reach Mailcow's public IP require NAT reflection (hairpin NAT) in OPNsense, or split-horizon DNS.

Without this, a container at the DC trying to send email to Mailcow's public IP would hit OPNsense's WAN interface from the inside and get dropped. The fix is either configuring NAT reflection in OPNsense or setting up an internal DNS record that resolves the mail hostname to Mailcow's private IP directly.

I went with the split-horizon DNS approach using Technitium - internal DNS resolves the mail hostname to the LXC's private address, while external DNS points to the public IP. This avoids the performance overhead of hairpin NAT and keeps the traffic path clean.

---

## Backup and Recovery

Mailcow's LXC container is backed up by Proxmox Backup Server as part of the standard nightly backup schedule. The backup captures the entire container, including:

- All Docker volumes (mail storage, database, configuration)
- Postfix queue
- DKIM keys
- SOGo data

Recovery testing is part of my quarterly DR validation. I restore the Mailcow container to a temporary VMID, verify it boots, and confirm the Docker Compose stack starts cleanly. Full mail flow testing during DR drills is on the roadmap but not yet automated.

---

## Lessons Learned

- **Docker-in-LXC is production-viable** - Mailcow's Docker Compose stack runs reliably inside an LXC container, and you get PBS backup integration without additional tooling
- **Dedicated IP for email is essential** - separating mail traffic from everything else protects sender reputation and simplifies firewall management
- **DNS-first workflow is non-negotiable** - always configure MX, SPF, DKIM, and DMARC records before enabling a domain in Mailcow
- **Server-side filtering with Sieve is worth the setup time** - consistent filtering across all clients reduces noise significantly
- **Hairpin NAT is a real problem in virtualized environments** - split-horizon DNS is the cleaner solution
- **Standardize the notification pipeline** - every service sending alerts through the same relay path makes troubleshooting straightforward

Self-hosting email is not for everyone, but for a homelab that runs real services with real clients, the control and visibility it provides are worth the operational investment.
