# WebUI Always-On Voice Mode Plan (Revised v3)

**Updated:** March 2026

## Summary

Implement an "always-on voice mode" for the WebUI similar to the local session's conversation mode. This enables hands-free operation where the browser continuously listens for a wake word, then automatically records and processes commands.

## Background

**Local Session Conversation Mode:**
The local session uses a state machine (in `dawn.c`):
- `SILENCE` → VAD detects speech → `WAKEWORD_LISTEN`
- `WAKEWORD_LISTEN` → ASR transcribes, checks for wake word → `COMMAND_RECORDING` or `PROCESS_COMMAND`
- `COMMAND_RECORDING` → Records until pause/timeout → `PROCESS_COMMAND`
- `PROCESS_COMMAND` → Direct match or LLM → back to `SILENCE`

**WebUI Current State:**
- Push-to-talk only (hold mic button)
- No continuous listening
- No wake word detection

**Goal:**
- Split button UI with dropdown for mode selection (Hold to Talk / Continuous Listening)
- Server-side wake word detection using Whisper (consistent with local session)
- Use same wake word as local client (AI_NAME from config)
- Use same VAD/ASR sensitivities as local session (no duplicate settings)
- Share VAD context with local session
- Automatic recording after wake word
- VAD-based pause detection for auto-stop
- Return to listening after response completes
- No barge-in support (future enhancement if needed)

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Wake word detection | Server-side (Whisper) | Consistent with local session, any word can be wake word |
| Wake word source | AI_NAME from config | Same as local client, unified experience |
| Auto-stop | VAD pause detection | Natural speech boundaries, same as local session |
| Sensitivity settings | Reuse local session values | No duplicate config, unified behavior |
| UI controls | Split button with dropdown (mirror `.attach-btn-wrapper` pattern) | Clean UI, single control point for audio modes |
| VAD context | **Per-connection** (shared ONNX env only) | LSTM state is mutable; sharing corrupts both streams |
| ASR worker allocation | Async dispatch to worker pool | VAD runs inline (<1ms); ASR dispatched to worker thread to avoid blocking LWS |
| Opus/resampler | Per-connection decoder + resampler | Shared `s_decode_mutex` bottlenecks continuous multi-client streams |
| Authentication | `conn_require_auth()` on enable | Prevent unauthenticated connections from activating server-side audio processing |
| Rate limiting | Per-connection byte-rate cap (~35 KB/s) | Prevent accelerated/crafted audio from exhausting server resources |
| Per-user limit | Max 1 always-on session per user | Multiple tabs CAN hold mic simultaneously; enforce server-side |
| Control messages | JSON text only | Consistency with existing protocol pattern |
| Barge-in | Not implementing | Adds complexity, can add later if needed |
| HTTPS | Required for always-on | Browser requires it for mic access over network anyway |
| Audio streaming | Continuous PCM/Opus to server | Server-side VAD filters; Opus preferred (~3 KB/s vs 32 KB/s raw) |
| Client-side VAD | Not in Phase 1 (future optimization) | Reduces bandwidth/CPU but adds ONNX Runtime Web dependency |
| Tab backgrounding | Supported (AudioWorklet continues) | Audio thread not subject to JS timer throttling |
| TTS echo prevention | Server-side discard (primary) + client mute (secondary) | Server discards frames immediately on PROCESSING + 300ms cooldown on return to LISTENING |
| `always_on_ctx_t` ownership | Pointer in `ws_connection_t`, allocate on enable | Same pattern as `music_state`; avoids 960KB waste per non-always-on connection |
| `sample_rate` validation | Whitelist: 8000, 16000, 22050, 44100, 48000 | Reject unexpected values to prevent integer issues in resampling |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WebUI Always-On Voice Mode Flow                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────┐   VAD speech    ┌──────────────┐   wake word   ┌───────────┐  │
│  │  IDLE   │ ───────────────→│ WAKE_LISTEN  │ ─────────────→│ RECORDING │  │
│  │(listening)│               │ (ASR active) │               │ (capture) │  │
│  └─────────┘                 └──────────────┘               └───────────┘  │
│       ▲                            │                              │         │
│       │                            │ timeout/                     │ pause/  │
│       │                            │ no wake word                 │ timeout │
│       │                            ▼                              ▼         │
│       │                      ┌──────────┐                  ┌───────────┐   │
│       │                      │  IDLE    │                  │ PROCESSING│   │
│       │                      └──────────┘                  └───────────┘   │
│       │                                                          │         │
│       │                    audio complete                        │         │
│       └──────────────────────────────────────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Audio Flow (Server-Side Processing)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Audio Processing Flow                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Browser (Always-On Mode)              Server (webui_always_on.c)            │
│  ─────────────────────────             ──────────────────────────            │
│                                                                              │
│  ┌─────────────────────┐               ┌─────────────────────────┐          │
│  │ Continuous audio    │  Opus/PCM     │ Receive audio stream    │          │
│  │ capture + streaming │ ────────────→ │ via WebSocket           │          │
│  └─────────────────────┘               └───────────┬─────────────┘          │
│                                                    │                         │
│                                                    ▼                         │
│                                        ┌─────────────────────────┐          │
│                                        │ VAD: Speech detected?   │          │
│                                        │ (share local VAD ctx)   │          │
│                                        └───────────┬─────────────┘          │
│                                                    │ yes                     │
│                                                    ▼                         │
│                                        ┌─────────────────────────┐          │
│                                        │ ASR: Transcribe chunk   │          │
│                                        │ (borrow from pool)      │          │
│                                        └───────────┬─────────────┘          │
│                                                    │                         │
│                                                    ▼                         │
│                                        ┌─────────────────────────┐          │
│                                        │ Contains AI_NAME?       │          │
│                                        │ (same check as dawn.c)  │          │
│                                        └───────────┬─────────────┘          │
│                                                    │ yes                     │
│                                                    ▼                         │
│  ┌─────────────────────┐               ┌─────────────────────────┐          │
│  │ State: RECORDING    │ ←───────────  │ Send WAKE_DETECTED msg  │          │
│  │ Visual feedback     │               │ Continue buffering      │          │
│  └─────────────────────┘               └───────────┬─────────────┘          │
│                                                    │                         │
│                                                    ▼                         │
│                                        ┌─────────────────────────┐          │
│                                        │ VAD: Pause detected?    │          │
│                                        │ (reuse local pause_ms)  │          │
│                                        └───────────┬─────────────┘          │
│                                                    │ yes                     │
│                                                    ▼                         │
│                                        ┌─────────────────────────┐          │
│                                        │ Full ASR → LLM → TTS    │          │
│                                        │ (existing pipeline)     │          │
│                                        └───────────┬─────────────┘          │
│                                                    │                         │
│                                                    ▼                         │
│  ┌─────────────────────┐               ┌─────────────────────────┐          │
│  │ Play response audio │ ←───────────  │ Send response audio     │          │
│  │ Return to LISTENING │               │ Send LISTENING state    │          │
│  └─────────────────────┘               └─────────────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Bandwidth and Scalability

