---
layout: post
title: "MISP Threat Intelligence Platform Setup"
date: 2026-03-29 18:00:00 +0200
categories: setup
tags: misp threat-intelligence security soc ioc
---

A SIEM tells you what happened. Incident response tools help you investigate. But without threat intelligence, you are always reacting - never anticipating. MISP (Malware Information Sharing Platform) is the threat intelligence layer in my homelab SOC stack. It aggregates Indicators of Compromise from community feeds and makes them available to Wazuh for detection and TheHive for investigation enrichment.

## What MISP Does

MISP is an open-source threat intelligence platform designed for collecting, storing, distributing, and sharing cybersecurity indicators and threats. In practical terms, it answers the question: "Is this IP, hash, or domain already known to be malicious?"

In my homelab, MISP serves three primary functions:

1. **Feed aggregation** - pulls IOC feeds from community sources (malware hashes, C2 domains, scanning IPs, phishing URLs)
2. **Wazuh enrichment** - Wazuh correlates real-time events against MISP IOCs for detection
3. **TheHive enrichment** - when investigating a case, TheHive queries MISP to determine if observables match known threats

Without MISP, every alert starts from zero. With MISP, I immediately know whether an IP hitting my firewall is a known scanner, whether a file hash matches a malware family, or whether a domain is associated with a phishing campaign.

---

## Architecture

MISP runs as an LXC container on the DC Proxmox node, on the same management network as Wazuh and TheHive. The three tools form a tightly integrated security pipeline.

### Integration Stack

```
OPNsense (ET Pro / Q-Feeds)
        |
    Wazuh SIEM
        |
    TheHive <---> MISP
```

Data flows in multiple directions:

- **Wazuh to MISP** - Wazuh queries MISP feeds for IOC correlation during alert processing
- **MISP to Wazuh** - MISP provides threat feeds that Wazuh uses as CDB lists for rule matching
- **MISP to TheHive** - Cortex analyzers query MISP when analyzing case observables
- **TheHive to MISP** - Investigated observables can be exported back to MISP for tracking and sharing

### Resource Allocation

| Resource | Allocation | Notes |
|---|---|---|
| RAM | 16 GB | Elasticsearch and feed processing are memory-intensive |
| Disk | 100 GB | IOC storage grows with feed subscriptions |
| CPU | 4 cores | Feed synchronization is CPU-intensive during pulls |

Like TheHive, MISP relies on Elasticsearch for its backend storage. The 16 GB RAM allocation is not optional - feed synchronization and IOC correlation under load will exhaust anything less.

---

## Deployment on Proxmox

### LXC Setup

MISP runs in its own dedicated LXC rather than sharing a container with TheHive. This separation provides:

- **Independent resource scaling** - MISP and TheHive have different resource profiles and can be tuned independently
- **Blast radius isolation** - a problem with MISP does not take down incident response capabilities
- **Clean upgrade paths** - MISP and TheHive release on different schedules

### Installation

MISP installation is notoriously complex compared to most security tools. The project provides an automated installer script that handles the dependency chain: Apache, PHP, Python, MySQL/MariaDB, Redis, and the MISP application itself.

Key post-installation steps:

1. **Change default credentials** - MISP ships with well-known default admin credentials that must be changed immediately
2. **Configure the base URL** - set the external URL for API access and feed synchronization
3. **Enable background workers** - MISP uses worker processes for feed synchronization, email notifications, and enrichment jobs
4. **Configure Redis** - required for the job queue and caching layer

---

## Feed Configuration

Feeds are the core value of MISP. Without feeds, it is just an empty database. With well-chosen feeds, it becomes a continuously updated repository of known threats.

### Feed Types

MISP supports several feed types:

| Feed Type | Description | Example |
|---|---|---|
| MISP Feed | Native MISP format - events with attributes | CIRCL OSINT Feed |
| Freetext | Plain text IOC lists (one per line) | abuse.ch URLhaus |
| CSV | Structured IOC lists with field mapping | Custom blocklists |

### Community Feeds I Use

| Feed | Content | Update Frequency |
|---|---|---|
| CIRCL OSINT Feed | Community-shared threat events | Real-time |
| abuse.ch URLhaus | Malicious URLs distributing malware | Every 5 minutes |
| abuse.ch Feodo Tracker | Botnet C2 server IPs | Hourly |
| abuse.ch ThreatFox | IOCs from various malware families | Hourly |
| Botvrij.eu | Dutch CERT threat intelligence | Daily |
| Blocklist.de | Brute-force and scanner IPs | Daily |
| PhishTank | Verified phishing URLs | Hourly |

### Feed Management

Feed synchronization is resource-intensive. Each pull downloads new IOCs, deduplicates against existing data, creates MISP events, and indexes the results. I schedule large feeds during off-peak hours to avoid impacting other services on the management network.

