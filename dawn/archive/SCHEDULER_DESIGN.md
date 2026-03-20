# Scheduled Events: Timers, Alarms, Reminders, Scheduled Tasks

**Status**: ✅ Fully Implemented (2026-02-19) | Satellite + WebUI cross-client tested (2026-02-25)
**Unit Tests**: 94 assertions across 16 tests (`tests/test_scheduler.c`)
**Manual Tests**: Sections 1-6 verified (daemon, voice creation, sound, dismiss/snooze, satellite overlay, WebUI notifications, cross-client sync)
**Key Files**: `src/core/scheduler.c`, `src/core/scheduler_db.c`, `src/tools/scheduler_tool.c`, `include/core/scheduler.h`, `common/src/audio/chime.c`

## Context

DAWN lacks a fundamental voice assistant feature: timers, alarms, and reminders. Every commercial assistant (Alexa, Google, Siri) supports these. Research shows a clear baseline that ALL assistants provide (named timers, recurring alarms, snooze, time-remaining queries), plus differentiators DAWN can uniquely offer: room-aware announcements via DAP2 satellites, scheduled tool execution ("turn off lights at midnight"), and multi-user event ownership.

The existing infrastructure is well-suited: SQLite for persistence, tool_registry for LLM integration, session_manager for user/satellite identity, TTS for announcements, and `pthread_cond_timedwait` for efficient scheduling.

## Feature Set

### Baseline (what all commercial assistants have)
- Multiple simultaneous **named timers** ("set a pasta timer for 12 minutes")
- **Time remaining queries** ("how much time is left on the pasta timer?")
- **Alarms** at specific times ("set an alarm for 7 AM")
- **Recurring alarms** (daily, weekdays, weekends, weekly, custom days)
- **Reminders** with custom messages ("remind me to call Mom at 3pm")
- **Snooze** with configurable duration (default 10 min, "snooze for 5 minutes")
- **Cancel by name** ("cancel the pasta timer")
- **List active events** ("what timers are running?")

