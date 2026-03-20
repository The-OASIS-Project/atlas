> **STATUS: COMPLETE** - Archived 2025-12-25

# Phase 1: DAWN Config System Core Infrastructure

## Overview

Create the core config infrastructure WITHOUT changing existing behavior. The config system will be fully functional but dormant until Phase 4 integrates it with existing modules.

## Files to Create

### New Directories
- `include/config/`
- `src/config/`

### New Files

| File | Purpose |
|------|---------|
| `include/tools/toml.h` | Vendored tomlc99 header |
| `src/tools/toml.c` | Vendored tomlc99 source |
| `include/config/dawn_config.h` | Main config struct definitions |
| `include/config/config_parser.h` | TOML parser interface |
| `include/config/config_validate.h` | Validation interface |
| `src/config/config_defaults.c` | Default value initialization |
| `src/config/config_parser.c` | TOML parsing implementation |
| `src/config/config_validate.c` | Validation implementation |

### Modified Files

| File | Change |
|------|--------|
| `CMakeLists.txt` | Add new source files to DAWN_SOURCES (~line 313) |

## Implementation Order

### Step 1: Vendor tomlc99 Library
- Download from https://github.com/cktan/tomlc99
- Place `toml.h` in `include/tools/`
- Place `toml.c` in `src/tools/`
- Adjust include: `#include "tools/toml.h"`
- Keep MIT license header intact

### Step 2: Create dawn_config.h
Main config struct with all subsystem configs:
- Buffer size constants (CONFIG_PATH_MAX=256, CONFIG_NAME_MAX=64, etc.)
- Nested structs for each [section]: general, persona, localization, audio, vad, asr, tts, commands, llm, search, url_fetcher, mqtt, network, tui, debug, paths
- Separate `secrets_config_t` for API keys
- Global declarations: `extern dawn_config_t g_config; extern secrets_config_t g_secrets;`
- Function prototypes: `config_set_defaults()`, `config_get()`, `config_cleanup()`

### Step 3: Create config_defaults.c
Initialize all fields with current compile-time defaults from dawn.h:
- `ai_name = "friday"`
- `vad.speech_threshold = 0.5f`
- `vad.speech_threshold_tts = 0.92f`
- `llm.cloud.model = "gpt-4o"`
- `tts.length_scale = 0.85f`
- (Match all values from dawn.h and dawn.c)

### Step 4: Create config_parser.h/c
TOML parsing using tomlc99 API:
- `config_parse_file()` - Main entry point
- `config_parse_secrets()` - Secrets file parsing
- `config_file_exists()` - Check file accessibility
- Helper functions per section (parse_general, parse_audio, etc.)
- Safe string copying with null-termination
- Error messages with context

### Step 5: Create config_validate.h/c
Validation with structured error reporting:
- `config_error_t` struct: field name + message
- `config_validate()` - Returns error count
- Range validation: thresholds 0.0-1.0, ports 1-65535
- Enum validation: processing_mode, llm.type
- Dependency validation: cloud LLM requires API key

### Step 6: Update CMakeLists.txt
Add after DAWN_SOURCES definition (~line 313):
```cmake
# Config subsystem
list(APPEND DAWN_SOURCES
    src/config/config_defaults.c
    src/config/config_parser.c
    src/config/config_validate.c
    src/tools/toml.c
)
```

## Key Design Decisions

1. **Vendored tomlc99**: Follow tinyexpr pattern (single-file library in src/tools/)
2. **Static allocation**: Fixed-size char arrays in structs (embedded focus)
3. **Thread-safe**: Read-only after init (no mutex needed for reads)
4. **Backwards compatible**: Works without config file (defaults only)
5. **No behavior changes**: Phase 1 is infrastructure only

## Validation Rules

| Field | Validation |
|-------|------------|
| `vad.speech_threshold` | 0.0 - 1.0 |
| `vad.speech_threshold_tts` | 0.0 - 1.0 |
| `tts.length_scale` | 0.5 - 2.0 |
| `mqtt.port` | 1 - 65535 |
| `network.port` | 1 - 65535 |
| `commands.processing_mode` | "direct_only", "llm_only", "direct_first" |
| `llm.type` | "cloud", "local" |

## Verification Checklist

- [ ] Project compiles without errors
- [ ] No changes to existing runtime behavior
- [ ] `config_set_defaults(&g_config)` can be called
- [ ] `config_validate()` returns 0 for default config
- [ ] Code passes `./format_code.sh --check`
- [ ] All new files have GPL header blocks

## Reference Files

- `include/tools/tinyexpr.h` - Pattern for vendored headers
- `src/tools/tinyexpr.c` - Pattern for vendored sources
- `docs/CONFIG_FILE_DESIGN.md` - Full spec with struct definitions
- `src/dawn.c` lines 92-112 - Current VAD defaults
- `src/dawn.c` lines 1396-1617 - CLI parsing to match