```
Feed sync schedule:
  - High-frequency feeds (URLhaus, Feodo): every 6 hours
  - Medium-frequency feeds (CIRCL, ThreatFox): every 12 hours
  - Low-frequency feeds (Blocklist.de, Botvrij): daily
```

A common mistake is subscribing to every available feed. More feeds means more IOCs, but it also means more false positives, more storage consumption, and longer correlation times. I start conservative and add feeds only when they prove their value by matching real events in my environment.

---

## Wazuh Integration

The integration between MISP and Wazuh is what transforms both tools from standalone utilities into a detection pipeline.

### How It Works

1. MISP aggregates IOCs from configured feeds
2. IOC lists are exported in a format Wazuh can consume (CDB lists)
3. Wazuh loads these lists and checks incoming events against them
4. When an event matches a MISP IOC, Wazuh generates an enriched alert with threat context

### Practical Example

An OPNsense firewall log arrives at Wazuh showing an outbound connection to an external IP. Wazuh checks this IP against the MISP-sourced CDB list. The IP matches the Feodo Tracker feed as a known Emotet C2 server. Wazuh generates a high-severity alert (level 12+) that includes:

- The original firewall log
- The MISP feed that matched (Feodo Tracker)
- The threat category (botnet C2)
- The source host that initiated the connection

This alert is immediately forwarded to TheHive for investigation. Without MISP, the same firewall log would have been a routine outbound connection with no context.

### CDB List Synchronization

The Wazuh-MISP integration uses CDB (Constant Database) lists - flat files of IOCs that Wazuh loads into memory for fast lookups. These lists need periodic synchronization:

| List Type | Content | Refresh |
|---|---|---|
| IP reputation | Known malicious IPs from MISP | Every 6 hours |
| Hash watchlist | Malware file hashes | Every 12 hours |
| Domain blocklist | Known C2 and phishing domains | Every 6 hours |

The synchronization process exports IOCs from MISP, formats them as CDB lists, and places them in the Wazuh manager's rules directory. A configuration reload picks up the updated lists without restarting the manager.

---

## TheHive Integration

MISP integrates with TheHive through Cortex - TheHive's analysis engine. When an analyst (me) submits an observable for analysis, Cortex queries MISP as one of its configured analyzers.

### What the MISP Analyzer Returns

When Cortex queries MISP with an observable (IP, hash, domain, URL), the response includes:

- **Match status** - whether the observable exists in any MISP event
- **Feed sources** - which feeds contributed the match
- **Threat level** - MISP's assessment of the threat (low, medium, high)
- **Related attributes** - other IOCs from the same MISP event (useful for pivoting)
- **Tags and taxonomies** - structured categorization (malware family, threat actor, TLP marking)
- **Event context** - the full MISP event description if available

### Bidirectional Sharing

The integration is not one-way. When I investigate a case in TheHive and identify new IOCs that are not already in MISP, I can export them back:

1. Mark observables in the case as IOCs
2. Export to MISP - creates a new MISP event with the case observables
3. The new event becomes part of MISP's dataset and is available for future correlation

This feedback loop is especially valuable for homelab-specific threats. If I see repeated scanning from an IP that is not in any community feed, adding it to MISP ensures it will be flagged if it appears again - in my environment or in shared community data.

---

## Taxonomy and Tagging

MISP's taxonomy system provides structured classification for threat intelligence. Proper tagging makes IOCs searchable, filterable, and meaningful at scale.

### Taxonomies I Use

| Taxonomy | Purpose | Example Tags |
|---|---|---|
| TLP | Traffic Light Protocol for sharing classification | tlp:white, tlp:green, tlp:amber |
| MITRE ATT&CK | Technique and tactic mapping | mitre-attack:T1071 (Application Layer Protocol) |
| Admiralty Scale | Source reliability and information credibility | admiralty-scale:source-reliability="b" |
| Malware Type | Classification of malware families | malware_classification:malware-category="ransomware" |

### Why Tagging Matters

Without taxonomies, a MISP instance full of IOCs is a haystack. With proper tagging:

- Wazuh can filter correlation by threat type (only check for C2 domains, not all IOCs)
- TheHive cases inherit MISP tags, providing immediate context
- Feed quality can be assessed by correlating tags with true/false positive rates
- Sharing with the community is meaningful because recipients can filter by relevance

---

## Event Management

MISP organizes IOCs into events. Each event represents a specific threat - a malware campaign, a scanning operation, a phishing wave. Events contain attributes (the actual IOCs) and objects (structured collections of related attributes).

### Event Lifecycle

| Stage | Action | Automation |
|---|---|---|
| Creation | Feed sync or manual import | Automated via feed pulls |
| Enrichment | Add context, tags, correlations | Semi-automated via MISP modules |
| Publication | Make available for sharing and correlation | Manual review before publishing |
| Expiration | Age out stale IOCs | Automated via event expiration policies |

