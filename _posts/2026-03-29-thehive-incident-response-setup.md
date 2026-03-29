---
layout: post
title: "TheHive for Incident Response in a Homelab"
date: 2026-03-29 17:50:00 +0200
categories: setup
tags: thehive incident-response security soc cortex
---

Running a SIEM generates alerts. Lots of alerts. The question is - what do you do with them? TheHive is my answer. It turns Wazuh alerts into structured investigations with cases, observables, tasks, and timelines. This post covers deploying TheHive in a homelab, integrating it with Wazuh and MISP, and building an incident response workflow that mirrors professional SOC operations.

## Why Incident Response in a Homelab?

The honest answer: because I want to be good at it. Incident response is a skill that atrophies without practice. Running TheHive on real alerts - even homelab alerts - builds muscle memory for the workflows that matter in a production SOC.

Beyond skill-building, there are practical reasons:

- **Structured documentation** - when something goes wrong, a case in TheHive is better than a mental note
- **Pattern recognition** - tracking incidents over time reveals trends (repeated brute-force sources, recurring misconfigurations)
- **Integration testing** - TheHive ties into Wazuh and MISP, creating a full detection-to-response pipeline
- **Portfolio value** - demonstrating a working SOC stack is more compelling than listing tools on a resume

---

## Architecture

TheHive runs as an LXC container on the DC Proxmox node, on the same management network as Wazuh and MISP. This co-location is intentional - security tools need low-latency, reliable communication with each other.

### Component Overview

| Component | Purpose |
|---|---|
| TheHive | Case management and incident response platform |
| Cortex | Analysis engine - runs analyzers and responders on observables |
| Elasticsearch | Backend data store for cases, alerts, and observables |

### Dependencies

| Dependency | Role |
|---|---|
| Wazuh | Alert source - high-severity alerts create TheHive cases |
| MISP | Threat intelligence - enriches observables with IOC context |
| Postfix | Email notifications for case updates and escalations |

TheHive depends on Elasticsearch heavily. This is the primary resource driver - Elasticsearch wants memory, and it wants a lot of it. Skimping on RAM allocation is the single most common mistake when deploying TheHive.

---

## Deployment on Proxmox

### LXC Configuration

TheHive runs in a dedicated LXC container with resources sized for Elasticsearch:

| Resource | Allocation | Notes |
|---|---|---|
| RAM | 16 GB | Elasticsearch is the bottleneck |
| Disk | 100 GB | Case data, attachments, and Elasticsearch indices |
| CPU | 4 cores | Adequate for single-analyst workload |

I initially tried running TheHive with 4 GB of RAM. Elasticsearch would start, appear healthy for a few hours, then OOM-kill itself under any meaningful query load. 16 GB resolved this completely.

### Installation Approach

TheHive offers multiple installation methods. I chose the package-based installation on Debian rather than Docker, for several reasons:

- **LXC compatibility** - Docker-in-LXC adds complexity (nested container runtimes, cgroup delegation)
- **Direct filesystem access** - easier to troubleshoot Elasticsearch data directories
- **Systemd integration** - standard service management with journald logging
- **Ansible-friendly** - package installs are straightforward to automate

The installation follows TheHive's official documentation: add the repository, install the package, configure the application.conf, and start the services.

---

## Key Configuration

### Application Configuration

TheHive's main configuration lives in `/etc/thehive/application.conf`. The critical sections are:

**Database backend:**

TheHive uses Elasticsearch as its primary data store. The connection is configured to point at the local Elasticsearch instance running on the same LXC.

**Authentication:**

For a single-analyst homelab setup, local authentication is sufficient. TheHive supports LDAP and OAuth2 if you need multi-user SSO, but I have not needed that complexity yet.

**Cortex integration:**

Cortex is TheHive's analysis engine. When you submit an observable (IP address, file hash, domain name), Cortex runs it through configured analyzers and returns enrichment data. My Cortex instance runs alongside TheHive in the same LXC.

Key analyzers I have enabled:

| Analyzer | Purpose | Source |
|---|---|---|
| MISP | Query MISP for matching IOCs | Local MISP instance |
| VirusTotal | File hash and URL reputation | VirusTotal API (free tier) |
| AbuseIPDB | IP reputation lookup | AbuseIPDB API |
| URLhaus | Malicious URL detection | abuse.ch feed |
| Shodan | Host reconnaissance | Shodan API |

Cortex analyzers turn a raw observable into context. An IP address is just a number - but knowing it is listed on AbuseIPDB, has open ports visible on Shodan, and matches a MISP threat feed makes it actionable intelligence.

### Email Notifications

TheHive sends email notifications for case updates, task assignments, and alert escalations. Postfix is configured as a local relay, forwarding through an external SMTP relay for delivery.

This was configured as part of the broader mail infrastructure - every security LXC in the stack uses the same Postfix relay configuration for outbound notifications.

---

## Wazuh Integration

The primary alert source for TheHive is Wazuh. High-severity Wazuh alerts automatically create TheHive alerts, which can then be promoted to full cases for investigation.

### Alert Flow

```
Wazuh detects event
    → Rule fires (level 8+)
    → Alert forwarded to TheHive via API
    → TheHive creates alert with context
    → Analyst reviews and promotes to case (or dismisses)
```

### What Gets Forwarded

Not every Wazuh alert becomes a TheHive alert. The integration is filtered by severity:

| Wazuh Level | TheHive Action |
|---|---|
| 0 - 7 | Not forwarded - handled in Wazuh dashboard |
| 8 - 11 | Alert created - review during daily triage |
| 12 - 15 | Alert created with high severity - investigate promptly |

This filtering is critical. Wazuh generates hundreds of low-level informational alerts daily. Forwarding all of them to TheHive would make the platform unusable. The threshold at level 8 strikes a balance - it captures genuinely interesting events without drowning in noise.

