# WebUI Aesthetic Overhaul - "Stark-Grade" Implementation Plan

## Goal

Convert WebUI from "pretty demo" to "operational instrument":
- Every major visual maps to a real runtime signal
- UI communicates: listening, thinking, streaming stability, memory pressure
- Avoid decorative-only elements until Phase 5+

## Current State (Summary)

- SVG waveform ring with 5 trailing paths (60fps via requestAnimationFrame)
- State colors: Cyan idle, Green listening, Amber thinking, Red error
- Context bar exists (simple progress)
- Basic metrics panel exists

## Key Constraint

OpenAI/Claude APIs do not expose confidence/logits/entropy.
Therefore: "confidence ring" becomes **Hesitation ring** using inter-token timing variance.

## Ring Data Model (Final)

| Ring | Data Source | Update Rate |
|------|-------------|-------------|
| Inner | Audio FFT amplitude (existing) | 60fps |
| Middle | Token throughput (tokens/sec, smoothed) | 10Hz while streaming |
| Outer | Token hesitation (stddev of inter-token intervals) | Per token + decay |

---

## Phase 1: Metrics Pipeline (Server → Client)

**Goal:** Expose minimal real metrics needed for rings + gauge.

### WebSocket Message

```json
{
  "type": "metrics_update",
  "payload": {
    "state": "idle|listening|thinking|speaking|error",
    "ttft_ms": 450,
    "token_rate": 45.2,
    "context_percent": 34
  }
}
```

### Send Rules

- On state change (immediate)
- On token chunk event (update token_rate, dt stats)
- Heartbeat 1Hz (context_percent, state durations if needed)

### Files

| File | Changes |
|------|---------|
| `include/webui/webui_server.h` | New WS message type |
| `src/webui/webui_server.c` | `webui_send_metrics_update()` |
| `www/js/dawn.js` | Handle `metrics_update`, store global metrics state |

---

## Phase 2: Multi-Ring SVG Structure (Core Visual Rework)

**Goal:** Restructure SVG into three ring groups. No polish features yet.

### SVG Groups

```svg
<g id="ring-fft">         <!-- existing waveform, scaled down -->
<g id="ring-throughput">  <!-- segmented arc, 64 segments -->
<g id="ring-hesitation">  <!-- segmented arc, 64 segments -->
```

### Ring Update Rates

- FFT: 60fps (existing)
- Throughput: whenever token_rate updates (or 10Hz while streaming)
- Hesitation: per token event + idle decay

---

## Phase 2A: Hesitation Ring (MVP Algorithm)

**Goal:** Outer ring reacts to token timing irregularity.

### Client State

```javascript
const dtWindow = [];      // size 16
let tPrevMs = 0;
let loadSmooth = 0;       // 0..1
```

### On Token Event

```javascript
dt = now - prev
push dt into dtWindow (cap 16)
std = stddev(dtWindow)
load = clamp((std - 10) / (40 - 10), 0..1)
loadSmooth = loadSmooth + 0.2 * (load - loadSmooth)
```

### Idle Decay

```javascript
// If no token for 300ms:
loadSmooth *= 0.95  // per frame or per 50ms tick
```

### Render

```javascript
jitterPx = lerp(0, 3, loadSmooth)
// Each segment radius:
r = base + noise(segId, time) * jitterPx
// noise must be smooth (no frame-to-frame randomness)
```

---

## Phase 3: Context Pressure Gauge (Replaces Progress Bar)

**Goal:** Context usage becomes a segmented arc/gauge (not another ring).

### Visual Design

- 12-16 segments with visible gaps
- Color policy:
  - <50%: Cyan
  - 50-75%: Cyan → Amber gradient
  - >75%: Amber
  - >90%: Brief red tick flash
- Optional transient labels later (Phase 5)

---

## Phase 4: Layout + Typography (System vs Human)

**Goal:** Separate visual planes.

### Layout

- Offset rings slightly left
- Telemetry panel right, dimmed when idle

### Font Roles

| Role | Font | Usage |
|------|------|-------|
| System | JetBrains Mono | Uppercase, spaced, telemetry |
| Human | Inter | Chat text |

No other typography rules in this phase.

---

## Phase 5: Polish (Optional, Only After Phases 1-4 Work)

Pick 1-2 items max:

- [ ] Imperfections engine (rare glow overshoot OR tiny rotation jitter)
- [ ] Idle breathing / irrational ring rotations
- [ ] Transient labels ("COMPRESSION ACTIVE", "LIMIT APPROACHING")
- [ ] Tick marks / calibration hints

---

## Color Rules

| Color | Hex | Meaning |
|-------|-----|---------|
| Cyan | #22d3ee | Nominal operation |
| Green | #22c55e | Listening (explicit exception) |
| Amber | #f59e0b | Attention/degradation |
| Red | #ef4444 | Critical/error |
| White | #e0e0e0 | Human text |

