---

```
title: Security Model
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Security/Platform Team]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [e.g., FERPA?|HIPAA?|N/A]
tags: [admin-guide, jira, security, sso, ldap, tls, secrets]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for Jira Security Model (identity, access, TLS, secrets, auditing). *Ref: `[Change/Ticket ID]`*

# Purpose

Establish **how users authenticate**, **how access is granted**, **how data is protected**, and **how changes are audited** for Jira Data Center v10+. This is the **source of truth** for SSO/LDAP, permission archetypes, secrets, TLS, and compliance hooks.

---

# Security Principles (Anchor)

* **Least privilege** for users, groups, roles, tokens, and service accounts.
* **Separation of duties** between platform admins, project admins, and DB/infra.
* **Defense in depth**: proxy → app → DB → backups; encryption in transit and at rest.
* **Auditability**: admin actions, permission changes, and access are reviewable.
* **Secure by default**: deny‑by‑default firewall, MFA for admins, pinned plugin versions.

---

# Controls at a Glance

| Domain   | Control                                               | Owner         | Evidence                              |
| -------- | ----------------------------------------------------- | ------------- | ------------------------------------- |
| Identity | SSO via `[IdP Name]` + LDAP/SCIM sync                 | `[Team/Role]` | IdP metadata, directory config export |
| Access   | Standard groups/roles + hardened global permissions   | `[Team/Role]` | Screenshots/exports of schemes        |
| Admin    | Break‑glass local admin; MFA on admin groups          | `[Team/Role]` | Admin group roster, MFA proof         |
| TLS      | Terminate at `[Proxy/LB]`; HSTS/CSP headers           | `[Team/Role]` | vhost config, SSL scan report         |
| Secrets  | Stored in `[Secrets Manager]`, rotated per `[Policy]` | `[Team/Role]` | Vault path/policy, rotation logs      |
| Logging  | App/proxy/DB logs retained `[90+ days]`               | `[Team/Role]` | Retention config, sample searches     |
| Backups  | Encrypted, restore‑verified quarterly                 | `[Team/Role]` | Restore report, change ref            |

---

# 1) Identity & Access Management

## 1.1 Directories & SSO

* **Directory**: `[Active Directory | LDAP]` (read‑only with local groups). Keep **Internal Directory** first with 2–4 local admins.
* **SSO**: `[SAML 2.0 | OIDC]` via `[IdP Name]`.

  * **ACS/Callback**: `[https://jira.example.org/plugins/servlet/samlconsumer]` (SAML) or OIDC callback URL.
  * **Claims**: `email`, `displayName`, **immutable ID** (e.g., `uid`), **groups** (if JIT mapping).
  * **JIT/SCIM**: Optional; map IdP groups → Jira groups.

**Risk Notes**

* Ensure **clock sync** (NTP) to prevent token/assertion issues.
* Maintain at least **one local sysadmin** not tied to IdP.

## 1.2 Group Model (Global)

| Group                 | Purpose                              |
| --------------------- | ------------------------------------ |
| `jira-sysadmins`      | Break‑glass local administrators     |
| `jira-administrators` | Day‑to‑day Jira admin (no OS access) |
| `jira-software-users` | Product access (license gate)        |
| `jira-readonly`       | View‑only baseline                   |
| `jira-automation`     | Bots/integrations/service accounts   |

> Assign **global permissions** to groups, not individuals. Keep admin groups small and MFA‑enforced.

## 1.3 Project Role Archetypes

| Role             | Use                           | Backing Group (example)           |
| ---------------- | ----------------------------- | --------------------------------- |
| `Administrators` | Project config and boards     | `jira-project-admins` + team lead |
| `Developers`     | Create/edit/transition issues | `dev-<team>`                      |
| `Contributors`   | Comment/attachments/log work  | `staff-<dept>`                    |
| `Viewers`        | Browse only                   | `jira-readonly`                   |
| `Automation`     | App/bot actions               | `jira-automation`                 |

## 1.4 Global Permissions — Hardened Defaults

| Global Permission               | Assign To                                             |
| ------------------------------- | ----------------------------------------------------- |
| Jira System Administrators      | `jira-sysadmins`                                      |
| Jira Administrators             | `jira-administrators`                                 |
| Browse Users / Share Dashboards | `jira-software-users`                                 |
| Create Shared Objects           | Restrict to `jira-administrators` or trusted creators |

## 1.5 Elevated Access & Break‑Glass

* Maintain **2–4 local admins** in Internal Directory.
* Enforce **MFA** for admin groups; time‑box elevated access.
* Record all admin group changes with `[change_ref]`.

## 1.6 Service Accounts & API Tokens

* Create **non‑person** accounts for automation/integration.
* Scope permissions minimally; avoid shared credentials.
* Store secrets in `[Secrets Manager]`; rotate per `[Policy]`.

## 1.7 Joiner/Mover/Leaver (JML)

* **Joiner**: IdP → group assignment → JIT/SCIM creates user → product access via `jira-software-users`.
* **Mover**: update groups; review project role mappings.
* **Leaver**: disable at IdP; remove from groups; transfer ownership of filters/dashboards.

---

# 2) Authentication & Session Settings

**Jira (server.xml)**

```xml
<Connector port="8080" proxyName="[jira.example.org]" proxyPort="443" scheme="https" secure="true" />
```

**Session**

* Timeout: `[30–60]` minutes (align with org policy).
* Remember‑me: `[enabled/disabled]` per policy.
* Cookies: `Secure`, `HttpOnly`; set domain to `[example.org]` as needed.

**CSRF/XSRF**

* Ensure built‑in protection is **enabled**; avoid custom endpoints that bypass checks.

---

# 3) Network, TLS & HTTP Security

## 3.1 Termination & Headers (at `[Proxy/LB]`)

**Apache vhost (template)**

```apache
<VirtualHost *:443>
  ServerName [jira.example.org]
  SSLEngine on
  SSLCertificateFile    [ /path/to/cert.crt ]
  SSLCertificateKeyFile [ /path/to/key.key ]

  # Security headers
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
  Header always set X-Frame-Options "DENY"
  Header always set X-Content-Type-Options "nosniff"
  # Consider a CSP tailored to your gadgets/plugins
  # Header always set Content-Security-Policy "default-src 'self'"

  ProxyPreserveHost On
  RequestHeader set X-Forwarded-Proto https
  RequestHeader set X-Forwarded-Port 443
  ProxyPass        /  http://127.0.0.1:8080/
  ProxyPassReverse /  http://127.0.0.1:8080/
</VirtualHost>
```

## 3.2 TLS Policy

| Control       | Target                                    |
| ------------- | ----------------------------------------- |
| Protocols     | `TLSv1.2+` (consider `1.3` if supported)  |
| Ciphers       | `[Org standard cipher suites]`            |
| OCSP/Stapling | `[enabled/disabled]` per proxy capability |
| HSTS          | `max-age ≥ 31536000`, includeSubDomains   |

## 3.3 Certificate Lifecycle

* **Source**: `[Internal CA | Public CA]` managed by `[Team/PKI]`.
* **Rotation**: renew ≥ **30 days** pre‑expiry; track in calendar.
* **Storage**: file paths recorded; permissions restricted; keys not in repos.

---

# 4) Secrets Management

* Store DB passwords, IdP keys, API tokens in **`[Secrets Manager]`**.
* Grant **least privilege** read to the Jira service; avoid world‑readable files.
* Rotate secrets per **`[Rotation Policy]`** (e.g., quarterly, on staff change).
* Never commit secrets to SCM; use environment variables or mounted files with tight ACLs.

---

# 5) Data Protection & Classification

| Layer        | Control                                                                  |
| ------------ | ------------------------------------------------------------------------ |
| In transit   | TLS `https://` via proxy with strict headers                             |
| At rest (DB) | Storage encryption `[LUKS/EBS/KMS]` or DB‑level if available             |
| Backups      | Encrypted at rest and in transit; access scoped; restore tests quarterly |
| Jira Home    | File permissions `jira:jira`, minimal `umask`                            |
| Attachments  | Virus/malware scanning `[Tool]` if policy requires                       |

**Classification**

* Define data classes `[Public, Internal, Confidential, Restricted]`.
* Prohibit storing `[PHI/PCI/SSN]` unless explicitly approved; enable **Issue Security Scheme** where needed.

---

# 6) Auditing & Retention

* Enable Jira’s **Audit Log** at **`[Coverage Level: base/advanced]`**.
* Retain app/proxy/DB logs for **`[N days]`** hot and **`[N days]`** cold per policy.
* Forward to `[SIEM/Log Platform]` if available.
* Review **admin group membership** and **global permission** changes **quarterly**.

---

# 7) Vulnerability & Patch Management

* **Cadence**: monthly OS patching; Jira/JDK/JDBC updates **per release policy**.
* **CVE triage**: patch **critical** within **≤ 7 days**, **high** within **≤ 30 days**.
* Track via `[Ticketing/Change System]`; link advisories and test notes.

---

# 8) Marketplace Apps (Add‑ons) Governance

* Maintain a **compatibility matrix** (see section 13).
* Evaluate **security posture** of each app (vendor, scope, data egress).
* Test in **Test**; pin versions; keep a **rollback** note.
* Disable/remove unused apps to reduce attack surface.

---

# 9) Security Runbooks

Create concise runbooks for:

* **Compromised account** (reset, token revoke, audit trail, comms)
* **SSO outage** (switch to local login, IdP escalation)
* **TLS cert expiry** (install new cert, verify headers)
* **Suspicious activity** (log review, session invalidation, evidence capture)

---

# 10) Compliance Mapping (Fill per org)

| Control Area    | Policy/Standard      | Evidence                              |
| --------------- | -------------------- | ------------------------------------- |
| Identity & MFA  | `[Org IAM Standard]` | IdP config, MFA report                |
| Data Protection | `[Policy ID]`        | TLS scans, backup encryption proof    |
| Logging & Audit | `[Policy ID]`        | Log retention settings, audit exports |
| Change Control  | `[Policy ID]`        | RFCs with plan/rollback/verification  |

---

# Variables Used on This Page

`[IdP Name]`, `[Proxy/LB]`, `[jira.example.org]`, `[Secrets Manager]`, `[Policy]`, `[Rotation Policy]`, `[SIEM/Log Platform]`, `[Ticketing/Change System]`, `[Team/Role]`.

# Import Notes

* Keep the **metadata code block** intact; Confluence preserves fenced blocks.
* Replace placeholders consistently; link to the **Add‑on Compatibility Matrix** and **Change Management** sections for continuity.
* Keep Apache/nginx header examples minimal; adapt to your proxy standard.
