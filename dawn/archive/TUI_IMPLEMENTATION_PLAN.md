# DAWN TUI Implementation Plan

## Overview

Add a console-based Terminal User Interface (TUI) to DAWN that provides real-time monitoring and statistics while maintaining the existing debug logging mode for development.

## Goals

- User-friendly real-time status display
- Performance metrics visualization (latency breakdown, token usage)
- State machine visibility
- Session statistics tracking
- Zero performance impact when disabled
- Maintain existing debug logging for development

## Development Phases

### Phase 1: Metrics Collection Infrastructure ✓ (DO FIRST)

**Objective:** Build the data collection layer before the UI

**Tasks:**
1. Define `dawn_metrics_t` structure in `include/ui/metrics.h`
2. Implement metrics collection in `src/ui/metrics.c`
3. Add instrumentation points:
   - State transitions (src/dawn.c)
   - ASR completion timing (src/asr/asr_whisper.c, src/asr/asr_vosk.c)
   - LLM token tracking (src/llm/llm_openai.c, src/llm/llm_claude.c)
   - LLM timing - TTFT and total (src/llm/llm_streaming.c)
   - TTS timing (src/tts/piper.cpp)
   - VAD probability updates (src/asr/vad_silero.c)
4. Implement activity log circular buffer
5. Test data collection with existing debug logging

**Deliverable:** Metrics collecting in background, verifiable via debug logs

### Phase 2: Basic TUI Framework

**Objective:** Create the visual interface with static/test data

**Tasks:**
1. ncurses initialization with color support
2. Implement three color schemes (see Color Schemes section)
3. Create static layout with placeholder data
4. Keyboard input handling:
   - Q: Quit
   - D: Toggle between TUI and debug log mode
   - R: Reset stats
   - 1/2/3: Switch color schemes
   - ?: Help screen
5. Screen refresh at 10 Hz (100ms interval)

**Deliverable:** Working TUI displaying static data, all color schemes functional

### Phase 3: Integration

**Objective:** Connect metrics to TUI display

**Tasks:**
1. Wire metrics data to TUI display elements
2. Implement real-time updates
3. Activity log rendering from circular buffer
4. State machine visualization
5. Token counter displays
6. Performance graphs/bars

**Deliverable:** Fully functional TUI displaying real-time DAWN metrics

### Phase 4: Polish

**Objective:** Final refinements and optimizations

**Tasks:**
1. Implement help screen (? key)
2. Reset stats command (R key) functionality
3. Stats export to JSON on exit
4. Performance optimization
5. CPU usage validation (ensure 10 Hz refresh is acceptable)
6. Error handling and graceful degradation
7. Documentation updates

**Deliverable:** Production-ready TUI with all features

## Metrics Structure

```c
typedef struct {
   // Session stats
   uint32_t queries_total;
   uint32_t queries_cloud;
   uint32_t queries_local;
   uint32_t errors_count;
   uint32_t fallbacks_count;

   // Token counters (cumulative)
   uint64_t tokens_cloud_input;
   uint64_t tokens_cloud_output;
   uint64_t tokens_local_input;
   uint64_t tokens_local_output;
   uint64_t tokens_cached;

   // Last query timing (milliseconds)
   double last_vad_time_ms;
   double last_asr_time_ms;
   double last_asr_rtf;              // Real-Time Factor
   double last_llm_ttft_ms;          // Time To First Token
   double last_llm_total_ms;
   double last_tts_time_ms;
   double last_total_pipeline_ms;

   // Averages (rolling or session)
   double avg_vad_ms;
   double avg_asr_ms;
   double avg_asr_rtf;
   double avg_llm_ttft_ms;
   double avg_llm_total_ms;
   double avg_tts_ms;
   double avg_total_pipeline_ms;

   // Real-time state
   float current_vad_probability;    // 0.0 - 1.0
   listeningState current_state;
   llm_type_t current_llm_type;
   cloud_provider_t current_cloud_provider;

   // State time tracking (seconds spent in each state)
   time_t state_time[INVALID_STATE];

   // Recent activity log (circular buffer)
   #define MAX_LOG_ENTRIES 100
   #define MAX_LOG_LENGTH 256
   char activity_log[MAX_LOG_ENTRIES][MAX_LOG_LENGTH];
   int log_head;
   int log_count;

   // Session timing
   time_t start_time;
   time_t end_time;

   // Thread safety
   pthread_mutex_t metrics_mutex;
} dawn_metrics_t;
```