### Per-Client Bandwidth (Always-On Streaming)

| Encoding | Bandwidth | Per Minute | Notes |
|----------|-----------|------------|-------|
| Raw PCM (16kHz/16-bit/mono) | 256 kbps (32 KB/s) | ~1.9 MB | Trivial to implement |
| Opus (24 kbps) | 24 kbps (~3 KB/s) | ~180 KB | DAWN already supports via `opus-worker.js` |

On a local network, raw PCM is fine for a handful of clients. **Opus should be preferred** when available — it's already implemented and reduces bandwidth by ~90%.

### Multiple Simultaneous Clients

| Clients | Raw PCM Total | Opus Total | ASR Impact |
|---------|---------------|------------|------------|
| 1 | 32 KB/s | 3 KB/s | Negligible |
| 3 | 96 KB/s | 9 KB/s | VAD filters most; ASR bursts only on speech |
| 5+ | 160 KB/s | 15 KB/s | Consider client-side VAD (Phase 2) |

Server-side VAD filters ~90% of incoming audio as silence before it reaches ASR, so CPU impact scales with actual speech, not connected clients. If 5+ simultaneous always-on clients become a reality, client-side VAD (see Future Enhancements) eliminates idle streaming entirely.

---

## Tab Backgrounding Behavior

### How Browsers Handle Backgrounded Tabs

When a user switches away from the DAWN tab:

| Browser Behavior | Impact on Always-On | Mitigation |
|------------------|---------------------|------------|
| `setTimeout`/`setInterval` throttled to 1s | No impact — audio uses AudioWorklet | None needed |
| `requestAnimationFrame` stops | Visualizer pauses (cosmetic only) | None needed |
| **AudioWorklet continues** | Audio capture and streaming continue | This is the key property |
| Tab freeze (Chrome, after extended background + memory pressure) | All processing stops | Heartbeat detection (see below) |

**AudioWorklet runs on the audio rendering thread**, which is driven by the hardware clock, not JavaScript timers. As long as the AudioContext is in `running` state with an active MediaStream source, the worklet's `process()` callback fires at the hardware rate regardless of tab visibility.

### Tab Freeze Detection

Chrome may freeze a tab after extended backgrounding under memory pressure. Mitigations:

1. **Server-side heartbeat**: If no audio frames arrive for >5 seconds during always-on mode, server sends a WebSocket ping. If no pong, mark client as stale.
2. **Client-side `visibilitychange` listener**: Log when tab is backgrounded; warn user if always-on may be interrupted.
3. **Audio keepalive**: The active MediaStream + AudioContext connection signals to the browser that the page is using audio resources, which protects against aggressive tab discarding.

```javascript
// In always-on.js — detect tab visibility changes
document.addEventListener('visibilitychange', () => {
   if (document.hidden && AlwaysOnMode.enabled) {
      console.log('Tab backgrounded — always-on audio continues via AudioWorklet');
   }
});
```

---

## Implementation Plan

### Phase 1: Server-Side Always-On State Machine

**New file: `src/webui/webui_always_on.c` / `include/webui/webui_always_on.h`**

