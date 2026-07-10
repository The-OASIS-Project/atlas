# Phone Call Audio — WebRTC APM (uplink + downlink), live tuning, WebUI

**Status:** SHIPPED — dawn `05e481b` (uplink APM) + `30d4359` (downlink APM + settings + live tuning), 2026-07. As-built record (consolidates the two working design docs, corrected to what actually shipped).

**Scope:** the host-side audio-processing layer for two-way cellular call audio — NOT the call/SMS state machine or the PCM transport (those are [PHONE_SMS_DESIGN.md](../../../dawn/docs/PHONE_SMS_DESIGN.md)), and NOT the wake-word echo canceller (that's [AEC_IMPLEMENTATION_GUIDE.md](AEC_IMPLEMENTATION_GUIDE.md) — a *separate* WebRTC APM instance; see §2).

---

## 1. Problem

The local-handset call bridge (`src/tools/phone_audio_bridge.c`) carries raw 16 kHz S16LE PCM between the SIM7600G-H modem's USB PCM tap (`/dev/ttyUSB4`) and the local mic/speaker in a single lockstep thread. Connectivity worked; *quality* didn't:

- **Uplink** (room mic → far end): "microphone-y / threshold-y / pumping" — a room mic driven through a static `tanh` gain, then re-gained by the cellular AMR codec's own AGC/VAD.
- **Downlink** (far end → speaker): soft for a quiet talker, clipping for a loud one.

**No modem-side fix exists.** The SIMCom *Audio Application Note* codec block diagram places all DSP (CTXVOL/CRXVOL, analog AGC, echo/NS, sidetone, CSDVC) inside the **NAU8810 analog codec**; the **USB PCM tap bypasses that codec entirely** — verified by a flat CRXVOL sweep on the PCM stream. The only USB-PCM-affecting AT command is `CPCMFRM` (sample rate, set to 16 kHz). So all level control is host-side, and the fundamental issue — *"a fixed gain can't win"* against the near-silence↔clipping span of real callers — needs AGC, not a constant.

The industry-standard fix is the **WebRTC Audio Processing Module (APM)** near-end chain (high-pass → noise-suppression → AGC2), which DAWN already vendors (`webrtc-audio-processing` v1.3).

---

## 2. Architecture — dedicated APM instances, not the wake-word singleton

`src/audio/aec_webrtc.cpp` is a **48 kHz singleton** owned by the wake-word capture path (AEC3 vs the TTS reference). The call audio uses **separate 16 kHz instances** (`src/audio/phone_apm.{cpp,h}`) — this is *required*, not convenient: the wake-word AEC runs on the capture thread *concurrently* with the call APM on the bridge thread, and `aec_webrtc.cpp` is a file-scope singleton whose own comment warns its state "must be moved to a per-stream context" before a second concurrent stream. Two independent `webrtc::AudioProcessing` instances (one uplink, one downlink) is the only thread-safe option; they share only the vendored library and the `AEC_BACKEND_WEBRTC` build flag.

`phone_apm.cpp` compiles unconditionally; all WebRTC use is fenced behind `#ifdef AEC_BACKEND_WEBRTC`, with `phone_apm_create()` returning `NULL` on non-WebRTC builds so the bridge degrades to the raw `tanh` soft-limiter.

```
mic ─▶ [uplink phone_apm: HPF→NS→AGC2(→AECM)] ─▶ modem uplink
modem downlink ─▶ [downlink phone_apm: HPF→AGC2] ─▶ speaker
                        (soft-limiter fallback if downlink APM off / no WebRTC)
```

---

## 3. As-built defaults (live-tuned by ear on real 2-party calls, 2026-07-09)

`phone_apm_default_config()` is the uplink default; `phone_audio_config_default()` derives the downlink from it with a few overrides.

| Knob | Uplink | Downlink | Notes |
|---|---|---|---|
| AGC2 (`agc`) | on | on (via `downlink_apm`) | the core level-normalizer |
| `fixed_gain_db` | **1.0** | **6.0** | uplink minimal (room mic close-ish; let AGC work); clamped **[0,49]** — see §5 |
| `agc_ramp_db_per_s` | 3.0 | 3.0 | lower = less pumping |
| `max_output_noise_dbfs` | -50 | -50 | AGC won't lift noise above this |
| `ns_level` | moderate | **off** | downlink NS off — far end already codec-NS/DTX'd; a 2nd pass = "underwater" |
| `high_pass` | on | on | DC/rumble |
| `echo_cancel` | **on (AECM)** | n/a | uplink AECM (mobile mode); N/A downlink (no reverse stream) |
| soft-limiter `downlink_gain` | — | 8 (bridge default) | only used when `downlink_use_apm` is off |

**Echo cancel uses AECM (`mobile_mode=true`), not AEC3** — AEC3 does not cancel at 16 kHz (only detects; `aec_webrtc.cpp:82-90`). The uplink feeds the just-played downlink as a per-block reverse reference so AECM has a reference; sourced from the *emitted* (post-downlink-APM) samples so the delay alignment holds.

---

## 4. Bridge integration (real-time lockstep loop)

The SIM7600 USB PCM is **synchronous full-duplex** — read and write must stay balanced per cycle or the modem stops draining uplink, hence the single lockstep thread paced by the ~20 ms blocking mic capture. APM adds:

- **320→2×160 staging** (both directions): WebRTC APM processes fixed 10 ms / 160-sample frames; the 20 ms / 320-sample capture stages into two blocks (`up_stage[]` / `dn_stage[]`, each sized `PHONE_APM_FRAME + PHONE_PCM_FRAME_SAMPLES` = 480; bound: 159 carry + 320 ≤ 480). Steady state is a pass-through; a transient short read carries a <160 remainder.
- **Balance preserved**: the downlink read stays keyed to the samples the uplink *emitted* this cycle, so staging only rechunks already-read bytes — the read/write ratio is byte-identical to the pre-APM loop.
- **arch-L1 uplink byte-carry** (`up_pending[]`): a partial `write()` never leaves the modem mid-sample and desyncs 16-bit framing (pending-then-new; bounded by one frame).
- **Two carries compose**: byte-level `dn_carry` (odd read byte) resolves first → whole samples → sample-level `dn_stage`.
- CPU: ~6 `ProcessStream` calls / 20 ms cycle at 16 kHz mono on Jetson Orin — sub-ms, well within budget; allocation-free steady state.

---

## 5. Live mid-call reconfigure (no restart) — and the v1.3 AGC2 gotcha

`phone_apm_reconfigure()` + `phone_bridge_reconfigure()` apply settings **to a call in progress**: the WebUI thread stashes the new config under a mutex + sets an atomic `s_cfg_dirty`; the bridge thread picks it up **at the top of the loop, before the blocking capture read** — so a submodule-toggle allocation spike is absorbed by the ~80 ms capture ring, never mid-lockstep.

**Vendored v1.3 quirk (verified in `audio_processing_impl.cc:574`):** `ApplyConfig` reinits AGC2 **only when its `enabled` flag flips**. So a mid-call fixed-gain/ramp/noise change with AGC staying on is a *silent no-op*. Handling:
- **fixed gain** → `SetRuntimeSetting(CreateCaptureFixedPostGain)` (the designed runtime path; applied on the next `ProcessStream`).
- **ramp / noise floor** (AGC staying on) → forced `enabled=false`→`true` reinit (one-cycle dip, rare).
- **NS level / HPF / echo / AGC enable-toggle** → plain `ApplyConfig` reinits correctly.

**Clamp is load-bearing and shared:** a `fixed_gain_db ≥ 50` fails v1.3 `GainController2::Validate` and *silently reverts the entire AGC2 block to disabled*. `phone_apm_clamp_config()` (fixed [0,49], ramp [0.1,100], noise [-90,0]) is applied both at create/reconfigure (via one shared `build_apm_config`) and before storing user config, so the persisted value matches what's applied.

---

## 6. Config: the save/load pipeline (and the clobber fix)

The audio config is a **tool-owned** struct (`phone_audio_config_t`), not part of `dawn_config_t`, so it does not round-trip through `SETTINGS_SCHEMA` — it follows the Home Assistant tool pattern.

**The bug that started the settings work:** `config_write_toml()` truncates + rewrites `dawn.toml`; the phone tool had a config *parser* but no *writer*, so **any WebUI settings save silently deleted the entire `[phone]` block** (all call-audio tuning reverted to defaults on reboot — same class as the messaging Phase-6.5 secrets clobber). Fixed by:
- Extracting parse+write into one dependency-light module (`phone_audio_config.{c,h}`) and registering a `.config_writer`.
- A Unity **write↔parse symmetry test** (`test_phone_audio_config`, CI) — a parser key without a matching writer key now fails the round-trip mechanically. This is the durable gate for the "saves but doesn't load" class.

Round-trip, end to end: parse (boot) → GET (`get_phone_audio_config`) → SET (`set_config` `payload.phone`) → `phone_tool_update_config` (clamp + store, mutex-guarded) → **second** `config_write_toml` (HA two-write, since the first ran before the tool struct changed) → clamped echo-back. Thread-safe: `s_config_rwlock → s_registry_mutex → s_phone_tool_mutex` (writer), `s_phone_tool_mutex`/`s_state_mutex`/`s_pending_lock` acquired strictly sequentially (apply).

---

## 7. WebUI

- `src/webui/webui_phone_config.c` — GET handler (current config + `webrtc_available` + `bridge_active`) and the set_config apply body; admin-gated; JSON built via `createElement`/`textContent` (no XSS).
- `www/js/admin/phone-audio.js` + `phone-audio.css` — schema-driven admin panel (Settings → Phone Call Audio): uplink and downlink groups using the shared `.dawn-toggle` primitive, debounced live-save with an "Applied" confirmation, clamped echo-back, disabled-when-unused controls, WebRTC-absent handling. Changes apply to a live call immediately.

---

## 8. Build configuration

AEC/WebRTC is opt-in (`ENABLE_AEC`), and both the wake-word AEC **and** the phone APM need it. During this work it was found `build-debug` had a stale cached `-DENABLE_AEC=OFF` (a worktree/coding-assistant convenience for non-voice builds — the `webrtc-audio-processing` submodule is a separate meson build not auto-populated in worktrees), which silently disabled the APM. Two changes:

- **`default`/`full`/`debug` presets now default `ENABLE_AEC=ON`** (matches the documented CMake option default; a preset cache-var overrides a stale cached OFF). `server`/`server-debug`/`ci` stay off (no local mic / no voice).
- **Graceful degradation**: if `webrtc-audio-processing` isn't built, the CMake AEC block now emits a **warning and disables AEC** instead of a fatal error — so worktree/CI/fresh-clone builds still succeed (AEC off) while AEC defaults on wherever the lib exists.

---

## 9. Latency hardening + deferred

**eff-M1 downlink backlog drain (shipped as a follow-up).** The balanced read
consumes exactly one cycle of downlink, so a scheduling stall that backs up the
modem's tty buffer would become permanent added mouth-to-ear latency.
`downlink_drain_backlog()` (phone_audio_bridge.c) checks `FIONREAD` after each
cycle's play and, when the backlog exceeds ~60 ms, discards the oldest bytes down
to ~20 ms (hysteresis), bounded to ~160 ms/cycle and terminating at `EAGAIN`.  It
reuses the play path's exact `dn_carry` byte accounting, so 16-bit framing stays
aligned regardless of drop size or odd `read()` returns; it drains downlink only,
so the uplink balance is untouched.  Landed separately from the staging change so
a balance regression would stay bisectable (it never fired live before).

**Still deferred:** the reconfigure observable-effect test — asserting a level
(dBFS) delta after a fixed-gain change needs an AEC-on integration harness;
live-verifiable meanwhile via the bridge's ~1 s level trace (uplink/downlink RMS
in dBFS, computed directly from the emitted/played PCM — 0 = full scale).

---

## 10. Files

Sources under `src/`, headers mirror under `include/`.

| Area | Files |
|---|---|
| APM | `src/audio/phone_apm.cpp` + `include/audio/phone_apm.h` |
| Config | `src/tools/phone_audio_config.c` + `include/tools/phone_audio_config.h`, `tests/test_phone_audio_config.c` |
| Bridge | `src/tools/phone_audio_bridge.c` + `include/tools/phone_audio_bridge.h` |
| Service/tool wiring | `src/tools/phone_service.c`, `src/tools/phone_tool.c` (+ their `include/tools/*.h`) |
| WebUI | `src/webui/webui_phone_config.c` + `include/webui/webui_phone_config.h`, `webui_config.c`, `webui_server.c`, `www/js/admin/phone-audio.js`, `www/css/components/phone-audio.css` |
| Config surface | `dawn.toml.example` (`[phone]`) |
| Build | `CMakeLists.txt`, `CMakePresets.json` |

**Review:** 10 agents across plan (4), RT-loop impl (2), WebUI impl (2), UI (1), reuse (1). Live-verified on real 2-party calls.