### Alert Content

Each forwarded alert includes:

- **Title** - the Wazuh rule description
- **Severity** - mapped from Wazuh alert level
- **Source** - the agent or syslog source that generated the event
- **Raw log** - the original log line for manual analysis
- **MITRE ATT&CK mapping** - technique ID if the Wazuh rule includes it
- **Observables** - extracted IPs, usernames, file paths (auto-extracted where possible)

The MITRE ATT&CK mapping is particularly valuable. When I see an alert tagged with T1110 (Brute Force) or T1078 (Valid Accounts), I immediately know the context without reading the full alert description.

---

## MISP Integration

TheHive and MISP have a bidirectional integration that enriches investigations with threat intelligence.

### TheHive to MISP

When investigating a case, I can export observables (IPs, hashes, domains) from TheHive to MISP. This serves two purposes:

1. **Community contribution** - sharing observed IOCs back to threat feeds
2. **Cross-case correlation** - if the same IOC appears in a future case, MISP will flag it

### MISP to TheHive (via Cortex)

The MISP Cortex analyzer queries the local MISP instance when analyzing observables. If an IP or hash matches a known threat feed, the analyzer returns:

- Which feed(s) matched
- Threat level and confidence score
- Related IOCs from the same event
- Tags and taxonomies

This closes the loop - Wazuh detects something suspicious, TheHive captures it as a case, and MISP tells me whether it is a known threat or something novel.

---

## Incident Response Workflow

Having tools is not enough - you need a repeatable process. Here is the workflow I follow for every alert that reaches TheHive:

### 1. Triage

Review the alert in TheHive. Key questions:

- Is this a true positive or a false positive?
- What is the severity? Does it need immediate attention?
- Which host(s) are affected?

If it is a false positive, dismiss the alert and optionally tune the Wazuh rule to reduce future noise.

### 2. Case Creation

If the alert warrants investigation, promote it to a case. Add:

- **Description** - what happened, initial observations
- **Severity** - low, medium, high, or critical
- **TLP** - Traffic Light Protocol marking (usually TLP:CLEAR for homelab)
- **Tags** - categorization (brute-force, malware, misconfiguration, etc.)

### 3. Observable Analysis

Extract and analyze observables:

- Run Cortex analyzers on IPs, hashes, and domains
- Check MISP for matches against threat feeds
- Correlate with Wazuh dashboard for related events on the same host
- Check Wazuh FIM data for file changes around the same timestamp

### 4. Containment

If the threat is active:

- Block the source IP in OPNsense (manual or via Wazuh active response)
- Isolate the affected host if compromise is suspected
- Rotate any potentially exposed credentials

### 5. Documentation and Closure

Document findings in the case:

- Root cause analysis
- Actions taken
- Recommendations for prevention
- Close the case with a resolution status (true positive resolved, false positive, etc.)

### Real-World Example: SSH Brute Force

A common workflow in my homelab:

1. Wazuh detects multiple failed SSH authentication attempts from a single IP (rule level 10)
2. Alert created in TheHive with the source IP as an observable
3. Cortex analyzers check AbuseIPDB (listed as known scanner), Shodan (open ports confirm scanning infrastructure), MISP (IP matches a threat feed)
4. Confirm true positive - source is a known scanner
5. Verify no successful authentication occurred (check Wazuh auth logs)
6. Add IP to OPNsense blocklist
7. Close case - true positive, no compromise, source blocked

Total time: about 10 minutes. But the structured workflow means nothing gets skipped, and the case history builds a record of security events over time.

---

## Operational Notes

### Resource Management

TheHive's Elasticsearch backend is the resource bottleneck. Monitor these metrics:

| Metric | Warning Threshold | Action |
|---|---|---|
| Heap usage | > 75% | Increase JVM heap size |
| Disk usage | > 80% | Archive old cases or expand storage |
| Query latency | > 2 seconds | Check index health, consider optimization |

### Backup Strategy

TheHive data is backed up as part of the LXC backup schedule on Proxmox Backup Server. The critical data is:

- Elasticsearch indices (case data, alerts, observables)
- TheHive configuration files
- Cortex analyzer configurations
- Uploaded attachments and evidence files

### Maintenance Routine

| Task | Frequency | Purpose |
|---|---|---|
| Alert triage | Daily | Review and process new alerts |
| Case review | Weekly | Follow up on open cases |
| Analyzer updates | Monthly | Update Cortex analyzer versions |
| Index health check | Monthly | Verify Elasticsearch index status |
| Full backup verification | Monthly | Restore test from PBS backup |

---

## Lessons Learned

- **Give Elasticsearch enough RAM** - 16 GB is not excessive. TheHive with an underpowered Elasticsearch is worse than no TheHive at all, because it creates the illusion of monitoring without actually being usable
- **Filter your alert sources** - forwarding every Wazuh alert to TheHive defeats the purpose. Set a severity threshold and tune it over time
- **Build a workflow before you need one** - incident response under pressure is not the time to figure out your process. Practice on low-severity alerts until the workflow is automatic
- **Cortex analyzers are force multipliers** - a raw IP address tells you nothing. An IP with AbuseIPDB, Shodan, and MISP context tells you everything
- **Close your cases** - an open case backlog is demoralizing and reduces the value of the platform. Be disciplined about resolution
- **Postfix configuration matters** - email notifications for critical alerts are useless if your mail relay is not configured. Set it up early and test it

TheHive transforms security monitoring from passive observation into active investigation. Combined with [Wazuh]({% post_url 2026-03-29-wazuh-siem-deployment %}) for detection and [MISP]({% post_url 2026-03-29-misp-threat-intelligence-setup %}) for threat intelligence, it completes the detection-to-response pipeline.
