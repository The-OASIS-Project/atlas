# Phase 2 & 3 Revised Implementation Plan: Real-Time Silero VAD + Intelligent Chunking

**Document Version:** 2.0 (Revised after whisper.cpp investigation)
**Date:** November 18, 2025
**Status:** Architecture Design - Ready for Review

---

## Executive Summary

This document outlines the implementation of real-time Silero VAD (Phase 3) and intelligent chunking (Phase 2) for DAWN's ASR system. Following investigation of whisper.cpp's VAD capabilities, we determined that a **custom streaming Silero VAD wrapper is required** to meet DAWN's real-time needs.

### Key Changes from v1.0

1. **whisper.cpp investigation complete**: Built-in Silero VAD is batch-only, unsuitable for streaming
2. **Simplified architecture**: Single Silero VAD handles all use cases (no hybrid approach)
3. **Four critical VAD use cases identified**:
   - Wake word detection (speech starts)
   - Speech end detection (command complete)
   - Pause detection (chunking boundaries)
   - **Interruption detection** (user interrupts TTS playback)
4. **Vosk compatibility confirmed**: Vosk uses same VAD for state machine, bypasses chunking
5. **No TTS queueing needed**: Chunking is for performance only, not progressive LLM responses

---

## Architecture Overview

### Four Critical VAD Use Cases in DAWN

```
┌─────────────────────────────────────────────────────────────────────┐
│                   DAWN Main Loop with Silero VAD                     │
│                                                                       │
│  Every 30ms audio chunk:                                             │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ Silero VAD: audio → speech_probability (0.0-1.0)           │     │
│  └────────────────────────────────────────────────────────────┘     │
│                              ↓                                        │
│         ┌────────────────────┴────────────────────┐                  │
│         │                                          │                  │
│         ▼                                          ▼                  │
│  ┌─────────────────┐                    ┌──────────────────┐        │
│  │ State Machine   │                    │ TTS Playback     │        │
│  │ Transitions     │                    │ Monitor          │        │
│  └─────────────────┘                    └──────────────────┘        │
│         │                                          │                  │
│         ├─ Use Case 1: SILENCE → WAKEWORD         │                  │
│         │   (Speech detected, prob > 0.5)         │                  │
│         │                                          │                  │
│         ├─ Use Case 2: RECORDING → PROCESS        │                  │
│         │   (Silence > 1.5s, prob < 0.3)          │                  │
│         │                                          │                  │
│         ├─ Use Case 3: Pause Detection            │                  │
│         │   (Chunk boundary, prob < 0.3)          │                  │
│         │                                          │                  │
│         └────────────────────────────────┬─────────┘                  │
│                                          │                            │
│                      Use Case 4: Interruption Detection              │
│                      (User speaks during TTS, prob > 0.5)            │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Use Case Details

**Use Case 1: Wake Word Detection** (Speech Starts)
- State: `SILENCE` → `WAKEWORD_LISTEN`
- Current: RMS exceeds background threshold
- Problem: False triggers on door slams, HVAC, keyboard clicks
- Solution: `speech_prob > 0.5` = actual speech detected
- Benefit: Robust in noisy environments (helmet, outdoor, variable background)

**Use Case 2: Speech End Detection** (Command Complete)
- State: `COMMAND_RECORDING` → `PROCESS_COMMAND`
- Current: RMS below threshold for 2-3 seconds
- Problem:
  - Background noise causes premature cutoff mid-sentence
  - Soft speech at end missed → awkward waiting
- Solution: `speech_prob < 0.3` for 1.5 seconds = confident silence
- Benefit: Reliable command boundary detection, faster response time

**Use Case 3: Pause Detection** (Chunking Boundaries - Whisper Only)
- State: Within `COMMAND_RECORDING`, Whisper engine
- Current: N/A (batch processing entire utterance)
- Problem: 30+ second utterances take too long to process (10+ seconds latency)
- Solution: `speech_prob < 0.3` for 500ms = natural sentence pause → chunk here
- Benefit: Break long utterances into manageable pieces, maintain responsiveness

**Use Case 4: Interruption Detection** (NEW)
- State: TTS playback active
- Current: User must wait for TTS to finish before speaking again
- Problem: Unnatural conversation flow, user frustration
- Solution: `speech_prob > 0.5` during TTS playback → stop TTS, return to `WAKEWORD_LISTEN`
- Benefit: Natural conversation interruption (like human dialogue)

---

## Why Custom Silero Implementation (Not whisper.cpp)

### whisper.cpp Investigation Results

**✅ Found:** whisper.cpp includes Silero VAD support
- Model: `ggml-silero-v5.1.2.bin` (GGML format, ~1.8MB)
- API: `whisper_vad_detect_speech()`, `whisper_vad_segments_from_probs()`
- Example: `examples/vad-speech-segments/speech.cpp`

**❌ Problem:** whisper.cpp's VAD is **batch-oriented**, not suitable for DAWN

```c
// whisper.cpp VAD workflow (BATCH ONLY)
float *complete_audio = load_wav_file("recording.wav");  // Need FULL file

