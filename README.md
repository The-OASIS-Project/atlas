# Atlas

Advanced Technical Library and Archival System

Historical design documentation for [The OASIS Project](https://github.com/The-OASIS-Project). These documents capture architectural decisions, completed feature designs, and implementation records that shaped the project. They are preserved here for reference after being retired from active repositories.

## DAWN Archive

Design documents from the [DAWN](https://github.com/The-OASIS-Project/dawn) voice assistant, organized by subsystem.

### Core Architecture

| Document | Description |
|----------|-------------|
| [MULTI_THREADED_CORE_DESIGN](dawn/archive/MULTI_THREADED_CORE_DESIGN.md) | Multi-threaded core: session manager, worker pool, per-session history, metrics |
| [UNIFIED_COMMAND_PLAN](dawn/archive/UNIFIED_COMMAND_PLAN.md) | Unified command registry replacing fragmented callback system |
| [CONFIG_FILE_DESIGN](dawn/archive/CONFIG_FILE_DESIGN.md) | Full TOML configuration schema (dawn.toml, secrets.toml) |
| [CONFIG_SYSTEM_PLAN](dawn/archive/CONFIG_SYSTEM_PLAN.md) | Config system core infrastructure (Phase 1) |
| [PERFORMANCE_ANALYSIS](dawn/archive/PERFORMANCE_ANALYSIS.md) | Benchmark data vs industry (ASR latency, LLM throughput, end-to-end) |
| [SECURITY_AUDIT](dawn/archive/SECURITY_AUDIT.md) | Static code audit: 15 findings across ~58K LOC (Dec 2025) |
| [TUI_IMPLEMENTATION_PLAN](dawn/archive/TUI_IMPLEMENTATION_PLAN.md) | Console TUI for real-time monitoring and statistics |

### Speech and Audio

| Document | Description |
|----------|-------------|
| [DAWN_ASR_UPGRADE_PLAN](dawn/archive/DAWN_ASR_UPGRADE_PLAN.md) | Vosk-to-Whisper ASR migration plan |
| [VAD_IMPLEMENTATION_NOTES](dawn/archive/VAD_IMPLEMENTATION_NOTES.md) | Silero VAD model selection rationale and Week 1 implementation |
| [PHASE_2_3_IMPLEMENTATION_PLAN](dawn/archive/PHASE_2_3_IMPLEMENTATION_PLAN.md) | Streaming ASR with Silero VAD + Whisper chunking (v1) |
| [PHASE_2_3_REVISED_PLAN](dawn/archive/PHASE_2_3_REVISED_PLAN.md) | Revised plan after whisper.cpp investigation (v2) |
| [PHASE_2_3_FINAL_DECISIONS](dawn/archive/PHASE_2_3_FINAL_DECISIONS.md) | Final implementation decisions — architecture review 9.0/10 |
| [AEC_DELAY_CALIBRATION](dawn/archive/AEC_DELAY_CALIBRATION.md) | Auto-calibrate AEC delay using TTS boot greeting |
| [AEC_IMPLEMENTATION_STATUS](dawn/archive/AEC_IMPLEMENTATION_STATUS.md) | Native 48kHz AEC with WebRTC AEC3 — working state |
| [AEC_IMPLEMENTATION_GUIDE](dawn/archive/AEC_IMPLEMENTATION_GUIDE.md) | WebRTC AEC3 setup, resampling strategy, tuning parameters |

### LLM Integration

| Document | Description |
|----------|-------------|
| [STREAMING](dawn/archive/STREAMING.md) | SSE streaming for OpenAI, Claude, and llama.cpp |
| [STREAMING_ARCHITECTURE](dawn/archive/STREAMING_ARCHITECTURE.md) | Streaming response architecture diagram and flow |
| [LLM_INTERRUPT_IMPLEMENTATION](dawn/archive/LLM_INTERRUPT_IMPLEMENTATION.md) | Non-blocking LLM interrupt with threading architecture |
| [LLM_RATE_LIMIT_DESIGN](dawn/archive/LLM_RATE_LIMIT_DESIGN.md) | Client-side rate limiter (sliding window, RPM-based) |
| [LLAMA_SERVER_OPTIMIZATION](dawn/archive/LLAMA_SERVER_OPTIMIZATION.md) | llama.cpp server tuning for Jetson |
| [MODEL_CONFIG_SYSTEM](dawn/archive/MODEL_CONFIG_SYSTEM.md) | Model-specific parameter optimization system |
| [NATIVE_TOOLS_PLAN](dawn/archive/NATIVE_TOOLS_PLAN.md) | Native tool/function calling implementation |
| [COMMAND_TAGS_DYNAMIC_GENERATION_PLAN](dawn/archive/COMMAND_TAGS_DYNAMIC_GENERATION_PLAN.md) | Dynamic command tag generation from tool registry |
| [SUMMARIZER_TFIDF_PLAN](dawn/archive/SUMMARIZER_TFIDF_PLAN.md) | Summarizer fallback fix + TF-IDF extractive summarization |

### WebUI

| Document | Description |
|----------|-------------|
| [WEBUI_DESIGN](dawn/archive/WEBUI_DESIGN.md) | WebUI architecture and feature documentation |
| [WEBUI_AESTHETIC_PLAN](dawn/archive/WEBUI_AESTHETIC_PLAN.md) | "Stark-Grade" visual overhaul plan |
| [WEBUI_SETTINGS_PLAN](dawn/archive/WEBUI_SETTINGS_PLAN.md) | Settings panel implementation |
| [WEBUI_VISION_DESIGN](dawn/archive/WEBUI_VISION_DESIGN.md) | Vision/image upload support design |
| [WEBUI_VISION_NEXT_STEPS](dawn/archive/WEBUI_VISION_NEXT_STEPS.md) | Vision feature implementation status |
| [WEBUI_IMAGE_STORAGE_DESIGN](dawn/archive/WEBUI_IMAGE_STORAGE_DESIGN.md) | Image storage strategy (replacing inline base64) |
| [CONVERSATION_HISTORY_DESIGN](dawn/archive/CONVERSATION_HISTORY_DESIGN.md) | Per-user conversation history UI |
| [CONVERSATION_EXPORT_DESIGN](dawn/archive/CONVERSATION_EXPORT_DESIGN.md) | JSON conversation export |

### Satellite and Protocol

| Document | Description |
|----------|-------------|
| [DAP2_DESIGN](dawn/archive/DAP2_DESIGN.md) | Dawn Audio Protocol 2.0 — full design (Phases 0-4) |
| [PLEX_INTEGRATION_DESIGN](dawn/archive/PLEX_INTEGRATION_DESIGN.md) | Plex Media Server music streaming integration |
| [SMARTTHINGS](dawn/archive/SMARTTHINGS.md) | SmartThings OAuth integration (blocked at AWS WAF) |

### Scheduler and Tools

| Document | Description |
|----------|-------------|
| [SCHEDULER_DESIGN](dawn/archive/SCHEDULER_DESIGN.md) | Timers, alarms, reminders, scheduled tasks — fully implemented |
| [CALDAV_DESIGN](dawn/archive/CALDAV_DESIGN.md) | CalDAV calendar integration (multi-account, RFC 4791, Google OAuth, RRULE) |
| [EMAIL_DESIGN](dawn/archive/EMAIL_DESIGN.md) | Email integration (IMAP/SMTP, Gmail REST API, multi-account, 10 LLM actions) |
| [TWO_STEP_TOOL_PATTERN](dawn/archive/TWO_STEP_TOOL_PATTERN.md) | Two-step tool pattern: load guidelines then execute (used by render_visual) |
| [VISUAL_RENDERING_TOOL](dawn/archive/VISUAL_RENDERING_TOOL.md) | Visual rendering tool: inline SVG/HTML diagrams via LLM tool calling, progress indicator investigation |
