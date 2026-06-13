# Cross-Device Utterance Dedup

**Status:** SHIPPED 2026-06-12. Live-verified on Jetson + `dawn-kitchen` (Tier 1 RPi satellite),
both directions. Archived design + as-built reference.
**Original design:** 2026-06-06. **Plan review:** 2026-06-12 (master-plan-reviewer — no Critical
gaps). **Code review:** 2026-06-12 (five-agent, no Critical/High; fixes folded in — see §Review fixes).

---

## Problem

When the local Jetson mic and a nearby satellite both hear one spoken command, each session
independently queries the LLM → **duplicate responses**. Observed live (2026-04-25): local +
`dawn-kitchen` both processed "what has happened in April?" within ~1.5s, producing parallel tool
calls and two spoken answers. This is the most likely failure for a multi-device demo and the
highest-value reliability fix for it.

---

## Scope & key decisions

Shipped **only** the server-side dedup gate plus two satellite-side cleanups it surfaced. A
satellite-streaming redesign was explored and set aside (see §Considered and deferred).

- **Content-free event-window key.** Suppress a command when a *different source* produced a
  command event within a short window. **No transcript comparison** — which also catches what
  transcript-matching cannot: cross-engine ASR divergence (Pi Whisper vs Jetson Whisper) and a
  device whose ASR returns empty/garbage. This matches how the field does multi-device
  arbitration (Alexa ESP / Sonos are all signal/event-based, never transcript-based).
- **Optimistic FCFS.** First source to reach the check wins and **never waits**; later sources
  from other devices are dropped. Zero added latency on the responding path.
- **Same-session exemption.** Only a *different* session suppresses, so deliberate repeats on one
  device ("next, next") keep working.
- **Window:** `[asr] dedup_window_sec`, default **4s**, `0` disables, runtime-tunable.

### Accepted trade-offs / limitations (v1, documented not fixed)

1. **Two different people, two devices, one window → one suppressed.** Non-issue for a
   single-presenter use; the documented limitation for multi-user. Transcript confirmation would
   remove it — deferred.
