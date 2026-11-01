# ServiceNow Portfolio

A curated collection of ServiceNow development, integration, and infrastructure projects. Each repository below was built and validated against live instances — no hypothetical code.

---

## Projects

| Project | Focus | Status | Link |
|---|---|---|---|
| `glideajax-scriptinclude-onchange-incident-warning` | GlideAjax Script Include + onChange client script - warns when an assignee already has more than 10 active incidents | Complete | [glideajax-scriptinclude-onchange-incident-warning](https://github.com/eltonaguiar/glideajax-scriptinclude-onchange-incident-warning) |
| `ServiceNowJDBC` | Getting at the data and keeping the plumbing alive — events DB (MySQL) queries, MID Server runbook for Linux + Windows, beginner guide to where the "service" and its config actually live | Complete | [ServiceNowJDBC](https://github.com/eltonaguiar/ServiceNowJDBC) |
| `ServiceNowScheduledExport` | System Export Sets — scheduled CSV exports to a MID Server, delta exports, conditional/pre/post scripts, Power BI / Qlik consumption | Complete | [ServiceNowScheduledExport](https://github.com/eltonaguiar/ServiceNowScheduledExport) |
| `ServiceNowMidServer` | ARM64 MID Server (NVIDIA GX10 / Grace Blackwell, Raspberry Pi, Apple Silicon) | Complete | [ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer) |

> **Not a project — Appendix A (experimental only):** `ServiceNowDBIntegration` moved out of the list above; it is kept as a documented experiment on out-of-platform AI + Table API changes: [Appendix A](#appendix-a--experimental-only-out-of-platform-ai--table-api-changes).

---

## 1. glideajax-scriptinclude-onchange-incident-warning — GlideAjax Script Include + onChange Client Script

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

---

## 2. ServiceNowJDBC — Getting at the Data, and Keeping the Plumbing Alive

**Skills demonstrated:** MySQL/JDBC data access against the events database, SQL diagnostics (counts, duplicates, freshness, per-source/per-month rollups), Docker and systemd operations, MID Server lifecycle management, root-cause analysis of silent failures, runbook writing for people new to Linux.

**What it answers:** the two questions that come up the moment an integration goes quiet — *"is the data actually in the database?"* and *"is the thing that feeds it still running?"*

### Part A — the events database

| | |
|---|---|
| Host | `mysql.50webs.com:3306` (the web host itself sees it as `localhost`), MySQL 8.0.32 |
| Database / user | `ejaguiar1_events` / `ejaguiar1_events` |
| Main table | **`events_log`** — one row per event, ~12.2 K rows, unique key on `event_id` |
| Bookkeeping tables | `event_pulls` (one row per ingest run, with `Inserted/Updated/Errors` notes), `event_sources`, `event_title_index`, `stats_summary` |

The README carries a copy-paste query set — total vs distinct rows, freshness (`MAX(last_updated)`), upcoming volume, per-month and per-source rollups, null dates, duplicate detection — plus the two gotchas that bite every time: the date column is **`event_date`**, not `date`, and `categories`/`tags` are **JSON stored as text** (use `JSON_CONTAINS`, not `=`).

### Part B — the MID Server runbook (Linux + Windows)

The MID on this estate runs **wrapper-less** in Docker on ARM64 (GX10). That has one nasty consequence, and the README leads with it: when the instance tells the MID to restart itself, the JVM logs *"Wrapper-less MID Server cannot be automatically restarted… MANUALLY START THE MID SERVER TO CONTINUE"* and exits — while `docker ps` still reports the container as `Up`, because PID 1 is the log `tail`, not the JVM. The instance shows `Status: Down` while the box looks healthy.

The fix is one command, and the README says so before explaining anything else:

```bash
docker restart sn-mid
```

then verification that actually proves recovery (new JVM PID, `MID Server is connected to instance host`, `Operational State=UP`, and the `ecc_agent` API showing `status: Up / validated: true`). Beyond that it covers stale pid files, keystore/`401` repairs, volume ownership, relaunching the supervisor, an optional watchdog, restoring from a dead MID, and the Windows 11 equivalent (`snc_mid` service, wrapper path, `sc query` / `net stop` / `net start`).

### Part C — "where is the service?" for people new to Linux

A section written for someone whose only mental model is `services.msc`: a field-by-field mapping (service name → container, *Path to executable* → `docker inspect … {{.Path}} {{.Args}}`, *Log On As: Local System* → `Config.User` / the `mid` user, *Startup type: Automatic* → `RestartPolicy=unless-stopped`), the four layers (`systemd → docker.service → container → supervisor → java`), GUI routes (Docker Desktop, System Monitor, `sudo -E nautilus` on the volume), and how to change config safely — re-run the setup script for credentials rather than hand-editing, because hand-editing is what produces the keystore/401 failures.

**Companion:** `ServiceNowMidServer` is the build/installation repo (how to get an ARM64 MID running); `ServiceNowJDBC` is the operating repo (how to tell whether it is running and how to fix it).

---

## 3. ServiceNowScheduledExport — Export Sets, Delta Exports, and BI Ingestion

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

---

## 4. ServiceNowMidServer — ARM64 MID Server on Unconventional Hardware

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

---

## Quick Skill Summary

Across all projects, the ServiceNow skills demonstrated include:

- **Client-server integration** via GlideAjax and the Table API
- **Schema management** (`sys_db_object`, `sys_dictionary`) — creating tables and columns programmatically
- **Data integration** — Push (REST Import Set API), Pull (JDBC), Remote Table patterns, and scheduled file exports
- **Data egress** — Export Sets, delta exports with watermarks, pre/post scripts, and BI ingestion (Power BI / Qlik)
- **SQL diagnostics** — row counts, duplicates, freshness, and per-dimension rollups against the integration database
- **ACL and security** — the platform restriction on REST ACL creation, minimum viable roles, credential isolation
- **MID Server** deployment on non-standard architectures (ARM64) *and* operations (wrapper-less restart behaviour, keystore/401 repair, service-vs-container mental model)
- **Plugin management** — ITOM Discovery installation, monitoring, and verification
- **Infrastructure as code** for ServiceNow metadata — idempotent, dry-run-capable scripts *(experimental — see Appendix A)*
- **Validation engineering** — round-trip data verification to catch silent precision loss

---

## Appendix A — EXPERIMENTAL ONLY: out-of-platform AI + Table API changes

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

---

## License

See individual repositories for license details. All projects are personal reference implementations built and tested against live (non-production) ServiceNow instances.
