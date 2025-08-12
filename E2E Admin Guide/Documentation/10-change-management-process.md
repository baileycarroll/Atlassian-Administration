---

```
title: Change Management Process
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, change, risk, rollout]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for a Jira platform **Change Management Process** (RFC, risk scoring, approvals, rollout, rollback, verification, comms). *Ref: `[Change/Ticket ID]`*

# Purpose

Define a lightweight but **auditable** process for modifying the Jira platform (app, DB, proxy, integrations). The goal is **predictability**, **safety**, and **clear ownership** without bogging teams down.

---

# 1) Scope & Definitions

**In‑scope changes** (examples)

* Jira application: upgrades, plugins/app updates, config/policy changes, JVM tuning
* Infrastructure: OS patches, proxy/TLS changes, storage moves
* Database: parameter changes, engine upgrades, schema tasks, replication changes
* Integrations: SSO/LDAP/AppLinks, SMTP changes, outbound webhooks

**Out‑of‑scope**

* Content/config inside a single project performed by project admins (unless platform‑wide impact)

**Change Types**

* **Standard**: pre‑approved, low‑risk, repeatable (e.g., monthly OS patches with no Jira update)
* **Normal**: assessed via risk score; requires peer/CAB review
* **Emergency**: to restore service or security posture; abbreviated approvals; PIR required

---

# 2) Roles

| Role               | Responsibilities                                                      |
| ------------------ | --------------------------------------------------------------------- |
| **Requester**      | Opens RFC, drafts plan/rollback/verification, coordinates with owners |
| **Implementer**    | Executes change in Dev/Test/Prod; owns on‑the‑day decisions           |
| **Reviewer**       | Peer review of plan, rollback, verification                           |
| **Approver**       | Final gate (Service Owner or delegate)                                |
| **Change Manager** | Schedules windows, chairs CAB (if used), enforces freeze/blackouts    |
| **Comms Owner**    | Sends notifications and status updates                                |

---

# 3) Maintenance & Freeze Windows

| Item                     | Default                                        |
| ------------------------ | ---------------------------------------------- |
| **Business Hours**       | `[Mon–Fri, 06:00–18:00, [TZ]]`                 |
| **Standard Maintenance** | `[Bi‑weekly, outside Business Hours]`          |
| **Blackout/Freeze**      | `[e.g., Finals week, Fiscal close]`            |
| **Emergency Window**     | Any time with Service Owner + on‑call approval |

> Schedule changes **outside Business Hours** unless the plan proves user impact is negligible.

---

# 4) Risk Scoring (Impact × Likelihood)

Compute **Risk Score = Impact × Likelihood** (1–5 each). Use the matrix to decide review depth.

| Score | Review                                        | Example                            |
| ----: | --------------------------------------------- | ---------------------------------- |
|   1–3 | Peer review; standard comms                   | Minor JVM flag toggle in Test      |
|   4–9 | Approver sign‑off; maintenance window         | Plugin update; proxy header change |
| 10–25 | CAB review; extended tests; backout rehearsed | Major Jira/DB upgrade              |

**Impact guide (choose highest that applies)**

* 1 = No user impact, reversible instantly
* 2 = Minor feature area, quick rollback
* 3 = Noticeable performance/availability risk
* 4 = Multi‑team outage possible; data at risk if misconfigured
* 5 = Broad outage/security exposure likely

**Likelihood guide**

* 1 = Routine, well‑tested; prior success ≥ 3 times
* 2 = Familiar; tested in non‑prod
* 3 = Some unknowns; partial tests
* 4 = Significant unknowns; complex dependencies
* 5 = First‑time/untested; vendor breaking changes

---

# 5) RFC Template (copy/paste into ticket)

```
Title: [Short, action-focused]
Change Type: [Standard | Normal | Emergency]
Requested Window: [YYYY-MM-DD HH:MM – HH:MM, TZ]
Environments: [Dev | Test | Prod]
Owner / Implementer: [Name]
Reviewer(s): [Name]
Approver: [Name]
Risk Score: Impact=[1-5], Likelihood=[1-5] → Score=[x]
User Impact Summary: [Who is affected? For how long?]
Backout Plan: [<= 5 steps, measurable]
Verification Plan: [<= 5 checks, with commands/paths]
Comms Plan: [Channels, timing, audience]
Dependencies: [DB, Proxy, IdP, SMTP, Storage, Backups]
Pre-Reqs Complete: [Yes/No; link to 03/04 checklists]
Attachments: [configs, scripts, diagrams]
```

---

# 6) Preparation — Required Before the Window

* [ ] Change rehearsed in **Test** (or Dev) with **timings** captured
* [ ] **Backup** assets identified and tested: DB snapshot/PITR point, Jira home snapshot, proxy config copy
* [ ] **Backout** is validated (dry‑run or lab) and **< 30 minutes** where feasible
* [ ] **Monitoring** in place: dashboards and alerts for availability/latency/error rate during change
* [ ] **Approvals** recorded in the RFC ticket
* [ ] **Comms** drafted and scheduled (pre/post notices)

---

# 7) Implementation Steps (Runbook Skeleton)

Use numbered, atomic steps. Include **expected outcome** and **fallback** at each step.

```
1. Announce start of change; place status to “Maintenance”.
2. Take seed backups (DB + Jira home). Confirm WAL archiving healthy.
3. Apply change step A (e.g., stop Jira; replace binaries; update server.xml).
   Expect: Jira stopped; ports closed. Fallback: revert service state.
