---
title: Installation & Initial Configuration
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, postgres]
---


# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for Jira Data Center v10+ installation on on‑prem servers with PostgreSQL. *Ref: `[Change/Ticket ID]`*

# Purpose

Provide a **repeatable, minimal‑surprise** procedure to install Jira Data Center v10+, wire it to PostgreSQL, and apply baseline settings (proxy, base URL, mail, identity). Use this with the **03 Pre‑Installation Requirements** page as the gatekeeper.

> If anything here deviates from your standards, annotate the step and capture the delta in a local runbook.

# Read Me First (Inputs You Need)

Have these values ready before you begin:

| Key                      | Example / Placeholder               |
| ------------------------ | ----------------------------------- |
| Jira Version             | `[Jira Version]` (e.g., 10.3.0)     |
| Download URL or artifact | `[Download URL]`                    |
| Service user             | `[jira]`                            |
| Install dir              | `[ /opt/atlassian/jira ]`           |
| Jira home                | `[ /data/atlassian/jira-home ]`     |
| JDK path                 | `[ /usr/lib/jvm/temurin-17 ]`       |
| Timezone                 | `[TZ]` (e.g., US/Pacific)           |
| Base URL                 | `[https://jira.example.org]`        |
| Proxy/LB hostname        | `[proxy.example.org]`               |
| DB host/port             | `[db.example.org]:[5432]`           |
| DB name/user/pass        | `[jira] / [atlassian] / [********]` |
| SMTP relay/port          | `[smtp.example.org]:[25]`           |
| From/prefix              | `[jira@example.org]` / `[JIRA]`     |
| IdP                      | `[IdP Name]` (SAML/OIDC)            |

---

# Installation Overview

1. Prepare OS, accounts, paths.
2. Install JDK 17 and verify.
3. Lay down Jira binaries (tar.gz preferred for repeatability).
4. Set JVM and service limits.
5. Configure reverse proxy + TLS headers.
6. Create database and `dbconfig.xml`.
7. Start Jira, complete setup wizard (or do silent bootstrap).
8. Apply base URL, mail, and identity settings.
9. Run smoke tests and snapshot the baseline.

---

# 1) System Preparation

```bash
# Create service user and folders
sudo useradd -m -s /bin/bash [jira] || true
sudo mkdir -p [ /opt/atlassian/jira ] [ /data/atlassian/jira-home ]
sudo chown -R [jira]:[jira] /opt/atlassian /data/atlassian

# Set timezone and NTP
sudo timedatectl set-timezone [TZ]
sudo systemctl enable --now chronyd || sudo systemctl enable --now ntp

# File descriptor limits (systemd drop-in)
sudo mkdir -p /etc/systemd/system/jira.service.d
cat <<'EOF' | sudo tee /etc/systemd/system/jira.service.d/limits.conf
[Service]
LimitNOFILE=16384
EOF
```

> Why: predictable ownership/paths make backups, upgrades, and DR much simpler.

---

# 2) Install JDK 17

Install your approved JDK (Temurin/Oracle/etc.). Verify:

```bash
java -version
update-alternatives --display java || which java
```

Set `JAVA_HOME` for the Jira service (in `setenv.sh`, later).

---

# 3) Lay Down Jira Binaries

```bash
cd /tmp
curl -fLO "[Download URL]"   # e.g., atlassian-jira-software-[Jira Version].tar.gz
sudo tar -xzf atlassian-jira-*.tar.gz -C /opt/atlassian/
sudo ln -sfn /opt/atlassian/atlassian-jira-* [ /opt/atlassian/jira ]
sudo chown -R [jira]:[jira] /opt/atlassian
```

> Prefer the tarball method to keep installs idempotent and clear of OS package manager quirks.

---

# 4) JVM & Service Settings

Edit `[ /opt/atlassian/jira ]/bin/setenv.sh`:

```bash
# Heap sizing (start equal; tune after observing GC)
JVM_MINIMUM_MEMORY="[8g]"
JVM_MAXIMUM_MEMORY="[8g]"

# Recommended flags
JVM_SUPPORT_RECOMMENDED_ARGS="-Datlassian.plugins.enable.wait=300 -Duser.timezone=[TZ]"

# Optional GC (start with defaults unless you have a reason)
# CATALINA_OPTS+=" -XX:+UseG1GC"
```

> Why: predictable GC and timezone behavior prevents odd timestamp/locale bugs.

---

# 5) Reverse Proxy (Apache) & TLS Headers

**server.xml** (`[ /opt/atlassian/jira ]/conf/server.xml`) — set proxy attributes on the HTTPS connector Jira actually listens on (usually 8080):

```xml
<Connector
    port="8080"
    protocol="HTTP/1.1"
    connectionTimeout="20000"
    redirectPort="8443"
    proxyName="[proxy.example.org]"
    proxyPort="443"
    scheme="https"
    secure="true"
/>
```

**Apache vhost** (template):

```apache
<VirtualHost *:443>
  ServerName [proxy.example.org]
  SSLEngine on
  SSLCertificateFile    [ /path/to/cert.crt ]
  SSLCertificateKeyFile [ /path/to/key.key ]

  ProxyPreserveHost On
  RequestHeader set X-Forwarded-Proto https
  RequestHeader set X-Forwarded-Port 443
  ProxyPass        /  http://127.0.0.1:8080/
  ProxyPassReverse /  http://127.0.0.1:8080/

  # Optional: hard limits to protect Jira
  ProxyTimeout 120
  LimitRequestFieldSize 16380
</VirtualHost>
```