---

## Implementation Order

1. **Phase 1** - Metrics pipeline ✅ COMPLETE
2. **Phase 2** - SVG multi-ring structure ✅ COMPLETE
3. **Phase 2A** - Hesitation ring (outer ring behavior) ✅ COMPLETE
4. **Phase 3** - Context pressure gauge ✅ COMPLETE
5. **Phase 4** - Layout + typography ✅ COMPLETE
6. **Phase 5** - Polish (idle breathing, irrational rotations) ✅ COMPLETE

---

## Implementation Summary

### Design Corrections Applied

Per authoritative design spec:
- **Rings are FIXED** - No rotation, orbiting, or floating
- **Allowed motion only**: Radial jitter (data-driven), glow changes, subtle idle breathing (±2%)
- **Layout is canonical**: `[RINGS] [TELEMETRY]` side-by-side, 50/50 split
- **Color is functional**: Every color communicates state, never decorative

### What Was Built

**Server-side (`src/webui/webui_server.c`):**
- `webui_send_metrics_update()` - Sends real-time metrics via WebSocket
- `WS_RESP_METRICS_UPDATE` message type with state, ttft_ms, token_rate, context_percent
- Uses `-1` sentinel for context_percent when data unavailable (prevents overwriting valid values)

**Server-side (`src/core/session_manager.c`):**
- Added streaming metrics fields to `session_t`: `stream_start_ms`, `first_token_ms`, `last_token_ms`, `stream_token_count`
- `get_time_ms()` helper for monotonic timing
- `session_text_chunk_callback()` tracks TTFT and token rate during streaming
- `combined_chunk_callback()` (for TTS) also tracks metrics
- Metrics sent every 5 tokens to avoid WebSocket flooding

**Client-side (`www/js/dawn.js`):**
- `metricsState` - Global metrics tracking with `last_ttft_ms` and `last_token_rate` for persistence
- `hesitationState` - Token timing variance tracking with rolling window of 16 samples
- `renderSegmentedArc()` - Generates 64-segment arcs for throughput/hesitation rings
- `initializeRings()`, `initializeContextGauge()` - Ring initialization
- `onTokenEvent()`, `updateHesitationRing()` - Hesitation algorithm with radial jitter
- `smoothNoise()` - Sine-based noise for organic jitter
- `renderContextGauge()` - 16-segment arc with color states
- `updateTelemetryPanel()` - Right-side numeric readouts with value persistence
- `updateContextDisplay()` - Updates both context gauge AND telemetry panel
- `handleMetricsUpdate()` - Handles -1 sentinel to preserve previous context value

**Styling (`www/css/dawn.css`):**
- Fixed 3-ring instrument assembly (no rotation)
- **50/50 layout**: Ring container and telemetry panel each use `flex: 1 1 50%`
- Ring SVG scales to fill container (100% width/height, max 240px)
- Telemetry panel vertically centered, dimmed when idle
- Typography system (system font vs human font)
- Context pressure gauge with 270° segmented arc
- Subtle idle breathing only (±2% scale)
- Responsive: stacks vertically on mobile (<600px)

**HTML (`www/index.html`):**
- `[RINGS + STATUS] [TELEMETRY + CONTEXT GAUGE]` canonical layout
- 3-ring SVG structure (hesitation, throughput, fft) - all sharing fixed center
- Status indicator (IDLE/LISTENING/THINKING/SPEAKING) positioned under the rings
- Telemetry panel with TOKEN RATE, TTFT, LATENCY readouts
- Context gauge with 16 segments at bottom of telemetry panel

---

## Telemetry Panel Metrics

| Metric | Source | Display | Persistence |
|--------|--------|---------|-------------|
| TOKEN RATE | Calculated from streaming chunk timing | `XX.X tok/s` | Last non-zero value preserved |
| TTFT | `first_token_ms - stream_start_ms` | `XXX ms` | Last non-zero value preserved |
| LATENCY | `hesitationState.loadSmooth` mapped to ms | `XX ms var` | Real-time during streaming |
| CONTEXT | Context gauge at panel bottom | `XX%` arc + text | Updated via gauge |

### Latency (ms var) Explained

The LATENCY reading shows **token timing variance** - a proxy for model "hesitation":

1. **Tracking**: Inter-token intervals stored in rolling window (16 samples)
2. **Calculation**: Standard deviation normalized to 0-1 range:
   - 3ms stddev = 0 (steady, confident)
   - 20ms stddev = 1 (irregular, uncertain)
3. **Display**: `loadSmooth` mapped to ~3-40ms range, shown as "XX ms var"
4. **Colors**: <10ms normal, 10-25ms amber (warning), >25ms red (danger)

This same data drives the outer hesitation ring's radial jitter.
