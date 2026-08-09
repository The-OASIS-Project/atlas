# Music Streaming — Dedicated Flow-Controlled Transport Design

**Status**: SHIPPED — 2026-08-08, PR #26 (`fix_audio_playback_jittering`, rebase-merged to `main`). Archived design record.
**Last updated**: 2026-08-08.
**Related**: DAP2_DESIGN, WEBUI_DESIGN, OTA_DESIGN, PLEX_INTEGRATION_DESIGN.

This document records how DAWN streams music audio to its clients (browser WebUI and
Tier-1 Raspberry Pi satellites) and how a live "music stutters while TTS is talking"
report was chased through two independent layers of contention to a single, unified,
flow-controlled transport. It captures the end state and the reasoning, including a
use-after-free found and fixed along the way and an OTA TLS bug surfaced during
deployment.

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Architecture overview](#2-architecture-overview)
3. [The dedicated music transport](#3-the-dedicated-music-transport)
4. [Server-side closed-loop flow control](#4-server-side-closed-loop-flow-control)
5. [Browser: decode off the main thread](#5-browser-decode-off-the-main-thread)
6. [Satellite: flow-control adoption](#6-satellite-flow-control-adoption)
7. [The `music_state` teardown use-after-free](#7-the-music_state-teardown-use-after-free)
8. [TTS resampler quality](#8-tts-resampler-quality)
9. [OTA `ssl_verify` (deploy-time fix)](#9-ota-ssl_verify-deploy-time-fix)
10. [Files](#10-files)
11. [Deferred follow-ups](#11-deferred-follow-ups)

---

## 1. Motivation

Music playing through the WebUI would stutter and progressively degrade whenever TTS
spoke at the same time. Diagnosis found **two independent layers of contention**, not
one:

- **Server-side.** The per-session music streaming thread had no feedback on how much
  audio the client had buffered; it also competed with the TTS path, whose
  `SINC_BEST` resampler produced a large synchronous CPU burst per sentence that
  starved the music thread.
- **Client-side (browser).** Music Opus decode ran via WebCodecs on the browser's
  **main thread**, competing with TTS scheduling, the FFT visualizer, per-message JSON
  handling, and (in this deployment) a `ha_state_changed` firehose. Under load the
  decode fell behind, the audio-worklet ring starved, and playback jittered.

The fix addresses both layers and, in doing so, retires a legacy second transport so
that every client streams music the same robust way.

## 2. Architecture overview

Music audio is delivered over a **dedicated WebSocket** (`dawn-music` subprotocol,
port 3001 = main + 1), distinct from the main DAP2 control/voice socket on port 3000.
This isolates high-bandwidth audio from control messages. It is the **sole** music
transport for every client — there is no music-over-the-main-WS fallback.

```
 ┌── browser ──────────────────────────────┐        ┌── daemon ─────────────────────┐
 │  main thread: control WS (3000)          │        │  main WS server (3000):       │
 │      │ music_subscribe/control           │◀──────▶│    control, state, position   │
 │      ▼                                    │        │                               │
 │  Web Worker: music WS (3001) + WebCodecs │◀══════▶│  music WS server (3001):      │
 │      │ decoded PCM (MessagePort)          │  Opus  │    per-session stream thread  │
 │      ▼                                    │ frames │    + closed-loop pacer        │
 │  AudioWorklet ring → speakers            │        │    + write ring               │
 └──────────────────────────────────────────┘        └───────────────────────────────┘

 ┌── Tier-1 satellite (RPi) ────────────────┐
 │  music_stream.c: music WS (3001) +       │◀══════▶ (same music WS server)
 │  Opus decode → ALSA; music_buffer reports│
 └──────────────────────────────────────────┘
```

Both clients participate in the same **closed-loop flow control**: the client reports
how much audio it has buffered, and the server paces production to hold a ~2 s lead.

## 3. The dedicated music transport

- **Server** (`webui_music_server.c`): a separate `libwebsockets` context + thread on
  port 3001, sharing the main server's TLS cert. Each connection authenticates with the
  session token (looked up against the main session), then registers its `wsi` with the
  session's music state (`webui_music_set_stream_wsi`).
- **Per-session send ring** (`webui_music.c`, `session_music_state_t.write_ring`): the
  stream thread produces Opus frames into a small pre-allocated ring
  (`WEBUI_MUSIC_WRITE_RING = 8` slots ≈ 160 ms); the lws service thread drains one slot
  per `WRITEABLE` callback (`webui_music_write_pending`). A short/failed `lws_write`
  discards the ring and closes rather than putting a truncated (framing-desyncing)
  frame on the wire.
- **Sole transport.** When no dedicated socket is registered (`music_wsi == NULL`),
  `queue_music_direct` **drops and counts** the frame rather than rerouting to the main
  WS. The legacy `queue_music_data` path and the `WS_RESP_MUSIC_DATA` response type were
  removed. A client whose dedicated socket is briefly down (startup race, reconnect)
  gets a short gap and resyncs when it registers. Returning `WEBUI_MUSIC_QUEUE_DROPPED`
  also keeps the pacer from crediting a frame the client never received.

## 4. Server-side closed-loop flow control

The client periodically sends `{"type":"music_buffer","buffered_ms":N}` up the music
socket. The server:

- **Clamps** `N` to `[0, WEBUI_MUSIC_CLIENT_BUFFER_MAX_MS]` (10 s = the client ring
  capacity) at ingest, and defensively again in `webui_music_report_buffer`, so a
  garbage/hostile report can never drive the pacer into a multi-second sleep.
- **Publishes** the value then the receipt timestamp as two seq-cst atomics
  (`client_buffered_ms`, `last_buffer_report_ms`), so a fresh timestamp implies an
  at-least-as-fresh value; the pacer reads them lock-free.
- **Paces** the per-session stream thread to hold a target lead
  (`WEBUI_MUSIC_TARGET_BUFFER_MS` ≈ 2000 ms): when the client is below target and the
  send ring is not backing up, it refills at up to ~4× real time; at target it paces at
  real time.
- **Falls back** to real-time pacing with a re-baseline when reports go stale
  (older than `WEBUI_MUSIC_REPORT_STALE_MS`), so a silent client can't wedge production.
- **Gapless auto-advance.** A natural track transition is tagged `advance:"auto"` in the
  state update; the client **suppresses** its buffer flush on that tag (a user seek/skip
  is untagged and still flushes), so the ~2 s already-buffered lead — which includes the
  start of the next track — is not truncated.

## 5. Browser: decode off the main thread

The browser moves the entire music **data path** off the main thread:

- **`music-decode-worker.js`** (a Web Worker) owns the music WebSocket (3001) and the
  WebCodecs stereo Opus `AudioDecoder`. It posts decoded PCM to the AudioWorklet over a
  **transferred `MessagePort`** — delivery lands on the audio render thread, never the
  main thread. Chosen over SharedArrayBuffer specifically to avoid COOP/COEP headers,
  which would sever `window.opener` and break the OAuth popup used for Google
  email/calendar connect. Copy-based transfer is immaterial at 48 kHz stereo.
- **`music-worklet-processor.js`** is dual-mode: it takes audio/clear on the worker port
  (primary) or the node port (the fallback path), and reports its ring depth back on the
  active channel. `MessagePort` delivery is ordered, so a `clear` reliably precedes later
  audio — no ack handshake is needed for the flush protocol.
- **`music-opus-decode.js`** is a shared helper (`forEachOpusFrame`,
  `audioDataToStereo`) used by **both** the worker and the retained main-thread decode
  path, so the length-prefixed frame parser and the WebCodecs format-conversion matrix
  (`f32`/`f32-planar`/`s16`/`s16-planar`, mono upmix) live in one place.
- **`music-playback.js`** orchestrates: it owns the AudioContext graph, keeps control on
  the main WS, decides the path once behind a capability gate, and relays token / clear /
  server-config to the worker. A no-Worker browser falls back to **main-thread decode
  over the same dedicated socket** (a decode-location fallback, not a transport
  fallback).
- **Flow-control reporting.** The worklet reports ring depth every ~32 ms; the worker
  adds the `AudioDecoder.decodeQueueSize` backlog and sends the combined depth up the WS
  at that cadence (what the pacer needs), while throttling the status post to the *main*
  thread to ~100 ms (the UI buffer bar's needs) so the worker isn't re-importing wakeups
  onto the thread this change exists to keep quiet.

## 6. Satellite: flow-control adoption

The Tier-1 satellite already had a dedicated music socket (`music_stream.c`, Opus →
ALSA), but it ran open-loop — it never told the server how much it had buffered, so it
was served by the server's real-time fallback rather than the closed loop. It now sends
periodic `music_buffer` reports (~200 ms cadence, well under the staleness threshold)
built from the existing `music_playback_get_buffered_ms()`. With that, the satellite is
paced by the same closed loop as the browser.

The satellite's own dead main-WS `0x20` music fallback handler and its
`dedicated_producer` gate (which existed only to arbitrate between the two transports)
were removed now that the dedicated socket is the sole path. Firmware bumped to **2.3.0**.

## 7. The `music_state` teardown use-after-free

A full-branch review surfaced a latent UAF made hot by the new report path. The
music-server lws thread reaches `conn->music_state` in three accessors
(`webui_music_report_buffer`, `webui_music_write_pending`, `webui_music_set_stream_wsi`)
while the **main** WS thread frees that state in `webui_music_session_cleanup` — a window
between `free()` and the `conn->music_state = NULL` store. On an ordinary tab close the
music thread could dereference freed memory; the ~32 ms buffer-report path widened the
exposure substantially.

Fix: a module mutex `s_music_teardown_mutex` serializes the three accessors against the
free. `cleanup` stops the producer thread first (it holds no teardown lock; joined
before the free), then takes the teardown mutex around the destroy + `free` +
NULL-store; each accessor takes it around its `conn->music_state` read + use. Lock order
is **teardown → write_mutex**, never the reverse; the producer takes only `write_mutex`
and is gone before the free, so it can't race. The pattern was checked under
ThreadSanitizer (a standalone harness modelling the accessors-vs-free race flags it
without the mutex and is clean with it). This was chosen over a refcount refactor for
being obviously correct with negligible cost at the handful-of-clients scale.

## 8. TTS resampler quality

The TTS output resampler was dropped from `SINC_BEST` to `SINC_MEDIUM`
(`resampler_create_ex()` with a selectable quality tier; ASR input and music keep
`BEST`). `SINC_BEST` is ~4–5× the cost of `SINC_MEDIUM`; the synchronous per-sentence
burst shrinks from roughly 680 ms to 140 ms, which is inaudible for 22.05 k→48 k speech
but stops the TTS thread from starving the real-time music stream thread. The write-ring
depth (`WEBUI_MUSIC_WRITE_RING = 8` ≈ 160 ms) is sized to absorb that ~140 ms burst —
a coupling documented in the header (if the resampler ever reverts to `BEST`, the ring
would under-cover and drop; watch `write_drop_count`).

## 9. OTA `ssl_verify` (deploy-time fix)

Deploying the 2.3.0 satellite over OTA surfaced an unrelated, pre-existing bug: the OTA
image download hardcoded full TLS verification (`CURLOPT_SSL_VERIFYPEER/HOST`) and never
consulted the satellite's `ssl_verify` config flag — which the WebSocket connection
*does* honor. On a self-signed / `ssl_verify=false` deployment the WS connected but the
OTA download failed at certificate verification, reported only as a generic "download
failed." `ssl_verify` is now threaded through the offer → download
(`ota_apply_handle_offer` → `ota_work_t` → `download_to_fd`) so it matches the WS; OTA
images are independently libsodium-signature-verified, so TLS verification here is
defense-in-depth, not the integrity guarantee. The real `CURLcode` is now logged on
failure (token elided).

## 10. Files

**Daemon (C).** `src/webui/webui_music.c` (pacer, write ring, drop accounting, teardown
mutex, transport removal), `src/webui/webui_music_server.c` (music-server thread,
`music_buffer` ingest), `src/webui/webui_send.c` + `include/webui/webui_server.h`
(`WS_RESP_MUSIC_DATA` removal), `include/webui/webui_music.h` +
`include/webui/webui_music_internal.h` (buffer-report API, send-ring + pacer fields),
`src/webui/webui_audio.c` + `src/audio/resampler.c` + `include/audio/resampler.h`
(`SINC_MEDIUM` tier).

**Browser (JS).** `www/js/audio/music-decode-worker.js` (new), `music-opus-decode.js`
(new, shared helper), `music-playback.js` (orchestration + fallback), `music-worklet-
processor.js` (dual-mode), `www/js/ui/music.js` (optimistic seek/track-change + slider
ARIA/keys), `www/js/dawn.js` (main-WS music routing removal), `www/index.html`,
`www/css/components/music.css`.

**Satellite (C).** `dawn_satellite/src/music_stream.c` (`music_buffer` reporting),
`ws_client.c` + `music_playback.c` + headers (0x20 fallback + `dedicated_producer`
removal), `ota_apply.c` + `ota_apply.h` (`ssl_verify` threading + CURLcode log),
`satellite_version.h` (2.3.0).

## 11. Deferred follow-ups

Tracked in DAWN `docs/TODO.md` under "Music transport / flow-control — follow-ups":

- Extract the ~67-line pacer out of `music_stream_thread` (and shrink `webui_music.c`
  back under the size limit) — deliberately kept off the functional-fix branch so the
  timing-sensitive refactor stays independently bisectable.
- Co-locate the ring-depth ↔ TTS-resampler-burst constant so the coupling isn't
  comment-only.
- Two narrow browser worker edge cases (post-clear frame during decoder re-init; a
  redundant double 0-report on flush).
- Remove now-dead symbols (`webui_get_queue_fill_pct`; client-unused
  `WS_BIN_MUSIC_DATA` constants).
- A narrow, pre-existing main-thread-handler vs music-server race on `conn->music_state`
  *creation* (distinct from the freed-state UAF fixed here; pointer-atomic on aarch64).
