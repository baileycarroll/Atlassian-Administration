---
title: User & Role Management
product: Jira Data Center
version_target: "10.x"
envs: [Dev, Test, Prod]
owner_role: [Team/Role]
last_verified: [YYYY-MM-DD]
change_ref: [Change/Ticket ID]
compliance_context: [e.g., FERPA?|HIPAA?|N/A]
tags: [admin-guide, jira, access, permissions]
---

# Version Control Notes

* **\[YYYY‑MM‑DD]** — Initial generalized template for Jira Data Center v10+ user, group, and permission management. *Ref: `[Change/Ticket ID]`*

# Purpose

Define how identities enter Jira, how access is granted, and how permissions are structured across projects. This page gives a **least‑privilege** baseline that scales and audits cleanly.

# Outcomes — Done‑When

* [ ] Directory integration (LDAP/AD or SCIM) connected and syncing groups/users.
* [ ] SSO (SAML/OIDC) configured with a **local break‑glass admin** retained.
* [ ] Baseline **org‑wide groups** created and documented.
* [ ] Standard **project roles** and **permission scheme** applied to new projects.
* [ ] (Optional) **Issue Security Scheme** defined for sensitive work.
* [ ] **Global permissions** hardened (no direct user assignments).
* [ ] Joiners/Movers/Leavers (**JML**) workflow documented with SLAs.
* [ ] Quarterly **access review** schedule in place.

---

# Core Concepts (Jira Access Model)

* **Users** authenticate via local directory or external IdP; authorization is via **groups** and **project roles**.
* **Groups** are global containers (e.g., `jira-software-users`) used for **product access** and **global permissions**.
* **Project roles** are per‑project mappings (e.g., `Developers`, `Administrators`) that permission schemes reference.
* **Permission schemes** bind **project roles/groups** to **actions** (browse, edit, transition…).
* **Issue security schemes** restrict visibility to subsets of users/roles per issue level.

---

# 1) Identity Sources & SSO

## 1.1 Directory Integration (LDAP/AD)

UI: **Administration → User Management → User Directories → Add Directory**

**Recommended settings**

* **Directory Type**: `[Active Directory | LDAP]` (read‑only with local groups)
* **Bind DN / Password**: `[LDAP Bind DN]` / `[secret]`
* **User DN**: `[OU=People,DC=example,DC=org]`
* **Group DN**: `[OU=Groups,DC=example,DC=org]`
* **User Filter** (example):

  ```
  (&(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
  ```
* **Group Filter** (example):

  ```
  (|(cn=jira-*)(cn=dev-*)(cn=proj-*)))
  ```
* **Nested Groups**: enable if your directory uses them (performance impact).
* **Synchronization**: scheduled + on demand; map **username** to `sAMAccountName` or `uid` consistently.

> Keep the **internal directory first** with at least one local admin. External directories should be **read‑only**.

## 1.2 SSO (SAML/OIDC)

* **Protocol**: `[SAML 2.0 | OIDC]` via `[IdP Name]`.
* **Required attributes**: `email`, `displayName`, **immutable ID** (e.g., `uid`), and **group claims** if using JIT mapping.
* **ACS/Callback URL**: `[https://jira.example.org/plugins/servlet/samlconsumer]` (SAML) or app’s OIDC callback.
* **JIT / SCIM**: Optional; if enabled, define mapping from IdP groups → Jira groups.
* **Session**: respect org idle/session TTLs; add bypass rule for `/plugins/servlet/samlconsumer` during setup.

---

# 2) Baseline Groups (Org‑Wide)

> Create once; manage membership via directory sync where possible.

| Group                 | Purpose                              | Typical Members          |
| --------------------- | ------------------------------------ | ------------------------ |
| `jira-sysadmins`      | Break‑glass local admins             | 2–4 platform owners      |
| `jira-administrators` | Daily Jira admin work (no OS access) | App admins               |
| `jira-project-admins` | Project configuration                | Delegated project owners |
| `jira-software-users` | Product access (license gate)        | All active users         |
| `jira-readonly`       | View‑only access                     | Auditors / stakeholders  |
| `jira-automation`     | App links, bots                      | Service accounts         |

**Rules**

* Assign **global permissions** to groups, **not** to individuals.
* Keep `jira-sysadmins` small; enforce MFA and time‑bound elevation if possible.

---

# 3) Project Roles (Standard Set)

