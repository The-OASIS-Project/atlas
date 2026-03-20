# Model-Specific Configuration System

## Overview

The testing system now uses **model-specific optimized settings** instead of one-size-fits-all parameters. Each model has been configured based on:

1. **HuggingFace model card recommendations** from the original model creators
2. **llama.cpp best practices** for optimal performance
3. **Empirical testing results** from our debugging sessions

## Architecture

### Configuration Files

```
model_configs.conf       # Model-specific parameter definitions
get_model_config.sh      # Parser that loads configurations
test_single_model.sh     # Uses config system for single model tests
benchmark_all_models.sh  # Uses config system for batch testing
```

### Configuration Format

Each model has a configuration line in `model_configs.conf`:

```
MODEL_FILENAME.gguf|gpu_layers|context|batch|ubatch|threads|temp|top_p|top_k|repeat_penalty|extra_flags
```

**Example:**
```
Llama-3.2-3B-Instruct-Q4_K_M.gguf|99|1024|512|512|4|0.5|0.85|20|1.3|--flash-attn
```

## Current Model Configurations

### Qwen Models (Standard Settings)

**Qwen3-4B-Q4_K_M** and **Qwen3-4B-Q6_K_L:**
- Temperature: 0.7 (standard for Qwen models)
- Top-P: 0.9
- Top-K: 40
- Context: 1024, Batch: 512
- **Works well with default settings**

**Qwen2.5-7B-Q4_K_M:**
- Temperature: 0.7
- Top-P: 0.9
- Top-K: 40
- **Context: 1024, Batch: 128** ← Reduced to fit all layers on GPU
- Unified memory requires smaller compute buffer for 7B models

### Llama Models (Conservative Sampling)

**Llama-3.2-3B-Instruct-Q4_K_M:**
- **Temperature: 0.5** ← Lower to prevent gibberish
- **Top-P: 0.85** ← Tighter nucleus sampling
- **Top-K: 20** ← Restricted token pool
- **Repeat Penalty: 1.3** ← Stronger to avoid repetition
- Context: 1024, Batch: 512

**Rationale:** Testing showed Llama produces gibberish with standard settings (temp 0.7). Conservative sampling produces coherent output.

### Phi Models (Conservative Sampling)

**Phi-3-mini-4k-instruct-q4:**
- **Temperature: 0.5** ← Same issue as Llama
- **Top-P: 0.85**
- **Top-K: 20**
- **Repeat Penalty: 1.3**
- Context: 1024, Batch: 512

**Rationale:** Similar to Llama, needs conservative sampling to avoid gibberish.

## Usage

### Testing Single Model

```bash
./test_single_model.sh <model_filename.gguf>
```

The script will:
1. Load model-specific configuration from `model_configs.conf`
2. Display the configuration being used
3. Start llama-server with optimized parameters
4. Run speed and quality tests
5. Save configuration used to `config_used.txt` in results directory

**Example:**
```bash
./test_single_model.sh Llama-3.2-3B-Instruct-Q4_K_M.gguf
```

### Batch Testing All Models

```bash
./benchmark_all_models.sh
```

Each model will be tested with its specific optimized configuration.

### Manual Configuration Lookup

```bash
source get_model_config.sh <model_filename.gguf>

# Now variables are available:
echo "GPU Layers: $GPU_LAYERS"
echo "Context: $CONTEXT"
echo "Temperature: $TEMP"
# etc.
```

## Adding New Models

### Step 1: Research Optimal Settings

1. **Check HuggingFace model card:**
   ```
   https://huggingface.co/<organization>/<model-name>
   ```
   Look for recommended temperature, top-p, top-k settings

2. **Check llama.cpp issues/discussions:**
   - Search for model name in llama.cpp GitHub
   - Look for community-reported optimal settings

3. **Start with similar model:**
   - If new Qwen model → use Qwen3-4B settings as baseline
   - If new Llama model → use Llama-3.2-3B settings
   - If new Phi model → use Phi-3-mini settings

### Step 2: Add Configuration Line

Edit `model_configs.conf` and add:

```bash
# New-Model-Name
# - Source: HuggingFace link or documentation
# - Recommended: temp X, top-p Y (from model card)
# - Notes: Any special considerations
New-Model-Name-Q4_K_M.gguf|99|1024|512|512|4|0.7|0.9|40|1.1|--flash-attn
```

