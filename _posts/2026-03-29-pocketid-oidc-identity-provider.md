---
layout: post
title: "PocketID - OIDC Identity Provider"
date: 2026-03-29 18:40:00 +0200
categories: setup
tags: pocketid oidc identity sso authentication
---

As the number of self-hosted services in my homelab grew, so did the number of login credentials I had to manage. Each service had its own user database, its own password policy, and its own session management. PocketID solves this by providing a centralized OIDC (OpenID Connect) identity provider that services can authenticate against.

## What Is OIDC and Why Does It Matter?

OpenID Connect is an identity layer built on top of OAuth 2.0. In practical terms, it lets you authenticate to multiple services using a single identity. Instead of creating a local account on every service, you configure the service to delegate authentication to an OIDC provider - in this case, PocketID.

The benefits are immediate:

- **Single sign-on (SSO)** - one login, access to all OIDC-enabled services
- **Centralized user management** - disable one account, and access to all services is revoked
- **Consistent authentication policy** - password complexity, session duration, and MFA are enforced at the identity provider level
- **Reduced credential sprawl** - fewer passwords means fewer passwords to rotate, leak, or forget

---

## Architecture

PocketID runs as an LXC container on the home cluster, on a secondary Proxmox node. It's a lightweight service with minimal resource requirements.

| Detail | Value |
|---|---|
| Type | LXC Container |
| Host | Secondary home node |
| Network | IoT VLAN |
| Public URL | pocketid.biggie.be |
| Access | Via Pangolin + Newt tunnel |

### Why the IoT VLAN?

This might seem like an unusual choice. The IoT VLAN in my network is intentionally isolated - devices on it can reach specific services but have no access to the management network. PocketID sits here because:

1. It doesn't need access to management infrastructure
2. It only needs to be reachable by the Newt tunnel agent (for public access) and by internal services that authenticate against it
3. VLAN isolation provides an additional boundary - even if PocketID were compromised, lateral movement to management systems would require crossing VLAN boundaries with firewall rules

Public access to PocketID goes through Pangolin and Newt, following the zero-trust model. No inbound firewall rules are needed.

---

## OIDC Configuration

PocketID acts as the identity provider (IdP) in the OIDC flow. Here's how the authentication process works:

1. A user navigates to a service (e.g., a dashboard or admin panel)
2. The service redirects the user to PocketID's authorization endpoint
3. The user authenticates with PocketID (username/password, or passkey)
4. PocketID issues an ID token and redirects back to the service
5. The service validates the token and creates a session

Each service that integrates with PocketID needs:

