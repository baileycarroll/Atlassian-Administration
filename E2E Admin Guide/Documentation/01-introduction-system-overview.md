---
title: Introduction & System Overview
product: Jira Data Center
version_target: 10.x
envs: All
owner_role: Atlassian Administrator
last_verified: 2025-08-11
change_ref: N/A
compliance_context: N/A
tags:
  - jira
  - admin-guide
  - postgres
---

# Version Control Notes

* **2025-08-11** — Initial draft of Overview page; creates canonical landing page and navigation pointers.

# Purpose & Scope
This guide enables administrators at **\[Company Name]** to deploy, operate, and evolve **Jira Data Center v10+** on on‑premises infrastructure backed by PostgreSQL. It is written as a reusable template: **all organization‑specific values appear as bracketed variables** (e.g., `[LB VIP]`, `[Shared Storage Path]`, `[PG Host]`). Replace these throughout when instantiating the guide for a given environment.

# Audience

System administrators, platform/SRE teams, and technical leads who are **new to this Jira deployment** but have strong general sysadmin knowledge.

# What “Data Center” Adds

Data Center adds active‑active clustering, zero‑downtime maintenance options, and scale‑out capacity compared to Server. Core concepts for this deployment:

* **Clustered app nodes** behind a load balancer at `[LB VIP]` with session affinity.
* **Shared home** at `[Shared Storage Path]` for attachments and cluster metadata.
* **PostgreSQL** at `[PG Host]:[PG Port]` / database `[PG Database]` (user `[PG User]`).
* **Search/index** is local to each node with **cluster replication** and a **shared index snapshot** under `[Index Path]`.
* **External services**: identity provider `[IdP Name]`, SMTP relay, reverse proxy/TLS, monitoring `[Monitoring Tool]`.

# Supported Scope (Replace per Org)

| Area          | Baseline in this Guide                                          |
| ------------- | --------------------------------------------------------------- |
| Jira          | Jira Software/Data Center **10.x**                              |
| OS/JVM        | Enterprise Linux (systemd), Temurin/Oracle JDK 17+              |
| Database      | PostgreSQL 15+ (primary + optional read replica)                |
| Load Balancer | \[Vendor/Pattern] with cookie‑based stickiness                  |
| AuthN/AuthZ   | LDAP/SCIM, SAML SSO via `[IdP Name]`                            |
| Observability | `[Monitoring Tool]` (metrics), logs shipped to `[Log Platform]` |

> Replace entries with your approved platform matrix during instantiation.

# High‑Level Architecture

```
           Users
             │
        [LB VIP]
          /   \
  [jira-app-01] [jira-app-02] ... [jira-app-N]
       │    │             │
   [Index Path] (per node)
       │    │             │
          └─── Shared Home ──> [Shared Storage Path]
                       │
                  PostgreSQL
                  [PG Host]:[PG Port]
```

**Key Characteristics**

* **Stateless requests** at the LB; **sticky sessions** recommended for admin and long‑running operations.
* **Shared home** mounted **read‑write** on all nodes; place on resilient storage.
* **Backups** must cover **DB + Shared Home** and any customization in app homes.

# Environments & Topology (At a Glance)

