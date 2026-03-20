# AEC Implementation Status

## Current State: WORKING - Native 48kHz Architecture

**Last Updated:** December 1, 2024

## Architecture Overview

The AEC system now uses **native 48kHz audio capture** for optimal WebRTC AEC3 performance. This eliminates the 16kHz processing limitation that was causing poor ERLE.

### Sample Rate Flow

```
Microphone (48kHz native) ──────────────────────────────────────────────────────┐
                                                                                │
                                                                                ▼
                                                              ┌─────────────────────────────┐
                                                              │   AEC Processor (48kHz)     │
                                                              │   - Echo cancellation       │
                                                              │   - Noise suppression       │
                                                              └─────────────────────────────┘
                                                                                │
                                                                                ▼
                                                              ┌─────────────────────────────┐
                                                              │  Downsample (48kHz→16kHz)   │
                                                              │  (in audio_capture_thread)  │
                                                              └─────────────────────────────┘
                                                                                │
                                                                                ▼
                                                              ┌─────────────────────────────┐
                                                              │   Ring Buffer (16kHz)       │
                                                              │   → ASR (Vosk @ 16kHz)      │
                                                              └─────────────────────────────┘

TTS Output (22050Hz Piper) ──────────────────────────────────────────────────────┐
                                                                                 │
                                                                                 ▼
                                                              ┌─────────────────────────────┐
                                                              │  Upsample (22050Hz→48kHz)   │
                                                              │  (in text_to_speech.cpp)    │
                                                              └─────────────────────────────┘
                                                                                 │
                                                                                 ▼
                                                              ┌─────────────────────────────┐
                                                              │   AEC Reference Buffer      │
                                                              │   (receives 48kHz directly) │
                                                              └─────────────────────────────┘
```

### Key Architectural Changes from Previous Version

| Aspect | Old (16kHz) | New (48kHz Native) |
|--------|-------------|-------------------|
| Mic capture rate | 16kHz | 48kHz |
| AEC processing rate | 48kHz (upsampled) | 48kHz (native) |
| Resamplers needed | 3 (mic up, mic down, ref up) | 2 (TTS up, AEC down) |
| TTS reference path | 16kHz → 48kHz in AEC | 22050Hz → 48kHz in TTS |

### Resampler Locations (Only 2 Required)

| Location | From | To | Purpose |
|----------|------|-----|---------|
| `text_to_speech.cpp` | 22050 Hz | 48000 Hz | Upsample TTS for AEC reference |
| `audio_capture_thread.c` | 48000 Hz | 16000 Hz | Downsample AEC output for ASR |

**Note:** The AEC processor itself no longer does any resampling - it receives and outputs 48kHz audio directly.

### Key Constants

| Constant | Value | Location | Purpose |
|----------|-------|----------|---------|
| `AEC_SAMPLE_RATE` | 48000 | `aec_processor.h:60` | AEC processing rate |
| `CAPTURE_RATE` | 48000 | `audio_capture_thread.c` | Microphone capture rate |
| `ASR_RATE` | 16000 | `audio_capture_thread.c` | Vosk ASR input rate |
| `DEFAULT_RATE` | 22050 | `text_to_speech.cpp:46` | Piper TTS native output |
| `AEC_FRAME_SAMPLES` | 480 | `aec_processor.h:73` | 10ms frame at 48kHz |

### Compile-Time Validation

The system enforces `CAPTURE_RATE == AEC_SAMPLE_RATE` at compile time in `audio_capture_thread.c` to prevent architecture mismatches.

## Test Results (December 1, 2024)

### Echo Attenuation Performance

```
AEC3@48k: delay=48ms atten=-29.5dB mic=44  ref=3984  out=1   → 97% reduction
AEC3@48k: delay=48ms atten=-20.6dB mic=328 ref=12743 out=31  → 90% reduction
AEC3@48k: delay=48ms atten=-38.6dB mic=474 ref=108   out=6   → 99% reduction
AEC3@48k: delay=48ms atten=-19.8dB mic=1432 ref=4670 out=147 → 90% reduction
AEC3@48k: delay=48ms atten=-24.7dB mic=956  ref=2156 out=56  → 94% reduction
```

### Performance Metrics

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| Delay Estimate | 48 ms | 50-70 ms | ✅ Stable and correct |
| Actual Attenuation | -19 to -38 dB | -10 to -20 dB | ✅ Excellent (90-99% reduction) |
| ERLE (WebRTC metric) | 0.2-2.8 dB | 10-20 dB | ⚠️ Metric unreliable but actual performance is good |
| Divergent Filter | 0.00 | <0.1 | ✅ Stable filter |

### Functional Results

- ✅ User speech correctly transcribed during TTS playback
- ✅ Echo cancellation working (90-99% reduction measured)
- ✅ Natural conversation flow maintained
- ✅ Real user speech passes through (not cancelled)

### Known Issues

1. **ERLE Metric Unreliable at 48kHz**: WebRTC's `echo_return_loss_enhancement` statistic shows 0.2-2.8 dB even when actual attenuation is 20-38 dB. Use the `atten` log field (calculated from mic/out RMS) for real performance monitoring.

2. **False VAD Triggers During TTS Transitions**: Some residual echo triggers VAD when:
   - TTS pauses between sentences
   - TTS stops and reference buffer drains
   - Example transcriptions: "Thank you - You're welcome", "situation awareness"

## VAD Configuration

Located in `dawn.c`:

```c
#define VAD_SPEECH_THRESHOLD 0.5f       // Normal threshold
#define VAD_SPEECH_THRESHOLD_TTS 0.85f  // Higher threshold during TTS
#define VAD_TTS_DEBOUNCE_COUNT 2        // Consecutive detections required
#define VAD_TTS_COOLDOWN_MS 1000        // Keep TTS threshold after TTS stops
```

### Tuning Recommendations

**For persistent false triggers:**
- Increase `VAD_TTS_DEBOUNCE_COUNT` to 3 or 4
- Increase `VAD_TTS_COOLDOWN_MS` to 1500-2000

**For missing barge-ins:**
- Decrease `VAD_SPEECH_THRESHOLD_TTS` to 0.80
- Decrease `VAD_TTS_DEBOUNCE_COUNT` to 1

**Acoustic delay tuning:**
- Default `acoustic_delay_ms`: 70ms (50ms ALSA buffer + 20ms acoustic path)
- Adjust based on hardware if echo cancellation timing is off

## Key Files

| File | Purpose |
|------|---------|
| `include/audio/aec_processor.h` | Public API and constants |
| `src/audio/aec_processor.cpp` | WebRTC AEC3 wrapper (48kHz native) |
| `src/audio/audio_capture_thread.c` | 48kHz capture with AEC + downsampling |
| `src/tts/text_to_speech.cpp` | TTS with 22050→48kHz resampling for AEC |
| `include/audio/resampler.h` | Resampler interface |
| `src/audio/resampler.c` | libsamplerate-based implementation |

## Historical Context

The previous implementation attempted 16kHz internal processing, but WebRTC AEC3 doesn't work properly at 16kHz (see historical section in previous version). The native 48kHz architecture resolves this fundamental limitation.
