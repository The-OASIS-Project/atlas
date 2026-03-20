# DAWN Configuration File System Design

## Overview

This document describes the design for a TOML-based configuration file system for DAWN. The goal is to provide a user-friendly way to configure the voice assistant without modifying source code or using complex command-line arguments.

## Design Goals

1. **User-friendly**: TOML format with comments, easy to edit by hand
2. **Backwards compatible**: Works without config file (sensible defaults)
3. **Layered configuration**: Defaults → Config file → Environment → CLI args
4. **Secure secrets handling**: Separate secrets file, never committed to git
5. **Future-proof**: Extensible structure for new features

## Configuration Priority (Lowest to Highest)

1. **Hardcoded defaults** (compile-time)
2. **Config file** (`dawn.toml` or `--config=PATH`)
3. **Environment variables** (for secrets and all config via `DAWN_` prefix)
4. **Command-line arguments** (highest priority, overrides everything)

## File Locations

Config files are searched in order:
1. `--config=PATH` (explicit path)
2. `./dawn.toml` (current directory)
3. `~/.config/dawn/config.toml` (user config)
4. `/etc/dawn/config.toml` (system-wide)

Secrets file:
1. `~/.config/dawn/secrets.toml` (mode 600, user-only read)

## Library Selection

**tomlc99** - Single-file C99 TOML parser
- ~1500 lines, easy to integrate
- Well-maintained, handles TOML 1.0
- No dependencies beyond standard C library
- MIT licensed

Integration: Add as git submodule or copy source files directly.

## Configuration Structure

### Main Config: `dawn.toml`

