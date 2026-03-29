---
layout: post
title: "Zero-Trust Access with Pangolin and Newt"
date: 2026-03-29 18:30:00 +0200
categories: setup
tags: pangolin newt zero-trust reverse-proxy tunnel
---

The single most impactful infrastructure decision I made was adopting Pangolin and Newt for public service access. Before Pangolin, every new service meant a new port forward, a new NAT rule, and another entry in the firewall. Now, zero services require inbound firewall rules at either site. Everything goes through outbound-only tunnels.

## The Problem Pangolin Solves

Traditional reverse proxy setups (Nginx, Traefik, Caddy) require the proxy to be on the same network as the services it fronts, or they require inbound port forwards through the firewall. In a multi-site homelab with services spread across a home cluster and a datacenter, this means:

- Port forwards at both sites for every service
- NAT rules that grow with each new deployment
- Firewall management that becomes increasingly complex
- Each open port is a potential attack vector

Pangolin flips this model. The proxy server runs in the cloud, and lightweight agents at each site create outbound tunnels to it. Traffic flows inbound through the cloud proxy and traverses the tunnel to reach internal services. No inbound firewall rules needed. Zero-trust by design.

---

## Architecture

The architecture has three components:

### Pangolin Server

The Pangolin server runs on a cloud VPS and serves as the central entry point for all public traffic. It handles:

- **TLS termination** - all HTTPS certificates are managed at the Pangolin server
- **Request routing** - routes incoming requests to the correct site and service based on hostname
- **Health checking** - monitors tunnel health and backend availability
- **Load balancing** - distributes across multiple tunnels when available

The server is managed through a web dashboard (Fossorial hosted) where I configure sites, resources, and routing rules.

### Newt Agents

Newt is the tunnel agent. Each agent runs as a lightweight service on an LXC container and maintains a persistent outbound connection to the Pangolin server. Key characteristics:

- **Outbound-only** - Newt initiates the connection to Pangolin, never the reverse
- **Persistent tunnel** - automatically reconnects if the connection drops
- **Lightweight** - minimal resource footprint, runs comfortably on small LXC containers
- **Site-aware** - each agent is registered to a specific site in Pangolin's configuration

### How Traffic Flows

```
User (internet)
    |
    v
Pangolin Server (cloud VPS)
    |  TLS terminated
    |  Request routed by hostname
    v
Outbound tunnel (maintained by Newt)
    |
    v
Internal service (on home or DC network)
```

The critical insight is that the tunnel is established from the inside out. The Newt agent reaches out to Pangolin - it never needs to accept inbound connections. This means the firewall at each site can have zero inbound rules for service access.

---

## High Availability - Dual Newt Agents Per Site

Each site runs two Newt agents on separate Proxmox nodes. This provides active-active high availability:

| Site | Agent | Node | Role |
|---|---|---|---|
| DC | Primary Newt | First DC node | Active |
| DC | HA Newt | Second DC node | Active |
| Home | Primary Newt | Primary home node | Active |
| Home | HA Newt | Secondary home node | Active |

Both agents at each site connect to Pangolin simultaneously. This is not active-passive failover - it is genuine active-active. Pangolin routes through both tunnels, and if one goes down, it seamlessly continues routing through the survivor.

### Real-World Validation

I validated this during node maintenance. When taking down the primary home node for updates, all public services continued working without interruption because the HA Newt on the secondary node was already connected and serving traffic. No manual failover, no DNS changes, no downtime.

This is the kind of resilience that would traditionally require a load balancer, health checks, and failover scripts. With dual Newt agents, it's built into the architecture.

---

## Installation and Configuration

### Newt Binary Installation

Newt is distributed as a single static binary. Installation is manual but straightforward:

```bash
curl -fsSL https://github.com/fosrl/newt/releases/latest/download/newt_linux_amd64 \
  -o /usr/local/bin/newt
chmod +x /usr/local/bin/newt
```

I initially tried using community install scripts, but they were unavailable at the time, so I went with the direct binary download approach. This has the advantage of being simple and reproducible.

### Systemd Service

Each Newt agent runs as a systemd service that starts on boot and automatically restarts on failure:

