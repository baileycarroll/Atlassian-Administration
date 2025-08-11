---
title: Pre‑Installation Requirements
product: Jira Data Center
version_target: 10.x
envs: 
owner_role: 
last_verified: 
change_ref: 
compliance_context: []
tags:
  - admin-guide
  - jira
  - postgres
---
# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for Jira Data Center v10+ on‑prem with PostgreSQL. *Ref: `[Change/Ticket ID]`*

# Purpose

Define **hardware, software, database, network, security, and operational prerequisites** that must be in place **before** installing Jira Data Center v10+. Use this as a gate: do not proceed to installation until every **Done‑When** item is met.

# Done‑When Checklist (Pre‑Install Gate)

* [ ] Approved **topology** and **environment scope** (Dev/Test/Prod, optional DR/GR) documented.
* [ ] Target **Jira version** `[Jira Version]` and **PostgreSQL version** `[PG Version]` align with Atlassian’s support matrix.
* [ ] **OS images** built and patched; SSH access verified; time sync (NTP) working.
* [ ] **Java 17 (LTS)** installed from `[JDK Vendor]`; `JAVA_HOME` set; `setenv.sh` writable.
* [ ] **Reverse proxy/TLS** decision made: `[Proxy Choice]` with certificate source `[Cert Authority/Team]`.
* [ ] **Database** provisioned with service accounts and connectivity from app host(s).
* [ ] **Filesystem layout** finalized; capacity and IOPS tested; `jira-home` path decided.
* [ ] **Email** relay reachable from app host(s); sender address/prefix approved.
* [ ] **Identity** integration chosen (LDAP/SAML/SCIM) and IdP contact confirmed.
* [ ] **Backups** and **PITR** tooling defined; initial restore test scheduled.
* [ ] **Change window** and **maintenance policy** approved; on‑call rota known.

---

# 1) Hardware & Capacity

## 1.1 Server Sizing (per Application Node)

| Resource         |     Baseline | Notes                                                        |
| ---------------- | -----------: | ------------------------------------------------------------ |
| vCPU             |     `[8‑16]` | Prioritize high clock speed; pin vCPU if noisy neighbor risk |
| RAM              | `[16‑64] GB` | Start with 2× heap + OS overhead; see JVM section            |
| Disk (OS)        |    `≥ 50 GB` | Separate from data where possible                            |
| Disk (jira‑home) | `≥ [200 GB]` | Attachments and indexes grow; plan for 12–24 months          |
| Disk (logs/tmp)  |    `≥ 50 GB` | Rotations and support bundles can spike                      |
| Network          |  `1–10 Gbps` | Low latency to DB host(s); avoid NAT where possible          |

> For **single‑node** deployments, sizing must include peak concurrency. For **clustered** deployments, repeat per node and ensure shared components meet aggregate throughput.

## 1.2 Storage Layout (Recommended)

```
/opt/atlassian/jira         → application install
/data/atlassian/jira-home   → Jira home (attachments, cache, plugins)
/var/log/jira               → logs (or keep inside jira-home/log)
/backups/jira               → app/home snapshots (if used)
```

* Use **XFS** or **ext4** with `noatime` for local disks. For network storage, require **low‑latency** (<2ms) and **IOPS** > `[3k]` sustained.
* Enable **NTP** and consistent **timezone** (e.g., `US/Pacific`) on all nodes.

## 1.3 File Descriptors & Kernel

* Set `nofile` (soft/hard) ≥ **`[16384]`** for the Jira service user.
* Kernel: ensure default `vm.max_map_count` and `fs.file‑max` support the descriptor target.

---

# 2) Operating System & Middleware

| Component     | Requirement                                                                 |
| ------------- | --------------------------------------------------------------------------- |
| OS            | Enterprise Linux with `systemd` (e.g., RHEL/Rocky/Ubuntu LTS) fully patched |
| Java          | **JDK 17** (Temurin/Adoptium, Oracle, or vendor‑approved)                   |
| Reverse Proxy | Apache or NGINX; HTTPS termination with HTTP→HTTPS redirect                 |
| Tools         | `curl`, `tar`, `gzip`, `unzip`, `rsync`, `openssl`, `netcat`, `lsof`        |
| Time          | NTP/chrony active; system timezone set to `[TZ]`                            |