### DAWN Differentiators
- **Room-aware announcements**: Timer fires on the satellite that set it (stored by UUID). If satellite is disconnected, WebUI notification only (no fallback to daemon speaker — it's likely in a server closet). Optionally "announce everywhere."
- **Scheduled tool execution**: "Turn off the lights at midnight" stores a tool call that auto-fires. Unique among open-source assistants.
- **Multi-user isolation**: Events tied to `user_id`. "List my timers" is per-user.
- **Missed event recovery**: If daemon restarts, pending events that should have fired are handled on startup (fire reminders/tasks, skip/reschedule alarms).

## Architecture

```
                    ┌──────────────────────────────────────┐
                    │           LLM Tool Call              │
                    │  "Set a pasta timer for 12 minutes"  │
                    └──────────┬───────────────────────────┘
                               │ scheduler_tool_callback()
                               ▼
┌──────────────────────────────────────────────────────────┐
│                   scheduler_db.c                         │
│         SQLite CRUD (scheduled_events table)             │
└──────────┬──────────────────────────────────┬────────────┘
           │ INSERT                           │ SELECT next
           ▼                                  ▼
┌──────────────────────────────────────────────────────────┐
│                   scheduler.c                            │
│    Background thread: pthread_cond_timedwait loop        │
│    Wakes at next fire_at, fires due events               │
└──────┬────────┬──────────┬──────────────────┬────────────┘
       │        │          │                  │
       ▼        ▼          ▼                  ▼
   Local TTS  Satellite  WebUI Toast    Tool Execute
   (speaker)  (by UUID)  (broadcast)    (scheduled task)
```

## Database Schema (v18 migration)

**Table: `scheduled_events`**

| Column | Type | Purpose |
|--------|------|---------|
| `id` | INTEGER PK | Auto-increment ID |
| `user_id` | INTEGER FK | Owner (from session metrics) |
| `event_type` | TEXT | `timer`, `alarm`, `reminder`, `task` |
| `status` | TEXT | `pending`, `ringing`, `fired`, `cancelled`, `snoozed`, `missed`, `dismissed`, `timed_out` |
| `name` | TEXT | User-assigned name ("pasta timer") |
| `message` | TEXT | Reminder text or task description |
| `fire_at` | INTEGER | Unix timestamp to fire |
| `created_at` | INTEGER | When event was created |
| `duration_sec` | INTEGER | Timer: original duration (for "time remaining") |
| `snoozed_until` | INTEGER | New fire time when snoozed |
| `recurrence` | TEXT | `once`, `daily`, `weekdays`, `weekends`, `weekly`, `custom` |
| `recurrence_days` | TEXT | CSV for custom: "mon,wed,fri" |
| `original_time` | TEXT | HH:MM for recurring (to recalculate next) |
| `source_uuid` | TEXT | Satellite UUID (NULL = local) |
| `source_location` | TEXT | Room name at creation time |
| `announce_all` | INTEGER | 1 = announce on all devices |
| `tool_name` | TEXT | Scheduled task: tool to call |
| `tool_action` | TEXT | Scheduled task: action param |
| `tool_value` | TEXT | Scheduled task: value param |
| `fired_at` | INTEGER | When event actually fired |
| `snooze_count` | INTEGER | Number of times snoozed |

**Indexes**: `(status, fire_at)`, `(user_id, status)`, `(user_id, status, name)`, `(source_uuid)`

## Key Design Decisions

1. **LLM handles time parsing**: The tool description tells the LLM to convert "in 12 minutes" → `duration_minutes: 12` and "7 AM tomorrow" → ISO 8601. No NLP time parser in C. The tool callback resolves ISO 8601 to Unix timestamp using the user's configured timezone (from `user_settings.timezone` or `g_config.localization.timezone` as fallback). The user's timezone is injected into the tool description so the LLM produces timezone-aware ISO 8601 strings.

2. **UUID-based routing, not session_id**: Sessions are transient (destroyed on disconnect), but satellite UUIDs persist across reconnections. Store `source_uuid` in DB, look up the current session at fire time. The public `session_find_by_uuid()` must acquire the rwlock (read), find the session, call `session_retain()` to increment `ref_count`, release the rwlock, and return the retained pointer. The caller must call `session_release()` after use. If satellite is disconnected, skip audio and rely on WebUI notification only.

3. **Condvar-based sleep with CLOCK_MONOTONIC**: Thread consumes zero CPU when idle, wakes precisely when needed. `scheduler_notify_new_event()` re-triggers on new events. Use `pthread_condattr_setclock(&attr, CLOCK_MONOTONIC)` to avoid NTP clock jump issues. Store fire times as wall-clock (DB persistence) and compute monotonic offsets at runtime: `mono_fire = clock_monotonic_now + (wall_fire - wall_now)`.

4. **Tool lifecycle hooks**: `scheduler_init()` / `scheduler_shutdown()` called via tool_registry's `init` / `cleanup` callbacks. No changes to `dawn.c`.

5. **Separate DB module**: `scheduler_db.c` accesses `s_db` directly (like `auth_db_conv.c`), keeping `scheduler.c` focused on thread logic and announcement routing. New scheduler statements added AFTER `stmt_conv_set_private` in `auth_db_state_t` struct. Update `last_stmt_end` in `finalize_statements()` accordingly.

6. **Non-blocking alarm sound playback**: Alarm sound playback runs on a separate short-lived detached thread (or via TTS queue integration on satellite). The scheduler thread only triggers TTS, signals the sound thread, updates DB status, and returns to the condvar wait loop. A `scheduler_ringing_t` struct tracks active ringing events with stop flags.

7. **Scheduled task safety**: Only tools with `TOOL_CAP_SCHEDULABLE` flag can be scheduled. Default safe tools: `smartthings`, `volume`, `audio_tools`, `weather`. Never allow `shutdown`. Re-validate `tool_registry_is_enabled()` at execution time. Create a synthetic session with the original `user_id` for proper attribution. Log all scheduled task executions.

8. **Authorization on all dismiss/snooze paths**: Server-side handler checks `event.user_id` against `conn->auth_user_id` (WebUI) or satellite's associated user (DAP2). Exception: alarms ringing on a specific satellite can be dismissed from that satellite regardless of owner (physical presence implies authorization).

9. **User_id filtering from Phase 1**: All list/cancel/query operations filter by `user_id` from the start. No deferred multi-user isolation.

10. **C enums for event fields**: Use C enums for `event_type`, `status`, `recurrence` in the in-memory struct. Convert at the DB read/write boundary. Enables switch-case dispatch instead of strcmp.

## Files To Create

### Daemon (server-side)
| File | Purpose | Est. Lines |
|------|---------|------------|
| `include/core/scheduler.h` | Public API: init, shutdown, notify, list, time_remaining, dismiss | ~80 |
| `include/core/scheduler_db.h` | DB layer: CRUD operations, event struct, enums | ~100 |
| `src/core/scheduler_db.c` | SQLite prepared statements, all queries | ~400 |
| `src/core/scheduler.c` | Thread, condvar loop, fire/announce/execute, alarm sound gen | ~650 |
| `include/tools/scheduler_tool.h` | Tool registration header | ~15 |
| `src/tools/scheduler_tool.c` | LLM tool callback: create/list/cancel/query/snooze/dismiss | ~450 |

### Satellite (SDL UI + alarm sound)
| File | Purpose | Est. Lines |
|------|---------|------------|
| `dawn_satellite/src/ui/ui_alarm.c` | Alarm overlay: render, dismiss/snooze buttons, notice banner | ~350 |
| `dawn_satellite/src/ui/ui_alarm.h` | Alarm overlay public API | ~40 |

### WebUI
| File | Purpose | Est. Lines |
|------|---------|------------|
| `www/js/ui/scheduler.js` | Notification banner, dismiss/snooze, optional timer countdown | ~200 |
| `www/css/components/scheduler.css` | Notification banner + missed notice styling | ~80 |

### Tests
| File | Purpose | Est. Lines |
|------|---------|------------|
| `tests/test_scheduler.c` | Unit tests for scheduler_db functions and string conversions | ~500 |
| `tests/test_scheduler_stub.c` | Stub providing `s_db` global for test linking | ~30 |

## Files To Modify

### Daemon
| File | Changes |
|------|---------|
| `include/auth/auth_db_internal.h` | Bump `AUTH_DB_SCHEMA_VERSION` 17→18, add ~10 prepared stmt pointers |
| `src/auth/auth_db_core.c` | Add `scheduled_events` CREATE TABLE, v18 migration, prepare/finalize stmts |
| `include/config/dawn_config.h` | Add `scheduler_config_t` struct + field in `dawn_config_t` |
| `src/config/config_defaults.c` | Default scheduler config values |
| `src/config/config_parser.c` | Add `parse_scheduler()` for `[scheduler]` TOML section |
| `include/tools/tool_registry.h` | Bump `TOOL_PARAM_MAX` 8→12, add `TOOL_CAP_SCHEDULABLE` flag |
| `cmake/DawnTools.cmake` | Add `DAWN_ENABLE_SCHEDULER_TOOL` option |
| `CMakeLists.txt` | Add `scheduler.c`, `scheduler_db.c` to `ENABLE_AUTH` block |
| `src/tools/tools_init.c` | Register scheduler tool |
| `include/core/session_manager.h` | Add `session_find_by_uuid()` declaration |
| `src/core/session_manager.c` | Implement `session_find_by_uuid()` (public wrapper for existing static function) |
| `include/webui/webui_server.h` | Add `WS_RESP_SCHEDULER_EVENT` response type |
| `src/webui/webui_server.c` | Handle scheduler_dismiss/snooze messages, serialize scheduler notifications |
| `src/webui/webui_server.c` | Add `[scheduler]` to config get/set handlers (same pattern as `[mqtt]`, `[network]`) |

### Satellite
| File | Changes |
|------|---------|
| `dawn_satellite/src/ui/sdl_ui.c` | Add alarm overlay render call, touch dispatch for dismiss/snooze |
| `dawn_satellite/src/ws_client.c` | Handle `scheduler_notification` messages from daemon, send dismiss/snooze |
| `dawn_satellite/CMakeLists.txt` | Add `ui_alarm.c` |

### WebUI
| File | Changes |
|------|---------|
| `www/index.html` | Include `scheduler.js` and `scheduler.css` |
| `www/js/ui/settings/schema.js` | Add `scheduler` section with all `[scheduler]` config fields |
| `www/js/ui/settings/config.js` | Add scheduler config get/set handling (follows existing pattern) |

### Tests
| File | Changes |
|------|---------|
| `tests/CMakeLists.txt` | Add `test_scheduler` target and include in `tests` aggregate |

## Tool Definition

Increase `TOOL_PARAM_MAX` from 8 to 12 in `tool_registry.h` (compile-time constant, no dynamic allocation impact). This allows discrete typed parameters instead of opaque JSON, which improves LLM reliability significantly.

```c
// Actions: create, list, cancel, query, snooze, dismiss
static const treg_param_t scheduler_params[] = {
   { .name = "action", .type = TOOL_PARAM_TYPE_ENUM, .required = true,
     .maps_to = TOOL_MAPS_TO_ACTION,
     .enum_values = {"create", "list", "cancel", "query", "snooze", "dismiss"},
     .enum_count = 6 },
   { .name = "details", .type = TOOL_PARAM_TYPE_STRING, .required = false,
     .maps_to = TOOL_MAPS_TO_VALUE,
     .description = "JSON: {type (timer|alarm|reminder|task), name, "
                    "duration_minutes (1-43200), fire_at (ISO 8601, future, within 1 year), "
                    "message (max 512 chars), recurrence (once|daily|weekdays|weekends|weekly|custom), "
                    "recurrence_days (csv: mon,tue,...), announce_all (bool), "
                    "tool_name (schedulable tools only), tool_action, tool_value, "
                    "event_id (for cancel/query/snooze/dismiss), snooze_minutes (1-120)}" },
};
// Aliases: "timer", "alarm", "reminder", "schedule"
```

**Note on `details` as JSON string**: While discrete parameters would be ideal, the scheduler needs 13+ fields across different actions. The JSON string approach keeps the parameter count at 2 (action + details). The tool callback validates all fields strictly — see Input Validation below.

**Input validation in `scheduler_tool_callback()`** (mandatory, Phase 1):
- `duration_minutes`: must be in [1, 43200] (max 30 days)
- `fire_at`: must be valid ISO 8601, in the future, within 1 year
- `snooze_minutes`: must be in [1, 120]
- `name`: truncate to 128 chars
- `message`: truncate to 512 chars; sanitize if re-injected into LLM context
- `event_type`: must match enum (timer, alarm, reminder, task)
- `recurrence`: must match enum
- `recurrence_days`: validate CSV format, known day names, no duplicates
- `tool_name`: must exist in registry AND have `TOOL_CAP_SCHEDULABLE` flag
- Reject unknown JSON keys
- Enforce `max_events_per_user` via `SELECT COUNT(*)` before INSERT

**Session context in callback** (for room-aware routing):
```c
session_t *ctx = session_get_command_context();
if (ctx && ctx->type == SESSION_TYPE_DAP2) {
   strncpy(event.source_uuid, ctx->identity.uuid, ...);
   strncpy(event.source_location, ctx->identity.location, ...);
}
event.user_id = ctx ? ctx->metrics.user_id : default_user_id;
```

## Announcement Routing (when event fires)

1. Look up satellite session: `session_find_by_uuid(event->source_uuid)` — returns retained pointer, caller must `session_release()`. Check `session->disconnected` before use.
2. **Satellite connected, Tier 1** (local TTS): `satellite_send_response(session, text)` → satellite plays TTS then alarm sound. `session_release()` after send.
3. **Tier 2 satellites**: Not supported for alarms. Tier 2 audio path is request-response only; unsolicited audio push + local alarm loop would require significant new ESP32 code. Tier 2 users get WebUI notification only.
4. **Satellite disconnected**: Skip audio — no fallback to daemon speaker (likely in server closet, useless). WebUI notification (step 6) provides coverage.
5. **Local mic source** (`source_uuid` is NULL): `text_to_speech(text)` on daemon speaker, then play alarm sound on detached thread via `audio_stream_playback_write()`
6. **`announce_all` set**: collect `{session_id, tier, uuid}` tuples under brief rwlock read, release rwlock, then iterate list using `session_get()` (does its own locking with retain). Skip Tier 2 satellites (unsupported). Do NOT hold rwlock during TTS generation or WebSocket I/O.
7. **WebUI**: always broadcast notification toast to all connected WebUI clients (with browser audio chime)
8. **Dismiss/snooze concurrency**: Use optimistic UPDATE: `UPDATE scheduled_events SET status = 'dismissed' WHERE id = ? AND status = 'ringing'`. Check `sqlite3_changes()` — if 0 rows changed, event was already handled. Return generic "already handled" for both "not found" and "not your event" to prevent enumeration.

**Announcement text:**
- Timer: "Your pasta timer is done!" / "Timer complete!"
- Alarm: "It's 7:00 AM. Your alarm is going off."
- Reminder: "Reminder: call Mom"
- Task: "Scheduled task complete: turned off living room lights"

## Alarm Sound Effects

**Sound generation**: Synthesize a pleasant ascending 3-tone chime (C5-E5-G5, ~1s total) algorithmically at init time into a statically-allocated buffer. Three sine waves with ADSR envelopes — roughly 20 lines of code. No external WAV files, no licensing concerns.

- **Daemon**: Generate at 22050Hz (native audio backend rate)
- **Satellite**: Generate at 48000Hz (native ALSA rate) to avoid resampling overhead
- Sound parameters (frequencies, envelope, duration) can be tuned without recompiling large arrays

**Two sound variants** (same generator, different parameters):
- **Timer/reminder chime**: Short 3-tone ascending chime, plays once after TTS
- **Alarm tone**: Repeating attention-getting tone with **200ms silence gap** between repetitions (prevents auditory fatigue, aids room localization). Loops until dismissed or timeout.

**Global alarm audio manager**: Only ONE alarm sound plays at a time. If a new alarm fires while one is ringing, queue it. Hard-cap `alarm_timeout_sec` to maximum 300 seconds regardless of config.

**Playback flow when event fires** (on a separate sound thread, NOT the scheduler thread):
1. TTS announcement plays (e.g., "Your pasta timer is done!")
2. Wait for TTS completion (`tts_wait_for_completion()` on daemon, `tts_playback_queue_is_active()` on satellite)
3. Play alarm sound — once for timers/reminders, looping for alarms
4. Alarm loop continues until **dismissed**, **snoozed**, or **timeout**

**Satellite playback**: Integrate with `tts_playback_queue` rather than calling `audio_playback_play_wav()` directly. Push alarm PCM as a queue entry; for looping, re-push after each completion until dismissed. This coordinates with the single shared ALSA device (avoids contention with TTS and music). During alarm ringing, music is paused. New TTS interrupts alarm (for voice dismiss commands).

**Daemon playback**: New `scheduler_play_sound()` on a detached thread. Opens audio backend, writes PCM frames, loops with stop flag check. Reuses `audio_stream_playback_write()` from `audio_backend.h`.

**Tier 2 satellites**: Not supported for alarms (audio path is request-response only). WebUI notification provides coverage for Tier 2 users.

## Alarm Lifecycle: Ringing → Dismiss/Snooze → Timeout → Notice

**States for a fired alarm/timer:**
```
FIRED → RINGING (sound playing, UI showing)
  ├→ DISMISSED (user taps dismiss on satellite/WebUI, or says "stop"/"dismiss")
  ├→ SNOOZED (user taps snooze or says "snooze for 5 minutes")
  └→ TIMED_OUT (no interaction after alarm_timeout_sec, default 60s)
       └→ Leaves "missed" notice on satellite display + WebUI
```

**Dismiss mechanisms:**
- **Satellite touchscreen**: Tap "Dismiss" or "Snooze" button on alarm overlay
- **WebUI**: Click dismiss/snooze button in notification banner
- **Voice**: "Stop" / "Dismiss" / "Cancel alarm" during ringing (routed through normal command flow)
- **Auto-timeout**: After `alarm_timeout_sec` (default 60s), stop sound, mark as timed out, leave notice

## Satellite SDL UI: Alarm Overlay

**New file**: `dawn_satellite/src/ui/ui_alarm.c` / `ui_alarm.h`

**Animation**: Use **fade-in** entrance (opacity 0→1 over 200ms) with subtle **scale-up** (0.95→1.0) on the card. NOT slide-in — alarm is a system interruption, not a user-opened panel. Scrim fades in simultaneously (use existing `render_scrim()` formula, multiply by 180 instead of 150 for ~70% opacity to signal urgency). Dismiss animation: card fades out in 150ms (faster than entry for responsiveness).

**Event type visual differentiation**:
| Event Type | Header Color | Tone |
|-----------|-------------|------|
| Timer | Theme accent | Informational ("done") |
| Alarm | Amber `#F0B429` (WARNING color) | Urgent ("wake up") |
| Reminder | Theme accent | Moderate ("don't forget") |
| Task | Green `#22C55E` (SUCCESS color) | Confirmatory ("done for you") |

**Overlay design** (centered on full screen, card 420x240px at (512, 280)):
```
┌──────────────────────────────────────────────────────┐
│                  (scrim: ~70% black)                 │
│                                                      │
│     ┌──────────────────────────────────────────┐     │
│     │     [icon]  TIMER COMPLETE               │     │  ← Icon + type (largest)
│     │                                          │     │
│     │     Pasta Timer                          │     │  ← Event name (primary text)
│     │     12:00                                │     │  ← Time context (secondary)
│     │                                          │     │
│     │  ┌────────────────┐  ┌─────────────────┐ │     │
│     │  │    DISMISS     │  │     SNOOZE      │ │     │  ← 56px tall, 160px+ wide
│     │  │                │  │     10 min      │ │     │    16px gap between
│     │  └────────────────┘  └─────────────────┘ │     │
│     └──────────────────────────────────────────┘     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Touch targets**: Button height **56px**, minimum width **160px**, **16px gap** between buttons. Card minimum **420px wide x 240px tall**. These dimensions are functional requirements for groggy morning alarm dismissal on a 7" touchscreen.

**Pulsing animation during active ringing**: Gentle opacity pulse on the icon (0.7→1.0 at 1Hz via sine wave). For alarms, add amber border glow pulse on card edges. Pulse stops immediately on dismiss.

For alarms (recurring), show both Dismiss and Snooze. For timers (one-shot), show only Dismiss. Snooze button shows action label ("SNOOZE") with duration below in smaller secondary text ("10 min").

**After timeout/dismiss** — missed notice banner at **bottom of transcript area** (not top, to avoid conflict with settings panel):
  ```
  ┌────────────────────────────────────────────┐
  │ Missed: Pasta Timer (12:34 PM)        [✕]  │  ← 40px tall, amber left border
  └────────────────────────────────────────────┘
  ```
- Banner persists until user taps X or next interaction clears it
- Multiple missed: show only most recent with count badge ("2 missed alarms - Tap to view")

**Passive timer countdown**: When a timer is active for this satellite's UUID, show remaining time in the transcript header area: "Pasta 4:32" in `text_secondary` color. Updates every second. Significant UX differentiator over commercial assistants that require voice queries.

**Integration points:**
- `alarm_overlay` struct in `struct sdl_ui` with `{ visible, closing, anim_start }` + alarm-specific fields (event_id, event_type, name, can_snooze, ringing, cached textures with dirty flag)
- Renders AFTER settings/music panels in `render_frame()` — highest priority overlay
- Touch dispatch: alarm overlay intercepts all taps when visible (early return in `handle_gesture()` before panel dispatch)
- Alarm overlay suppresses screensaver: `ui_screensaver_activity()` on alarm fire, prevent screensaver render while `alarm_overlay.visible`
- Keep `sdl_ui.c` integration to ~20 lines (struct fields, render call, touch dispatch, screensaver guard)

**Texture caching**: Pre-render button textures and label textures on state change (not per-frame). Follow `build_white_tex()` + `SDL_SetTextureColorMod()` pattern. Rebuild only when `cache_dirty` flag is set (triggered by new alarm event, never in render loop).

## WebUI: Alarm Notification Banner

**New WebSocket message type**: `scheduler_notification`

**Server → Client when event fires:**
```json
{
   "type": "scheduler_notification",
   "event_id": 5,
   "event_type": "timer",
   "name": "Pasta Timer",
   "message": "Your pasta timer is done!",
   "status": "ringing",
   "timeout_sec": 60,
   "can_snooze": false
}
```

**Client → Server to dismiss/snooze:**
```json
{"type": "scheduler_dismiss", "event_id": 5}
{"type": "scheduler_snooze", "event_id": 5, "minutes": 10}
```

**WebUI display:**
- Notification banner slides down from top-center of page, `max-width: min(500px, 90vw)`
- Shows event name, dismiss button, snooze button (if alarm/reminder)
- Plays browser notification chime via Web Audio API (`OscillatorNode` frequency sweep, ~15 lines JS)
- `z-index: 1500` (above settings modal 1000, below toasts 2000)
- Ringing state: amber border + subtle box-shadow glow
- After timeout: banner changes to missed styling with muted border (`text-tertiary`)
- Live countdown for active timers (optional sidebar widget or status bar)
- Keyboard accessible: `role="alertdialog"`, `aria-live="assertive"`, Escape to dismiss, auto-focus Dismiss button
- Mobile (<600px): banner expands to full-width, buttons stack vertically if needed

**New JS module**: `www/js/ui/scheduler.js` — handles notification display, dismiss/snooze actions, optional timer countdown widget

## Scheduler Thread Logic

```
scheduler_thread():
   recover_missed_events()           // Handle events missed during downtime
   while (!shutdown):
      next = scheduler_db_next_fire_time()
      if next == 0:
         pthread_cond_wait(cond)      // No events, sleep indefinitely
      else:
         pthread_cond_timedwait(cond, next)  // Sleep until next event
      if timed_out:
         fire_due_events(now())       // Query + fire all events at/before now
```

**Missed event recovery** (on startup, after 30s subsystem init cooldown):
- Timers/reminders: fire immediately with "missed" prefix
- Alarms: if recurring → skip to next occurrence; if one-shot → mark as missed
- Tasks: controlled by `missed_task_policy` config (default `skip`):
  - `skip`: mark as missed, notify user (safest default)
  - `execute`: run the tool call if within `missed_task_max_age_sec` (default 300s)
  - Never auto-execute missed tasks for `TOOL_CAP_DANGEROUS` tools regardless of policy

**Event retention**: `scheduler_db_cleanup_old_events()` called on startup, deletes events where `status IN ('fired', 'cancelled', 'missed')` AND `fired_at < now() - event_retention_days * 86400`. Default 30-day retention.

## Config

```toml
[scheduler]
enabled = true                  # Master enable/disable
default_snooze_minutes = 10     # Default snooze duration
max_snooze_count = 10           # Auto-cancel after this many snoozes
max_events_per_user = 50        # Prevent runaway event creation per user
max_events_total = 200          # Hard cap across all users
missed_event_recovery = true    # Handle missed events on startup
missed_task_policy = "skip"     # "skip" (notify only) or "execute" (run if fresh)
missed_task_max_age_sec = 300   # Skip tasks older than this even if policy=execute
alarm_timeout_sec = 60          # Stop ringing after this (hard max: 300)
alarm_volume = 80               # Alarm sound volume (0-100, clamped at parse time)
event_retention_days = 30       # Clean up fired/cancelled/missed events after N days
```

## Voice Command Examples

| Command | Tool Call |
|---------|-----------|
| "Set a timer for 10 minutes" | create: {type:"timer", duration_minutes:10} |
| "Set a pasta timer for 12 minutes" | create: {type:"timer", name:"pasta", duration_minutes:12} |
| "How much time is left on the pasta timer?" | query: {name:"pasta"} |
| "Set an alarm for 7 AM every weekday" | create: {type:"alarm", fire_at:"07:00", recurrence:"weekdays"} |
| "Remind me to call Mom at 3pm" | create: {type:"reminder", message:"call Mom", fire_at:"15:00"} |
| "Turn off the lights at midnight" | create: {type:"task", tool_name:"smartthings", tool_action:"disable", tool_value:"lights", fire_at:"00:00"} |
| "What timers do I have?" | list: {type:"timer"} |
| "Cancel the pasta timer" | cancel: {name:"pasta"} |
| "Snooze for 5 minutes" | snooze: {snooze_minutes:5} |
| "Set a timer for 10 minutes on all devices" | create: {type:"timer", duration_minutes:10, announce_all:true} |

## Implementation Phases

### Phase 1: Core Infrastructure + Timers (~2-3 days)
DB schema (v18 migration) with all indexes including `(user_id, status, name)`. `scheduler_db.c` CRUD with prepared statements. `scheduler.c` thread + CLOCK_MONOTONIC condvar + fire logic + separate sound playback thread. `scheduler_tool.c` with create/list/cancel/query actions + full input validation. Config with all fields. CMake with `DAWN_ENABLE_SCHEDULER_TOOL`. `session_find_by_uuid()` with retain/release semantics. Algorithmic alarm chime synthesis at init. User_id filtering on all queries from day one. Timezone-aware ISO 8601 → Unix timestamp conversion. Event retention cleanup on startup. Deliverable: named timers work from local mic and satellites with TTS announcement + chime sound + room-aware routing.

### Phase 2: Alarms, Reminders, Recurrence, Snooze (~1-2 days)
Recurring alarm scheduling (daily/weekdays/weekends/weekly/custom) with DST-aware recalculation via `mktime()` with `tm_isdst = -1`. Reminder messages. Snooze action with custom duration (snooze updates `fire_at` directly, keeps `snoozed_until` as audit). Max snooze count enforcement. Looping alarm sound for alarms (vs single chime for timers) with 200ms gap between repetitions. Alarm timeout with auto-stop. Optimistic concurrency on dismiss/snooze (`WHERE status = 'ringing'` + check `sqlite3_changes()`). Batch `fire_due_events()` with `LIMIT 10` loop. Deliverable: full alarm/reminder lifecycle with proper ringing behavior.

### Phase 3: Satellite Alarm Overlay (~1-2 days)
New `ui_alarm.c` with fade-in animation (not slide), event type visual differentiation (amber for alarms), 56px-tall touch targets, pulsing icon animation during ringing. Dismiss/snooze buttons on SDL touchscreen. Alarm overlay renders on top of scrim (~70% opacity), suppresses screensaver. After timeout: missed notice banner at bottom of transcript area (not top). Passive timer countdown in transcript header. Handle `scheduler_notification` messages in `ws_client.c`. ALSA coordination: alarm integrates with `tts_playback_queue`, pauses music during ringing. Keep `sdl_ui.c` additions under 20 lines. Deliverable: satellite touchscreen shows alarm with interactive dismiss/snooze + passive countdown.

### Phase 4: WebUI Notifications + Settings (~1-2 days)
New `scheduler.js` with notification banner (top-center, z-index 1500), dismiss/snooze buttons with authorization checks, browser audio chime via Web Audio API. `scheduler_dismiss`/`scheduler_snooze` WebSocket messages with server-side `user_id` ownership verification. Ringing state: amber border + glow. Missed notices with muted styling. Keyboard accessible: `role="alertdialog"`, Escape to dismiss. Mobile responsive. Optional: live timer countdown widget. Add `scheduler` section to WebUI settings (`schema.js` + `config.js`) exposing all `[scheduler]` config fields with appropriate input types (toggles, number inputs with min/max, dropdowns for `missed_task_policy`). Server-side config get/set handlers in `webui_server.c`. Deliverable: browser shows alarm notifications with interactive dismiss + scheduler config in settings panel.

### Phase 5: Scheduled Task Execution (~1 day)
Add `TOOL_CAP_SCHEDULABLE` flag to `tool_registry.h`. Tag safe tools (smartthings, volume, audio_tools, weather). `scheduler_execute_task()` validates `TOOL_CAP_SCHEDULABLE` + `tool_registry_is_enabled()` at execution time. Create synthetic session with original `user_id` for proper attribution. Log all executions. Missed task policy: default `skip`, configurable `execute` with max age. Never auto-execute `TOOL_CAP_DANGEROUS` tools. Include tool result in announcement text. Deliverable: "turn off the lights at midnight" works safely.

### Phase 6: Polish (~1 day)
Announce-everywhere mode. Name-based cancel/query fuzzy matching. Voice dismiss during ringing: lightweight direct-command pattern in `text_to_command_nuevo` that intercepts "stop"/"dismiss"/"cancel alarm" before LLM when `scheduler_get_ringing()` returns active events (sub-second dismiss).

## Automated Tests

**Binary**: `tests/test_scheduler` (build: `make -C build-debug test_scheduler`, run: `./build-debug/tests/test_scheduler`)

**Files**: `tests/test_scheduler.c` (~500 lines), `tests/test_scheduler_stub.c` (~30 lines)

Uses an in-memory SQLite database via a stubbed `s_db` global — no auth_db machinery needed. The stub provides `auth_db_state_t s_db` with `PTHREAD_MUTEX_INITIALIZER`, and `setup_db()` creates the same DDL as the v18 migration plus minimal `users` rows. Each test function gets a fresh database (setup/teardown per test).

**94 assertions across 16 test functions:**

| Test | Assertions | Coverage |
|------|-----------|----------|
| `test_string_conversions` | 26 | All enum values for event_type, status, recurrence + unknown/NULL fallbacks + to_str round-trips |
| `test_insert_and_get` | 11 | Insert returns positive ID, sets id/created_at; get verifies all fields; nonexistent ID returns -1 |
| `test_insert_checked_limits` | 9 | Per-user limit (-2), global limit (-3), TOCTOU-safe atomic insert, count verification |
| `test_update_status` | 4 | pending→ringing→dismissed transitions verified via get |
| `test_update_status_fired` | 3 | Status + fired_at dual-field update |
| `test_cancel_optimistic` | 4 | Cancel pending succeeds; cancel already-cancelled or dismissed returns -1 |
| `test_dismiss_optimistic` | 5 | Dismiss pending fails; dismiss ringing succeeds with fired_at set; dismiss twice returns -1 |
| `test_snooze` | 7 | Snooze ringing→snoozed, fire_at updated, snooze_count increments, second snooze works |
| `test_due_events` | 2 | Past events returned, future excluded, ordered by fire_at ASC |
| `test_list_user_events` | 3 | Per-user filtering, type filter, cross-user isolation |
| `test_find_by_name` | 5 | Case-insensitive match, "%" not treated as wildcard, nonexistent returns -1, wrong user returns -1 |
| `test_count_events` | 2 | Cancelled events excluded from count_user and count_total |
| `test_get_ringing` | 2 | Returns only ringing events, correct event ID |
| `test_cleanup_old_events` | 3 | Old fired deleted, recent fired preserved, pending with old created_at preserved |
| `test_next_fire_time` | 2 | Returns earliest pending; after cancel returns next |
| `test_get_active_by_uuid` | 2 | Correct UUID returns matches, unknown UUID returns 0 |
| `test_get_missed_events` | 2 | Past pending returned, future pending excluded |

**What the automated tests cover:**
- All 19 `scheduler_db_*` functions
- All 3 string conversion pairs (type, status, recurrence)
- State machine correctness (optimistic cancel/dismiss/snooze)
- SQL safety (COLLATE NOCASE without wildcards)
- TOCTOU-safe atomic insert with limit enforcement
- Event retention/cleanup logic
- Multi-user isolation

**What the automated tests do NOT cover** (requires manual testing):
- Audio playback, chime generation, volume scaling
- SDL overlay rendering, animations, touch input
- WebSocket broadcast and WebUI notification banners
- Voice command end-to-end (ASR → LLM → tool → fire)
- Thread lifecycle (scheduler_init/shutdown, condvar timing)
- scheduler_tool.c callback (needs g_config + tool_registry)
- Recurring event rescheduling (mktime/DST)
- Missed event recovery on startup

## Manual Verification

**Last tested:** 2026-02-25

### 1. Daemon Startup & DB Migration ✅

- [x] Start daemon fresh — confirm no migration errors in log
- [x] Check log for `scheduler: initialized`, `generated chime PCM`, `generated alarm tone PCM`
- [x] Verify DB indexes: `sqlite3 data/dawn.db ".indexes scheduled_events"` → 4 indexes
- [x] Verify table structure: 21 columns matching schema

### 2. Event Creation via Voice/LLM ✅

- [x] **Timer**: "Set a 2-minute timer" — event created, fires after 2 min
- [x] **Named timer**: "Set a pasta timer for 12 minutes" — name column contains "pasta"
- [x] **Alarm**: "Set an alarm for [5 min from now]" — event_type=alarm, fires at requested time
- [x] **Reminder**: "Remind me to check the oven in 3 minutes" — event_type=reminder, message correct
- [x] **Invalid tool**: "Schedule a system shutdown at midnight" — returns schedulability error
- [ ] **Bad recurrence**: Duplicate days ("mon,mon") rejection — SKIPPED (hard to trigger via voice)
- [ ] **Limit check**: max_events_per_user enforcement — SKIPPED

### 3. Alarm Sound & Auto-Dismiss ✅

- [x] **Timer fires**: Chime plays (ascending C5-E5-G5), auto-dismisses after chime
- [x] **Reminder fires**: Same behavior as timer — auto-dismiss after chime
- [x] **Alarm fires**: Alarm tone loops (alternating A5/E5), stays ringing until user acts
- [x] **Alarm timeout**: After alarm_timeout_sec (60s), sound stops, DB status → timed_out

### 4. Dismiss & Snooze ✅

- [x] **Voice dismiss during ringing**: Sound stops immediately, DB status = dismissed
- [x] **Voice snooze during ringing**: Sound stops, alarm re-fires after snooze duration, snooze_count increments
- [ ] **Max snooze enforcement**: Auto-dismiss after max_snooze_count — DEFERRED

### 5. Satellite Overlay ✅

- [x] Timer fires on satellite → solid card overlay appears with DISMISS button
- [x] Alarm fires → overlay shows both DISMISS and SNOOZE buttons
- [x] Tap dismiss → overlay closes, sound stops, satellite sends `scheduler_action` to daemon
- [x] Tap snooze → overlay closes, alarm re-fires after delay
- [x] Screensaver suppresses while overlay is visible
- [x] Satellite chime plays through satellite speaker (common chime library)
- [x] Alarm sound thread auto-dismisses overlay after 120s timeout
- [x] Timer overlay persists through daemon auto-dismiss (satellite ignores "Auto-dismissed")
- [x] WebUI dismiss propagates to satellite overlay (server rebroadcasts "Dismissed")

### 6. WebUI Notifications ✅

- [x] Timer fires → notification banner slides down with DISMISS button + browser chime
- [x] Alarm fires → banner shows DISMISS + SNOOZE, amber border + glow animation
- [x] Click dismiss → banner closes, action sent to server
- [x] Keyboard: Escape dismisses banner
- [x] Alarm timeout → banner stays visible until user explicitly dismisses
- [x] WebUI dismiss of auto-dismissed timer → rebroadcast clears satellite overlay
- [x] Local dismiss suppression (WebUI doesn't re-show its own rebroadcast)
- [ ] Mobile responsive (<600px) — DEFERRED

### 7. Persistence & Recovery

- [ ] Timer survives restart — NOT TESTED
- [ ] Alarm recovery after restart — NOT TESTED
- [ ] Recurring alarm recovery — NOT TESTED

### 8. Recurring Events

- [ ] Daily alarm fires, next occurrence scheduled — NOT TESTED

### 9. Scheduled Tasks

- [ ] Tool execution on timer fire — NOT TESTED (requires SmartThings or similar)

### 10. Multi-User Isolation

- [ ] Separate event visibility per user — NOT TESTED
- [ ] Cancel isolation — NOT TESTED
- [ ] Dismiss authorization — NOT TESTED

### Cross-Client Synchronization ✅ (NEW — tested 2026-02-25)

- [x] Satellite receives `scheduler_notification` broadcasts (conn->authenticated set on registration)
- [x] Satellite `scheduler_action` (dismiss/snooze) authorized via satellite session bypass
- [x] NOT_FOUND response on dismiss race logged as warning (not error)
- [x] WebUI dismiss of already-auto-dismissed event → server rebroadcasts → satellite clears overlay
- [x] Satellite ignores daemon auto-dismiss and timed_out signals (overlay persists for user)
- [x] Ping/pong log noise suppressed on satellite

## Agent Review Summary

Plan reviewed by all four agents (architecture-reviewer, embedded-efficiency-reviewer, security-auditor, ui-design-architect). All critical and high findings have been incorporated into the plan above. Key changes from original plan:

**Critical fixes incorporated:**
- `session_find_by_uuid()` with retain/release semantics (prevents use-after-free)
- Alarm sound on separate thread (prevents blocking scheduler from firing other events)
- `TOOL_CAP_SCHEDULABLE` allowlist for scheduled task execution (prevents arbitrary tool calls)
- Authorization on dismiss/snooze (prevents cross-user alarm cancellation)
- Missed task policy defaults to `skip` (prevents stale/dangerous auto-execution)
- Fade-in animation for alarm overlay (distinguishes from user-initiated panels)
- Concrete touch target dimensions (56px height for morning alarm usability)

**High fixes incorporated:**
- Timezone handling in Phase 1 (user timezone injected into tool description)
- Full input validation on all JSON fields with sane ranges
- Single alarm audio manager (only one alarm sound at a time)
- Algorithmic PCM generation at native sample rate (eliminates 44KB static data + resampling)
- ALSA device coordination on satellite (integrate with tts_playback_queue)
- Event type visual differentiation (amber for alarms, accent for timers, green for tasks)
- Pulsing animation during active ringing
- Missed notice banner at bottom of transcript (avoids settings panel conflict)
- WebUI CSS specificity with z-index, border treatment, keyboard accessibility
- CLOCK_MONOTONIC for condvar (avoids NTP clock jump issues)
- User_id filtering from Phase 1 (not deferred to Phase 6)
- Optimistic concurrency on dismiss/snooze
- Event retention/cleanup mechanism (30-day default)
- Batch fire_due_events with LIMIT 10
- C enums for event fields in memory (convert at DB boundary)
