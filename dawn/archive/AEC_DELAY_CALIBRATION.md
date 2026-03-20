# AEC Delay Calibration Design

## Overview

Automatic acoustic delay calibration using the boot greeting TTS to measure the actual speaker-to-microphone delay for optimal AEC performance.

## Problem

WebRTC AEC3 benefits from a delay hint to align the reference signal with the echo in the microphone input. The current default (70ms) is a rough estimate based on:
- ALSA buffer latency (~50ms)
- Acoustic path delay (~20ms)

However, actual delay varies by hardware, speaker placement, and room acoustics. Incorrect delay hints degrade echo cancellation quality.

## Solution

Use the boot greeting ("Hello sir" / "Good morning sir") as a calibration signal:
1. Capture TTS reference and mic input during greeting
2. Cross-correlate to find actual acoustic delay
3. Update AEC delay hint with measured value

## Architecture

### New Module: `aec_calibration.c`

Standalone module usable with the WebRTC AEC backend (or future alternatives).

```
include/audio/aec_calibration.h
src/audio/aec_calibration.c
```

### API

```c
/**
 * Initialize calibration system
 * @param sample_rate Audio sample rate (e.g., 48000)
 * @param max_delay_ms Maximum delay to search for (e.g., 200)
 * @return 0 on success
 */
int aec_cal_init(int sample_rate, int max_delay_ms);

/**
 * Start calibration capture
 * Called by TTS when greeting playback begins
 */
void aec_cal_start(void);

/**
 * Add reference samples during calibration
 * Called from aec_add_reference() path
 */
void aec_cal_add_reference(const int16_t *samples, size_t num_samples);

/**
 * Add mic samples during calibration
 * Called from aec_process() path
 */
void aec_cal_add_mic(const int16_t *samples, size_t num_samples);

/**
 * Stop calibration and compute delay
 * Called by TTS when greeting playback ends
 * @param delay_ms Output: measured delay in milliseconds
 * @return 0 on success, non-zero if correlation failed
 */
int aec_cal_finish(int *delay_ms);

/**
 * Check if calibration is in progress
 */
bool aec_cal_is_active(void);

/**
 * Cleanup calibration resources
 */
void aec_cal_cleanup(void);
```

## Implementation Details

### Buffer Requirements

At 48kHz with 200ms max delay search:
- Reference buffer: ~2 seconds = 96,000 samples (~192KB)
- Mic buffer: ~2 seconds + 200ms margin = 105,600 samples (~211KB)
- Total: ~400KB

### Cross-Correlation Algorithm

Use normalized cross-correlation to find delay:

```c
// For each possible delay d in [0, max_delay_samples]:
//   correlation[d] = sum(ref[i] * mic[i + d]) / sqrt(sum(ref^2) * sum(mic^2))
//
// Peak correlation index = estimated delay in samples
// Convert to ms: delay_ms = peak_index * 1000 / sample_rate
```

Optimization: Use FFT-based correlation for large buffers (optional).

### Confidence Check

Reject calibration if:
- Peak correlation < 0.3 (weak echo, possibly no speaker output)
- Multiple peaks with similar magnitude (ambiguous)
- Measured delay outside expected range (0-200ms)

### Integration Points

**TTS Module (`text_to_speech.cpp`):**
```cpp
// In greeting playback function:
void play_greeting(void) {
    aec_cal_start();

    // ... existing TTS playback code ...
    // aec_add_reference() calls will route to calibration

    // After playback completes:
    int measured_delay_ms;
    if (aec_cal_finish(&measured_delay_ms) == 0) {
        LOG_INFO("AEC calibration: measured delay = %d ms", measured_delay_ms);
        aec_set_delay_hint(measured_delay_ms);
    } else {
        LOG_WARNING("AEC calibration failed, using default delay");
    }
}
```

**AEC Module (`aec_webrtc.cpp`):**
```cpp
// In aec_add_reference():
void aec_add_reference(const int16_t *samples, size_t num_samples) {
    // Feed calibration if active
    if (aec_cal_is_active()) {
        aec_cal_add_reference(samples, num_samples);
    }
    // ... existing reference handling ...
}

// In aec_process():
void aec_process(const int16_t *mic_in, int16_t *clean_out, size_t num_samples) {
    // Feed calibration if active
    if (aec_cal_is_active()) {
        aec_cal_add_mic(mic_in, num_samples);
    }
    // ... existing AEC processing ...
}

// New function to update delay hint:
void aec_set_delay_hint(int delay_ms);
```

### Sequence Diagram

```
Boot
  |
  v
dawn.c: Initialize TTS, AEC
  |
  v
dawn.c: Call tts_speak_greeting()
  |
  v
TTS: aec_cal_start()
  |
  v
TTS: Generate audio chunks
  |   |
  |   +---> aec_add_reference() ---> aec_cal_add_reference()
  |   |
  |   +---> Audio to speaker
  |
  v
[Audio plays, echo returns to mic]
  |
  v
Capture thread: aec_process()
  |   |
  |   +---> aec_cal_add_mic()
  |
  v
TTS: Playback complete
  |
  v
TTS: aec_cal_finish(&delay_ms)
  |   |
  |   +---> Cross-correlate ref and mic buffers
  |   |
  |   +---> Return measured delay (or error)
  |
  v
TTS: aec_set_delay_hint(delay_ms)
  |
  v
AEC: Update g_acoustic_delay_ms
  |
  v
Normal operation with calibrated delay
```

## Fallback Behavior

If calibration fails:
1. Log warning with reason (low correlation, timeout, etc.)
2. Keep default delay (70ms for WebRTC, configurable)
3. Continue normal operation

Calibration failures are expected in:
- Very quiet rooms (no audible echo)
- Headphone use (no speaker output to mic)
- Muted speakers

## Configuration

Add to `aec_config_t`:
```c
bool enable_calibration;      // Enable auto-calibration (default: true)
int calibration_max_delay_ms; // Max delay to search (default: 200)
int calibration_timeout_ms;   // Max time to wait for correlation (default: 3000)
```

## Testing

1. **Unit test**: Feed known delayed signal, verify correct delay detection
2. **Integration test**: Boot with different speaker volumes, verify calibration
3. **Edge cases**:
   - Silent boot (no greeting) - should timeout gracefully
   - Very loud echo - should still find correct delay
   - Multiple echoes (reverberant room) - should find primary peak

## Future Enhancements

1. **Periodic recalibration**: Re-run during quiet periods to adapt to changes
2. **Per-device calibration**: Store measured delay in config file
3. **Multi-channel support**: Calibrate each mic channel independently
4. **Adaptive delay**: Track delay drift during operation

## Files to Create/Modify

**New files:**
- `include/audio/aec_calibration.h`
- `src/audio/aec_calibration.c`

**Modified files:**
- `src/audio/aec_webrtc.cpp` - Add calibration hooks, `aec_set_delay_hint()`
- `src/tts/text_to_speech.cpp` - Trigger calibration around greeting
- `include/audio/aec_processor.h` - Add config options, `aec_set_delay_hint()` declaration
- `CMakeLists.txt` - Add new source file

## Implementation Order

1. Create `aec_calibration.h/c` with buffer management and correlation
2. Add `aec_set_delay_hint()` to AEC modules
3. Hook calibration into `aec_add_reference()` and `aec_process()`
4. Add calibration trigger in TTS greeting playback
5. Test and tune correlation parameters
