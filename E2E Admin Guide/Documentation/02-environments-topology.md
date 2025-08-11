---
title: Environments & Topology
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
  - topology
  - network
---
-
# Version Control Notes

* **2025-08-11** — Initial generalized draft for environment definitions, node topology, and networking.

# Purpose

This section defines the **\[Environment Name]** tiers, node counts and roles, `[Network Zone]` segmentation, load balancer configuration, DNS and certificates, shared storage paths, and the cluster indexing model.

# Environment Tiers

| Environment (`[Environment Name]`) | Purpose                            | Node Count | Network Zone     | Base URL                          |
| ---------------------------------- | ---------------------------------- | ---------: | ---------------- | --------------------------------- |
| Dev                                | Development and feature validation |       \[#] | `[Network Zone]` | `https://jira-dev.[Domain Name]`  |
| Test                               | Pre‑production testing and staging |       \[#] | `[Network Zone]` | `https://jira-test.[Domain Name]` |
| Prod                               | Production user‑facing environment |       \[#] | `[Network Zone]` | `[Jira Base URL]`                 |

> Adjust node counts and roles per organizational capacity and HA requirements.

# Node Roles

* **Application Nodes** — Run Jira application services, handle HTTP(S) traffic via `[LB VIP]`.
* **Indexer Role** — Each node maintains a local Lucene index at `[Index Path]` with cluster replication.
* **Database Server(s)** — PostgreSQL primary at `[PG Host]:[PG Port]` with optional replicas.
* **Shared Storage** — Mounted at `[Shared Storage Path]` on all app nodes (read/write).

# Network & Port Policy

| Source             | Destination    | Port/Protocol   | Purpose                              |
| ------------------ | -------------- | --------------- | ------------------------------------ |
| `[LB VIP]`         | App Nodes      | 8080/TCP (HTTP) | Application traffic                  |
| `[LB VIP]`         | App Nodes      | 443/TCP (HTTPS) | Application traffic (TLS terminated) |
| App Nodes          | `[PG Host]`    | 5432/TCP        | PostgreSQL                           |
| Admin Workstations | App Nodes      | 22/TCP (SSH)    | Administrative access                |
| Monitoring         | App Nodes / DB | 9100/TCP        | Metrics scraping (Node Exporter)     |

> Update for your `[Network Zone]` firewall rules; restrict access to required systems only.

# Load Balancer Configuration

* **Virtual IP**: `[LB VIP]`
* **Session Affinity**: Enabled (cookie‑based)
* **Health Checks**: `/status` endpoint on each node
* **TLS Termination**: At LB or proxy per security policy; re‑encrypt to nodes as needed.

# DNS & Certificates

* **DNS entries** for each `[Environment Name]` base URL.
* **Certificates** stored at `[TLS Cert Path]` and renewed ≥30 days before expiry.
* Use SANs if covering multiple environments under one certificate.

# Shared Home & Indexing Model

* **Shared Home Path**: `[Shared Storage Path]`

  * Stores attachments, avatars, plugin data, cluster metadata.
  * Must be on resilient, low‑latency storage.
* **Index Replication**: Each node keeps a local index at `[Index Path]`; cluster syncs changes.
* **Index Snapshot**: Periodically stored in Shared Home for new node bootstrap.

# Topology Diagram (ASCII)

```
          Users
            │
       [LB VIP]
        /    \
 [jira-app-01]  [jira-app-02] ... [jira-app-N]
      │     │              │
   [Index Path] (per node)
      │     │              │
         └──── Shared Home ────> [Shared Storage Path]
                       │
                   PostgreSQL
              [PG Host]:[PG Port]
```

# Known Pitfalls

* Missing session stickiness can cause login issues and failed admin operations.
* Slow or unreliable shared storage can stall indexing and degrade performance.
* Firewall misconfiguration may block inter‑node or DB connections.
* Certificates not renewed before expiry can cause outages.

# Key Variables Used on This Page

`[Environment Name]`, `[Domain Name]`, `[Jira Base URL]`, `[LB VIP]`, `[Network Zone]`, `[Server Hostname]`, `[Shared Storage Path]`, `[Index Path]`, `[PG Host]`, `[PG Port]`, `[TLS Cert Path]`.

# Cross‑References

* **Introduction & System Overview** → `./01-introduction-system-overview.md`
* **Pre‑Installation Requirements** → `./03-pre-installation-requirements.md`
* **Security Model** → `./12-security-model.md`

# Import Notes

* Preserve YAML front‑matter on import to Confluence.
* Replace relative links with Confluence page links after import.
* ASCII diagram can be swapped for uploaded image in `_assets/diagrams/`.
