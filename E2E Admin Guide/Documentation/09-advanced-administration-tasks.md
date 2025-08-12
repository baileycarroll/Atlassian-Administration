---

```
title: Advanced Administration Tasks
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, advanced, performance, security, integrations]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for advanced admin tasks (traffic, integrations, performance, security). *Ref: `[Change/Ticket ID]`*

# Purpose

Provide **playbooks and patterns** for non‑routine operations: traffic management (single‑node or clustered), external integrations (LDAP/SSO/app‑links), performance tuning scenarios, bulk/data operations, and hardening. Use these to reduce risk and standardize changes.

---

# 1) Traffic Management & Clustering

> If you run **single‑node**, use §1.1. If you run **Data Center cluster**, use §1.2–1.4.

## 1.1 Single‑Node Reverse Proxy Pattern

**server.xml** (`[Jira Install]/conf/server.xml`):

```xml
<Connector port="8080" protocol="HTTP/1.1"
  connectionTimeout="20000" redirectPort="8443"
  proxyName="[jira.example.org]" proxyPort="443"
  scheme="https" secure="true" />
```

**Apache vhost (template)**:

```apache
<VirtualHost *:443>
  ServerName [jira.example.org]
  SSLEngine on
  SSLCertificateFile    [ /path/to/cert.crt ]
  SSLCertificateKeyFile [ /path/to/key.key ]

  ProxyPreserveHost On
  RequestHeader set X-Forwarded-Proto https
  RequestHeader set X-Forwarded-Port 443
  ProxyPass        /  http://127.0.0.1:8080/
  ProxyPassReverse /  http://127.0.0.1:8080/
