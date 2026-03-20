# Phase 2 & 3 Implementation Plan: Streaming ASR with Advanced VAD

**Document Version:** 1.0
**Date:** November 18, 2025
**Status:** Architecture Design - Pending Review

---

## Executive Summary

This document outlines the implementation of intelligent chunking (Phase 2) and advanced Voice Activity Detection (Phase 3) for DAWN's ASR system. These phases are now considered **mandatory** following Phase 1's successful Whisper integration, as they enable robust operation across diverse acoustic environments and natural conversation flow.

### Key Requirements

1. **Phase 3 First** (VAD): Implement Silero VAD to replace RMS-based voice detection
2. **Phase 2 Second** (Chunking): Use VAD to enable intelligent chunking for long utterances
3. **Whisper-Specific**: Apply chunking only to batch-based ASR (Whisper), not streaming engines (Vosk)
4. **Two-Tier Approach**: Start with Option 1 (VAD-triggered chunking), upgrade to Option 3 (sliding window) if needed

---

## Architecture Overview

### Current DAWN Architecture (Phase 1)

```
┌─────────────────────────────────────────────────────────────┐
│                    DAWN Main Thread                          │
│                                                               │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────┐    │
│  │   State     │───▶│  RMS-based   │───▶│   Batch     │    │
│  │  Machine    │    │     VAD      │    │  Whisper    │    │
│  └─────────────┘    └──────────────┘    └─────────────┘    │
│        │                    │                     │          │
│        │                    ▼                     ▼          │
│        │         (Threshold comparison)    (30s fixed)      │
│        │                                                     │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Ring Buffer (Audio Capture Thread)          │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘

LIMITATIONS:
- RMS VAD: False triggers on background noise, misses soft speech
- Batch processing: No response until utterance fully completes
- Long utterances: 30+ second wait before any processing
```

### Proposed Architecture (Phase 2 + 3)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         DAWN Enhanced System                              │
│                                                                            │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────────┐        │
│  │   State     │───▶│  Silero VAD  │───▶│  Chunking Manager    │        │
│  │  Machine    │    │  (ML-based)  │    │  (Whisper-specific)  │        │
│  └─────────────┘    └──────────────┘    └──────────────────────┘        │
│        │                    │                      │                      │
│        │                    │                      ├─▶ Option 1:         │
│        │                    │                      │   VAD-triggered     │
│        │                    │                      │   (sentence chunks) │
│        │                    │                      │                      │
│        │                    │                      └─▶ Option 3:         │
│        │                    │                          Sliding window    │
│        │                    │                          (if needed)       │
│        │                    ▼                                             │
│        │         Speech/Silence probabilities                            │
│        │         (adaptive, no manual tuning)                            │
│        │                                                                  │
│  ┌───────────────────────────────────────────────────────────────┐      │
│  │           Ring Buffer (Audio Capture Thread)                   │      │
│  │  Continuous 16kHz mono PCM from PulseAudio/ALSA                │      │
│  └───────────────────────────────────────────────────────────────┘      │
│                                                                            │
│  ┌─────────────┐    ┌─────────────┐                                      │
│  │   Vosk      │    │   Whisper   │                                      │
│  │  (bypass    │    │  (chunked   │                                      │
│  │  chunking)  │    │  processing)│                                      │
│  └─────────────┘    └─────────────┘                                      │
└──────────────────────────────────────────────────────────────────────────┘

BENEFITS:
- Silero VAD: Robust speech detection across all environments
- Chunking: Progressive responses for long utterances
- Engine-aware: Vosk streams natively, Whisper gets chunked
```

---

## Phase 3: Silero VAD Integration (PRIORITY 1)

### Why Silero VAD?

| Feature | RMS (Current) | Silero VAD (Proposed) |
|---------|---------------|----------------------|
| **Accuracy** | Poor - triggers on any loud sound | Excellent - distinguishes speech from non-speech |
| **Adaptation** | Manual threshold tuning required | Self-adaptive to noise environment |
| **Model Size** | N/A | 1.8 MB (negligible) |
| **Inference Speed** | Instant | <1ms per 30ms chunk (real-time capable) |
| **Dependency** | None | ONNX Runtime (already in DAWN) |
| **Soft Speech** | Often missed | Detected reliably |
| **Background Noise** | Constant false positives | Robust filtering |
| **Commercial Use** | Everywhere | Google, Alexa-class quality (open-source) |

### Silero VAD C++ Integration Options

#### **Option A: Minimal Custom Wrapper** (RECOMMENDED)

**Approach:** Directly use ONNX Runtime with Silero VAD model (onnx file)

**Pros:**
- Minimal dependencies (only ONNX Runtime, which DAWN already has)
- Full control over integration
- No extra libraries or wrappers
- Simple implementation (~200 lines)

**Cons:**
- Need to handle ONNX Runtime API directly
- Manual audio preprocessing (resampling if needed)

**Implementation:**
```c
// vad_silero.h
typedef struct silero_vad_context silero_vad_context_t;