whisper_vad_detect_speech(vad_ctx, complete_audio, n_samples);  // Process all

whisper_vad_segments *segments = whisper_vad_segments_from_probs(vad_ctx, params);
// Returns: segment 0: t0=0.5s, t1=2.3s (timestamps AFTER processing)
```

**Why this doesn't work for DAWN:**
1. Requires complete audio upfront (can't detect speech end in real-time)
2. Returns timestamps after-the-fact (can't make state decisions during recording)
3. No incremental API (can't process 30ms chunks continuously)
4. `stream.cpp` example uses `vad_simple()` (energy-based), NOT Silero

**Conclusion:** Must implement custom Silero ONNX wrapper for streaming use.

---

## Phase 3: Real-Time Silero VAD Implementation

### Objective

Replace RMS-based voice detection with ML-based Silero VAD that provides **continuous, real-time speech probability** for all four use cases.

### Implementation: Custom ONNX Runtime Wrapper

**Files to Create:**
- `vad_silero.c` - Silero VAD implementation using ONNX Runtime
- `vad_silero.h` - Public API for DAWN integration
- `models/silero_vad.onnx` - Silero VAD v5 model (~1.8MB)

**API Design:**

```c
// vad_silero.h

#ifndef VAD_SILERO_H
#define VAD_SILERO_H

#include <stddef.h>

/**
 * @brief Silero VAD context (opaque)
 */
typedef struct silero_vad_context silero_vad_context_t;

/**
 * @brief Initialize Silero VAD
 *
 * Loads ONNX model and initializes inference session. Model must be
 * Silero VAD v5 in ONNX format.
 *
 * @param model_path Path to silero_vad.onnx model file
 * @param sample_rate Audio sample rate (8000 or 16000 Hz)
 * @return VAD context, or NULL on failure
 */
silero_vad_context_t *silero_vad_init(const char *model_path, int sample_rate);

/**
 * @brief Process audio chunk and return speech probability
 *
 * Processes a single chunk of audio (typically 30ms = 480 samples at 16kHz)
 * through Silero VAD neural network. Maintains internal state across calls.
 *
 * Thread safety: NOT thread-safe. Call from single thread only.
 *
 * @param ctx VAD context
 * @param audio Audio samples (float, normalized to [-1.0, 1.0])
 * @param samples Number of samples (typically 480 for 30ms at 16kHz)
 * @return Speech probability [0.0 = silence, 1.0 = definite speech]
 */
float silero_vad_process(silero_vad_context_t *ctx, const float *audio, size_t samples);

/**
 * @brief Reset VAD internal state
 *
 * Resets hidden/cell state tensors. Call between utterances or when
 * starting fresh (e.g., after interruption).
 *
 * @param ctx VAD context
 */
void silero_vad_reset(silero_vad_context_t *ctx);

/**
 * @brief Clean up VAD context
 *
 * Frees ONNX session, model, and internal state. Context invalid after this.
 *
 * @param ctx VAD context
 */
void silero_vad_cleanup(silero_vad_context_t *ctx);

#endif  // VAD_SILERO_H
```

**Implementation Details (vad_silero.c):**

```c
// Internal structure
struct silero_vad_context {
    const OrtApi *ort_api;
    OrtEnv *env;
    OrtSession *session;
    OrtMemoryInfo *memory_info;

    // Model inputs
    float *input_audio;      // Audio chunk buffer
    int64_t sr;              // Sample rate (8000 or 16000)
    float *h;                // Hidden state (2 * 64 floats)
    float *c;                // Cell state (2 * 64 floats)

    // Model outputs
    float *output_prob;      // Speech probability
    float *hn;               // Next hidden state
    float *cn;               // Next cell state

    int sample_rate;
};

silero_vad_context_t *silero_vad_init(const char *model_path, int sample_rate) {
    // 1. Allocate context
    silero_vad_context_t *ctx = calloc(1, sizeof(*ctx));

    // 2. Initialize ONNX Runtime
    ctx->ort_api = OrtGetApiBase()->GetApi(ORT_API_VERSION);
    ctx->ort_api->CreateEnv(ORT_LOGGING_LEVEL_WARNING, "silero_vad", &ctx->env);

    // 3. Create session options
    OrtSessionOptions *session_options;
    ctx->ort_api->CreateSessionOptions(&session_options);

    // 4. Load model
    ctx->ort_api->CreateSession(ctx->env, model_path, session_options, &ctx->session);

    // 5. Initialize state tensors
    ctx->h = calloc(2 * 64, sizeof(float));  // Hidden state
    ctx->c = calloc(2 * 64, sizeof(float));  // Cell state
    ctx->sr = sample_rate;

    // 6. Allocate I/O buffers
    ctx->input_audio = calloc(512, sizeof(float));  // Max chunk size
    ctx->output_prob = calloc(1, sizeof(float));
    ctx->hn = calloc(2 * 64, sizeof(float));
    ctx->cn = calloc(2 * 64, sizeof(float));

    return ctx;
}

