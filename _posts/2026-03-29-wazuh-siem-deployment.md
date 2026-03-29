---
layout: post
title: "Deploying Wazuh SIEM on Proxmox"
date: 2026-03-29 17:40:00 +0200
categories: setup
tags: wazuh siem security monitoring proxmox
---

In the [SOC stack overview]({% post_url 2026-03-29-security-monitoring-soc-stack %}), I introduced Wazuh as the SIEM centerpiece. This post goes deeper - how I deployed Wazuh on Proxmox, configured agent-based and agentless log collection, wrote custom decoders, and integrated it into a multi-site homelab.

## Why Wazuh?

I evaluated several open-source SIEM solutions before landing on Wazuh. The deciding factors were:

- **All-in-one architecture** - manager, indexer, and dashboard in a single deployment
- **Native agent support** - lightweight agents for Linux hosts with built-in FIM and syscollector
- **Agentless syslog ingestion** - for devices that can't run an agent (firewalls, NAS boxes, network gear)
- **Active community** - decoders and rules for virtually every log source
- **REST API** - enables automation and integration with TheHive and MISP

Commercial SIEMs like Splunk or Elastic SIEM are powerful but prohibitively expensive for a homelab. Wazuh gives me 90% of the capability at zero licensing cost.

---

## Architecture

Wazuh runs as an LXC container on the DC Proxmox node, sitting on the dedicated management network alongside the rest of the security stack.

### Components

| Component | Purpose |
|---|---|
| Wazuh Manager | Receives agent data, processes rules, generates alerts |
| Wazuh Indexer | Stores and indexes alert data (OpenSearch-based) |
| Wazuh Dashboard | Web UI for alert visualization and management |

### Network Ports

| Port | Protocol | Purpose |
|---|---|---|
| 443 | HTTPS | Dashboard and web UI |
| 1514 | TCP/UDP | Agent communication |
| 1515 | TCP | Agent enrollment |
| 5140 | UDP | Agentless syslog reception |
| 55000 | HTTPS | Wazuh REST API (internal only) |

The dashboard runs on port 443 over HTTPS, accessible through the management network. The API on port 55000 is used by TheHive for alert forwarding and by Ansible for agent management automation.

---

## Agent Deployment with Ansible

Every VM and LXC in the infrastructure runs a Wazuh agent. Manual agent installation does not scale - and it definitely does not stay consistent. An Ansible role in my GitLab CI pipeline handles deployment across all DC guests.

### The Wazuh Agent Role

The `wazuh_agent` role in my `dc-pve-ansible` repository handles:

- Installing a pinned agent version for consistency across all hosts
- Configuring the manager connection (TCP on port 1514)
- Enabling automatic enrollment (port 1515)
- Deploying a standardized `ossec.conf` configuration

```bash
# Dry run to verify changes
ansible-playbook playbooks/wazuh.yml --check

# Deploy agents to all DC hosts
ansible-playbook playbooks/wazuh.yml
```

When I provision a new LXC through Terraform, the next Ansible run automatically picks it up and deploys the agent. No manual steps, no forgotten hosts.

### What Agents Monitor

Every agent collects the same baseline:

**Log sources:**

- `/var/log/syslog` - system events
- `/var/log/auth.log` - authentication events (SSH, sudo, PAM)
- `/var/log/dpkg.log` - package installation and removal

**File Integrity Monitoring (FIM):**

FIM watches critical system directories for unauthorized changes:

| Directory | What It Catches |
|---|---|
| `/etc` | Config file tampering, unauthorized user creation |
| `/usr/bin`, `/usr/sbin` | Binary replacement, rootkit installation |
| `/bin`, `/sbin` | Core utility modification |
| `/boot` | Bootloader or kernel tampering |

- Scan frequency: every 12 hours (full scan)
- Real-time monitoring: enabled for immediate detection between scans
- Alert on: file creation, modification, deletion, permission changes

FIM is one of the most underrated security controls. Log analysis can miss a sophisticated attacker, but FIM will catch them the moment they modify a system binary or drop a config file.

**Syscollector:**

