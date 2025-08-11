---
title: Routine Maintenance
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, maintenance, postgres]
---

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for day‑to‑day, weekly, and monthly care of Jira Data Center v10+ and PostgreSQL. *Ref: `[Change/Ticket ID]`*

# Purpose

Provide a **repeatable operating rhythm**: what to check, how often, and how to act without surprises. Use these tasks to keep performance stable, reduce incidents, and ensure fast recovery when things break.

> This page assumes you have completed **03 Pre‑Installation Requirements** and **04 Installation & Initial Configuration**.

---

# Cadence at a Glance

| Frequency     | Focus                                                                        |
| ------------- | ---------------------------------------------------------------------------- |
| **Daily**     | Service health, queue backlogs, disk/headroom, error hotspots                |
| **Weekly**    | Index health, app & plugin hygiene, minor config drift checks                |
| **Monthly**   | Patching, DB maintenance windows (VACUUM/ANALYZE focus), backup drills       |
| **Quarterly** | Restore verification, capacity review, permission audits, TLS/license review |

---

# Daily Tasks — Done‑When

* [ ] Homepage and login respond < **2.5s p95** (or your SLO) and return **HTTP 200**.
* [ ] No **critical errors** in `atlassian-jira.log` in the last 24h.
* [ ] **Disk usage** on app and DB hosts < **80%**; log dirs < **10 GB** (or your limit).
* [ ] **SMTP** test mail succeeds; mail queue is empty.
* [ ] **Backup jobs** for last 24h show **SUCCESS**; WAL shipping/replication green.

**Quick Commands**

```bash
# App host
curl -s -o /dev/null -w "%{http_code}\n" [https://jira.example.org]
sudo du -sh [Jira Home Path]/logs 2>/dev/null || true
sudo journalctl -u jira --since "-24h" | egrep -i "(SEVERE|ERROR)" | tail -n 50

# DB health (psql)
SELECT now() - pg_last_xact_replay_timestamp() AS replica_lag;  -- on replicas
SELECT count(*) FROM pg_replication_slots; -- slots status
```

---

# Weekly Tasks — Done‑When

* [ ] **Foreground reindex not required** (no persistent index warnings; search is consistent).
* [ ] **Application logs** rotated and archived as per policy.
* [ ] Review **top slow web actions** and **GC pauses**; no regression from last week.
* [ ] **Plugin/app updates** reviewed and **pinned**; no unapproved upgrades pending.
* [ ] **Directory/SSO** syncs complete; no stale or failed sync jobs.

**How‑To: Rotate Jira Logs (example if using system logrotate)**

```bash
sudo logrotate -f /etc/logrotate.d/jira  # or verify built-in rotation settings
```

**How‑To: Assess Index Health**

* UI → **System → Indexing**: ensure status is **Online** and recent.
* If issues or schema changes: schedule a **Foreground Reindex** during a low‑usage window.

---

# Monthly Tasks — Done‑When

* [ ] **OS patches** applied to app and DB hosts (outside business hours) and Jira rebooted cleanly.
* [ ] **Java & JDBC** updates evaluated and, if approved, applied.
* [ ] **Database maintenance** executed (see section below) with results recorded.
* [ ] **Backups**: at least one **test restore** to a disposable environment performed.
* [ ] **SLO report** generated (availability/performance) and reviewed with stakeholders.

---

# Quarterly Tasks — Done‑When

* [ ] **Restore verification**: full DB + Jira home restore; Jira starts and a sample project is usable.
* [ ] **Capacity review**: heap, CPU, I/O, DB connections vs. ceilings; plan upgrades if >70% sustained.
* [ ] **Permission & group audit**: admin groups minimal; shared objects reviewed.
* [ ] **TLS & license** review: expiry ≥ 30 days away; renewal tracked.
* [ ] **Change process & runbooks** kept current; DR contact tree verified.

---

# Standard Procedures

## A) Controlled Restart (Single Node)

**When**: patches, config changes, or memory leaks suspected.

1. Announce outage (if Prod) and confirm maintenance window.
2. Take current backup (DB snapshot + Jira home tar/snapshot).
3. `sudo systemctl stop jira`
4. Validate port 8080 closed; tail logs to zero.
5. `sudo systemctl start jira` and watch logs.
6. Run smoke tests (homepage, login, issue create, mail test).
7. Close change; attach logs and timing.

