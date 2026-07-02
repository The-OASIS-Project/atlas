# STAT System Telemetry — Design & Implementation

**Status: SHIPPED (2026-07-02, branch `stat-telemetry-tool`).** Gives DAWN's assistant (Friday) on-demand access to the host's live hardware telemetry — temperatures, battery, CPU/memory load, cooling fan — and its downsampled history ("how hot did the GPU get overnight?"). Consumes the external **STAT** daemon's MQTT feed; stores history in its own SQLite file; exposes one LLM tool.

**Ship commits (branch `stat-telemetry-tool`):**
- `4924bf4` — `feat(stat)`: the subsystem itself (ingest service + self-contained DB + `system_status` tool + wiring + 6 unit tests).
- `ee50533` — `fix(llm)`: companion tool-description truncation fix (see §9). STAT surfaced a pre-existing bug where long tool descriptions were truncated (and mid-UTF-8 cuts crashed the WebSocket); this fixes the whole tool-schema path, not just STAT.
- `211368f` — `feat(webui)`: surfaces the exact `tools[]` schema in the System-Prompt debug inspector (the validation surface for §9) + auth-gates that endpoint.

**Key files:**
- `src/core/stat_service.c` / `include/core/stat_service.h` — MQTT ingest, in-memory latest-value cache, rollup accumulators, history flush. **Layer 1 (core)** — a leaf service, like `component_status.c`.
- `src/core/stat_db.c` / `include/core/stat_db.h` — self-contained `stat.db` (own handle/mutex/WAL): bucket insert, SQL-aggregated history query, retention prune.
- `src/tools/stat_tool.c` / `include/tools/stat_tool.h` — the LLM descriptor + callback (**Layer 3**).
- `tests/test_stat_service.c` — 6 Unity tests (ingest → fold → flush → SQL-aggregation round-trip).
- Wiring: `cmake/DawnTools.cmake` (`DAWN_ENABLE_STAT_TOOL`), `src/tools/tools_init.c`, `src/mosquitto_comms.c`, `src/auth/auth_maintenance.c`, `dawn.toml.example` (`[stat]`).

**Unit tests:** 6 tests in `tests/test_stat_service.c` (topic gating, never-seen staleness, live-cache latest-value, accumulator fold + SQL aggregation, empty-bucket skip, multi-bucket aggregation). Part of `tests-ci`.

**Manual tests:** live-verified on Jetson — Friday sees `system_status`, calls `action=all`, returns real telemetry (`Battery 100% (19.9 V, 0.70 A, 13.9 W), discharging, ~14h 42m remaining, health NORMAL. Temperature: SoC 64.6 °C. Load: CPU 12%, memory 46%. Fan 4685 rpm (77% load).`). History/trend path is unit-tested; live end-to-end (real accumulated buckets) pending ≥1 real 15-min flush.

---

## Context — why this exists

STAT (**System Telemetry and Analytics Tracker**, `~/code/The-OASIS-Project/stat`) is the OASIS "diagnostic heartbeat": a C daemon on the Jetson/ARK carrier that samples I2C/sysfs sensors (INA238/INA3221/Daly BMS, thermal zones, fan tach) and **publishes over MQTT**. It is deliberately *broadcast-only* — it persists nothing and offers no query surface.

Before this feature, Friday could not answer "how's the system doing — temps, battery, load?" or "how hot did it get overnight?" The data existed on the wire; nothing consumed it. This subsystem makes DAWN a consumer: it ingests the STAT feed, caches live values, keeps a downsampled history, and hands both to an LLM tool.

**The organizing decision (developer-chosen):** keep the store **DAWN-side**. DAWN was going to subscribe to the STAT feed for live values anyway, so it becomes the single writer of history too — rather than adding a query/store responsibility to STAT (which fights its broadcast-only design) or spinning up a third daemon that re-consumes the same firehose.

---

## What STAT publishes (the wire contract)

Transport is MQTT only. Two topics:

| Topic | QoS | Retained | Content |
|---|---|---|---|
| `stat/telemetry` | 0 | no | All metric types, discriminated by a `type` field inside an OCP v1.4 envelope. STAT emits **one publish per metric type each cycle** (~5–8 msg/s at the default 1 s interval). |
| `stat/status` | 1 | **yes** (+ Last-Will) | `{device, msg_type:"status", status:"online"\|"offline", timestamp}` |

