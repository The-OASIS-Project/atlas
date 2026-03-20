# AEC Integration Guide for DAWN

This document describes the Acoustic Echo Cancellation (AEC) implementation in DAWN, enabling barge-in (interrupt) capability without requiring special hardware like ReSpeaker arrays.

## Backend Selection

DAWN uses WebRTC AEC3 for echo cancellation:

| Backend | Status | Notes |
|---------|--------|-------|
| **WebRTC AEC3** | ✅ Active | Auto delay estimation, excellent performance |

Build options:
```bash
cmake -DENABLE_AEC=ON ..   # Enable WebRTC AEC3 (default)
cmake -DENABLE_AEC=OFF ..  # Disable AEC entirely
```

> **Historical Note**: A Speex AEC backend was evaluated but removed due to
> ineffective echo cancellation. Speex's synchronous API doesn't handle the
> timing variability between ALSA playback and capture well, resulting in
> negative attenuation (output louder than input). WebRTC AEC3's internal
> delay estimation handles this automatically. See git history for the
> Speex implementation if needed.

## Overview

### The Problem
When DAWN speaks through speakers, the microphone picks up its own voice. Without echo cancellation:
- VAD triggers on DAWN's own speech
- Wake word detection may false-trigger
- User speech is mixed with DAWN's output

### The Solution
AEC uses a reference signal (what DAWN is playing) to subtract the echo from the microphone input, leaving only the user's voice.

## Architecture

### Native 48kHz Design

DAWN uses **native 48kHz audio capture** for optimal WebRTC AEC3 performance. This design was chosen because:

1. WebRTC AEC3 is optimized for 48kHz operation
2. Native capture eliminates mic-side resampling artifacts
3. Higher frequency resolution improves echo cancellation quality

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
                                                              │   → ASR (Vosk/Whisper)      │
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

### Key Constants

| Constant | Value | Location | Purpose |
|----------|-------|----------|---------|
| `AEC_SAMPLE_RATE` | 48000 | `aec_processor.h` | AEC processing rate |
| `CAPTURE_RATE` | 48000 | `audio_capture_thread.c` | Microphone capture rate |
| `ASR_RATE` | 16000 | `audio_capture_thread.c` | Vosk/Whisper input rate |
| `DEFAULT_RATE` | 22050 | `text_to_speech.cpp` | Piper TTS native output |
| `AEC_FRAME_SAMPLES` | 480 | `aec_processor.h` | 10ms frame at 48kHz |

### Resamplers (Only 2 Required)

| Location | From | To | Purpose |
|----------|------|-----|---------|
| `text_to_speech.cpp` | 22050 Hz | 48000 Hz | Upsample TTS for AEC reference |
| `audio_capture_thread.c` | 48000 Hz | 16000 Hz | Downsample AEC output for ASR |

### Compile-Time Validation

The system enforces `CAPTURE_RATE == AEC_SAMPLE_RATE` at compile time in `audio_capture_thread.c` to prevent architecture mismatches.

## Thread Model

### Thread Safety

- **Capture thread**: Reads mic at 48kHz, processes through AEC, downsamples to 16kHz
- **TTS thread**: Upsamples 22050Hz→48kHz, writes to AEC reference buffer (lock-free)
- **Main thread**: Reads 16kHz audio from ring buffer for VAD/ASR

The AEC processor uses:
- Per-frame locking (10ms granularity) for WebRTC API calls
- Lock-free ring buffer writes from TTS thread
- Atomic flags for initialization state

### Initialization Order

1. Initialize TTS (creates upsampling resampler)
2. Initialize AEC (creates WebRTC AudioProcessing instance)
3. Start audio capture (creates downsampling resampler)

## Configuration

### AEC Configuration (`aec_config_t`)

```c
typedef struct {
   bool enable_noise_suppression;          // Enable NS (adds CPU load)
   aec_ns_level_t noise_suppression_level; // NS aggressiveness
   bool enable_high_pass_filter;           // Remove DC offset
   bool mobile_mode;                       // Use AECM instead of AEC3 (lower CPU)
   size_t ref_buffer_ms;                   // Reference buffer size in ms (default: 500)
   int16_t noise_gate_threshold;           // Post-AEC noise gate (0=disabled)
   size_t acoustic_delay_ms;               // Speaker→mic delay (default: 70ms)
} aec_config_t;
```

### VAD Configuration (dawn.c)

```c
#define VAD_SPEECH_THRESHOLD 0.5f       // Normal threshold
#define VAD_SPEECH_THRESHOLD_TTS 0.92f  // Higher threshold during TTS
#define VAD_TTS_DEBOUNCE_COUNT 3        // Consecutive detections required
#define VAD_TTS_COOLDOWN_MS 1500        // Keep TTS threshold after TTS stops
```

## Files

### Core Implementation

| File | Purpose |
|------|---------|
| `include/audio/aec_processor.h` | Public API and constants |
| `src/audio/aec_webrtc.cpp` | WebRTC AEC3 backend (48kHz native) |
| `src/audio/audio_capture_thread.c` | 48kHz capture with AEC + downsampling |
| `src/tts/text_to_speech.cpp` | TTS with 22050→48kHz resampling for AEC |
| `include/audio/resampler.h` | Resampler interface |
| `src/audio/resampler.c` | libsamplerate-based implementation |

### Build Configuration

AEC is enabled/disabled via CMake (see Backend Selection above for options):