### Step 3: Test and Refine

```bash
./test_single_model.sh New-Model-Name-Q4_K_M.gguf
```

If results are poor:
- **Gibberish output** → Try conservative sampling (temp 0.5, top-k 20)
- **Too slow** → Reduce context or batch size
- **GPU OOM** → Reduce batch size first, then context
- **Poor quality** → Adjust temperature (higher = more creative, lower = more focused)

### Step 4: Document

Add comments in `model_configs.conf` explaining:
- Source of recommended settings
- Any deviations from defaults and why
- Performance characteristics

## Key Parameters Explained

### GPU Layers (`--gpu-layers`)
- **99** = Offload all possible layers to GPU
- On Jetson (unified memory), this is always optimal
- Only limited by compute buffer size, not memory capacity

### Context Size (`-c`)
- **1024** = Optimal for DAWN (rarely exceeds this)
- **512** = Minimum for complex prompts with FRIDAY persona
- **256** = Too small, will truncate system prompt
- Larger = more memory for compute buffer

### Batch Size (`-b`, `-ub`)
- **512** = Optimal for 3-4B models on Jetson
- **128** = Required for 7B models to fit compute buffer
- **64** = Fallback if still OOM
- Larger = better GPU utilization, faster inference

### Temperature
- **0.3-0.6** = Focused, deterministic (use for Llama/Phi)
- **0.7** = Balanced (use for Qwen)
- **0.8-1.0** = Creative, random (not recommended for DAWN)

### Top-P (Nucleus Sampling)
- **0.85** = Tight control (Llama/Phi)
- **0.9** = Standard (Qwen)
- **0.95** = Wider variety (not needed for DAWN)

### Top-K (Token Pool)
- **20** = Restricted (Llama/Phi to prevent gibberish)
- **40** = Standard (Qwen)
- **80+** = Too wide (causes randomness)

### Repeat Penalty
- **1.1** = Mild (standard for Qwen)
- **1.3** = Strong (Llama/Phi to avoid repetitive gibberish)
- **1.5+** = Too harsh (causes unnatural output)

## Configuration Source References

### Qwen Models
- HuggingFace: `Qwen/Qwen3-4B-Instruct`, `Qwen/Qwen2.5-7B-Instruct`
- Recommended by Qwen team: temp 0.7, top-p 0.8
- Our testing: top-p 0.9 works slightly better for DAWN

### Llama Models
- HuggingFace: `meta-llama/Llama-3.2-3B-Instruct`
- Recommended by Meta: temp 0.6, top-p 0.9
- Our testing: **temp 0.5 required** to avoid gibberish (critical finding!)

### Phi Models
- HuggingFace: `microsoft/Phi-3-mini-4k-instruct`
- Recommended by Microsoft: temp 0.7
- Our testing: **temp 0.5 required** to avoid gibberish (same as Llama)

## Troubleshooting

### Model produces gibberish
→ Reduce temperature to 0.5 and top-k to 20 (conservative sampling)

### Model is too slow
→ Check if all layers are on GPU (should be 28/28 or 36/36)
→ If not, reduce batch size to fit compute buffer

### GPU out of memory
→ Reduce batch size first: 512→256→128→64
→ Then reduce context if needed: 1024→512
→ Never reduce GPU layers on Jetson (unified memory)

### Quality score low but output is coherent
→ May need to adjust FRIDAY system prompt
→ Check `config_used.txt` to verify settings were applied
→ Compare with working model's configuration

## Best Practices

1. **Always check config_used.txt** in results directory to verify settings
2. **Start with similar model** when adding new configurations
3. **Test before committing** configuration changes
4. **Document sources** for recommended settings
5. **Save working configurations** immediately after finding optimal settings

## Future Enhancements

Potential improvements to the configuration system:

1. **Auto-detection** of model architecture to suggest initial config
2. **Performance profiling** to recommend optimal batch/context sizes
3. **Quality-guided tuning** that adjusts parameters based on test scores
4. **Hardware-specific configs** for different Jetson models (Orin, Xavier, etc.)

---

**Last Updated:** 2025-11-21
**System Version:** DAWN Local LLM Testing v2.0
