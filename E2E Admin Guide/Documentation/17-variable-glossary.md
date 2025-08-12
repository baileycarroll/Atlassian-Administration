---

```
title: Variable Glossary
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod, DR]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, variables, glossary]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized **Variable Glossary** capturing canonical placeholders used across the guide. *Ref: `[Change/Ticket ID]`*

# Purpose

Provide a **single source of truth** for all **placeholders** used throughout this admin guide. Each variable includes a description, example, owner, and where it appears, so updates are **consistent** and **auditable**.

> Convention: Variables are written in **square brackets** like `[Example Variable]`. Replace globally with **search/replace** after agreeing on values.

---

# How to Use This Glossary

1. **Decide** the value with the relevant owner (e.g., DBA for DB vars, Platform for proxy/TLS).
2. **Update** the table below and record a reference in `change_ref`.
3. **Propagate** the change by replacing across documents (or inject via your templating tool).
4. **Re‑verify** impacted sections and bump `last_verified`.

---

# Naming Rules & Conventions

* Keep variable names **Descriptive Title Case** inside brackets: `[Jira Home Path]`.
* Use **one variable per concept**; avoid synonyms (e.g., prefer `[Jira Home Path]` over multiple “home path” variants).
* **Scope** variables when environment‑specific: `[Prod App Hostname]` vs `[App Hostname]`.
* For lists, keep **comma‑separated** values inside a single variable (e.g., `[Synthetic Location(s)]`).

---

# Master Variable Registry

| Variable                  | Description                                    | Example                                                    | Default/Options               | Owner             | Source of Truth  | Sections Using                     |
| ------------------------- | ---------------------------------------------- | ---------------------------------------------------------- | ----------------------------- | ----------------- | ---------------- | ---------------------------------- |
| `[Company Name]`          | Organization label used in titles and examples | `The University of Example`                                | —                             | Service Owner     | Brand/Legal      | 01, 02, 03, headers                |
| `[Environment Name]`      | Environment identifier                         | `Prod` / `Test` / `Dev` / `DR`                             | `{Dev, Test, Prod, DR}`       | Platform          | CMDB             | 02, 03, 04, 06                     |
| `[Network Zone]`          | Network segment/tag for hosts                  | `Campus‑Internal`                                          | —                             | NetOps            | Network docs     | 02                                 |
| `[LB VIP]`                | Load balancer VIP or DNS for cluster           | `lb-jira.example.org`                                      | —                             | NetOps            | LB config        | 02, 09, 14                         |
| `[Port Policy]`           | Statement of allowed ports per role            | `443 in; 5432 app→db`                                      | —                             | NetOps            | Firewall policy  | 02, 03                             |
| `[App Hostname]`          | Jira application host FQDN                     | `jira-app01.example.org`                                   | Per env                       | Platform          | DNS/CMDB         | 02, 03, 04, 06, 11                 |
| `[DB Hostname]`           | PostgreSQL host FQDN                           | `pg01.example.org`                                         | Per env                       | DBA               | DNS/CMDB         | 02, 03, 04, 06, 08, 11             |
| `[Jira Base URL]`         | Public URL for Jira                            | `https://jira.example.org`                                 | Per env                       | Platform          | Proxy/DNS        | 02, 03, 04, 06, 07, 10, 11, 14, 16 |
| `[Shared Storage Path]`   | Shared home for clustered nodes                | `/mnt/jira-shared`                                         | Optional                      | Platform          | Storage runbook  | 02, 04, 09, 14                     |
| `[Jira Home Path]`        | Jira data directory                            | `/data/atlassian/jira-home`                                | Required                      | Platform          | Server SOP       | 02, 03, 04, 06, 11, 16             |
| `[Jira Install Path]`     | Jira install directory                         | `/opt/atlassian/jira`                                      | Required                      | Platform          | Server SOP       | 04, 11, 16                         |
| `[JDK Vendor]`            | Approved Java 17 build                         | `Temurin 17`                                               | `{Temurin, Oracle, ...}`      | Platform          | Standards        | 03, 04, 16                         |
| `[JVM_HEAP_SIZE]`         | Xms/Xmx heap value                             | `8g`                                                       | `4g–32g`                      | Platform          | setenv.sh        | 03, 04, 06, 11, 16                 |
| `[JVM Flags]`             | Extra JVM args                                 | `-Datlassian.plugins.enable.wait=300 -Duser.timezone=[TZ]` | —                             | Platform          | setenv.sh        | 03, 04, 06, 16                     |
| `[File Descriptor Limit]` | Process open files limit                       | `16384`                                                    | ≥ `16384`                     | Platform          | systemd          | 03, 04, 06                         |
| `[Proxy Choice]`          | Reverse proxy technology                       | `Apache`                                                   | `{Apache, nginx, LB}`         | NetOps            | Proxy config     | 03, 04, 09, 12, 16                 |
| `[TLS Termination Point]` | Where TLS terminates                           | `Apache vhost`                                             | —                             | NetOps            | Proxy config     | 02, 03, 04, 12, 16                 |
| `[TLS Cert Path]`         | Active cert/key file paths                     | `/etc/pki/tls/...`                                         | —                             | PKI/NetOps        | Proxy `ssl.conf` | 02, 12, 16                         |
| `[Cert Authority/Team]`   | Certificate issuer/owner team                  | `Unix Engineering`                                         | —                             | PKI               | PKI portal       | 02, 03, 06, 12                     |
| `[SMTP Host]`             | Outbound mail relay host                       | `smtp.example.org`                                         | —                             | Messaging         | Mail docs        | 03, 04, 06, 10, 11, 15, 16         |
| `[SMTP Port]`             | Outbound mail port                             | `25`                                                       | `25/587/465`                  | Messaging         | Mail docs        | 03, 04, 06, 10, 11, 15, 16         |
| `[From Address]`          | Default mail sender                            | `jira@example.org`                                         | —                             | Platform          | App config       | 03, 04, 10, 15                     |
| `[Subject Prefix]`        | Mail subject prefix                            | `[JIRA]`                                                   | —                             | Platform          | App config       | 03, 04, 10, 15                     |
| `[IdP Name]`              | Identity Provider label                        | `Okta`                                                     | —                             | IAM               | IdP portal       | 03, 04, 05, 07, 10, 11, 12         |
| `[LDAP Bind DN]`          | Directory bind DN                              | `CN=jira,OU=svc,DC=example,DC=org`                         | —                             | IAM               | Directory        | 05, 12                             |
| `[PG Version]`            | PostgreSQL major/minor                         | `15.4`                                                     | Supported matrix              | DBA               | DBA SOP          | 03, 04, 08, 14, 16                 |
| `[JDBC Version]`          | JDBC driver version                            | `42.7.3`                                                   | Match PG major                | DBA/Platform      | Driver repo      | 08, 16                             |
| `[PG Host]`               | DB hostname (alias)                            | `pg01.example.org`                                         | Per env                       | DBA               | DNS/CMDB         | 03, 04, 08, 11, 16                 |
| `[PG Port]`               | DB port                                        | `5432`                                                     | —                             | DBA               | DB config        | 03, 04, 08, 11, 16                 |
| `[PG Database]`           | DB name for Jira                               | `jira`                                                     | —                             | DBA               | DB config        | 03, 04, 08, 11, 16                 |
| `[PG User]`               | App DB user                                    | `atlassian`                                                | —                             | DBA               | DB config        | 03, 04, 08, 11, 16                 |
| `[Replica Slot Name]`     | Physical replication slot                      | `jira_slot`                                                | —                             | DBA               | DB config        | 08                                 |
| `[Backup Tool]`           | DB backup technology                           | `Barman`                                                   | `{Barman, pgBackRest}`        | DBA               | Backup SOP       | 03, 06, 08, 14                     |
| `[Backup Target]`         | Backup destination                             | `nfs://backup01/...`                                       | —                             | DBA               | Backup SOP       | 03, 06, 08, 14                     |
| `[Backup Retention]`      | Retention policy                               | `7 days WAL, 12 monthly`                                   | —                             | DBA               | Backup SOP       | 06, 08, 16                         |
| `[Secrets Manager]`       | Secret storage system                          | `Vault`                                                    | —                             | Security          | Security SOP     | 06, 08, 12, 13, 16                 |
| `[Rotation Policy]`       | Secret rotation cadence                        | `90 days`                                                  | —                             | Security          | Policy doc       | 06, 08, 12, 13                     |
| `[SIEM/Log Platform]`     | Log aggregation                                | `Splunk`                                                   | —                             | Security/Platform | SIEM             | 07, 11, 12, 16                     |
| `[Compliance Context]`    | Applicable frameworks                          | `FERPA/HIPAA`                                              | —                             | Security          | GRC registry     | 01, 02, 12                         |
| `[Business Hours TZ]`     | Timezone label                                 | `US/Pacific`                                               | Olson TZ                      | Platform          | NTP/Policy       | 06, 07, 10, 15                     |
| `[TZ]`                    | System timezone                                | `US/Pacific`                                               | Olson TZ                      | Platform          | OS config        | 03, 04, 06, 07, 11, 16             |
| `[Cadence/Timing]`        | Maintenance schedule                           | `Bi‑weekly`                                                | —                             | Platform          | Ops calendar     | 06, 07, 10, 15                     |
| `[Synthetic Location(s)]` | Probe locations                                | `US‑West, US‑East`                                         | —                             | SRE               | Monitoring       | 07                                 |
| `[On‑Call Group]`         | Paging target                                  | `DevTools On‑Call`                                         | —                             | SRE               | On‑call tool     | 07, 11                             |
| `[On‑Call System]`        | Paging system                                  | `PagerDuty`                                                | `{PagerDuty, Opsgenie, SNOW}` | SRE               | Tooling          | 07, 11                             |
| `[Change/Ticket ID]`      | Change record reference                        | `CHG‑12345`                                                | —                             | Change Mgr        | Change tool      | All headers, 10, 14, 16            |
| `[X MB]`                  | Attachment size cap                            | `50 MB`                                                    | Policy                        | Platform          | App policy       | 04, 09, 12                         |
| `[Team/Role]`             | Owner placeholder                              | `Platform Team`                                            | —                             | —                 | —                | Many                               |