**JVM Settings (initial)**

In `setenv.sh`:

```bash
JVM_MINIMUM_MEMORY="[8g]"
JVM_MAXIMUM_MEMORY="[8g]"
JVM_SUPPORT_RECOMMENDED_ARGS="-Datlassian.plugins.enable.wait=300 -Duser.timezone=[TZ]"
```

> Start with equal Xms/Xmx; measure GC before tuning.

---

# 3) Database (PostgreSQL) Prerequisites

## 3.1 Versions & Engine

* Target PostgreSQL: **`[PG Version]`** (ensure supported for Jira `[Jira Version]`).
* Optional connection pooler: **PgBouncer** in session mode (if many short‑lived connections).

## 3.2 Database Provisioning

| Item               | Value (replace)                                                                       |
| ------------------ | ------------------------------------------------------------------------------------- |
| DB Host            | `[PG Host]`                                                                           |
| Port               | `[PG Port]` (default 5432)                                                            |
| DB Names           | `[jira]`, `[atlassian]` (or single DB)                                                |
| DB Users           | `[atlassian]` (app), `[replicator]` (replication), `[read_only]` (optional analytics) |
| Encoding/Collation | `UTF8`, `C` or OS default (no case‑folding locales)                                   |

Create DB and role (example):

```sql
CREATE ROLE "[PG User]" LOGIN PASSWORD '[PG Password]';
CREATE DATABASE "[PG Database]" OWNER "[PG User]" ENCODING 'UTF8';
```

## 3.3 Required Parameters (PostgreSQL 15 Optimized)

| Parameter              | Production (High RAM) | Test/Dev (Lower RAM) | Why                              |
| ---------------------- | --------------------: | -------------------: | -------------------------------- |
| max\_connections       | `200`                 | `150`                | Jira + admin tools               |
| shared\_buffers        | `25% of RAM`          | `25% of RAM`         | Working set in memory            |
| effective\_cache\_size | `65% of RAM`          | `65% of RAM`         | Planner hint                     |
| work\_mem              | `64MB`                | `32MB`               | Per sort/hash; avoid OOM         |
| maintenance\_work\_mem | `1GB`                 | `512MB`              | VACUUM/CREATE INDEX              |
| wal\_level             | `replica`             | `replica`            | Required for backups/replication |
| max\_wal\_size         | `8GB`                 | `4GB`                | Reduce checkpoints               |
| checkpoint_timeout    | `15min`               | `10min`              | Balance latency vs. write amp    |
| checkpoint_completion_target | `0.9`        | `0.9`                | Spread checkpoint writes         |
| autovacuum             | `on`                  | `on`                 | Prevent bloat                    |
| autovacuum_max_workers | `3`                   | `2`                  | Limit concurrent autovacuum      |
| autovacuum_vacuum_scale_factor | `0.1`        | `0.1`                | More aggressive vacuum           |
| autovacuum_analyze_scale_factor | `0.05`     | `0.05`               | More frequent analyze            |
| random_page_cost       | `1.1`                 | `1.1`                | SSD optimization                 |
| effective_io_concurrency | `200`            | `100`                | SSD parallel I/O                 |
| max_parallel_workers   | `4`                   | `2`                  | Parallel query workers           |
| max_parallel_workers_per_gather | `2`     | `1`                  | Per-query parallelism            |
| log_min_duration_statement | `1000ms`     | `1000ms`             | Log slow queries                 |
| log_checkpoints        | `on`                  | `on`                 | Monitor checkpoint performance   |
| log_connections        | `on`                  | `on`                 | Security audit                   |
| log_disconnections     | `on`                  | `on`                 | Security audit                   |

> Apply via `postgresql.conf` and `pg_hba.conf`; restart or reload as required. Test environment values may need reduction if co-resident with application.

## 3.4 Network Access & Security

### pg_hba.conf Security Best Practices

**Critical Security Considerations:**
- **Host-Specific Access**: Restrict connections to specific application hosts only
- **User-Specific Permissions**: Grant access only to required databases per user role
- **SSL Enforcement**: Use `hostssl` for production connections when possible
- **Remove Global Access**: Never allow connections from `0.0.0.0/0` for application users
- **Replication Security**: Restrict replication connections to specific standby hosts