## TUI Layout Design

```
┌─ DAWN Status ──────────────────────────────────────────────────────────────┐
│ State: WAKEWORD_LISTEN        Uptime: 02:34:17        Mode: LLM_ONLY       │
│ LLM: Cloud (OpenAI GPT-4o)    ASR: Whisper base       VAD: Silero          │
└────────────────────────────────────────────────────────────────────────────┘

┌─ Session Stats ────────────────────────────────────────────────────────────┐
│ Queries:  12 total  (8 cloud, 4 local)    Errors: 0    Fallbacks: 1       │
│                                                                             │
│ Token Usage (This Session):                                                │
│   Cloud:  2,847 input  +  1,523 output  =  4,370 total                     │
│   Local:    892 input  +    453 output  =  1,345 total                     │
│   Cached: 1,234 tokens (90% cost savings)                                  │
└────────────────────────────────────────────────────────────────────────────┘

┌─ Last Query Performance ───────────────────────────────────────────────────┐
│ Command: "What time is it?"                                                │
│                                                                             │
│ Latency Breakdown:                                                         │
│   VAD Detection:         42 ms  █                                          │
│   ASR (RTF: 0.109):     156 ms  ████                                       │
│   LLM TTFT:             138 ms  ███                                        │
│   LLM Total:            423 ms  ██████████                                 │
│   TTS Generation:        87 ms  ██                                         │
│   ─────────────────────────────                                           │
│   Total Pipeline:       846 ms                                             │
└────────────────────────────────────────────────────────────────────────────┘

┌─ Real-Time Audio ──────────────────────────────────────────────────────────┐
│ VAD Speech Probability: ████░░░░░░ 42%                                     │
│ Audio Buffer: 512 / 2048 samples                                           │
└────────────────────────────────────────────────────────────────────────────┘

┌─ Recent Activity ──────────────────────────────────────────────────────────┐
│ 14:23:42  User: "Turn on armor display"                                   │
│ 14:23:43  FRIDAY: "HUD online, boss."                                     │
│ 14:24:15  User: "What time is it?"                                        │
│ 14:24:16  FRIDAY: "2:24 PM, sir."                                         │
│ 14:25:03  [WARN] Cloud API rate limit, fallback to local                  │
└────────────────────────────────────────────────────────────────────────────┘

 [D]ebug Logs  [R]eset Stats  [1/2/3] Theme  [Q]uit           Press ? for help
```

## Color Schemes

### Scheme 1: Apple ][ Classic Green

**Aesthetic:** Retro computing, high contrast, nostalgic
**Best for:** Low-light environments, retro vibe, demos

```
Background:     Black (#000000)
Primary text:   Bright green (#00FF00)
Borders/boxes:  Green (#00CC00)
Highlights:     Lighter green (#33FF33)
Dim text:       Dark green (#008800)
Bars:           Green gradient (░▒▓█)
```

**ncurses colors:**
- `COLOR_BLACK` for background
- Custom green pairs for text hierarchy
- Bold attribute for highlights

### Scheme 2: Modern Glowing Blue (JARVIS-inspired)

**Aesthetic:** Sci-fi HUD, JARVIS/Iron Man themed
**Best for:** Demos, cool factor, thematic consistency with OASIS

```
Background:     Very dark blue (#0A0E1A)
Primary text:   Cyan-blue (#00D9FF)
Borders/boxes:  Electric blue (#0099FF)
Highlights:     Bright cyan (#00FFFF)
Dim text:       Steel blue (#4B6B8C)
Bars:           Blue gradient with glow effect
Accent:         Teal for important values (#00CCA3)
```

**ncurses colors:**
- `COLOR_BLACK` for background (closest to dark blue)
- `COLOR_CYAN` and `COLOR_BLUE` custom pairs
- Bold + color for glow effect simulation

### Scheme 3: High Contrast B/W

**Aesthetic:** Maximum readability, professional
**Best for:** High ambient light, accessibility, screenshots/documentation

```
Background:     Black (#000000)
Primary text:   White (#FFFFFF)
Borders/boxes:  Bright white (#FFFFFF)
Highlights:     Bold white (with A_REVERSE for emphasis)
Dim text:       Gray (#888888)
Bars:           White blocks (░▒▓█)
Warnings:       Reverse video (black on white)
```

**ncurses colors:**
- `COLOR_BLACK` and `COLOR_WHITE` only
- Heavy use of `A_BOLD` and `A_REVERSE` attributes
- Clean and simple

## Command-Line Interface