float silero_vad_process(silero_vad_context_t *ctx, const float *audio, size_t samples) {
    // 1. Copy audio to input buffer
    memcpy(ctx->input_audio, audio, samples * sizeof(float));

    // 2. Create input tensors: [audio, sr, h, c]
    OrtValue *input_tensors[4];

    // audio tensor: shape [1, samples]
    int64_t audio_shape[] = {1, samples};
    ctx->ort_api->CreateTensorWithDataAsOrtValue(
        ctx->memory_info, ctx->input_audio, samples * sizeof(float),
        audio_shape, 2, ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT, &input_tensors[0]);

    // sr tensor: shape [1] (scalar)
    int64_t sr_shape[] = {1};
    ctx->ort_api->CreateTensorWithDataAsOrtValue(
        ctx->memory_info, &ctx->sr, sizeof(int64_t),
        sr_shape, 1, ONNX_TENSOR_ELEMENT_DATA_TYPE_INT64, &input_tensors[1]);

    // h tensor: shape [2, 1, 64]
    int64_t state_shape[] = {2, 1, 64};
    ctx->ort_api->CreateTensorWithDataAsOrtValue(
        ctx->memory_info, ctx->h, 2 * 64 * sizeof(float),
        state_shape, 3, ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT, &input_tensors[2]);

    // c tensor: shape [2, 1, 64]
    ctx->ort_api->CreateTensorWithDataAsOrtValue(
        ctx->memory_info, ctx->c, 2 * 64 * sizeof(float),
        state_shape, 3, ONNX_TENSOR_ELEMENT_DATA_TYPE_FLOAT, &input_tensors[3]);

    // 3. Run inference
    const char *input_names[] = {"input", "sr", "h", "c"};
    const char *output_names[] = {"output", "hn", "cn"};
    OrtValue *output_tensors[3];

    ctx->ort_api->Run(ctx->session, NULL,
                      input_names, input_tensors, 4,
                      output_names, 3, output_tensors);

    // 4. Extract speech probability
    float *output_data;
    ctx->ort_api->GetTensorMutableData(output_tensors[0], (void **)&output_data);
    float speech_prob = output_data[0];

    // 5. Update hidden/cell state for next iteration
    float *hn_data, *cn_data;
    ctx->ort_api->GetTensorMutableData(output_tensors[1], (void **)&hn_data);
    ctx->ort_api->GetTensorMutableData(output_tensors[2], (void **)&cn_data);
    memcpy(ctx->h, hn_data, 2 * 64 * sizeof(float));
    memcpy(ctx->c, cn_data, 2 * 64 * sizeof(float));

    // 6. Release tensors
    for (int i = 0; i < 4; i++) ctx->ort_api->ReleaseValue(input_tensors[i]);
    for (int i = 0; i < 3; i++) ctx->ort_api->ReleaseValue(output_tensors[i]);

    return speech_prob;
}

void silero_vad_reset(silero_vad_context_t *ctx) {
    memset(ctx->h, 0, 2 * 64 * sizeof(float));
    memset(ctx->c, 0, 2 * 64 * sizeof(float));
}

void silero_vad_cleanup(silero_vad_context_t *ctx) {
    if (!ctx) return;

    ctx->ort_api->ReleaseSession(ctx->session);
    ctx->ort_api->ReleaseEnv(ctx->env);

    free(ctx->input_audio);
    free(ctx->output_prob);
    free(ctx->h);
    free(ctx->c);
    free(ctx->hn);
    free(ctx->cn);
    free(ctx);
}
```

**Model Acquisition:**

```bash
# Download Silero VAD v5 ONNX model
cd /home/jetson/code/The-OASIS-Project/dawn/models
wget https://github.com/snakers4/silero-vad/releases/download/v5.0/silero_vad.onnx
```

**Memory Footprint:**
- Model: 1.8 MB (loaded once)
- Context: ~2 KB (state tensors, buffers)
- Per-inference: <100 KB (temporary tensors)
- Total: ~2 MB (negligible on Jetson)

**Performance:**
- Inference time: <1ms per 30ms chunk (tested on similar hardware)
- Real-time factor: 0.03 (processes 30ms audio in <1ms)
- CPU usage: <5% on Jetson (single core)

---

## Integration with DAWN State Machine

### Modified Main Loop (dawn.c)

```c
// Global VAD context (initialized once)
static silero_vad_context_t *vad_ctx = NULL;