- A **client ID** and **client secret** (generated in PocketID's admin interface)
- The **authorization endpoint** URL
- The **token endpoint** URL
- The **userinfo endpoint** URL
- A configured **redirect URI** that PocketID will send the user back to after authentication

### Services Using PocketID

Any self-hosted service that supports OIDC can authenticate through PocketID. This includes dashboards, admin panels, monitoring tools, and collaboration platforms. The list grows as I add new services and configure them for SSO.

---

## Email Notifications

PocketID sends email notifications for security-relevant events. This is configured through the admin UI:

### SMTP Configuration

| Setting | Value |
|---|---|
| SMTP Host | Mailcow instance |
| SMTP Port | 587 (STARTTLS) |
| TLS | Enabled |
| Certificate Verification | Enabled (not skipped) |

### What Triggers Notifications

PocketID sends emails for the following events:

| Event | Purpose |
|---|---|
| New device login | Alerts when authentication occurs from an unrecognized device or location |
| Email verification | Confirms ownership of the email address during registration |
| Login code from admin | Admin-initiated one-time login codes |
| API key expiration | Warning before API keys expire |
| Login code requested by user | Self-service one-time codes |

All of these notifications are enabled. The new-device login notification is particularly valuable - it provides immediate visibility when someone (or something) authenticates from an unexpected location.

### Email Verification Policy

Two important settings:

- **Require Email Address** - enabled. Every user must have a verified email address
- **Emails Verified by Default** - disabled. Users must verify their email before they can authenticate

This ensures that no account exists without a valid, verified email address. It prevents situations where a user account is created with a typo in the email and notifications go to the wrong person (or nowhere).

---

## System-Level Notifications

In addition to the user-facing email notifications, PocketID has Postfix configured at the system level for infrastructure notifications:

| Setting | Value |
|---|---|
| Relay | Mailcow (port 587) |
| From | pocketid@biggie.be |
| To | notifications@biggie.be |

These system-level notifications cover things like service restarts, errors, and administrative events that aren't part of the OIDC flow but are important for operational awareness.

---

## Passkey Support

PocketID supports passkeys (WebAuthn/FIDO2) as an authentication method. This is a significant improvement over traditional password-based authentication:

- **Phishing resistant** - passkeys are bound to the origin (domain), so they can't be used on phishing sites
- **No passwords to leak** - authentication uses public-key cryptography
- **Biometric integration** - can use fingerprint or face recognition on supported devices
- **Cross-device** - passkeys can be synced across devices through the platform authenticator

For day-to-day use, I authenticate with a passkey stored on my devices. Password-based login remains available as a fallback, but passkeys are the primary authentication method.

---

## Future: LDAP Integration

PocketID is a stepping stone toward a more comprehensive identity architecture. The current roadmap includes:

1. **PocketID (current)** - OIDC authentication for services that support it
2. **LDAP directory (planned)** - centralized user directory backed by Synology Directory Server
3. **PocketID + LDAP (goal)** - PocketID authenticates against the LDAP directory, providing both OIDC and LDAP authentication from a single user database

The LDAP directory hasn't been deployed yet, but the plan is to use the Synology NAS at the home site as the directory server. This would give services that only support LDAP (and there are still many) the ability to authenticate centrally, while OIDC-capable services continue using PocketID.

---

## Security Considerations

Running your own identity provider is a responsibility. PocketID is a high-value target - compromise it, and an attacker gets access to every service that trusts it for authentication.

### Mitigation Measures

- **VLAN isolation** - PocketID runs on a segmented network with restricted cross-VLAN access
- **Pangolin access only** - no direct port forwards; all access goes through the zero-trust tunnel
- **Wazuh monitoring** - the LXC container runs a Wazuh agent reporting to the SOC stack
- **PBS backups** - nightly backups with the standard Proxmox Backup Server schedule
- **Email alerts on new device login** - immediate notification of any authentication from an unknown device
- **TLS everywhere** - all communication with PocketID is encrypted
- **Certificate verification enabled** - SMTP connections verify the mail server's certificate (no skipping)

### Token Security

OIDC tokens issued by PocketID have limited lifetimes. Services must re-authenticate when tokens expire, which limits the window of opportunity if a token is somehow intercepted. The token lifetime is configurable per client in PocketID's admin interface.

---

## Operational Notes

### Monitoring Authentication Failures

Failed authentication attempts are logged by PocketID and picked up by the Wazuh agent. Repeated failures from the same source trigger alerts that escalate through the standard security pipeline (Wazuh to TheHive).

### Account Lifecycle

When I decommission a service, I also revoke its OIDC client credentials in PocketID. This ensures that even if someone has the old client secret, they can't authenticate against the identity provider.

Similarly, when adding a new service, the workflow is:

1. Create a new OIDC client in PocketID
2. Note the client ID and client secret
3. Store credentials in Passbolt
4. Configure the service with PocketID's endpoints
5. Test authentication flow end-to-end
6. Verify that email notifications fire on first login

---

## Lessons Learned

- **Centralized identity reduces operational overhead** - managing one user database is dramatically simpler than managing accounts on each service
- **OIDC is widely supported** - most modern self-hosted applications support it natively
- **Email verification should never be optional** - require verified email addresses from the start to avoid notification gaps
- **Passkeys are the future** - phishing-resistant authentication with no passwords to manage
- **Monitor your identity provider aggressively** - it's a high-value target and deserves extra scrutiny
- **Plan for LDAP compatibility** - some services only support LDAP, so plan for a directory server even if you start with OIDC
- **Store OIDC client secrets in your password manager** - treat them like any other credential

PocketID might be one of the smaller services in my infrastructure, but it's one of the most impactful. Single sign-on across the homelab transforms the daily experience of managing multiple services.