### Invocation

```bash
# Traditional debug log mode (default, unchanged behavior)
./dawn

# TUI mode with default scheme (blue/JARVIS)
./dawn --tui

# TUI with specific color scheme
./dawn --tui --theme=green     # Apple ][ Classic
./dawn --tui --theme=blue      # JARVIS (default)
./dawn --tui --theme=bw        # High Contrast B/W

# Environment variable alternative
DAWN_TUI=1 ./dawn
DAWN_TUI_THEME=green ./dawn --tui
```

### Runtime Controls

| Key | Action |
|-----|--------|
| `Q` / `Esc` | Quit DAWN |
| `D` | Toggle between TUI and debug log mode |
| `R` | Reset session statistics |
| `L` | Switch to Local LLM (session-only, not persisted) |
| `C` | Switch to Cloud LLM (session-only, not persisted) |
| `1` | Switch to Green (Apple ][) theme |
| `2` | Switch to Blue (JARVIS) theme |
| `3` | Switch to B/W (High Contrast) theme |
| `?` | Show help screen |

**Note:** LLM selection via `L`/`C` keys is session-only and reverts to the default (or command-line specified) setting on restart.

## Update Frequency

**Screen Refresh:** 10 Hz (every 100ms)
- Balances responsiveness with CPU usage
- Smooth enough for VAD probability bar
- Can be reduced to 5 Hz if CPU impact is too high

**Metrics Collection:** Event-driven
- State changes: Immediate
- ASR completion: On result
- LLM tokens/timing: On response
- VAD probability: Every inference (~100ms naturally)
- TTS timing: On completion

**Activity Log:** Event-driven (user interactions and responses)

## Stats Export on Exit

### File Location
Same directory as conversation history

### Filename Format
`dawn_stats_<timestamp>.json`

Example: `dawn_stats_20251124_143052.json`

### JSON Structure

```json
{
  "session": {
    "start_time": "2025-11-24T14:30:00Z",
    "end_time": "2025-11-24T16:45:23Z",
    "duration_seconds": 8123,
    "exit_reason": "user_quit"
  },
  "queries": {
    "total": 47,
    "cloud": 35,
    "local": 12,
    "errors": 2,
    "fallbacks": 1
  },
  "tokens": {
    "cloud": {
      "input": 12847,
      "output": 8523,
      "total": 21370,
      "cached": 4234,
      "cache_savings_percent": 90.0
    },
    "local": {
      "input": 3892,
      "output": 2453,
      "total": 6345
    }
  },
  "performance": {
    "averages": {
      "vad_ms": 38.5,
      "asr_ms": 142.3,
      "asr_rtf": 0.112,
      "llm_ttft_ms": 135.7,
      "llm_total_ms": 458.2,
      "tts_ms": 92.1,
      "total_pipeline_ms": 866.8
    },
    "last_query": {
      "vad_ms": 42.0,
      "asr_ms": 156.0,
      "asr_rtf": 0.109,
      "llm_ttft_ms": 138.0,
      "llm_total_ms": 423.0,
      "tts_ms": 87.0,
      "total_pipeline_ms": 846.0
    }
  },
  "state_distribution_seconds": {
    "SILENCE": 4523,
    "WAKEWORD_LISTEN": 3200,
    "COMMAND_RECORDING": 245,
    "PROCESS_COMMAND": 155,
    "VISION_AI_READY": 0,
    "NETWORK_PROCESSING": 0
  },
  "llm_configuration": {
    "type": "cloud",
    "provider": "openai",
    "model": "gpt-4o"
  }
}
```

### Save Triggers

- Normal exit (Ctrl+C signal)
- Q key in TUI mode
- SIGTERM signal
- Appended if file exists (for crash recovery analysis)

## Build System Integration

### CMakeLists.txt Changes

```cmake
# Add optional TUI support
option(ENABLE_TUI "Enable Terminal UI" ON)

if(ENABLE_TUI)
  find_package(Curses REQUIRED)
  if(CURSES_FOUND)
    target_link_libraries(dawn ${CURSES_LIBRARIES})
    target_include_directories(dawn PRIVATE ${CURSES_INCLUDE_DIR})
    add_definitions(-DENABLE_TUI)
    message(STATUS "TUI support enabled with ncurses")
  else()
    message(WARNING "ncurses not found, TUI support disabled")
  endif()
endif()
```

### File Structure