// Timing trackers
static float silence_duration = 0.0f;
static float speech_duration = 0.0f;

int main(int argc, char **argv) {
    // ... existing initialization ...

    // Initialize Silero VAD
    vad_ctx = silero_vad_init("models/silero_vad.onnx", 16000);
    if (!vad_ctx) {
        LOG_ERROR("Failed to initialize Silero VAD");
        return FAILURE;
    }

    // ... main loop ...

    while (!get_quit()) {
        // Read audio from ring buffer (500ms chunks)
        size_t bytes_read = audio_capture_read(capture_ctx, pcm_buffer, buffer_size);
        size_t samples = bytes_read / sizeof(int16_t);

        // Convert int16 to float for VAD
        for (size_t i = 0; i < samples; i++) {
            float_buffer[i] = pcm_buffer[i] / 32768.0f;
        }

        // Process through VAD in 30ms chunks
        for (size_t offset = 0; offset < samples; offset += 480) {  // 480 = 30ms at 16kHz
            size_t chunk_size = (offset + 480 <= samples) ? 480 : (samples - offset);

            float speech_prob = silero_vad_process(vad_ctx, float_buffer + offset, chunk_size);

            // Use Case 1: Wake Word Detection
            if (recState == SILENCE && speech_prob > SPEECH_THRESHOLD) {
                LOG_INFO("Speech detected (prob=%.2f), entering WAKEWORD_LISTEN", speech_prob);
                recState = WAKEWORD_LISTEN;
                silenceNextState = COMMAND_RECORDING;
                silero_vad_reset(vad_ctx);  // Fresh start for new utterance
                silence_duration = 0.0f;
                speech_duration = 0.0f;
            }

            // Use Case 2: Speech End Detection
            if (recState == COMMAND_RECORDING) {
                if (speech_prob < SILENCE_THRESHOLD) {
                    silence_duration += 0.03f;  // 30ms

                    if (silence_duration >= END_OF_SPEECH_DURATION) {
                        LOG_INFO("Speech ended (%.1fs silence), processing command", silence_duration);
                        recState = PROCESS_COMMAND;
                    }
                } else {
                    silence_duration = 0.0f;  // Reset on any speech
                    speech_duration += 0.03f;
                }

                // Use Case 3: Pause Detection for Chunking (Whisper only)
                if (asr_engine == ASR_ENGINE_WHISPER) {
                    if (speech_prob < SILENCE_THRESHOLD &&
                        speech_duration >= MIN_CHUNK_DURATION &&
                        silence_duration >= CHUNK_PAUSE_DURATION) {
                        LOG_INFO("Pause detected (%.1fs), chunking here", silence_duration);
                        chunk_and_accumulate();
                        silence_duration = 0.0f;
                    }

                    // Force chunk on max duration
                    if (speech_duration >= MAX_CHUNK_DURATION) {
                        LOG_INFO("Max chunk duration reached, forcing chunk");
                        chunk_and_accumulate();
                        speech_duration = 0.0f;
                    }
                }
            }

            // Use Case 4: Interruption Detection
            pthread_mutex_lock(&tts_mutex);
            if (tts_playback_state == TTS_PLAYBACK_PLAY && speech_prob > INTERRUPTION_THRESHOLD) {
                LOG_INFO("User interruption detected (prob=%.2f), stopping TTS", speech_prob);
                tts_playback_state = TTS_PLAYBACK_STOP;
                pthread_mutex_unlock(&tts_mutex);

                // Return to listening state
                recState = WAKEWORD_LISTEN;
                silenceNextState = COMMAND_RECORDING;
                silero_vad_reset(vad_ctx);

                // Clear any partial recordings
                audio_capture_clear(capture_ctx);
            } else {
                pthread_mutex_unlock(&tts_mutex);
            }
        }

        // ... rest of state machine logic ...
    }

    // Cleanup
    silero_vad_cleanup(vad_ctx);

    return SUCCESS;
}
```

### Configuration Parameters (dawn.h or config file)

```c
// VAD Thresholds (tunable)
#define SPEECH_THRESHOLD 0.5f           // Prob > 0.5 = speech detected
#define SILENCE_THRESHOLD 0.3f          // Prob < 0.3 = confident silence
#define INTERRUPTION_THRESHOLD 0.6f     // Prob > 0.6 = intentional interruption

// Timing Parameters
#define END_OF_SPEECH_DURATION 1.5f     // 1.5s silence = command complete
#define MIN_CHUNK_DURATION 1.0f         // Don't chunk before 1s speech
#define CHUNK_PAUSE_DURATION 0.5f       // 500ms pause = sentence boundary
#define MAX_CHUNK_DURATION 10.0f        // Force chunk every 10s (safety)

