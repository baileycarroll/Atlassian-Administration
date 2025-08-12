---
title: PostgreSQL Operations
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod, DR]
owner_role: [DBA/Platform Team]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, postgres, ha, backup, dr]
---

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for PostgreSQL operations supporting Jira Data Center v10+. *Ref: `[Change/Ticket ID]`*

# Purpose

Operate PostgreSQL safely and predictably for Jira: **provisioning, HA/replication, backups & PITR, maintenance, upgrades**, and **restore verification**. Use this as the DBA runbook and as the baseline for change reviews.

---

# 1) Supported Versions & Compatibility

| Component   | Target                      | Notes                                                  |
| ----------- | --------------------------- | ------------------------------------------------------ |
| PostgreSQL  | `[PG Version]` (e.g., 15.x) | Must be supported by Jira `[Jira Version]`             |
| JDBC Driver | `[JDBC Version]`            | Match major version to server; ship with Jira if newer |
| OS          | `[RHEL/Rocky/Ubuntu LTS]`   | Fully patched; tuned for I/O                           |

> Confirm against Atlassian’s support matrix before upgrades.

---

# 2) Roles & Responsibilities (DBA Scope)

| Area                 | Primary       | Backup        | Notes                                  |
| -------------------- | ------------- | ------------- | -------------------------------------- |
| Backups & PITR       | `[Team/Role]` | `[Team/Role]` | Base backups, WAL archiving, restores  |
| Replication/Failover | `[Team/Role]` | `[Team/Role]` | Patroni/repmgr/manual, promotion tests |
| Performance & Tuning | `[Team/Role]` | `[Team/Role]` | Query tuning, vacuum, bloat control    |
| Security & Patching  | `[Team/Role]` | `[Team/Role]` | Users/roles, TLS, minor/major updates  |

---

# 3) Standard Layout & Access

## 3.1 Filesystem Layout (recommendation)

```
/var/lib/pgsql/data        → PGDATA
/var/lib/pgsql/wal         → WAL (separate volume recommended)
/backups/postgres          → backup target (NFS/object) [if used]
```

## 3.2 Service & Users

* Service account: `[postgres]`
* DB owners: `[atlassian]` (app), `[replicator]` (replica), `[readonly]` (analytics)

## 3.3 Network Access

* `pg_hba.conf` allows **app hosts** and **DBA jump hosts** only.
* TLS for remote connections when traversing untrusted networks.

---

# 4) Provisioning Jira Databases & Roles

Create roles and databases (adjust names as needed):

```sql
-- Roles
CREATE ROLE "[atlassian]" LOGIN PASSWORD '[PG Password]';
CREATE ROLE "[replicator]" REPLICATION LOGIN PASSWORD '[Replica Password]';
CREATE ROLE "[readonly]" LOGIN PASSWORD '[RO Password]';

-- Databases
CREATE DATABASE "[jira]" OWNER "[atlassian]" ENCODING 'UTF8' TEMPLATE template0;
COMMENT ON DATABASE "[jira]" IS 'Jira application database';
```

`pg_hba.conf` example:

```
# TYPE  DATABASE  USER            ADDRESS           METHOD
host    [jira]    [atlassian]     [App Host CIDR]   scram-sha-256
host    replication [replicator]  [Replica CIDR]    scram-sha-256
```

Core settings (`postgresql.conf`) starting points:

```
max_connections = 200                  # or lower if using PgBouncer
shared_buffers  = 25% of RAM           # DB server RAM
effective_cache_size = 65% of RAM
work_mem = 16MB                        # per sort/hash; tune cautiously
maintenance_work_mem = 512MB
wal_level = replica
max_wal_size = 8GB
checkpoint_timeout = 15min
autovacuum = on
```

---

# 5) Connection Pooling

**Recommendation**: Use **PgBouncer** in **session mode** for Jira.

| Setting             | Suggested | Notes                             |
| ------------------- | --------- | --------------------------------- |
| `max_client_conn`   | `[400]`   | Based on user load                |
| `default_pool_size` | `[100]`   | Match DB `max_connections` budget |
| `pool_mode`         | `session` | Jira requires stable sessions     |

