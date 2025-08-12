---

```
title: Add‑on / Plugin Compatibility Matrix
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Platform/App Governance]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [N/A]
tags: [admin-guide, jira, marketplace, plugins, governance]
---
```

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for Marketplace app governance and compatibility tracking. *Ref: `[Change/Ticket ID]`*

# Purpose

Provide a **single source of truth** for Jira Marketplace add‑ons/plugins: versions, licensing, security posture, data egress, test status, and **go/no‑go** gates for upgrades. Use this to minimize outage risk and keep upgrades predictable.

---

# Governance Overview

* **Catalog** all installed/approved apps with owners and business justification.
* **Pin versions** per environment; upgrades flow **Dev → Test → Prod** with evidence.
* **Review** security/egress on intake and at each major Jira version change.
* **Rollback** plan documented per app; keep last known good (LKG) version handy.

---

# 1) Compatibility Matrix (Master Table)

> Populate one row per app. Keep environments and Jira/DC compatibility explicit.

| App (Vendor)              | Key                | Category                          | License       | Owner         | Jira DC Support | Approved Versions (Dev/Test/Prod)         | LKG       | Data Egress                      | Security Review | Notes              |
| ------------------------- | ------------------ | --------------------------------- | ------------- | ------------- | --------------- | ----------------------------------------- | --------- | -------------------------------- | --------------- | ------------------ |
| `[App Name]` (`[Vendor]`) | `[com.vendor.key]` | `[Issue Panels/Workflow/Utility]` | `[Server/DC]` | `[Team/Role]` | `[Yes/No]`      | `Dev=[x.y.z], Test=[x.y.z], Prod=[x.y.z]` | `[x.y.z]` | `[None/Cloud/SaaS/External API]` | `[Date/Link]`   | `[Why we need it]` |

**Column guidance**

* **Key**: the plugin key (e.g., `com.atlassian.tempo.tempo-plugin`).
* **Jira DC Support**: confirm the app is **Data Center compatible**.
* **Data Egress**: list outbound destinations or `None`.
* **Security Review**: link to ticket/assessment; include review date.

---

# 2) Intake Checklist (New App Request)

* [ ] Business case approved; owner named and accountable.
* [ ] Vendor **Data Center compatibility** verified.
* [ ] Licensing model and cost understood; renewal path defined.
* [ ] **Security** review opened (egress, scopes, third‑party services).
* [ ] **Compliance** implications reviewed (PII/PHI/regulated data?).
* [ ] Test plan drafted; success criteria defined.
* [ ] Rollback plan defined (disable, revert, restore).

**Decision**: Approve / Defer / Reject — with rationale.

---

# 3) Upgrade Gates (Per App)

* **Dev**

  * [ ] Install target version; capture startup logs; verify no critical errors.
  * [ ] Run smoke: create/edit/transition issues; boards; JQL; attachments.
  * [ ] Run app‑specific checks (see §6).
* **Test**

  * [ ] Upgrade from LKG → target; **data migration** steps executed.
  * [ ] Foreground **Reindex** if required by release notes.
  * [ ] Performance sanity: p95 within baseline; error rate unchanged.
* **Prod**

  * [ ] Maintenance window scheduled; users notified.
  * [ ] Snapshots taken (DB + Jira home); backout rehearsed.
  * [ ] Post‑upgrade smoke + 30–60 min watch; sign‑off recorded.

---

# 4) Rollback Strategy (Template)

1. Disable app (safe mode if necessary) and assess impact.
2. Reinstall **LKG** version; apply vendor rollback steps if schema changes.
3. If corruption suspected: **restore DB + Jira home** snapshots taken pre‑change.
4. Reindex if required; run smoke tests.
5. Document cause and prevent recurrence (pin, blocklist if needed).

---

# 5) Risk & Impact Scoring (Lightweight)

| Factor      | Low (1)          | Med (2)          | High (3)                         |
| ----------- | ---------------- | ---------------- | -------------------------------- |
| Scope       | Cosmetic/utility | Workflow/fields  | Core (auth/indexing/performance) |
| Data Egress | None             | Internal only    | External (SaaS/API)              |
| Vendor Risk | Established      | Moderate         | New/unknown or poor support      |
| Migration   | None             | Minor data model | Major data migration             |

**Risk Score** = sum. **≥6** requires Service Owner sign‑off. Link to RFC in §10 Change Process.

---

# 6) App‑Specific Test Plans (Examples)

Use/clone these patterns to define **what “works” means** for each app.

* **Timesheets** (e.g., Tempo): create worklog, edit, approve, export.
* **Automation**: trigger on create; field update; webhook fired.
* **Workflow Extension**: transition with validator/condition; post‑function result.
* **Reporting/Gadgets**: dashboard loads; data sources connect; filter scope correct.
* **Custom Fields**: create/search/sort; export; REST payloads stable.

Capture: steps, expected results, screenshots, and **pass/fail**.

---

# 7) Environment Policy

* **Pin** exact versions per environment; never auto‑update.
* **Match** Prod configuration flags to non‑prod before testing.
* **Lock** app administration to platform admins; project admins only configure app settings where safe.
* **Blocklist** risky apps by key if disallowed (write in `jira-config.properties`).

---

# 8) Licensing & Renewals

| App          | License Type           | Seats | Renewal Date   | Owner         | Notes          |
| ------------ | ---------------------- | ----: | -------------- | ------------- | -------------- |
| `[App Name]` | `[Data Center/Server]` | `[n]` | `[YYYY‑MM‑DD]` | `[Team/Role]` | `[PO#/Budget]` |

* Track **renewal reminders** ≥ 30 days ahead.
* Keep license keys/certs in `[Secrets Manager]` (not in repo).

---

# 9) Vendor Contacts & Support

| App          | Vendor Contact      | Support URL                    | SLA                       |
| ------------ | ------------------- | ------------------------------ | ------------------------- |
| `[App Name]` | `[name@vendor.com]` | `[https://support.vendor.tld]` | `[e.g., 2 business days]` |

* Maintain **support tickets** and release notes per app.

---

# 10) Change Management Hooks

* Every app install/upgrade → **RFC** (see §10 Change Management Process).
* Attach: test plan/results, logs, screenshots, and **backout**.
* Suppress alerts during maintenance; watch availability/latency/error rate after.

---

# 11) Known Incompatibilities (Examples)

| App       | Conflict                   | Notes/Workaround                      |
| --------- | -------------------------- | ------------------------------------- |
| `[App A]` | Conflicts with `[App B]`   | Disable one; vendor fix pending       |
| `[App C]` | Breaks on Jira `[version]` | Use LKG `[x.y.z]` until vendor update |

---

# 12) Historical Record

Log past upgrades with outcome and time‑to‑recover (if rollback occurred).

| Date           | App/Version → Version | Env  | Result               | Notes                 |
| -------------- | --------------------- | ---- | -------------------- | --------------------- |
| `[YYYY‑MM‑DD]` | `[App x.y.z → x.y.z]` | Prod | `[Success/Rollback]` | `[Impact/Follow‑ups]` |

---

# Variables Used on This Page

`[App Name]`, `[Vendor]`, `[com.vendor.key]`, `[Team/Role]`, `[x.y.z]`, `[Jira Version]`, `[Secrets Manager]`, `[Change/Ticket ID]`.

# Import Notes

* Keep the **metadata code block** intact.
* The large matrix tables import cleanly to Confluence; widen columns if needed.
* For many apps, split the matrix by category and link to subpages, but keep a master index here.