// VAD Processing
#define VAD_CHUNK_SIZE_MS 30            // Process VAD every 30ms
#define VAD_CHUNK_SAMPLES 480           // 30ms at 16kHz
```

---

## Phase 2: Intelligent Chunking for Whisper

### Objective

Use Silero VAD pause detection to chunk long Whisper utterances into manageable pieces, improving responsiveness while maintaining accuracy.

### Chunking Strategy

**When to chunk:**
1. Natural pause detected: `speech_prob < 0.3` for 500ms
2. Minimum chunk size met: `speech_duration >= 1.0s`
3. OR maximum duration reached: `speech_duration >= 10.0s` (force chunk)

**How to chunk:**
1. Accumulate audio in buffer during `COMMAND_RECORDING`
2. On pause/max duration: finalize current chunk
3. Send chunk to `asr_finalize()` for transcription
4. Accumulate transcription text
5. Reset ASR context for next chunk: `asr_reset()`
6. Continue recording

**When recording ends:**
1. Finalize last chunk (if any pending audio)
2. Concatenate all chunk transcriptions
3. Send complete text to LLM

### Implementation (Whisper-Specific)

**File: chunking_manager.c/h** (Whisper backend integration)

```c
// chunking_manager.h

#ifndef CHUNKING_MANAGER_H
#define CHUNKING_MANAGER_H

#include "asr_interface.h"

/**
 * @brief Chunking manager context
 */
typedef struct chunking_manager chunking_manager_t;

/**
 * @brief Initialize chunking manager
 *
 * @param asr_ctx ASR context (Whisper)
 * @return Chunking manager, or NULL on failure
 */
chunking_manager_t *chunking_manager_init(asr_context_t *asr_ctx);

/**
 * @brief Add audio to current chunk
 *
 * Accumulates audio into current chunk buffer. Call this continuously
 * during COMMAND_RECORDING state.
 *
 * @param cm Chunking manager
 * @param audio Audio samples (int16)
 * @param samples Number of samples
 * @return SUCCESS or FAILURE
 */
int chunking_manager_add_audio(chunking_manager_t *cm, const int16_t *audio, size_t samples);

/**
 * @brief Finalize current chunk and get transcription
 *
 * Sends accumulated audio to ASR for transcription, resets chunk buffer.
 * Call this when pause detected or max duration reached.
 *
 * @param cm Chunking manager
 * @param text_out Output transcription (caller must free)
 * @return SUCCESS or FAILURE
 */
int chunking_manager_finalize_chunk(chunking_manager_t *cm, char **text_out);

/**
 * @brief Get complete concatenated transcription
 *
 * Returns all chunk transcriptions concatenated. Call after command recording ends.
 *
 * @param cm Chunking manager
 * @return Complete text (caller must free), or NULL if no chunks
 */
char *chunking_manager_get_complete_text(chunking_manager_t *cm);

/**
 * @brief Reset chunking manager for new utterance
 *
 * @param cm Chunking manager
 */
void chunking_manager_reset(chunking_manager_t *cm);

/**
 * @brief Clean up chunking manager
 *
 * @param cm Chunking manager
 */
void chunking_manager_cleanup(chunking_manager_t *cm);

#endif  // CHUNKING_MANAGER_H
```

```c
// chunking_manager.c

struct chunking_manager {
    asr_context_t *asr_ctx;

    // Audio accumulation
    int16_t *audio_buffer;
    size_t buffer_samples;
    size_t buffer_capacity;  // Max: 15s at 16kHz = 240,000 samples

    // Transcription accumulation
    char **chunk_texts;      // Array of chunk transcriptions
    size_t num_chunks;
    size_t chunk_capacity;
};

chunking_manager_t *chunking_manager_init(asr_context_t *asr_ctx) {
    chunking_manager_t *cm = calloc(1, sizeof(*cm));
    cm->asr_ctx = asr_ctx;

    // Preallocate audio buffer (15 seconds max)
    cm->buffer_capacity = 15 * 16000;  // 15s at 16kHz
    cm->audio_buffer = calloc(cm->buffer_capacity, sizeof(int16_t));

    // Preallocate chunk array (max 30 chunks = 15s / 0.5s pauses)
    cm->chunk_capacity = 30;
    cm->chunk_texts = calloc(cm->chunk_capacity, sizeof(char *));

    return cm;
}

int chunking_manager_add_audio(chunking_manager_t *cm, const int16_t *audio, size_t samples) {
    // Check for buffer overflow
    if (cm->buffer_samples + samples > cm->buffer_capacity) {
        LOG_ERROR("Chunking: Audio buffer overflow (max 15s)");
        return FAILURE;
    }

    // Append audio
    memcpy(cm->audio_buffer + cm->buffer_samples, audio, samples * sizeof(int16_t));
    cm->buffer_samples += samples;

    return SUCCESS;
}

