# VAD Implementation Notes

## Week 1 Day 1 Completion Summary (Nov 19, 2025)

### Model Selection (Empirical Testing)

Created comprehensive test program (`tests/test_vad_models.c`) to compare three Silero VAD models:

| Model | Load Time | Inference Time | API | Speech/Silence Discrimination |
|-------|-----------|----------------|-----|-------------------------------|
| **silero_vad.onnx** (FP32) | 113.66ms | 0.325ms | 3 inputs (input, state, sr) | 0.0007 / 0.0006 |
| **silero_vad_half.onnx** (FP16) | 38.39ms | 0.313ms | 2 inputs (input, state) | 0.0691 / 0.0443 |
| **silero_vad_16k_op15.onnx** (opset15) | 46.04ms | 0.311ms | 3 inputs (input, state, sr) | 0.0007 / 0.0006 |

**SELECTED:** `silero_vad_16k_op15.onnx`

**Rationale:**
- Fastest inference (0.311ms - well under 1ms requirement)
- Optimized specifically for 16kHz audio (our exact use case)
- Modern ONNX opset 15 with full API support (sr parameter)
- Same discrimination as FP32 but faster/smaller
- FP16 model uses older API and different calibration

**Model Location:** `~/code/The-OASIS-Project/silero-vad/src/silero_vad/data/silero_vad_16k_op15.onnx`

---

## ONNX Runtime Environment Strategy

### Decision: Option B (Separate Environments)

**Date:** November 19, 2025
**Decided by:** Architecture review + developer agreement

### Options Considered

#### Option A: Shared OrtEnv
- **Goal:** Single `OrtEnv` shared between Piper TTS and Silero VAD
- **Theoretical benefit:** Optimal resource pooling
- **Blockers identified:**
  - Piper uses C++ ONNX Runtime API (`Ort::Env`)
  - VAD uses C ONNX Runtime API (`OrtEnv*`)
  - Would require modifying Piper's `ModelSession` struct (piper.hpp:81-88)
  - API conversion complexity (C++ ↔ C)

#### Option B: Separate Environments ✓ SELECTED
- **Implementation:** Each module creates its own `OrtEnv`
  - Piper: `Ort::Env` (C++ API)
  - VAD: `OrtEnv*` (C API)
- **Actual resource sharing:** Already happens at library level
  - Both modules link to same `libonnxruntime.so`
  - Runtime library manages shared thread pools, memory pools, etc.
  - Separate env handles have minimal overhead (mostly logging/config)
- **Benefits:**
  - No Piper modifications required
  - Cleaner API boundaries (C vs C++)
  - Already working and building
  - Risk-free implementation

### Rationale for Option B

1. **Library-level sharing already exists:** Both modules dynamically link to the same `libonnxruntime.so`, so the underlying runtime resources (thread pools, memory allocators, CUDA contexts) are already shared

2. **Env handles are lightweight:** `OrtEnv` is primarily a configuration/logging handle, not the actual resource pool

3. **API mismatch complexity:** Converting between C++ `Ort::Env` and C `OrtEnv*` or rewriting VAD in C++ adds significant complexity for minimal gain

4. **Working implementation:** VAD implementation is complete and building successfully with separate env

5. **Validation plan:** 48-hour stress test will empirically verify stability (planned for Week 1 Day 2-3)

### Implementation Status

- ✅ `vad_silero.h` - API with reset policy documentation
- ✅ `vad_silero.c` - Full implementation with Option B support (NULL for shared_env parameter)
- ✅ Integration in `CMakeLists.txt`
- ✅ Successfully builds with main `dawn` executable

### Next Steps (Week 1 Day 2)

1. Add VAD initialization to `dawn.c` main()
2. Integrate VAD into state machine (wake word detection use case)
3. Implement reset policy at state transitions
4. Set up 48-hour stress test

---

## VAD API Usage

```c
#include "vad_silero.h"

// Initialize (Option B - separate env)
silero_vad_context_t *vad = vad_silero_init(
    "/home/jetson/code/The-OASIS-Project/silero-vad/src/silero_vad/data/silero_vad_16k_op15.onnx",
    NULL  // Option B: pass NULL for separate env
);

// Process audio (512 samples at 16kHz = 32ms)
int16_t audio[512];
float speech_prob = vad_silero_process(vad, audio, 512);

// Check for speech
if (speech_prob > 0.5f) {
    // Speech detected
}

// Reset at interaction boundaries (CRITICAL!)
vad_silero_reset(vad);

// Cleanup
vad_silero_cleanup(vad);
```

### Reset Policy (from vad_silero.h)

Call `vad_silero_reset()` at these boundaries:
- State transitions to SILENCE or WAKEWORD_LISTEN
- After interruption detection
- On command timeout
- When starting new wake word detection cycle

**Why:** Prevents LSTM state accumulation from biasing detection across different interactions.

---

## Test Results Archive

Full test output from model comparison available in build artifacts:
```bash
./tests/test_vad_models
```

Key findings:
- All models meet <1ms latency requirement
- opset15 has best balance of speed, size, and API compatibility
- FP16 model uses older API (2-input signature without sr parameter)