### Recommended pg_hba.conf Structure

```bash
# PostgreSQL 15 pg_hba.conf Template
# File: /etc/postgresql/15/main/pg_hba.conf

# DO NOT DISABLE!
# Database administrative login by Unix domain socket
local   all             postgres                                trust

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer

# IPv4 local connections:
host    all             all             127.0.0.1/32            md5

# Application connections (replace with actual hostnames)
host    jira            atlassian       [App Hostname]          md5
host    atlassian       atlassian       [App Hostname]          md5

# Analytics connections (restricted)
host    jira            data_analytics  [App Hostname]          md5
host    atlassian       data_analytics  [App Hostname]          md5

# Test environment (if co-resident)
host    jira            atlassian       127.0.0.1/32            md5
host    atlassian       atlassian       127.0.0.1/32            md5

# IPv6 local connections:
host    all             all             ::1/128                 md5

# Replication connections
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
host    replication     replicator      [Standby Hostname]      md5

# SSL connections (recommended for production)
# hostssl jira            atlassian       [App Hostname]          md5
# hostssl atlassian       atlassian       [App Hostname]          md5
```

### Security Migration Steps

1. **Backup Current Config**: `cp /etc/postgresql/[version]/main/pg_hba.conf /etc/postgresql/[version]/main/pg_hba.conf.backup`
2. **Test New Config**: Validate syntax before applying
3. **Gradual Rollout**: Apply to test environment first
4. **Monitor Connections**: Watch for connection failures after changes
5. **Reload Configuration**: `sudo systemctl reload postgresql` or `SELECT pg_reload_conf();`

* Limit `pg_hba.conf` to **app hosts** and **DBA jump hosts**.
* Enforce TLS for remote DB connections when traversing untrusted networks.

## 3.5 PostgreSQL 15 Configuration Template

### Production Environment Template

```bash
# PostgreSQL 15 postgresql.conf Template
# File: /etc/postgresql/15/main/postgresql.conf

# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 200

# Memory Settings (adjust based on available RAM)
shared_buffers = 25% of RAM
effective_cache_size = 65% of RAM
work_mem = 64MB
maintenance_work_mem = 1GB

# WAL Settings
wal_level = replica
max_wal_size = 8GB
min_wal_size = 80MB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

# Autovacuum Settings
autovacuum = on
autovacuum_max_workers = 3
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.05

# Performance Tuning
random_page_cost = 1.1
effective_io_concurrency = 200
max_parallel_workers = 4
max_parallel_workers_per_gather = 2

# Logging
log_min_duration_statement = 1000ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_line_prefix = '%m [%p] %q%u@%d '

# Replication
max_wal_senders = 10
wal_keep_size = 1GB

# Timezone
timezone = '[TZ]'
log_timezone = '[TZ]'

# SSL (configure with proper certificates)
ssl = on
ssl_cert_file = '[SSL Cert Path]'
ssl_key_file = '[SSL Key Path]'
```

### Test/Development Environment Template

```bash
# PostgreSQL 15 postgresql.conf Template (Test/Dev)
# File: /etc/postgresql/15/main/postgresql.conf

# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 150

# Memory Settings (reduced for co-resident applications)
shared_buffers = 25% of RAM
effective_cache_size = 65% of RAM
work_mem = 32MB
maintenance_work_mem = 512MB

# WAL Settings
wal_level = replica
max_wal_size = 4GB
min_wal_size = 40MB
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9

# Autovacuum Settings (reduced due to co-residence)
autovacuum = on
autovacuum_max_workers = 2
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.05

# Performance Tuning (reduced due to co-residence)
random_page_cost = 1.1
effective_io_concurrency = 100
max_parallel_workers = 2
max_parallel_workers_per_gather = 1

# Logging
log_min_duration_statement = 1000ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_line_prefix = '%m [%p] %q%u@%d '

# Replication
max_wal_senders = 5
wal_keep_size = 512MB

# Timezone
timezone = '[TZ]'
log_timezone = '[TZ]'

# SSL
ssl = on
ssl_cert_file = '[SSL Cert Path]'
ssl_key_file = '[SSL Key Path]'
```

---

# 4) Network, DNS, and TLS