> Tip: For env‑specific values, either suffix the variable (e.g., `[Prod App Hostname]`) or maintain a **per‑environment override table** below.

---

# Environment Overrides (Optional)

If values differ by environment, capture them here for quick lookup.

| Variable                  | Dev | Test | Prod | DR/GR |
| ------------------------- | --- | ---- | ---- | ----- |
| `[App Hostname]`          |     |      |      |       |
| `[DB Hostname]`           |     |      |      |       |
| `[Jira Base URL]`         |     |      |      |       |
| `[Jira Home Path]`        |     |      |      |       |
| `[JVM_HEAP_SIZE]`         |     |      |      |       |
| `[SMTP Host]:[SMTP Port]` |     |      |      |       |
| `[PG Host]:[PG Port]`     |     |      |      |       |

---

# Update Workflow (Governance)

1. **Propose** change with reason and impact (link to tickets as needed).
2. **Review** with the variable owner team.
3. **Approve** and update this glossary (commit message includes `[Change/Ticket ID]`).
4. **Search/replace** in affected sections → run **spot checks**.
5. **Note** the update in **Version Control Notes** for the changed pages.

---

# Cross‑References

* **02 Environments & Topology** — where hostnames, DNS, and port policy are defined.
* **03 Pre‑Installation** — inputs worksheet that maps to many variables here.
* **04 Installation & Initial Config** — uses `[Jira Home Path]`, `[Jira Install Path]`, `server.xml`, `setenv.sh`.
* **06 Routine Maintenance** — uses `[SMTP Host]`, `[Base URL]`, schedules.
* **07 Observability & SLOs** — uses Business Hours, SLO targets, on‑call routing.
* **08 PostgreSQL Operations** — uses `[PG *]`, backups, replication.
* **10 Change Management** — uses `[Change/Ticket ID]`, freeze windows.
* **13 Add‑on Matrix** — uses app keys and LKG versions.
* **16 Appendix & References** — ports/paths/commands link back here for canonical values.

---

# Import Notes

* Keep the **metadata code block** intact for portability.
* When importing to Confluence, widen the **Master Variable Registry** table if columns wrap excessively.
* Prefer updating this page first, then propagating values across other sections to prevent drift.