```toml
# DAWN Voice Assistant Configuration
# Command-line arguments override these settings
# Environment variables: DAWN_<SECTION>_<KEY> (e.g., DAWN_AUDIO_BACKEND=alsa)
# Save as: ~/.config/dawn/config.toml or ./dawn.toml

# =============================================================================
# General Settings
# =============================================================================
[general]
ai_name = "friday"                    # Wake word (lowercase, triggers listening)
log_file = ""                         # Empty = stdout, or path like "/var/log/dawn.log"

# =============================================================================
# Persona (AI personality and behavior)
# =============================================================================
[persona]
# Description is the system prompt defining AI personality
# If empty, uses compile-time default from dawn.h
description = ""
# Future: separate persona files, multi-persona support

# =============================================================================
# Localization
# =============================================================================
[localization]
location = ""                         # Default location for weather/context (e.g., "Austin, TX")
timezone = ""                         # Empty = system default, or "America/Chicago"
units = "imperial"                    # "imperial" or "metric" (affects weather, etc.)

# =============================================================================
# Audio Configuration
# =============================================================================
[audio]
backend = "auto"                      # "auto", "pulseaudio", "alsa"
capture_device = "default"            # Device name (format depends on backend)
playback_device = "default"           # Device name (format depends on backend)
output_rate = 44100                   # Playback sample rate: 44100 (CD) or 48000 (pro)
output_channels = 2                   # Playback channels: 2 (stereo, required for dmix)
# Note: Capture sample rate (16kHz mono) is fixed for ASR compatibility

[audio.bargein]
enabled = true                        # Allow interrupting TTS with speech
cooldown_ms = 1500                    # Keep high VAD threshold after TTS stops
startup_cooldown_ms = 300             # Block barge-in when TTS starts (AEC settle)

# =============================================================================
# Voice Activity Detection (VAD)
# =============================================================================
[vad]
speech_threshold = 0.5                # Probability to detect speech start (0.0-1.0)
speech_threshold_tts = 0.92           # Higher threshold during TTS (reduce false triggers)
silence_threshold = 0.3               # Probability for end-of-utterance detection
end_of_speech_duration = 1.2          # Seconds of silence to end recording
max_recording_duration = 30.0         # Maximum recording length (seconds)
preroll_ms = 500                      # Audio buffer before VAD trigger

[vad.chunking]
enabled = true                        # Enable natural pause detection (Whisper only)
pause_duration = 0.3                  # Silence duration for chunk boundary (seconds)
min_chunk_duration = 1.0              # Minimum speech before creating chunk
max_chunk_duration = 10.0             # Force chunk boundary after this duration

# =============================================================================
# Speech Recognition (ASR)
# =============================================================================
[asr]
# Engine selection is compile-time: default is Whisper, use -DENABLE_VOSK=ON for Vosk
model = "base"                        # Whisper: "tiny", "base", "small", "medium"
models_path = "models/whisper.cpp"    # Path to Whisper model files
# Note: Vosk has no additional runtime settings beyond model path

# =============================================================================
# Text-to-Speech (TTS)
# =============================================================================
[tts]
models_path = "models"                # Path to TTS model files
voice_model = "en_GB-alba-medium"     # Piper voice model name
length_scale = 0.85                   # Speaking rate: <1.0 = faster, >1.0 = slower
                                      # (0.85 = ~1.18x speed, current default)

# =============================================================================
# Commands (how user input is processed)
# =============================================================================
[commands]
processing_mode = "direct_first"      # "direct_only", "llm_only", "direct_first"
                                      # direct_only: Pattern match only, no LLM
                                      # llm_only: Always use LLM
                                      # direct_first: Try patterns, fall back to LLM

# =============================================================================
# LLM Configuration
# =============================================================================
[llm]
type = "cloud"                        # "cloud" or "local"
max_tokens = 4096                     # Max response tokens

[llm.cloud]
provider = "openai"                   # "openai" or "claude"
openai_model = "gpt-4o"               # Model for OpenAI API
claude_model = "claude-sonnet-4-20250514"  # Model for Claude API
endpoint = ""                         # Empty = default, or custom endpoint for proxies
vision_enabled = true                 # Cloud models typically support vision
# API keys: use secrets.toml or environment variables (OPENAI_API_KEY, ANTHROPIC_API_KEY)

[llm.local]
endpoint = "http://127.0.0.1:8080"    # Local llama-server
model = ""                            # Optional model name (server decides if empty)
vision_enabled = false                # Enable for vision models (LLaVA, Qwen-VL, etc.)

# =============================================================================
# Search & Tools
# =============================================================================
[search]
engine = "searxng"
endpoint = "http://127.0.0.1:8384"    # SearXNG instance URL

[search.summarizer]
backend = "disabled"                  # "disabled", "local", "default"
threshold_bytes = 3072                # Summarize results larger than this
target_words = 600                    # Target summary length

# =============================================================================
# URL Fetcher (web content extraction for LLM)
# =============================================================================
[url_fetcher]
# Whitelist allows access to private/internal URLs that would otherwise be
# blocked by SSRF protection. By default, localhost, private IP ranges
# (10.x.x.x, 172.16-31.x.x, 192.168.x.x), and cloud metadata endpoints are blocked.
#
# Supported whitelist entry formats:
#   - Specific URL prefix: "http://192.168.1.100:8080/api"
#   - Hostname: "wiki.local"
#   - Single IP: "10.0.0.5"
#   - CIDR network: "192.168.1.0/24"
#
# Examples:
#   whitelist = ["192.168.1.0/24", "wiki.local", "http://localhost:8888"]
whitelist = []

[url_fetcher.flaresolverr]
# FlareSolverr is a proxy service that uses headless Chromium to bypass
# Cloudflare and other anti-bot protections. When enabled, URL fetcher
# automatically falls back to FlareSolverr on 403 Forbidden errors.
#
# Docker: docker run -d --name flaresolverr -p 8191:8191 ghcr.io/flaresolverr/flaresolverr
#
# SECURITY WARNING: FlareSolverr runs full browser JavaScript and is
# resource-intensive (~200MB RAM). Only enable if needed for specific sites.
enabled = false                       # Auto-fallback on 403 errors
endpoint = "http://127.0.0.1:8191/v1" # FlareSolverr API endpoint
timeout_sec = 60                      # Request timeout (browser rendering is slow)
max_response_bytes = 4194304          # 4MB max response (rendered pages can be large)

# =============================================================================
# MQTT Integration
# =============================================================================
[mqtt]
enabled = true
broker = "127.0.0.1"
port = 1883
# Credentials: see [mqtt] section in secrets.toml

# =============================================================================
# Network Audio Server (ESP32 satellites)
# =============================================================================
[network]
enabled = false                       # Enable network audio server
host = "0.0.0.0"                      # Bind address
port = 5000                           # Listen port
workers = 4                           # Concurrent processing threads
                                      # (each loads ASR model, sessions = 2x workers)
socket_timeout_sec = 30               # Client timeout
session_timeout_sec = 300             # Idle session expiry (conversation history)
llm_timeout_ms = 30000                # Per-request LLM timeout

# =============================================================================
# Terminal UI
# =============================================================================
[tui]
enabled = false                       # Enable TUI dashboard (if compiled with ENABLE_TUI)
# theme = "default"                   # Future: color themes

# =============================================================================
# Debug & Recording
# =============================================================================
[debug]
mic_record = false                    # Record raw microphone input
asr_record = false                    # Record ASR input audio
aec_record = false                    # Record AEC processed audio
record_path = "/tmp"                  # Directory for debug recordings

# =============================================================================
# Paths
# =============================================================================
[paths]
music_dir = "~/Music"                 # Music library location
commands_config = "commands_config_nuevo.json"  # Device/command mappings
```

