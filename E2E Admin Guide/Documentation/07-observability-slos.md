---

```
title: Observability & SLOs
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, observability, slos, monitoring]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for Observability & SLOs covering metrics, logs, synthetics, alerting, and reporting. *Ref: `[Change/Ticket ID]`*

# Purpose

Define a **measurable reliability contract** for Jira: what we watch (SLIs), the targets (SLOs), how we alert (policies), and how we learn (reports/runbooks). This page is tool‑agnostic; substitute your stack (e.g., Prometheus/Grafana, Datadog, New Relic, Splunk).

---

# 1) SLIs — What We Measure

| SLI                       | Scope        | Definition (high level)                                  | Collection Method                                          |
| ------------------------- | ------------ | -------------------------------------------------------- | ---------------------------------------------------------- |
| **Availability**          | Web UI       | p(Availability) over time; homepage returns **HTTP 200** | Synthetic probe(s) hitting `[Jira Base URL]/status` or `/` |
| **Latency**               | Web UI       | p95 page render time; p99 for critical paths             | RUM/Synthetics or app access logs                          |
| **Error Rate**            | Web UI/API   | 5xx and key 4xx ratio over requests                      | Reverse proxy logs or APM                                  |
| **Background Jobs**       | Scheduler    | Success rate for indexers, mail handlers, automation     | App logs / JMX / APM custom metrics                        |
| **DB Health**             | PostgreSQL   | Replication lag, connection saturation, slow queries     | pg                                                         |
| db exporter / SQL queries |              |                                                          |                                                            |
| **JVM Health**            | App node     | Heap usage %, GC pause time, thread count                | JMX exporter / APM agent                                   |
| **Storage**               | App/DB hosts | Disk fullness %, inode pressure, IOPS                    | Node exporter / OS agent                                   |

> Keep SLI definitions **stable**. If you change a definition, mark the date and reset the SLO calendar.

---

# 2) SLOs — Targets & Error Budgets

## 2.1 Availability (example)

* **Target**: **99.9%** monthly during **Business Hours** `[Mon–Fri, 06:00–18:00, [Business Hours TZ]]`.
* **Error Budget**: \~**43 minutes / 30‑day month** within the measured window.
* **Measurement**: Synthetic uptime checks from **\[Synthetic Location(s)]** request homepage and expect **HTTP 200** within **\[3s]**.
* **Exclusions**: Approved maintenance windows `[link/ref]`.

## 2.2 Latency (example)

* **Target**: p95 < **2.5s** and p99 < **5s** (Business Hours).
* **Paths**: `/secure/Dashboard.jspa`, `JQL search`, `/browse/*`.

## 2.3 Error Rate (example)

* **Target**: 5xx < **0.5%** of requests; 429/401 excluded if rate‑limited by policy.

## 2.4 Background Jobs

* **Target**: Scheduled jobs succeed **≥ 99%**; queue age < **\[5 min]**.

> Align SLOs with **product expectations**. Lower targets reduce noise; higher targets require budget to maintain.

---

# 3) Alerting Policy — Multi‑Window Burn Rates

Use **fast** and **slow** burn windows for availability/error‑budget alerts. Example policies below (tool agnostic; adapt math to your platform).

| Severity | Condition                                                                        | Intent                                |
| -------- | -------------------------------------------------------------------------------- | ------------------------------------- |
| **P1**   | 2‑hour burn rate > **14.4×** of monthly budget **and** 6‑hour burn rate > **6×** | Major outage; page immediately        |
| **P2**   | 24‑hour burn rate > **3×** of monthly budget                                     | Degraded over time; page non‑urgent   |
| **P3**   | 3‑day burn rate > **1×** of monthly budget                                       | Budget exhaustion risk; create ticket |

**Example (PromQL‑style) availability burn**

```
# Up is 1 for success, 0 for failure in your synthetic time series
burn_2h  = (1 - avg_over_time(up{job="jira_synth"}[2h])) / (1 - 0.999)  # denominator is (1 - SLO)
burn_6h  = (1 - avg_over_time(up{job="jira_synth"}[6h])) / (1 - 0.999)
alert P1 if burn_2h > 14.4 and burn_6h > 6
```

**Latency alerts** (p95): alert if p95 > target for **\[15m]** sustained during Business Hours.

**Error‑rate alerts**: alert if 5xx > **\[1%]** for **\[10m]** (proxy log metric).

---

# 4) Synthetic Monitoring

* Endpoints: `[Jira Base URL]/` (homepage), `[Jira Base URL]/rest/api/2/serverInfo` (API ping).
* Method: **GET** with **timeout** `[3s]`; validate **status 200** and **string match** `[e.g., "Atlassian Jira"]`.
* Locations: `[Region A]`, `[Region B]` (at least two to avoid false positives).
* Interval: **\[1m]** (prod), **\[5m]** (non‑prod).

> Record probe IP ranges and ensure they’re allowed through WAF/firewall if applicable.

---

# 5) Metrics & Dashboards (Reference Layout)

## 5.1 Executive Dashboard

* SLO status (availability, latency) and **remaining error budget**.
* Incidents in last **30/90** days; MTTR trends.

## 5.2 Operations Dashboard

* JVM heap %, GC pause time, thread count.
* Web 5xx/4xx rates, request throughput, p95/p99 latency.
* DB connections used vs. max; replication lag; slow queries.
* Disk usage / IOPS; inode exhaustion; network errors.

## 5.3 Database Dashboard (PostgreSQL)

* `pg_stat_activity` sessions; blocked queries; replication status.
* Checkpoint frequency; WAL generation rate; autovacuum activity.

---

# 6) Logs & Traces

* **App logs**: `atlassian-jira.log`, access logs if enabled; ensure rotation.
* **Proxy logs**: Request/response status and latency for SLI derivation.
* **DB logs**: Statement logging for slow queries; rotation policy.
* **Trace (optional)**: APM agent spans for slow web actions/JQL queries.

Retention targets (example): **7–14 days** hot, **90 days** cold; adjust per policy.

---

# 7) Alert Routing & Paging

| Trigger                 | Severity | Route To                                 | Ack/Resolve                                 |
| ----------------------- | -------- | ---------------------------------------- | ------------------------------------------- |
| Synthetic **down** (BH) | P1       | `[On‑Call Group]` via `[On‑Call System]` | Page immediately; 5‑min ack, 30‑min resolve |
| p95 **latency** breach  | P2       | `[On‑Call Group]`                        | Non‑urgent page; 30‑min ack, 2‑h resolve    |
| **Error rate** spike    | P2       | `[On‑Call Group]`                        | Page and investigate logs/APM               |
| **Disk ≥ 90%**          | P3       | `[Ticket Queue]`                         | Ticket; schedule cleanup                    |

* **Escalation ladder**: On‑call → secondary → service owner in **\[N]** minutes.
* **Maintenance suppression**: Silence alerts during approved windows with tags `[change_ref]`.

---

# 8) Runbooks (Link or Stub)

Create/attach a runbook per paged alert. Standard sections:

1. **Symptom**
2. **Impact**
3. **Immediate Actions** (10‑minute path)
4. **Diagnostics** (commands/queries)
5. **Rollback/Restore**
6. **Comms** (stakeholders/status page)
7. **Exit Criteria**

> Keep runbooks concise and copy‑pastable. Update after each incident.

---

# 9) Reporting & Review

* **Weekly**: SLO status, error budget remaining, top regressions.
* **Monthly**: Availability report (with exclusions), p95/p99 latency chart, incident summary & MTTR, capacity trends.
* **Quarterly**: SLI definition review; SLO target revalidation; dependency scorecard (DB/SMTP/IdP).

Include **post‑incident reviews** (PIRs) with action items and owners.

---

# 10) Business Hours & Maintenance

* **Business Hours**: `[Mon–Fri, 06:00–18:00, [Business Hours TZ]]`.
* **Maintenance Windows**: `[Cadence/Timing]`; alerts suppressed; SLI data **excluded** from SLOs.

> Maintain a calendar (shared) for on‑call and maintenance windows.

---

# 11) Data Sources & Integrations (Fill per org)

| Domain     | Tool/Source    | Notes              |                  |                      |
| ---------- | -------------- | ------------------ | ---------------- | -------------------- |
| Metrics    | \`\[Prometheus | Datadog            | New Relic]\`     | JVM, host, DB, proxy |
| Logs       | \`\[Splunk     | ELK                | CloudWatch]\`    | Access, app, DB      |
| Synthetics | \`\[Pingdom    | Datadog Synthetics | UptimeRobot]\`   | Homepage/API         |
| APM        | \`\[New Relic  | Datadog APM        | OpenTelemetry]\` | Traces & spans       |
| On‑Call    | \`\[PagerDuty  | Opsgenie           | ServiceNow]\`    | Paging & escalation  |

---

# 12) Governance

* **Change control**: Alerts tied to `[change_ref]` for silencing and audit.
* **SLO resets**: When SLI definitions change, **reset** budget and document rationale.
* **Ownership**: Primary `[Team/Role]`, backup `[Team/Role]`; See Overview Roles.

---

# 13) Examples (Replace with your stack)

**Nginx access log → request SLI metric (vector example)**

```
# count 5xx over total
rate(nginx_http_requests_total{status=~"5.."}[5m])
/ ignoring(status)
rate(nginx_http_requests_total[5m])
```

**Datadog monitor (pseudo)**

```
avg(last_5m): (100 - 100 * (sum:nginx.5xx{service:jira} / sum:nginx.requests{service:jira}))) < 99.9
```

**PostgreSQL replication lag (seconds)**

```
pg_last_xact_replay_timestamp() -- on replica
```

---

# Variables Used on This Page

`[Jira Base URL]`, `[Business Hours TZ]`, `[Synthetic Location(s)]`, `[On‑Call Group]`, `[On‑Call System]`, `[change_ref]`, `[Cadence/Timing]`, `[Team/Role]`.

# Import Notes

* Keep the **metadata code block** intact; it imports cleanly to Confluence.
* Replace placeholders systematically; keep SLI/SLO math close to the dashboards to reduce drift.
* If your tool can’t do burn‑rate math natively, approximate with rolling‑window thresholds and revisit later.