int chunking_manager_finalize_chunk(chunking_manager_t *cm, char **text_out) {
    if (cm->buffer_samples == 0) {
        *text_out = NULL;
        return SUCCESS;  // No audio to process
    }

    LOG_INFO("Chunking: Finalizing chunk (%zu samples, %.2fs)",
             cm->buffer_samples, cm->buffer_samples / 16000.0f);

    // Send accumulated audio to ASR
    asr_result_t *result = asr_finalize(cm->asr_ctx);
    if (!result || !result->text) {
        LOG_ERROR("Chunking: ASR finalize failed");
        return FAILURE;
    }

    // Save transcription
    if (cm->num_chunks >= cm->chunk_capacity) {
        LOG_ERROR("Chunking: Too many chunks (max %zu)", cm->chunk_capacity);
        asr_result_free(result);
        return FAILURE;
    }

    cm->chunk_texts[cm->num_chunks++] = strdup(result->text);
    *text_out = strdup(result->text);

    LOG_INFO("Chunking: Chunk %zu transcribed: \"%s\"", cm->num_chunks, result->text);

    // Reset for next chunk
    cm->buffer_samples = 0;
    asr_reset(cm->asr_ctx);
    asr_result_free(result);

    return SUCCESS;
}

char *chunking_manager_get_complete_text(chunking_manager_t *cm) {
    if (cm->num_chunks == 0) return NULL;

    // Calculate total length
    size_t total_len = 0;
    for (size_t i = 0; i < cm->num_chunks; i++) {
        total_len += strlen(cm->chunk_texts[i]);
        if (i < cm->num_chunks - 1) total_len += 1;  // Space between chunks
    }

    // Concatenate
    char *complete = calloc(total_len + 1, sizeof(char));
    for (size_t i = 0; i < cm->num_chunks; i++) {
        strcat(complete, cm->chunk_texts[i]);
        if (i < cm->num_chunks - 1) strcat(complete, " ");
    }

    LOG_INFO("Chunking: Complete transcription (%zu chunks): \"%s\"",
             cm->num_chunks, complete);

    return complete;
}

void chunking_manager_reset(chunking_manager_t *cm) {
    cm->buffer_samples = 0;

    for (size_t i = 0; i < cm->num_chunks; i++) {
        free(cm->chunk_texts[i]);
    }
    cm->num_chunks = 0;
}

void chunking_manager_cleanup(chunking_manager_t *cm) {
    if (!cm) return;

    free(cm->audio_buffer);

    for (size_t i = 0; i < cm->num_chunks; i++) {
        free(cm->chunk_texts[i]);
    }
    free(cm->chunk_texts);

    free(cm);
}
```

### Integration Example (COMMAND_RECORDING state)

```c
// dawn.c - COMMAND_RECORDING case

case COMMAND_RECORDING:
    // Read audio from ring buffer
    size_t bytes_read = audio_capture_read(capture_ctx, pcm_buffer, buffer_size);

    if (asr_engine == ASR_ENGINE_WHISPER) {
        // Add to chunking manager
        chunking_manager_add_audio(chunk_mgr, (int16_t *)pcm_buffer, bytes_read / 2);

        // Check if pause detected (from VAD processing above)
        if (pause_detected) {
            char *chunk_text = NULL;
            if (chunking_manager_finalize_chunk(chunk_mgr, &chunk_text) == SUCCESS) {
                LOG_INFO("Chunk: %s", chunk_text);
                free(chunk_text);
            }
        }
    } else {
        // Vosk: stream directly (no chunking)
        asr_process_partial(asr_ctx, (int16_t *)pcm_buffer, bytes_read / 2);
    }
    break;

case PROCESS_COMMAND:
    char *final_text = NULL;

    if (asr_engine == ASR_ENGINE_WHISPER) {
        // Finalize any pending chunk
        char *chunk_text = NULL;
        chunking_manager_finalize_chunk(chunk_mgr, &chunk_text);
        free(chunk_text);

        // Get complete concatenated text
        final_text = chunking_manager_get_complete_text(chunk_mgr);
        chunking_manager_reset(chunk_mgr);
    } else {
        // Vosk: get final result
        asr_result_t *result = asr_finalize(asr_ctx);
        final_text = result ? strdup(result->text) : NULL;
        asr_result_free(result);
    }

    if (final_text) {
        LOG_INFO("Final transcription: %s", final_text);
        // Send to LLM...
        free(final_text);
    }
    break;