### Secrets File: `secrets.toml`

```toml
# DAWN Secrets - DO NOT COMMIT TO VERSION CONTROL
# Save as: ~/.config/dawn/secrets.toml (mode 600)

[api_keys]
openai = "sk-..."
claude = "sk-ant-..."

[mqtt]
username = ""
password = ""
```

## Audio Backend Architecture

### Design

Support multiple audio backends at runtime without recompilation. Separate interfaces
for capture and playback to match existing code architecture where these are managed
independently (capture thread vs TTS playback thread).

**Error Handling:** All functions returning `int` use SUCCESS (0) and FAILURE (1) per project conventions.

**Capture Backend Interface:**
```c
typedef struct {
    const char *name;
    int (*open)(const char *device, unsigned int *actual_rate);
    int (*read)(int16_t *buffer, size_t frames);
    size_t (*get_latency_frames)(void);  // For AEC delay calibration
    int (*stop)(void);                   // Stop capture, discard buffered data
    void (*close)(void);
} audio_capture_backend_t;
```

**Playback Backend Interface:**
```c
typedef struct {
    const char *name;
    int (*open)(const char *device, unsigned int rate, unsigned int channels);
    int (*write)(const int16_t *buffer, size_t frames);
    int (*drain)(void);                   // Wait for playback to complete
    int (*flush)(void);                   // Discard pending samples (for barge-in)
    size_t (*get_latency_frames)(void);   // For AEC delay calibration
    void (*close)(void);
} audio_playback_backend_t;
```

### Backend Selection Logic

```
backend = "auto" (default):
  1. Check if PulseAudio daemon is running (pa_context_connect)
  2. If available, use PulseAudio
  3. Otherwise, fall back to ALSA

backend = "pulseaudio":
  - Use PulseAudio directly, fail if unavailable

backend = "alsa":
  - Use ALSA directly
```

### Build Integration

- **Dynamic linking** for both `libasound` and `libpulse` (NOT static)
  - Static linking breaks ALSA plugin devices (`plughw:`, `dmix`)
  - PulseAudio requires daemon connection regardless of linking
  - Both libraries are standard on Linux systems with audio support
- Runtime selection via config
- Optional: use `dlopen()` for truly optional backends (future enhancement)

### Device Naming

Different backends use different device naming:
- **ALSA**: `hw:2,0`, `default`, `plughw:1,0`
- **PulseAudio**: `alsa_input.usb-...`, or just `default`

The `capture_device` and `playback_device` config values are passed directly to the selected backend.

## Implementation Plan

### Phase 1: Core Infrastructure
1. Add tomlc99 as dependency (submodule or vendored)
2. Create `src/config/config_parser.c` - TOML parsing
3. Create `src/config/config_validate.c` - Validation logic
4. Create `include/config/dawn_config.h` - Config struct definition
5. Create `src/config/config_defaults.c` - Default value initialization

### Phase 2: Config Loading
1. Implement config file search logic
2. Implement secrets file loading (separate, secure)
3. Add `--config` and `--dump-config` CLI arguments
4. Integrate with existing CLI argument parsing (CLI overrides config)
5. Add `DAWN_` environment variable prefix support