```c
// Always-on state machine states (mirrors dawn.c local session)
typedef enum {
   ALWAYS_ON_DISABLED,        // Feature off, normal push-to-talk
   ALWAYS_ON_LISTENING,       // Streaming, waiting for VAD trigger
   ALWAYS_ON_WAKE_CHECK,      // VAD triggered, accumulating audio for ASR
   ALWAYS_ON_WAKE_PENDING,    // ASR dispatched to worker thread, awaiting result
   ALWAYS_ON_RECORDING,       // Wake word confirmed, recording command
   ALWAYS_ON_PROCESSING       // ASR → LLM → TTS (server discards audio, client mutes)
} always_on_state_t;

// Per-connection always-on context (allocated on enable, freed on disable/disconnect)
// Stored as pointer in ws_connection_t: always_on_ctx_t *always_on; (same as music_state)
typedef struct {
   _Atomic always_on_state_t state;  // Atomic for lock-free reads from worker threads
   pthread_mutex_t mutex;            // Protects state transitions and buffer access
   uint8_t *audio_buffer;            // Circular buffer for streaming audio (heap-allocated)
   size_t audio_write_pos;           // Write position in circular buffer
   size_t audio_read_pos;            // Read position for VAD/ASR
   size_t audio_valid_len;           // Valid unprocessed data length
   int64_t last_speech_time_ms;      // For pause detection
   int64_t last_audio_received_ms;   // For heartbeat / auto-disable (10s no-audio)
   int64_t wake_word_time_ms;        // When wake word was detected
   int64_t state_entry_time_ms;      // For timeout handling
   silero_vad_context_t *vad_ctx;    // Per-connection VAD (own LSTM state, shared ONNX env)
   OpusDecoder *opus_decoder;        // Per-connection Opus decoder (stateful for PLC)
   resampler_t *resampler;           // Per-connection 48kHz→16kHz resampler
   uint32_t client_sample_rate;      // From always_on_enable message (validated)
   int64_t cooldown_until_ms;        // Post-TTS cooldown: discard audio until this time
} always_on_ctx_t;

// Buffer size: 30 seconds @ 16kHz mono 16-bit = 960,000 bytes
// Allocated lazily on always_on_enable(), freed on always_on_disable()
#define ALWAYS_ON_BUFFER_SIZE (16000 * 2 * 30)

// Timeout values (reuse from local session where applicable)
#define ALWAYS_ON_WAKE_CHECK_TIMEOUT_MS  5000   // Max time in WAKE_CHECK before returning to LISTENING
#define ALWAYS_ON_RECORDING_TIMEOUT_MS   15000  // Max recording duration (same as local session)

// Public API
int always_on_init(always_on_ctx_t *ctx);
void always_on_cleanup(always_on_ctx_t *ctx);
int always_on_process_audio(always_on_ctx_t *ctx, const uint8_t *audio, size_t len,
                            session_t *session, struct lws *wsi);
void always_on_enable(always_on_ctx_t *ctx);
void always_on_disable(always_on_ctx_t *ctx);
always_on_state_t always_on_get_state(always_on_ctx_t *ctx);
```

