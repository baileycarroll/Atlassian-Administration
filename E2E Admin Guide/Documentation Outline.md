# End-to-End Administration Guide — Jira Data Center v10+

1. **Introduction & System Overview**
   Purpose, scope, supported versions; high-level architecture (nodes, `[LB VIP]`, `[Shared Storage Path]`, `[PG Host]`), core integrations (LDAP/SSO, SMTP, monitoring), and what “Data Center” adds vs Server.

2. **Environments & Topology**
   Define `[Environment Name]` tiers; node counts and roles; `[Network Zone]` & firewall ports; DNS, certificates, `[LB VIP]` & session affinity; shared home layout; cluster indexing model.

3. **Pre-Installation Requirements**
   Hardware sizing & JVM baseline (`[JVM_HEAP_SIZE]`); supported OS/Java; PostgreSQL version policy; network prereqs; service accounts; storage & IOPS; dependency matrix.

4. **Installation & Initial Configuration**

   * Prepare OS (packages, limits, time sync).
   * Database: create `[PG Database]`/`[PG User]`, drivers, connectivity test.
   * Jira nodes: install, point to shared home, cluster config, license, base URL, mail.
   * Service management (systemd), backup of clean state, smoke tests.

5. **User & Role Management**
   Directory integration (LDAP/SAML/SSO), group sync mapping, admin boundaries; permission schemes & project role archetypes; audit guidance.

6. **Routine Maintenance**
   Patch cadence (core + apps), JVM/GC tuning, log rotation, application indexing/reindexing strategy, housekeeping jobs, backup jobs (app home + DB), capacity checks.

7. **Observability & SLOs**
   Standard dashboards (app/JVM/DB/system), alert thresholds, synthetic checks, runbooks, paging & error budgets; sample SLOs and starter alerts (replace with org values).

8. **PostgreSQL Operations**
   HA model (placeholders: primary/replica); backups (nightly full + WAL), PITR drills; vacuum/analyze cadence and bloat; connection pooling (PgBouncer); upgrade playbook; **restore verification** checklist.

9. **Advanced Administration Tasks**
   Load-balancing patterns (SSL/TLS, health checks, stickiness); cluster/node lifecycle (join, drain, replace); performance tuning scenarios; external integrations (Dev tools, mail relays, proxies).

10. **Change Management Process**
    RFC template, risk scoring, CAB/approvals, freeze windows; implementation steps; validation gates; rollback strategy; comms plan.

11. **Troubleshooting & Diagnostics**
    Log taxonomy & search paths; thread/heap dumps; support zip; indexing & search faults; DB hotspots; auth/sessions; common known faults; escalation map.

12. **Security Model**
    SSO/SAML patterns; LDAP/SCIM sync; permission scheme archetypes; secrets management; TLS termination & rotation schedule; plugin vetting and CVE response.

13. **Add-on/Plugin Compatibility Matrix**
    Supported versions per environment, test plan, upgrade gates, rollback notes; owner and review cadence.

14. **Upgrade & Migration Playbooks**
    Minor vs major strategy; blue/green or canary; index handling; post-upgrade validation & sign-off.

15. **Operational Checklists**
    Daily/weekly/monthly routines with “done-when” criteria; pre-release and post-release checks.

16. **Appendix & References**
    Glossary, command reference, SQL snippets, file paths, diagrams; curated links to Atlassian docs and internal runbooks.

17. **Variable Glossary**
    Canonical list of all placeholders used (e.g., `[Company Name]`, `[Server Hostname]`, `[LB VIP]`, `[PGDATA]`, etc.) with example values.

18. **Import Notes (Obsidian ↔ Confluence)**
    Known importer limits, anchors/TOC handling, image/link formatting, macro equivalents; how to keep YAML front-matter intact on import.

---

### Global Conventions (apply to every page)

* **YAML front-matter** at top (title, product, version\_target, envs, owner\_role, last\_verified, change\_ref, compliance\_context, tags).
* **Variables in brackets** for anything org-specific.
* **Portable Markdown** only; code blocks for shell/SQL/config; tables for tunables; text diagrams if images unavailable.