4. Apply change step B (e.g., DB schema patch).
   Expect: migration completes in ≤ [N] min. Fallback: restore snapshot.
5. Start Jira; watch logs; run smoke tests.
6. Lift maintenance; monitor for [N] minutes under real traffic.
7. Close change with results and attach artifacts.
```

---

# 8) Rollback Strategy

Define a **single primary path** for backout; avoid branching.

**Examples**

* **Application upgrade** → stop Jira → restore previous install dir → restore Jira home snapshot if schema/plugins changed → start Jira → smoke
* **DB parameter change** → revert `postgresql.conf`/`ALTER SYSTEM` → reload/restart → confirm plan/latency
* **Proxy/TLS change** → reinstall prior vhost/cert → reload Apache → curl Base URL → confirm headers and expiry

**Done‑When**

* Prior version is running; smoke tests pass; errors back to baseline; incident ticket (if any) closed

---

# 9) Verification Gates (Exit Criteria)

Minimum checks to declare success:

| Check            | Method                  | Pass Criteria                         |
| ---------------- | ----------------------- | ------------------------------------- |
| Homepage         | `curl -I [Base URL]`    | `200 OK`, TLS valid, expected headers |
| Login            | UI                      | Admin login succeeds                  |
| DB               | UI → System Info        | No DB errors; connection pool healthy |
| Search/Index     | UI → Recent issues; JQL | Results return; no index warnings     |
| Email            | Send test mail          | Delivered via `[SMTP Host]`           |
| SSO (if changed) | IdP login               | Redirect + group mapping OK           |

> For high‑risk changes, add **p95 latency** and **error rate** watch for `[30–60]` minutes.

---

# 10) Communication Plan

**Audiences**

* Users (stakeholders/projects), Support/Helpdesk, Platform/SRE, Leadership (as needed)

**Channels & Timing**

* Pre‑notice: **`[X business days]`** before (email/status page)
* Start notice: at maintenance start
* End notice: on completion with **result + next steps**
* Incident notice: if change fails or is rolled back

**Template Snippet**

```
Subject: [Change] Jira [action/version] — [Date, TZ]
Window: [Start–End]
Impact: [User-visible impact]
Risk/Backout: [1-liner]
Contact: [On-call / support channel]
```

---

# 11) Emergency Changes

* Allowed only to **restore service** or patch critical security issues
* **Approvals**: On‑call + Service Owner (post‑facto CAB review mandatory)
* **PIR** within **\[2 business days]** with action items assigned

---

# 12) Post‑Implementation Review (PIR)

Within **\[5 business days]**, review:

* What was the **goal** and **result**?
* What changed vs. plan (timings, steps)?
* What alarms fired; how fast did we detect/respond?
* What should we **standardize** or **automate** next time?
* Update runbooks/thresholds; close action items by owners and dates

---

# 13) Artifacts & Evidence

Attach to the RFC ticket:

* Final **configs** (`server.xml`, `setenv.sh`, proxy vhost, `dbconfig.xml` diffs)
* **Logs** covering the window (`journalctl -u jira`, DB logs, proxy access)
* **Screenshots** of success metrics/dashboards
* **Backups** IDs/paths and restore points

---

# 14) Mapping to Other Sections

* **Pre‑Install Requirements** → environment readiness
* **Installation & Initial Config** → baseline files and directories
* **Routine Maintenance** → standard procedures reused as change steps
* **Observability & SLOs** → gating and monitoring
* **PostgreSQL Operations** → DB‑specific changes and restore tests
* **Troubleshooting & Diagnostics** → fallback and investigation patterns

---

# Variables Used on This Page

`[TZ]`, `[Base URL]`, `[SMTP Host]`, `[Change/Ticket ID]`, `[Bi‑weekly]`, `[X business days]`.

# Import Notes

* Keep the **metadata code block** intact; Confluence preserves fenced blocks.
* Tables and code fences import cleanly; convert relative links post‑import.
* Use the RFC template block verbatim inside your ticketing system for consistency.