silero_vad_context_t* silero_vad_init(const char* model_path, int sample_rate);
float silero_vad_process(silero_vad_context_t* ctx, const float* audio, size_t samples);
void silero_vad_reset(silero_vad_context_t* ctx);
void silero_vad_cleanup(silero_vad_context_t* ctx);
```

**Files needed:**
- `silero_vad.c/h` - ONNX Runtime wrapper for Silero VAD
- `silero_vad_v5.onnx` - Pre-trained model (~1.8MB)

**Key References:**
- Official C++ example: https://github.com/snakers4/silero-vad/tree/master/examples/cpp
- Model download: https://github.com/snakers4/silero-vad/releases

#### **Option B: RealTimeCutVADCXXLibrary**

**Approach:** Use pre-built C++ library with WebRTC APM integration

**Pros:**
- Includes WebRTC Audio Processing Module (noise suppression, AGC)
- Ready-to-use callback interface
- Well-tested production code

**Cons:**
- Extra dependency (WebRTC APM)
- DAWN already has ReSpeaker hardware AEC (may be redundant)
- More complex than needed
- iOS/Android focus (may need adaptation for Linux)

**Verdict:** **REJECT** - WebRTC APM is overkill given DAWN's hardware AEC

#### **Option C: whisper.cpp VAD Integration**

**Approach:** Use whisper.cpp's built-in VAD support

**Investigation:** whisper.cpp's `stream` example supports `--vad` flag

**Pros:**
- Already integrated with whisper.cpp
- Proven combination

**Cons:**
- Tied to whisper.cpp streaming architecture
- May not work with Vosk or future ASR engines
- Less control over VAD behavior

**Verdict:** **CONSIDER** - Good for Phase 2 Option 3 (sliding window), but Phase 3 VAD should be engine-agnostic

### Recommended Approach: **Option A (Custom Wrapper)**

Implement minimal Silero VAD wrapper using ONNX Runtime directly:

1. **Simplicity**: ~200 lines, no extra dependencies
2. **Reusability**: Works with any ASR engine (Vosk, Whisper, future)
3. **Control**: Full control over thresholds, state management
4. **Performance**: Direct ONNX Runtime access, no overhead

---

## Phase 2: Intelligent Chunking Architecture

### Option 1: VAD-Triggered Chunking (START HERE)

**Concept:** Process audio on sentence boundaries detected by Silero VAD

```
User speaks: "Hello Friday, turn on the lights and play some music"
             ^              ^                     ^                ^
             |              |                     |                |
         Speech start   Pause (500ms)        Pause (500ms)    Speech end

Processing:
  Chunk 1: "Hello Friday," → Whisper → LLM (immediate response)
  Chunk 2: "turn on the lights" → Whisper → Execute command
  Chunk 3: "and play some music" → Whisper → Execute command

Timeline:
  0.0s - User starts speaking
  2.5s - First pause detected → Process chunk 1
  3.0s - LLM responds: "Yes, sir?"
  5.5s - Second pause → Process chunk 2
  6.0s - Lights turn on
  8.5s - Speech ends → Process chunk 3
  9.0s - Music starts
```

**Implementation Architecture:**

```c
// Chunking manager state
typedef enum {
    CHUNK_IDLE,          // No active recording
    CHUNK_RECORDING,     // Recording in progress
    CHUNK_PROCESSING     // Chunk being transcribed
} chunk_state_t;

