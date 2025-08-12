---

```
title: Troubleshooting & Diagnostics
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, troubleshooting, diagnostics]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for incident triage and diagnostics for Jira Data Center v10+ (app, proxy, DB). *Ref: `[Change/Ticket ID]`*

# Purpose

Provide a **repeatable path** to go from symptom → root cause → verified fix. This is the **on‑call quick kit** and the deeper **investigation guide**.

---

# 1) Severity & First Moves

| Sev    | Definition                               | First Moves                                                      |
| ------ | ---------------------------------------- | ---------------------------------------------------------------- |
| **P1** | Broad outage or login/search unusable    | Declare incident, start comms, snapshot metrics, begin §3 triage |
| **P2** | Degraded (slow, partial feature failure) | Notify stakeholders, collect logs/metrics, schedule fix window   |
| **P3** | Minor/isolated issue                     | Ticket with impact and owner, backlog if non‑urgent              |

**10‑Minute Decision Tree**

```
Is homepage 200? ── yes ─┬─ Can users log in? ─ yes ─┬─ DB error in logs? → §6
                       │                      │
                       │                      └─ Slow only → §7 Performance
                       │
                       └─ no → Check proxy/DNS → §4  ▸ If DB conn errors → §6
```

---

# 2) Symptom → Likely Causes → First Checks

| Symptom                                | Likely Causes                                                | First Checks                                                   |
| -------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------- |
| **Homepage down**                      | Proxy misconfig, Jira not running, TLS/cert expiry           | `curl -I [Base URL]`; `systemctl status jira`; proxy error log |
| **Login fails**                        | IdP outage, clock skew, SSO plugin config, cookie/domain     | Try **local admin**; check SAML/OIDC response; server time/NTP |
| **500/503 spikes**                     | Thread/DB pool exhaustion, GC pauses, reverse proxy timeouts | `journalctl -u jira`; DB connections; GC logs; proxy timeout   |
| **Search broken / "reindex" warnings** | Index corruption, low disk/IOPS                              | UI → System → **Indexing**; disk usage; foreground reindex     |
| **Attachments fail**                   | Proxy `MaxRequestSize`, tmp space, permissions               | Proxy/vhost limits; `/tmp` and home `export/` space            |
| **Email not sending**                  | SMTP relay down, auth/port wrong                             | Test from UI; `nc -vz [SMTP Host] [SMTP Port]`; mail logs      |
| **Slow pages**                         | DB locks, heavy boards/JQL, low heap, noisy neighbor         | p95 latency; `pg_stat_activity`; heap/GC; access log hotspots  |
| **DB replication lag**                 | Network/WAL backlog, slot bloat                              | `pg_last_xact_replay_timestamp()`; WAL disk; slots             |

---

# 3) Collect Facts (5 minutes)

* **Timestamps**: incident start, detection, first page, mitigations.
* **Scope**: all users vs project; UI vs API.
* **Recent changes**: deployments, config, DB, network, certificates.
* **Artifacts**: screenshots, error IDs, request IDs, issue keys.

Create an **incident note**:

```
When:
What changed:
Who is impacted:
Hypotheses:
Next 2 actions:
```

---

# 4) Network / Proxy / TLS

**Quick checks**

```bash
# Status & headers
curl -sSI https://[Base URL] | sed -n '1,15p'

# Resolve & route
getent hosts [Base URL]

# Local port bound only (expected for single-node)
sudo ss -ltnp | egrep '8080|443'
```

**Apache (example hints)**

* Ensure `ProxyPass / http://127.0.0.1:8080/` and `ProxyPreserveHost On`.
* Set headers: `X-Forwarded-Proto https`, `X-Forwarded-Port 443`.
* Check access/error logs for 5xx/timeouts.

**Jira `server.xml`**

```xml
<Connector port="8080" proxyName="[Base Host]" proxyPort="443" scheme="https" secure="true" />
```

---

# 5) Application Health (JVM/Tomcat)

**Service & logs**

```bash
sudo systemctl status jira
sudo journalctl -u jira --since "-30m" | tail -n +1
```

**Heap/GC and threads**

```bash
ps -o pid,etimes,rss,vsz -C java
lsof -p $(pgrep -f catalina) | wc -l
# If enabled, review GC lines
grep -i gc [Jira Home Path]/logs/atlassian-jira.log | tail -n 50
```