**Key implementation details:**
- Atomic state field for lock-free reads; mutex for state transitions and buffer writes
- **Per-connection VAD context** — each always-on client gets its own `silero_vad_context_t` with independent LSTM state. The ONNX Runtime environment is shared (pass `shared_env` from the daemon's existing VAD init). Cost: ~1.5 KB per connection.
- **Per-connection Opus decoder** (~17 KB) and **resampler** (~4 KB) — eliminates `s_decode_mutex` contention for continuous multi-client streams.
- **Async ASR dispatch** — VAD runs inline in the LWS callback (<1ms). When VAD detects speech end, audio is handed to a worker thread for Whisper transcription. State transitions to `WAKE_PENDING` while ASR is in flight; new audio continues to buffer but no duplicate ASR is dispatched. Worker callback transitions state based on result.
- Wake word string matching extracted into a shared utility (currently inline in `dawn.c` using file-scope statics — needs refactoring into a standalone function)
- Get AI_NAME from config (same as local session)
- Get pause duration from config (`vad.pause_ms` or default 1500ms)
- **Audio routing**: When `always_on != NULL && state != DISABLED`, all `WS_BIN_AUDIO_IN` frames route to `always_on_process_audio()` and `WS_BIN_AUDIO_IN_END` is ignored. Mode switches only allowed when both paths are idle.
- **Rate limiting**: Track bytes received per second per connection. Drop frames exceeding ~35 KB/s (raw PCM rate + margin). Log warning.
- **Overflow detection**: When `audio_valid_len` would exceed `ALWAYS_ON_BUFFER_SIZE`, advance `audio_read_pos` to discard oldest data with a log warning. Never silently overwrite.
- Proper circular buffer with read/write positions

**TTS Echo Prevention (Defense in Depth):**

1. **Server-side discard (primary):** When transitioning to PROCESSING, immediately set `cooldown_until_ms = 0` (meaning: discard all audio). `always_on_process_audio()` drops all frames while in PROCESSING state. When transitioning back to LISTENING, set `cooldown_until_ms = now + 300ms` — discard audio for 300ms after TTS ends to drain in-flight echo frames before resuming VAD.

2. **Client-side mute (secondary):** On receiving `always_on_state: "processing"`, the client pauses the worklet via `muteContinuous()`. On `always_on_state: "listening"`, the client resumes via `unmuteContinuous()`. This reduces wasted bandwidth during TTS but is not relied upon for correctness.

The server-side mechanism is authoritative — it handles the race window between state change and client mute arrival.

**Timeout and Error Recovery:**

| State | Timeout | Recovery Action |
|-------|---------|-----------------|
| WAKE_CHECK | 5 seconds | Return to LISTENING, log timeout |
| WAKE_PENDING | 10 seconds | ASR worker timed out; return to LISTENING, log error |
| RECORDING | 15 seconds | Process what we have, then return to LISTENING |
| PROCESSING | 30 seconds (LLM timeout) | Send error to client, return to LISTENING (client unmutes) |
| LISTENING (no audio) | 10 seconds | Auto-disable always-on, reclaim resources, notify client |
| ASR unavailable | N/A | Log warning, stay in current state, retry on next audio |

### Phase 2: WebSocket Message Types (JSON Text Only)

**JSON text messages for control (consistency with existing protocol):**

```javascript
// Client → Server
{ "type": "always_on_enable" }
{ "type": "always_on_disable" }

// Server → Client
{ "type": "always_on_state", "payload": { "state": "listening" } }
{ "type": "always_on_state", "payload": { "state": "wake_check" } }
{ "type": "always_on_state", "payload": { "state": "recording" } }
{ "type": "always_on_state", "payload": { "state": "processing" } }
{ "type": "always_on_state", "payload": { "state": "disabled" } }
{ "type": "wake_detected", "payload": { "transcript": "hey friday" } }
{ "type": "recording_end" }
{ "type": "always_on_error", "payload": { "code": "TIMEOUT", "message": "..." } }
```

**Note:** Binary WebSocket frames remain audio-only. All always-on control uses JSON text messages for consistency with existing protocol pattern.

### Phase 3: Server Integration

**Modify `include/webui/webui_internal.h`:**

Add pointer to `ws_connection_t` (same pattern as `music_state`):
```c
always_on_ctx_t *always_on; /* NULL if not enabled */
```

**Modify `src/webui/webui_server.c`:**

1. Handle `always_on_enable` / `always_on_disable` JSON messages:
   - **Gate with `conn_require_auth()`** — unauthenticated connections must not activate always-on
   - Validate `sample_rate` from payload against whitelist (8000, 16000, 22050, 44100, 48000)
   - **Enforce per-user limit**: check if another connection with same `auth_user_id` already has `always_on != NULL`. If so, reject with `ALREADY_ACTIVE` error.
   - On enable: allocate `always_on_ctx_t`, init VAD context (shared ONNX env), create Opus decoder + resampler, allocate circular buffer
   - On disable: free all per-connection resources, set pointer to NULL
2. **Audio routing in `handle_binary_message()`**:
   - If `conn->always_on != NULL && state != DISABLED`: route `WS_BIN_AUDIO_IN` to `always_on_process_audio()`, ignore `WS_BIN_AUDIO_IN_END`
   - Otherwise: existing push-to-talk accumulate-and-process path
   - Reject `always_on_enable` if push-to-talk audio is in progress (`conn->audio_buffer` non-empty)
   - Reject push-to-talk `start` if always-on is active
3. **Per-connection rate limiting**: Track bytes received per second. Drop frames exceeding ~35 KB/s.
4. When processing completes:
   - Send response audio
   - Set `cooldown_until_ms = now + 300ms` (post-TTS echo drain)
   - Automatically return to `ALWAYS_ON_LISTENING` state
5. On WebSocket disconnect:
   - Free `always_on_ctx_t` and all resources
   - Set pointer to NULL
6. **Auto-disable**: If no audio received for 10 seconds during always-on, transition to DISABLED, free resources, notify client. This prevents connection exhaustion from idle clients.

**Sample rate handling:**

Browsers may deliver audio at 48kHz despite requesting 16kHz (the constraint is a hint, not guaranteed). The server already handles resampling for DAP2 satellites — reuse the same path for always-on audio. Include `audioContext.sampleRate` in the `always_on_enable` message so the server knows the incoming rate.

```javascript
// In always_on_enable message
{ "type": "always_on_enable", "payload": { "sample_rate": 48000 } }
```

### Phase 4: Browser Client Updates

**New file: `www/js/audio/always-on.js`**

```javascript
// Browser-side always-on coordinator
const AlwaysOnMode = {
   enabled: false,
   state: 'disabled', // disabled, listening, wake_check, recording, processing

   enable() {
      if (!window.isSecureContext) {
         console.error('Always-on requires HTTPS');
         return false;
      }
      // Request mic permission if not already granted
      // Start continuous audio capture
      // Send always_on_enable message to server
      // Update UI state
      this.enabled = true;
      this.updateUI();
   },

   disable() {
      // Stop continuous audio capture (but keep mic available for push-to-talk)
      // Send always_on_disable message to server
      // Reset UI state
      this.enabled = false;
      this.state = 'disabled';
      this.updateUI();
   },

   onStateChange(newState) {
      this.state = newState;

      // TTS echo prevention: mute mic streaming during processing/playback
      if (newState === 'processing') {
         DawnAudioCapture.muteContinuous();   // Pause worklet streaming (mic stays acquired)
      } else if (newState === 'listening') {
         DawnAudioCapture.unmuteContinuous(); // Resume worklet streaming
      }

      this.updateUI();
      this.announceStateChange(newState); // Accessibility
   },

   updateUI() {
      // Update split button appearance
      // Update visualizer state colors
      // Update status text
   },

   announceStateChange(state) {
      // ARIA live region announcement
      const messages = {
         'listening': 'Listening for wake word',
         'wake_check': 'Checking for wake word',
         'recording': 'Recording your command',
         'processing': 'Processing your request',
         'disabled': 'Continuous listening disabled'
      };
      // Announce to screen readers
   }
};
```

**Modify `www/js/audio/capture.js`:**

The current `capture.js` acquires a new MediaStream on every `start()` and releases all tracks + suspends AudioContext on `stop()`. Always-on mode requires a persistent audio pipeline:

- Add `startContinuous()` method:
  - Acquires MediaStream **once** and keeps it alive
  - Keeps AudioContext in `running` state (no suspend)
  - Sets worklet `isRecording = true` indefinitely
  - Audio streams to server continuously without button interaction
- Add `stopContinuous()` method:
  - Sends worklet `stop` message
  - Releases MediaStream tracks
  - Suspends AudioContext
- Push-to-talk `start()`/`stop()` remain unchanged for non-always-on mode
- **Key change**: The worklet already returns `true` from `process()` to stay alive — no worklet changes needed, only the main-thread lifecycle in `capture.js`

```javascript
// New API surface for always-on mode
DawnAudioCapture.startContinuous()  // Acquire mic, start streaming, keep alive
DawnAudioCapture.stopContinuous()   // Release mic, stop streaming
DawnAudioCapture.isContinuous()     // Check if in continuous mode
DawnAudioCapture.muteContinuous()   // Pause worklet streaming (mic stays acquired)
DawnAudioCapture.unmuteContinuous() // Resume worklet streaming
```

**TTS echo prevention (`muteContinuous` / `unmuteContinuous`):**

When the server enters PROCESSING state (ASR → LLM → TTS), the client mutes the audio stream to prevent the microphone from picking up TTS playback and feeding it back to the server as a false wake word trigger. The mic and AudioContext stay alive — only the worklet's `isRecording` flag is toggled:

```javascript
function muteContinuous() {
   if (!continuousMode || !audioProcessor) return;
   audioProcessor.port.postMessage({ type: 'stop' });  // Pause streaming
   console.log('Always-on: muted during TTS playback');
}

function unmuteContinuous() {
   if (!continuousMode || !audioProcessor) return;
   audioProcessor.port.postMessage({ type: 'start' }); // Resume streaming
   console.log('Always-on: unmuted, resuming listening');
}
```

The mute/unmute is driven by `always_on_state` messages from the server: mute on `processing`, unmute on `listening`. This is simpler and more reliable than trying to detect TTS playback end on the client side — the server knows exactly when the response is complete.

**Modify `www/js/core/websocket.js`:**

- Handle `always_on_state`, `wake_detected`, `recording_end`, `always_on_error` messages
- Route to `AlwaysOnMode.onStateChange()`
- Handle `ALREADY_ACTIVE` error (show user message: "Always-on is active in another tab")

**Modify `www/js/dawn.js`:**

- Initialize always-on module
- Connect split button dropdown to mode selection
- Guard mode switching: prevent `always_on_enable` while push-to-talk is in progress (`DawnState.getIsRecording()`)

**Browser-side error recovery:**

- **Mic permission revoked**: `mediaStream.getTracks()[0].onended` fires → auto-disable always-on, show user message
- **WebSocket disconnect**: `stopContinuous()` on disconnect. On reconnect, if localStorage preference is `continuous`, auto-send `always_on_enable` to resume
- **`sendAudioChunk` gate**: Check `DawnAudioCapture.isContinuous()` as alternative to `DawnState.getIsRecording()` for continuous mode
- **Tab return re-sync**: On `visibilitychange` returning to visible, request current state from server to update UI if state changed while backgrounded

### Phase 5: UI Updates - Split Button with Dropdown

**Pattern:** Mirror the existing `.attach-btn-wrapper` / `.attach-dropdown` / `.dropdown-item` pattern exactly. Do not create new class names for the same concept.

**Modify `www/index.html`:**

Replace the mic button with a split button wrapper:

```html
<!-- Split button for audio modes (mirrors .attach-btn-wrapper pattern) -->
<div class="mic-btn-wrapper">
   <button id="mic-btn" title="Hold to talk" aria-label="Hold to talk">
      <svg><!-- microphone icon --></svg>
   </button>
   <button id="mic-dropdown-btn" title="Audio mode" aria-label="Select audio mode"
           aria-haspopup="true" aria-expanded="false" aria-controls="mic-dropdown">
      <svg><!-- chevron down icon --></svg>
   </button>
   <div id="mic-dropdown" class="mic-dropdown hidden" role="menu">
      <button role="menuitem" class="dropdown-item" data-mode="push-to-talk">
         <svg><!-- mic icon --></svg>
         Hold to Talk
      </button>
      <button role="menuitem" class="dropdown-item" data-mode="continuous">
         <svg><!-- broadcast/wave icon --></svg>
         Continuous Listening
      </button>
   </div>
</div>
```

**Modify `www/css/components/input.css`:**

Follow the `.attach-btn-wrapper` CSS pattern (border joining, hover highlighting, `aria-expanded` styling, chevron rotation). Reuse `.dropdown-item` for menu entries. Use correct CSS variable names.

```css
/* Mic split button — mirrors .attach-btn-wrapper pattern */
.mic-btn-wrapper {
   display: flex;
   position: relative;
}

.mic-btn-wrapper #mic-btn {
   border-top-right-radius: 0;
   border-bottom-right-radius: 0;
   border-right: none;
}

#mic-dropdown-btn {
   border-top-left-radius: 0;
   border-bottom-left-radius: 0;
   padding: 0 0.5rem;
   min-width: 32px;  /* Touch-friendly (WCAG 2.5.5) */
}

/* Reuse existing .dropdown-item class for menu entries */
.mic-dropdown {
   position: absolute;
   bottom: 100%;
   right: 0;  /* Right-aligned since mic is rightmost control */
   margin-bottom: 4px;
   background: var(--bg-secondary);
   border: 1px solid var(--border-color);
   border-radius: var(--border-radius);
   box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
   min-width: min(180px, calc(100vw - 1rem));
   z-index: 100;
   opacity: 1;
   transform: translateY(0);
   transition: opacity 0.15s, transform 0.15s;
}

.mic-dropdown.hidden {
   opacity: 0;
   transform: translateY(4px);
   visibility: hidden;
   pointer-events: none;
}

/* Active mode indicator — use dawn-status-dot or SVG, not Unicode */
.mic-dropdown .dropdown-item.active {
   color: var(--success);
}

/* State-specific styling when continuous listening is active */
.mic-btn-wrapper.always-on-listening #mic-btn {
   animation: pulse-listening 3s ease-in-out infinite;
   background: rgba(34, 197, 94, 0.2);
   border-color: var(--success);
}

.mic-btn-wrapper.always-on-wake-check #mic-btn {
   background: rgba(34, 197, 94, 0.4);
   border-color: var(--success);
   animation: pulse-listening 1.5s ease-in-out infinite;
}

.mic-btn-wrapper.always-on-recording #mic-btn {
   background: var(--success);
   color: white;
   animation: pulse-recording 1s infinite;
}

.mic-btn-wrapper.always-on-processing #mic-btn {
   background: var(--warning);
   color: white;
}

@keyframes pulse-listening {
   0%, 100% {
      box-shadow: 0 0 0 0 rgba(34, 197, 94, 0.4);
      opacity: 0.7;
   }
   50% {
      box-shadow: 0 0 0 8px rgba(34, 197, 94, 0);
      opacity: 1;
   }
}

@media (prefers-reduced-motion: reduce) {
   .mic-btn-wrapper.always-on-listening #mic-btn,
   .mic-btn-wrapper.always-on-wake-check #mic-btn,
   .mic-btn-wrapper.always-on-recording #mic-btn {
      animation: none;
   }
}
```

**Modify `www/css/layout/responsive.css`:**

Update the 600px breakpoint to target the wrapper:

```css
.mic-btn-wrapper {
   flex: 1;
}

.mic-btn-wrapper #mic-btn {
   flex: 1;
}
```

**Modify `www/css/components/visualizer.css`:**

Add always-on state colors (aligned with existing design system):

```css
/* Always-on listening - dimmed green, slow pulse */
#ring-container.always-on-listening #ring-svg path,
#ring-container.always-on-listening .ring-base {
   stroke: var(--success);
   opacity: 0.5;
}

#ring-container.always-on-listening #ring-inner {
   animation: core-idle-pulse 4s ease-in-out infinite;
}

/* Always-on wake check - brighter green, faster pulse */
#ring-container.always-on-wake-check #ring-svg path {
   stroke: var(--success);
   opacity: 0.8;
}

/* Always-on recording - full green, same as manual recording */
#ring-container.always-on-recording #ring-svg path {
   stroke: var(--success);
   opacity: 1;
}

#ring-container.always-on-recording #ring-inner {
   animation: core-recording 0.5s ease-in-out infinite;
}

/* Always-on processing - amber, same as thinking */
#ring-container.always-on-processing #ring-svg path {
   stroke: var(--warning);
   opacity: 1;
}
```

**SVG Icons:**

Microphone icon (existing):
```html
<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
   <path d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"/>
   <path d="M19 10v2a7 7 0 0 1-14 0v-2"/>
   <line x1="12" y1="19" x2="12" y2="23"/>
   <line x1="8" y1="23" x2="16" y2="23"/>
</svg>
```

Broadcast/continuous listening icon:
```html
<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
   <path d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"/>
   <path d="M19 10v2a7 7 0 0 1-14 0v-2"/>
   <!-- Radio waves -->
   <path d="M5 3L3 5" opacity="0.6"/>
   <path d="M19 3l2 2" opacity="0.6"/>
   <path d="M5 21L3 19" opacity="0.6"/>
   <path d="M19 21l2-2" opacity="0.6"/>
</svg>
```

Chevron down icon:
```html
<svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
   <polyline points="6 9 12 15 18 9"/>
</svg>
```

### Phase 6: Accessibility

**ARIA Announcements:**

Add a live region for state announcements:

```html
<div id="always-on-announcer" aria-live="polite" aria-atomic="true" class="sr-only"></div>
```

```javascript
// In always-on.js
announceStateChange(state) {
   const announcer = document.getElementById('always-on-announcer');
   const messages = {
      'listening': 'Continuous listening active. Waiting for wake word.',
      'wake_check': 'Speech detected. Checking for wake word.',
      'recording': 'Wake word detected. Recording your command.',
      'processing': 'Processing your request.',
      'disabled': 'Continuous listening disabled. Hold microphone button to talk.'
   };
   announcer.textContent = messages[state] || '';
}
```

**Keyboard Navigation:**

- Dropdown opens with Enter/Space on chevron button
- Arrow keys navigate dropdown options
- Escape closes dropdown
- Tab moves focus appropriately

---

## Files to Modify/Create

### Server-side (C)

| File | Action | Changes |
|------|--------|---------|
| `include/webui/webui_always_on.h` | **Create** | Always-on state machine API, `always_on_ctx_t` struct |
| `src/webui/webui_always_on.c` | **Create** | State machine, per-connection VAD, async ASR dispatch, rate limiting |
| `include/webui/webui_internal.h` | Modify | Add `always_on_ctx_t *always_on` pointer to `ws_connection_t` |
| `src/webui/webui_server.c` | Modify | Auth-gated enable/disable handlers, audio routing, per-user limit |
| `src/webui/webui_audio.c` | Modify | Audio routing switch between push-to-talk and always-on |
| `src/core/wake_word.c` (or similar) | **Create** | Extract wake word string matching from `dawn.c` statics into shared utility |
| `CMakeLists.txt` | Modify | Add new source files |

### Browser-side (JavaScript/CSS)

| File | Action | Changes |
|------|--------|---------|
| `www/js/audio/always-on.js` | **Create** | Always-on mode coordinator |
| `www/js/audio/capture.js` | Modify | Add continuous streaming mode, mute/unmute, error recovery |
| `www/js/core/websocket.js` | Modify | Handle always-on message types, ALREADY_ACTIVE error |
| `www/js/dawn.js` | Modify | Initialize always-on, wire up split button, mode switch guards |
| `www/index.html` | Modify | Replace mic button with `.mic-btn-wrapper` split button |
| `www/css/components/input.css` | Modify | `.mic-btn-wrapper` styles (mirror `.attach-btn-wrapper` pattern) |
| `www/css/components/visualizer.css` | Modify | Always-on state animations |
| `www/css/layout/responsive.css` | Modify | Update 600px breakpoint for `.mic-btn-wrapper` flex |

---

## Configuration

**No new server configuration needed.** The always-on mode reuses existing settings:

| Setting | Source | Used For |
|---------|--------|----------|
| Wake word | `general.ai_name` | Wake word detection |
| VAD sensitivity | `vad.*` settings | Speech detection |
| Pause duration | `vad.pause_ms` | End-of-utterance detection |
| ASR model | `asr.model` | Transcription |

**Requirement:** HTTPS must be enabled (`webui.https = true`) for always-on mode. This is a browser requirement for microphone access over network.

**Mode preference persistence:** Stored in `localStorage` (browser-local, not synced to server). The mode selection is per-browser, not per-user — a user may want always-on on their desktop but push-to-talk on their phone.

**Audio encoding:** Uses Opus when available (via existing `opus-worker.js`), falls back to raw PCM. Opus is expected to be available in all modern browsers.

---

## Verification Checklist

### Functional
1. [ ] Split button displays correctly with dropdown (mirrors attach-button pattern)
2. [ ] Dropdown shows "Hold to Talk" and "Continuous Listening" options
3. [ ] Selecting "Continuous Listening" enables always-on mode
4. [ ] Visual feedback shows current state (listening/wake-check/recording/processing)
5. [ ] Say wake word → verify state changes to recording
6. [ ] Speak command → verify auto-stop on pause
7. [ ] Verify response plays, then returns to listening state
8. [ ] Test timeout when no wake word detected (returns to listening)
9. [ ] Test max duration timeout during recording (15 seconds)
10. [ ] Verify push-to-talk still works when "Hold to Talk" is selected
11. [ ] Test with local session simultaneously (no VAD interference — separate contexts)
12. [ ] Test reconnection: always-on state resets to disabled, resumes from localStorage on reconnect

### Security
13. [ ] Unauthenticated connection cannot send `always_on_enable`
14. [ ] Second tab for same user gets `ALREADY_ACTIVE` error
15. [ ] Verify HTTPS enforcement (always-on rejected over HTTP)
16. [ ] Audio rate exceeding 35 KB/s is dropped with warning
17. [ ] Invalid `sample_rate` in enable message is rejected
18. [ ] Auto-disable after 10s no audio (idle connection reclaimed)

### Accessibility & UI
19. [ ] ARIA announcements work for screen readers
20. [ ] Keyboard navigation works for dropdown (Enter/Space/Escape/arrows)
21. [ ] `prefers-reduced-motion` disables pulse animations
22. [ ] Responsive layout works at 600px breakpoint
23. [ ] Mic permission revocation auto-disables always-on with user message

### Echo Prevention
24. [ ] TTS playback does not trigger false wake word (server discards during PROCESSING)
25. [ ] 300ms cooldown after TTS prevents echo frames on LISTENING resume
26. [ ] Client mute/unmute toggles correctly on state transitions

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| False wake word triggers | Uses same detection as local session (proven) |
| Server CPU load | VAD filters before ASR, same as local session |
| Privacy concerns | Clear visual indicator when listening, easy toggle; browser always shows mic indicator |
| Browser mic permission | Handle permission denial gracefully, show helpful message |
| Race condition with push-to-talk | Explicit audio routing: always-on active → reject PTT; PTT active → reject enable |
| Thread safety | All state transitions protected by mutex |
| Timeout handling | Explicit timeouts for each state with recovery actions |
| Tab freeze (Chrome) | AudioWorklet survives backgrounding; heartbeat detects freeze; auto-recover on focus |
| Sample rate mismatch | Include `audioContext.sampleRate` in enable message; server resamples (reuses DAP2 path) |
| Multiple always-on clients | Per-user limit (max 1); per-connection rate cap (35 KB/s); client-side VAD future option |
| Echo from TTS playback | Server-side discard (primary) + 300ms cooldown + client mute (secondary) |
| Multiple browser tabs | Per-user always-on limit enforced server-side; second tab gets `ALREADY_ACTIVE` error |
| Unauthenticated access | `conn_require_auth()` gate on `always_on_enable` handler |
| Accelerated/crafted audio | Per-connection byte-rate cap; ASR trigger debounce |
| Malformed sample_rate | Whitelist validation (8000/16000/22050/44100/48000); reject others |
| Idle connection exhaustion | Auto-disable after 10s no audio; reclaim all resources |

---

## Not Implementing (Future Enhancements)

- **Barge-in**: Interrupting TTS playback when user speaks. Would require keeping VAD active during playback and handling echo cancellation edge cases. See `FUTURE_WORK.md` §1 Phase 4.
- **Browser-side wake word**: No practical library exists for arbitrary wake words in the browser. Porcupine (WASM) requires pre-trained wake words. openWakeWord has no browser runtime. Server-side Whisper remains the correct approach for DAWN's configurable AI_NAME.
- **Dedicated VAD/ASR resources**: Uses shared resources like DAP clients.
- **Pre-roll buffer**: 500ms circular buffer in browser to capture the start of the wake word. Low priority — Whisper handles partial wake words well.
- **Cancel word support**: "stop", "cancel", "nevermind" during TTS to cancel without new command. Depends on barge-in.

### Client-Side VAD (Phase 2 Optimization)

When multiple always-on clients become a concern, add browser-side Silero VAD as a pre-filter:

- **Library**: `@ricky0123/vad-web` (v0.0.30+) — wraps Silero VAD v6 via ONNX Runtime Web (~2MB model)
- **How it works**: VAD runs in browser, only streams audio when speech is detected. Eliminates ~90% of idle streaming.
- **Impact**: Reduces per-client bandwidth from 32 KB/s (PCM) or 3 KB/s (Opus) to near-zero during silence. Reduces server VAD CPU to zero per idle client.
- **Trade-off**: Adds ~2MB ONNX model download to WebUI; adds ONNX Runtime Web as a browser dependency
- **Integration point**: `capture.js` `startContinuous()` would feed audio to local VAD first; only when `onSpeechStart` fires does it begin streaming to server. Server still does wake word detection via Whisper.

```
Phase 1 (this plan):  Browser ──[continuous audio]──▶ Server VAD ──▶ ASR
Phase 2 (future):     Browser VAD ──[speech only]──▶ Server ASR (no server VAD needed)
```

### Landscape Notes (March 2026)

No production voice assistant (Alexa, Google, Home Assistant) runs wake word detection in the browser. All use dedicated hardware or server-side detection. DAWN's server-side Whisper approach is consistent with industry practice. The browser's role is limited to audio capture and streaming — the AudioWorklet API handles this reliably, including in backgrounded tabs.

---

## Implementation Order

1. **Server: Extract wake word matching** - Move string matching from `dawn.c` statics into shared utility
2. **Server: Create `webui_always_on.c/h`** - State machine with per-connection VAD, async ASR dispatch, rate limiting, overflow detection, cooldown timer
3. **Server: Integrate into `webui_server.c` and `webui_audio.c`** - Auth-gated enable/disable, audio routing, per-user limit, sample_rate validation, auto-disable on idle
4. **Browser: Create `always-on.js`** - Client coordinator, ARIA announcements, error recovery, tab return re-sync
5. **Browser: Modify `capture.js`** - Add `startContinuous`/`stopContinuous`/`muteContinuous`/`unmuteContinuous`, mic permission revocation handler
6. **Browser: Add WebSocket handlers** - Process state change messages, ALREADY_ACTIVE error
7. **Browser: Implement split button UI** - `.mic-btn-wrapper` HTML/CSS (mirror attach-button), responsive fix, `prefers-reduced-motion`
8. **Browser: Add visualizer states** - CSS for always-on state colors
9. **Testing** - End-to-end verification per checklist (functional, security, accessibility, echo)