typedef struct {
    chunk_state_t state;
    float *audio_buffer;     // Accumulating audio
    size_t buffer_size;      // Current buffer size
    size_t buffer_capacity;  // Max buffer size

    // VAD state tracking
    float silence_duration;  // How long silence detected
    float speech_duration;   // How long speech detected

    // Configuration
    float min_chunk_duration;    // Don't chunk before 1.0s
    float pause_threshold;       // Chunk after 500ms silence
    float max_chunk_duration;    // Force chunk after 10s
} chunking_manager_t;
```

**Algorithm:**

```c
void process_audio_frame(chunking_manager_t *cm, const float *audio, size_t samples) {
    // Run Silero VAD on this frame
    float speech_prob = silero_vad_process(vad_ctx, audio, samples);
    bool is_speech = (speech_prob > 0.5);  // Threshold

    if (is_speech) {
        // Accumulate audio
        append_to_buffer(cm, audio, samples);
        cm->speech_duration += frame_duration;
        cm->silence_duration = 0;

        // Force chunk if too long
        if (cm->speech_duration >= cm->max_chunk_duration) {
            LOG_INFO("Forcing chunk after %.1fs continuous speech",
                     cm->speech_duration);
            process_chunk(cm);
        }
    } else {
        cm->silence_duration += frame_duration;

        // Chunk on pause (if we have enough audio)
        if (cm->speech_duration >= cm->min_chunk_duration &&
            cm->silence_duration >= cm->pause_threshold) {
            LOG_INFO("Pause detected, processing chunk (%.1fs speech)",
                     cm->speech_duration);
            process_chunk(cm);
        }
    }
}

void process_chunk(chunking_manager_t *cm) {
    // Send accumulated audio to Whisper
    asr_result_t *result = asr_finalize(asr_ctx);

    // Send to LLM immediately (don't wait for more speech)
    send_to_llm(result->text);

    // Reset for next chunk
    clear_buffer(cm);
    cm->speech_duration = 0;
    cm->silence_duration = 0;
    asr_reset(asr_ctx);
}
```

**Configuration Parameters:**

```c
#define MIN_CHUNK_DURATION_SEC  1.0   // Don't chunk before 1 second
#define PAUSE_THRESHOLD_SEC     0.5   // Chunk after 500ms silence
#define MAX_CHUNK_DURATION_SEC 10.0   // Force chunk every 10 seconds
#define SPEECH_PROBABILITY_THRESHOLD 0.5  // Silero VAD threshold
```

**Integration with DAWN State Machine:**

```c
// In main loop (dawn.c)
switch (recState) {
    case COMMAND_RECORDING:
        // Read from ring buffer
        size_t bytes_read = audio_capture_read(capture_ctx, buffer, buffer_size);

        // Convert int16 to float for VAD
        convert_pcm16_to_float(buffer, float_buffer, bytes_read / 2);

        // Process through chunking manager (Whisper only!)
        if (asr_engine == ASR_ENGINE_WHISPER) {
            chunking_manager_process(chunk_mgr, float_buffer, samples);
        } else {
            // Vosk: stream directly (no chunking)
            asr_process_partial(asr_ctx, (int16_t*)buffer, samples);
        }
        break;
}
```

**Pros:**
- Natural conversation flow (responds at sentence boundaries)
- Simple to implement (~300 lines)
- Low CPU overhead (VAD is <1ms per frame)
- Works great for 90% of conversations

**Cons:**
- Long monologues without pauses still wait 10s (max_chunk_duration)
- Doesn't handle mid-sentence clarity (user pauses to think)

**Fallback Trigger:** If user feedback indicates frequent 10-second forced chunks, upgrade to Option 3

---

### Option 3: Sliding Window with Overlap (FALLBACK)

**Concept:** Process overlapping windows continuously, merge results

```
Audio timeline: [=============30 seconds of speech=============]

Windows:
  Window 1: [0-5s]                → "Hello Friday turn on the..."
  Window 2:    [3-8s]             → "turn on the lights and play..."
  Window 3:       [6-11s]         → "lights and play some music..."
  Window 4:          [9-14s]      → "music and set the temperature..."