**Thread dump (only if needed)**

```bash
jstack $(pgrep -f catalina) > /tmp/jstack.$(date +%s).txt
```

---

# 6) Database Connectivity & Health (PostgreSQL)

**Connectivity**

```bash
nc -vz [PG Host] [PG Port]      # from app host
psql -h [PG Host] -U [PG User] -d [PG Database] -c 'select 1'
```

**Hotspots & locks**

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity WHERE wait_event IS NOT NULL;

SELECT relname, n_live_tup
FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 20;
```

**Replication (if used)**

```sql
SELECT now() - pg_last_xact_replay_timestamp() AS replica_lag;
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
```

---

# 7) Performance Deep Dive

* **p95/p99 latency**: dashboard or proxy logs.
* **DB pool saturation**: compare Jira pool max vs DB `max_connections`.
* **GC pauses**: sustained > 500ms indicates pressure; consider heap tune.
* **Noisy boards/JQL**: profile slow filters; add indexes only with care.

**Access log hotspot (nginx example)**

```
awk '{print $7}' access.log | sort | uniq -c | sort -nr | head -20
```

---

# 8) Search / Index Troubleshooting

* UI → **System → Indexing**: status **Online**? size plausible?
* If warnings persist after changes, schedule **Foreground Reindex**.
* Verify disk space and IOPS for `jira-home`.

**Foreground Reindex (safe path)**

1. Recent backups confirmed.
2. Schedule off‑hours window.
3. Run **Lock and Rebuild**.
4. Validate JQL and recent issues.

---

# 9) Email Troubleshooting

* UI test email; review result.
* From app host:

```bash
nc -vz [SMTP Host] [SMTP Port]
```

* Check relay logs and spam/bounce handling.

---

# 10) SSO / Directory

* Try **local admin** (internal directory) to bypass IdP.
* Decode SAML response (if SAML) and verify NameID/group claims.
* Check clock skew (`timedatectl`) and certificate validity.

---

# 11) Storage & Filesystem

```bash
df -h
inode - if available
sudo du -sh [Jira Home Path]/{logs,temp,export}
```

* Ensure Jira home permissions are `jira:jira` and writable.
* Large support zips or export files can fill disk quickly.

---

# 12) Known Faults & Quick Fixes (Fill per org)

| Fault                 | Signature                         | Fix                                             |
| --------------------- | --------------------------------- | ----------------------------------------------- |
| Proxy header mismatch | Infinite redirect, wrong base URL | Fix proxy headers + `server.xml` proxyName/Port |
| SMTP port blocked     | Test email fails fast             | Open 25/587; correct relay and from address     |
| DB pool exhausted     | 500s, `Cannot get a connection`   | Reduce app threads; increase pool; check DB max |
| Index out of date     | Search misses recent issues       | Foreground reindex during low‑traffic window    |

---

# 13) Escalation & Vendor Support

* **When to escalate**: P1, security exposure, or blocked diagnosis > **30 minutes**.
* **Who**: `[On‑Call Group]` → `[Escalation Group]` → **Service Owner**.
* **Vendor**: Open a support case with **Atlassian**; attach logs, support zip, timestamps, and reproduction steps.

**Support Zip**

* UI: **System → Troubleshooting and Support → Create Support Zip** (include logs, thread dumps if instructed).

---

# 14) Incident Closeout Checklist

* [ ] Root cause (or best hypothesis) documented.
* [ ] Fix validated with **smoke tests**.
* [ ] Metrics back to baseline; error budget impact noted.
* [ ] Action items created (owner + date) for prevention.
* [ ] Comms sent (closure + learnings as appropriate).

---

# Variables Used on This Page

`[Base URL]`, `[Jira Home Path]`, `[SMTP Host]`, `[SMTP Port]`, `[PG Host]`, `[PG Port]`, `[PG User]`, `[PG Database]`, `[On‑Call Group]`, `[Escalation Group]`.

# Import Notes

* Keep the **metadata code block** intact for portability.
* Tables and code fences import cleanly to Confluence; convert relative links post‑import.
* Replace placeholders systematically; keep commands copy‑pastable and minimal.
