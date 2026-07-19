---
layout: post
title: "One Worker, Four Databases: Building an Edge Status Monitor on Cloudflare"
date: 2026-07-19 10:00:00 +0200
categories: homelab
tags: cloudflare workers serverless monitoring edge d1 kv r2 durable-objects
---

Everyone who runs a homelab eventually learns the same lesson: the worst place to host your status page is your homelab. When the hypervisor is down, so is the page that's supposed to tell you the hypervisor is down. I've had monitoring inside the lab for a long time - Wazuh watches security events, Home Assistant pings things, PBS mails me about backups - but all of it lives *inside* the walls it's supposed to be watching.

So I spent an afternoon building the opposite: a status monitor that lives on Cloudflare's edge, checks my public services every five minutes from outside, keeps its history in a database I never have to patch, and costs exactly nothing. Along the way it grew a link shortener, a file drop, per-IP rate limiting and a login - because the real goal was to kick the tires on the whole Cloudflare Workers platform, not just one feature.

## The Platform in One Paragraph

Cloudflare Workers are serverless functions that run on V8 isolates in 300+ locations - sub-millisecond cold starts, no containers to babysit. What makes them interesting for a project like this isn't the compute, it's the storage that plugs in next to it. There are four primitives, and instead of reading about them I wanted each one doing a real job in the same app: **KV** (an eventually-consistent key-value store, replicated everywhere, perfect for read-heavy data), **D1** (a real SQLite database with migrations and point-in-time recovery), **R2** (S3-compatible object storage with zero egress fees), and **Durable Objects** (the odd one - a globally-unique, single-threaded instance that owns its own storage).

---

## Who Does What

The division of labour fell out naturally, and it's a nice illustration of when to use which:

**KV holds the latest status snapshot.** A cron trigger probes every monitored URL each five minutes and writes one JSON blob. The status page reads it from the nearest edge location, so it loads in milliseconds from anywhere - and it keeps working when my entire lab is a brick. Eventual consistency (writes take up to a minute to propagate globally) is completely irrelevant for data that's refreshed every five minutes anyway. That's the KV sweet spot: written rarely, read fast, staleness tolerated.

**D1 holds everything relational.** Check history (which powers 24h/7d uptime percentages and the latency sparklines), the list of monitored targets - editable from the page itself, no redeploy - plus the shortlink table and file metadata. It's honest SQLite: prepared statements, batch transactions, `wrangler d1 migrations apply`. Thirty days of five-minute checks across a dozen targets is ~100k tiny rows, which D1 doesn't even notice.

**R2 is the file drop.** Upload streams straight from the request body into the bucket - the Worker never buffers the file - and the killer feature is zero egress fees. As someone who pays attention to what off-site backup providers charge for restores, object storage where getting data *out* is free changes the math.

**A Durable Object does the rate limiting**, and this is the primitive worth understanding. Each client IP maps to exactly one instance (`idFromName(ip)`), and that instance processes requests one at a time. A request counter that can never race, without locks, transactions, or prayer - something you fundamentally cannot build on an eventually-consistent store. Thirty requests a minute per IP, and the 31st gets a clean HTTP 429.

---

## The Details That Make Monitoring Honest

A status page that just shows the last ping is a toy. Three things make this one behave like it means it:

*Three states, not two.* A site that answers in four seconds isn't "up" in any way that matters. Anything over a per-target threshold (default 1500 ms) shows as SLOW - amber, not green - with the threshold editable per target.

*Flap protection.* State only flips after **two consecutive** identical observations, so a single dropped probe doesn't page anyone at 3 AM. Every real transition is logged to D1 and optionally POSTed as JSON to a webhook - ntfy, Home Assistant, anything that eats JSON. Until a webhook is configured, transitions just show in a "Recent events" table on the page.

*Heartbeats for the things the edge can't see.* This is my favourite part. An external monitor can't probe LAN-only services - Proxmox nodes, the NAS, internal DNS. So for those, the logic inverts from pull to push: the service proves it's alive.

```bash
# /etc/cron.d on any LAN host - every 5 minutes
*/5 * * * * root curl -s -X POST \
  "https://<worker-url>/api/heartbeat/pve0?interval=300" \
  -H "Authorization: Bearer $HEARTBEAT_TOKEN" >/dev/null
```

Miss two intervals and the service goes DOWN through the exact same flap-guard-and-alert pipeline as the public checks. Dead man's switch monitoring, no inbound firewall holes, no agents - one line of cron.

---

## Small Things I Liked

The whole site sits behind a login: a password stored as a Worker secret, an HMAC-signed cookie, no session storage anywhere - rotating the secret instantly logs everyone out. Login attempts run through the same Durable Object rate limiter, so brute force gets throttled by design. The sparklines are hand-rolled inline SVG computed from D1 in one grouped query - no chart library, no client-side framework, the status page is a single HTML response. And local development needs no Cloudflare account at all: Miniflare simulates all four storage primitives on disk, cron included.

---

## Gotchas for the Search Engines

A few things that cost me minutes so they don't cost you hours: R2 must be enabled once in the dashboard before the API can create buckets (error 10042). The first request right after a deploy can hit a stale edge location and serve the previous version - retry before you debug. `wrangler dev` does not hot-reload `.dev.vars`. Shortlink routes need to accept HEAD as well as GET, because `curl -I` is the first thing everyone tries. And my favourite: you cannot write the cron expression `*/5` inside a JSDoc comment, because `*/` ends the comment. Spell it out in words.

---

## The Bill

Zero. 100k Worker requests a day free, and the KV/D1/R2/DO allowances cover this workload with room to spare. The cron fires 288 times a day; the databases hold a few megabytes. For a monitor watching ten public sites and a growing list of internal heartbeats, the free tier isn't a trial - it's the plan.

*Honest footnote: this project was pair-built with Claude driving the Cloudflare skills and wrangler from a cloud session - design decisions and the homelab context were mine, the typing mostly wasn't. The future is weird and I'm here for it.*