Every telemetry payload carries `{device:"stat", msg_type:"telemetry", type:<subtype>, timestamp:<epoch-ms>}`. DAWN consumes three `type`s (the headline set):

- **`SystemMetrics`** → `cpu_usage`, `memory_usage`, `system_temp` (SoC junction temp — there is no separate GPU sensor on this platform).
- **`Fan`** → `rpm`, `load`, `pwm`.
- **`BatteryStatus`** (the *unified* one — preferred over the raw INA238/Daly messages to avoid triple-counting) → `voltage`, `current`, `power`, `battery_level`, `temperature`, `charging_state`, `time_remaining_min`, `battery_status` (OK/WARNING/CRITICAL), `status_reason`, and `critical/warning/info_fault_count`.

`Battery`(raw)/`SystemPower`(INA3221)/`BatteryHealth` are ignored (per-rail power is a possible future `power` action).

---

## Architecture

```
STAT daemon ──MQTT──> stat/telemetry (~5-8/s, QoS0) + stat/status (retained/LWT, QoS1)
                          │
   mosquitto_comms.c on_connect(): subscribe;  on_message(): route "stat/*"
       ── stat_service_handle_mqtt()   [routed ABOVE the per-message INFO log — see §7]
                          │
   src/core/stat_service.c  (Layer 1 leaf)
     • json-c length-bounded parse (payload not NUL-terminated)
     • latest-value cache  ── mutex-guarded, TTL-stale (component_status idiom)
     • rollup accumulators ── per family: min/max/sum/count
                          │
   src/auth/auth_maintenance.c  (900s maintenance thread, nice(10))
     → stat_history_flush(): copy+reset accumulators under the cache lock,
       then (lock-free of the cache) write one wide bucket row + prune +
       passive WAL checkpoint on stat.db's OWN handle
                          │
   src/tools/stat_tool.c  (Layer 3)
     system_status callback: live actions read the cache snapshot;
     history/trend actions SQL-aggregate stat.db → bounded text for the LLM
```

### Tool-family split (service / db / tool)

This is not "just a tool" — ingestion is a *continuous, stateful* concern (a persistent MQTT subscription, a cache that outlives any one call, a background writer). So it follows DAWN's established tool-family split, the same shape as the ECHO phone integration:

- **`stat_service`** = ingest + cache + accumulators + flush (business logic/state).
- **`stat_db`** = persistence.
- **`stat_tool`** = the LLM-facing descriptor + callback.