## B) Rolling Restart (Clustered) — Template

> Skip if you run single‑node; keep for future scaling.

For each node **N**:

1. Drain traffic (LB or remove node from pool).
2. Stop Jira on node N; apply change.
3. Start Jira; wait for node to rejoin cluster and index replicate.
4. Return node to pool; move to node N+1.

## C) Safe Reindex

1. Confirm recent backups.
2. UI → System → Indexing → **Lock and Rebuild** (foreground preferred for consistency).
3. Monitor logs for `IndexTask` errors; ensure completion.
4. Run search sanity tests (recent issues, JQL filters).

## D) Plugin/App Update Workflow

1. Review **release notes** and compatibility for target Jira version.
2. Snapshot current state (DB + Jira home).
3. Upgrade **in Test** → run smoke and project workflows.
4. Approve and schedule **Prod** update in a maintenance window.

## E) TLS Certificate Rotation (at Proxy/LB)

1. Obtain new cert/key from `[Cert Authority/Team]`.
2. Install into proxy (Apache/nginx) and reload service.
3. Verify `curl -I https://[Base URL]` shows the new expiry date.

## F) Email Health

* Send a test email monthly; confirm delivery path with `[SMTP Host]`.
* Check mail logs for rejects; adjust SPF/DMARC if applicable.

---

# PostgreSQL Maintenance

> Coordinate with DBAs; adjust windows to match data size.

## Recommended Cadence

| Task            | Cadence               | Notes                                                |
| --------------- | --------------------- | ---------------------------------------------------- |
| VACUUM (auto)   | Continuous            | Ensure `autovacuum` thresholds tuned for Jira tables |
| ANALYZE         | Weekly                | Keeps planner statistics fresh                       |
| REINDEX         | Quarterly or on bloat | Target heavily updated indexes                       |
| `pg_dump` smoke | Monthly               | Quick logical dump to test permissions and size      |

## Useful Queries

```sql
-- Table/index bloat (approx)
SELECT schemaname, relname, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 20;

-- Autovacuum activity
SELECT relname, last_vacuum, last_autovacuum, vacuum_count, autovacuum_count
FROM pg_stat_user_tables ORDER BY last_autovacuum NULLS FIRST LIMIT 20;

-- Lock wait hot spots
SELECT pid, usename, state, wait_event_type, wait_event, query
FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

**Reindex Example**

```sql
REINDEX INDEX CONCURRENTLY IF EXISTS public.issue_pkey; -- example; choose targets with care
```

---

# Housekeeping

* **Temp & cache**: Do **not** delete Jira caches while running. If needed, stop Jira, then clear `plugins/.bundled-plugins` only when directed by support notes.
* **Support zips**: Collect and purge old support zip bundles from `[Jira Home Path]/export`.
* **Attachments hygiene**: Large orphaned attachments are rare; prefer app‑level cleanup.

---

# Observability Hooks

* Dashboards for **p95 latency**, **error rate**, **queue lengths**, **heap usage**, **GC**.
* Alerts for: JVM heap > **85% sustained**, DB connection saturation, 5xx rate spike, homepage not **200**.
* Nightly job to flag **log ERROR spikes** vs. 7‑day baseline.

---

# Reporting Pack (Monthly)

* Availability vs. SLO, excluding approved maintenance.
* Top 5 slow operations (web actions/JQL/reports).
* Incident summary and MTTR.
* Capacity trend: CPU, heap, DB size, index size.

---

# Known Pitfalls

* Skipping **restore tests** — backups untested are backups you don’t have.
* Letting **logs grow** without rotation — fills disk, crashes app/DB.
* **Foreground reindex** during peak hours — causes user pain; schedule off‑hours.
* **Unvetted plugin updates** — test first; pin versions.
* **Proxy header drift** — breaks Base URL/attachments; monitor headers after proxy updates.

---

# Variables Used on This Page

`[Base URL]`, `[Jira Home Path]`, `[SMTP Host]`, `[Cert Authority/Team]`.

# Import Notes

* Keep the **metadata code block** intact for portability.
* Tables and code fences import cleanly to Confluence; convert relative links post‑import.
* Replace placeholders consistently across environments for clarity.