### Handling Stale Intelligence

Threat intelligence has a shelf life. An IP that was a C2 server six months ago may be a legitimate host today. MISP supports expiration policies that automatically deprecate old events:

- Events older than 90 days are marked as expired by default
- Expired events are excluded from Wazuh CDB list exports
- Manual override for long-lived IOCs (APT infrastructure, known scanning ranges)

Getting expiration right is important. Too aggressive and you lose valuable historical context. Too lenient and your CDB lists fill with stale IOCs that generate false positives.

---

## Operational Notes

### Performance Tuning

MISP performance depends on several factors:

| Factor | Impact | Mitigation |
|---|---|---|
| Number of feeds | More feeds = more IOCs = slower correlation | Curate feeds ruthlessly |
| Event count | Large event databases slow searches | Archive old events, use expiration |
| Worker count | Insufficient workers = feed sync backlog | Scale workers to feed count |
| Elasticsearch heap | Too small = slow queries and OOM risk | Allocate at least 4 GB heap |

### Monitoring MISP Health

Key indicators that MISP is healthy and operational:

- **Worker status** - all background workers running and processing jobs
- **Feed sync timestamps** - feeds updating on schedule, no stale feeds
- **Event count growth** - steady growth indicates feeds are active
- **Correlation count** - correlations between events indicate IOC overlap (expected)
- **API response time** - slow API degrades Wazuh and TheHive integration

### Backup and Recovery

MISP data is backed up through the standard LXC backup on Proxmox Backup Server. Critical components:

- **MySQL/MariaDB database** - all events, attributes, users, and configuration
- **Redis data** - job queue state (can be rebuilt, but losing it delays feed sync)
- **Uploaded files** - malware samples and attachments
- **Configuration files** - server settings, feed configurations, organization details

### Email Notifications

Like TheHive, MISP uses Postfix for outbound email notifications. This covers:

- Alert notifications when new high-priority events are created
- Feed synchronization failure alerts
- User account notifications

The Postfix relay is configured identically to the TheHive LXC - both forward through the same external relay.

---

## Security Considerations

Running a threat intelligence platform introduces its own security considerations:

- **MISP contains malware IOCs** - some feeds include actual malware hashes and C2 infrastructure details. The platform itself must be well-secured
- **API keys are sensitive** - MISP API keys grant full read/write access to threat data. Store them securely and rotate periodically
- **Network isolation** - MISP sits on the management network with firewall rules restricting access to only Wazuh, TheHive, and administrative hosts
- **Feed source verification** - not all threat feeds are trustworthy. Stick to established sources (CIRCL, abuse.ch, national CERTs)

---

## The Complete SOC Pipeline

With MISP in place, the full SOC pipeline is operational:

```
1. OPNsense detects suspicious traffic (ET Pro / Q-Feeds)
2. Wazuh agents report host-level events (auth, FIM, syscollector)
3. Wazuh correlates events against MISP IOC feeds
4. High-severity alerts forwarded to TheHive
5. Analyst triages alert, runs Cortex analyzers (including MISP lookup)
6. Investigation documented as a case
7. New IOCs exported back to MISP for future detection
```

Each tool handles one part of the pipeline:

| Tool | Role in Pipeline |
|---|---|
| OPNsense | Network-level detection (IDS/IPS rules) |
| Wazuh | Host and network log aggregation, correlation, alerting |
| MISP | Threat intelligence - known-bad context for events |
| TheHive | Investigation management - structured incident response |
| Cortex | Observable analysis - automated enrichment |

---

## Lessons Learned

- **Start with fewer feeds** - subscribe to 3 or 4 high-quality feeds first. Add more only when you understand the noise profile and can evaluate their value against real events
- **Curate aggressively** - a MISP instance with 50 feeds and no curation is worse than one with 5 well-maintained feeds. Quality over quantity, always
- **Expiration policies are essential** - stale IOCs generate false positives that erode trust in the platform. Set expiration and review it quarterly
- **The bidirectional loop matters** - exporting observables from TheHive back to MISP closes the intelligence cycle. Do not treat MISP as read-only
- **Monitor your workers** - MISP background workers are silent when they fail. Check worker status regularly or set up alerts for stalled jobs
- **Allocate real resources** - like TheHive, MISP needs substantial RAM for Elasticsearch. 16 GB is the practical minimum for a feed-heavy deployment
- **Tag everything** - untagged IOCs are almost useless for filtering and correlation. Enforce taxonomy usage from day one

MISP completes the SOC stack by adding the intelligence layer that transforms raw alerts into contextual investigations. Combined with [Wazuh]({% post_url 2026-03-29-wazuh-siem-deployment %}) for detection and [TheHive]({% post_url 2026-03-29-thehive-incident-response-setup %}) for incident response, it forms a pipeline that mirrors what you would find in a professional security operations center - running entirely on homelab infrastructure.