Timeline:
  0.0s - Speech starts
  2.0s - Window 1 processes → Emit "Hello Friday"
  4.0s - Window 2 processes → Emit "turn on the lights" (merge/dedup)
  6.0s - Window 3 processes → Emit "and play some music" (merge/dedup)
  ...continuous updates every 2 seconds
```

**Implementation Architecture:**

```c
typedef struct {
    float *window_buffer;        // Current 5-second window
    size_t window_samples;       // Samples in window

    float *overlap_buffer;       // Previous 2.5s (for next window)
    size_t overlap_samples;

    char *previous_transcript;   // Last result (for deduplication)

    // Timing
    uint64_t last_process_time;
    uint64_t process_interval_ms;  // 2000ms = every 2 seconds
} sliding_window_t;
```

**Algorithm:**

```c
void sliding_window_process(sliding_window_t *sw, const float *audio, size_t samples) {
    // Accumulate into window
    append_to_window(sw, audio, samples);

    // Check if it's time to process (every 2 seconds)
    uint64_t now = get_timestamp_ms();
    if (now - sw->last_process_time >= sw->process_interval_ms) {
        // Send current 5-second window to Whisper
        asr_result_t *result = asr_process_window(asr_ctx,
                                                   sw->window_buffer,
                                                   sw->window_samples);

        // Deduplicate against previous result
        char *new_text = deduplicate_transcript(result->text,
                                                 sw->previous_transcript);

        if (new_text && strlen(new_text) > 0) {
            LOG_INFO("Sliding window emits: %s", new_text);
            send_to_llm_partial(new_text);
        }

        // Update state
        free(sw->previous_transcript);
        sw->previous_transcript = strdup(result->text);
        sw->last_process_time = now;

        // Slide window (keep last 2.5s for overlap)
        slide_window(sw);
    }
}