| Project Role     | Intended Use                            | Suggested Backing Group                |
| ---------------- | --------------------------------------- | -------------------------------------- |
| `Administrators` | Configure project, workflows, schemes   | `jira-project-admins` + team leads     |
| `Developers`     | Create/edit issues, transition, comment | Team group(s) (e.g., `dev-<team>`)     |
| `Contributors`   | Comment, attach, log work               | Wider org group (e.g., `staff-<dept>`) |
| `Reporters`      | Create issues, view own                 | All users or requesters                |
| `Viewers`        | Browse only                             | `jira-readonly`                        |
| `Automation`     | Bot actions                             | `jira-automation`                      |

> Project roles are **per‑project**; the same role can map to different groups in different projects.

---

# 4) Permission Scheme — Reference Template

Apply this as the **default** for new software projects; tailor as needed.

| Permission                 | Assign To                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------- |
| Browse Projects            | `Project Role: Viewers`, `Project Role: Developers`, `Project Role: Administrators` |
| Create Issues              | `Project Role: Developers`, `Project Role: Reporters`                               |
| Edit Issues                | `Project Role: Developers`                                                          |
| Transition Issues          | `Project Role: Developers`, `Automation`                                            |
| Add Comments / Attachments | `Developers`, `Contributors`                                                        |
| Assign Issues              | `Administrators`, optionally `Developers`                                           |
| Manage Sprints / Boards    | `Administrators`                                                                    |
| Project Administration     | `Administrators` only                                                               |

> Keep schemes simple. When in doubt, **reduce** direct group assignments and rely on **project roles**.

---

# 5) Issue Security Scheme (Optional, Recommended)

Define **visibility levels** for sensitive work:

| Level        | Who Can See                                    |
| ------------ | ---------------------------------------------- |
| `Public`     | All with Browse Projects                       |
| `Internal`   | `Developers`, `Administrators`                 |
| `Restricted` | Named groups/roles (e.g., `Legal`, `Security`) |

Set a **default level** and use automation to enforce on issue create in scoped projects.

---

# 6) Global Permissions — Hardened Defaults

UI: **System → Global Permissions**

| Global Permission                 | Assign To                                          |
| --------------------------------- | -------------------------------------------------- |
| Jira System Administrators        | `jira-sysadmins`                                   |
| Jira Administrators               | `jira-administrators`                              |
| Browse Users / Share Dashboards   | `jira-software-users`                              |
| Create Shared Objects             | `jira-administrators`, optionally trusted creators |
| Manage Group Filter Subscriptions | `jira-administrators`                              |

> Avoid giving `Create Shared Objects` to everyone; it can leak data via filters/dashboards.

---

# 7) Product Access (Licensing Gate)

UI: **User Management → Product Access**

* Grant Jira Software access to `jira-software-users`.
* Keep service accounts in `jira-automation` with access as needed.

---

# 8) Joiner / Mover / Leaver (JML) Workflow

## 8.1 Joiner (New User)

* Provision identity in directory/IdP.
* Add to appropriate **IdP groups**; allow sync/JIT to create in Jira.
* Confirm product access via `jira-software-users`.

## 8.2 Mover (Role/Team Change)

* Update team groups; review project role mappings.
* Remove stale group memberships.

## 8.3 Leaver (Departing)

* Disable in IdP; remove from groups.
* Transfer issue ownership; disable tokens; archive personal filters/dashboards if policy requires.

**SLA Targets** (sample): Joiner ≤ 1 business day; Leaver within same business day.

---

# 9) Elevated Access & Break‑Glass

* Maintain **2–4 local admins** in the internal directory for SSO outages.
* Enforce MFA on admin groups; time‑box membership where possible.
* Log and review admin group changes (see **Auditing**).

---

# 10) Auditing & Reviews

* **Quarterly**: export admin group membership and compare to approvals.
* **Monthly**: report public projects, anonymous access, and shared filter owners.
* **On change**: record permission scheme edits with change/ticket reference.

---

# 11) Troubleshooting

* **User can’t log in**: check IdP assertion, directory sync, and duplicate usernames.
* **Can’t see project/issue**: verify project role assignment, permission scheme, and issue security level.
* **Group mismatch**: nested groups disabled or filter too restrictive; adjust directory config.
* **Admin lockout**: use local break‑glass admin in internal directory.

---

# Variables Used on This Page

`[LDAP Bind DN]`, `[IdP Name]`, `[OU=People]`, `[OU=Groups]`, `[TZ]`, `[Jira Version]`, `[Download URL]`.

# Import Notes

* Keep the **metadata code block** intact for portability.
* Tables import cleanly to Confluence; convert any relative links after import.
* For very large permission matrices, consider linking to a separate page per scheme.
