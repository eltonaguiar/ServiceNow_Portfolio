# ServiceNow Portfolio

A curated collection of ServiceNow development, integration, and infrastructure projects. Each repository below was built and validated against live instances — no hypothetical code.

---

## Projects

| Project | Focus | Status | Link |
|---|---|---|---|
| `servicenow_scriptincludesampleBETTER` | GlideAjax client-server scripting pattern | Complete | [servicenow_scriptincludesampleBETTER](https://github.com/eltonaguiar/servicenow_scriptincludesampleBETTER) |
| `ServiceNowDBIntegration` | Python REST API integration — schema creation, table creation, round-trip verification | Complete | [ServiceNowDBIntegration](https://github.com/eltonaguiar/ServiceNowDBIntegration) |
| `ServiceNowMidServer` | ARM64 MID Server (NVIDIA GX10 / Grace Blackwell, Raspberry Pi, Apple Silicon) | Complete | [ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer) |

---

## 1. servicenow_scriptincludesampleBETTER — Client-Server Scripting with GlideAjax

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

## 2. ServiceNowDBIntegration — Python REST API for Schema & Data

**Skills demonstrated:** ServiceNow Table API, `sys_db_object` (tables), `sys_dictionary` (columns), Import Set REST API, REST message scripting, ACL management, field type precision handling, idempotent schema migrations, CI/automation integration.

This repository brings an external database into ServiceNow programmatically rather than through point-and-click. It uses **Python standard library only** — no `pip install` required.

### The core insight
ServiceNow's metadata model is just three surfaces:

```
sys_db_object   <- one row per TABLE      ("here is a cupboard")
sys_dictionary  <- one row per COLUMN     ("this cupboard has a shelf for u_name")
/api/now/table/ <- the actual DATA        ("here are the things on the shelf")
```

Creating a table is an `INSERT` into `sys_db_object`. Creating a column is an `INSERT` into `sys_dictionary`. Once you see that, schema management becomes scriptable.

### What's in the repo

| File | Purpose |
|---|---|
| `sn_api.py` | Minimal Table API client — no dependencies, reads auth from env vars |
| `create_table.py` | Creates a table **and all columns** from a JSON spec. Idempotent. |
| `verify_table.py` | Writes a probe row, reads it back, compares every value, deletes it |
| `create_acl.py` | Attempts ACL creation over REST, then explains why the platform blocks it |
| `specs/u_toronto_event.json` | A complete 21-field worked example (validated on a live instance) |

### The precision trap (documented with reproduction)
A `decimal` column declared with `max_length=15` silently stores `43.667712` as `43.67` — no error, no warning. `max_length` controls precision for numeric types, not just length. The fix: use `float` with `max_length=40`. **You only discover this by round-tripping real data** — `verify_table.py` exists specifically to catch this class of bug, and it exits with code 1 on any drift.

### Five ways to get data into ServiceNow (decision framework)

| | Remote Table | JDBC pull | JSON pull | REST push | Custom probe |
|---|---|---|---|---|---|
| Real table (ACLs, reports, relations) | ❌ | ✅ | ✅ | ✅ | ✅ |
| Needs MID Server | ❌ | ✅ | optional | ❌ | ✅ |
| Live vs snapshot | **live** | snapshot | snapshot | snapshot | snapshot |
| Effort | high | high | medium | **low** | high |
| Beginner friendly | ❌ | ❌ | ✅ | **✅** | ❌ |

**Bottom line:** If you can write a Python script, push data via REST (Option 4). It's the shortest path from A to B and the easiest to debug.

### ACL creation is UI-only
Creating ACLs over REST returns `HTTP 403 "ACL Exception Insert Failed due to security constraints"` — even for a user holding `security_admin`. The `sys_security_acl` table requires an **elevated session**, which is a browser/UI concept the REST API has no equivalent for. `create_acl.py` detects this, explains it, and prints the 3 UI steps that work.

---

## 3. ServiceNowMidServer — ARM64 MID Server on Unconventional Hardware

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
- **Data integration** — Push (REST Import Set API), Pull (JDBC), and Remote Table patterns
- **ACL and security** — the platform restriction on REST ACL creation, minimum viable roles, credential isolation
- **MID Server deployment** on non-standard architectures (ARM64), including Java version management and native launch
- **Plugin management** — ITOM Discovery installation, monitoring, and verification
- **Infrastructure as code** for ServiceNow metadata — idempotent, dry-run-capable scripts
- **Validation engineering** — round-trip data verification to catch silent precision loss

---

## License

See individual repositories for license details. All projects are personal reference implementations built and tested against live (non-production) ServiceNow instances.