Update Jira’s `dbconfig.xml` to point at PgBouncer (`[pool_host]:6432`).

---

# 6) High Availability & Replication

## 6.1 Topology Options

| Option                           | Summary                            | RPO/RTO           | Complexity |
| -------------------------------- | ---------------------------------- | ----------------- | ---------- |
| Single Primary only              | No replicas                        | RPO ≥ last backup | Low        |
| Primary + Standby (streaming)    | Hot/warm standby via WAL streaming | RPO minutes       | Medium     |
| Managed cluster (Patroni/repmgr) | Automated failover & health checks | RPO≈0‑mins        | Higher     |

## 6.2 Streaming Replication (Manual Pattern)

On **primary**:

```sql
CREATE ROLE "[replicator]" REPLICATION LOGIN PASSWORD '[Replica Password]';
SELECT pg_create_physical_replication_slot('jira_slot');
```

Take a **base backup** for the replica (one of):

```bash
# pg_basebackup (built-in)
pg_basebackup -h [primary_host] -U replicator -D /var/lib/pgsql/data -R -C -S jira_slot -X stream
```

Replica `postgresql.auto.conf` will include primary conn info. Start replica and verify:

```sql
SELECT now() - pg_last_xact_replay_timestamp() AS replica_lag;
```

> Use **replication slots** to prevent WAL loss. Monitor slot lag and disk usage.

## 6.3 Promotion / Failover

Planned:

```bash
-- Stop Jira app (to avoid split‑brain writes)
-- Promote standby
pg_ctl -D /var/lib/pgsql/data promote
-- Repoint Jira/PgBouncer to new primary; run smoke tests
```

Unplanned: Follow DR runbook (Patroni/repmgr users: `patronictl` / `repmgr standby promote`).

---

# 7) Backups & PITR

## 7.1 Strategy

* **Base backups**: nightly (or as policy dictates) via `[Barman|pgBackRest|pg_basebackup]` to `[Backup Target]`.
* **WAL archiving**: continuous streaming/archiving for **Point‑In‑Time Recovery**.
* **Encryption**: at rest (backup target) and in transit.
* **Retention**: e.g., `full=7 days`, `WAL=7 days`, `monthly=12` (adjust per policy).

## 7.2 Tool Examples

**Barman** (snippet):

```ini
[barman]
configuration_files_directory = /etc/barman.d

[jira]
host = [primary_host]
user = barman
backup_method = postgres
streaming_archiver = on
retention_policy = RECOVERY WINDOW OF 7 DAYS
```

**pgBackRest** (snippet):

```ini
[global]
repo1-path=/backups/postgres
repo1-retention-full=7

[jira]
pgt1-host=[primary_host]
pgt1-path=/var/lib/pgsql/data
```

## 7.3 Verifying Backups

* List last backups and WAL continuity.
* Perform **regular restore tests** (see section 9).

---

# 8) Maintenance: VACUUM/ANALYZE/REINDEX

## 8.1 Autovacuum

Tune thresholds for Jira’s busiest tables (examples):

```sql
ALTER TABLE public.jiraissue SET (autovacuum_vacuum_scale_factor = 0.1);
ALTER TABLE public.changeitem SET (autovacuum_vacuum_scale_factor = 0.1);
ALTER TABLE public.worklog SET (autovacuum_vacuum_scale_factor = 0.2);
```

## 8.2 Analyze

Run weekly (or auto‑analyze):

```sql
ANALYZE;
```

## 8.3 Reindex

Quarterly or if bloat detected:

```sql
REINDEX INDEX CONCURRENTLY IF EXISTS public.issue_pkey;  -- example
```

> Always take a fresh backup before large maintenance.

---

# 9) Restore & PITR — Verification Checklist

**Objective**: Prove that backups are usable by restoring to an isolated host and starting Jira against it.

