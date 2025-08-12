---

```
title: Operational Checklists
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, operations, checklists]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized operational checklists for Jira Data Center v10+. *Ref: `[Change/Ticket ID]`*

# Purpose

Provide **repeatable, auditable checklists** to keep Jira healthy: daily, weekly, monthly, and quarterly tasks; pre/post‑change gates; on‑call handoffs; and DR drills. Use “**Done‑When**” criteria to determine completion.

---

# Quick Reference — Operating Envelope

| Item                | Target / Policy                                                             |
| ------------------- | --------------------------------------------------------------------------- |
| Business Hours      | `[Mon–Fri, 06:00–18:00, [TZ]]`                                              |
| Availability SLO    | `99.9%` during Business Hours (see §07)                                     |
| Maintenance Cadence | `[Bi‑weekly]`, outside Business Hours                                       |
| SMTP                | `[SMTP Host]:[SMTP Port]`, From `[From Address]`, Prefix `[Subject Prefix]` |
| DB Port             | `5432` (unless overridden)                                                  |
| Proxy               | HTTPS at `[Base URL]` → reverse proxy → Jira on `127.0.0.1:8080`            |

---

# Daily Checklist (10–15 min)

**Done‑When**

* [ ] Homepage returns **200 OK**; p95 < **\[2.5s]**.
* [ ] **Error spikes** absent in last 24h (`SEVERE/ERROR` review complete).
* [ ] **Disk usage** < **80%** on app/DB hosts; Jira logs < **\[10 GB]** total.
* [ ] **DB health** nominal; replica lag < **\[60s]** (if applicable).
* [ ] **Outgoing mail** test succeeds; queue empty.
* [ ] **Backups** (DB + Jira home) **SUCCESS** in last 24h.

**Quick Commands (reference)**

```bash
# Availability
curl -s -o /dev/null -w "%{http_code}\n" [https://[Base URL]]

# Logs (last day)
sudo journalctl -u jira --since "-24h" | egrep -i "(SEVERE|ERROR)" | tail -n 100

# Disk
df -h | awk 'NR==1 || /\/opt|\/data|\/var/'

# DB replica lag (on replica)
psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS replica_lag;"
```

**Notes**

* Record any exceptions in the daily ops log with owner and follow‑up date.

---

# Weekly Checklist (30–45 min)

**Done‑When**

* [ ] **Index status** is Online; no rebuild required; search consistent.
* [ ] **Logs rotated** and archived per policy.
* [ ] **Top slow actions/GC pauses** reviewed; no regression vs. last week.
* [ ] **Plugins/apps** reviewed; versions pinned; no pending risky updates.
* [ ] **Directory/SSO** syncs green; no orphaned/duplicate users.

**How‑To Notes**

* UI → **System → Indexing** (status, last run).
* If drift detected, schedule a **Foreground Reindex** for off‑hours.

---

# Monthly Checklist (60–120 min)

**Done‑When**

* [ ] **OS patches** applied (app + DB); Jira restarted cleanly if required.
* [ ] **Java/JDBC** updates evaluated/applied if approved.
* [ ] **DB maintenance** executed (VACUUM/ANALYZE focus) with results captured.
* [ ] **Backup drill**: at least one **test restore** to disposable env completed.
* [ ] **SLO report** (availability/latency) produced and reviewed.

**Artifacts to Attach**

* Patch/change ticket IDs, logs around the window, and validation screenshots.

---

# Quarterly Checklist (2–4 h)

**Done‑When**

* [ ] **Restore verification** (DB + Jira home) completed; Jira boots and runs smoke.
* [ ] **Capacity review**: CPU/heap/IO/DB connections; plan if >70% sustained.
* [ ] **Permission audit**: admin groups minimal; shared objects reviewed.
* [ ] **TLS/license** review: expiry ≥ 30 days away; renewals tracked.
* [ ] **DR drill** executed (tabletop or hands‑on) with notes.

**Evidence**

* Restore timings, screenshots, comparison to previous quarter.

---

# Pre‑Change Gate (Before Any Platform Change)

```
[ ] RFC approved (Risk Score documented)
[ ] Backups verified (DB PITR point + Jira home snapshot)
[ ] Maintenance window scheduled (outside Business Hours)
[ ] Monitoring in place (dashboards & alert suppression tags)
[ ] Backout plan rehearsed or validated
[ ] Comms drafted (pre, start, end)
```

---

# Post‑Change Verification Gate (Exit Criteria)

```
[ ] Homepage 200 OK; TLS valid; expected headers present
[ ] Admin login works (local + SSO if applicable)
[ ] DB pool healthy; no errors in logs
[ ] Search returns recent issues; no index warnings
[ ] Attachments upload/preview OK
[ ] Test email delivered via [SMTP Host]
[ ] Background jobs healthy (no new failures)
```

---

# On‑Call Handoff Template (Shift Change)

```
Date/Time (TZ):
Primary/Secondary:
Open Incidents/Actions:
Risky Areas (today):
Recent Changes (last 24–48h):
SLO/SLI Status (availability, latency):
Known Issues & Workarounds:
Next Window / DR Drills:
```

---

# DR Drill Script (Warm DR Pattern — Adapt)

1. Confirm current backups and WAL status.
2. Quiesce writes (stop Jira or isolate).
3. Restore DB snapshot to DR target; replay WAL to chosen point.
4. Sync **Jira home** (attachments/config) to DR target.
5. Start Jira on DR; validate base URL, mail, SSO.
6. Run smoke tests; capture timings.
7. Tear down or keep warm per policy; log lessons and gaps.

**Pass Criteria**

* RTO ≤ **\[1h]**, RPO ≤ **\[1h]** (adjust to your SLOs).

---

# Weekly/Monthly Reporting Templates

**Weekly Ops Note**

```
Week of: [YYYY‑MM‑DD]
Availability (BH): [99.9% target] — Actual: [ ]
Top Regressions: [ ]
Incidents/MTTR: [ ]
Capacity Trends: CPU [ ], Heap [ ], DB conns [ ]
Actions/Owners: [ ] → [date]
```

**Monthly SLO Report**

```
Month: [YYYY‑MM]
Availability (BH): Target 99.9% — Actual: [ ] (exclusions: [ ])
Latency p95/p99: [ ] / [ ] (BH)
Incidents: count [ ], MTTR [ ]
Backups/Restore Tests: [pass/fail]
Changes Executed: [#], Rollbacks: [#]
Notes/Risks: [ ]
```

---

# Artifacts & Evidence (Attach to Tickets)

* Config diffs (`server.xml`, `setenv.sh`, proxy vhost, `dbconfig.xml`).
* Logs around the window (app, DB, proxy).
* Screenshots of dashboards for SLOs and health.
* Backup IDs and restore points.

---

# Variables Used on This Page

`[TZ]`, `[Bi‑weekly]`, `[Base URL]`, `[SMTP Host]`, `[SMTP Port]`, `[From Address]`, `[Subject Prefix]`.

# Import Notes

* Keep the **metadata code block** intact for portability.
* Checklists and templates copy cleanly into tickets and runbooks.
* Align thresholds with §07 **Observability & SLOs** and §06 **Routine Maintenance** to avoid drift.
