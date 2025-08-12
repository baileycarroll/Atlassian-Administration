---

```
title: Appendix & References
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, appendix, references]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized **Appendix & References** for the Jira Data Center v10+ admin guide. *Ref: `[Change/Ticket ID]`*

# Purpose

Provide a compact **one‑stop reference**: official docs index, quick command cookbook, PostgreSQL query snippets, file/port paths, support tools, acronyms, and cross‑links to other sections of this guide.

---

# A. Official Documentation Index (Fill URLs per org)

> Keep these links current; pin to a specific major/minor when your platform standardizes.

* **Atlassian**

  * Jira Data Center **Installation & Upgrade** — `[URL]`
  * **Supported Platforms / Support Matrix** — `[URL]`
  * **Jira Application Properties** (`jira-config.properties`) — `[URL]`
  * **Jira REST API** — `[URL]`
  * **JVM Tuning & GC Guidance** — `[URL]`
  * **Mail (SMTP/IMAP/POP) Configuration** — `[URL]`
* **PostgreSQL**

  * **Server Docs** (target major) — `[URL]`
  * **JDBC Driver Compatibility** — `[URL]`
  * **pg\_upgrade / logical dump & restore** — `[URL]`
* **Reverse Proxy**

  * **Apache HTTP Server** reference — `[URL]`
  * **nginx** reference (if applicable) — `[URL]`

---

# B. Ports & Paths Quick Reference

| Area                    | Default / Example                                             |
| ----------------------- | ------------------------------------------------------------- |
| Base URL → Proxy → Jira | `https://[Base URL]` → Apache/nginx → `http://127.0.0.1:8080` |
| Jira Install Dir        | `[ /opt/atlassian/jira ]`                                     |
| Jira Home Dir           | `[ /data/atlassian/jira-home ]`                               |
| Main Log                | `[Jira Home]/logs/atlassian-jira.log`                         |
| Tomcat Config           | `[Jira Install]/conf/server.xml`                              |
| JVM Config              | `[Jira Install]/bin/setenv.sh`                                |
| DB Config               | `[Jira Home]/dbconfig.xml`                                    |
| SMTP Relay              | `[SMTP Host]:[SMTP Port]`                                     |
| DB Host/Port            | `[PG Host]:[5432]`                                            |

---

# C. Command Cookbook (Copy‑Paste Friendly)

## C.1 System & Service

```bash
# Service lifecycle
sudo systemctl status jira
sudo systemctl stop jira && sudo systemctl start jira
sudo journalctl -fu jira

# Time & TZ
sudo timedatectl status
sudo timedatectl set-timezone [TZ]

# File descriptors
ulimit -n
sudo grep -R "LimitNOFILE" /etc/systemd/system/jira.service*
```

## C.2 Proxy/TLS

```bash
# Headers & status
curl -sSI https://[Base URL] | sed -n '1,15p'
# Certificate expiry (via OpenSSL)
openssl s_client -servername [Base URL] -connect [Base URL]:443 < /dev/null 2>/dev/null | openssl x509 -noout -enddate
```

## C.3 PostgreSQL

```bash
# Connectivity
psql -h [PG Host] -U [PG User] -d [PG Database] -c 'select 1'
# Sessions & waits
psql -c "SELECT pid, usename, state, wait_event_type, wait_event FROM pg_stat_activity ORDER BY state;"
# Replica lag (on replica)
psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS replica_lag;"
# Bloat/size (quick glance)
psql -c "SELECT schemaname, relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC LIMIT 20;"
```

## C.4 Jira Health

```bash
# GC lines & recent errors
grep -Ei "(gc|SEVERE|ERROR)" [Jira Home]/logs/atlassian-jira.log | tail -n 100
# Index status (UI path)
#  System → Indexing → Status: Online (manual check)
```

---

# D. Config Snippets (Reference)

## D.1 `setenv.sh` (baseline)

```bash
JVM_MINIMUM_MEMORY="[8g]"
JVM_MAXIMUM_MEMORY="[8g]"
JVM_SUPPORT_RECOMMENDED_ARGS="-Datlassian.plugins.enable.wait=300 -Duser.timezone=[TZ]"
```

## D.2 `server.xml` Connector (behind TLS proxy)

```xml
<Connector port="8080" protocol="HTTP/1.1"
  connectionTimeout="20000" redirectPort="8443"
  proxyName="[Base Host]" proxyPort="443"
  scheme="https" secure="true" />
```