1. **Prepare target** host with same PG version.
2. **Restore base backup** using your tool of choice.
3. **Replay WAL** to a recovery point (timestamp/LSN).
4. Start PostgreSQL in recovery; verify `pg_is_in_recovery()`.
5. Run integrity checks (connect, counts on key tables).
6. Point a **test Jira** at the restored DB (`dbconfig.xml`).
7. Login, browse projects, run JQL; verify attachments where possible.
8. Capture timings and results; attach to `[change_ref]`.

**Smoke Queries**

```sql
SELECT count(*) FROM project;
SELECT count(*) FROM jiraissue;
SELECT max(updated) FROM jiraissue;
```

---

# 10) Upgrades — Minor vs Major

## 10.1 Minor (e.g., 15.3 → 15.4)

* Schedule maintenance.
* Backup DB & Jira home.
* Apply packages, restart PostgreSQL.
* Run Jira smoke tests.

## 10.2 Major (e.g., 14 → 15)

* Read release notes; confirm Jira compatibility.
* Choose method: **`pg_upgrade`** (in‑place, fast) or **logical dump/restore** (slow, clean).
* Take full backup + replica snapshot.
* If using `pg_upgrade`:

  ```bash
  pg_upgrade -b /usr/pgsql-14/bin -B /usr/pgsql-15/bin -d /var/lib/pgsql/14/data -D /var/lib/pgsql/15/data -U postgres
  ```
* Rebuild invalid indexes; `ANALYZE`.
* Repoint replicas; refresh replication slots.

> Always validate in **Test** first and record timings.

---

# 11) Performance Diagnostics — Quick Kit

```sql
-- Active sessions
SELECT pid, usename, state, wait_event_type, wait_event, query
FROM pg_stat_activity ORDER BY state, query_start NULLS LAST;

-- Locks
SELECT * FROM pg_locks pl JOIN pg_stat_activity a USING (pid) WHERE NOT granted;

-- Slow query log review (if enabled)
-- check postgres log files via your log platform or on-host
```

Linux/host checks:

```bash
vmstat 1 5; iostat -xz 1 5; sar -n DEV 1 5
```

---

# 12) Monitoring & Alerts (Reference)

| Metric              | Threshold        | Action                                              |
| ------------------- | ---------------- | --------------------------------------------------- |
| Replication lag     | > `[60s]`        | Investigate network/WAL; consider throttling writes |
| Connections used    | > `[80%]` of max | Increase pool or tune Jira/pgbouncer                |
| Disk fullness (WAL) | > `[85%]`        | Prune, expand, or fix slots                         |
| Checkpoint warning  | frequent         | Increase `max_wal_size`, tune `checkpoint_timeout`  |
| Autovacuum lag      | `n/a`            | Adjust scale factors, work mem                      |

Integrate with your platform (Prometheus, Datadog, etc.).

---

# 13) Security & Compliance

* Use **SCRAM‑SHA‑256** authentication; disable `trust`.
* Enforce TLS for remote DB connections when required.
* Rotate passwords/keys per policy; store secrets in `[Secrets Manager]`.
* Limit superuser access; use role‑based grants.
* Retain logs per `[Retention Policy]`.

---

# 14) Change Management Hooks

* Every DB change ties to `[change_ref]` with **plan**, **rollback**, and **verification** steps.
* Maintenance executed **outside business hours** unless emergency.
* Post‑change: attach metrics (lag, CPU, errors) and smoke results.

---

# Variables Used on This Page

`[PG Version]`, `[Jira Version]`, `[JDBC Version]`, `[RHEL/Rocky/Ubuntu LTS]`, `[postgres]`, `[atlassian]`, `[replicator]`, `[readonly]`, `[PG Password]`, `[Replica Password]`, `[RO Password]`, `[App Host CIDR]`, `[Replica CIDR]`, `[Backup Target]`, `[Barman|pgBackRest|pg_basebackup]`, `[pool_host]`, `[change_ref]`, `[Secrets Manager]`, `[Retention Policy]`.

# Import Notes

* Keep the **metadata code block** intact for portability.
* Tables and code fences import well into Confluence.
* Replace placeholders consistently; keep SQL and shell examples minimal and auditable.
