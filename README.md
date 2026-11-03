# ServiceNow Portfolio

A curated collection of ServiceNow development, integration, and infrastructure projects. Each repository below was built and validated against live instances — no hypothetical code.

---

## Quick Discovery

Hands-on Quick Discovery against a mixed lab (Windows MID on Windows 11 + Linux MID on ASUS GX10), including the resulting Computer CI, ECC Queue probes, Windows Classify probe, MID Server capability differences (Windows vs Linux vs Mac), and how to scan a whole subnet instead of one IP at a time.

**Repo:** [https://github.com/eltonaguiar/ServiceNow_QuickDiscovery](https://github.com/eltonaguiar/ServiceNow_QuickDiscovery)

---

## Projects

| # | Project | Focus | Key pieces (breadcrumbs) | Detail | Repo |
|---|---|---|---|---|---|
| **#1** | `glideajax-scriptinclude-onchange-incident-warning` | Warn when an assignee already has more than 10 active incidents | `IncidentUtils.getOpenIncidentCount()` · onChange client script on `assigned_to` · `getXMLAnswer()` callback · `GlideAggregate` COUNT on `incident` (`active = true`) | [read ↓](#1-glideajax-scriptinclude-onchange-incident-warning) | [repo ↗](https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning) |
| **#2** | `ServiceNowMidServer` | ARM64 MID Server on GX10 / Raspberry Pi / Apple Silicon — **the prerequisite** both integrations below run through | `setup-mid-fast.sh` · `mid-supervisor.sh` (the wrapper-less replacement) · ARM64 JDK 17 with `--add-opens` ×14 · `docker run --platform linux/arm64 --restart unless-stopped` · MID `win11-mid` / `gx10-mid` · `ecc_agent` status verification | [read ↓](#2-servicenowmidserver) | [repo ↗](https://github.com/eltonaguiar/ServiceNowMidServer) |
| **#3** | `ServiceNowJDBC` | MySQL → ServiceNow over **JDBC** — the **data source's *Use MID Server*** field points at `win11-mid` | `sys_data_source` `torontoeventsv2` (JDBC, `org.mariadb.jdbc.Driver`) · **helper field `last_run_database_field = last_updated`** + the single `last_run_datetime` watermark parameter · staging `u_torontoevents2` · `TorontoEventsTransformMap` (**coalesce on `u_event_id`**) · target `u_torontoeventsfinal` | [read ↓](#3-servicenowjdbc) | [repo ↗](https://github.com/eltonaguiar/ServiceNowJDBC) |
| **#4** | `ServiceNowScheduledExport` | Scheduled CSV export sets — target is **File on MID Server** `win11-mid`, reading the table #3 fills | `sys_export_definition` (`u_torontoeventsfinal`) · `sys_export_target` (`win11-mid`, `toronto/events`) · `sys_export_set` (`torontoevents.csv`, append timestamp) · `scheduled_data_export` (daily, delta on `sys_updated_on`) · conditional / pre / post scripts | [read ↓](#4-servicenowscheduledexport) | [repo ↗](https://github.com/eltonaguiar/ServiceNowScheduledExport) |

The numbers are a **dependency order**, not alphabetical. **#1** is client-side behaviour on a single record and
stands alone. **#2 is the prerequisite**: the MID Server is the thing both integrations actually run through — the
JDBC data source's *Use MID Server* field sends its SQL out through `win11-mid`, and the export target writes its CSV
to that same MID's disk. So **#3** is how rows get *into* ServiceNow, and **#4** is how they get back out.
Section N below is the deep dive for row **#N**. Every repo now opens with a **prerequisites checklist** and then
**numbered steps** — each step is a breadcrumb, its prerequisites, the fields to fill, one suggestion and a
checkpoint — so the section here summarises and the repo is where you actually execute.

**Jump to:** [#1 GlideAjax](#1-glideajax-scriptinclude-onchange-incident-warning) ·
[#2 MID Server](#2-servicenowmidserver) · [#3 JDBC import](#3-servicenowjdbc) ·
[#4 Export sets](#4-servicenowscheduledexport) · [Quick skill summary](#quick-skill-summary) ·
[Appendix A](#appendix-a--experimental-only-out-of-platform-ai--table-api-changes)

---

*(The rest of the detailed project sections remain unchanged below.)*
