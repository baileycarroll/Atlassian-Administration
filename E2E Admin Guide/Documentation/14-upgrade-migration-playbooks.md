---

```
title: Upgrade & Migration Playbooks
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, upgrades, migrations, blue-green, canary]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized playbooks for Jira & PostgreSQL upgrades/migrations. *Ref: `[Change/Ticket ID]`*

# Purpose

Provide **repeatable, low‑risk** procedures for upgrading Jira (application & add‑ons) and PostgreSQL, including **blue/green**, **canary**, and **in‑place** paths. Each playbook includes **prechecks, backups, rollout, validation, and rollback**.

---

# Strategy Overview

| Strategy                   | When to Use                           |     Downtime | Complexity | Notes                                                          |
| -------------------------- | ------------------------------------- | -----------: | ---------: | -------------------------------------------------------------- |
| **In‑place (single‑node)** | Minor Jira updates; fast patching     | Short window |        Low | Stop → upgrade → start; simplest path                          |
| **Rolling (cluster)**      | Data Center cluster with LB           |    Near‑zero |        Med | Drain node → upgrade → rejoin → repeat                         |
| **Blue/Green**             | Major Jira upgrades; config refactors |     Optional |   Med‑High | Stand up Green, validate, switch traffic; easy rollback        |
| **Canary**                 | Risky changes with good telemetry     |    Near‑zero |        Med | Send **\[5–10%]** traffic to new version; abort if SLOs breach |
| **DB `pg_upgrade`**        | Major PG upgrade on same host         |          Low |        Med | Fast in‑place catalog upgrade; requires disk & downtime        |
| **Logical dump/restore**   | Cross‑platform or big jumps           |         High |       High | Clean move; long outage; great for defrag/cleanup              |

> Choose the simplest strategy that meets risk and SLO requirements.

---

# Global Preconditions (All Playbooks)

* **Section 03** (Pre‑Install) and **Section 10** (Change Process) complete.
* **Section 13** (Add‑on Matrix) reviewed; all apps **Data Center compatible** and pinned.
* **Backups ready**: DB base backup/PITR point; Jira home snapshot; proxy config copied.
* **Monitoring**: dashboards for availability, p95/p99, error rate; alert suppression during window.
* **Comms**: user notice drafted; on‑call briefed; status page ready.

---

# Playbook A — Jira Minor Upgrade (Single‑Node, In‑Place)

## A.1 Pre‑Checks

1. Confirm target **Jira version** `[x.y.z]` supports your **PostgreSQL** and **JDK 17**.
2. Verify free disk: `df -h` on install and home paths.
3. Export configs: `server.xml`, `setenv.sh`, `dbconfig.xml`, `jira-config.properties`.
4. Snapshot **DB** + **Jira home**; record IDs in the RFC.

## A.2 Execution (Maintenance Window)

1. Announce start; set status to Maintenance.
2. `systemctl stop jira`; wait for ports to close.
3. Lay down new binaries (tar):

   ```bash
   tar -xzf atlassian-jira-software-[x.y.z].tar.gz -C /opt/atlassian/
   ln -sfn /opt/atlassian/atlassian-jira-[x.y.z] /opt/atlassian/jira
   chown -R jira:jira /opt/atlassian/jira
   ```
4. Reapply `setenv.sh` and `server.xml` deltas.
5. Start Jira; tail logs: `journalctl -fu jira`.
6. Run **Smoke** (see §Validation).

## A.3 Validation (Done‑When)

* Homepage `200 OK`; admin login works.
* DB pool healthy; no errors in logs.
* Search returns fresh issues; attachments OK.
* Mail test delivered.

## A.4 Rollback

* `stop` → restore previous install symlink → restore **Jira home** snapshot if schema/plugins migrated → `start` → re‑run smoke.

---

# Playbook B — Jira Major Upgrade (Blue/Green)

> Works for single‑node or cluster. Green is a **parallel environment** running the target version.

## B.1 Prepare Green

1. Provision **Green** app host(s) and point to a **copy** of production data (refresh from latest backups).
2. Install target Jira `[x.y]` with matching **JDK 17**.
3. Import plugins at target versions per §13 matrix; reindex.
4. Validate end‑to‑end workflows with key projects; run performance sanity.

## B.2 Cutover Plan

1. Freeze changes on Blue (announce content/code freeze if needed).
2. Take final **DB** backup + **Jira home** sync; restore to Green.
3. Flip traffic: update **DNS/LB** from Blue to Green (TTL ≤ `[5m]`).
4. Monitor SLOs for **\[30–60] min**; keep Blue idle for fast rollback.

## B.3 Rollback (Switchback)

* Repoint DNS/LB back to Blue; announce rollback; create PIR; analyze Green failure.

---

# Playbook C — Jira Upgrade (Canary Release)

## C.1 Preconditions

* LB supports weighted routing; session stickiness honored.
* Telemetry in place for availability, p95, error rate per node.

## C.2 Steps

1. Upgrade **Canary node** to new version; keep others on current.
2. Route **\[5–10%]** traffic to canary for **\[N hours]**.
3. Abort thresholds: p95 > **\[target + 30%]** or 5xx > **\[1%]**.
4. If healthy, roll forward remaining node(s) or increase weight progressively.

---

# Playbook D — PostgreSQL Minor Upgrade

1. Schedule window; snapshot DB.
2. Apply minor packages (e.g., 15.3 → 15.4); restart PostgreSQL.
3. Run health checks: connections, replication, `ANALYZE`.
4. Run Jira smoke tests.

---

# Playbook E — PostgreSQL Major Upgrade (`pg_upgrade`)

## E.1 Preconditions

* Target PG `[major]` supported by Jira `[x.y]`.
* Disk space for **new cluster** + **old** side‑by‑side.
* Backups verified; replicas paused/removed.

## E.2 Steps (same host example)

```bash
# Stop Jira to eliminate writes
systemctl stop jira
# Stop old PG and prepare directories
systemctl stop postgresql
pg_upgrade \
  -b /usr/pgsql-[old]/bin \
  -B /usr/pgsql-[new]/bin \
  -d /var/lib/pgsql/[old]/data \
  -D /var/lib/pgsql/[new]/data \
  -U postgres
