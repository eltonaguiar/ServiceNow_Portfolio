# ServiceNow Portfolio

A curated collection of ServiceNow development, integration, and infrastructure projects. Each repository below was built and validated against live instances — no hypothetical code.

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
Section N below is the deep dive for row **#N**; each links to the same material (build order, parameters,
watch-outs, validation) in that repo's README.

**Jump to:** [#1 GlideAjax](#1-glideajax-scriptinclude-onchange-incident-warning) ·
[#2 MID Server](#2-servicenowmidserver) · [#3 JDBC import](#3-servicenowjdbc) ·
[#4 Export sets](#4-servicenowscheduledexport) · [Quick skill summary](#quick-skill-summary) ·
[Appendix A](#appendix-a--experimental-only-out-of-platform-ai--table-api-changes)

---

<details>
<summary><strong>1. glideajax-scriptinclude-onchange-incident-warning <span>—</span> GlideAjax Script Include + onChange client script</strong> &nbsp;—&nbsp; <em>collapsed by default, click to expand</em></summary>

<a id="1-glideajax-scriptinclude-onchange-incident-warning"></a>

**Repo:** [https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning](https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning) · *GlideAjax Script Include + onChange client script — warn-only guard on incident assignment*

**In the repo:** [deployment order (breadcrumbs) ↗](https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning#how-to-deploy-this--order-of-operations) · [parameters to consider ↗](https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning#parameters-to-consider) · [watch-outs ↗](https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning#things-to-watch-out-for) · [5-minute validation ↗](https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning#validate-it-5-minutes-no-code)

**Skills demonstrated:** ServiceNow client scripts, Script Includes, GlideAjax, form client API (`g_form`), onChange client scripts, server-side data validation, asynchronous JavaScript callback patterns, scoped vs global application contexts.

This repository shows a complete **onChange client script + Script Include** pair that warns users when an incident assignee already has more than 10 active incidents. It covers three GlideAjax response patterns with a clear comparison table:

| Method | Sync/Async | Callback receives | When to use |
|---|---|---|---|
| `getXMLAnswer(callback)` | Async | Answer string only | **Preferred** for most cases |
| `getXML(callback)` | Async | Full XML response | When you need more than just the answer |
| `getXMLWait()` + `getAnswer()` | Sync (blocks UI) | (none — read via `getAnswer()`) | Critical `onSubmit` validation only |

**Key lessons captured:**
- Stale-response guards (ignore answers after the field has changed again)
- Input validation on the server side (sys_id format check before querying)
- The difference between `global.AbstractAjaxProcessor` (scoped) and `AbstractAjaxProcessor` (global)
- Why client-callable does **not** mean unrestricted access — authorization still matters

</details>

**[↑ Projects](#projects)** · next: **[#2 →](#2-servicenowmidserver)**

---

<details>
<summary><strong>2. ServiceNowMidServer <span>—</span> ARM64 MID Server + recovery runbook <em>(prerequisite for #3 and #4)</em></strong> &nbsp;—&nbsp; <em>collapsed by default, click to expand</em></summary>

<a id="2-servicenowmidserver"></a>

**Repo:** [https://github.com/eltonaguiar/ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer) · *ARM64 MID Server on unconventional hardware, plus the recovery runbook*

**In the repo:** [order of operations — instance first, then the box ↗](https://github.com/eltonaguiar/ServiceNowMidServer#order-of-operations--instance-first-then-the-box) · [recovering a Down MID (the exact runbook) ↗](https://github.com/eltonaguiar/ServiceNowMidServer#recovering-a-down-mid--the-runbook-we-actually-ran) · [troubleshooting matrix ↗](https://github.com/eltonaguiar/ServiceNowMidServer#troubleshooting--every-error-we-hit)

**Skills demonstrated:** ServiceNow MID Server, ARM64 architecture workarounds, Java/JDK management, Docker container operations, MID Server discovery, ITOM plugin management, troubleshooting production integration issues.

**The problem:** ServiceNow only publishes MID Server downloads for Intel/AMD (`x86-64`). On ARM64 machines (NVIDIA GX10 / GB10 Grace Blackwell, Raspberry Pi, Apple Silicon), the official installer won't run.

*(ELI5: it's like being handed a left-handed glove when you're right-handed — it just doesn't fit.)*

### The solution
A MID Server is really just a Java program. We:
1. Skip the Intel-only installer — run the Java files directly
2. Install a native ARM64 JDK 17 (the bundled Java 11 is too old for current MID releases)
3. Download ServiceNow's official MID core files (pure, chip-independent Java)
4. Unzip with system `unzip` (bypasses MID's own zip size limit on upgrade)
5. Start MID directly with `java`, skipping the Intel-only Tanuki wrapper
6. Add `--add-opens` flags (×14) + `-Djava.security.manager=allow` for Java 17 compatibility

**Result:** A native ARM64 MID Server running the exact release your instance expects — no emulation, full speed.

### Five version/architecture mismatches worked around

| # | Mismatch | Fix |
|---|---|---|
| 1 | CPU type: `arm64` vs `x86-64` downloads | Skip the `.deb`; run Java files directly |
| 2 | MID release: community image is Zurich, instance wants Australia | Overlay the official Australia MID files |
| 3 | Java 11 vs 17 | Install ARM64 JDK 17 |
| 4 | Intel-only starter program | Start `com.service_now.mid.Main` directly with `java` |
| 5 | Java 17 module system restrictions | Add `--add-opens` flags + `java.security.manager=allow` |

### Verified end-to-end
Tested on an **NVIDIA DGX Spark / GX10** (GB10 Grace Blackwell, `aarch64`, Ubuntu 24.04.5 LTS) against an **Australia patch3** instance. The screenshot in the repo README shows `Status: Up`, `Validated: Yes`, `Version: australia-02-11-2026__patch3-05-25-2026`.

### Companion: ITOM Discovery guide
`DISCOVERY.md` covers installing and running ITOM Discovery with this ARM64 MID — including how to distinguish the MID user (authenticates MID → instance) from Discovery credentials (how the MID logs into target hosts), and a troubleshooting matrix for the three different install-status sources (`sys_progress_worker` vs `sys_plugin_log` vs the stale console badge).

</details>

**[↑ Projects](#projects)** · next: **[#3 →](#3-servicenowjdbc)**

---

<details>
<summary><strong>3. ServiceNowJDBC <span>—</span> JDBC import pipeline + MID Server runbook</strong> &nbsp;—&nbsp; <em>collapsed by default, click to expand</em></summary>

<a id="3-servicenowjdbc"></a>

**Depends on [#2 ServiceNowMidServer](#2-servicenowmidserver)** — the data source's *Use MID Server* field
is what makes the SQL run from `win11-mid` on our network rather than from the instance.

**Repo:** [https://github.com/eltonaguiar/ServiceNowJDBC](https://github.com/eltonaguiar/ServiceNowJDBC) · *MySQL → ServiceNow over JDBC + the MID Server runbook*

**In the repo:** [build order — the 7 pages we visited ↗](https://github.com/eltonaguiar/ServiceNowJDBC#a-build-order--how-the-jdbc-import-was-wired) · [parameters ↗](https://github.com/eltonaguiar/ServiceNowJDBC#parameters-to-consider) · [what to watch out for ↗](https://github.com/eltonaguiar/ServiceNowJDBC#things-to-watch-out-for) · [validation ↗](https://github.com/eltonaguiar/ServiceNowJDBC#validation--four-checks-with-numbers-from-this-instance) · [where the "service" is + Linux autostart ↗](https://github.com/eltonaguiar/ServiceNowJDBC#part-c-where-the-service-is)

**Skills demonstrated:** ServiceNow JDBC data sources (`sys_data_source`), MID-executed scheduled imports and import sets, transform maps with coalesce keys, table configuration, MySQL diagnostics, Docker/systemd operations, MID Server lifecycle troubleshooting.

Two halves: **how the data gets in** (MySQL → ServiceNow over JDBC) and **how to keep the machinery alive** (the MID Server runbook), plus a walkthrough for people whose only mental model is `services.msc`.

```
MySQL  events_log                        (source of truth, ~12.2 K rows, unique event_id)
  └─ JDBC Data Source  torontoeventsv2   executed by MID win11-mid, driver org.mariadb.jdbc.Driver
       └─ Import Set staging  u_torontoevents2
            └─ Transform Map  TorontoEventsTransformMap   coalesce on u_event_id
                 └─ Table  u_torontoeventsfinal           (this is what [project #4](#4-servicenowscheduledexport) exports)
```

### Idempotency, and "only newer records" — one helper field, one parameter

| Lever | Setting | Value here |
|---|---|---|
| Only newer rows | `use_last_run_datetime` + helper field `last_run_database_field` | `true` / **`last_updated`** |
| The one parameter | `last_run_datetime` | `'2026-11-01 12:09:41'` (a single quoted datetime) |
| No duplicates on re-delivery | `coalesce = true` on `sys_transform_entry` | `u_event_id` |

1. **Newer only** — with *Use last run datetime* on and the helper field set to `last_updated`, ServiceNow appends
   `AND last_updated >= '<last_run_datetime>'` to the `sql_statement`. That one parameter is the watermark (the
   timestamp of the last successful run), so a daily import pulls the edits instead of the full table.
2. **Idempotent** — the transform map coalesces on `u_event_id`, so a row delivered twice (overlapping window,
   retry, manual *Execute Now*) **updates** the record instead of duplicating it. `COUNT(*)` and
   `COUNT(DISTINCT u_event_id)` stay equal however often the import runs.
3. **Recovery** — clearing `last_run_datetime` forces a full re-read. Handy after a broken window, but it
   re-delivers everything, which is only safe because of (2).

Also worth knowing: `u_last_updated` is a **string** (max 40) on the target table, not a date — it carries the
MySQL text through untouched, so do date maths in the transform script rather than on that column.

### The rest of the repo

| Part | Content |
|---|---|
| A. Database | Connection (`mysql.50webs.com:3306` / `ejaguiar1_events`), `events_log` plus the bookkeeping tables, and a copy-paste query set — total vs distinct, freshness, upcoming, per month, per source, null dates. Gotchas: the date column is `event_date` (not `date`); `categories`/`tags` are JSON in text |
| B. MID Server runbook | Why `docker ps` lies (PID 1 is a `tail`, the JVM can be dead), the real liveness test, `docker restart sn-mid`, verification via logs and `ecc_agent`, keystore / `401` / volume-ownership repairs, the wrapper-less root cause and `mid-supervisor.sh`, plus the Windows 11 `snc_mid` service |
| C. [Linux for `services.msc` people ↗](https://github.com/eltonaguiar/ServiceNowJDBC#part-c-where-the-service-is) | *Where is the service?* — the four layers (systemd `docker.service` → container `sn-mid` → `mid-supervisor.sh` → `java`), **how boot autostart actually works** (`systemctl is-enabled docker` **and** `RestartPolicy=unless-stopped`), the optional `sn-mid.service` unit for a single `services.msc`-style row, a field-by-field Windows → Linux table (*Path to executable*, *Log On As*, *Startup type*, *Recovery*, *Event Viewer*), GUI routes, and why to re-run the setup script instead of hand-editing `config.xml` |

</details>

**[↑ Projects](#projects)** · next: **[#4 →](#4-servicenowscheduledexport)**

---

<details>
<summary><strong>4. ServiceNowScheduledExport <span>—</span> export sets, delta exports, BI ingestion</strong> &nbsp;—&nbsp; <em>collapsed by default, click to expand</em></summary>

<a id="4-servicenowscheduledexport"></a>

**Depends on [#2 ServiceNowMidServer](#2-servicenowmidserver)** — the export target is *File on MID Server*
(`win11-mid`, path `toronto/events`), so no MID means a `completed` history row and no file.

**Repo:** [https://github.com/eltonaguiar/ServiceNowScheduledExport](https://github.com/eltonaguiar/ServiceNowScheduledExport) · *System Export Sets, delta exports, and BI ingestion*

**In the repo:** [build order — the 7 pages we visited ↗](https://github.com/eltonaguiar/ServiceNowScheduledExport#a-build-order--the-pages-to-visit-in-order) · [when to use export sets (and when not) ↗](https://github.com/eltonaguiar/ServiceNowScheduledExport#when-to-use-export-sets--and-when-not) · [Qlik Sense delta alternatives ↗](https://github.com/eltonaguiar/ServiceNowScheduledExport#alternatives-for-qlik-sense--loading-only-the-deltas)

**Skills demonstrated:** System Export Sets (`sys_export_definition` → `sys_export_target` → `sys_export_set` → `scheduled_data_export`), delta exports and watermarks, conditional / pre-export / post-export scripts, CSV encoding and file-naming semantics, Power BI and Qlik Sense incremental ingestion patterns.

**The thing everyone gets wrong first:** the exported file name.

```
torontoevents_20261101050335.csv
└─────┬─────┘ └──────┬───────┘
   File name       yyyyMMddHHmmss  =  2026-11-01 05:03:35
```

It is ISO-8601 **basic** (`yyyyMMddHHmmss`, no separators, no timezone, 24-hour), appended before the extension by the *Append timestamp* checkbox. The format is **not configurable**. Worse, the stamp is rendered in the **instance/user time zone**, not UTC — proved on this instance by pairing `sys_export_set_run` rows to the files on disk (run at `13:03:35` UTC → file named `…_050335.csv`, exactly 8 h apart, because the default time zone was US/Pacific). One export run therefore shows three different times: `13:04:06` UTC in the run history, `08:04:06` on a Toronto wall clock, `050406` in the file name. Setting `glide.sys.default.tz` (or the run-as user's Time zone) is the one-line fix.

**What the repo covers**

| Section | Content |
|---|---|
| Filename | Decoding the stamp, the time-zone trap with evidence, and the four levers you actually have (time zone, prefix/suffix, no stamp, rename on the file host) |
| Architecture | The four-record model, and why a stale *definition* produces a perfectly valid file with the wrong columns |
| Live config | The real values pulled from this instance: file name, format, max rows 100 K, display values on, MID `win11-mid`, path `toronto/events`, daily 08:00, delta on `sys_updated_on` |
| Options reference | Every field of all four tables, what it does, and where to set it |
| Delta exports | `Updated` vs `Created`, the watermark, the documented limits (no database views; cannot also filter on `sys_updated_on`), and how to force a full re-export |
| Scripts | Conditional (skip a run when nothing changed), pre-export (validate), post-export (audit + alert), with working scripts and their hard limit — they run **server-side** and cannot touch the file on the MID |
| Recipes | Rename/sortable names, never land an empty file, date-partitioned folders, retention, weekly-full + daily-delta, alerting on silence, human-shaped vs machine-shaped exports |
| BI | Power BI (Folder connector, Windows-1252, `Group` by `sys_id` keeping max `sys_updated_on`, incremental refresh) and Qlik Sense (folder load, `Exists()` dedupe, incremental QVD) — with an incident-table example and the delta-vs-full cost table |
| Troubleshooting | 12 symptom → cause → action rows, including the header-only 172-byte file and CSV mojibake |

**The headline number:** on this instance a full export is 2,128,595 bytes and a "nothing changed" delta is **172 bytes** (header only). Delta for freshness, weekly full for truth, dedupe by `sys_id` — full accuracy at roughly 1–2 % of the bytes.

</details>

**[↑ Projects](#projects)** · next: **[skills →](#quick-skill-summary)**

---

## License

See individual repositories for license details. All projects are personal reference implementations built and tested against live (non-production) ServiceNow instances.

---

~~**Not a project — Appendix A (experimental only).** `ServiceNowDBIntegration` was taken out of the list above and is kept here, underneath, as a documented experiment on out-of-platform AI + Table API changes. Expand the block below for what was tried, the real pros, the disqualifying cons, and the verdict.~~

<details>
<summary><s><strong>Appendix A — EXPERIMENTAL ONLY: out-of-platform AI + Table API changes</strong></s> &nbsp;—&nbsp; <em>archived experiment · collapsed by default, click to expand</em></summary>

<a id="appendix-a--experimental-only-out-of-platform-ai--table-api-changes"></a>

<del>

> **Status: experimental. Not a recommended path for anything that must be promoted or supported.**
> This is `ServiceNowDBIntegration` ([repo](https://github.com/eltonaguiar/ServiceNowDBIntegration)), kept as a documented experiment: is it effective to have an admin-granted, *outside-ServiceNow* AI agent modify the platform and auto-generate reporting objects?

### What was tried

An external agent, holding admin credentials, drove the **Table API** directly from outside the platform — creating `sys_db_object` / `sys_dictionary` rows to stand up tables and columns, probing them with real data, and generating the reporting views behind them.

### Pros (real, and they are not small)

| Pro | Detail |
|---|---|
| **Speed** | Populating tables through the Table API was dramatically faster than clicking through the UI — a 21-field table plus its columns, seeded with real rows, in one script run instead of an afternoon of forms |
| **Repeatable** | The same spec re-runs safely (idempotent), so a rebuilt or expired instance can be stood back up exactly |
| **Verifiable** | A round-trip probe (write → read back → compare every value → delete) catches silent corruption that the UI would never show — e.g. `decimal` + `max_length=15` storing `43.667712` as `43.67` with no error |
| **Schema as code** | Tables, columns and shapes live in JSON specs and diffs, so they can be reviewed like any other change |
| **Cheap exploration** | Ideal for proving out a data model or a dashboard idea before anyone spends real platform time on it |

### Cons (why it is an appendix and not a project)

| Con | Detail |
|---|---|
| **Bypasses update sets** | This is the big one. Changes made over the Table API are **not captured in an update set**. They exist on the instance and nowhere else: nothing to review, nothing to promote to test/prod, nothing to retrieve if the instance is refreshed or expires |
| **No review trail** | There is no "who changed this metadata and why" record that a platform team expects. In a governed instance this alone disqualifies the approach |
| **Mistakes are hard to trace or undo** | If the agent created a column with the wrong type, or a table in the wrong scope, there is no update-set rollback. You reconstruct what happened from the script log — if you kept one |
| **Invisible drift** | An instance with API-made metadata silently diverges from the configuration stored in source control. The next refresh wipes it and nobody notices until a report breaks |
| **Incomplete coverage** | Some things simply cannot be done this way — ACL creation over REST returns `403 "ACL Exception Insert Failed due to security constraints"` even for `security_admin`, because it needs an **elevated UI session** |
| **Admin blast radius** | It runs as an admin, outside the platform, with no guardrails: a bad script is a production change with no approval step |
| **Auto-generated reports are the weakest case** | Reports and dashboards are exactly the objects a stakeholder will ask to change tomorrow, and exactly the objects that must be promotable — so they are the worst fit for an untracked path |

### The rule that came out of it

**Use AI + the Table API for speed and exploration on a throwaway/dev instance; capture the steps and make the real change in a way that lands in an update set.**

For anything that must survive an instance refresh or be promoted:

1. Let the agent produce the **spec / the steps**, not the final state.
2. Perform (or re-perform) the change through the UI or a tracked mechanism, so an update set exists.
3. Or wrap the API calls in a **Scoped Application with an authored update set** and commit the XML to source control.
4. Keep the probe/verification script either way — that part was unambiguously worth it.

The experiment's honest verdict: **fast, clever, and not governable.** The round-trip verification idea is the piece worth keeping; the "let it write to the platform as admin" part is not.

</del>
</details>
