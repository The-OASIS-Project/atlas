# llama-server Optimization for DAWN

## Current Configuration Analysis

**Your current command:**
```bash
/usr/local/bin/llama-server \
  -m /var/lib/llama-cpp/models/Qwen_Qwen3-4B-Q6_K_L.gguf \
  --gpu-layers 99 \
  -c 2048 \
  -b 128 \
  -ub 64 \
  -t 6 \
  -ctk q8_0 \
  -ctv q8_0 \
  --flash-attn \
  --temp 0.7 \
  --top-p 0.8 \
  --top-k 20 \
  --min-p 0 \
  --jinja \
  --chat-template-file /var/lib/llama-cpp/templates/qwen3_nonthinking.jinja \
  --host 127.0.0.1 \
  --port 8080
```

**Performance:** 11.88 tokens/sec (too slow)

---

## Optimized Configuration (For Real-Time Conversational AI)

```bash
/usr/local/bin/llama-server \
  -m /var/lib/llama-cpp/models/<YOUR_MODEL>.gguf \
  --gpu-layers 99 \
  -c 1024 \
  -b 512 \
  -ub 512 \
  -t 4 \
  --flash-attn \
  --temp 0.7 \
  --top-p 0.9 \
  --top-k 40 \
  --repeat-penalty 1.1 \
  --mirostat 0 \
  --jinja \
  --chat-template-file /var/lib/llama-cpp/templates/qwen3_nonthinking.jinja \
  --host 127.0.0.1 \
  --port 8080 \
  --parallel 1 \
  --cont-batching \
  --metrics
```

---

## Parameter Changes Explained

### Performance Optimizations

| Parameter | Old | New | Why |
|-----------|-----|-----|-----|
| **-c (context)** | 2048 | **1024** | DAWN conversations rarely exceed 1024 tokens. Smaller context = faster + less memory |
| **-b (batch)** | 128 | **512** | Larger batch size improves GPU utilization for single-user scenario |
| **-ub (ubatch)** | 64 | **512** | Match batch size for continuous batching efficiency |
| **-t (threads)** | 6 | **4** | With all layers on GPU, CPU threads matter less. Reduce for efficiency |
| **-ctk/-ctv** | q8_0 | **removed** | Cache quantization adds latency. With 1024 context, not needed |
| **--parallel** | (none) | **1** | Optimize for single user (DAWN = one conversation at a time) |
| **--cont-batching** | (none) | **enabled** | Improves streaming response performance |
| **--metrics** | (none) | **enabled** | Enable performance monitoring at /metrics endpoint |

### Sampling Parameters (Quality)

| Parameter | Old | New | Why |
|-----------|-----|-----|-----|
| **--temp** | 0.7 | 0.7 | Keep same - good for conversational AI |
| **--top-p** | 0.8 | **0.9** | Slightly less restrictive for more natural responses |
| **--top-k** | 20 | **40** | Allow more token candidates (less repetitive) |
| **--repeat-penalty** | (none) | **1.1** | Prevent model from repeating phrases |
| **--mirostat** | (none) | **0** | Disable mirostat (use top-p/top-k instead for consistency) |
| **--min-p** | 0 | **removed** | Not needed with top-p/top-k |

### Unchanged (Already Optimal)

| Parameter | Value | Reason |
|-----------|-------|--------|
| **--gpu-layers 99** | 99 | Correct - use all GPU layers |
| **--flash-attn** | enabled | Correct - faster attention mechanism |
| **--jinja** | enabled | Correct - template support |
| **--host/port** | 127.0.0.1:8080 | Correct - local only |

---

## Alternative Configurations

### For Maximum Speed (Sacrifice Some Quality)

If you need even faster responses:

```bash
/usr/local/bin/llama-server \
  -m /var/lib/llama-cpp/models/<MODEL>.gguf \
  --gpu-layers 99 \
  -c 768 \
  -b 1024 \
  -ub 1024 \
  -t 4 \
  --flash-attn \
  --temp 0.6 \
  --top-p 0.85 \
  --top-k 30 \
  --repeat-penalty 1.15 \
  --host 127.0.0.1 \
  --port 8080 \
  --parallel 1 \
  --cont-batching
```

Changes:
- Context reduced to 768 (faster, but less conversation history)
- Batch increased to 1024 (maximize GPU throughput)
- Temperature 0.6 (more deterministic, faster sampling)
- Higher repeat penalty (prevent loops)

### For Maximum Quality (Slower)

If you prioritize quality over speed:

```bash
/usr/local/bin/llama-server \
  -m /var/lib/llama-cpp/models/<MODEL>.gguf \
  --gpu-layers 99 \
  -c 2048 \
  -b 256 \
  -ub 256 \
  -t 6 \
  --flash-attn \
  --temp 0.8 \
  --top-p 0.95 \
  --top-k 50 \
  --repeat-penalty 1.05 \
  --host 127.0.0.1 \
  --port 8080 \
  --parallel 1
```

Changes:
- Larger context (more history)
- Moderate batch sizes (balance)
- Higher temperature and top-p (more creative)
- Lower repeat penalty (more natural)

---

## Expected Performance Improvements

### With Optimized Parameters + Q4_K_M Model

| Scenario | Current (Q6_K_L) | Optimized (Q4_K_M) | Improvement |
|----------|------------------|---------------------|-------------|
| **Tokens/sec** | 11.88 | 25-35 | 2-3x |
| **Simple response (20 tokens)** | 1.7s | 0.6-0.8s | 2x |
| **Typical response (50 tokens)** | 4.3s | 1.5-2.0s | 2x |
| **Long response (100 tokens)** | 8.4s | 3.0-4.0s | 2x |

### Impact on DAWN End-to-End Latency

```
BEFORE (Cloud LLM):
Silence (1.2s) + ASR (0.5s) + Cloud LLM (3-5s) + TTS (0.2s) = 5.0-6.9s

CURRENT (Local Q6_K_L):
Silence (1.2s) + ASR (0.5s) + Local LLM (4.3s) + TTS (0.2s) = 6.2s
❌ NO IMPROVEMENT

AFTER (Optimized Q4_K_M):
Silence (1.2s) + ASR (0.5s) + Local LLM (1.8s) + TTS (0.2s) = 3.7s
✅ 40% FASTER than cloud, 2.3s improvement
```

---

## Monitoring Performance

### Check Real-Time Metrics

With `--metrics` enabled:
```bash
curl http://127.0.0.1:8080/metrics
```

Gives Prometheus-style metrics including:
- Request latency
- Tokens per second
- Queue depth
- GPU utilization

### Quick Performance Test

Run the test script:
```bash
cd /home/jetson/code/The-OASIS-Project/dawn
./test_llama_performance.sh
```

Target results:
- ✅ Tokens/sec: 25+ (acceptable), 40+ (excellent)
- ✅ TTFT: <200ms
- ✅ 50-token response: <2 seconds

---

## Troubleshooting

### Still Getting <20 tokens/sec?

1. **Check GPU utilization:**
   ```bash
   nvidia-smi dmon -s u
   # Should show high GPU util during inference
   ```

2. **Verify all layers on GPU:**
   ```bash
   curl -s http://127.0.0.1:8080/props | grep gpu
   # Should show n_gpu_layers = 99
   ```

3. **Check model quantization:**
   ```bash
   ls -lh /var/lib/llama-cpp/models/
   # Q6_K_L models are 2x larger than Q4_K_M
   # Larger = slower
   ```

4. **Try smaller model:**
   - Llama 3.2 3B is faster than 7B models
   - Phi-3 Mini is very efficient

### High Memory Usage?

1. **Reduce context:**
   ```bash
   -c 768  # or even 512 for simple commands
   ```

2. **Check for memory leaks:**
   ```bash
   watch -n 1 'ps aux | grep llama-server'
   ```

3. **Unload Whisper during LLM inference** (if both don't fit)

### Response Quality Issues?

1. **Adjust temperature:** 0.7-0.8 for conversation
2. **Increase top-k:** 40-50 for more diverse responses
3. **Fine-tune system prompt** for smaller models
4. **Try larger model:** Qwen2.5-7B instead of 3B models

---

## Recommended Next Steps

1. **Download Q4_K_M model** (see DOWNLOAD_FASTER_MODELS.md)
2. **Apply optimized parameters** (use command above)
3. **Run benchmark** (`./test_llama_performance.sh`)
4. **Verify 25+ tokens/sec** target achieved
5. **Integrate into DAWN** (replace cloud LLM calls)

---

## For DAWN Integration

Once performance is acceptable (25+ tokens/sec):

1. Update `openai.c` to support local endpoint:
   ```c
   #define LOCAL_LLM_ENDPOINT "http://127.0.0.1:8080/v1/chat/completions"
   ```

2. Add configuration flag:
   ```c
   #define USE_LOCAL_LLM 1  // Set to 0 for cloud fallback
   ```

3. Dual-path system:
   - Simple queries → local LLM
   - Complex queries / vision → cloud LLM

Expected result: **40% faster responses** with zero API costs.