Reload Apache and ensure 8080 is localhost‑only.

---

# 6) Database Creation & Connectivity

Create database and user (example):

```sql
CREATE ROLE "[PG User]" LOGIN PASSWORD '[PG Password]';
CREATE DATABASE "[PG Database]" OWNER "[PG User]" ENCODING 'UTF8';
```

Allow app host in `pg_hba.conf`:

```
# TYPE  DATABASE      USER          ADDRESS            METHOD
host    [PG Database] [PG User]     [App CIDR]/32      scram-sha-256
```

Create Jira’s DB config file in **Jira home**:

**`[ /data/atlassian/jira-home ]/dbconfig.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
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

> Why: supplying `dbconfig.xml` lets you bypass the DB step in the setup wizard and keeps secrets out of screenshots.

---

# 7) First Start & Bootstrap

Create a **systemd** unit (template):

```ini
# /etc/systemd/system/jira.service
[Unit]
Description=Atlassian Jira
After=network.target

[Service]
Type=forking
User=[jira]
PIDFile=[/opt/atlassian/jira]/work/catalina.pid
ExecStart=[/opt/atlassian/jira]/bin/start-jira.sh
ExecStop=[/opt/atlassian/jira]/bin/stop-jira.sh
LimitNOFILE=16384
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Start and watch logs:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now jira
sudo journalctl -fu jira
```

Browse to `[Base URL]` and complete the setup wizard (or continue with silent config below).

---

# 8) Silent Bootstrap (Optional)

Pre‑seed **`jira-config.properties`** to skip certain UI steps:

```
# [Jira Home]/jira-config.properties
jira.baseurl = [https://jira.example.org]
jira.title = [Company Jira]
jira.import.id = 1
```

> You’ll still need to paste the license key and create the first admin.

---

# 9) Mail (Outgoing SMTP)

UI path: **Administration → System → Mail → Outgoing Mail**

Settings to capture:

* Host: `[SMTP Host]`
* Port: `[SMTP Port]`
* From: `[From Address]`
* Prefix: `[Subject Prefix]`
* Auth/TLS: `[None/STARTTLS/SSL]` as required by your relay

Send a test message and confirm delivery.

---

# 10) Base URL & General Settings

UI path: **Administration → System → General Configuration**

Set:

* **Base URL**: `[Base URL]`
* **Time zone**: `[TZ]`
* **Attachment size limits**: `[X MB]` if you have a policy

---

# 11) Identity Integration (Directory & SSO)

* Connect to **LDAP/AD** (read‑only) or SCIM.
* Configure **SSO** via SAML/OIDC using `[IdP Name]` metadata.
* Map IdP groups → Jira groups (e.g., `jira-users`, `jira-administrators`).

> Keep at least one **local sysadmin** account for break‑glass access.

---

# 12) Logging & Rotation

* Ensure application logs rotate (either via built‑in log4j or system logrotate).
* Capture **access log** at the reverse proxy for request tracing.

---

# 13) Indexing & Health

* Run a **foreground reindex** after initial setup.
* Check **System Info** for node uptime, heap usage, DB latency.

---

# 14) Seed Backups

* Take an initial **database backup** and **Jira home snapshot** post‑install.
* Store artifacts with the install `change_ref` for audit and rollback.

---

# 15) Smoke Tests (Done‑When)

| Check       | Command/Path               | Done‑When                                         |
| ----------- | -------------------------- | ------------------------------------------------- |
| Homepage    | `curl -I [Base URL]`       | Returns `200 OK` with expected headers            |
| Login       | UI                         | Local admin can log in                            |
| DB          | UI → System Info           | No connection errors; latency reasonable          |
| Mail        | UI → Test email            | Delivered                                         |
| Attachments | Create issue + attach file | Upload succeeds, preview works                    |
| SSO         | IdP login                  | Redirect flow works; user lands in expected group |

---

# 16) Rollback Notes

* Stop Jira, restore **DB** from snapshot/PITR, restore **Jira home** from snapshot.
* Revert **server.xml** and proxy vhost changes if needed.
* Reapply service unit and `setenv.sh` from version control.

---

# Appendix A — Hardening Hints

* Bind Jira to `127.0.0.1:8080`; expose only via reverse proxy.
* Set `X-Frame-Options`, `Content-Security-Policy`, and `X-Content-Type-Options` in Apache/nginx.
* Restrict admin endpoints by IP if policy allows.

# Appendix B — Quick Diagnostics

```bash
# Threads/FDs
ps -o pid,comm,etimes,rss,vsz -C java | sed -n '1,2p'
lsof -p $(pgrep -f catalina) | wc -l

# GC and heap (JMX or logs)
grep -i gc /opt/atlassian/jira/logs/atlassian-jira.log | tail -n 20

# Proxy headers
curl -k -I [Base URL]
```

---

# Import Notes

* Keep the **metadata code block** intact for portability (Confluence preserves fenced blocks).
* Replace bracketed placeholders systematically (search/replace works well).
* Diagrams and code fences import cleanly; convert relative links after import.