```ini
[Unit]
Description=Newt Tunnel Agent
After=network.target

[Service]
ExecStart=/usr/local/bin/newt --id <AGENT_ID> --secret <AGENT_SECRET> --endpoint https://pangolin.biggie.be
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

The agent ID and secret are generated in the Pangolin dashboard when registering a new site. These credentials are stored in Passbolt - if a Newt agent needs to be rebuilt, the credentials are retrievable without generating new ones.

### Service File Deployment

An interesting detail of my setup: because the Newt agents run inside LXC containers, I can write the systemd service file directly to the container's filesystem from the Proxmox host:

```bash
cat > /path/to/container/rootfs/etc/systemd/system/newt.service << 'EOF'
[Unit]
Description=Newt Tunnel Agent
After=network.target
[Service]
ExecStart=/usr/local/bin/newt --id <ID> --secret <SECRET> --endpoint https://pangolin.biggie.be
Restart=always
User=root
[Install]
WantedBy=multi-user.target
EOF
```

This is useful during initial provisioning or when deploying the HA agents on new nodes.

---

## Health Check Configuration

One operational gotcha cost me some troubleshooting time. Pangolin performs health checks on backend services to determine availability. The default behavior is to check the root path (`/`), but many services redirect from `/` to a login page or dashboard.

This redirect causes a loop in the health check that results in 503 errors - Pangolin thinks the service is down, even though it's perfectly healthy.

The fix: configure health checks to hit `/status` or another endpoint that returns a clean 200 response without redirects. This is a small configuration detail, but it's the difference between a service appearing healthy and throwing intermittent 503s.

---

## Postfix Relay on Newt Agents

Each primary Newt agent runs Postfix as a relay host for sending system notifications. The configuration differs slightly between sites:

| Site | Relay Target | Purpose |
|---|---|---|
| DC primary | Mailcow via private IP | Direct LAN relay (no public transit) |
| Home primary | Mailcow via public hostname | Relay through public endpoint |

The DC agent relays directly to Mailcow's private IP because they're on the same network. The home agent relays through the public mail hostname because it's on a different site. Both use authenticated SMTP on port 587.

Notification emails from Newt agents use a dedicated sender address and route to a shared notifications inbox. This means tunnel status changes, reconnection events, and health check failures all land in the same place as other infrastructure alerts.

---

## What Goes Through Pangolin

Almost every public-facing service in my infrastructure routes through Pangolin:

| Service | Site | Purpose |
|---|---|---|
| Blog | Home | This blog (blog.biggie.be) |
| Wiki | Home | Infrastructure documentation (wiki.biggie.be) |
| PocketID | Home | OIDC identity provider |
| Passbolt | Home | Password manager |
| Plex | Home | Media server |
| Home Assistant | Home | Home automation |
| Guacamole | DC | Remote desktop gateway |
| Monitoring dashboards | Both | Various internal dashboards |

The exceptions are HestiaCP and Mailcow, which need direct public exposure for DNS and email respectively.

---

## Security Model

The security benefits of this architecture are significant:

- **No inbound firewall rules** - the attack surface at each site is reduced to zero for service access
- **TLS termination at the edge** - certificates are managed centrally at the Pangolin server
- **Tunnel authentication** - each Newt agent authenticates with a unique ID and secret
- **Credential rotation** - tunnel credentials can be rotated without changing firewall rules
- **Site isolation** - a compromise of one site's Newt credentials doesn't affect the other site

The only service that needs to accept inbound connections from the internet is the Pangolin server itself. And since it runs on a managed cloud VPS with minimal attack surface (just the Pangolin application and SSH), it's a much smaller target than having multiple services exposed at each site.

---

## Version Management

Newt agent versions are tracked per agent. I don't auto-update - each version bump is a deliberate action:

| Agent | Version |
|---|---|
| DC primary | v1.10.1 |
| DC HA | v1.10.1 |
| Home primary | v1.9.0 |
| Home HA | v1.10.2 |

The home primary agent is still on an older version. This is intentional - it's been stable, and I'll update it during the next scheduled maintenance window. The HA agents were deployed more recently and got newer versions.

---

## Lessons Learned

- **Zero-trust is simpler than traditional port forwarding** - no more maintaining NAT rule tables that grow with every new service
- **Active-active HA with dual agents is trivial to set up** - two Newt agents per site eliminated single points of failure without complex failover logic
- **Health checks need explicit endpoint configuration** - the root path redirect gotcha is common and causes confusing 503 errors
- **Store tunnel credentials in your password manager** - rebuilding a Newt agent is painless when you can retrieve the credentials from Passbolt
- **Not everything should go through the tunnel** - DNS and email need direct public exposure; Pangolin is for everything else
- **Outbound-only connections simplify firewall auditing** - when no inbound rules exist, there's nothing to misconfigure

Pangolin and Newt transformed how I think about service exposure. Instead of "which ports do I need to open?" the question became "which services does Pangolin route to?" That shift in mental model makes the entire infrastructure more secure and easier to manage.