## 4.1 DNS

* Create DNS records for each environment:

  * `[Jira Base URL]` → CNAME to `[App Hostname]` (or LB VIP)

## 4.2 Firewall & Ports (per environment)

**Application Host**

| Direction |   Port | Purpose                       |
| --------- | -----: | ----------------------------- |
| Inbound   |    443 | HTTPS → reverse proxy         |
| Inbound   |     80 | Optional HTTP → 301 redirect  |
| Inbound   |     22 | SSH from admin jump hosts     |
| Inbound   |   8080 | Localhost only (proxy → Jira) |
| Outbound  |   5432 | PostgreSQL to `[PG Host]`     |
| Outbound  |     25 | SMTP relay `[SMTP Host]`      |
| Outbound  | 53/123 | DNS/NTP                       |

**Database Host**

| Direction |   Port | Purpose                   |
| --------- | -----: | ------------------------- |
| Inbound   |   5432 | From app hosts only       |
| Inbound   |     22 | SSH from admin jump hosts |
| Outbound  |   5432 | To primary (if replica)   |
| Outbound  | 53/123 | DNS/NTP                   |

## 4.3 TLS Termination

* Terminate TLS at `[Proxy/LB]`; forward to Jira over `http://127.0.0.1:8080` (or upstream host) with required `X‑Forwarded‑*` headers.
* Certificates provided by `[Cert Team/CA]`. Record active paths in the metadata table.

---

# 5) Identity & Email

| Area         | Requirement                                                                                 |
| ------------ | ------------------------------------------------------------------------------------------- |
| Directory    | LDAP/AD read‑only bind account `[LDAP Bind DN]` with scoped search base                     |
| SSO          | SAML/OIDC via `[IdP Name]` (entityID, ACS URL, cert)                                        |
| Just‑in‑Time | Optional user provisioning (map IdP groups → Jira groups)                                   |
| SMTP         | Relay `[SMTP Host]:[SMTP Port]`; sender `[From Address]`; subject prefix `[Subject Prefix]` |

---

# 6) Backups, DR, and Restore Testing

## 6.1 Scope

* **Database**: Full + WAL (PITR) via `[Tool: Barman/pgBackRest]` to `[Backup Target]`.
* **Jira Home**: File‑level backups and rsync/snapshots to `[Backup Target]`.
* **Configs**: Capture `server.xml`, `setenv.sh`, proxy vhost, and any custom scripts.

## 6.2 Objectives

| Metric | Target   |
| ------ | -------- |
| RPO    | `≤ [1h]` |
| RTO    | `≤ [1h]` |

## 6.3 Validation

* Quarterly **restore verification** to an isolated environment.
* Document **cutover/runbook** if a DR site or GR environment exists.

---

# 7) Security & Compliance

* **Access**: Principle of least privilege; sudoers defined for `[Jira User]`.
* **Hardening**: CIS or org baseline applied; only required packages installed.
* **Secrets**: Store DB and SAML secrets in `[Secrets Manager/Vault]`.
* **Logging**: System and application logs retained per policy; forward to `[Log Platform]` if available.
* **Compliance**: Align with `[Compliance Context]` (e.g., FERPA/HIPAA). Avoid PHI/PII in issues unless sanctioned.

---

# 8) Change Management & Maintenance

* **Windows**: `[Cadence]` maintenance outside `[Business Hours]`.
* **Approvals**: RFC in `[Change System]`; rollback plan attached.
* **Comms**: Notify via `[Channel]` `[X]` days prior; update status page.

---

# 9) Installation Inputs Worksheet (Fill Before Proceeding)

| Key                                            | Value             |
| ---------------------------------------------- | ----------------- |
| `[Company Name]`                               |                   |
| `[Environment Name]`                           | Dev / Test / Prod |
| `[App Hostname]`                               |                   |
| `[DB Hostname]`                                |                   |
| `[Jira Base URL]`                              |                   |
| `[Shared Storage Path]` (if clustered)         |                   |
| `[Jira Home Path]`                             |                   |
| `[JDK Vendor]`                                 |                   |
| `[JVM_HEAP_SIZE]`                              |                   |
| `[SMTP Host]:[SMTP Port]`                      |                   |
| `[From Address]` / `[Subject Prefix]`          |                   |
| `[IdP Name]` / metadata                        |                   |
| `[PG Version]` / `[PG Database]` / `[PG User]` |                   |
| `[Backup Tool]` / `[Backup Target]`            |                   |

