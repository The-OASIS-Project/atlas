# Remote & Local ASR Latency — end-of-speech, resample, Whisper tuning, audio_ctx

**Status:** Shipped, live-verified (WebUI + Aurora + local mic). Author: Friday. 2026-09-01.
**Reviewed:** E1 (`audio_ctx`) + the always-on refactor passed a 4-lens review
(correctness / architecture / embedded-efficiency / security) — 0 Critical/High/Medium; A1
equivalence and auto-scale-never-under-provisions both proven; config wiring called
"model-citizen." The earlier fixes (DTX end-of-speech, resample, greeting/dwell) were each
reviewed inline as they shipped.

**Motivating report:** Kris: "reduce the latency on processing remote (WebUI, Aurora) ASR —
when I'm done speaking it takes a while for what I said to appear in chat." What started as a
latency-tuning pass surfaced a real end-of-speech **bug** and became a full sweep of the
speak-to-transcript path on every client.

This documents the whole arc: the pipeline, the DTX end-of-speech bug (the marquee find), each
shipped fix, the E1 `audio_ctx` mechanism + benchmark, and the one lever deliberately left on the
table (the dwell). It exists mainly so the next person doesn't re-derive the DTX trap the hard way.

---

## 1. The pipeline (stop-speaking → transcript)

Perceived latency for "my words appear in chat" is **only** end-of-speech detection + Whisper —
the transcript is sent immediately after ASR, *before* the LLM/TTS (`webui_audio.c`
`webui_send_transcript_ex`). Stages:

```
speech ends → end-of-speech detection (dwell) → [extract buffer] → resample 48k→16k → Whisper batch → send transcript
```

There are **two** remote end-of-speech mechanisms and one local:
- **Push-to-talk** (WebUI mic button, Aurora/ESP32): client-driven end marker; no server dwell.
- **Always-on** (WebUI "always listening"): server-side Silero VAD in `webui_always_on.c`.
- **Local mic** (`dawn.c` state machine): server-side VAD + `chunking_manager` (chunks ≤10s).

Measured post-fix on Jetson (small.en, GPU): end-of-speech fires ~1.0–1.5s after you stop;
resample ~15ms/sec of audio (MEDIUM); Whisper RTF ~0.17–0.21. The **dwell dominates** for short
commands (see §5).

---

## 2. The marquee bug — DTX starves a frame-gated end-of-speech check

**Symptom:** always-on utterances ended erratically — 1.2s on WebUI, but **3–20s** on Aurora, and
sometimes hung until the 30s backstop. Same code, same machine, same mic.