```

---

## Vosk Compatibility (Verified Safe)

### Vosk Does NOT Use Chunking

**Vosk has native streaming recognition:**
- `vosk_recognizer_accept_waveform()` - Feed audio continuously
- `vosk_recognizer_partial_result()` - Get partial text during speech
- `vosk_recognizer_final_result()` - Get complete text when done

**Chunking is Whisper-specific:**
```c
if (asr_engine == ASR_ENGINE_WHISPER) {
    // Use chunking manager
} else {  // Vosk
    // Stream directly to Vosk (no chunking)
}
```

**Vosk DOES use Silero VAD:**
- For state machine transitions (same as Whisper)
- `SILENCE` → `WAKEWORD_LISTEN` (speech detected)
- `COMMAND_RECORDING` → `PROCESS_COMMAND` (silence detected)
- Interruption detection (stop TTS on user speech)

**No conflicts, clean separation.**

---

## Implementation Timeline

### Week 1: Silero VAD Core Implementation
- Day 1-2: Implement `vad_silero.c` (ONNX Runtime wrapper)
- Day 3: Unit tests (`test_vad.c`)
- Day 4-5: Integration with DAWN main loop (Use Cases 1 & 2)

**Deliverables:**
- `vad_silero.c/h` - Working Silero VAD
- `models/silero_vad.onnx` - Model file
- Wake word detection + speech end detection working

### Week 2: VAD Testing & Tuning
- Day 1-2: Test in diverse environments (quiet, noisy, outdoor, helmet)
- Day 3: Tune thresholds (`SPEECH_THRESHOLD`, `SILENCE_THRESHOLD`, `END_OF_SPEECH_DURATION`)
- Day 4: Implement interruption detection (Use Case 4)
- Day 5: Long-running stability test (24 hours)

**Deliverables:**
- Environment test results (false positive/negative rates)
- Optimized threshold configuration
- Working interruption detection

### Week 3: Whisper Chunking Implementation
- Day 1-2: Implement `chunking_manager.c/h`
- Day 3: Integrate pause detection (Use Case 3)
- Day 4-5: Test with 30+ second utterances

**Deliverables:**
- `chunking_manager.c/h` - Pause-based chunking
- Chunking working for Whisper (Vosk bypasses)

### Week 4: Integration Testing & Documentation
- Day 1-2: End-to-end testing (wake word → command → chunking → LLM → TTS → interruption)
- Day 3: Performance benchmarking (RTF, latency, CPU usage)
- Day 4: Code review + address feedback
- Day 5: Documentation updates (CLAUDE.md, DAWN_ASR_UPGRADE_PLAN.md)

**Deliverables:**
- Complete Phase 2 + 3 implementation
- Test results report
- Updated documentation

---

## Testing Strategy

### Phase 3: VAD Testing

**Test Suite 1: Wake Word Detection (Use Case 1)**
- 20 recordings with wake word in quiet environment
- 20 recordings with wake word + fan noise
- 20 recordings with wake word + outdoor noise
- 10 recordings WITHOUT wake word (rejection test)
- **Metric**: False positive rate <5%, false negative rate <2%

**Test Suite 2: Speech End Detection (Use Case 2)**
- 20 commands with natural pauses mid-sentence
- 20 commands that trail off (soft speech at end)
- 20 commands with sudden stop
- **Metric**: End detection accuracy >95%, average latency <2s

**Test Suite 3: Interruption Detection (Use Case 4)**
- 10 scenarios: speak during TTS playback
- 10 scenarios: don't interrupt (TTS plays fully)
- **Metric**: Interruption detected <500ms, no false triggers

**Test Suite 4: Noise Robustness**
- Door slams, keyboard clicks, HVAC cycling
- **Metric**: <3% false triggers on non-speech sounds

### Phase 2: Chunking Testing

**Test Suite 5: Long Utterances**
- 10 monologues (30-60 seconds)
- 10 multi-sentence commands
- **Metric**: <5% forced chunks (max duration), >90% pause-based chunks

**Test Suite 6: Chunking Accuracy**
- Compare chunked vs batch transcription WER
- Verify no audio lost at boundaries
- **Metric**: WER delta <2%, no audio loss

**Test Suite 7: Real-World Scenarios**
- Command + explanation
- Interrupted speech (user pauses mid-sentence)
- Rapid-fire commands
- **Metric**: User satisfaction survey

---

## Success Criteria

### Phase 3: Silero VAD
- [ ] False positive rate <5% in noisy environments
- [ ] False negative rate <2% for soft speech
- [ ] Speech end detection latency <2 seconds (avg)
- [ ] Interruption detection latency <500ms
- [ ] No manual threshold tuning needed per environment
- [ ] <1ms VAD inference time per 30ms chunk
- [ ] Stable 24+ hour operation (no memory leaks)

### Phase 2: Chunking (Whisper)
- [ ] >80% of chunks align with natural pauses
- [ ] <20% forced chunks (max duration trigger)
- [ ] Chunked WER within 2% of batch WER
- [ ] No audio loss at chunk boundaries (verified)
- [ ] Average response latency <3s for 30s utterances
- [ ] Vosk unaffected (no regression in Vosk mode)

### Integration
- [ ] All four use cases working (wake word, speech end, chunking, interruption)
- [ ] Works with both Vosk and Whisper (engine-aware)
- [ ] User can interrupt TTS naturally
- [ ] Long utterances (60s+) handled gracefully

---

## Risk Mitigation

### Risk 1: ONNX Runtime Conflicts with Piper TTS

**Issue**: Both Silero VAD and Piper TTS use ONNX Runtime. Potential version conflicts or session management issues.

**Mitigation:**
- Verify ONNX Runtime version compatibility (Piper uses 1.16+)
- Share single `OrtEnv` between Piper and Silero (if possible)
- Test both TTS and VAD active simultaneously
- Profile memory usage (Piper ~100MB + Silero 2MB = acceptable)

**Investigation Step:**
```bash
grep -r "OrtEnv\|CreateEnv" text_to_speech.cpp piper.cpp
```

### Risk 2: VAD Inference Latency Blocks Main Thread

**Issue**: If Silero VAD takes >10ms per inference, could delay audio processing.

**Mitigation:**
- Benchmark VAD on Jetson under load (Whisper + Piper + VAD concurrent)
- If latency >5ms consistently, consider separate VAD thread
- Set alert threshold: LOG_WARNING if VAD >5ms
- Optimization: Process VAD only every 60ms instead of 30ms (acceptable for state transitions)

### Risk 3: Chunking Introduces Audio Artifacts

**Issue**: Cutting audio at VAD-detected pauses might split words.

**Mitigation:**
- Add hysteresis: Only chunk if silence sustained for 500ms (not single 30ms frame)
- Add safety margin: Extend chunk boundaries by 100ms before/after pause
- Test extensively with recordings
- Fall back to batch processing if chunking causes accuracy issues

### Risk 4: Interruption Detection Too Sensitive

**Issue**: Background speech (TV, other people) triggers interruption.

**Mitigation:**
- Higher threshold for interruption: `INTERRUPTION_THRESHOLD = 0.6` (vs 0.5 for wake word)
- Require sustained speech: Detect speech for 200ms before interrupting
- User preference: Make interruption detection optional (config flag)
- Test with background conversation recordings

---

## File Structure

```
dawn/
├── vad_silero.c/h               # NEW: Silero VAD ONNX wrapper
├── chunking_manager.c/h         # NEW: Whisper chunking logic
├── asr_interface.c/h            # MODIFIED: Chunking integration
├── dawn.c                       # MODIFIED: VAD + chunking state machine
├── CMakeLists.txt               # MODIFIED: Add new files
├── models/
│   ├── silero_vad.onnx         # NEW: Silero VAD model (~1.8MB)
│   └── ggml-base.bin           # Existing: Whisper model
├── tests/
│   ├── test_vad.c              # NEW: VAD unit tests
│   └── test_chunking.c         # NEW: Chunking integration tests
└── test_recordings/             # Existing: Test audio samples
```

### CMakeLists.txt Changes

```cmake
# Add Silero VAD source
set(DAWN_SOURCES
    ${DAWN_SOURCES}
    vad_silero.c
    chunking_manager.c
)