## D.3 `dbconfig.xml` (PostgreSQL)

```xml
<jira-database-config>
  <name>defaultDS</name>
  <delegator-name>default</delegator-name>
  <database-type>postgres72</database-type>
  <jdbc-datasource>
    <url>jdbc:postgresql://[PG Host]:[PG Port]/[PG Database]</url>
    <driver-class>org.postgresql.Driver</driver-class>
    <username>[PG User]</username>
    <password>[PG Password]</password>
    <validation-query>select 1</validation-query>
    <pool-min-size>20</pool-min-size>
    <pool-max-size>100</pool-max-size>
    <pool-max-wait>30000</pool-max-wait>
  </jdbc-datasource>
</jira-database-config>
```

## D.4 Apache vhost (reverse proxy)

```apache
<VirtualHost *:443>
  ServerName [Base URL]
  SSLEngine on
  SSLCertificateFile    [ /path/to/cert.crt ]
  SSLCertificateKeyFile [ /path/to/key.key ]
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
  Header always set X-Frame-Options "DENY"
  Header always set X-Content-Type-Options "nosniff"
  ProxyPreserveHost On
  RequestHeader set X-Forwarded-Proto https
  RequestHeader set X-Forwarded-Port 443
  ProxyPass        /  http://127.0.0.1:8080/
  ProxyPassReverse /  http://127.0.0.1:8080/
</VirtualHost>
```

---

# E. SQL Snippets (Jira‑Centric)

> Use read‑only where possible. Test in non‑prod first.

```sql
-- Top projects by issue count
SELECT p.pkey, COUNT(*) AS issues
FROM jiraissue i JOIN project p ON p.id = i.project
GROUP BY p.pkey ORDER BY issues DESC LIMIT 10;

-- Recently updated issues (timestamps)
SELECT id, project, updated FROM jiraissue ORDER BY updated DESC LIMIT 50;

-- Users with many filters (potential cleanup)
SELECT authorname, COUNT(*) FROM searchrequest GROUP BY authorname ORDER BY COUNT(*) DESC LIMIT 20;
```

---

# F. Support Zips & Evidence

* **Generate Support Zip**: UI → **System → Troubleshooting and Support → Create Support Zip**.
* Include: logs, thread dumps (when asked), Node info, config files.
* Attach to: `[Change/Ticket ID]` or vendor case; redact secrets before sharing.

---

# G. Acronyms & Abbreviations

| Term        | Meaning                                            |
| ----------- | -------------------------------------------------- |
| **BH**      | Business Hours                                     |
| **DR/GR**   | Disaster Recovery / Global Redundancy              |
| **GC**      | Garbage Collection                                 |
| **JMX**     | Java Management Extensions                         |
| **LKG**     | Last Known Good                                    |
| **PIR**     | Post‑Implementation Review                         |
| **PITR**    | Point‑In‑Time Recovery                             |
| **RPO/RTO** | Recovery Point Objective / Recovery Time Objective |
| **SLI/SLO** | Service Level Indicator / Objective                |

---

# H. Cross‑References (Within This Guide)

* **03 Pre‑Installation Requirements** — environment readiness & gates
* **04 Installation & Initial Configuration** — base setup
* **06 Routine Maintenance** — cadences and standard procedures
* **07 Observability & SLOs** — metrics, alerts, burn rates
* **08 PostgreSQL Operations** — backups, replication, upgrades
* **10 Change Management Process** — RFCs, approvals, verification
* **11 Troubleshooting & Diagnostics** — on‑call quick paths
* **13 Add‑on Compatibility Matrix** — app governance
* **14 Upgrade & Migration Playbooks** — strategies & runbooks
* **17 Variable Glossary** — canonical placeholders used across the guide

---

# Variables Used on This Page

`[Base URL]`, `[Jira Home]`, `[Jira Install]`, `[SMTP Host]`, `[SMTP Port]`, `[TZ]`, `[PG Host]`, `[PG Port]`, `[PG Database]`, `[PG User]`, `[PG Password]`.

# Import Notes

* Keep the **metadata code block** intact; Confluence preserves fenced blocks.
* Replace placeholders consistently; where possible, centralize values in **§17 Variable Glossary** and link here.
* ASCII code blocks and tables import cleanly; widen table columns after import if they wrap excessively.