Syscollector runs hourly and inventories:

- Hardware and OS details
- Network interfaces and open ports
- Installed packages and versions
- Running processes

This creates a continuously updated asset inventory. When a vulnerability is disclosed, I can immediately query which hosts have the affected package installed.

---

## Agentless Syslog Sources

Not everything can run an agent. Network appliances, NAS devices, and IoT gateways need a different approach - syslog forwarding to the Wazuh manager on UDP port 5140.

### Configured Sources

| Source | Type | Log Format | Notes |
|---|---|---|---|
| OPNsense | Firewall/router | BSD syslog | Firewall rules must have "Log" enabled |
| Synology NAS (Home) | Network storage | BSD (RFC 3164) | Arrives via WireGuard management tunnel |
| Synology NAS (DC) | Network storage | BSD (RFC 3164) | Direct on management network |
| UniFi Gateway | Network gateway | CEF (Common Event Format) | Arrives via WireGuard management tunnel |

### OPNsense Integration

OPNsense is one of the most valuable syslog sources. It forwards firewall logs using the built-in BSD syslog format, which Wazuh decodes natively with its pfSense decoder set.

Key configuration points:

- Syslog level set to Notice for useful detail without overwhelming volume
- Remote logging configured under System - Log Files - Settings
- Firewall rules must individually have the "Log" checkbox enabled to generate filterlog events
- The built-in decoder handles both pass and block events

Combined with ET Pro and Q-Feeds threat detection rules running on OPNsense, this gives Wazuh visibility into malicious traffic before it even reaches internal hosts.

### Synology NAS Monitoring

Both Synology NAS units forward logs covering system events, user activity, file transfers, connections, and backup operations. I use the community Synology DSM decoder and ruleset, which provides structured parsing of Synology-specific log formats.

A particularly useful rule is login success monitoring - every DSM login generates an alert mapped to MITRE ATT&CK technique T1078 (Valid Accounts). In a homelab where NAS access should be infrequent and predictable, unexpected logins stand out immediately.

### UniFi Gateway - Custom Decoder

The UniFi Cloud Gateway Max was the most challenging syslog source. Unlike OPNsense (which uses standard BSD syslog), UniFi devices output logs in CEF (Common Event Format) - a structured format that Wazuh does not decode out of the box.

I wrote a custom decoder chain to handle this:

**Decoder structure:**

1. **Base decoder** - matches the CEF header (`CEF:0|Ubiquiti|`)
2. **Field extractor** - parses device type, version, event ID, name, and severity
3. **Category decoder** - extracts the UniFi event category (auth, admin, system)

**Custom rules built on top of the decoder:**

| Alert Level | Description | Trigger |
|---|---|---|
| 3 | UniFi event (baseline) | Any CEF event from a Ubiquiti device |
| 3 | Camera detection | Smart detect zone or motion events |
| 5 | Auth/Admin event | Authentication or administrative actions |
| 3 | System event | System or firmware update events |
| 8 | High severity event | CEF severity field 7 through 9 |

The high-severity rule (level 8) is the important one - it catches critical events like firmware tampering, unauthorized access attempts, or device compromise indicators. Level 8 is high enough to trigger downstream integrations with TheHive.

---

## Cross-Site Syslog Routing

Devices at the home site cannot reach the Wazuh manager directly - they sit behind a different firewall on a different network. Syslog traffic from home devices arrives through the WireGuard management tunnel.

This means Wazuh sees home devices with their tunnel endpoint address rather than their actual LAN IP. This is important for correlation - you need to know which source IP maps to which device, or your dashboards become confusing.

### The WireGuard Routing Gotcha

There is a subtle but critical networking issue when running Wazuh behind an OPNsense gateway with WireGuard tunnels. If you accidentally add the management network subnet to a WireGuard peer's AllowedIPs, OPNsense creates a host route that sends all Wazuh-bound traffic into the tunnel instead of delivering it locally. This breaks all agent communication and syslog reception.

The fix is simple once you know the cause - remove the conflicting route and correct the WireGuard peer configuration. But diagnosing it the first time cost me a frustrating evening.