```
include/ui/
  ├── metrics.h      # Metrics structure and API
  └── tui.h          # TUI interface

src/ui/
  ├── metrics.c      # Metrics collection implementation
  └── tui.c          # TUI rendering and control
```

## Dependencies

- **ncurses** - Terminal UI library (standard on most Linux distributions)
- **pthread** - For mutex protection (already a dependency)

### Installation

```bash
# Debian/Ubuntu
sudo apt-get install libncurses5-dev libncursesw5-dev

# Fedora/RHEL
sudo dnf install ncurses-devel

# Arch
sudo pacman -S ncurses
```

## Graceful Degradation

### Non-TTY Detection
If stdout is not a TTY (piped, redirected):
- Automatically disable TUI even if `--tui` specified
- Fall back to traditional logging
- Warning message: "TUI mode requires interactive terminal, falling back to log mode"

### Missing ncurses
If ncurses not available at build time:
- Compile without TUI support
- `--tui` flag shows error: "DAWN was built without TUI support"
- Traditional logging works normally

### Terminal Size
Minimum terminal size: 80x24
- If terminal too small: Warning message and TUI disabled
- If terminal resized during runtime: Adapt or disable gracefully

## Thread Safety

### Current Architecture (Single-threaded)
- Global `dawn_metrics_t` instance
- Mutex protection on all metric updates
- No separate UI thread (updates in main loop)

### Future Multi-Client Architecture
- Metrics structure already has `pthread_mutex_t`
- Each worker thread updates metrics through mutex-protected functions
- TUI reads metrics safely through same mutex
- Activity log uses lock during append/read

### API Design

```c
// Thread-safe metric updates
void metrics_update_state(listeningState new_state);
void metrics_record_asr_timing(double ms, double rtf);
void metrics_record_llm_tokens(llm_type_t type, int input, int output, int cached);
void metrics_record_llm_timing(double ttft_ms, double total_ms);
void metrics_log_activity(const char *format, ...);

// Thread-safe reads for TUI
void metrics_get_snapshot(dawn_metrics_t *snapshot);
```

## Design Principles

1. **Zero impact when disabled** - No performance cost if using traditional logging
2. **Thread-safe from day 1** - Prepare for future multi-client architecture
3. **Export-friendly** - All stats designed to be easily serialized to JSON
4. **Extensible** - Easy to add new metrics without redesigning UI layout
5. **Accessible** - B/W theme ensures usability for everyone
6. **Fail gracefully** - Always fall back to working logging mode

## Future Enhancements (NOT NOW)

### MQTT Stats Integration 📝

**Note for later:** Design unified OASIS stats format for all systems

**Planned features:**
- Publish real-time metrics to `oasis/dawn/stats` topic
- Allow MIRAGE HUD to display DAWN status
- Cross-system performance correlation
- Centralized monitoring dashboard

**Requirements:**
- Define OASIS-wide stats schema (JSON format)
- Consistent topic structure across DAWN, MIRAGE, AURA, SPARK
- Versioned message format for backward compatibility

**Wait until:**
- Other OASIS systems are ready for stats integration
- Schema design discussion with full system context

## Testing Strategy

### Phase 1 Testing (Metrics)
- Unit tests for metrics collection functions
- Verify timing measurements are accurate (compare to manual timing)
- Test circular buffer wraparound
- Validate thread safety with stress testing

### Phase 2 Testing (TUI)
- Visual verification of all three color schemes
- Test on different terminal emulators (gnome-terminal, xterm, screen, tmux)
- Verify keyboard input handling
- Test screen refresh with varying terminal sizes

### Phase 3 Testing (Integration)
- End-to-end testing with real DAWN interactions
- Verify metrics update correctly during state transitions
- Test all latency measurements against known delays
- Validate token counting against API responses

### Phase 4 Testing (Polish)
- Load testing (extended sessions, high query rate)
- CPU usage profiling (ensure 10 Hz is acceptable)
- Memory leak testing (valgrind)
- Stats export validation (JSON structure, data accuracy)

## Success Criteria

- ✅ TUI displays accurately in real-time at 10 Hz
- ✅ All three color schemes render correctly
- ✅ Metrics collection has <1% CPU overhead
- ✅ Stats export contains accurate session data
- ✅ Debug log mode toggle works seamlessly
- ✅ Graceful degradation on non-TTY or missing ncurses
- ✅ No memory leaks over 24+ hour sessions
- ✅ Proper cleanup on all exit paths (normal, Ctrl+C, SIGTERM)

## Implementation Notes

### Instrumentation Points Summary