**Root cause:** end-of-speech was evaluated **only inside the per-frame VAD loop** in
`always_on_process_audio` — i.e. it could only ask "has it been silent for the dwell?" *when an
audio frame arrived to run the check*. But both clients enable **Opus DTX** (discontinuous
transmission), which **stops sending frames during silence** — exactly the silence we're trying to
detect. No frame → no check → the utterance couldn't end until the next stray frame trickled in,
and that arrival time is erratic (down to whenever the client's DTX comfort-noise cadence or the
user's next sound produced a frame). It was never Aurora's audio — the arriving silence frames
scored correctly (~0.02); the detector was simply *starved*.

**Fix:** make end-of-speech **timer-driven, not frame-gated.** `always_on_check_timeouts()` (the
periodic tick, `webui_always_on.c`) now evaluates the wall-clock dwell for WAKE_CHECK and RECORDING
independent of frame arrival, firing the same `dispatch_wake_check`/`dispatch_cmd_transcribe` the
per-frame path uses. The tick was sped 1Hz → **4Hz** (`webui_server.c`) so the fire lands within
~250ms of the dwell. The per-frame check stays (fires instantly when frames flow); the timer is the
backstop that closes the DTX gap. Both callers run on the single lws thread, so the shared
`dispatch_*` (which transitions state) is not double-fired.

**Lesson for future clients:** any server-side endpoint that consumes a DTX/VAD stream must not
depend on frame arrival to detect *absence* of frames. This is latent for any new streaming client.

---

## 3. Config-driven VAD dwell (and the `[vad]` coherence fix)

Remote always-on had hardcoded `#define`s (`0.5` speech gate, `1500ms` dwell) that silently
diverged from the `[vad]` config the **local** mic honors. Wired remote always-on to
`g_config.vad.speech_threshold` + `.end_of_speech_duration`, and dropped the shared default 1.2s →
**1.0s**. Now one knob tunes both paths; tunable from `dawn.toml`/WebUI without a rebuild.

**Known asymmetry (accepted):** remote always-on does *not* honor `speech_threshold_tts` (the
local path's TTS-echo-resistance gate) or `silence_threshold` hysteresis. Mitigated by client-side
mic muting during PROCESSING; the residual case (a client with no mute + no AEC) is a client
problem. Noted, not built.

---

## 4. The bare-wake-word flow — greeting echo, dropped command, ding

The bare wake word ("Friday" alone → greet → record command) uses the RECORDING state, which the
DTX fix made *reachable* for the first time and thereby exposed several latent bugs:

- **Dropped command:** RECORDING was entered with a **stale** `last_speech_ms` (from wake
  detection), so the new timer fired end-of-speech on the next tick and transcribed an empty
  buffer. Fix: reset `last_speech_ms = 0` on RECORDING entry — the `>0` guard then defers the dwell
  until the first *command* word (also closes the "slow to start speaking" case).
- **Wake word in the transcript:** `extract_buffered_audio` reads from `wake_start_pos`, never
  moved on RECORDING entry, so the command re-included the "Okay Friday" audio. Fix: reset the ring
  (`wake_start_pos`/`read_pos`/`valid_len`) on RECORDING entry.
- **Greeting echo hang:** the server-side spoken "Hello." greeting played through the client
  speaker *while the recording mic was live*; on a client whose TTS output isn't AEC-referenceable
  (Aurora played TTS through a bare Web Audio graph the browser echo-canceller doesn't see) it
  echoed straight back in, the VAD scored it as speech, and end-of-speech hung. **Fix: drop the
  server greeting entirely.** The `recording` state frame is the ready cue; replaced by a
  **client-side ding** — a short non-speech Web Audio tone on the `always_on_state:recording` frame,
  which is echo-safe because Silero won't score a pure tone as speech (WebUI: `www/js/audio/
  always-on.js`; Aurora landed the equivalent in its own repo). The ding is deliberately **not**
  TTS-gated (it's a UI cue, not Friday's voice).
- **No-command timeout:** bare wake word + silence used to hang to the 30s RECORDING backstop. Added
  a 6s `ALWAYS_ON_NO_COMMAND_TIMEOUT_MS` → return to LISTENING.

**Protocol precedence trap (documented in WEBSOCKET_PROTOCOL.md):** during the greeting DAWN emits
top-level `state:speaking` *concurrent with* `always_on_state:recording`. A client that mutes on
`state:speaking` gags its own command window — `always_on_state:recording` must win. This bit
Aurora; the note exists so it doesn't bite the next client.

---

## 5. The dwell is now the dominant latency — the lever left on the table

After the above, the short-command pipeline is roughly **dwell(1000ms) + resample(~50ms) +
Whisper(~600–900ms)**. The two downstream stages are optimized; the **flat second of dwell after
you stop talking is the floor**, and nothing shipped touches it. The real remaining lever for
*perceived* latency is a **smarter endpoint — an adaptive/shorter dwell that re-opens if you resume
speaking** — but that's a UX design change with premature-cutoff risk, not a tweak. Deliberately
deferred; captured in TODO.md. Flag it before chasing more ms off Whisper: the dwell is the bigger
number.

---

## 6. Resample 48k→16k: BEST → MEDIUM

The always-on/WebUI path resamples the whole utterance 48k→16k **single-threaded after endpoint**,
in series before Whisper (`resample_48k_to_16k`, `webui_audio.c`). It was using
`SRC_SINC_BEST_QUALITY` — ~48ms/sec of pure pre-Whisper latency (250ms @ 5s, 788ms @ 17s). Dropped
to `RESAMPLER_QUALITY_MEDIUM` (SINC_MEDIUM, ~15ms/sec, ~3–4× faster); Whisper is robust to the
artifacts (the TTS path already made this call). **The local capture resampler stays BEST** by
design — it resamples per-frame (streaming, amortized), so there's no synchronous spike to cut.

The remaining structural win here — **resample-on-ingest** (resample each chunk as it arrives into a
16k ring, ~0 endpoint cost + 3× ring memory + removes `s_decode_mutex` contention) — is deferred
(E4). MEDIUM already banked most of the *latency*; the memory/lock wins are the stronger case now.

---

## 7. Whisper decode tuning (shared ASR path)

Model: English deployments should use the `.en` variants (`base.en`/`small.en`) — faster *and* more
accurate than the multilingual base of the same size. Kris runs **small.en** (RTF ~0.17–0.18 on
this Jetson, only ~1.6× base.en despite 3× params — GPU absorbs it; nailed
"supercalifragilisticexpialidocious" where base would mangle it).

Decode params (`asr_whisper.c`, shared by every client):
- `single_segment = true` — command audio is one window (ring caps 30s, local chunks 10s); skip
  multi-segment decoder passes.
- `temperature_inc = 0.0f` — disable the fallback ladder so a blank/noisy decode doesn't re-run
  inference up to ~5×. Bounds worst-case tail latency; no accuracy risk for command-length audio.
- Freed a per-utterance leak of the discarded `asr_process_partial` result in
  `webui_audio_transcribe` (Whisper has no streaming partials; it was the only leaking caller).

---

## 8. E1 — scale `audio_ctx` to utterance length

**Problem:** Whisper's encoder processes a full 30s window (`audio_ctx = 1500`) regardless of clip
length, so a short command pays for 30s of encoder every time.

**Mechanism** (`common/src/asr/asr_whisper.c`, config-agnostic — the satellite links this lib and is
unaffected): per-context `audio_ctx_override` + `asr_whisper_set_audio_ctx()` + a resolver applied
before each `whisper_full`. Encoding: `0` = model default (byte-identical, opt-in); `>0` = fixed
(diagnostic only — a too-small fixed value is dangerous, see below); `<0` = auto-scale, floor
encoded as `-value`. **Auto never sets ctx below the audio's own token need + margin** (50 tok/sec,
+32), which is why it is safe where a fixed value is not. Plumbed via `asr_set_audio_ctx()`
(`asr_interface.c`, no-op for Vosk), wired at both daemon ASR init sites (`dawn.c`,
`worker_pool.c`) as `floor > 0 ? -floor : 0`. Exposed as `[asr] audio_ctx_floor` (default **768**, 0
disables) via the full 9-file config path.

**Why fixed is a trap (benchmark finding):** a fixed `audio_ctx` smaller than a clip needs makes
Whisper hallucinate/loop — WER in the hundreds of % *and* 2–3× slower. Only auto-scale is viable.
The benchmark caught this before it shipped; the naive "fixed 256" the first mini-test made look
great (it only had 5s clips) would have broken every long utterance in production.

**Floor = 768, chosen for cross-model safety:**

| floor | small.en Δlatency / ΔWER | base.en Δlatency / ΔWER |
|---|---|---|
| auto/768 | +20% / +1.0pp | +5% / +0.0pp |
| auto/512 | +23% / +5.0pp | +15% / +0.0pp |
| auto/256 | +25% / +4.0pp | +2% / **+40.8pp** 💥 |
| baseline | — | — |

base.en **craters at 256** (smaller model hallucinates more readily when context-starved); 768 is
≤+1pp WER on both models. The win is real on small.en (Kris's model) and harmless on base.en. Note:
768 exceeds the auto-scaled value for any clip < ~14.7s, so for normal commands it's effectively a
**flat 768** (a half-window) — the auto ramp only engages for long-form audio. Observable in the
log: `asr_whisper_finalize: "…" (Nms, RTF: X, audio_ctx: 768)`.

**Benchmark harness** (reproducible, committed): `tests/asr_benchmark --audio-ctx`, plus
`test_recordings/e1_audio_ctx_sweep.sh` + `e1_wer_analysis.py` (WER vs `recording_guide.txt`
references) + the raw sweep CSVs + `E1_AUDIO_CTX_RESULTS.md`. Repair note: the `asr_benchmark`
target was bit-rotted (stale includes + a missing metrics link); fixed the includes and stubbed
`metrics_record_asr_timing` rather than drag in metrics.c's transitive deps.

---

## 9. Left on the table

- **Adaptive/shorter dwell that re-opens on resumed speech** (§5) — the biggest *perceived*-latency
  lever; UX design change, TODO.md.
- **E4 resample-on-ingest** (§6) — ~0 endpoint resample + 3× ring memory + kills `s_decode_mutex`
  contention; MEDIUM effort, trigger-gated.
- **Per-model floor** — 768 over-provisions the local mic's small (1.5s) chunks; a lower floor would
  help *only* if committed to small.en and local-mic latency becomes a focus (base.en safety caps
  it universally at 768).
- **Trigger-gated polish** (from the E1 review): extract the floor→signed mapping helper (2
  duplicated call sites); split the tri-mode `audio_ctx` API if a 2nd production caller of the fixed
  mode appears; move the `[0,1500]` clamp into a shared `config_clamp_asr` so the env path is
  clamped at the field.

---

## Key files

- `src/webui/webui_always_on.c` — always-on VAD state machine, timer-driven end-of-speech,
  `always_on_eos_reached()` helper, bare-wake-word flow, no-command timeout.
- `src/webui/webui_server.c` — the 4Hz always-on tick.
- `src/webui/webui_audio.c` — resample (MEDIUM), transcribe, transcript send.
- `common/src/asr/asr_whisper.c` — decode params, `audio_ctx` resolver + setter, finalize log.
- `src/asr/asr_interface.c` — `asr_set_audio_ctx` (Vosk-guarded).
- `src/dawn.c`, `src/core/worker_pool.c` — E1 wiring at ASR init.
- `[asr] audio_ctx_floor`, `[vad] end_of_speech_duration` — the tunable knobs.
- `www/js/audio/always-on.js` — the client ding.
- `docs/WEBSOCKET_PROTOCOL.md` — the `state:speaking` vs `always_on_state:recording` precedence note.
- `test_recordings/E1_AUDIO_CTX_RESULTS.md` + `e1_*` — the E1 benchmark + data.
