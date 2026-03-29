---
layout: post
title: "Security & Monitoring — Building a Homelab SOC Stack"
date: 2026-03-29 13:00:00 +0200
categories: homelab
tags: security wazuh thehive misp siem soc monitoring
---

Running a multi-site homelab with production services means security can't be an afterthought. This post covers the SOC (Security Operations Center) stack I've built — from log collection to incident response and threat intelligence.

## The Stack at a Glance

Three tools form the core of the security pipeline, all running as LXC containers on the DC Proxmox node:

| Tool | Purpose |
|---|---|
| **Wazuh** | SIEM — log collection, intrusion detection, file integrity monitoring |
| **TheHive** | Incident response — case management and investigation |
| **MISP** | Threat intelligence — IOC feeds enriching alerts |

These aren't just installed and forgotten. They're integrated into a pipeline that flows from raw events to actionable cases.

---

## Wazuh SIEM — The Eyes and Ears

Wazuh is the centerpiece. It collects logs from every host, monitors file integrity, detects anomalies, and feeds alerts downstream.

### What Wazuh Monitors

**Agent-based collection** — every VM and LXC across both sites runs a Wazuh agent:

- System logs (syslog, auth, package management)
- **File Integrity Monitoring (FIM)** — watches critical directories (`/etc`, `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, `/boot`) for unauthorized changes
- **Syscollector** — inventories hardware, OS, network, packages, ports, and processes hourly
- Real-time FIM alerts for immediate detection of file tampering

**Agentless sources:**

- **OPNsense** — firewall logs via syslog, enriched with ET Pro and Q-Feeds threat detection rules
- Network device logs forwarded via syslog

### Agent Deployment via Ansible

Agents aren't installed manually. An Ansible role in my GitLab CI pipeline handles deployment:

```
ansible-playbook playbooks/wazuh.yml
```

This deploys the Wazuh agent to all DC guests with:
- Automatic enrollment to the Wazuh manager
- Pinned agent version for consistency
- Standardized log monitoring and FIM configuration

New containers get agents automatically when provisioned through the IaC pipeline.

---

## TheHive — Incident Response

TheHive is where alerts become investigations. When Wazuh fires a high-severity alert, it creates a case in TheHive with all the context attached.

### What TheHive Provides

- **Case management** — structured investigations with observables, tasks, and timelines
- **Alert triage** — Wazuh alerts flow in and can be promoted to cases or dismissed
- **Collaboration** — if I ever expand the team, TheHive supports multi-analyst workflows
- **MISP integration** — observables in cases are enriched with threat intelligence automatically

For a homelab, TheHive might seem like overkill. But practicing incident response workflows on real alerts builds skills that transfer directly to professional SOC work.

---

## MISP — Threat Intelligence

MISP (Malware Information Sharing Platform) is the threat intel layer. It aggregates IOC (Indicators of Compromise) feeds and makes them available to both Wazuh and TheHive.

### How MISP Fits In

- **Feed aggregation** — MISP pulls from community threat feeds (malware hashes, suspicious IPs, known C2 domains)
- **Wazuh enrichment** — Wazuh checks events against MISP IOCs for correlation
- **TheHive enrichment** — when investigating a case, TheHive queries MISP to see if observables match known threats
- **Bidirectional sync** — TheHive and MISP share observables both ways

---

## The Integration Pipeline

Here's how everything flows together:

```
OPNsense (ET Pro / Q-Feeds)
    ↓ syslog
Wazuh SIEM
    ↓ alerts
TheHive ←→ MISP
```

1. **OPNsense** detects suspicious traffic using ET Pro and Q-Feeds rulesets and forwards logs to Wazuh
2. **Wazuh** correlates these with agent data (FIM changes, auth failures, syscollector anomalies) and generates alerts
3. High-severity alerts create cases in **TheHive**
4. **MISP** enriches cases with threat context — is this IP a known C2 server? Has this hash been seen in malware campaigns?
5. Analysts (me) investigate, document findings, and close cases

### Alert Examples

Some real-world alerts this pipeline catches:

- SSH brute-force attempts against public-facing services
- Unauthorized file modifications on critical system paths
- DNS queries to known malicious domains (via MISP feeds)
- Firewall rule matches against threat intelligence IOCs
- New packages installed outside of maintenance windows

---

## Resource Allocation

The SOC stack is resource-hungry — especially TheHive and MISP, which are backed by Elasticsearch:

| Service | RAM | Disk | Notes |
|---|---|---|---|
| Wazuh | 4 GB | 25 GB | Indexer is the bottleneck — monitor disk usage |
| TheHive | ~16 GB | 100 GB | Elasticsearch needs memory |
| MISP | ~16 GB | 100 GB | Feed storage + Elasticsearch |

Total: ~36 GB of RAM dedicated to security monitoring. On a 128 GB DC node, that's a significant allocation — but security visibility is worth it.

---

## Future: Honeypot Infrastructure

The next evolution is deploying honeypots on the DC node:

- **Low-interaction honeypots** mimicking common services (SSH, HTTP, SMB)
- Deployed in an isolated network segment
- All honeypot activity forwarded to Wazuh as high-fidelity alerts
- Any interaction with a honeypot is inherently suspicious — zero false-positive rate

This will add a proactive detection layer alongside the reactive monitoring Wazuh already provides.

---

## Lessons Learned

- **Start with Wazuh alone** — get agent deployment and log collection solid before adding TheHive/MISP complexity
- **Ansible for agent deployment is non-negotiable** — manual agent installs don't scale and inevitably drift
- **FIM is underrated** — file integrity monitoring catches things that log analysis misses
- **Allocate real resources** — Elasticsearch-backed tools need memory. Don't try to run TheHive on 2 GB of RAM
- **Practice incident response** — even on false positives, working through the triage workflow builds muscle memory

The SOC stack transforms a homelab from "services I run" to "services I can defend." Next up: how all of this gets backed up.