| Location | Metric | When |
|----------|--------|------|
| `src/dawn.c` | State changes | Every state transition |
| `src/asr/asr_whisper.c` | ASR timing, RTF | On final result |
| `src/asr/asr_vosk.c` | ASR timing | On final result |
| `src/asr/vad_silero.c` | VAD probability | Every inference |
| `src/llm/llm_openai.c` | Token counts | After response parsed |
| `src/llm/llm_claude.c` | Token counts, cache hits | After response parsed |
| `src/llm/llm_streaming.c` | TTFT | On first chunk callback |
| `src/llm/llm_streaming.c` | LLM total time | On stream complete |
| `src/tts/piper.cpp` | TTS timing | After generation |
| `src/dawn.c` | Activity log | User input and AI responses |

### Coding Standards

Follow `CODING_STYLE_GUIDE.md`:
- Use `snake_case` for functions and variables
- Use `UPPER_CASE` for constants and macros
- Return `SUCCESS` (0) or `FAILURE` (1)
- Always check return values
- Use Doxygen-style comments for public APIs
- Format with clang-format before committing

## Implementation Status

### Completed (Dec 2024)

#### Phase 1-4: Core TUI ✅
All original phases completed. TUI is fully functional with:
- Real-time metrics display at 10 Hz
- Three color themes (Green/Blue/B&W)
- Keyboard controls (Q, D, R, 1/2/3, ?)
- Activity log with circular buffer
- Session stats and performance metrics
- JSON export on exit

#### Recent Enhancements (Dec 3, 2024)

**DAWN Status Panel:**
- Added version display in panel title: `DAWN v1.0.0 (abc123)`
- Added AEC status line showing backend and calibration: `AEC: WebRTC (55ms, corr:0.74)`
- Added barge-in count display
- LLM display now shows provider and model: `Cloud (OpenAI: gpt-4o)`

**Real-Time Audio Panel:**
- Added "Heard:" field showing last ASR transcription (works with Whisper chunks)
- Shows what the system heard even without wake word detection

**Keyboard Controls:**
- Added Esc key as alternative quit (alongside Q)
- Added `L` key to switch to Local LLM
- Added `C` key to switch to Cloud LLM
- Updated footer hints: `[L]ocal  [C]loud  [D]ebug  [R]eset  [1/2/3] Theme  [Q]uit`
- Updated help overlay with new hotkeys

**Activity Log:**
- Long entries now wrap across multiple lines instead of truncating
- Maintains chronological order with newest entries visible

**Boot Sequence Improvements:**
- Barge-in detection disabled during initial TTS greeting and AEC calibration
- Audio buffer flushed after calibration to discard stale boot audio
- VAD state reset after calibration to prevent false triggers
- Prevents system from getting stuck in "listening" mode when background talking during startup

#### Phase 5: Text Input Mode ✅ (Dec 4, 2024)

**Objective:** Allow typing commands as alternative/parallel input to voice

**Implementation:**
- Press `I` to enter text input mode
- 512 character input buffer with visual cursor
- Character counter display in panel title bar
- Thread-safe text queue with mutex protection
- Arrow keys, Home/End, Delete key support
- Enter submits, Esc cancels

**UI:**
```
┌─ Text Input ─────────────────────────────────────────────────── 12/512 ─┐
│ > Hello Friday, what time is it?█                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

**Keyboard Controls in Input Mode:**
| Key | Action |
|-----|--------|
| `Enter` | Submit text to DAWN |
| `Esc` | Cancel input, return to normal TUI |
| `Backspace` | Delete character before cursor |
| `Delete` | Delete character at cursor |
| `←/→` | Move cursor |
| `Home/End` | Jump to start/end |
| Printable chars | Insert at cursor |

**File Changes:**
- `src/ui/tui.c`: Input mode state, buffer, panel rendering, keyboard handling
- `include/ui/tui.h`: Text queue API (`tui_has_text_input()`, `tui_get_text_input()`, `tui_is_input_mode()`)
- `src/dawn.c`: Poll text queue in SILENCE and WAKEWORD_LISTEN states

**Integration:**
- Main loop polls queue during idle states
- Text input bypasses VAD/ASR, goes directly to PROCESS_COMMAND
- Works same as voice commands (no wake word needed for typed input)

---

## References

- CLAUDE.md - Project architecture and development guidelines
- CODING_STYLE_GUIDE.md - Code formatting and style rules
- README.md - Build instructions and dependencies
- ARCHITECTURE.md - System design and component interactions