# ONNX Runtime already linked for Piper TTS - reuse!
# No new dependencies needed

# Copy VAD model to build directory
file(COPY ${CMAKE_SOURCE_DIR}/models/silero_vad.onnx
     DESTINATION ${CMAKE_BINARY_DIR}/models/)
```

---

## References

- **Silero VAD Official**: https://github.com/snakers4/silero-vad
- **Silero VAD ONNX Model**: https://github.com/snakers4/silero-vad/releases/tag/v5.0
- **ONNX Runtime C API**: https://onnxruntime.ai/docs/api/c/
- **whisper.cpp Investigation**: `/home/jetson/code/The-OASIS-Project/dawn/whisper.cpp/`
  - `examples/vad-speech-segments/` - Batch Silero VAD example
  - `examples/stream/` - Streaming example (uses energy VAD, not Silero)

---

## Appendix: Configuration File Example

```json
// config.json (future enhancement)
{
  "vad": {
    "model_path": "models/silero_vad.onnx",
    "speech_threshold": 0.5,
    "silence_threshold": 0.3,
    "interruption_threshold": 0.6,
    "end_of_speech_duration": 1.5,
    "enable_interruption": true
  },
  "chunking": {
    "enabled": true,
    "min_chunk_duration": 1.0,
    "chunk_pause_duration": 0.5,
    "max_chunk_duration": 10.0,
    "apply_to_vosk": false
  }
}
```

---

**END OF REVISED IMPLEMENTATION PLAN**