char* deduplicate_transcript(const char *current, const char *previous) {
    // Find longest common suffix of previous and prefix of current
    // Return only the NEW portion of current
    // Example:
    //   previous: "Hello Friday turn on the"
    //   current:  "turn on the lights and play"
    //   common:   "turn on the"
    //   return:   "lights and play"

    if (!previous) return strdup(current);

    // Use longest common substring algorithm
    // Return allocated string with new portion only
    // ... implementation details ...
}
```

**Configuration:**

```c
#define WINDOW_SIZE_SEC     5.0   // 5-second windows
#define WINDOW_OVERLAP_SEC  2.5   // 50% overlap
#define PROCESS_INTERVAL_MS 2000  // Update every 2 seconds
```

**Pros:**
- Continuous updates (no 10s wait)
- Best for long monologues
- Context preserved across windows

**Cons:**
- 2-3x CPU usage (processing overlapping audio)
- Complex deduplication logic
- Potential for repetition if dedup fails
- Higher memory usage

**When to Implement:** Only if Option 1 proves insufficient in real-world testing

---

## Implementation Order & Timeline

### Phase 3: Silero VAD (Week 1-2)

**Week 1: VAD Integration**

1. **Download Silero VAD model**
   ```bash
   cd /home/jetson/code/The-OASIS-Project/dawn
   mkdir -p models
   wget https://github.com/snakers4/silero-vad/releases/download/v5.0/silero_vad.onnx \
        -O models/silero_vad_v5.onnx
   ```

2. **Implement `silero_vad.c/h`**
   - Initialize ONNX Runtime session with Silero model
   - `silero_vad_process()` - Run inference on 30ms audio chunks
   - Return speech probability (0.0 = silence, 1.0 = definite speech)

3. **Create abstraction layer: `vad_interface.c/h`**
   - Support both RMS (legacy) and Silero VAD
   - Runtime selection via config/flag
   - Unified API for state machine

**Week 2: VAD Testing & Tuning**

4. **Replace RMS VAD in `dawn.c`**
   - Modify SILENCE → WAKEWORD_LISTEN transition
   - Modify COMMAND_RECORDING → PROCESS_COMMAND transition
   - Log both RMS and Silero in parallel initially

5. **Test in diverse environments**
   - Quiet room (baseline)
   - Fan noise (constant background)
   - HVAC cycling (variable background)
   - Outdoor (wind, traffic)
   - Multiple speakers (rejection test)

6. **Tune Silero threshold**
   - Default: 0.5 (50% probability)
   - Adjust based on false positive/negative rates
   - Create test suite with recordings

**Deliverables:**
- `silero_vad.c/h` - ONNX Runtime wrapper
- `vad_interface.c/h` - VAD abstraction layer
- `models/silero_vad_v5.onnx` - Model file
- Test recordings + results report

---

### Phase 2: Intelligent Chunking (Week 3-4)

**Week 3: Option 1 Implementation**

1. **Create `chunking_manager.c/h`**
   - Implement VAD-triggered chunking algorithm
   - Audio buffer management
   - Chunk detection logic (pause threshold, max duration)

2. **Integrate with ASR abstraction layer**
   - Modify `asr_interface.c` to support chunking callbacks
   - Add `asr_process_chunk()` API
   - Handle chunk boundaries cleanly

3. **Modify DAWN state machine**
   - Enable chunking for Whisper only
   - Vosk bypasses chunking (direct streaming)
   - Add CHUNK_PROCESSING sub-state?

**Week 4: Testing & Refinement**

4. **Test with long utterances**
   - 30-second monologues
   - Multi-sentence commands
   - Conversational back-and-forth

5. **Tune chunking parameters**
   - Min chunk duration (start: 1.0s)
   - Pause threshold (start: 0.5s)
   - Max chunk duration (start: 10.0s)

6. **Monitor for issues**
   - False chunking (interrupts mid-sentence)
   - Delayed chunking (waits too long)
   - Memory leaks (buffer growth)

**Deliverables:**
- `chunking_manager.c/h` - Chunking implementation
- Modified `asr_interface.c` with chunking support
- Modified `dawn.c` state machine
- Test suite with 30+ long utterances
- Performance report (latency, accuracy)

**Option 3 Decision Point:**
- If >20% of test cases trigger max_chunk_duration (10s forced chunks)
- OR user feedback indicates poor responsiveness
- THEN implement Option 3 (Sliding Window) in Week 5-6

---

## Architecture Integration Points

### File Structure

```
dawn/
├── silero_vad.c/h               # NEW: Silero VAD ONNX wrapper
├── vad_interface.c/h            # NEW: VAD abstraction layer
├── chunking_manager.c/h         # NEW: Intelligent chunking
├── asr_interface.c/h            # MODIFIED: Add chunking support
├── dawn.c                       # MODIFIED: Integrate VAD + chunking
├── models/
│   └── silero_vad_v5.onnx      # NEW: VAD model (~1.8MB)
└── tests/
    ├── test_vad.c               # NEW: VAD unit tests
    └── test_chunking.c          # NEW: Chunking integration tests
```

### CMakeLists.txt Changes

```cmake
# Phase 3: Silero VAD support
set(DAWN_SOURCES
    ${DAWN_SOURCES}
    silero_vad.c
    vad_interface.c
)

# Phase 2: Chunking support (Whisper only)
set(DAWN_SOURCES
    ${DAWN_SOURCES}
    chunking_manager.c
)

# Already have ONNX Runtime from Piper TTS - reuse!
# No new dependencies needed
```

### API Design

```c
// vad_interface.h - VAD Abstraction
typedef enum {
    VAD_ENGINE_RMS,      // Legacy RMS-based
    VAD_ENGINE_SILERO    // ML-based Silero VAD
} vad_engine_type_t;

typedef struct vad_context vad_context_t;

vad_context_t* vad_init(vad_engine_type_t type, const char *model_path);
float vad_process(vad_context_t *ctx, const float *audio, size_t samples);
void vad_cleanup(vad_context_t *ctx);

// chunking_manager.h - Chunking Manager
typedef struct chunking_manager chunking_manager_t;

typedef void (*chunk_callback_t)(const float *audio, size_t samples, void *user_data);

chunking_manager_t* chunking_manager_init(vad_context_t *vad_ctx,
                                          chunk_callback_t callback,
                                          void *user_data);
void chunking_manager_process(chunking_manager_t *cm,
                              const float *audio,
                              size_t samples);