### Phase 3: Audio Backend Abstraction
1. Create `include/audio/audio_backend.h` - Backend interfaces (capture + playback)
2. Create `src/audio/audio_alsa_capture.c` - ALSA capture implementation
3. Create `src/audio/audio_alsa_playback.c` - ALSA playback implementation
4. Create `src/audio/audio_pulse_capture.c` - PulseAudio capture implementation
5. Create `src/audio/audio_pulse_playback.c` - PulseAudio playback implementation
6. Create `src/audio/audio_backend.c` - Auto-detection and selection
7. Update CMakeLists.txt to link both libraries (dynamic)

### Phase 4: Module Integration
1. Update `dawn.c` to use config struct instead of #defines
2. Update audio capture thread to use backend abstraction
3. Update TTS module to read `length_scale` from config and use backend abstraction
4. Update VAD module to read thresholds from config
5. Update LLM module to read provider/model from config
6. Update network module to read worker count from config

### Phase 5: Documentation & Examples
1. Generate example `dawn.toml` with all options commented
2. Generate example `secrets.toml.example`
3. Update README with config file documentation
4. Add config file to `.gitignore` patterns

## Config Struct Design

Use nested structs for module isolation and clearer namespacing:

```c
// Buffer size constants (use consistently across codebase)
#define CONFIG_PATH_MAX 256
#define CONFIG_NAME_MAX 64
#define CONFIG_DEVICE_MAX 128

// Nested config structs for each subsystem
typedef struct {
    char ai_name[CONFIG_NAME_MAX];
    char log_file[CONFIG_PATH_MAX];
} general_config_t;

// Secrets struct (loaded separately from secrets.toml)
typedef struct {
    char openai_api_key[128];
    char claude_api_key[128];
    char mqtt_username[64];
    char mqtt_password[64];
} secrets_config_t;

// Global secrets instance (loaded separately, never logged)
extern secrets_config_t g_secrets;

typedef struct {
    char description[2048];  // System prompt (can be large)
} persona_config_t;

typedef struct {
    char location[128];
    char timezone[CONFIG_NAME_MAX];
    char units[16];
} localization_config_t;

typedef struct {
    bool enabled;
    int cooldown_ms;
    int startup_cooldown_ms;
} bargein_config_t;

typedef struct {
    char backend[16];
    char capture_device[CONFIG_DEVICE_MAX];
    char playback_device[CONFIG_DEVICE_MAX];
    bargein_config_t bargein;
} audio_config_t;

typedef struct {
    bool enabled;
    float pause_duration;
    float min_duration;
    float max_duration;
} vad_chunking_config_t;

typedef struct {
    float speech_threshold;
    float speech_threshold_tts;
    float silence_threshold;
    float end_of_speech_duration;
    float max_recording_duration;
    int preroll_ms;
    vad_chunking_config_t chunking;
} vad_config_t;

typedef struct {
    char model[CONFIG_NAME_MAX];
    char models_path[CONFIG_PATH_MAX];
} asr_config_t;

typedef struct {
    char voice_model[128];
    float length_scale;
} tts_config_t;

typedef struct {
    char processing_mode[16];  // "direct_only", "llm_only", "direct_first"
} commands_config_t;

typedef struct {
    char provider[16];
    char model[CONFIG_NAME_MAX];
    char endpoint[CONFIG_PATH_MAX];
    bool vision_enabled;              // Cloud models typically support vision
} llm_cloud_config_t;

typedef struct {
    char endpoint[CONFIG_PATH_MAX];
    char model[CONFIG_NAME_MAX];
    bool vision_enabled;              // Enable for LLaVA, Qwen-VL, etc.
} llm_local_config_t;

typedef struct {
    char type[16];
    int max_tokens;
    llm_cloud_config_t cloud;
    llm_local_config_t local;
} llm_config_t;

typedef struct {
    char backend[16];
    size_t threshold_bytes;
    size_t target_words;
} summarizer_config_t;

typedef struct {
    char engine[32];
    char endpoint[CONFIG_PATH_MAX];
    summarizer_config_t summarizer;
} search_config_t;

#define URL_FETCHER_MAX_WHITELIST 16
typedef struct {
    bool enabled;
    char endpoint[CONFIG_PATH_MAX];
    int timeout_sec;
    size_t max_response_bytes;
} flaresolverr_config_t;

typedef struct {
    char *whitelist[URL_FETCHER_MAX_WHITELIST];  // Array of whitelist entries
    int whitelist_count;
    flaresolverr_config_t flaresolverr;
} url_fetcher_config_t;

typedef struct {
    bool enabled;
    char broker[CONFIG_PATH_MAX];
    int port;
} mqtt_config_t;

typedef struct {
    bool enabled;
    char host[CONFIG_NAME_MAX];
    int port;
    int workers;
    int socket_timeout_sec;
    int session_timeout_sec;
    int llm_timeout_ms;
} network_config_t;

typedef struct {
    bool enabled;
} tui_config_t;

typedef struct {
    bool mic_record;
    bool asr_record;
    bool aec_record;
    char record_path[CONFIG_PATH_MAX];
} debug_config_t;

typedef struct {
    char music_dir[CONFIG_PATH_MAX];
    char commands_config[CONFIG_PATH_MAX];
} paths_config_t;

// Main config struct - all subsystems
typedef struct {
    general_config_t general;
    persona_config_t persona;
    localization_config_t localization;
    audio_config_t audio;
    vad_config_t vad;
    asr_config_t asr;
    tts_config_t tts;
    commands_config_t commands;
    llm_config_t llm;
    search_config_t search;
    url_fetcher_config_t url_fetcher;
    mqtt_config_t mqtt;
    network_config_t network;
    tui_config_t tui;
    debug_config_t debug;
    paths_config_t paths;
} dawn_config_t;

// Global config instance (read-only after initialization)
extern dawn_config_t g_config;
```