**Lesson:** document your WireGuard AllowedIPs meticulously, and always verify that local network routes take precedence over tunnel routes.

---

## Wazuh Dashboard and Alert Triage

The Wazuh Dashboard (accessible on port 443) provides the primary interface for day-to-day security monitoring.

### Key Dashboard Views I Use Daily

- **Security Events** - real-time feed of all processed alerts, filterable by severity, source, and rule group
- **Integrity Monitoring** - FIM alerts showing which files changed, on which hosts, and when
- **Vulnerabilities** - CVE scanning results correlated with syscollector package data
- **MITRE ATT&CK** - alerts mapped to the ATT&CK framework for structured analysis

### Alert Severity and Escalation

Wazuh alert levels range from 0 to 15. My escalation thresholds:

| Level Range | Action |
|---|---|
| 0 - 3 | Informational - logged, no action |
| 4 - 7 | Review during daily triage |
| 8 - 11 | Investigate promptly - potential TheHive case |
| 12 - 15 | Immediate response - auto-escalate to TheHive |

Alerts at level 8 and above are forwarded to TheHive for case creation. This keeps the noise manageable while ensuring nothing critical slips through.

---

## Integration Points

Wazuh does not operate in isolation. It feeds into and receives from the broader SOC stack:

```
Agents (all VMs/LXCs)
    → Wazuh Manager (agent data, TCP 1514)

Agentless devices
    → Wazuh Manager (syslog, UDP 5140)

Wazuh Manager
    → Wazuh Indexer (stores/indexes alerts)
    → TheHive (high-severity alerts → cases)
    → MISP (IOC correlation)

MISP
    → Wazuh (threat feeds for rule enrichment)
```

The integration with TheHive and MISP transforms Wazuh from a log aggregator into a detection and response platform. Details on those integrations are covered in their respective posts.

---

## Operational Notes

### Resource Consumption

Wazuh is surprisingly resource-efficient for what it does. The indexer is the main consumer - it needs disk space for alert storage and memory for search performance.

| Resource | Allocation | Notes |
|---|---|---|
| RAM | 4 GB | Adequate for current agent count |
| Disk | 25 GB | Monitor usage - indexer grows over time |
| CPU | 2 cores | Rarely a bottleneck |

The disk allocation needs watching. Alert volume grows with the number of agents and syslog sources, and the indexer does not automatically purge old data. I run index lifecycle management to rotate and delete indices older than 90 days.

### Maintenance Tasks

- **Index rotation** - automated, but verify monthly that old indices are actually being deleted
- **Agent version updates** - coordinated through Ansible, never updated ad-hoc
- **Decoder/rule updates** - tested in a staging environment before deploying custom rules
- **Dashboard backups** - custom dashboards and saved searches are backed up alongside the LXC

### Common Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Agent not connecting | Firewall rule missing for port 1514 | Add rule on management network |
| Syslog not arriving | UDP 5140 not open or wrong source IP | Check `ss -ulnp` and firewall rules |
| Dashboard slow | Indexer running out of memory | Increase RAM or reduce retention |
| FIM alerts flooding | Noisy directory in monitored paths | Add exclusion to the FIM configuration |

---

## Lessons Learned

- **Pin your agent version** - letting agents auto-update across dozens of hosts is a recipe for inconsistency and surprise breakages
- **Custom decoders are worth the effort** - the UniFi CEF decoder took an afternoon to write, but it unlocked visibility into an entire device category
- **FIM is your safety net** - even if an attacker evades log-based detection, file integrity monitoring will catch the moment they touch the filesystem
- **Test syslog paths end-to-end** - use `wazuh-logtest` to verify that raw log lines parse correctly before declaring a source "configured"
- **Document your WireGuard routes** - cross-site syslog routing through VPN tunnels adds complexity that will bite you during troubleshooting

Wazuh is the foundation that makes everything else in the SOC stack possible. With reliable agent deployment and comprehensive log collection in place, the next step is turning those alerts into actionable investigations - which is where [TheHive]({% post_url 2026-03-29-thehive-incident-response-setup %}) comes in.