2. **Non-co-located sessions.** A *remote* WebUI always-on session (user at a desk elsewhere,
   physically can't hear the room) is suppressed silently if it fires within the window. Per-user
   keying can't fix this — the local mic is `LOCAL_SESSION_ID` (user 0) while satellites map to
   real user IDs via `satellite_mappings`, so "suppress only same-user" would break the exact
   local-vs-kitchen case the feature exists for. Future precision upgrade: area-scoped suppression
   (satellites already carry `HomeAssistant_Area`), well short of transcript matching.
3. **FCFS is "fastest pipeline wins," not arbitrary arrival.** The local path is GPU ASR with no
   network hop; a Tier-1 Pi does CPU Whisper *then* a network round-trip — so the local Jetson
   almost always wins, even when the user addressed the kitchen satellite. Confirmed in live test
   (the Jetson won every shared utterance). A *systematic device priority*, not a coin flip.
4. **Skew grows with utterance length.** Batch Whisper on a Pi for a long (10–15s) utterance can
   put the loser's `check()` several seconds after the winner's, past a 4s window → dedup miss
   (extra double-response, fail-open, no harm beyond the original bug). The 4s default held for
   the utterances tested; revisit only if a real miss is observed.
5. **Wake-acknowledgment still duplicates.** Arbitration fires at command-complete time, so both
   devices still play their wake chime/greeting — only the spoken *answer* is deduplicated. Fixing
   this needs wake-time arbitration (the streaming redesign); out of scope.

---

## The module — `src/core/utterance_dedup.{c,h}` (Layer 1)

A **static singleton** (one global gate, four call sites across subsystems) — module-level static
slots + one mutex + monotonic clock + LRU. Conceptually similar to `src/core/rate_limiter.c` but
*not* a structural copy: rate_limiter is **instance-based** (caller-owned `rate_limiter_t`) with
`rand()` fallback eviction; this module is a singleton with LRU. Smaller — no hashing/text.

```c
void   utterance_dedup_init(int window_sec);             /* <=0 disables */
void   utterance_dedup_set_window(int window_sec);        /* live retune from set_config */
bool   utterance_dedup_check(uint32_t session_id);        /* true = suppress (a different source fired) */
void   utterance_dedup_set_clock(uint64_t (*fn)(void));   /* test seam; NULL = monotonic ms */
void   utterance_dedup_reset_all(void);                   /* test isolation */
```

Internals:
- Static slots (`UTTERANCE_DEDUP_SLOTS` = 16): `{ uint32_t session_id; uint64_t ts_ms; bool used; }`.
  `PTHREAD_MUTEX_INITIALIZER`, `static int s_window_sec`, `static uint64_t (*s_clock)(void)`.
- **Monotonic clock.** Default clock is `clock_gettime(CLOCK_MONOTONIC)` → milliseconds, **not**
  `time(NULL)`. A backward NTP step (common on devices that boot without network) would make a
  wall-clock `now - ts` go negative under unsigned math so slots never expire — the gate would
  *silently eat user commands*. Monotonic never steps backward, and ms resolution removes the ±1s
  edge slop a whole-second window would have. Window comparison: `now_ms - ts_ms >= window_ms`,
  where `window_ms = s_window_sec * 1000`. The `set_clock` seam takes `uint64_t (*)(void)`
  returning ms; the injected clock **must** be monotonic non-decreasing (documented on the seam —
  the unsigned subtraction depends on it).
- **`check(session_id)`** under the mutex:
  1. If `s_window_sec <= 0` → return false (disabled).
  2. `now = s_clock()`. Purge slots with `now - ts_ms >= window_ms`.
  3. If any live slot has `session_id != caller` → **return true (suppress); do NOT record.**
     Anchor to the first/winning event; not recording suppressed events prevents a suppressed
     device from later suppressing the winner's own deliberate repeat. Emit an activity-log line on
     suppress so the multi-user false-positive rate is measurable in production.
  4. Else record `(session_id, now)` — **update the caller's existing slot if present, else claim
     an empty/LRU slot** → return false (allowed). Updating in place keeps occupancy = number of
     *distinct* sources, so a chatty same-session repeat can't burn through slots.
- Same-session exemption falls out of step 3.
- **Pre-init safe.** With `PTHREAD_MUTEX_INITIALIZER` and `s_window_sec` static-zero, a `check()`
  that races ahead of `utterance_dedup_init()` is safely disabled (step 1) — fail-open is correct
  startup behavior.
- **Leaf-lock invariant:** the dedup mutex is acquired with **no session or global lock held**.
  Stated in the header and in `ARCHITECTURE.md`'s per-module lock list.
- **Slot saturation:** because suppressed events are not recorded, at most one winner holds a slot
  in any window, so the 16-slot array is defensive headroom; LRU eviction under >16 distinct
  concurrent in-window sources would at worst drop a slot (a rare extra double-response, never a
  crash — fail-open).

### Walkthrough
- A fires `T` → no other session in window → allow, record A@T.
- B fires `T+1` → finds A@T (different) → suppress, don't record.
- C fires `T+2` → finds A@T (still in window) → suppress.  → only A proceeds.
- A fires again `T+2` (deliberate) → finds only its own event → allow. ✓ repeat works.

---

## Hook points — at each voice-command entry, after the supersede check, before history-add

`session_id` is the discriminator: `LOCAL_SESSION_ID` (0) for local mic, nonzero per webui/sat.
The module never sees text — each call site passes only `session->session_id` and logs its own
transcript. Init in `src/dawn.c` `main` next to `wake_word_init()`:
`utterance_dedup_init(g_config.asr.dedup_window_sec);`.

| Path | Location | On suppress |
|---|---|---|
| **Local mic** | `src/dawn.c`, top of `DAWN_STATE_PROCESS_COMMAND`, after the empty-command early-out and **before the processing-mode branch** (covers `direct_first`/`direct_only`, not just LLM mode) | log + transcript, `free(command_text)` + NULL, return to listening via `reset_for_new_utterance(...)`, `break` |
| **Tier 1 satellite** | `src/webui/webui_satellite.c`, after supersede, before `session_add_message` | log, **`satellite_send_stream_end(session,"complete")`** then `satellite_send_state(session,"idle")`, `goto cleanup`. The stream_end is mandatory — see §Satellite cleanups. |
| **Tier 2 / WebUI PTT** | `src/webui/webui_audio.c`, after supersede, before `session_add_message` | log, `webui_send_state(session,"idle")`, `session_release` + `free(transcript)` + `free(work)` + `return NULL` (mirrors the adjacent supersede cleanup). Bare `"idle"` is sufficient for the browser — its pre-existing LLM-fail path does the same. |
| **WebUI always-on** | `src/webui/webui_always_on.c`, both `webui_process_text_input` sites | log, `free(cmd)` (transcript detached from `ctx` under the mutex — the hook still owns it), then **`always_on_processing_complete(ctx)`** — NOT a hand-rolled "send idle." The wake+command branch sets `ALWAYS_ON_PROCESSING` before the dispatch; skipping the dispatch without resetting state would wedge the session in PROCESSING. Mirrors the existing ASR-fail path (which emits the correct `listening` state). Call is on the LWS thread after the ctx mutex is released — leaf-lock holds. |

---

## Satellite cleanups (surfaced in live test 2026-06-12)

Two Tier-1-satellite issues that the suppress path exposed. Both are in `dawn_satellite/` and need
a satellite rebuild + OTA to take effect; the daemon change alone is not enough.

### 1. Stream-end release — satellite spun to its 50s timeout

The satellite enters `VOICE_STATE_WAITING` after sending a `satellite_query` and exits it **only**
when `on_stream_callback(is_end=true)` sets `response_complete` — i.e., on a `stream_end` message.
The first cut of the suppress path sent only `satellite_send_state(session,"idle")`; on the
satellite the `"idle"` state callback merely updates `last_server_activity`, which *restarts* the
50s response timeout. The satellite spun for the full timeout, then gave up. Fix: the suppress path
sends `satellite_send_stream_end(session,"complete")` (safe to call with no active stream — it just
queues the end message; empty response buffer = nothing spoken) to release WAITING immediately,
then `"idle"` for the NeoPixel. The browser (Tier 2/PTT) and always-on paths don't need this — they
exit on `"idle"`/`always_on_processing_complete` respectively.

### 2. Deferred transcript display — deduped utterance left a dangling user line

The satellite optimistically wrote the captured command to `ctx->user_text` and the SDL transcript
showed it immediately. A deduped turn produced no response, leaving a "You: …" line with no reply.
Rather than remove it after the fact, the satellite now **withholds the user line until the server
confirms it's processing** (`voice_processing.c`): capture stores the text and sets a new
`user_text_pending` flag; `on_state_callback` reveals it (sets `user_text_new`) when the server's
`"thinking"` state arrives — and `"thinking"` is sent **only after** the daemon's dedup check
passes (`webui_satellite.c`). A deduped turn never gets `"thinking"` (only `stream_end`+`"idle"`),
so its line never appears. `atomic_exchange` makes the reveal a one-shot (robust against repeated
`"thinking"`); keyboard/text mode is unaffected (it doesn't use the `user_text` path and never sets
the pending flag). Trade-off: the user's own line now appears after the `"thinking"` round-trip
(~tens of ms on LAN) instead of instantly — imperceptible on a normal turn, exactly "never appears"
on a deduped one.

---

## Config — `int dedup_window_sec` in `asr_config_t`

Default `ASR_DEDUP_WINDOW_SEC_DEFAULT` (4), range 0–`ASR_DEDUP_WINDOW_SEC_MAX` (60), 0 disables,
runtime-tunable. Both bounds are named constants in `include/config/dawn_config.h` (JS schema
mirrors them with a sync comment, since JS can't include the C header). Plumbed end-to-end:
`config_defaults.c`, `config_parser.c` (`known_keys` + `PARSE_INT`), `config_validate.c`
(`VALIDATE_RANGE_INT 0..MAX`), all `config_env.c` sites (ENV override, every printf dump,
`config_to_json`, `config_write_toml` — the last is the repo's known "drop-on-save" trap),
`www/js/ui/settings/schema.js` (asr section), `src/webui/webui_config.c` (asr apply block +
live `utterance_dedup_set_window(...)`), and `dawn.toml.example`. It is deliberately **not** in
`s_restart_required_fields` — it retunes live.

**WebUI set-config clamp.** `handle_set_config` does not run `config_validate`, so the apply path
clamps `dedup_window_sec` to `[0, ASR_DEDUP_WINDOW_SEC_MAX]` (via the existing `CONFIG_CLAMP`) before
applying and persisting. Without the clamp an admin could persist an out-of-range value that then
makes the daemon fail to boot (startup `config_validate` is fatal).

---

## Test & verify

- **Unit** `tests/test_utterance_dedup.c` (Unity; `LABEL ci`), injectable clock + `reset_all`
  between cases: cross-session suppress; same-session repeat allowed; three devices → only first
  proceeds; window expiry; disabled when window=0; winner-anchor walkthrough; half-open window
  boundary (+3999ms suppressed, +4000ms allowed); chatty same-session keeps its anchor; saturation
  no-crash; live retune; `reset_all`. 11 cases, all pass; full CI 79/79.
- **Live E2E (2026-06-12, Jetson + dawn-kitchen):**
  - Both hear one command → exactly one spoken answer (Jetson wins); satellite logs
    `Dedup: suppressing session N`, `Satellite: Dedup suppressed…`, `Stream end … reason=complete`
    and returns to listening immediately (no spin); **no user line shown on the satellite.**
  - Local muted, satellite only → normal turn, response streamed, user line displayed on
    `"thinking"`. Unaffected.

---

## As-built / history

Shipped in two logical phases, both in the single ship commit:
- **Module + config + tests (inert):** the singleton, full config plumbing, unit suite — daemon
  behavior unchanged until hooked.
- **Hooks + satellite cleanups + E2E:** the four daemon hooks + init, then the two satellite-side
  fixes the live test surfaced (stream-end release, deferred transcript display).

Code-review fixes folded in before ship: monotonic-clock invariant documented; WebUI set-config
clamp; default/ceiling promoted to named constants. Skipped (with reason) by triage: multi-user
cross-tenant suppression (by-design, documented limitation 2; not weaponizable — session_id is
server-assigned), test-seam exposure (not network-reachable), `@file`/`extern "C"` style nits
(match closest peers).

### Critical files
- `src/core/utterance_dedup.{c,h}` (new), `tests/test_utterance_dedup.c` (new)
- `src/dawn.c` (init + local-mic hook), `src/webui/webui_satellite.c`, `src/webui/webui_audio.c`,
  `src/webui/webui_always_on.c`
- `include/config/dawn_config.h`, `src/config/{config_defaults,config_parser,config_validate,config_env}.c`,
  `src/webui/webui_config.c`, `www/js/ui/settings/schema.js`, `dawn.toml.example`
- `dawn_satellite/src/voice_processing.c` (deferred display), `dawn_satellite/include/satellite_version.h` (2.1.0)
- `CMakeLists.txt`, `tests/CMakeLists.txt`, `ARCHITECTURE.md` (leaf-lock list)

---

## Considered and deferred

### Transcript-matching dedup (fuzzy)
Hash + word-overlap match across transcripts. Rejected as the primary key because the local mic
(Jetson Whisper) and a Tier 1 satellite (Pi Whisper) produce *different* transcripts for the same
utterance, so an exact match misses and the design would rest entirely on a fuzzy threshold (~0.85)
calibrated against real local-vs-Pi pairs — fragile. Content-free sidesteps this. Transcript
confirmation remains the clean way to remove the multi-user false-positive *later*, if needed.

### Satellite-streaming redesign (Tier 1 features + server-side ASR)
Idea: satellites stream audio to the server so all ASR runs on the Jetson → uniform transcripts
(exact dedup) and audio server-side (enables "loudest wins"). **Set aside after architecture +
efficiency review:**
- **The latency justification does not hold.** The end-of-speech *silence wait* dominates
  wake-to-response (local mic 1200ms; always-on 1500ms), not ASR compute — so centralizing onto
  the always-on path *adds* ~300ms. ASR transit was never the bottleneck.
- **It concentrates contention.** The shared ASR pool is 4 Whisper contexts; multiple satellites
  waking on one room-wide command serialize, and an overflow caller can stall up to 5s.
- **Mode A (continuous always-on) is costly** — a per-connection Silero VAD on the Jetson 24/7 per
  satellite + ~2.88 MB buffer + ~256 kbit/s continuous TX, scaling linearly with N.
- Reviewers concluded the **dedup module alone fixes the bug** with no firmware and no latency risk.

If revived, justify on **dedup precision + unlocking loudness, never on latency.** Then: gated
streaming (Mode B) as default, not always-on (Mode A); fix the capability-gated routing
(`webui_process_text_input` gates TTS on `conn->tts_enabled`, not `capabilities.local_tts`); make
the always-on per-user uniqueness check per-device.

### "Loudest device wins" arbitration
The natural winner heuristic (loudest ≈ closest ≈ intended), but: raw RMS isn't comparable across
heterogeneous mics (XVF3800 / ESP32 MAX9814 / RPi / browser AGC) without per-device gain
calibration; and picking the loudest forces holding the first response ~window to compare (adds
latency) unless a pre-ASR "claim" message is added. Depends on the redesign (audio must be
server-side). Deferred; RMS math can be lifted from `src/audio/aec_calibration.c`.