void chunking_manager_cleanup(chunking_manager_t *cm);
```

---

## Testing Strategy

### Phase 3: VAD Testing

**Test Suite 1: Noise Robustness**
- 20 recordings in quiet environment (baseline)
- 20 recordings with fan noise (constant background)
- 20 recordings with HVAC (variable background)
- 20 recordings outdoors (wind, traffic)
- Metric: False positive rate, false negative rate

**Test Suite 2: Speech Characteristics**
- 10 soft-spoken utterances (low volume)
- 10 loud utterances (high volume)
- 10 fast speech patterns
- 10 slow speech patterns
- Metric: Detection accuracy vs RMS baseline

**Test Suite 3: Edge Cases**
- Sudden loud noises (door slam, dropped object)
- Music playback in background
- Multiple speakers (rejection test)
- Whispered commands
- Metric: Robustness score (did system crash/fail?)

### Phase 2: Chunking Testing

**Test Suite 4: Long Utterances**
- 10 monologues (30+ seconds)
- 10 multi-sentence commands
- 10 conversational exchanges
- Metric: Time to first response, total response time

**Test Suite 5: Chunking Accuracy**
- Verify no lost audio at chunk boundaries
- Compare chunked vs batch transcription WER
- Measure chunk size distribution
- Metric: WER delta, chunk count, chunk size stats

**Test Suite 6: Real-World Scenarios**
- Command + explanation (e.g., "Turn on lights because it's dark")
- Interrupted speech (user pauses mid-sentence)
- Rapid-fire commands ("Lights on, music on, temperature up")
- Metric: User experience rating (subjective)

---

## Success Criteria

### Phase 3: Silero VAD

- [ ] False positive rate <5% in noisy environments
- [ ] False negative rate <2% for soft speech
- [ ] No manual threshold tuning required for different environments
- [ ] <1ms inference time per 30ms audio chunk
- [ ] Stable operation over 24+ hour stress test

### Phase 2: Intelligent Chunking (Option 1)

- [ ] Time to first response <2 seconds for multi-sentence utterances
- [ ] Chunked transcription WER within 2% of batch WER
- [ ] >80% of chunks align with natural sentence boundaries
- [ ] Max forced chunks <20% (prefer natural pauses)
- [ ] No audio loss at chunk boundaries (verified by test suite)

### Phase 2: Sliding Window (Option 3 - if implemented)

- [ ] Progressive updates every 2 seconds
- [ ] Deduplication accuracy >95% (minimal repetition)
- [ ] CPU usage <70% during continuous speech
- [ ] Memory stable over extended operation

---

## Risk Mitigation

### Risk 1: ONNX Runtime Version Conflicts

**Issue:** DAWN uses ONNX Runtime for Piper TTS. Silero VAD requires compatible version.

**Mitigation:**
- Verify ONNX Runtime version compatibility (>=1.16.1 recommended)
- Test Silero VAD with DAWN's existing ONNX Runtime build
- If incompatible, upgrade ONNX Runtime and retest Piper TTS

### Risk 2: Chunking Introduces Latency

**Issue:** Processing chunks adds overhead vs single batch.

**Mitigation:**
- Measure end-to-end latency in all test cases
- Set strict latency budgets (<2s time to first response)
- If latency unacceptable, adjust chunk parameters or revert

### Risk 3: Deduplication Failures (Option 3)

**Issue:** Sliding window may emit duplicate text.

**Mitigation:**
- Implement robust longest-common-substring algorithm
- Test extensively with overlapping speech patterns
- Allow user to disable deduplication if problematic

### Risk 4: Vosk Regression

**Issue:** Refactoring might break Vosk streaming support.

**Mitigation:**
- Implement chunking ONLY for Whisper (engine-specific)
- Test Vosk with and without chunking enabled
- Maintain separate code paths for batch vs streaming ASR

---

## Future Enhancements (Post-Phase 2/3)

1. **Adaptive Thresholds**
   - Learn optimal chunk parameters from user behavior
   - Adjust pause threshold based on speaking rate
   - Personalized VAD sensitivity

2. **Multi-Speaker Support**
   - Use Silero VAD + speaker diarization
   - Route different speakers to different conversation contexts
   - Requires speaker embedding model (out of scope for now)

3. **Emotion Detection**
   - Analyze acoustic features during VAD processing
   - Detect user frustration/urgency
   - Adjust response tone accordingly

4. **GPU Acceleration**
   - Move Silero VAD inference to GPU (if available)
   - Enable Whisper small with CUDA (best accuracy)
   - Requires CUDA build testing on Jetson

---

## References

- Silero VAD Official: https://github.com/snakers4/silero-vad
- ONNX Runtime C++ API: https://onnxruntime.ai/docs/api/c/
- whisper.cpp Streaming: https://github.com/ggml-org/whisper.cpp/tree/master/examples/stream
- Silero VAD Paper: "One Voice Detector to Rule Them All" (https://thegradient.pub/one-voice-detector-to-rule-them-all/)

---

## Appendix: Code Examples

### Example 1: Minimal Silero VAD Wrapper

```c
// silero_vad.c
#include "silero_vad.h"
#include <onnxruntime/onnxruntime_c_api.h>