## Config Validation

Validation layer with structured error reporting:

```c
typedef struct {
    char field[64];
    char message[256];
} config_error_t;

/**
 * @brief Validate configuration values
 *
 * Checks:
 * - Range validation (thresholds 0.0-1.0, ports 1-65535, etc.)
 * - Dependency validation (cloud LLM requires API key)
 * - Path validation (model paths exist if required)
 *
 * @param config Configuration to validate
 * @param errors Array to receive error details
 * @param max_errors Maximum errors to report
 * @return Number of errors found (0 = valid)
 */
int config_validate(const dawn_config_t *config,
                    config_error_t *errors,
                    size_t max_errors);
```

### Validation Rules

| Field | Validation |
|-------|------------|
| `vad.speech_threshold` | 0.0 - 1.0 |
| `vad.speech_threshold_tts` | 0.0 - 1.0 |
| `vad.silence_threshold` | 0.0 - 1.0 |
| `tts.length_scale` | 0.5 - 2.0 |
| `network.port` | 1 - 65535 |
| `mqtt.port` | 1 - 65535 |
| `llm.type` | "cloud" or "local" |
| `commands.processing_mode` | "direct_only", "llm_only", "direct_first" |
| `asr.models_path` | Directory must exist (if ASR used) |
| `paths.commands_config` | File must exist and be valid JSON |
| `url_fetcher.flaresolverr.timeout_sec` | 1 - 300 |
| `url_fetcher.flaresolverr.max_response_bytes` | 1KB - 16MB |
| `url_fetcher.flaresolverr.endpoint` | Valid URL (if enabled) |

## API

```c
/**
 * @brief Load configuration from file
 *
 * @param config_path Path to config file, or NULL to search default locations
 * @return 0 on success, 1 on failure
 */
int config_load(const char *config_path);

/**
 * @brief Initialize config with default values
 *
 * @param config Config struct to initialize
 */
void config_set_defaults(dawn_config_t *config);

/**
 * @brief Load secrets from secrets.toml
 *
 * @return 0 on success, 1 on failure (secrets file optional)
 */
int config_load_secrets(void);

/**
 * @brief Dump all configuration values (for debugging)
 *
 * @param config Config to dump
 */
void config_dump(const dawn_config_t *config);

/**
 * @brief Dump only non-default configuration values
 *
 * @param config Config to compare against defaults
 */
void config_dump_diff(const dawn_config_t *config);
```