```bash
cmake -DENABLE_AEC=ON ..   # Enable WebRTC AEC3 (default)
cmake -DENABLE_AEC=OFF ..  # Disable AEC entirely
make -j4
```

## Tuning Guide

### Echo Not Fully Cancelled

**Symptom**: DAWN's voice still triggers VAD/wake word

**Possible causes**:
1. Reference buffer too small (audio delay > buffer size)
2. Acoustic path delay not aligned
3. ERLE not converging

**Fixes**:
- Increase `ref_buffer_ms` in config (try 750ms or 1000ms)
- Adjust `acoustic_delay_ms` to match your hardware (default: 70ms)
- Check AEC stats: `delay`, `atten`, `ERLE` values

### False Barge-Ins During TTS

**Symptom**: System interrupts during TTS playback

**Fixes**:
- Increase `VAD_SPEECH_THRESHOLD_TTS` to 0.95
- Increase `VAD_TTS_DEBOUNCE_COUNT` to 4
- Increase `VAD_TTS_COOLDOWN_MS` to 2000

### Missing Barge-Ins

**Symptom**: Can't interrupt DAWN when speaking

**Fixes**:
- Decrease `VAD_SPEECH_THRESHOLD_TTS` to 0.85
- Decrease `VAD_TTS_DEBOUNCE_COUNT` to 2
- Ensure microphone gain is adequate

### High CPU Usage

**Symptom**: System sluggish when AEC enabled

**Fixes**:
- Enable mobile mode: `config.mobile_mode = true`
- Disable noise suppression: `config.enable_noise_suppression = false`
- Reduce NS level: `config.noise_suppression_level = AEC_NS_LEVEL_LOW`

## Monitoring

### AEC Statistics

```c
aec_stats_t stats;
if (aec_get_stats(&stats) == 0) {
   // estimated_delay_ms: Should be ~50-70ms
   // ref_buffer_samples: ~4000-5000 during TTS
   // is_active: Should be true
   // frames_processed: Should increase during TTS
   // erle_db: Echo Return Loss Enhancement (higher = better)
}
```

### Log Output

During operation, the AEC logs periodic stats:

```
AEC3@48k: ERL=14.2dB ERLE=0.2dB delay=48ms atten=-19.8dB div=0.00 queued=102 read=2860 mic=397 ref=11726 out=345
```

| Field | Meaning | Good Values |
|-------|---------|-------------|
| `ERL` | Echo Return Loss (detection) | 10-20 dB |
| `ERLE` | Echo Return Loss Enhancement | >6 dB (see note below) |
| `delay` | Estimated acoustic delay | 150-200 ms typical |
| `atten` | Actual attenuation (mic RMS / out RMS) | Positive dB = reduction |
| `mic` | Input mic RMS | Varies |
| `ref` | Reference buffer RMS | **>0 during TTS** (critical!) |
| `out` | Output RMS after AEC | Should be << mic |

**Note on delay**: WebRTC auto-estimates delay at 150-200ms typically. This includes ALSA buffering, USB audio latency, and acoustic path. The old 40-80ms estimate was too conservative.

## Known Issues

### WebRTC Backend

1. **ERLE Metric Unreliable at 48kHz**: WebRTC's reported ERLE may show low values (0.2-2.8 dB) even when actual attenuation is excellent. Use the calculated `atten` field for real performance monitoring.

2. **Noise Suppression Disabled**: NS is disabled by default (`enable_noise_suppression = false`) because it causes "underwater" audio distortion. Enable only if you have specific noise issues and can tolerate some distortion.

3. **False VAD Triggers During TTS Transitions**: Can occur when TTS pauses between sentences or stops. Mitigated by VAD debounce and cooldown settings.

### Historical: Speex Backend (Removed)

A Speex AEC backend was evaluated in December 2025 but removed due to fundamental architectural incompatibilities:

1. **Synchronous API Mismatch**: Speex expects perfectly synchronized capture and playback streams. ALSA's timing variability between playback and capture caused chronic reference/capture desync.

2. **No Auto Delay Estimation**: Unlike WebRTC AEC3, Speex requires manual delay configuration. The actual system delay (150-200ms including ALSA buffering) varied too much for a static setting.

3. **Negative Attenuation**: The above issues caused Speex to often *amplify* rather than cancel echo, showing negative attenuation values (-2 to -4 dB).

The implementation is preserved in git history (see `src/audio/aec_speex.cpp` before commit removing Speex support) for reference if future hardware with better timing guarantees is used.

## Dependencies

- WebRTC audio processing >= 1.0 (freedesktop.org variant)
- libsamplerate (for high-quality resampling)
- ALSA or PulseAudio backend

```bash
# Install dependencies
sudo apt-get install libsamplerate0-dev libwebrtc-audio-processing-dev
```

## Performance

### Memory Footprint

| Component | Size |
|-----------|------|
| WebRTC AudioProcessing | ~100KB |
| Reference delay buffer | ~96KB (500ms @ 48kHz) |
| Resampler buffers | ~128KB |
| Capture AEC buffer | ~96KB |
| **Total** | **~420KB** |

### CPU Overhead

| Platform | Mobile Mode OFF | Mobile Mode ON |
|----------|-----------------|----------------|
| Jetson Orin | 5-10% | 3-5% |
| Raspberry Pi 4 | 15-25% | 8-12% |

### Latency

- AEC processing: ~10ms per frame
- Downsampling: ~1-2ms per chunk
- Total added latency: ~12-15ms