struct silero_vad_context {
    const OrtApi *ort_api;
    OrtEnv *env;
    OrtSession *session;
    OrtMemoryInfo *memory_info;

    float *h;  // Hidden state
    float *c;  // Cell state
    int64_t sr; // Sample rate
};

silero_vad_context_t* silero_vad_init(const char *model_path, int sample_rate) {
    silero_vad_context_t *ctx = calloc(1, sizeof(*ctx));

    // Initialize ONNX Runtime
    ctx->ort_api = OrtGetApiBase()->GetApi(ORT_API_VERSION);
    ctx->ort_api->CreateEnv(ORT_LOGGING_LEVEL_WARNING, "silero_vad", &ctx->env);

    // Load model
    OrtSessionOptions *session_options;
    ctx->ort_api->CreateSessionOptions(&session_options);
    ctx->ort_api->CreateSession(ctx->env, model_path, session_options, &ctx->session);

    // Initialize state
    ctx->sr = sample_rate;
    ctx->h = calloc(2 * 64, sizeof(float));  // Hidden state
    ctx->c = calloc(2 * 64, sizeof(float));  // Cell state

    return ctx;
}

float silero_vad_process(silero_vad_context_t *ctx, const float *audio, size_t samples) {
    // Prepare inputs: [audio, sr, h, c]
    // Run inference
    // Extract output probability
    // Update hidden/cell state
    // Return speech probability (0.0-1.0)

    // ... ONNX Runtime inference code ...

    return speech_probability;
}
```

### Example 2: VAD-Triggered Chunking

```c
// chunking_manager.c
void chunking_manager_process(chunking_manager_t *cm, const float *audio, size_t samples) {
    float speech_prob = vad_process(cm->vad_ctx, audio, samples);
    bool is_speech = (speech_prob > cm->speech_threshold);

    float frame_duration = (float)samples / cm->sample_rate;

    if (is_speech) {
        // Accumulate audio
        if (cm->state == CHUNK_IDLE) {
            cm->state = CHUNK_RECORDING;
            LOG_INFO("Speech detected, start recording chunk");
        }

        append_audio(cm->audio_buffer, audio, samples);
        cm->speech_duration += frame_duration;
        cm->silence_duration = 0;

        // Force chunk if too long
        if (cm->speech_duration >= cm->max_chunk_duration) {
            LOG_INFO("Max chunk duration reached, processing chunk");
            finalize_chunk(cm);
        }
    } else {
        cm->silence_duration += frame_duration;

        // Chunk on pause
        if (cm->state == CHUNK_RECORDING &&
            cm->speech_duration >= cm->min_chunk_duration &&
            cm->silence_duration >= cm->pause_threshold) {
            LOG_INFO("Pause detected (%.2fs silence), processing chunk",
                     cm->silence_duration);
            finalize_chunk(cm);
        }
    }
}

static void finalize_chunk(chunking_manager_t *cm) {
    if (cm->audio_buffer_size == 0) return;

    LOG_INFO("Finalizing chunk: %.2fs of audio", cm->speech_duration);

    // Invoke callback with accumulated audio
    cm->chunk_callback(cm->audio_buffer, cm->audio_buffer_size, cm->user_data);

    // Reset for next chunk
    cm->audio_buffer_size = 0;
    cm->speech_duration = 0;
    cm->silence_duration = 0;
    cm->state = CHUNK_IDLE;
}
```

---

**END OF IMPLEMENTATION PLAN**