---

# 10) Pre‑Flight Validation Commands

```bash
# OS basics
id jira || sudo useradd -m -s /bin/bash jira
sudo timedatectl set-timezone [TZ]
sudo systemctl enable --now chronyd || sudo systemctl enable --now ntp

# Java
java -version

# File descriptors
ulimit -n

# Paths & permissions
sudo mkdir -p /opt/atlassian/jira [Jira Home Path]
sudo chown -R jira:jira /opt/atlassian/jira [Jira Home Path]

# Network reachability
nc -vz [PG Host] [PG Port]
nc -vz [SMTP Host] [SMTP Port]

# PostgreSQL 15 validation
sudo -u postgres psql -c "SELECT version();"
sudo -u postgres psql -c "SHOW shared_buffers;"
sudo -u postgres psql -c "SHOW max_connections;"
sudo -u postgres psql -c "SHOW work_mem;"
sudo -u postgres psql -c "SHOW maintenance_work_mem;"
sudo -u postgres psql -c "SHOW effective_cache_size;"
sudo -u postgres psql -c "SHOW wal_level;"
sudo -u postgres psql -c "SHOW max_wal_size;"
sudo -u postgres psql -c "SHOW checkpoint_timeout;"
sudo -u postgres psql -c "SHOW autovacuum;"
sudo -u postgres psql -c "SHOW log_min_duration_statement;"
sudo -u postgres psql -c "SHOW timezone;"
sudo -u postgres psql -c "SHOW ssl;"

# pg_hba.conf validation
sudo -u postgres psql -c "SHOW hba_file;"
sudo -u postgres psql -c "SELECT pg_reload_conf();"
sudo -u postgres psql -c "SELECT * FROM pg_hba_file_rules WHERE database = 'jira';"
sudo -u postgres psql -c "SELECT * FROM pg_hba_file_rules WHERE user_name = 'atlassian';"

# Connection testing
psql -h [PG Host] -U [PG User] -d [PG Database] -c "SELECT current_database(), current_user;"

# Proxy headers (after proxy config)
curl -I https://[Jira Base URL]
```

---

# Known Pitfalls (Pre‑Install)

* **Skipping PITR setup** — Full backups alone are not enough; validate WAL archiving and restores.
* **Under‑sized heap or FD limits** — Leads to OOMs or file handle exhaustion during reindexing/attachments.
* **Slow storage for `jira-home`** — Causes indexing and search failures; verify IOPS.
* **Locale mismatches** — Non‑UTF8 DB or OS locales cause subtle errors; enforce UTF8.
* **Proxy misconfig** — Missing `X‑Forwarded‑Proto/Host` breaks redirects and base URL.
* **PostgreSQL security risks** — Global access patterns (`0.0.0.0/0`) in pg_hba.conf create security vulnerabilities. Restrict to specific hostnames.
* **PostgreSQL upgrade considerations** — Older configurations may have severely under-sized memory parameters. Plan for downtime during upgrade and configuration changes.
* **Co-resident applications** — When PostgreSQL shares a host with Jira, reduce memory and worker settings to prevent resource contention.

---

# Variables Used on This Page

`[Company Name]`, `[Environment Name]`, `[App Hostname]`, `[DB Hostname]`, `[Jira Base URL]`, `[Shared Storage Path]`, `[Jira Home Path]`, `[JDK Vendor]`, `[JVM_HEAP_SIZE]`, `[SMTP Host]`, `[SMTP Port]`, `[From Address]`, `[Subject Prefix]`, `[IdP Name]`, `[PG Version]`, `[PG Database]`, `[PG User]`, `[Backup Tool]`, `[Backup Target]`, `[Proxy Choice]`, `[Cert Authority/Team]`, `[TZ]`.

# Import Notes

* Keep the metadata code block intact; Confluence may drop unknown keys but preserves the block when pasted.
* Tables and code fences import cleanly. Convert relative links to page links post‑import.
* ASCII filesystem and network snippets are intentionally portable.