**Layer placement — the deliberate asymmetry.** `stat_service`/`stat_db` live in **`src/core/` (Layer 1)**, not `src/tools/`, because they are true *leaves* (deps: only `logging`, `dawn_error`, `core/path_utils`) and Layer-2 modules call them (`mosquitto_comms` → `stat_service_handle_mqtt`; `auth_maintenance` → `stat_history_flush`). Putting them in `tools/` (Layer 3) would make those **upward** L2→L3 edges. Their exact analog `component_status.c` (the daemon's own MQTT status/TTL service) lives in core for the same reason. The LLM descriptor `stat_tool.c` stays in Layer 3.

Contrast `phone_service.c`, which stays in `tools/` despite also being MQTT-ingest: it depends on `messaging_engine` (L3), `tts`, `contacts_db`, `session_manager` — an *orchestration node*, not a leaf. Moving it to core would create worse (core→L3) violations. **Rule: leaf ingest services → core; orchestration services → tools.**

All three files are gated by `DAWN_ENABLE_STAT_TOOL` in `cmake/DawnTools.cmake` (they're appended to `TOOL_SOURCES` from their `core/`+`tools/` paths), so the whole subsystem is compile-time removable.

---

## Live cache & staleness

A module-static struct in `stat_service.c`, guarded by one mutex (`s_mutex`), holds the latest value of each headline metric plus `last_seen` and an `online` flag. Ingest runs on the MQTT callback thread; the tool callback runs on a worker thread → the tool reads via a **copy-out snapshot under the lock** (`stat_service_get_snapshot`), then computes staleness outside the lock.

- **Staleness:** a read older than `stale_after_sec` (config, default 30) reports "no fresh telemetry" (via `TOOL_RESULT_ERROR_MARK`) rather than quoting stale numbers — the `component_status.c` TTL idiom.
- **Online/offline:** driven directly by the retained `stat/status` topic + Last-Will; receiving any telemetry also implies online (`mark_seen_locked`).

`s_active` (enabled + initialized) is an `atomic_bool` — it's written by the main thread at init/shutdown and read by the MQTT + maintenance threads.

---

## History store — downsampled, self-contained `stat.db`

Raw ingest (~5–8 msg/s) is **never stored**. Instead the service keeps in-memory **rollup accumulators** (per family: min, max, sum, count) updated on every sample. Every `STAT_FLUSH_INTERVAL_SEC` (currently `== AUTH_MAINTENANCE_INTERVAL_SEC`, 900 s), the maintenance thread calls `stat_history_flush()`:

1. Lock the cache mutex, **copy + reset** the accumulators into a bucket row, unlock.
2. Write **one wide bucket row** to `stat.db`, prune rows older than `history_retention_days` (default 30), passive WAL checkpoint — all on stat.db's own handle, lock-free of the cache.

Because min/max are computed from *every* sample seen in the bucket, **peak detection is exact** even though the bucket time-resolution is 15 min — which is what makes "how hot did it get overnight?" accurate. Empty buckets (STAT offline) are skipped. The 900 s cadence adds **zero new threads** (it rides the existing `nice(10)` maintenance loop, off every hot path); a comment at the call site pins the granularity==interval coupling.

### Why a dedicated `stat.db` (not shared `auth.db`)

Both the architecture and embedded reviews converged on this, and it directly serves the "cleanly removable" goal:

- **Removability:** no schema-version bump, no `SCHEMA_SQL` edit, no `auth_db_internal` coupling, and **no dead table left behind** on removal.
- **Semantics:** telemetry is **host-scoped, not per-user** — it doesn't belong in the per-user auth DB.
- **Lifecycle:** owning the open/close **dissolves the auth-DB ordering hazards** (auth.db opens ~470 lines *after* tools register and closes *before* tools clean up; a shared handle would be closed under `stat_service_shutdown`'s final flush). Template: `src/audio/music_db.c`.

### Schema (`stat_samples`)

Wide row per bucket, following the `session_metrics` time-series convention (INTEGER epoch, `REAL` samples, `(bucket_start DESC)` index). No FK to `users`.

```sql
CREATE TABLE IF NOT EXISTS stat_samples (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  bucket_start INTEGER NOT NULL,               -- unix epoch (seconds)
  cpu_usage_avg REAL, cpu_usage_max REAL,
  mem_usage_avg REAL, mem_usage_max REAL,
  system_temp_avg REAL, system_temp_min REAL, system_temp_max REAL,
  batt_level_avg REAL, batt_level_min REAL, batt_level_max REAL,
  batt_voltage_avg REAL, batt_power_avg REAL, batt_temp_max REAL,
  fan_rpm_avg REAL, fan_rpm_max REAL,
  sys_count INTEGER NOT NULL, batt_count INTEGER NOT NULL, fan_count INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_stat_samples_time ON stat_samples(bucket_start DESC);
```

**Per-family sample counts** (`sys_count`/`batt_count`/`fan_count`), not a single count: STAT polls battery on a separate cadence, so families arrive at different rates; per-family counts keep cross-bucket weighted re-aggregation correct. Steady-state at 30-day/15-min retention ≈ 2,880 rows.

Statements are prepared per-call (write = 1 insert / 15 min, query = human cadence — caching is pointless and would recouple to core teardown). WAL, `SQLITE_OPEN_FULLMUTEX`.

---

## The `system_status` tool (LLM-facing)

Config-bearing `tool_metadata_t`. Capabilities `TOOL_CAP_SCHEDULABLE | TOOL_CAP_INFORMATIONAL`, `is_getter=true`, **no `TOOL_CAP_NETWORK`** (reads a local cache/DB). `INFORMATIONAL` means a scheduled "battery check" auto-promotes into a Friday-summarized briefing, like weather.

Params:
- `action` (enum, required): `all` (full status report) · `temps` · `battery` · `performance` (live, from cache) · `history` · `trend` (from stat.db). **Note:** `trend` is currently an alias of `history` — both emit the same min/max/avg aggregate (`stat_tool.c` routes them to the same branch). A true per-bucket trend *series* is not yet implemented (§12).
- `period` (optional): `"last hour"`, `"overnight"`, `"today"`, `"24h"`, `"7d"` — a purpose-built duration-window parser (not `time_query_parser`, which is point/midpoint-oriented for retrieval scoring, not duration windows).
- `metric` (optional): narrow history to `temperature`/`battery`/`cpu`/`memory`/`fan`.

**Bounded output:** `history` (and its `trend` alias) aggregates **in SQL** (`MIN`/`MAX`/sample-weighted-`AVG` + peak-bucket lookup) → a fixed-size result regardless of window, so a 7-day window can't blow up the LLM-facing string. (If a true per-bucket `trend` series is added later, it must cap the emitted point count — see §12.)

Config `[stat]` (in `dawn.toml`): `enabled`, `telemetry_topic`, `status_topic`, `db_path` (default `/var/lib/dawn/stat.db`), `stale_after_sec`, `history_retention_days`. Tool-owned config, like `phone`/`weather`/`home_assistant` — deliberately *not* in the WebUI `SETTINGS_SCHEMA` (which maps only to the global config struct); a dedicated admin panel is the correct future UI surface.

**What Friday can/can't answer.** Covered: "how's power/battery", "how long till it dies" (`time_remaining_min`), "are we running hot", "is everything nominal / full diagnostic" (health/faults are cached), "how hot overnight" (exact bucket peak + time). Bounded honestly: "GPU temp specifically" (single SoC junction temp — the description says so), "what's drawing the most power" (per-rail, deferred). **The one thing a read-only tool can't do: speak *unprompted* ("sir, battery's low")** — that's a future proactive-alert composition with the scheduler/notification path; the live cache built here is exactly what such a watcher would poll.

---

## §7. Threading, lock ordering, lifecycle

- **Two leaf locks, never nested:** `stat_service`'s cache mutex (`s_mutex`) and `stat_db`'s own mutex. `stat_history_flush` copies+resets under the cache lock, releases, *then* does DB I/O under the DB lock — the ARCHITECTURE.md "copy out, release, then process" rule. Touched by three threads (MQTT ingest, worker snapshot, maintenance flush); no inversion is reachable.
- **MQTT ordering:** tools register (`.init` → `stat_service_init`) *before* mosquitto connects, so the service does no MQTT at init. Subscriptions are centralized in `on_connect`; dispatch in `on_message`; the mosq handle is never cached at init. `stat_service_is_active()` gates the subscribe.
- **Shutdown ordering (race-free):** `auth_maintenance_stop()` joins the flushing thread, `mosquitto_loop_stop()` stops ingest, and only then does `tool_registry_shutdown()` → `stat_service_shutdown()` flush the final partial bucket and close stat.db. No live thread exists when the handle closes.

---

## §8. Security — untrusted-MQTT hardening

`stat/telemetry` + `stat/status` are parsed from **untrusted** MQTT input (any authorized broker publisher). The 5-agent review (§10) drove this hardening:

- **Length-bounded parse** via `json_tokener_parse_ex(tok, payload, len)` — mosquitto payloads aren't NUL-terminated; no `%s` on raw payload; `json_object_put` on every path.
- **`isfinite()` guard** in `json_get_double`: `inf`/`nan` from a publisher are rejected before they reach the accumulators, the DB, or the `double→int` cast in the formatter.
- **Complete-sample folding:** a `SystemMetrics` folds into the average only when all three fields are present (they share `sys_n` as divisor); battery folds only when `battery_level` is present — a partial/garbage message can't skew a bucket.
- **Control-char stripping** on the free-text `status_reason` (and the short enum-ish fields) before it's surfaced into the LLM tool result — it's MQTT-sourced text.
- **`fmt_duration_min` upper-bound** guards the `double→int` cast against a huge finite `time_remaining_min` (UB).
- **Parameterized SQL** throughout; the one `%s`-built query (`query_extreme_bucket`, column name — can't be a bound parameter) is pinned by an allowlist `assert` + comment (internal literals only).
- **Bounded tool output** (SQL aggregation + trend cap).
- **Auth gate (commit `211368f`):** the WebUI `get_system_prompt` debug endpoint was reachable pre-auth and now also emits the tools schema — added the missing `conn_require_auth`, matching `handle_get_config`.

The internal MQTT broker uses TLS + auth, so `status_reason` prompt-injection requires an authenticated publisher (low), but the field is stripped defensively regardless.

---

## §9. Companion fix — tool descriptions were being truncated (commit `ee50533`)

STAT surfaced a pre-existing, general bug. The `system_status` description (~700 B with an em-dash) exposed it, but it affected many tools:

- `llm_tools.c` copied each tool's `.description` into a fixed **512-byte** buffer before sending it to the LLM, silently truncating long descriptions (`scheduler` ~5.8 KB) — and when the cut split a multi-byte UTF-8 character, the orphaned byte was **invalid UTF-8 that crashed the WebUI WebSocket frame** (browser: "Could not decode a text frame as UTF-8" → 1006 → reconnect loop → Tool-Calling list never rendered). The MCP `cbm` server's em-dash-laden descriptions were the original trigger.
- **Fix:** tool descriptions are now read **from the registry at schema-generation time** (`tool_effective_description`), the same source parameters already use — so nothing is clipped. MCP descriptions are UTF-8-sanitized at ingest (`mcp_schema_wrap_description`), and `TOOL_DESC_MAX` was raised 512→2048 so verbose MCP servers (cbm ~1.7 KB) fit. The dead `tool_registry_generate_llm_schema` (unreferenced) was deleted.
- **Validation surface (commit `211368f`):** the System-Prompt debug inspector now renders the exact `tools[]` array the LLM receives, so truncation is visible before/after.

**Trust-based asymmetry:** compiled-in (trusted, authored) descriptions are delivered in full (`const char*` literal, unbounded); untrusted **MCP** descriptions stay capped at `TOOL_DESC_MAX` — a deliberate prompt-injection-surface + token-budget policy.

---

## §10. Review outcome

Full 5-agent review of the branch (architecture, embedded-efficiency, security, UI, coding-standards): **0 critical.** Fixed: 1 HIGH (a CI test build break from the dead-code deletion — two tests exercised the deleted duplicate generator, removed with zero live-coverage loss), 3 MEDIUM (auth gate; `SUCCESS`/`FAILURE` return; collapsible a11y), and 11 LOW (the §8 hardening + the core/ move + atomic `s_active` + comments). Skipped with rationale: tokener reuse (marginal), token divisor (cosmetic), config-struct duplication (deliberate split), `!= 0`→`!= SUCCESS` (deferred project-wide sweep), MCP enum allowlist (over-engineering).

Post-review, all `tests-ci` (96 tests) pass, both build configs (tool ON/OFF) are clean.

---

## §11. Removability

Compile-time gated. To remove: revert the four wiring edits (`DawnTools.cmake` block, `tools_init.c` register, `mosquitto_comms.c` subscribe+dispatch, `auth_maintenance.c` flush hook), delete the three tool-family files + their headers + the test, and delete `stat.db`. **Nothing residual** — no dead table, no schema-version artifact, no auth.db coupling.

---

## §12. Future work (not shipped)

- **Proactive telemetry alerts** — a background threshold-watch (battery/temp) that fires via the scheduler/notification path so Friday warns *without being asked*. The natural Phase 2; the live cache here is what it would poll.
- **Per-rail power** — a `power` action over STAT's `SystemPower`/INA3221 for "what's drawing the most?".
- **True `trend` series** — `trend` currently aliases `history` (a single min/max/avg + peak aggregate); the tool/param descriptions were aligned to say exactly that (doc-accuracy review, 2026-07-02). Optional future work: implement a bucketed, point-capped time series for `trend` distinct from `history`'s single aggregate.
- **Finer history granularity** — the 15-min bucket is the maintenance-interval piggyback; a config'd finer cadence (own timer) if telemetry ever needs sub-15-min resolution.
- **Tool-registry deep-copy** (tracked in `docs/TODO.md`) — the registry shallow-copies metadata (retains the caller's `.rodata` pointers), a blocker for the future loadable-`.so` tool modules. The `tool_effective_description` registry-read path (§9) is aligned with, and rides on, that eventual change.