## Environment Variables

### Generic Prefix Pattern

All config values accessible via `DAWN_` prefix:
- `DAWN_AUDIO_BACKEND=alsa`
- `DAWN_VAD_SPEECH_THRESHOLD=0.6`
- `DAWN_LLM_TYPE=local`
- `DAWN_NETWORK_WORKERS=8`

Nested values use underscore:
- `DAWN_LLM_CLOUD_PROVIDER=claude`
- `DAWN_SEARCH_SUMMARIZER_BACKEND=local`

### Secret-Specific Variables (Higher Priority)

Standard API key environment variables:
- `OPENAI_API_KEY` → `api_keys.openai`
- `ANTHROPIC_API_KEY` → `api_keys.claude`
- `MQTT_USERNAME` → `mqtt.username`
- `MQTT_PASSWORD` → `mqtt.password`

## Thread Safety

**Assumption:** Configuration is loaded once at startup and read-only during runtime.

- Single load in `main()` before any threads are spawned
- Global `g_config` is read-only after initialization
- No mutex required for concurrent reads
- If runtime reload is ever needed, atomic config swap or reader-writer locks required

This assumption should be documented in the header file.

## What's NOT in Config

These remain as compile-time constants (#define):
- Protocol constants (packet sizes, magic bytes, checksums)
- Internal buffer sizes
- API URL paths (e.g., `/v1/chat/completions`)
- ASR engine selection (Whisper vs Vosk) - cmake option

## CLI Arguments

### New Arguments

- `--config=PATH` - Explicit config file path
- `--dump-config` - Print effective configuration and exit

### Existing Arguments (Override Config)

After config file support, these CLI args become shortcuts:
- `--capture=X` → `audio.capture_device`
- `--playback=X` → `audio.playback_device`
- `--llm=local|cloud` → `llm.type`
- `--summarizer=X` → `search.summarizer.backend`

CLI args still work and override config file values.

## Risk Assessment

| Risk Level | Component | Mitigation |
|------------|-----------|------------|
| Low | TOML parsing, config file search, CLI override | Isolated new code |
| Medium | Replacing #defines, VAD/ASR thresholds | Extensive testing, gradual rollout |
| Higher | Audio backend abstraction | Optional compile-time flag initially |

### Audio Backend Risk Mitigation

The audio backend abstraction touches critical real-time audio paths and AEC integration.
Implement as optional initially:

1. Add cmake flag: `-DENABLE_AUDIO_BACKEND_ABSTRACTION=OFF` (default)
2. When OFF, use existing direct ALSA code
3. When ON, use new backend abstraction
4. Validate extensively before making it the default

---

## Config Integration Status (Phase 5)

This section tracks config settings and their integration status.

### Completed Settings

| Setting | Status | Notes |
|---------|--------|-------|
| `general.log_file` | DONE | CLI override pattern, config fallback |
| `persona.description` | DONE | Used in llm_command_parser.c and dawn.c for system prompt |
| `audio.backend` | DONE | CLI override pattern, config selects ALSA/Pulse/auto |
| `commands.processing_mode` | DONE | CLI override pattern |
| `llm.type` | DONE | Selects cloud vs local in llm_interface.c |
| `llm.cloud.provider` | DONE | Selects OpenAI vs Claude |
| `llm.cloud.endpoint` | DONE | Custom proxy endpoints supported |
| `llm.cloud.model` | DONE | Model selection |
| `llm.local.endpoint` | DONE | Local LLM endpoint |
| `llm.local.model` | DONE | Local model selection |
| `llm.cloud.vision_enabled` | DONE | Vision capability for cloud LLM |
| `llm.local.vision_enabled` | DONE | Vision capability for local LLM (LLaVA, Qwen-VL) |
| `llm.max_tokens` | DONE | Used in llm_openai.c and llm_claude.c |
| `asr.model` | DONE | CLI override pattern |
| `asr.models_path` | DONE | CLI override pattern |
| `tts.voice_model` | DONE | Loaded from config in text_to_speech.cpp |
| `tts.length_scale` | DONE | Speech rate in text_to_speech.cpp |
| `search.endpoint` | DONE | Used in mosquitto_comms.c for web search |
| `audio.bargein.enabled` | DONE | Checked in dawn.c |
| `audio.bargein.cooldown_ms` | DONE | Used in dawn.c |
| `audio.bargein.startup_cooldown_ms` | DONE | Used in dawn.c |
| `audio.output_rate` | DONE | Accessor in audio_converter.c |
| `audio.output_channels` | DONE | Accessor in audio_converter.c |
| `audio.capture_device` | DONE | Config fallback in dawn.c |
| `audio.playback_device` | DONE | Config fallback in dawn.c |
| `vad.chunking.enabled` | DONE | Gate chunking logic in dawn.c |
| `vad.speech_threshold` | DONE | Used in dawn.c VAD init |
| `vad.silence_threshold` | DONE | Used in dawn.c VAD init |
| `vad.speech_duration_ms` | DONE | Used in dawn.c VAD init |
| `vad.silence_duration_ms` | DONE | Used in dawn.c VAD init |
| `vad.chunking.chunk_ms` | DONE | Used in dawn.c chunking init |
| `vad.chunking.overlap_ms` | DONE | Used in dawn.c chunking init |
| `mqtt.enabled` | DONE | Checked before connecting in dawn.c |
| `mqtt.broker` | DONE | Used in mosquitto_connect |
| `mqtt.port` | DONE | Used in mosquitto_connect |
| `network.enabled` | DONE | CLI override pattern, gate server startup |
| `network.host` | DONE | Used in dawn_server_start |
| `network.port` | DONE | Used in dawn_server_start |
| `network.socket_timeout_sec` | DONE | Used in dawn_server.c |
| `network.session_timeout_sec` | DONE | Used in session_manager.c |
| `tui.enabled` | DONE | CLI override pattern |
| `url_fetcher.whitelist` | DONE | Loaded in url_fetcher_init |
| `url_fetcher.flaresolverr.*` | DONE | Accessor functions in url_fetcher.c |
| `debug.mic_record` | DONE | CLI override pattern |
| `debug.asr_record` | DONE | CLI override pattern |
| `debug.aec_record` | DONE | CLI override pattern |
| `debug.record_path` | DONE | Applied if set in config |
| `paths.commands_config` | DONE | Used in dawn.c and llm_command_parser.c |
| `paths.music_dir` | DONE | Used in mosquitto_comms.c with fallback |
| `localization.location` | DONE | Default for weather in mosquitto_comms.c |
| `localization.units` | DONE | Added to system prompt in llm_command_parser.c |
| `secrets.mqtt_username` | DONE | Used with mosquitto_username_pw_set |
| `secrets.mqtt_password` | DONE | Used with mosquitto_username_pw_set |

### Recently Completed (Phase 5 Final)

| Setting | Status | Notes |
|---------|--------|-------|
| `vad.preroll_ms` | DONE | Dynamic preroll buffer (16kHz, max 2s) |
| `network.workers` | DONE | Dynamic worker pool (1-8, from config) |
| `network.llm_timeout_ms` | DONE | Curl TIMEOUT_MS on all LLM requests |

### About `search.engine`

The `search.engine` config field exists in the schema for future extensibility, but currently
only SearXNG is implemented. The field is defined but ignored by the code.

**Current status:** SearXNG is the only supported search backend. The `search.endpoint` config
allows pointing to any SearXNG instance.

**Future work:** If additional search engines are desired (DuckDuckGo, Brave, etc.), they would
require implementing new adapter code in `src/tools/web_search.c`. Each engine has different
APIs, authentication requirements, and rate limits. This is a feature addition beyond config
wiring scope.

### Notes

- Settings marked "CLI override pattern" respect command-line args as highest priority
- Search summarizer has its own `g_config` struct in `search_summarizer.c` - separate config
- Wake words loaded from config via `init_wake_words()` in dawn.c

---

**Document Version:** 2.6
**Date:** 2025-12-13
**Status:** Phase 5 config integration 100% complete - all settings wired
**Reviewer:** architecture-reviewer agent