# Start new PG; run analyze script from pg_upgrade output
```

* Recreate replication slots; re‑establish replicas.
* Start Jira; run smoke; monitor.

## E.3 Rollback

* Stop new PG; start old cluster; restore WAL as needed; start Jira.

---

# Playbook F — PostgreSQL Migration (Logical Dump/Restore)

1. Quiesce writes (stop Jira or cut traffic).
2. `pg_dump` source → `pg_restore` target (with `--create --clean` as policy allows).
3. Recreate roles, extensions, and grants.
4. Point Jira to new DB (update `dbconfig.xml`); run smoke.

---

# Index Rebuild Strategy

* **Foreground Reindex** after major upgrades, plugin schema changes, or large data moves.
* Prefer low‑traffic windows; ensure backups first.
* Validate with JQL filters and recently updated issues.

---

# Add‑on (Plugin) Upgrade Batch — Safe Pattern

1. Review §13 matrix for each app; confirm DC compatibility.
2. **Dev → Test**: upgrade and validate app‑specific checks; reindex if required.
3. **Prod**: maintenance window; snapshot DB + Jira home; upgrade **one app at a time**.
4. Monitor logs for migration tasks; rollback to **LKG** if errors.

---

# Post‑Upgrade Validation (All Playbooks)

| Check        | Method               | Pass Criteria                  |
| ------------ | -------------------- | ------------------------------ |
| Availability | `curl -I [Base URL]` | `200 OK`, TLS valid            |
| Login/SSO    | UI                   | Admin login; IdP flow works    |
| DB Health    | UI/System Info       | Pool healthy; no errors        |
| Search       | JQL on recent issues | Results correct; no warnings   |
| Attachments  | Create/view          | Upload succeeds; preview works |
| Mail         | Test email           | Delivered via `[SMTP Host]`    |
| Jobs         | Scheduler            | No failing jobs in last 30 min |

* Watch p95/p99 and 5xx for **\[30–60] minutes**; compare to baseline.

---

# Rollback Triggers

* Error rate > **\[1%]** sustained for **\[10m]**.
* p95 latency > **\[target + 30%]** sustained for **\[15m]**.
* Critical feature broken (login, search, issue create).

> If any trigger hits, **abort and rollback** per the strategy’s section.

---

# DR/GR Coordination (If Applicable)

* Pause replication or ensure compatibility of catalog changes.
* Update replica after success; run a **sanity restore** from latest backups.
* Document GR cutover steps if a failover would occur during the upgrade window.

---

# Artifacts to Attach (Audit)

* Version numbers and binaries checksums.
* Config diffs (`server.xml`, `setenv.sh`, proxy vhost, `dbconfig.xml`).
* Backup/Snapshot IDs.
* Logs covering the window (app, DB, proxy).
* Screenshots of dashboards (SLOs) before/after.

---

# Known Pitfalls

* Skipping plugin compatibility checks → startup failures or data migration errors.
* Forgetting proxy headers after new install → redirect loops/broken attachments.
* `pg_upgrade` without enough disk → partial migrations; plan space.
* No index rebuild after big changes → stale search results.
* Canary without stickiness/telemetry → false signals or session loss.

---

# Variables Used on This Page

`[x.y.z]`, `[Base URL]`, `[SMTP Host]`, `[Jira Version]`, `[major]`, `[Change/Ticket ID]`.

# Import Notes

* Keep the **metadata code block** intact.
* Link to your **Change Management** (Section 10), **Add‑on Matrix** (Section 13), and **Observability & SLOs** (Section 7) for gating and evidence.
* Replace bracketed placeholders before use; keep commands minimal and copy‑pastable.
