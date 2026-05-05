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
| [UNIFIED_LOGGING_DESIGN](dawn/archive/UNIFIED_LOGGING_DESIGN.md) | Canonical logging.h/logging.c shared byte-identically across DAWN, ECHO, MIRAGE, and STAT (OLOG_* namespace, syslog + console suppression + ms timestamps, copy-based sync) |
| [CONFIG_FILE_DESIGN](dawn/archive/CONFIG_FILE_DESIGN.md) | Full TOML configuration schema (dawn.toml, secrets.toml) |
| [CONFIG_SYSTEM_PLAN](dawn/archive/CONFIG_SYSTEM_PLAN.md) | Config system core infrastructure (Phase 1) |
| [PERFORMANCE_ANALYSIS](dawn/archive/PERFORMANCE_ANALYSIS.md) | Benchmark data vs industry (ASR latency, LLM throughput, end-to-end) |
| [SECURITY_AUDIT](dawn/archive/SECURITY_AUDIT.md) | Static code audit: 15 findings across ~58K LOC (Dec 2025) |
| [USER_AUTH_DESIGN](dawn/archive/USER_AUTH_DESIGN.md) | User authentication: deployment modes, dawn-admin CLI, setup wizard, multi-user, RBAC, session mgmt, audit log, IP rate limit, DAP2 registration key (Phases 0–4) |
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
| [WEBUI_ALWAYS_ON_PLAN](dawn/archive/WEBUI_ALWAYS_ON_PLAN.md) | Always-on continuous voice listening (server-side VAD, wake word, unified action button) |

### Satellite and Protocol

| Document | Description |
|----------|-------------|
| [DAP2_DESIGN](dawn/archive/DAP2_DESIGN.md) | Dawn Audio Protocol 2.0 — full design (Phases 0-4) |
| [PLEX_INTEGRATION_DESIGN](dawn/archive/PLEX_INTEGRATION_DESIGN.md) | Plex Media Server music streaming integration |
| [SMARTTHINGS](dawn/archive/SMARTTHINGS.md) | SmartThings OAuth integration (blocked at AWS WAF) |

### Memory Subsystem

Memory has its own subdirectory at [`dawn/memory/`](dawn/memory/) — see the [memory README](dawn/memory/README.md) for an annotated index. Entries below are the canonical pointers from the top-level Atlas index.

| Document | Description |
|----------|-------------|
| [SYSTEM_DESIGN](dawn/memory/SYSTEM_DESIGN.md) | Persistent memory: entity graph, relations, facts, semantic embeddings, hybrid search, contacts, entity merge, retrieval benchmarking (Phases 1–6.7 + S4 + 13) |
| [INJECTION_FILTER](dawn/memory/INJECTION_FILTER.md) | Memory injection filter: shared blocklist module with Unicode normalization (homoglyphs, accents, fullwidth, invisible chars), ~118 patterns across 17 categories, data-marking framing, all-path coverage (tool/extraction/import), 137 unit tests |
| [CAT2_TEMPORAL](dawn/memory/CAT2_TEMPORAL.md) | Cat-2 temporal extraction collapse (LoCoMo): failure-mode taxonomy (A1/A2/B/C/D/E), Phase 1 = conversation anchor injection (schema v42 `conversations.anchor_date`), live result `recall_generation` cat-2 0.022 → 0.321 (+29.9pp), overall +7.1pp. Phase 2 (`event_when` field) re-scoped after Phase 1 exceeded the original L1+L2 mid-projection. Companion: [CAT2_TEMPORAL_INVESTIGATION_PLAN](dawn/memory/CAT2_TEMPORAL_INVESTIGATION_PLAN.md) |
| [RERANKER_INVESTIGATION](dawn/memory/RERANKER_INVESTIGATION.md) | Cross-encoder reranker investigation: implemented (ms-marco-MiniLM-L-6-v2 int8 ONNX with CUDA EP, integration across memory + RAG paths, 5 config keys) then reverted after empirical results and literature review showed no net benefit on conversational data and only marginal lift on LongMemEval at 10× latency. Kept artifacts: shared WordPiece tokenizer (`memory_embed_tokenizer`), `rerank_shootout.py` test harness |
| [LOCOMO_CAT3_PROFILING](dawn/memory/LOCOMO_CAT3_PROFILING.md) | LoCoMo cat-3 failure-mode profiling, session-neighbor boost (Tier 2 quick win, +3.0pp dialog overall / +20.0pp cat-3), and memory-pipeline bench mode (Tier 1, Phase 0/1/1.5): end-to-end LoCoMo evaluation against extracted memory at production parity, `recall_reach` metric, Haiku 4.5 result of 0.742 / 0.646 cat-3 (+9.3pp / +20pp over dialog baseline). Identifies retrieval vs answer-support framing for closing the gap to leaders |

### RAG (Document Search)

| Document | Description |
|----------|-------------|
| [RAG_DESIGN](dawn/archive/RAG_DESIGN.md) | Document search / RAG: chunking, embeddings, hybrid semantic+keyword search, WebUI Document Library, admin management, `document_index` URL tool. Sibling to memory — shares `embedding_engine.c` but operates on `document_chunks`, not `memory_facts`. |

### Scheduler and Tools

| Document | Description |
|----------|-------------|
| [SCHEDULER_DESIGN](dawn/archive/SCHEDULER_DESIGN.md) | Timers, alarms, reminders, scheduled tasks — fully implemented |
| [TOOL_PLAN_EXECUTOR_DESIGN](dawn/archive/TOOL_PLAN_EXECUTOR_DESIGN.md) | Plan executor DSL: multi-step tool orchestration, conditionals, loops, variable binding, safety controls, sleep step |
| [WEB_IMAGE_SEARCH](dawn/archive/WEB_IMAGE_SEARCH.md) | Image search tool: SearXNG, curl_multi concurrent fetch, SSRF DNS pinning, magic byte validation, LRU cache, WebUI lightbox |
| [CALDAV_DESIGN](dawn/archive/CALDAV_DESIGN.md) | CalDAV calendar integration (multi-account, RFC 4791, Google OAuth, RRULE) |
| [EMAIL_DESIGN](dawn/archive/EMAIL_DESIGN.md) | Email integration (IMAP/SMTP, Gmail REST API, multi-account, 10 LLM actions) |
| [TWO_STEP_TOOL_PATTERN](dawn/archive/TWO_STEP_TOOL_PATTERN.md) | Two-step tool pattern: load guidelines then execute (used by render_visual) |
| [VISUAL_RENDERING_TOOL](dawn/archive/VISUAL_RENDERING_TOOL.md) | Visual rendering tool: inline SVG/HTML diagrams via LLM tool calling, progress indicator investigation |