</VirtualHost>
```

## 1.2 Cluster (Data Center) Pattern (Reference)

* **LB VIP**: `[lb.example.org]` with **cookie‑based stickiness** (`JSESSIONID`).
* **Shared Home**: `[Shared Storage Path]` mounted RW on all nodes.
* **Node list**: `[jira-app-01]`, `[jira-app-02]`, …

**LB health checks**

* Path: `/status` or `/plugins/servlet/health` (if enabled)
* Expect: `200 OK` within `[3s]`

## 1.3 Session Affinity & Uploads

* Enforce **stickiness** on LB for admin pages, bulk operations, and uploads.
* If stickiness breaks: symptoms include lost sessions, failed attachments, and inconsistent admin screens.

## 1.4 Blue/Green or Canary (Zero‑Downtime Tactics)

* Stand up **Green** on the new version, reindex, validate → switch DNS/LB to Green.
* For **canary**, shift `[5–10%]` of traffic to new node(s), monitor, then roll forward.

---

# 2) External Integrations

## 2.1 Directory / LDAP (Read‑Only Recommended)

* Directory type: `[Active Directory | LDAP]` (read‑only with local groups).
* Bind DN: `[Bind DN]`; Search base: `[OU=People]`, `[OU=Groups]`.
* Filters:

  ```
  user:  (&(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
  group: (|(cn=jira-*)(cn=dev-*)(cn=proj-*))
  ```
* Sync cadence: scheduled + manual. Keep **internal directory first** with a local admin.

## 2.2 SSO (SAML/OIDC)

* IdP: `[IdP Name]`. Required claims: `email`, `displayName`, **immutable ID** (e.g., `uid`).
* Callback/ACS: `[https://jira.example.org/plugins/servlet/samlconsumer]` (SAML) or OIDC callback.
* **JIT/SCIM**: Optional; map IdP groups → Jira groups.

## 2.3 Application Links (AppLinks)

* Tools: `[Confluence]`, `[Bitbucket]`, `[Bamboo]`, `[Artifactory]`.
* Auth: **OAuth (impersonation)** preferred.
* Validate: link both directions where supported, test with least‑privilege user.

## 2.4 Webhooks / Automation / REST

* Outbound webhooks: whitelist `[targets]` and secure with secrets/headers.
* REST API tokens: store in `[Secrets Manager]`; scope to minimum permissions.
* Rate limiting: honor any upstream limits to avoid self‑inflicted outages.

## 2.5 Email (Outgoing & Incoming)

* Outgoing: `[SMTP Host]:[Port]`, From `[From Address]`, Prefix `[JIRA]`.
* Incoming (optional): POP/IMAP handlers; use a dedicated mailbox; test loops.

---

# 3) Performance Tuning Scenarios

> Start with **measurement** (observability) before tuning. Change **one dimension** at a time.

## 3.1 JVM & GC

* Heap: `-Xms[8g] -Xmx[8g]` (equal values to start).
* Flags (baseline): `-Datlassian.plugins.enable.wait=300 -Duser.timezone=[TZ]`.
* Observe: GC pause time, allocation rate, old‑gen occupancy.

## 3.2 Thread & Connection Pools

* Tomcat threads: tune `maxThreads` only if 5xx backlog under load.
* DB pool (`dbconfig.xml`): start `pool-min-size=[20]`, `pool-max-size=[100]`; align with DB `max_connections`.
* Consider **PgBouncer session mode** for bursty loads.

## 3.3 Search Indexing

* Prefer **Foreground Reindex** after big upgrades or scheme changes.
* Background/as‑you‑go indexing for routine ops; monitor index lag.

## 3.4 Database Hotspots

* Watch: slow queries (JQL heavy joins), lock waits, replication lag.
* Tune PostgreSQL: `shared_buffers`, `work_mem`, autovacuum scale factors on `jiraissue`, `changeitem`, `worklog`.

## 3.5 Large Instance Hygiene

* Archive old projects/issue types.
* Cap **attachment size**; enforce image thumbnails.
* Limit expensive filters (wide JQL, large boards).

---

# 4) Security Hardening

* Reverse proxy headers: set `X‑Forwarded‑Proto`, `X‑Frame‑Options: DENY`, `X‑Content‑Type‑Options: nosniff`, **HSTS**.
* Content Security Policy (CSP): define if custom gadgets/plugins are controlled.
* Global permissions: assign to **groups**, not users; keep admin groups small; MFA enforced.
* Secrets: store DB/IdP/API tokens in `[Secrets Manager]`; avoid in repo.
* Audit: enable admin/audit logs; retain per `[Retention Policy]`.

---

# 5) Data & Bulk Operations

## 5.1 Project Archive/Unarchive

* Preconditions: snapshot DB + Jira home; communicate impact.
* Archive via UI (Data Center) or export/import if unsupported.

## 5.2 Bulk Changes (Issues/Fields)

* Use a **non‑peak window**; test in non‑prod first.
* After completion, **reindex** affected projects; verify with saved filters.

## 5.3 Attachment Moves / Storage Changes

* Plan rsync with Jira **stopped** for consistency, or use maintenance window.
* Update `jira.path.attachments` if relocated; validate permissions.

---

# 6) Change Patterns & Playbooks

| Pattern              | Steps (summary)                                     | Rollback                           |
| -------------------- | --------------------------------------------------- | ---------------------------------- |
| Plugin/App upgrade   | Test → snapshot → upgrade → smoke → release         | Revert to snapshot; disable plugin |
| Base URL change      | Update server.xml + General Config + proxy; restart | Restore config; DNS revert         |
| Certificate rotation | Install at proxy/LB; reload; verify expiry          | Reinstall prior cert; reload       |
| JVM heap change      | Edit `setenv.sh`; restart; watch GC                 | Revert values; restart             |
| Schema add‑ons       | Install in Test; reindex; validate                  | Remove add‑on; restore             |

Attach detailed runbooks per change in your **Change Management** section.

---

# 7) Troubleshooting Playbooks (Quick Paths)

## 7.1 High CPU / 5xx Spikes

1. Check heap/GC; thread dumps if needed.
2. Review reverse proxy logs for surge or errors.
3. Inspect DB for lock waits or slow queries.

## 7.2 Slow Search / Index Drift

1. Check index status; consider Foreground Reindex.
2. Validate `jira-home` storage IOPS/latency.

## 7.3 Email Failures

1. Test SMTP connectivity; review mail queue.
2. Verify from/prefix; check relay throttling.

## 7.4 SSO/Login Issues

1. Use local admin; inspect IdP assertions.
2. Check clock skew, certificate expiry.

---

# 8) Operational Guardrails

* Max attachment size: `[X MB]`.
* Board/Filter limits: `[guideline]` for JQL complexity.
* Marketplace apps: **pin versions**; dev/test first; keep a rollback note.
* Avoid editing **system schemes**; clone then modify.

---

# Known Pitfalls (Advanced)

* Disabling stickiness in a cluster → broken admin screens/uploads.
* Migrating attachments **while Jira is running** → inconsistent state.
* Oversizing DB connection pool → idle bloat and timeouts.
* CSP too strict without testing → gadgets/dashboards break.
* Unbounded logs at proxy/app → disk exhaustion.

---

# Variables Used on This Page

`[jira.example.org]`, `[Jira Install]`, `[Shared Storage Path]`, `[lb.example.org]`, `[Bind DN]`, `[OU=People]`, `[IdP Name]`, `[Secrets Manager]`, `[Retention Policy]`, `[TZ]`, `[X MB]`.

# Import Notes

* Keep the **metadata code block** intact; it imports cleanly to Confluence.
* Diagrams and config snippets are minimal and portable; expand locally if needed.
* Link playbooks here from your Change Management and Troubleshooting sections for continuity.
