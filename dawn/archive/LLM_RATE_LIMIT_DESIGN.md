# LLM API Client-Side Rate Limiter

## Context

The Claude API has a 50 RPM limit. DAWN's tool iteration loop fires up to 5 API calls per user request with zero delay between them. Multiple concurrent sessions (WebUI, mic, satellites) compound this. Once we hit 429 it's too late — we need proactive throttling to stay under the limit.

## Approach

A process-wide sliding window rate limiter. A circular buffer of timestamps (one per API call) tracks how many calls were made in the last 60 seconds. Before every cloud LLM API call, check the window — if at the limit, sleep until the oldest entry expires.

**Applies to all cloud LLM providers** (Claude, OpenAI, Gemini) — the gate is before provider routing. Local LLM calls (Ollama, llama.cpp) are NOT throttled.

**Default: enabled at 40 RPM** (20% headroom under 50). Configurable via TOML and WebUI with enable/disable toggle + RPM value.

## New Files

### `include/llm/llm_rate_limit.h`

- `llm_rate_limit_init(int max_rpm)` — called from `llm_init()`, pass 0 to disable
- `llm_rate_limit_wait(void)` — returns 0 on success, 1 if interrupted
- `llm_rate_limit_set_rpm(int max_rpm)` — runtime update from WebUI settings
- `llm_rate_limit_cleanup(void)` — destroy mutex on shutdown
- Constant: `LLM_RATE_LIMIT_MAX_SLOTS 128` (power-of-two, 2 KB, ample for any tier)

### `src/llm/llm_rate_limit.c`

- `s_limiter` static global with `PTHREAD_MUTEX_INITIALIZER`
- `expire_old_entries()` — walk from tail, discard entries older than 60s (bounded by slot count, microseconds on ARM64)
- `llm_rate_limit_wait()`:
  1. Early out: if `max_rpm <= 0` return 0 immediately (disabled — zero overhead)
  2. Lock mutex, expire old entries
  3. **Re-check loop** (fixes TOCTOU race): `while (count >= max_rpm)` — compute wait, unlock, sleep in 100ms chunks checking `llm_is_interrupt_requested()`, re-lock, expire again
  4. Record timestamp, unlock, return 0
- `llm_rate_limit_set_rpm()`: validate `max_rpm <= LLM_RATE_LIMIT_MAX_SLOTS`, clamp if exceeded
- Uses `CLOCK_MONOTONIC` (VDSO on ARM64, no kernel entry)

## Insertion Points (3 total)

Each insertion point skips the gate for local LLM calls (`type == LLM_LOCAL`).

### 1. `src/llm/llm_tool_loop.c` ~line 292

Before `params->provider_fn()` call. Skip if `params->llm_type == LLM_LOCAL`. This gates every tool iteration — each iteration IS a separate API call that counts against the provider's RPM, so we must count each one.

### 2. `src/llm/llm_interface.c` — `llm_chat_completion()` ~line 799

Before the `if (type == LLM_LOCAL)` routing block. Covers non-streaming single-shot calls (context summarization, search summarization, etc.).

### 3. `src/llm/llm_interface.c` — `llm_chat_completion_with_config()` ~line 1256

Before the `if (config->type == LLM_LOCAL)` routing block. Covers non-streaming with explicit config (memory extraction, etc.).

Streaming paths go through `llm_tool_iteration_loop()` so insertion point #1 covers them. The duplicate-detection forced call at `tool_loop.c:353` also goes through `provider_fn` inside the loop, so it is also gated.

**Design intent**: All cloud API calls count — the goal is protecting the API key from exceeding provider limits. Internal calls (summarization, memory extraction) are real API calls that consume quota and must be throttled.

## Config Changes

### `include/config/dawn_config.h` — `llm_config_t`

Add two fields (LLM operational policy, not network transport):

- `bool rate_limit_enabled;`
- `int rate_limit_rpm;`

### `src/config/config_defaults.c`

```c
config->llm.rate_limit_enabled = true;   // On by default
config->llm.rate_limit_rpm = 40;          // 20% headroom under 50 RPM
```

### `src/config/config_parser.c` — `parse_llm()` (or appropriate LLM section)

Add `"rate_limit_enabled"`, `"rate_limit_rpm"` to `known_keys[]` and parse them.

### `src/config/config_env.c`

- Add env overrides `DAWN_LLM_RATE_LIMIT_ENABLED` and `DAWN_LLM_RATE_LIMIT_RPM`
- Add to `config_to_json()`: serialize both fields to `llm` JSON object
- Add to print/export functions

### `src/webui/webui_config.c` — `apply_config()`

Add JSON_TO_CONFIG parsing for both fields under the `llm` section, plus runtime update:

```c
JSON_TO_CONFIG_BOOL(section, "rate_limit_enabled", config->llm.rate_limit_enabled);
JSON_TO_CONFIG_INT(section, "rate_limit_rpm", config->llm.rate_limit_rpm);
llm_rate_limit_set_rpm(config->llm.rate_limit_enabled ? config->llm.rate_limit_rpm : 0);
```

### `www/js/ui/settings/schema.js` — llm section

Add two fields to the LLM settings section:

```js
rate_limit_enabled: {
   type: 'checkbox',
   label: 'API Rate Limiting',
   hint: 'Throttle outbound API calls to avoid provider rate limits (429 errors)',
},
rate_limit_rpm: {
   type: 'number',
   label: 'Max Requests/Min',
   min: 1,
   max: 128,
   hint: 'Maximum cloud LLM API calls per minute (default: 40)',
},
```

## Initialization

In `llm_init()` (`src/llm/llm_interface.c:277`), add:

```c
llm_rate_limit_init(g_config.llm.rate_limit_enabled ? g_config.llm.rate_limit_rpm : 0);
```

In `llm_cleanup()` (or equivalent shutdown path), add `llm_rate_limit_cleanup()`.

## CMake

Add `src/llm/llm_rate_limit.c` to the source list in `CMakeLists.txt`.

## Agent Review Summary

This design was reviewed by the architecture-reviewer and embedded-efficiency-reviewer agents. Key findings incorporated:

- **TOCTOU race fix**: `wait()` uses a re-check `while` loop after waking from sleep. Without this, concurrent threads could burst through the limiter simultaneously.
- **Config placement**: Fields live in `llm_config_t` (LLM operational policy) rather than `network_config_t` (transport parameters).
- **Buffer sizing**: 128 slots (power-of-two, 2 KB) with validation/clamping in `set_rpm()`. Sufficient for any realistic API tier.
- **Efficiency**: 3.2 KB static allocation is negligible on Jetson. O(128) expire walk takes microseconds vs seconds for HTTP. CLOCK_MONOTONIC uses VDSO on ARM64 (no kernel entry). Zero overhead when disabled.
- **Cleanup function**: `llm_rate_limit_cleanup()` for consistent resource management on shutdown.

## Verification

1. `cmake --preset debug && make -C build-debug -j8` — builds cleanly
2. Set `rate_limit_rpm = 5` in `[llm]` section of dawn.toml, make rapid requests, verify LOG_WARNING messages showing throttle waits
3. Confirm local LLM calls are NOT throttled
4. Confirm user can interrupt during a rate-limit wait
5. Toggle enable/disable in WebUI and verify runtime update works
