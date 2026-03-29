---
layout: post
title: "HestiaCP for Web and DNS Hosting"
date: 2026-03-29 18:20:00 +0200
categories: setup
tags: hestiacp web-hosting dns nginx apache
---

Every homelab eventually needs to host something publicly - a website, a blog, or an API endpoint. Rather than spinning up individual Nginx configs for each project, I run HestiaCP as a lightweight hosting control panel at the DC site. It manages web hosting, DNS zones, SSL certificates, and user accounts from a single interface.

## Why HestiaCP?

I evaluated several options before settling on HestiaCP:

- **cPanel/Plesk** - commercial, expensive, overkill for my use case
- **CloudPanel** - modern but limited DNS management
- **Raw Nginx/Apache configs** - flexible but time-consuming to manage at scale
- **HestiaCP** - open source, lightweight, full DNS management, Apache + Nginx combo

HestiaCP hits the sweet spot. It's a fork of VestaCP with active development, proper security practices, and enough features to handle real hosting workloads without the bloat of enterprise panels.

---

## Architecture

HestiaCP runs on a dedicated LXC container at the DC site on the Clients network segment. It's one of the few services in my infrastructure that is directly exposed to the public internet - not through Pangolin, but via traditional OPNsense port forwards.

### Why Not Pangolin?

This is a deliberate choice. HestiaCP serves as an authoritative DNS server and web host for domains that need to resolve globally. DNS requires direct port 53 access, and web hosting benefits from a direct path without tunnel overhead. Pangolin is excellent for internal services that need public access, but HestiaCP's role as a DNS authority requires traditional exposure.

### Network Exposure

HestiaCP is reachable from the public internet on these ports:

| Port | Protocol | Purpose |
|---|---|---|
| 53 | DNS (TCP/UDP) | Authoritative DNS for hosted domains |
| 80 | HTTP | Web hosting (redirects to HTTPS) |
| 443 | HTTPS | Web hosting with SSL |
| 8083 | HTTPS | HestiaCP admin panel |
| Custom SSH port | SSH | Shell access (non-standard for reduced noise) |
| 21 | FTP | File transfer for web content |

Every one of these is a potential attack surface, which is why hardening is critical.

---

## What HestiaCP Manages

### Web Hosting

HestiaCP uses a **dual-stack web server** approach - Apache behind Nginx:

- **Nginx** handles the front-facing connection, SSL termination, static file serving, and caching
- **Apache** processes dynamic content via PHP-FPM
- Each domain gets its own **PHP-FPM pool**, providing process isolation between hosted sites

This architecture means a misbehaving PHP application on one domain can't starve resources from another - each pool has its own process limits and memory allocation.

SSL certificates are automated via Let's Encrypt. HestiaCP handles certificate issuance, renewal, and installation for every hosted domain. No manual certificate management required.

### DNS Hosting

This is where HestiaCP becomes critical infrastructure. It serves as the authoritative DNS server for all hosted domains, including the domains used by Mailcow for email delivery.

HestiaCP manages:

- **A/AAAA records** - pointing domains to hosting IPs
- **MX records** - directing email to Mailcow
- **SPF records** - declaring authorized mail senders
- **DKIM records** - public keys for email signature verification
- **DMARC records** - policy for handling authentication failures
- **CNAME records** - aliases and autodiscover endpoints
- **NS records** - nameserver delegation

The relationship between HestiaCP DNS and Mailcow is the most operationally sensitive integration in my infrastructure. Every mail domain depends on HestiaCP serving correct DNS records. An MX record pointing to the wrong IP means bounced mail. A missing DKIM record means failed signature verification and spam folder placement.

### FTP and SSH Access

HestiaCP provides per-user FTP and SSH accounts with chroot isolation by default. Each user can only access their own web root - no traversal to other users or system directories.

FTP is available for legacy workflows, but I primarily use SFTP over the non-standard SSH port for file transfers.

---

## Security Hardening

Being directly exposed to the internet means HestiaCP needs serious hardening. Here's what I've implemented:

### SSH Hardening

- **Key-only authentication** - password login is completely disabled
- **Non-standard port** - SSH runs on a custom port to reduce automated scanning noise
- **Root login disabled** - all access goes through a regular user account with sudo

### fail2ban

fail2ban monitors multiple services and bans IPs after failed authentication attempts:

| Service | Max Retries | Ban Duration |
|---|---|---|
| SSH | 3 attempts | Progressive (increases with repeat offenders) |
| HestiaCP admin | 5 attempts | 10 minutes initially |
| FTP | 5 attempts | 10 minutes initially |

fail2ban is configured to ban at the firewall level, not just the application level. Banned IPs are dropped before they even reach the service.

### Wazuh Agent

A Wazuh agent runs on the HestiaCP container, reporting to the SOC stack at the DC. This provides:

- **File Integrity Monitoring** - detects unauthorized changes to web roots, configuration files, and system binaries
- **Log analysis** - SSH attempts, web server errors, and DNS query anomalies
- **Syscollector** - inventory of installed packages and open ports
- **Real-time alerting** - high-severity events create cases in TheHive

### SSL/TLS

All web traffic and the admin panel are served over HTTPS with modern TLS configurations. HTTP requests are redirected to HTTPS. The admin panel on port 8083 uses its own certificate.

---

## DNS Coordination with Mailcow

I covered this from the Mailcow side in the previous post, but it's worth repeating from the HestiaCP perspective because this is the most common source of operational incidents.

The workflow for adding a new mail domain is:

1. **Create the DNS zone in HestiaCP** (or add records to an existing zone)
2. **Add MX record** pointing to the mail server
3. **Add SPF TXT record** authorizing the mail server IP
4. **Generate DKIM key in Mailcow** and copy the public key
5. **Add DKIM TXT record** in HestiaCP with the public key
6. **Add DMARC TXT record** with your policy (reject, quarantine, or none)
7. **Enable the domain in Mailcow**
8. **Test** - send emails and verify DKIM/SPF pass in received headers

Skipping steps or doing them out of order leads to deliverability issues that can take days to resolve. I've documented this workflow as a runbook in my wiki and follow it every time.

---

## Hosted Content

HestiaCP hosts several types of content:

| Type | Examples |
|---|---|
| Static sites | Personal and project pages |
| PHP applications | Legacy web apps |
| DNS zones | All public domain zones |
| Client projects | Isolated per-user hosting |

Each user account in HestiaCP represents an isolated hosting environment with its own:

- Web root directory
- PHP-FPM pool
- FTP/SSH credentials
- Disk and bandwidth quotas
- SSL certificates

---

## Troubleshooting Notes

### MariaDB RRD Stats After Reboot

After a reboot, the monitoring system that tracks MariaDB statistics can fail with TLS certificate errors. The root cause is a cached credentials file that gets corrupted during unclean shutdowns.

The fix is straightforward - remove the cached MySQL credentials file and let HestiaCP regenerate it on the next cron run. I've added this to the post-reboot checklist, and I'm considering a systemd oneshot unit that runs automatically on boot to prevent this from happening silently.

### DNS Propagation Timing

When adding new DNS records, remember that propagation takes time. I've been burned by adding a domain to Mailcow immediately after creating DNS records, only to have mail fail because external resolvers hadn't picked up the new MX records yet. Now I wait for propagation to confirm (using tools like dig against multiple resolvers) before enabling anything on the Mailcow side.

---

## Backup Strategy

The HestiaCP container is backed up nightly by Proxmox Backup Server as part of the standard DC backup schedule. This captures:

- All web content and databases
- DNS zone files
- SSL certificates and private keys
- User accounts and configuration
- HestiaCP's own configuration

For DNS specifically, I also maintain a copy of all zone files in my GitLab repository. This means even if the HestiaCP container needs to be rebuilt from scratch, I can restore DNS zones from version-controlled source rather than relying solely on PBS backups.

---

## Performance Considerations

The Nginx-in-front-of-Apache architecture provides good performance out of the box:

- **Static content** is served directly by Nginx without touching Apache
- **Gzip compression** is enabled at the Nginx level
- **PHP opcache** is configured per pool
- **Connection pooling** between Nginx and Apache keeps resource usage efficient

For the scale I'm running - a handful of sites with moderate traffic - the default configuration is more than adequate. I've tuned PHP-FPM pool sizes down from defaults to save memory, since my sites don't need dozens of worker processes each.

---

## Integration Points

HestiaCP touches several other systems in the infrastructure:

| System | Integration |
|---|---|
| **Mailcow** | DNS records for all mail domains |
| **Wazuh** | Security monitoring via agent |
| **OPNsense** | Firewall rules for public access |
| **Proxmox Backup Server** | Nightly container backups |
| **Ansible** | Wazuh agent deployment and configuration |
| **Let's Encrypt** | Automated SSL certificate management |

---

## Lessons Learned

- **Direct exposure requires serious hardening** - when a service faces the public internet without a reverse proxy, every layer of defense matters
- **DNS is the most operationally sensitive service** - a DNS misconfiguration affects email, web, and everything that depends on name resolution
- **Non-standard SSH ports actually help** - they don't provide security, but they dramatically reduce log noise from automated scanners
- **Chroot isolation per user is essential** - even in a homelab, treat each hosting account as untrusted
- **Document the DNS-to-Mailcow workflow** - the coordination between HestiaCP and Mailcow is the most common source of incidents
- **Automate post-reboot checks** - cached state files that survive reboots can cause subtle failures that go unnoticed

HestiaCP fills a specific niche in my infrastructure - the traditional web and DNS hosting that needs direct public exposure. Everything else goes through Pangolin.