| Environment (`[Environment Name]`) | Node Count | Network Zone     | Base URL                          |
| ---------------------------------- | ---------: | ---------------- | --------------------------------- |
| Dev                                |       \[#] | `[Network Zone]` | `https://jira-dev.[Domain Name]`  |
| Test                               |       \[#] | `[Network Zone]` | `https://jira-test.[Domain Name]` |
| Prod                               |       \[#] | `[Network Zone]` | `[Jira Base URL]`                 |

> Detailed ports, firewall rules, and node roles live in **[Environments & Topology](./02-environments-topology.md)**.

# How to Use This Guide

1. **Read this page** to understand scope, components, and conventions.
2. **Provision & install** using **[Pre‑Installation Requirements](./03-pre-installation-requirements.md)** and **[Installation & Initial Configuration](./04-installation-initial-configuration.md)**.
3. **Integrate identity & access** via **[User & Role Management](./05-user-role-management.md)** and **[Security Model](./12-security-model.md)**.
4. **Operationalize** with **[Routine Maintenance](./06-routine-maintenance.md)** and **[Observability & SLOs](./07-observability-slos.md)**.
5. **Database care** is centralized in **[PostgreSQL Operations](./08-postgresql-operations.md)**.
6. **Plan changes and upgrades** with **[Change Management](./10-change-management-process.md)** and **[Upgrade/Migration Playbooks](./14-upgrade-migration-playbooks.md)**.

# Conventions & Variables

* **Variables** are written like `[Placeholder]` and must be replaced consistently; see **[Variable Glossary](./17-variable-glossary.md)**.
* **Code blocks** contain shell, SQL, or config that you can copy/paste after substituting variables.
* **Tables** enumerate tunables (JVM, DB, LB, storage). Keep them as the source of truth.
* **Diagrams** are provided as ASCII for portability; drop images in `_assets/diagrams/` if you have them.

# Core Integrations

* **Identity & Access**: LDAP sync and SAML SSO via `[IdP Name]` (just‑in‑time provisioning optional).
* **Email**: Outbound via `[SMTP Relay]`; inbound mail handlers optional.
* **TLS/Reverse Proxy**: Terminate at `[Proxy/LB]`; re‑encrypt to nodes as per security policy. Certs at `[TLS Cert Path]`.
* **Dev Tooling (optional)**: Bitbucket/Bamboo/Artifactory via application links.
* **Observability**: Metrics/alerts in `[Monitoring Tool]`, logs forwarded to `[Log Platform]`, tracing if available.

# SLOs at a Glance (Sample Defaults — replace per org)

* **Availability**: 99.9% monthly for core endpoints; error budget ≈ 43.2 min/month.
* **Performance**: p95 page render < 2.5s; p99 < 5s.
* **Backups/DR**: Nightly full + hourly WAL; **RPO ≤ 1h**, **RTO ≤ 1h** single‑node.
* **Security Hygiene**: Critical CVEs patched ≤ 7 days; TLS rotated ≥ 30 days pre‑expiry.

Details, dashboards, and alert thresholds: **[Observability & SLOs](./07-observability-slos.md)**.

# Roles & Responsibilities (Customize)

| Area                | Primary           | Backup        | Notes                                  |
| ------------------- | ----------------- | ------------- | -------------------------------------- |
| Platform ownership  | `[Team/Role]`     | `[Team/Role]` | Budget, roadmap, capacity              |
| On‑call / incidents | `[Team/Role]`     | `[Team/Role]` | See paging policy in Observability     |
| Database            | `[DBA Team]`      | `[DBA Team]`  | Backups, upgrades, performance         |
| Change control      | `[Change System]` | —             | RFCs prefixed `[Change Ticket Prefix]` |

# Quick Start Paths

* **Greenfield install**: Start at [03 Pre‑Installation](./03-pre-installation-requirements.md), then [04 Installation](./04-installation-initial-configuration.md).
* **Existing cluster** (tuning/ops): Jump to [06 Maintenance](./06-routine-maintenance.md), [07 Observability](./07-observability-slos.md), and [11 Troubleshooting](./11-troubleshooting-diagnostics.md).

# Known Pitfalls (You’ll thank yourself later)

* **Missing session stickiness** causes admin screens and uploads to fail sporadically.
* **Backups of DB *or* shared home only** are insufficient for a consistent restore.
* **Shared home on slow storage** = indexing stalls and timeouts.
* **Unpinned plugin versions** break during upgrades; use the **[Add‑on Matrix](./13-addon-plugin-compatibility-matrix.md)**.

# Key Variables Used on This Page

`[Company Name]`, `[Environment Name]`, `[Domain Name]`, `[Jira Base URL]`, `[Server Hostname]`, `[LB VIP]`, `[Network Zone]`, `[Shared Storage Path]`, `[Index Path]`, `[JVM_HEAP_SIZE]`, `[PG Host]`, `[PG Port]`, `[PG Database]`, `[PG User]`, `[IdP Name]`, `[TLS Cert Path]`, `[Monitoring Tool]`, `[Log Platform]`, `[Change System]`, `[Change Ticket Prefix]`.

# Cross‑References

* **Environments & Topology** → `./02-environments-topology.md`
* **Pre‑Installation Requirements** → `./03-pre-installation-requirements.md`
* **Installation & Initial Configuration** → `./04-installation-initial-configuration.md`
* **Security Model** → `./12-security-model.md`
* **Variable Glossary** → `./17-variable-glossary.md`

# Import Notes

* Keep the YAML front‑matter intact; Confluence importers may ignore unknown keys but will retain the block if pasted verbatim.
* Relative links (e.g., `./02-environments-topology.md`) import cleanly to Confluence if converted to page links post‑import.
* ASCII diagrams render reliably; replace with images using the same headings to preserve anchors.
* Tables map cleanly; avoid nested tables or HTML.
