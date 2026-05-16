# DAWN Memory System Design

**Status:** Phases 1-6.7, S4-S8 Complete - Core Memory, Decay, WebUI Viewer, Import/Export, Contacts WebUI, Entity Merge, Entity Graph, Embeddings, Injection Filter, Categorization, Temporal Scoring, Contradiction Detection
**Date:** January 2026
**Authors:** Kris Kersey, with input from community proposals
**Last Updated:** 2026-04-26

---

## What This Document Is

A comprehensive design for DAWN's persistent memory system with integrated RAG (Retrieval-Augmented Generation) for document search. All major design decisions have been finalized and documented here.

**Implementation Status:**

- **Phases 1-4 (Core Memory):** ✅ Complete - Storage, tool, context injection, and extraction
- **Phase 4.5 (Privacy Toggle):** ✅ Complete - Per-conversation privacy flag
- **Phase 5 (Decay/Maintenance):** ✅ Complete - Nightly decay job, pruning, reinforcement
- **Phase 6 (Memory WebUI):** ✅ Complete - Viewer with facts/preferences/summaries/graph/contacts tabs
- **Phase 6.5 (Import/Export):** ✅ Complete - JSON and plain text export, multi-format import with duplicate detection
- **Phase 6.6 (Contacts WebUI):** ✅ Complete - 5th tab in Memory popover, WebSocket CRUD, modal add/edit, entity cross-link from Graph tab
- **Phase 6.7 (Entity Merge):** ✅ Complete - Transactional merge with relation/contact dedup, LLM tool action + WebUI two-click merge
- **S4 (Entity Graph):** ✅ Complete - Entities, relations, embeddings, graph search, WebUI graph tab
- **S4b (Entity Dedup):** ✅ Complete - Existing entities fed into extraction prompt
- **S5 (Injection Filter):** ✅ Complete - Blocklist-based filter with Unicode normalization, 118+ patterns, wired into all storage paths
- **S6 (Fact Categorization):** ✅ Complete - 8-label taxonomy, embedding-centroid backfill, LLM recategorization, category-filtered search
- **S7 (Temporal Scoring):** ✅ Complete - Time-expression parser, Gaussian decay scoring, temporal relation bounds (valid_from/valid_to)
- **S8 (Contradiction Detection):** ✅ Complete - Relation-driven fact supersede, expanded exclusive relations (12), contradictory pairs (4), extraction prompt with 20 relation types
- **Phases 7-11 (RAG):** ✅ Implemented — see `docs/RAG_DESIGN.md` (separate system)
- **Retrieval Benchmarks:** ✅ Complete - LongMemEval turn-level 97.0% R@5 (27.2pp above published SOTA), ConvoMem 99.0%, LoCoMo 81.6% (bge-small int8 + proper-noun boost) — see `benchmarks/README.md`

---

## 1. Problem Statement

DAWN currently has no memory between sessions. Every conversation starts fresh. Users must re-explain context, preferences, and history. This makes DAWN feel like a tool rather than an assistant that knows you.

**What we want:**

- DAWN remembers facts you've told it ("I'm allergic to shellfish")
- DAWN adapts to your communication style over time
- DAWN can reference past conversations ("Last week you asked about...")
- DAWN can search your personal documents for answers
- DAWN does this without requiring explicit configuration

**What we don't want:**

- Massive storage requirements
- Significant latency added to conversations
- Privacy nightmares (storing full conversation logs indefinitely)
- Complexity that makes the system unmaintainable

---

## 2. Prior Art and Research

### 2.1 ChatGPT Memory ([OpenAI](https://openai.com/index/memory-and-new-controls-for-chatgpt/))

**Two-tier system:**

1. **Saved Memories** - Explicit facts ("Remember I'm vegetarian")
   - Injected directly into system prompt
   - User can view/edit/delete
   - Works like custom instructions

2. **Chat History** - Implicit learning from past conversations
   - Free users: "lightweight short-term continuity" (recent conversations only)
   - Plus/Pro users: "longer-term understanding" (searches across all history)

**Technical implementation** ([Analysis](https://embracethered.com/blog/posts/2025/chatgpt-how-does-chat-history-memory-preferences-work/)):

- Saved memories appear to be direct context injection
- Chat history may use RAG-style retrieval
- Context budget not publicly documented, but saved memories are compact

### 2.2 Claude Memory ([Anthropic](https://www.anthropic.com/news/memory))

**File-based, project-scoped:**

- Memory stored in CLAUDE.md files (plain Markdown)
- "Automatically loaded into context when launched"
- **Project-scoped** - memories don't leak between projects

**Best practices** ([Guide](https://support.claude.com/en/articles/11817273-using-claude-s-chat-search-and-memory-to-build-on-previous-context)):

- Keep CLAUDE.md **minimal** - only essential information
- Store detailed knowledge in separate docs, reference only when needed
- Use `/clear` between tasks, `/compact` to summarize

### 2.3 Key Insights for DAWN

From studying commercial implementations:

- **Hybrid loading**: Small set of core facts always loaded, additional context retrieved on-demand
- **Budget consciousness**: Neither loads everything into context
- **User control**: Full transparency about what's remembered, easy deletion
- **Separation**: Explicit vs inferred memories treated differently

### 2.4 Other Systems

| System                        | Insight                                   | Why Not Use Directly                |
| ----------------------------- | ----------------------------------------- | ----------------------------------- |
| MemGPT/Letta                  | Tiered memory with LLM-managed operations | Python, cloud-focused, adds latency |
| LangChain                     | Multiple memory backend types             | Python dependency                   |
| Vector DBs (Pinecone, Chroma) | Good for large-scale search               | Overkill for personal assistant     |

---

## 3. Design Decisions (Finalized)

### 3.1 User Identification

| Interface           | Strategy               | Memory Behavior                                   |
| ------------------- | ---------------------- | ------------------------------------------------- |
| **WebUI**           | Auth username          | Full memory storage and retrieval                 |
| **Local mic**       | Configurable mapping   | Default: no memory. Can map to user in config     |
| **DAP2 satellites** | Admin-managed DB mapping | Default: no memory. Assign user via WebUI Satellite Management panel |
| **Future**          | Speaker identification | sherpa-onnx integration (Phase 5+)                |

**Decision:** Voice interfaces (local mic, satellites) do NOT store memories by default. They operate as "guest" sessions. Users can optionally map these interfaces to authenticated users.

**Configuration:**

```toml
[memory.voice_mapping]
# Map local mic to authenticated user
# If not mapped, voice sessions don't store memories (guest mode)
local_mic = "krisk"              # Local mic → user "krisk"
# DAP2 satellites: assign user via WebUI Satellite Management admin panel
# (stored in satellite_mappings DB table, not TOML config)
```

**Why guest by default?**

- Multi-person households: Anyone can talk to DAWN without polluting the owner's memory
- Privacy: Visitors don't accidentally store personal facts
- Explicit opt-in: Users must consciously enable memory for voice interfaces
- Speaker ID future: When implemented, will provide automatic user resolution

### 3.2 Extraction Model

Extraction uses a **dedicated model configuration** separate from conversation:

```toml
[memory]
enabled = true
context_budget_tokens = 800
session_timeout_minutes = 15

[memory.extraction]
provider = "local"           # "local", "openai", "claude", "ollama"
model = "qwen2.5:7b"         # Model name for that provider
# If not specified, falls back to conversation model
```

**Decision:** Globally configurable provider + model for consistency across all extraction.

#### 3.2.1 Four-Model Extraction Sweep (May 2026)

> **Headline-number policy**: numbers in this section (and any other section
> that cites bench results) are a **snapshot in time**. The single source of
> truth for the current leader-comparable position lives in
> [STATE.md](STATE.md) — refresh that file when results move, and treat
> anything quoted here as a historical comparison point rather than an
> up-to-date scoreboard. The `recall_reach` numbers in particular are an
> internal diagnostic metric and are **not directly comparable** to
> published leader headlines (those are LLM-judge generation under the
> Mem0 protocol — see `dawn/benchmarks/README.md`).

Full LoCoMo (10 convs, 1982 QA) was run with four extraction models, holding
embedding model + judge + generator constant at `bge-small-en-v1.5-int8` /
`claude-haiku-4-5` / `claude-haiku-4-5`:

| Extraction model | reach | ent | gen |
|---|---|---|---|
| `claude-haiku-4-5` | 0.7392 | 0.2568 | 0.2084 |
| `claude-sonnet-4-6` | **0.7506** | **0.2583** | **0.2250** |
| `claude-opus-4-6` | 0.6739 | 0.1821 | 0.1599 |
| `Qwen3.6-35B-A3B-Q4_K_M` (local) | 0.6186 | 0.1700 | 0.1564 |

**Findings:**

1. **Sonnet wins overall but only barely** — `recall_generation` +1.7 pp
   over Haiku, `recall_entailment` essentially tied. For ~5× the cost, this
   is not a meaningful price/performance gain.
2. **Opus is meaningfully worse than Haiku** — `gen` -4.9 pp, `ent` -7.5 pp.
   Inspection shows Opus produces 2-3× the entity count per conversation
   (~43 entities/conv vs Haiku's ~15), suggesting it over-fragments the
   entity graph; the resulting facts retrieve less reliably even though
   raw fact counts are similar.
3. **Local Qwen3.6-35B-A3B trails the cloud field** — `gen` 0.156 is just
   below Opus and well below Haiku/Sonnet. For a fully-local deployment
   it's a real option (free, runs in ~3 hours); for cloud-OK deployments
   Haiku is plainly better.
4. **Cat-2 temporal is broken across all four models** — 0.02-0.03
   `recall_generation`. The bottleneck is extraction-side info loss
   (dates getting dropped during fact extraction), not model strength.
   This is independent of the model-selection question and is the
   highest-leverage future improvement target.

**Recommendation:** `claude-haiku-4-5` for both `compact_model` and
`extraction_model`. The same model for summarization and extraction is fine —
the sweep gives no evidence that a stronger Claude tier produces better
extraction shape, and gives clear evidence that the strongest tier (Opus)
is actively worse. Going larger is *not* a free win; the model has to follow
the extraction prompt's structural expectations, not just generate text.

Raw data: `benchmarks/snapshots/sweep_results.json`. Reproduction:
`benchmarks/README.md` "four-model extraction sweep" section.

### 3.3 Context Budget

**800 tokens default**, configurable via `memory.context_budget_tokens`.

| Type                       | Loading Strategy    | Budget                     |
| -------------------------- | ------------------- | -------------------------- |
| **Core facts/preferences** | Always injected     | ~400 tokens                |
| **Recent summaries**       | Last 3 sessions     | ~400 tokens                |
| **RAG documents**          | Retrieved on-demand | 0 base, ~500 when relevant |

**Decision:** Core facts always loaded. RAG adds context only when query needs it (hybrid approach matching commercial implementations).

### 3.4 Document Formats (RAG)

| Phase       | Format  | Library          | Notes                                           |
| ----------- | ------- | ---------------- | ----------------------------------------------- |
| **Phase 1** | TXT, MD | stdlib           | No dependencies                                 |
| **Phase 2** | PDF     | poppler          | `apt install poppler-utils` or libpoppler C API |
| **Phase 3** | DOCX    | libzip + libxml2 | DOCX is ZIP of XML files                        |

**Decision:** Implement all three phases. Skip legacy `.doc` format (binary nightmare).

### 3.5 Embedding Model

Multi-provider embedding support configured via `[memory.embeddings]`:

| Provider             | Model                  | Dimensions | Speed  | Setup                              |
| -------------------- | ---------------------- | ---------- | ------ | ---------------------------------- |
| **Ollama** (default) | nomic-embed-text       | 768        | ~20ms  | `ollama pull nomic-embed-text`     |
| **OpenAI**           | text-embedding-3-small | 1536       | ~100ms | Requires API key                   |
| **ONNX**             | bge-small-en-v1.5 int8 | 384        | ~15ms  | **Current default** (May 2026); +7.2pp LoCoMo vs MiniLM; shares ONNX Runtime with Piper TTS |
| **ONNX** (previous) | all-MiniLM-L6-v2 int8  | 384        | ~8ms   | Superseded by bge-small; retained on disk for rollback |

**Decision:** Support multiple embedding providers to match the LLM provider flexibility. Ollama is the default for local deployments. ONNX leverages existing runtime dependency. OpenAI provides cloud option for higher quality embeddings. Embeddings are used for both facts and entities.

### 3.6 Storage Architecture

**Single SQLite database** (`auth.db`) with prefixed tables:

```
auth.db
├── users (existing auth)
├── sessions (existing auth)
├── memory_facts           # Discrete facts with optional embeddings
├── memory_preferences     # Communication style preferences
├── memory_summaries       # Conversation digests
├── memory_entities        # People, places, pets, projects (with embeddings)
├── memory_relations       # Entity-to-entity relationships
├── rag_documents          # (future)
└── rag_chunks             # (future)
```

**Decision:** Shared connection pool, single backup file, foreign keys to users table.

### 3.7 Consolidation Timing

| Trigger         | Purpose                                                  |
| --------------- | -------------------------------------------------------- |
| **Session end** | Primary extraction (WebSocket disconnect/timeout)        |
| **Nightly job** | Decay, pruning, crash recovery (unconsolidated sessions) |

**Decision:** Session-end extraction + nightly maintenance at configurable hour.

#### 3.7.1 Session End Detection

A "session end" triggers memory extraction. Detection varies by interface:

| Interface           | Session End Signal                    | Implementation                      |
| ------------------- | ------------------------------------- | ----------------------------------- |
| **WebUI**           | WebSocket close                       | `onclose` event in `webui_server.c` |
| **WebUI (timeout)** | No messages for N minutes             | Server-side timer per connection    |
| **Local mic**       | Silence timeout + no pending response | Existing VAD timeout mechanism      |
| **DAP satellites**  | TCP disconnect                        | Connection close handler            |

**Configuration:**

```toml
[memory]
session_timeout_minutes = 15    # Inactivity timeout before session end
```

**Implementation approach:**

1. Track `last_activity_timestamp` per session
2. Check on each message: if `now - last_activity > timeout`, trigger extraction first
3. On WebSocket close: trigger extraction (if not already done)
4. Store sessions with `consolidated = false` until extraction completes
5. Nightly job processes any `consolidated = false` sessions (crash recovery)

**Race Condition Handling:**
If a user starts a new session while extraction from the previous session is still running:

- Load existing (stale) memories immediately - don't block on extraction
- Don't update memories during the new session until extraction completes
- New session's extraction runs after the pending one completes
- This keeps the UX responsive while ensuring eventual consistency

### 3.8 Memory Disclosure

**Full transparency:**

- Voice: Summarize on request ("I remember these things about you...")
- WebUI: Dedicated "Memory" section showing all stored facts/preferences
- Delete capability: "Forget that I'm vegetarian" or WebUI delete button

**Decision:** Users can always see and delete their memories.

---

## 4. Core Architecture

### 4.1 The "Sleep Consolidation" Model

Memory processing happens **after sessions end**, not during. Inspired by human memory consolidation during sleep (from "Building Your Own JARVIS" presentation).

**Leveraging Existing Infrastructure:** DAWN already stores conversation messages in the `messages` table (via `auth_db_conv.c`). Memory extraction reads from this existing storage - no separate transcript storage needed.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DURING CONVERSATION (Working Memory)              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  • Full conversation in LLM context window                          │
│  • No extraction happening (zero latency impact)                    │
│  • Core facts + preferences pre-loaded at session start             │
│  • RAG retrieves relevant documents on-demand                       │
│  • Messages stored in existing `messages` table (per conversation)  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                │ Session ends (disconnect/timeout)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SESSION END (Short-term → Long-term)              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Extraction LLM processes transcript:                                │
│                                                                      │
│  1. Extract FACTS                                                    │
│     "User mentioned they are vegetarian" → memory_facts             │
│     "User's daughter is named Dawn" → memory_facts                  │
│                                                                      │
│  2. Extract PREFERENCES                                              │
│     "User asked for shorter responses twice" → memory_preferences   │
│                                                                      │
│  3. Extract ENTITIES + RELATIONS                                     │
│     "Jon" (person), "Buddy" (pet) → memory_entities                │
│     "Jon" → owns → "Buddy" → memory_relations                     │
│     Existing entity names fed into prompt to prevent duplicates     │
│                                                                      │
│  4. Generate SUMMARY                                                 │
│     "Discussed home automation setup for garage lights"             │
│     → memory_summaries                                               │
│                                                                      │
│  5. Generate embeddings for new facts and entities                   │
│                                                                      │
│  6. Mark transcript as "consolidated"                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                │ Nightly job (configurable hour)
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    NIGHTLY CLEANUP (Memory Decay)                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Process any unconsolidated sessions (crash recovery)            │
│                                                                      │
│  2. Apply confidence decay (atomic SQL UPDATE with powf()):          │
│     • Inferred facts: decay 5% per week unused (floor configurable) │
│     • Explicit facts: decay 2% per week (floor 0.50)               │
│     • Preferences: decay 3% per week (floor 0.40)                  │
│                                                                      │
│  3. Prune low-confidence items:                                      │
│     • confidence < 0.25 → log for audit trail, then delete          │
│     • Explicit facts floor at 0.50 (never fully forgotten)          │
│     • Inferred facts floor configurable (default 0.0)               │
│                                                                      │
│  4. Prune old summaries:                                             │
│     • Keep last N days of summaries (configurable, default 30)      │
│                                                                      │
│  5. Reinforce accessed items (time-gated):                           │
│     • Facts loaded into context: confidence += 0.05                 │
│     • Time-gated: only boosts if last_accessed > 1 hour ago         │
│     • Prevents confidence pinning from automated queries            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Why Batch Processing ("Sleep") Instead of Real-Time

| Real-Time Extraction                   | Batch/Sleep Extraction           |
| -------------------------------------- | -------------------------------- |
| Adds latency to every Nth response     | Zero conversation latency        |
| Must use fast model (quality tradeoff) | Can use slower, better model     |
| Interrupts conversation flow           | Invisible to user                |
| Complex state management               | Simple: process transcript, done |
| Hard to handle long conversations      | Natural boundary at session end  |

**Tradeoff:** Information isn't available until next session. User says "remember I hate cilantro" - DAWN won't "know" this until after consolidation runs. Acceptable for most use cases.

---

## 5. RAG (Retrieval-Augmented Generation)

### 5.1 What is RAG?

RAG allows DAWN to search your personal documents and include relevant excerpts in responses.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INDEXING (once per document)                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Document: "The garage door opener uses a 315MHz frequency..."      │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────┐                                                │
│  │ Embedding Model │  (all-MiniLM-L6-v2 via ONNX Runtime)           │
│  └────────┬────────┘                                                │
│           ▼                                                          │
│  Vector: [0.12, -0.45, 0.78, ...]  (384 floats)                     │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │    SQLite DB    │  Store: text chunk + vector + metadata         │
│  └─────────────────┘                                                │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                         RETRIEVAL (every query)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User: "What frequency does my garage door use?"                    │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────┐                                                │
│  │ Embedding Model │  Same model as indexing                        │
│  └────────┬────────┘                                                │
│           ▼                                                          │
│  Query Vector: [0.08, -0.52, 0.81, ...]                             │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │  Vector Search  │  Find closest matches (cosine similarity)      │
│  │    (SQLite)     │                                                │
│  └────────┬────────┘                                                │
│           ▼                                                          │
│  Top 3 chunks (similarity > 0.5):                                    │
│  1. "The garage door opener uses a 315MHz frequency..." (0.91)      │
│  2. "Remote controls typically operate on 315 or 433MHz" (0.78)     │
│           │                                                          │
│           ▼                                                          │
│  Inject into LLM prompt as context                                   │
│           │                                                          │
│           ▼                                                          │
│  LLM: "Your garage door opener uses 315MHz frequency."              │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 What Are Embeddings?

An **embedding** represents text as a list of numbers (a **vector**) where **similar meanings produce similar numbers**.

```
"I love pizza"     → [0.82, -0.14, 0.67, 0.23, ...]  (384 numbers)
"Pizza is great"   → [0.79, -0.11, 0.71, 0.19, ...]  (very similar!)
"The stock market" → [-0.45, 0.88, -0.12, 0.55, ...] (very different)
```

**Cosine similarity** measures how "aligned" two vectors are:

- 1.0 = identical meaning
- 0.0 = unrelated
- -1.0 = opposite meaning

### 5.3 Document Ingestion

**Configuration:**

```toml
[rag]
enabled = true
documents_dir = "~/Documents/dawn-knowledge"
chunk_size_tokens = 256
chunk_overlap_tokens = 50
similarity_threshold = 0.5
max_results = 5
```

**Supported formats:**
| Format | Library | Implementation |
|--------|---------|----------------|
| TXT, MD | stdlib | Direct file read |
| PDF | poppler | `pdftotext` CLI or libpoppler C API |
| DOCX | libzip + libxml2 | Extract `word/document.xml`, parse `<w:t>` elements |

**Chunking strategy:**

- Chunk by paragraph with configurable overlap
- ~256 tokens per chunk (tunable)
- 50-token overlap for context continuity

### 5.4 Ingestion Timing

Documents are indexed at multiple points to balance freshness with performance:

| Trigger            | When              | Behavior                                          |
| ------------------ | ----------------- | ------------------------------------------------- |
| **Startup scan**   | DAWN starts       | Check configured directory for new/changed files  |
| **File watcher**   | Runtime           | inotify (Linux) watches documents_dir for changes |
| **Manual trigger** | User request      | WebUI "Re-index" button or voice command          |
| **Nightly job**    | Configurable hour | Full directory scan, cleanup orphaned chunks      |

**Configuration:**

```toml
[rag]
enabled = true
documents_dir = "~/Documents/dawn-knowledge"
reindex_on_startup = true       # Scan for changes at startup
watch_for_changes = false       # Disabled by default on embedded (saves resources)
max_chunks = 5000               # Warn when approaching limit
max_document_size_mb = 10       # Skip files larger than this
```

**Scaling Limits:**

- `max_chunks = 5000`: Vector search is O(n), stays fast up to ~5000 chunks (~50-100 documents)
- `max_document_size_mb = 10`: Prevents memory issues during PDF/DOCX parsing
- Warnings logged when limits approached

**Document Management (v1):**

- Admin places files in `documents_dir` manually (SSH, SFTP, file manager)
- All authenticated users can search all indexed documents (shared knowledge)
- WebUI shows index status, allows re-indexing, but no file upload

**Change detection:**

- File hash (SHA256) stored in `rag_documents.file_hash`
- On scan, compare current hash to stored hash
- If different: delete old chunks, re-index entire file
- If missing from disk: delete document and chunks from DB

**Indexing priority:**

1. Small files processed first (quick availability)
2. Large files processed in background (non-blocking)
3. New files prioritized over re-indexing changed files

### 5.5 Storage Cost

Each chunk: `384 floats × 4 bytes = 1,536 bytes` per embedding

For 1,000 document chunks: ~1.5 MB of vector data (trivial)

---

## 6. Storage Schema

Using SQLite (already a DAWN dependency for auth). Current schema version: **v48**. The schemas below regenerate from `src/auth/auth_db_core.c` (`SCHEMA_SQL` + the v33-onward migration blocks); the version constant lives in `include/auth/auth_db_internal.h:51` (`AUTH_DB_SCHEMA_VERSION`).

Convention notes:
- All `created_at` / `last_accessed` / `first_seen` / `last_seen` / `valid_from` / `valid_to` / `linked_at` / `proposed_at` columns are **`INTEGER` Unix epoch seconds**, not SQL `TIMESTAMP`. Defaults are `(strftime('%s','now'))` for "set on insert" columns and literal constants (`0` / `NULL`) for columns added via post-v33 migrations (literal-constant defaults take SQLite's O(1) metadata-only ALTER path — no full-table rewrite under the auth_db lock at startup).
- All foreign keys to `users(id)` cascade on user delete. Schema text below shows the live database shape; the migration history adds columns one at a time, so column ordering reflects insert-history rather than logical grouping.
- All "source_*" provenance columns (added v40 / extended v42 in Phase B) point back into `conversations` / `messages` so retrieval can render the original utterance — see [PROVENANCE.md](PROVENANCE.md).
- v43-v46 add the entity-alias / user-identity / summary-embedding workstream on top of the v33-v42 base. See §6.4 (entities `canonical_id` + `is_user_self`), §6.4.1-§6.4.2 (alias audit log + review-band proposals), §6.3 (`memory_summaries.embedding` BLOB), §6.9 (users `real_name` / `preferred_address` / `identity_aliases`).
- **v47** adds `memory_facts.subject_entity_id` — a hard FK from each fact to the entity it concerns (Phase 0 extraction-prompt redesign, May 2026; see §6.1). Nullable during transition; backfilled from linked relations on first boot at v47.
- **v48** adds the `memory_facts_fts` FTS5 virtual table — a contentless BM25 keyword index over Porter2-stemmed fact text (Phase 1 of the Mem0 architectural parity plan, May 2026; see §6.11 and §9.1.1 Search Strategy). Backfilled at migration time; tracked under `[memory] bm25_enabled` (default `false` until Phase 1 is bench-validated end-to-end).

### 6.1 Memory Facts Table

```sql
CREATE TABLE memory_facts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    fact_text TEXT NOT NULL,            -- "User is allergic to shellfish"
    confidence REAL DEFAULT 1.0,        -- 0.0-1.0
    source TEXT DEFAULT 'inferred',     -- 'explicit', 'inferred'
    created_at INTEGER NOT NULL DEFAULT (strftime('%s','now')),
    last_accessed INTEGER,
    access_count INTEGER DEFAULT 0,
    superseded_by INTEGER,              -- FK to newer fact if corrected
    normalized_hash INTEGER DEFAULT 0,  -- FNV-1a hash for fast dup detection
    embedding BLOB DEFAULT NULL,        -- bge-small-en-v1.5-int8 dim, see §3.5
    embedding_norm REAL DEFAULT NULL,   -- pre-computed L2 norm
    category TEXT NOT NULL DEFAULT 'general',  -- v34 — 8-label taxonomy, see §6.1.1
    source_conversation_id INTEGER DEFAULT NULL,  -- v40 provenance triple
    source_msg_id_start    INTEGER DEFAULT NULL,
    source_msg_id_end      INTEGER DEFAULT NULL,
    subject_entity_id      INTEGER DEFAULT NULL,  -- v47 — hard FK to subject entity
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (superseded_by) REFERENCES memory_facts(id) ON DELETE SET NULL,
    FOREIGN KEY (subject_entity_id) REFERENCES memory_entities(id) ON DELETE SET NULL
);

CREATE INDEX idx_memory_facts_user           ON memory_facts(user_id);
CREATE INDEX idx_memory_facts_confidence     ON memory_facts(user_id, confidence DESC);
CREATE INDEX idx_memory_facts_hash           ON memory_facts(user_id, normalized_hash);
CREATE INDEX idx_memory_facts_user_category  ON memory_facts(user_id, category);
CREATE INDEX idx_memory_facts_subject        ON memory_facts(subject_entity_id)
   WHERE subject_entity_id IS NOT NULL;
```

**`subject_entity_id` (v47)** is the Phase 0 extraction-prompt redesign's structural link from fact → entity. The redesigned prompt (May 2026) requires every fact to declare its subject (precedence ladder: `real_name` → named entity → descriptor → `"User"`); the parser resolves the subject string to an entity row before insert and stores the resulting FK here. Pre-v47 facts get backfilled from their linked relations at migration time (`UPDATE memory_facts SET subject_entity_id = (SELECT MIN(r.subject_entity_id) FROM memory_relations r WHERE r.fact_id = id AND r.subject_entity_id IS NOT NULL) WHERE subject_entity_id IS NULL`). Currently NULLABLE; planned to tighten to `NOT NULL` once production telemetry confirms the post-Phase-0 NULL rate stays near zero. The hard FK eliminated the legacy fact↔relation text-matching code (`find_fact_for_relation` retired) and lifted Phase 1A entity-graph retrieval (see §9.1.1) from a side path to the default search posture.

#### 6.1.1 Fact category taxonomy (v34)

`category` is one of 8 fixed labels — `personal` / `professional` / `relationships` / `health` / `interests` / `practical` / `preferences` / `general` — defined as a single source of truth in `include/memory/memory_types.h` (`MEMORY_FACT_CATEGORIES[]` + `MEMORY_FACT_CATEGORY_COUNT`). Validated in C; unknown labels are coerced to `general`. Pre-filtered at the SQL level via `memory_db_fact_search_by_category()` so hybrid scoring only sees the right slice when the LLM passes `category` to the search tool.

### 6.2 Memory Preferences Table

```sql
CREATE TABLE memory_preferences (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    category TEXT NOT NULL,             -- 'verbosity', 'humor', 'formality', 'detail_level'
    value TEXT NOT NULL,                -- "prefers concise responses"
    confidence REAL DEFAULT 0.5,
    source TEXT DEFAULT 'inferred',     -- 'explicit', 'inferred'
    created_at INTEGER NOT NULL DEFAULT (strftime('%s','now')),
    updated_at INTEGER NOT NULL DEFAULT (strftime('%s','now')),
    reinforcement_count INTEGER DEFAULT 1,
    source_conversation_id INTEGER DEFAULT NULL,  -- v42 — Phase B provenance
    source_msg_id_start    INTEGER DEFAULT NULL,
    source_msg_id_end      INTEGER DEFAULT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE(user_id, category)
);
```

### 6.3 Memory Summaries Table

```sql
CREATE TABLE memory_summaries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    session_id TEXT NOT NULL,
    summary TEXT NOT NULL,              -- "Discussed home automation setup..." (~850 chars after v45 prompt bump)
    topics TEXT,                        -- JSON array or comma-joined: "home automation, mqtt"
    sentiment TEXT,                     -- 'positive', 'neutral', 'frustrated'
    created_at INTEGER NOT NULL DEFAULT (strftime('%s','now')),
    message_count INTEGER,
    duration_seconds INTEGER,
    consolidated INTEGER DEFAULT 0,
    source_conversation_id INTEGER DEFAULT NULL,  -- v42 — Phase B provenance
    source_msg_id_start    INTEGER DEFAULT NULL,
    source_msg_id_end      INTEGER DEFAULT NULL,
    embedding BLOB DEFAULT NULL,        -- v45 — semantic summary search (bge-small-en-v1.5-int8)
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (source_conversation_id) REFERENCES conversations(id) ON DELETE SET NULL
);

CREATE INDEX idx_memory_summaries_user ON memory_summaries(user_id, created_at DESC);
```

`embedding` (v45) is populated by `memory_embeddings_embed_and_store_summary()` at extraction time and backfilled by the recompute worker for pre-v45 rows. The v46 migration NULLs `users.embeddings_model_id` so the recompute worker picks every user up as stale and backfills the column on next boot. The summary adapter (`memory_focus_adapters.c`) hybrids keyword + cosine when this is populated; NULL rows fall back to keyword-only matching. See [SUMMARY_SEMANTIC.md](SUMMARY_SEMANTIC.md) if it exists or §11 / S9 for the surrounding workstream.

### 6.4 Memory Entities Table

```sql
CREATE TABLE memory_entities (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    name TEXT NOT NULL,                 -- Display name (e.g., "Jonathan Smith")
    entity_type TEXT NOT NULL,          -- "person", "place", "pet", "project", "thing"
    canonical_name TEXT NOT NULL,       -- Lowercase normalized (e.g., "jonathan smith")
    embedding BLOB DEFAULT NULL,        -- Float array for semantic search
    embedding_norm REAL DEFAULT NULL,   -- Pre-computed L2 norm
    photo_id TEXT DEFAULT NULL,         -- Image-store id for the entity portrait
    first_seen INTEGER NOT NULL DEFAULT (strftime('%s','now')),
    last_seen INTEGER,
    mention_count INTEGER DEFAULT 1,
    canonical_id INTEGER DEFAULT NULL REFERENCES memory_entities(id) ON DELETE SET NULL,
                                        -- v43 — soft-alias self-FK (NULL = self is canonical)
    is_user_self INTEGER NOT NULL DEFAULT 0,
                                        -- v43 — exactly-one-per-user flag (anchor entity)
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE(user_id, canonical_name)
);

CREATE INDEX idx_memory_entities_user      ON memory_entities(user_id);
CREATE INDEX idx_memory_entities_canonical ON memory_entities(canonical_id) WHERE canonical_id IS NOT NULL;
CREATE UNIQUE INDEX idx_memory_entities_user_self
   ON memory_entities(user_id) WHERE is_user_self = 1;
```

**Soft-alias semantics (v43)**: `canonical_id IS NULL` means the row is canonical; `canonical_id = N` means the row is a soft alias of entity N. The link is reversible — `memory_db_entity_alias_unlink()` clears `canonical_id` back to NULL without losing the row. Read paths (`memory_db_entity_get_by_name`, `memory_db_entity_search`, the admin canonical-list) aggregate `mention_count` / `first_seen` / `last_seen` across the canonical's equivalence class via correlated subqueries — see Bundle 2 (commit `0e9ae1e`, 2026-05-13). The single-level alias invariant is enforced at `alias_link` time: a row whose own id is already referenced as another row's `canonical_id` cannot itself become an alias, which bounds equivalence-class depth to 1.

**`is_user_self` (v43)**: the partial UNIQUE index `idx_memory_entities_user_self WHERE is_user_self = 1` enforces "exactly one user-self entity per user." Auto-promoted at extraction when a fresh canonical matches `users.real_name`; also promoted lazily on Settings save by `memory_db_entity_auto_promote_user_self_by_real_name()`. Carries a +0.20 bonus in the entity-merge composite scorer so user-self mentions cluster tightly. See §11 Phase 6.7+ for the entity-merge workstream.

### 6.4.1 Memory Entity Aliases (v43)

Append-only audit log for soft + hard merges. Not consulted on hot read paths — those use `canonical_id` JOINs — but answers "why was X linked to Y, and when?" for the WebUI Graph tab and `dawn-admin memory entity history`.

```sql
CREATE TABLE memory_entity_aliases (
    id                    INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id               INTEGER NOT NULL,
    source_entity_id      INTEGER,                 -- nullable: SET NULL on hard-merge delete
    target_entity_id      INTEGER NOT NULL,
    source_canonical_name TEXT NOT NULL,           -- preserved so the audit row survives the row delete
    target_canonical_name TEXT NOT NULL,
    link_kind             TEXT NOT NULL,           -- "soft", "hard", "user_self_promote", "synthetic_self"
    reason                TEXT NOT NULL,           -- "auto_merge_phase2", "operator_webui", "auto_promote_realname", ...
    composite_score       REAL,                    -- entity-merge composite at link time (NULL for promotions)
    evidence_json         TEXT,                    -- per-stage scoring breakdown for debugging
    linked_at             INTEGER NOT NULL,
    consolidated_at       INTEGER,                 -- timestamp of soft → hard promotion (NULL if still soft)
    unlinked_at           INTEGER,                 -- timestamp of unlink operation (NULL if still active)
    unlink_reason         TEXT,
    FOREIGN KEY (user_id)          REFERENCES users(id)            ON DELETE CASCADE,
    FOREIGN KEY (source_entity_id) REFERENCES memory_entities(id)  ON DELETE SET NULL,
    FOREIGN KEY (target_entity_id) REFERENCES memory_entities(id)  ON DELETE SET NULL
);

CREATE INDEX idx_memory_entity_aliases_user_target
   ON memory_entity_aliases(user_id, target_entity_id)
   WHERE unlinked_at IS NULL;
```

### 6.4.2 Memory Entity Merge Proposals (v43)

Review-band staging for the Phase 2 auto-merge gate. When the entity-merge cascade scores a candidate pair between `review_threshold` (default 0.50) and `auto_threshold` (default 0.90), it lands here instead of being applied directly. Approving a proposal writes the soft link via the regular alias path; rejecting just stamps `resolved_at`. Cleared by reextract alongside `memory_entity_aliases` — both are derived state.

```sql
CREATE TABLE memory_entity_merge_proposals (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id          INTEGER NOT NULL,
    source_entity_id INTEGER NOT NULL,
    target_entity_id INTEGER NOT NULL,
    composite_score  REAL NOT NULL,
    evidence_json    TEXT NOT NULL,
    proposed_at      INTEGER NOT NULL,
    resolved_at      INTEGER,
    resolution       TEXT,                         -- "approved", "rejected", "deferred"
    FOREIGN KEY (user_id)          REFERENCES users(id)            ON DELETE CASCADE,
    FOREIGN KEY (source_entity_id) REFERENCES memory_entities(id)  ON DELETE CASCADE,
    FOREIGN KEY (target_entity_id) REFERENCES memory_entities(id)  ON DELETE CASCADE
);

CREATE INDEX idx_merge_proposals_pending
   ON memory_entity_merge_proposals(user_id, proposed_at)
   WHERE resolved_at IS NULL;
```

WebUI surfaces the pending count via the memory-icon dot indicator and a Suggested-Merges panel on the Graph tab. First click on the lit icon auto-routes to the Graph tab with a 600 ms accent-glow flash; subsequent clicks respect whichever tab was last active.

`idx_contacts_field_lvalue` (functional index on `lower(value)`, added in the same v43 post-migration block) backs the contacts self-join inside `compute_contact_field_overlap` so Phase 2's per-extraction scoring stays O(log N) per JOIN instead of O(N²).

### 6.5 Memory Relations Table

```sql
CREATE TABLE memory_relations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    subject_entity_id INTEGER NOT NULL,
    relation TEXT NOT NULL,             -- "lives_in", "owns", "works_on", etc.
    object_entity_id INTEGER,           -- FK to entity (NULL if literal value)
    object_value TEXT,                  -- Literal value (e.g., "golden retriever")
    fact_id INTEGER,                    -- Optional FK to originating fact
    confidence REAL DEFAULT 0.8,
    created_at INTEGER NOT NULL DEFAULT (strftime('%s','now')),
    valid_from INTEGER DEFAULT NULL,    -- v33 — bitemporal validity (NULL = open)
    valid_to   INTEGER DEFAULT NULL,
    source_conversation_id INTEGER DEFAULT NULL,  -- v42 — Phase B provenance
    source_msg_id_start    INTEGER DEFAULT NULL,
    source_msg_id_end      INTEGER DEFAULT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (subject_entity_id) REFERENCES memory_entities(id) ON DELETE CASCADE,
    FOREIGN KEY (object_entity_id) REFERENCES memory_entities(id) ON DELETE SET NULL,
    FOREIGN KEY (fact_id) REFERENCES memory_facts(id) ON DELETE SET NULL
);

CREATE INDEX idx_memory_relations_subject       ON memory_relations(subject_entity_id);
CREATE INDEX idx_memory_relations_object        ON memory_relations(object_entity_id);
CREATE INDEX idx_memory_relations_user          ON memory_relations(user_id);
CREATE INDEX idx_memory_relations_user_validity ON memory_relations(user_id, valid_to);
CREATE INDEX idx_memory_relations_subject_open  ON memory_relations(subject_entity_id, relation)
   WHERE valid_to IS NULL;
```

The partial `idx_memory_relations_subject_open` index keeps the auto-close path (`memory_db_relation_supersede` — see §11 / S8) cheap: only currently-open relations are indexed.

### 6.6 Conversations (anchor for cat-2 temporal extraction)

```sql
CREATE TABLE conversations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    title TEXT NOT NULL DEFAULT 'New Conversation',
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    message_count INTEGER DEFAULT 0,
    is_archived INTEGER DEFAULT 0,
    -- ... omitted: webui-only / compaction / per-conversation LLM config columns ...
    last_extracted_msg_count INTEGER DEFAULT 0,
    last_extracted_msg_id    INTEGER NOT NULL DEFAULT 0,  -- v41 — ID-based extraction cursor
    extraction_attempts          INTEGER DEFAULT 0,        -- v39 — recovery worker
    extraction_last_attempt_at   INTEGER DEFAULT 0,
    is_private INTEGER DEFAULT 0,
    anchor_date INTEGER NOT NULL DEFAULT 0,                -- v42 — Cat-2 Phase 1
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

`anchor_date` is the conversation's logical "now" — production writers populate it with `time(NULL)` at insert; the bench harness overrides it per LoCoMo session. The extraction prompt prepends `Conversation anchor: YYYY-MM-DD` so the LLM can resolve relative phrases ("yesterday", "last month") against it. See [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md).

### 6.7 Summary nodes (hierarchical compaction DAG)

```sql
CREATE TABLE summary_nodes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    conversation_id INTEGER NOT NULL,
    prior_node_id INTEGER,                          -- linked-list backbone
    depth INTEGER NOT NULL DEFAULT 0,
    msg_id_start INTEGER NOT NULL,
    msg_id_end   INTEGER NOT NULL,
    level INTEGER NOT NULL DEFAULT 0,               -- 0=raw window, >0=meta-summary
    summary_text TEXT NOT NULL,
    token_count INTEGER,
    created_at INTEGER,
    FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE,
    FOREIGN KEY (prior_node_id) REFERENCES summary_nodes(id) ON DELETE SET NULL
);

CREATE INDEX idx_summary_nodes_conv ON summary_nodes(conversation_id);
```

Backs the LCM Phase-4 lossless-pointer system: `[COMPACTED conv=N msgs=X-Y node=Z depth=D]` tags reference a `summary_nodes` row, and `context_expand` walks the chain to retrieve original messages by ID range.

### 6.8 System metadata

```sql
CREATE TABLE system_metadata (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
```

Daemon-wide key/value store. Used by the embedding-recompute worker to record `embedding_model_id` (current production embedder) — when the value changes, `users.embeddings_model_id` is compared per-user and out-of-date users are re-embedded in the background. See [EMBEDDING_UPGRADE.md](EMBEDDING_UPGRADE.md).

### 6.9 Users (memory-relevant columns)

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    is_admin INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,
    last_login INTEGER,
    failed_attempts INTEGER DEFAULT 0,
    lockout_until INTEGER DEFAULT 0,
    categories_backfilled_at INTEGER DEFAULT 0,    -- v34 — fact-category backfill marker
    embeddings_model_id      TEXT DEFAULT NULL,    -- v41 — last completed re-embed model
    real_name                TEXT DEFAULT NULL,    -- v44 — required for link-user-self
    preferred_address        TEXT DEFAULT NULL,    -- v44 — "They prefer to be addressed as ..."
    identity_aliases         TEXT DEFAULT NULL     -- v44 — newline-separated alternate names
);
```

Memory-related columns only. The admin/auth columns are documented under the auth subsystem.

**v44 user-identity columns** retire the persona-description borrow that conflated AI behavior with user identity. `real_name` is required by the entity-merge `link-user-self` synthetic-seed path (gate enforced in `memory_alias_link_user_self_run`). `preferred_address` and `identity_aliases` feed both the LLM system prompt's identity block (`build_identity_block` in `src/webui/webui_server.c`) and the synthetic-self resolver token set used by Phase 1.5 of the entity-merge cascade. `identity_aliases` is parsed at use-site (split on `\n`, strip whitespace, drop empties, dedupe case-insensitive). All three are nullable TEXT — existing rows migrate to NULL.

**v46 recompute trigger**: the v46 migration runs `UPDATE users SET embeddings_model_id = NULL` so every user is picked up as stale by the embedding-recompute worker on next boot, backfilling the v45-added `memory_summaries.embedding` column. Side effect: facts + entities also get re-embedded — wasted work in the strict sense, but bounded (the dev's ~2k facts + 300 entities + 270 summaries take ~10 s total against the local ONNX engine). The all-three-or-redo-all trade-off is documented inline in `src/memory/memory_embed_recompute.c`.

### 6.10 Document chunks (RAG)

```sql
CREATE TABLE documents (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,                    -- per-user storage; NULL allowed for is_global=1
    filename TEXT NOT NULL,
    filepath TEXT NOT NULL,
    filetype TEXT NOT NULL,             -- 'txt', 'md', 'pdf', 'docx'
    file_hash TEXT NOT NULL,            -- SHA256 for change detection
    num_chunks INTEGER NOT NULL,
    is_global INTEGER DEFAULT 0,        -- 1 = shared across users
    created_at INTEGER NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_documents_user ON documents(user_id);
CREATE INDEX idx_documents_hash ON documents(file_hash);

CREATE TABLE document_chunks (
    id INTEGER PRIMARY KEY,
    document_id INTEGER NOT NULL,
    chunk_index INTEGER NOT NULL,       -- Order within document
    text TEXT NOT NULL,
    embedding BLOB NOT NULL,
    embedding_norm REAL NOT NULL,
    created_at INTEGER NOT NULL DEFAULT 0,  -- v37 — backfilled from parent.created_at
    FOREIGN KEY (document_id) REFERENCES documents(id) ON DELETE CASCADE
);

CREATE INDEX idx_doc_chunks_doc ON document_chunks(document_id);
```

`document_chunks.created_at` powers the same temporal-query proximity scoring that facts use — see §7.2 / temporal-weight tuning. `is_global=1` documents are visible to every authenticated user; per-user uploads default to `is_global=0`.

### 6.11 Memory facts BM25 index (v48)

```sql
CREATE VIRTUAL TABLE memory_facts_fts USING fts5(
    fact_stems,                            -- pre-stemmed fact_text (Porter2)
    content='',                            -- contentless — app owns rows
    tokenize='unicode61 remove_diacritics 2'
);
```

`memory_facts_fts` is a contentless FTS5 virtual table backing the BM25 keyword path added in the Mem0 architectural parity work (May 2026). "Contentless" means SQLite stores only the inverted index — the application is responsible for keeping it in sync with `memory_facts` via explicit `INSERT INTO memory_facts_fts(rowid, fact_stems) VALUES (?, ?)` / `DELETE FROM memory_facts_fts WHERE rowid = ?` calls at fact create / supersede / delete time.

**Stemming pre-pass**: each `fact_text` is run through libstemmer (Porter2 English) in C before insert via `memory_stem_text()` in `src/memory/memory_stem.c`. Pre-stemming in C rather than via an FTS5 tokenizer extension keeps the index portable across SQLite builds and lets the same stem helper run on query text at search time. `unicode61` handles diacritic folding; `remove_diacritics 2` is the full NFD-aware variant.

**Backfill**: the v48 migration block walks every existing `memory_facts` row, stems each `fact_text`, and inserts into `memory_facts_fts`. A `v48_ok` guard tracks both `CREATE VIRTUAL TABLE` and backfill success — the version stamp only advances when both have completed, so a transient backfill failure doesn't leave the DB advertised as v48 with a partial index.

**Querying**: BM25 is the first-pass keyword scorer when `[memory] bm25_enabled = true`. Raw `bm25()` scores are normalized to `[0,1]` via a query-length-adaptive sigmoid in `memory_bm25.c` (midpoint + steepness selected from five tiers — short / medium / long / very-long / extreme — adapted from Mem0's `mem0/utils/scoring.py::get_bm25_params` under Apache-2.0; see `NOTICE`). The normalized score blends with the existing semantic-cosine path in the hybrid composite.

See §9.1.1 Search Strategy below for the path through the search dispatcher.

---

## 7. Configuration

### 7.1 dawn.toml Memory Section

```toml
[memory]
enabled = true
context_budget_tokens = 800
session_timeout_minutes = 15

# Extraction model (separate from conversation model for consistency)
[memory.extraction]
provider = "local"           # "local", "openai", "claude", "ollama"
model = "qwen2.5:7b"         # Model name for that provider

# Decay settings (applied by nightly maintenance job)
[memory.decay]
enabled = true                           # Enable nightly confidence decay
hour = 2                                 # Run at 2 AM (local time, 0-23)
inferred_weekly = 0.95                   # 5% decay per week for inferred facts
explicit_weekly = 0.98                   # 2% decay per week for explicit facts
preference_weekly = 0.97                 # 3% decay per week for preferences
inferred_floor = 0.0                     # Inferred facts can decay to zero
explicit_floor = 0.50                    # Explicit facts never go below this
preference_floor = 0.40                  # Preferences never go below this
prune_threshold = 0.25                   # Delete facts below this confidence
summary_retention_days = 30              # Delete summaries older than this
access_reinforcement_boost = 0.05        # Confidence boost when fact is accessed
```

### 7.2 dawn.toml Embeddings Section

```toml
[memory.embeddings]
provider = "ollama"                  # "ollama", "openai", "onnx"
model = "nomic-embed-text"           # Embedding model name
endpoint = "http://localhost:11434"  # Provider endpoint
dimensions = 768                     # Embedding vector dimensions
keyword_weight = 0.4                 # Hybrid search: keyword component (0.0-1.0)
semantic_weight = 0.6                # Hybrid search: semantic component (0.0-1.0)
```

**Provider Details:**

| Provider | Model                  | Dimensions | Speed  | Notes                               |
| -------- | ---------------------- | ---------- | ------ | ----------------------------------- |
| Ollama   | nomic-embed-text       | 768        | ~20ms  | Local, no API key needed            |
| OpenAI   | text-embedding-3-small | 1536       | ~100ms | Cloud, requires API key             |
| ONNX     | all-MiniLM-L6-v2 int8  | 384        | ~8ms   | Current default; local, shares ONNX Runtime with TTS |
| ONNX     | bge-small-en-v1.5      | 384        | ~35ms fp32 / ~10ms int8 (est.) | Recommended upgrade; +7.4pp LoCoMo retrieval quality |

**Hybrid Search:** Memory search combines keyword matching (tokenized LIKE queries) with semantic similarity (cosine distance on embeddings). The `keyword_weight` and `semantic_weight` control the blend. Results are deduplicated and sorted by combined score.

### 7.3 dawn.toml RAG Section

```toml
[rag]
enabled = true
documents_dir = "~/Documents/dawn-knowledge"
chunk_size_tokens = 256
chunk_overlap_tokens = 50
similarity_threshold = 0.5
max_results = 5
reindex_on_startup = false              # Check for changed files at startup
```

---

## 8. Extraction Process

### 8.1 Extraction Prompt

```
You are analyzing a conversation to extract memorable information.

CONVERSATION TRANSCRIPT:
{transcript}

EXISTING USER PROFILE:
{current_facts_and_preferences}

Extract the following, being CONSERVATIVE (only high-confidence items):

1. FACTS: Discrete pieces of information worth remembering.
   - USER FACTS: Personal info, relationships, work, health, interests
     Examples: "User is vegetarian", "User's daughter is named Dawn"
   - DECISIONS: Conclusions reached during conversation
     Examples: "Decided to use Zigbee for garage automation"
   - PLANS: Future intentions mentioned
     Examples: "Planning to visit mom in Florida next month"
   - Only extract if clearly stated or strongly implied
   - Format: Short declarative sentences (self-describing)
   - Mark as "explicit" if user directly stated, "inferred" if deduced

2. PREFERENCES: How the user likes to interact.
   - Categories: verbosity (concise/detailed), humor (enjoys/dislikes),
     formality (casual/professional), technical_level (beginner/expert)
   - Only extract if there's CLEAR evidence (user complained, corrected, praised)

3. CORRECTIONS: Did the user correct a previous assumption?
   - If yes, note what was wrong and what's correct

4. SUMMARY: 2-4 sentence summary including:
   - Main topics discussed
   - Key decisions made (if any)
   - Action items or next steps (if any)

5. TOPICS: List of main topics (max 5).

OUTPUT FORMAT (JSON):
{
  "facts": [
    {"text": "...", "source": "explicit|inferred", "confidence": 0.0-1.0}
  ],
  "preferences": [
    {"category": "...", "value": "...", "source": "explicit|inferred", "confidence": 0.0-1.0}
  ],
  "corrections": [
    {"old_fact": "...", "new_fact": "...", "reason": "..."}
  ],
  "entities": [
    {"name": "Jon", "type": "person"},
    {"name": "Buddy", "type": "pet"},
    {"name": "Atlanta", "type": "place"}
  ],
  "relations": [
    {"subject": "Jon", "relation": "lives_in", "object": "Atlanta"},
    {"subject": "Buddy", "relation": "is_a", "object": "golden retriever"}
  ],
  "summary": "...",
  "topics": ["...", "..."]
}

RULES:
- When in doubt, DON'T extract. False memories are worse than missing memories.
- Confidence 0.9+ only for explicit statements ("I am a vegetarian")
- Confidence 0.6-0.8 for strong implications (user repeatedly avoids meat options)
- Confidence <0.6: probably don't bother storing
- Never store: passwords, API keys, sensitive financial info, medical diagnoses
- Phrase facts to be self-describing (include context in the text itself)
```

### 8.2 Handling Extraction Failures

| Failure          | Handling                                                               |
| ---------------- | ---------------------------------------------------------------------- |
| Malformed JSON   | Log warning, mark session as "extraction_failed", retry in nightly job |
| Model timeout    | Retry with exponential backoff (max 3 attempts)                        |
| Empty extraction | Valid result - some sessions have no memorable content                 |

---

## 9. Memory Storage and Retrieval

### 9.0 Two Paths for Storing Facts

Facts enter the memory system through two distinct paths:

```
┌─────────────────────────────────────────────────────────────────────┐
│  PATH 1: Remember Tool (Real-Time, During Conversation)             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User: "Remember that I'm allergic to peanuts"                      │
│                         │                                            │
│                         ▼                                            │
│           Conversation LLM (has tool access)                        │
│                         │                                            │
│                         ▼                                            │
│           Tool call: {"action": "remember", "value": "..."}         │
│                         │                                            │
│                         ▼                                            │
│           Immediate storage: source="explicit", confidence=1.0      │
│                                                                      │
│  • User explicitly asks to remember something                        │
│  • Stored immediately during conversation                            │
│  • Conversation LLM has access to memory tool                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  PATH 2: Extraction Process (Batch, After Session)                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Session ends (disconnect/timeout)                                   │
│                         │                                            │
│                         ▼                                            │
│           Load conversation from messages table                      │
│                         │                                            │
│                         ▼                                            │
│           Extraction LLM (NO tool access, structured output)        │
│                         │                                            │
│                         ▼                                            │
│           Returns JSON: {"facts": [...], "summary": "..."}          │
│                         │                                            │
│                         ▼                                            │
│           Application parses JSON, validates, stores                 │
│                                                                      │
│  • Automatic extraction of facts user didn't explicitly mention     │
│  • Batch processing at session end (no latency during conversation) │
│  • Extraction LLM does NOT have tools - returns structured JSON     │
│  • Application controls storage (guardrails applied)                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Why two paths?**

- **Remember tool**: For explicit requests ("Remember that...") - immediate, high confidence
- **Extraction**: For implicit facts the user mentioned but didn't ask to remember - batch, inferred

### 9.1 Memory Tool

The conversation LLM can actively search and store memories via tool calls.

#### 9.1.1 Search Action

Handles questions like:

- "What did I tell you about my car?"
- "Do you remember my daughter's name?"
- "What did we talk about last Thursday?"
- "Where did Caroline work in 2020?"

**Tool Definition:**

```json
{ "device": "memory", "action": "search", "value": "daughter name" }
```

**With optional filters:**

```json
{ "device": "memory", "action": "search", "value": "garage", "time_range": "30d" }
```

```json
{ "device": "memory", "action": "search", "value": "diet",
  "category": "health", "with_source": "true" }
```

```json
{ "device": "memory", "action": "search", "value": "Caroline employer",
  "as_of": "2020-06-01", "include_historical": "true" }
```

**Parameters** (parsed in `src/memory/memory_callback.c:1044` `memoryCallback("search")` and forwarded to `memory_action_search`):

| Parameter | Type | Description |
|-----------|------|-------------|
| `value` | string | Keywords to search (small tokens, not full sentences). May be empty when paired with `time_range` for time-based discovery. |
| `time_range` | string | Window like `"24h"` / `"7d"` / `"1w"` / `"1y"`; restricts both fact `created_at` and summary recency. Parsed by `parse_time_period()` — units h/m/d/w/y, magnitude cap ~10 years. |
| `category` | enum | One of the 8 fact categories (see §6.1.1). Pre-filters fact set at SQL level via `memory_db_fact_search_by_category()`. Unknown labels are logged and ignored. |
| `as_of` | ISO-8601 date | Evaluates relation validity at a historical point. Accepts `YYYY-MM-DD` or bare `YYYY` (= Jan 1). Parsed by `strptime` + `timegm`. |
| `include_historical` | "true" / "false" | When `true`, the entity-recall path bypasses the open-relation filter and returns superseded relations too. Default `false`. |
| `with_source` | "true" / "false" | When `true`, retrieved items append a verbatim source excerpt drawn from the original conversation, deduped within the call and budget-bounded by `[memory] source_budget_chars` from `dawn.toml`. Default `false`. See [PROVENANCE.md](PROVENANCE.md). |

The LLM translates natural language to these parameters: "last Thursday" → `time_range`, "back when she worked at X" → `as_of`, "what did Caroline used to do" → `include_historical=true`. The legacy `date` / `date_from` / `date_to` parameters from earlier revisions of this doc were never wired to production — they did not survive the time_range / as_of consolidation.

**Design Principles:**

- **Unified search**: One tool searches facts, preferences, summaries, and the entity graph in a single call.
- **Keyword-light + structured filters**: `value` carries the lexical signal; `category` / `time_range` / `as_of` carry structure that hybrid scoring can't extract from natural language reliably.
- **Bitemporal correctness**: `as_of` + `include_historical` let the LLM ask "who did X work for in 2020" without contaminating present-tense queries.
- **Provenance optional, never default**: `with_source` is opt-in because verbatim excerpts are token-expensive; the response without `with_source` is what the original LongMemEval / LoCoMo runs measure.
- **Strbuf-based output**: The action assembles the response in a growable `strbuf_t` (`include/core/strbuf.h`) — no fixed 8 KB ceiling, sticky-OOM, max 256 KiB. Earlier fixed-buffer versions silently truncated when `with_source` budgets exceeded ~3 KB; see [PROVENANCE.md](PROVENANCE.md) §"Strbuf truncation fix."

**Search Strategy** (current implementation, after v47 Phase 0 + v48 BM25):

1. If `category` is set, pre-filter the candidate fact ID set via SQL `WHERE category = ?` index.
2. **Entity-graph grounding (Phase 1A, default-on at v48)**: `memory_graph_retrieval.c` parses proper nouns from the query, resolves them against `memory_entities`, walks `memory_relations` 1-hop, and adds an `entity_grounding_bonus` (default 0.40) to the composite for any fact whose `subject_entity_id` (v47) or relation-graph neighborhood matches a resolved entity. Validated +13.4pp internal `recall_reach` on LoCoMo when shipped (2026-05-13, Phase 0 + Phase 1A); see `[memory.graph_retrieval]` block in `dawn.toml.example`.
3. **Keyword channel**: when `[memory] bm25_enabled = true`, score keyword overlap via the FTS5 `memory_facts_fts` index (Porter2 stems, sigmoid-normalized — see §6.11). When `false` (default in v1), fall back to the legacy SQL `LIKE` + multi-token match-count path. Same fact ID space; the dispatcher decides which scorer runs.
4. **Semantic channel**: cosine similarity against fact embeddings (`bge-small-en-v1.5-int8` — see §3.5 / [EMBEDDING_UPGRADE.md](EMBEDDING_UPGRADE.md)).
5. **Composite + adjustments**: weighted blend of (2) + (3) + (4) plus temporal-query proximity (when the query contains a temporal expression — `src/core/time_query_parser.c`) and the proper-noun boost. The `search_score_floor` knob drops marginal-cosine hits below the floor before they reach the LLM.
6. Apply `time_range` window if set (also applied to summaries).
7. For each top-K matching fact, look up entity links and currently-valid relations (or historical, if `include_historical`).
8. If `with_source`, batch-fetch provenance via the four sibling readers in `memory_db_provenance.h`: `memory_db_facts_get_sources`, `memory_db_relations_get_sources`, `memory_db_summaries_get_sources`, `memory_db_prefs_get_sources`. Each returns parallel `conv_id` / `msg_id_start` / `msg_id_end` arrays for any positive N (auto-chunked at 32 IDs per SQL pass). Source rendering selects the top-K most-relevant messages from the range via case-insensitive (query ∪ fact_text) overlap, re-sorted into original dialog order, header-suppressed if zero messages match — see `memory_callback.c::append_source_excerpt_from_range`. Verbatim excerpts append up to the per-call `source_budget_chars` limit.

The BM25 path is opt-in in v1 because it shipped as a documented experiment ahead of full end-to-end bench-validation under the leader-comparable protocol; expected to default-on once Phase 1 of the Mem0 parity plan is closed. The Phase 1A entity-graph grounding is default-on — see [STATE.md](STATE.md) for the current leader-comparable baseline and the headline-number policy.

**Search Response Format** (text, not JSON — the LLM reads it directly):

```
FACTS:
- [ID:42] User's daughter is named Dawn (explicit, 2026-01-10)
- [ID:118] Decided to use Zigbee for garage automation (inferred, 2026-01-16)
  ┌── source [conv 7, msgs 14-16] ────────────────────────────
  │ user: "let's go with zigbee for the garage"
  │ assistant: "noted — switching the project to zigbee"
  └────────────────────────────────────────────────────────────

ENTITIES:
- Dawn (person) — daughter_of Jon
- Garage (place) — has_protocol zigbee (since 2026-01-16)

RECENT CONVERSATIONS:
- [3 hours ago] Discussed garage automation...
  Topics: home automation, zigbee
```

The text format is canonical because the model is reading it as system input — JSON would force a parse step on the model side without buying anything. Full grammar in `memory_action_search()` and `memory_action_recent()`.

#### 9.1.2 Remember Action

Handles immediate storage when the user shares a fact or asks DAWN to remember something.

**Tool Definition:**

```json
{ "device": "memory", "action": "remember", "value": "User is vegetarian" }
```

**Design Principles:**

- **Immediate storage**: Fact available in same session (no waiting for extraction)
- **LLM decides**: LLM determines what's worth remembering and how to phrase it
- **Self-describing text**: Fact text should include context ("User is vegetarian" not just "vegetarian")
- **Complements extraction**: Extraction still runs at session-end to catch missed facts
- **Natural response**: LLM confirms storage ("I'll remember that")

**Remember Implementation:**

```c
// Store a fact immediately from tool call
int memory_remember(const char *user_id, const char *fact_text);
```

**Storage Behavior:**

- `source` set to `"explicit"` (user directly stated)
- `confidence` set to `1.0` (explicit facts start at full confidence)
- Duplicate detection: If similar fact exists, update confidence rather than create duplicate
- Guardrails applied: Pattern filter rejects instruction-like content

**Remember Response Format:**

```json
{ "status": "stored", "fact": "User is vegetarian" }
```

#### 9.1.3 Recent Action

Handles time-based queries when the user wants to see what's been learned recently — or earliest, or within a specific past window — without specific keywords.

**Tool Definition:**

```json
{ "device": "memory", "action": "recent", "value": "24h" }
```

**Bundle 3 parameters (2026-05-13, commit `32861d5`)** turn `recent` into a fully bounded retrieval surface — newest-or-oldest within an arbitrary `[now − query, now − before]` window:

```json
{ "device": "memory", "action": "recent",
  "value": "1y", "sort": "oldest", "limit": 1 }
```

```json
{ "device": "memory", "action": "recent",
  "value": "30d", "before": "7d", "limit": 20 }
```

| Parameter | Type / range | Description |
|-----------|-------------|-------------|
| `value` (= `query`) | time period string, default `"7d"` | Lower bound of the window — how far back the query reaches. Examples: `"24h"`, `"7d"`, `"2w"`, `"30d"`, `"1y"`, `"10y"`. Upper bound is `"10y"` (the parser's overflow cap). |
| `limit` | int, 1-50, default 20 facts / 10 summaries | Result count per category — facts and summaries each capped to this. |
| `sort` | `"newest"` (default) / `"oldest"` | Ordering within the window. `"oldest"` is what makes "find my earliest memory" answerable (every pre-Bundle-3 SELECT was DESC-only). |
| `before` | time period string, default unset | Upper bound — restrict to entries older than `before` ago. Combine with `query` to bracket a window in the past. |
| `category` | enum | Same 8-label taxonomy as `search` — pre-filters facts at SQL level. |

**Supported time-period units** (`parse_time_period()` in `include/tools/time_utils.h`):

| Unit | Suffix | Multiplier |
|------|--------|-----------|
| Minutes | `m` / `M` | 60 |
| Hours | `h` / `H` / (no suffix) | 3600 |
| Days | `d` / `D` | 86400 |
| Weeks | `w` / `W` | 604800 |
| Years | `y` / `Y` | 31536000 (365 days, not leap-aware — fine for approximate windowing) |

The parser rejects negative values and caps the numeric magnitude at ~10 years to prevent overflow. The `'y'` unit was added as a Bundle 3 fold-in on 2026-05-13 — pre-Bundle-3, `time_range:"year"` silently fell through to "no filter."

**Design Principles:**

- **`query` is a window bound, not a count**: encodes the principle "how far back do we reach"; `limit` controls how many results come back. The pre-Bundle-3 descriptor confused these and the LLM passed integers as periods.
- **Sort applies WITHIN the window**: `sort=oldest` returns the earliest entries reachable from `[now − query, now]` — not the earliest in the entire store. To reach the absolute earliest, widen `query` (its upper bound is `"10y"`).
- **Two-bound windowing**: `query` + `before` together bracket `[now − query, now − before]`, enabling "what happened 3-7 days ago" style queries without keyword overlap.
- **Search remains hybrid**: the new params are explicitly recent-only; `search` keeps its own hybrid-scoring top-N pattern (the descriptors call this out).

**Implementation:**

```c
// New public DB helpers (Bundle 3) — until_ts=0 sentinel resolves to INT64_MAX internally.
int memory_db_fact_list_window(int user_id, time_t since_ts, time_t until_ts,
                               bool sort_asc, int limit,
                               memory_fact_t *facts_out, int *count_out);
int memory_db_summary_list_window(int user_id, time_t since_ts, time_t until_ts,
                                  bool sort_asc, int limit,
                                  memory_summary_t *summaries_out, int *count_out);
```

Backed by four prepared statements (ASC / DESC × fact / summary). The legacy `stmt_memory_fact_list_since` + `stmt_memory_summary_list_since` are now subsumed by `_list_window` with `until_ts=0` — flagged in TODO for retirement.

**Recent Response Format:**

```
RECENT FACTS (oldest first, 1 result within 1y):
- [ID:5004] Initial workspace setup notes from token-streaming work (inferred, 15 weeks ago)

RECENT SUMMARIES:
- [3 hours ago] Discussed memory system implementation...
  Topics: memory, dawn, sqlite

Total: 1 fact, 1 summary
```

#### 9.1.4 Forget Action

Deletes a specific fact by numeric ID. Requires exact ID match to prevent accidental bulk deletion.

**Tool Definition:**

```json
{ "device": "memory", "action": "forget", "value": "42" }
```

**Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `value` | Yes | Numeric fact ID (from search results) |

**Design Principles:**

- **Exact ID match**: Prevents fuzzy string matching from deleting unintended facts
- **Numeric only**: Rejects non-numeric input to avoid ambiguity
- **Ownership check**: Only deletes facts belonging to the requesting user

#### 9.1.5 Contact Management Actions

Four actions for managing contact information linked to memory entities.

**Save Contact:**

```json
{ "device": "memory", "action": "save_contact", "value": "{\"entity_name\":\"Mom\",\"field_type\":\"phone\",\"value\":\"+1-555-0123\",\"label\":\"mobile\"}" }
```

**Find Contact:**

```json
{ "device": "memory", "action": "find_contact", "value": "{\"name\":\"Mom\",\"field_type\":\"phone\"}" }
```

**List Contacts:**

```json
{ "device": "memory", "action": "list_contacts", "value": "{\"field_type\":\"email\"}" }
```

**Delete Contact:**

```json
{ "device": "memory", "action": "delete_contact", "value": "7" }
```

**Parameters:**
| Action | Parameters | Description |
|--------|-----------|-------------|
| `save_contact` | entity_name, field_type, value, label | Store email/phone/address for an entity |
| `find_contact` | name, field_type (optional) | Lookup contact by entity name |
| `list_contacts` | field_type (optional) | List all contacts, optionally filtered |
| `delete_contact` | contact_id (numeric) | Remove a contact by ID |

Contacts are linked to memory entities — saving a contact for "Mom" automatically associates it with the "Mom" entity in the knowledge graph.

#### 9.1.6 Merge Entities Action

Soft-links two entities that refer to the same person / pet / place / thing. The link is **reversible** — both rows remain in `memory_entities`; the source row's `canonical_id` gets set to the target row's id. The original 6.7 hard-merge (relations reassigned, source row deleted) is preserved as the `dawn-admin memory entity consolidate` admin path for when the operator wants to harden the link.

**Tool Definition:**

```json
{ "device": "memory", "action": "merge_entities",
  "value": "Jon", "target_name": "Jonathan Smith" }
```

**Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `value` (source name) | Yes | Entity name that becomes a soft alias. The row is preserved. |
| `target_name` | Yes | Entity that stays canonical. Read paths via `COALESCE(canonical_id, id)` resolve here. |

**Soft-link behavior** (`memory_db_entity_alias_link()` in `src/memory/memory_db_alias.c`):

1. Resolve both names to entity IDs via canonical-name lookup.
2. Verify the single-level alias invariant: refuse if the source row already has dependents (other rows referencing it as `canonical_id`).
3. Set `source.canonical_id = target.id`. The source row stays — `mention_count`, `first_seen`, `last_seen` etc. are still readable, and equivalence-class aggregation surfaces totals across the canonical + all aliases (Bundle 2, commit `0e9ae1e`).
4. Write an audit row to `memory_entity_aliases` (`link_kind="soft"`, `reason="operator_webui"` / `"auto_merge_phase2"` / etc., plus the `composite_score` and `evidence_json` from the cascade if applicable).
5. Atomically invalidate the in-process entity-embedding cache so the next hybrid-search pass sees the new alias chain.

**Read-path semantics**: Searches return the canonical's display fields for both rows in the equivalence class. Bundle 2 added correlated subqueries on `memory_db_entity_get_by_name` + `memory_db_entity_search` + the admin canonical-list so `mention_count` / `first_seen` / `last_seen` are summed / minned / maxed across `WHERE id = e.id OR canonical_id = e.id` — single-level alias bound, EXPLAIN QUERY PLAN verified sub-ms at current scale.

**Reversibility**: `memory_db_entity_alias_unlink()` (admin only) clears `canonical_id` back to NULL and stamps `unlinked_at` on the audit row. Used when an operator-approved merge turns out to be wrong; the entity goes back to standing alone.

**Phase 2 auto-merge gate** (May 2026): during extraction, the cascade scores every fresh canonical against existing entities. Composite ≥ `auto_threshold` (default 0.90) triggers a silent `alias_link`; composite ∈ [`review_threshold`, `auto_threshold`) (default 0.50-0.90) lands in `memory_entity_merge_proposals` for operator review. Approving a proposal calls the same `alias_link` path. See §11 / Phase 6.7+.

#### 9.1.7 Entity Graph Context

Entity graph context is automatically appended to memory search results — it is not a separate tool action. When a search returns facts, the system also queries for related entities and appends an ENTITIES section showing entity names, types, and their relationships. This provides the LLM with graph context without requiring a separate tool call.

#### 9.1.8 Tool Descriptor (canonical source)

The pre-Bundle-3 prompt-addition block has been retired — DAWN now ships native function-calling schemas via the tool registry, and the canonical descriptor lives in `src/tools/memory_tool.c` (`memory_params[]`). The registry generates per-provider schemas (OpenAI tool_calls, Claude tool_use, native local-LLM formats) from a single struct. The schema includes the 11 params documented in §9.1.1-§9.1.6 plus the action enum. The legacy `<command>` tag fallback is still wired through `text_to_command_nuevo.c` for local models that don't speak any function-calling dialect — it parses the same JSON shape.

The descriptor itself encodes generic *principles* (e.g., "`query` is a window bound, sort orders within the window") rather than baked-in recipes — refined post-Bundle-3 after live testing surfaced the LLM picking wrong windows when the descriptor showed only example values.

#### 9.1.9 When to Use Each

| User Says                                | LLM Action                                               |
| ---------------------------------------- | -------------------------------------------------------- |
| "I'm vegetarian"                         | Call `remember` with "User is vegetarian"                |
| "Remember that I hate cilantro"          | Call `remember` with "User hates cilantro"               |
| "What do you know about me?"             | Call `search` with broad keywords or list injected facts |
| "Do you remember my daughter's name?"    | Call `search` with "daughter name"                       |
| "What did we talk about last Thursday?"  | Call `search` with the phrase in `value` (the `time_query_parser` handles "last Thursday"), or fall through to `recent` with `query="7d"` if no keywords |
| "What did we decide about the garage?"   | Call `search` with "garage decide"                       |
| "What have you learned about me lately?" | Call `recent` with "7d" or "1w"                          |
| "What's new in the past 24 hours?"       | Call `recent` with "24h"                                 |
| "What were we working on last week?"     | Call `recent` with "1w"                                  |
| "Catch me up on the past few days"       | Call `recent` with "3d"                                  |

#### 9.1.10 Three-Tier Retrieval

| Tier                | Mechanism         | When                          | Budget                                  |
| ------------------- | ----------------- | ----------------------------- | --------------------------------------- |
| **Always loaded**   | Passive injection | Session start                 | ~800 tokens (400 facts + 400 summaries) |
| **Memory search**   | Tool call         | LLM needs to recall something | ~500 tokens per search                  |
| **Document search** | Tool call (RAG)   | LLM needs info from files     | ~500 tokens per search                  |

#### 9.1.11 Active Store vs Session-End Extraction

Both mechanisms work together:

| Mechanism                  | What It Catches                                | When Stored   |
| -------------------------- | ---------------------------------------------- | ------------- |
| **Active store (tool)**    | Explicit statements, "remember that..."        | Immediately   |
| **Session-end extraction** | Inferred facts, things LLM missed, preferences | After session |

The extraction job can also detect if a fact was already stored via tool call (duplicate detection) and skip it or merge confidence scores.

### 9.2 Session Start Loading

```
┌─────────────────────────────────────────────────────────────────────┐
│  SYSTEM PROMPT AUGMENTATION (~800 tokens budget)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Base AI_DESCRIPTION (personality, capabilities)                     │
│  +                                                                   │
│  User Preferences Section (~100 tokens):                             │
│  "This user prefers concise responses and enjoys dry humor."        │
│  +                                                                   │
│  Key Facts Section (~300 tokens, top N by confidence/recency):       │
│  "Known facts about this user:                                       │
│   - Vegetarian (explicit, high confidence)                           │
│   - Works as a software developer                                    │
│   - Has two children: Dawn and Sam                                   │
│   - Lives in Atlanta area                                            │
│   - Timezone: US Eastern"                                            │
│  +                                                                   │
│  Recent Context Section (~400 tokens):                               │
│  "Recent conversations:                                              │
│   - Yesterday: Discussed home automation for garage lights           │
│   - Last week: Helped debug MQTT connection issues                   │
│   - 3 days ago: Reviewed Python script for data analysis"           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 RAG Retrieval (On-Demand)

RAG retrieval happens **during conversation** when the user's query might benefit from document search.

**Trigger heuristics:**

- User asks a question (detected by "?" or question words)
- Query contains keywords suggesting lookup ("what does", "how do I", "according to")
- No confident answer from LLM's base knowledge

**Integration point:** Before LLM call, embed user message, search RAG, prepend results to context.

---

## 10. Confidence Decay

### 10.1 Decay Algorithm

Decay uses a custom SQLite `powf()` function registered at DB init, enabling atomic
UPDATE statements — no C-side row iteration needed. This eliminates TOCTOU races and
minimizes mutex hold time.

```sql
-- Fact decay (single atomic UPDATE per user)
UPDATE memory_facts SET confidence =
  CASE WHEN source = 'explicit'
    THEN MAX(:explicit_floor, confidence * powf(:explicit_rate,
         (CAST(strftime('%s','now') AS REAL) - last_accessed) / 604800.0))
    ELSE MAX(:inferred_floor, confidence * powf(:inferred_rate,
         (CAST(strftime('%s','now') AS REAL) - last_accessed) / 604800.0))
  END
WHERE user_id = :uid AND superseded_by IS NULL AND last_accessed IS NOT NULL

-- Preference decay (same pattern)
UPDATE memory_preferences SET confidence =
  MAX(:pref_floor, confidence * powf(:pref_rate,
      (CAST(strftime('%s','now') AS REAL) - updated_at) / 604800.0))
WHERE user_id = :uid
```

**Key behaviors:**

- Facts with NULL `last_accessed` are skipped (never been loaded into context)
- Decay is proportional to time since last access, not fixed per-day
- Explicit facts have a configurable floor (default 0.50) — never fully forgotten
- Inferred facts have a separate configurable floor (default 0.0 — can decay to zero)
- Preferences have their own floor (default 0.40)

**Pruning** runs after decay:

- Facts below `prune_threshold` (default 0.25) are logged for audit trail, then deleted
- Audit log and delete are wrapped in a transaction for consistency
- Old superseded facts are pruned per `prune_superseded_days`
- Summaries older than `summary_retention_days` are deleted

### 10.2 Reinforcement

When a fact is loaded into context (via `memory_build_context()`), its access metadata
is updated. Reinforcement is **time-gated** to prevent confidence pinning:

```sql
UPDATE memory_facts SET
  last_accessed = ?,
  access_count = access_count + 1,
  confidence = CASE
    WHEN (CAST(strftime('%s','now') AS REAL) - last_accessed) > 3600
    THEN MIN(1.0, confidence + :boost)
    ELSE confidence
  END
WHERE id = ? AND user_id = ?
```

The 1-hour cooldown ensures that multiple accesses within the same conversation
don't stack reinforcement. The `user_id` check enforces ownership isolation.

### 10.3 Orchestration

Decay runs from `memory_run_nightly_decay()` in `src/memory/memory_maintenance.c`,
called by the auth maintenance thread (15-minute cycle). Guards:

1. `g_config.memory.enabled && g_config.memory.decay_enabled` must both be true
2. Current local hour must match `g_config.memory.decay_hour`
3. At least 20 hours must have passed since last run (prevents double-execution)

Per-user processing order: decay facts → decay preferences → prune low-confidence →
prune superseded → prune old summaries → `usleep(1000)` courtesy yield.

Aggregate totals are logged; no per-user logging (avoids user enumeration in logs).

---

## 11. Implementation Phases

Memory and RAG are independent features. Memory can be fully implemented and deployed without RAG.

---

### MEMORY SYSTEM

### Phase 1: Memory Storage Foundation ✅ COMPLETE

- [x] Add SQLite tables: memory_facts, memory_preferences, memory_summaries
- [x] Create migration system (schema v14-v16)
- [x] Basic CRUD operations in C (`memory_db.c`)
- [x] Prepared statements for all operations

### Phase 2: Memory Tool ✅ COMPLETE

- [x] Memory tool with four actions:
   - [x] `search` - keyword search across facts/preferences/summaries
   - [x] `recent` - time-based retrieval (e.g., "24h", "7d", "1w")
   - [x] `remember` - immediate fact storage from LLM
   - [x] `forget` - delete matching facts
- [x] Add `memory` device to commands_config_nuevo.json
- [x] Register callback in mosquitto_comms.c (MEMORY device type)
- [x] Duplicate detection via similarity matching
- [x] Guardrails: blocked patterns prevent instruction injection

### Phase 3: Context Injection ✅ COMPLETE

- [x] Load facts and preferences at session start
- [x] Augment system prompt with user context (`memory_build_context()`)
- [x] Context budget management (~800 tokens)
- [x] Format: preferences, facts by confidence, recent summaries

### Phase 4: Automated Extraction ✅ COMPLETE

- [x] Session-end consolidation trigger (WebSocket disconnect/timeout)
- [x] Extraction prompt with JSON output format
- [x] JSON parsing and validation
- [x] Fact/preference/summary storage
- [x] Threaded extraction (non-blocking)
- [x] Privacy toggle: conversations marked private skip extraction

### Phase 4.5: Privacy Toggle ✅ COMPLETE (Bonus Feature)

- [x] Per-conversation `is_private` flag in database
- [x] WebSocket handler (`set_private` / `set_private_response`)
- [x] Frontend toggle in LLM controls bar (eye/eye-off icon)
- [x] Keyboard shortcut: Ctrl+Shift+P
- [x] Can set before conversation starts (pending state applied on creation)
- [x] Privacy badge in conversation history list

### Phase 5: Decay and Maintenance ✅ COMPLETE

- [x] Nightly decay job (configurable hour, local time)
- [x] Atomic SQL decay via custom SQLite `powf()` function
- [x] Confidence reinforcement on access (time-gated, 1-hour cooldown)
- [x] Configurable floors for explicit facts, inferred facts, and preferences
- [x] Pruning low-confidence items with audit trail logging
- [x] Summary retention management (configurable days)
- [x] Superseded fact pruning
- [x] Double-execution prevention (20-hour gap guard)
- [x] User isolation fix (`AND user_id = ?` on access update)
- [x] WebUI settings for all decay parameters
- [x] NaN/Inf guard on `powf()` custom function
- [ ] Crash recovery for unconsolidated sessions (deferred)

### Phase 6: Memory WebUI ✅ COMPLETE

- [x] Memory viewer panel in WebUI (expandable section, not settings)
- [x] Four tabs: Facts, Preferences, Summaries, Graph (entities)
- [x] Delete individual memories (per-item delete button with confirmation)
- [x] Memory statistics display (facts, preferences, summaries, entities counts)
- [x] "Forget Everything" option with typed DELETE confirmation modal
- [x] Search across all memory types
- [x] Graph tab: entity cards with type badges, expandable relations (→/← arrows)
- [x] Keyboard accessibility (tabindex, ARIA roles, keydown handlers)
- [x] WebUI endpoints: `/api/memory/{facts,preferences,summaries,entities,stats}`

### Phase 6.5: Import / Export ✅ COMPLETE

- [x] Export memories as JSON (facts, preferences, entities, relations with metadata)
- [x] Export memories as plain text (human-readable, portable to other AIs)
- [x] Import from DAWN JSON (lossless restore from export)
- [x] Import from ChatGPT / Claude (plain text, one fact per line)
- [x] Auto-detection: JSON vs plain text based on input content
- [x] Two-phase workflow: preview (dry-run) → confirm → commit
- [x] Duplicate detection: FNV-1a hash for exact matches, Jaccard similarity (0.7 threshold) as fallback
- [x] Limits: 200 lines per import, 256KB max paste, 500 facts / 200 preferences / 200 entities export caps
- [x] WebUI modal with paste, file upload, and help tabs
- [x] WebSocket messages: `export_memories` / `import_memories` with response types

**Import flow:**

```
Paste text or upload file
         │
         ▼
Auto-detect format (JSON vs plain text)
         │
         ▼
Preview (commit=false) → show new items, duplicates skipped
         │
         ▼
User confirms → Commit (commit=true) → write to DB
```

**Duplicate detection:**

1. Normalize text → FNV-1a hash → fast O(1) lookup against existing facts
2. If no hash match → Jaccard similarity on word sets (threshold 0.7)
3. Duplicates are counted and reported but silently skipped

**Files:** `src/webui/webui_memory.c` (handlers), `include/memory/memory_similarity.h` (dedup), `www/js/ui/memory.js` (UI)

### Phase 6.6: Contacts WebUI ✅ COMPLETE

- [x] 5th "Contacts" tab in Memory popover with contact count in stats bar
- [x] WebSocket CRUD: `contacts_list`, `contacts_add`, `contacts_update`, `contacts_delete`, `contacts_search`
- [x] Contact card rendering: field_type badge, value, label badge, entity name, hover-reveal edit/delete
- [x] Add/edit modal: entity name with typeahead, field_type select, value input (adapts type), label select
- [x] Entity cross-link from Graph tab: person entities show contact count badge, clicking navigates to filtered Contacts tab
- [x] Server-side search via `contacts_find()` (case-insensitive LIKE with escaping)
- [x] Pagination with Load More (PAGE_SIZE=20)
- [x] Mobile-responsive layout with touch targets

**Files:** `src/webui/webui_contacts.c` (handlers), `www/js/ui/contacts.js` (UI), `www/css/components/contacts.css` (styles)

### Phase 6.7: Entity Merge (hard) ✅ COMPLETE

- [x] `memory_db_entity_merge()` — transactional SQL (BEGIN IMMEDIATE → COMMIT/ROLLBACK)
- [x] MERGE_EXEC macro for error-checked ad-hoc prepared statements with goto-based rollback
- [x] Reassign all relations (subject + object) and contacts from source to target entity
- [x] Delete self-referencing relations after reassignment
- [x] Deduplicate relations via ROW_NUMBER() window function (PARTITION BY subject, relation, object, COALESCE(object_value, ''))
- [x] Deduplicate contacts via ROW_NUMBER() (PARTITION BY entity_id, field_type, value)
- [x] Absorb mention count and time range from source entity
- [x] Delete source entity (hard merge, not alias)
- [x] LLM tool: `merge_entities` action on memory tool (resolves by canonical name)
- [x] WebUI: two-click merge in Graph tab (click merge button on source → entity highlights → click target → confirm)
- [x] WebSocket handler with source_id == target_id validation

**Files:** `src/memory/memory_db.c` (merge logic), `src/memory/memory_callback.c` (tool action), `src/webui/webui_memory.c` (WS handler), `www/js/ui/memory.js` (UI)

**Superseded by Phase 6.8+ (May 2026) for the LLM-facing surface** — the `merge_entities` tool action now soft-links via `memory_db_entity_alias_link()`; the original hard-merge logic remains available as `dawn-admin memory entity consolidate` for operators who want to harden a soft link permanently.

### Phase 6.8: Entity Merge — soft aliases + Phase 2 auto-merge (May 2026) ✅ COMPLETE

Equivalence-class soft-merge surface that resolves the user-identity duplicate problem (e.g., "Jon" / "Jonathan Smith" / "User" arriving as three separate entity rows from re-extraction). Schema v43 adds `memory_entities.canonical_id` self-FK + `is_user_self` flag + `memory_entity_aliases` audit log + `memory_entity_merge_proposals` review-band staging + 5 partial indexes including the `idx_contacts_field_lvalue` functional index backing the cascade's contact-overlap scoring.

- [x] **Phase 1 — resolver cascade + operator surface** (commits `9bfda37` + family): new module `src/memory/memory_db_alias.c`; six-stage cascade (exact-canonical → token-Jaccard candidates → type filter with `thing` carve-out → embedding cosine → exclusive-relation / contact overlap → composite-band routing); composite scorer (0.30 name_jaccard + 0.30 embedding_cosine + 0.25 exclusive_relation_overlap + 0.10 contact_field_overlap + 0.05 type_match, +0.10 substring bonus, +0.20 `user_self` bonus, type-veto); soft-link write paths (`alias_link` / `alias_unlink`) with atomic entity-cache invalidation; equivalence-class relation listing via `IN (SELECT id WHERE id=? UNION ALL SELECT id WHERE canonical_id=?)`; single-SQL `exclusive_relation_overlap` with `ORDER BY valid_from DESC`; `canonical_priority` lexicographic comparator with sibling variant taking explicit `is_user_self` flags. Six new admin commands: `dawn-admin memory entity {merge,split,aliases,history,list,link-user-self}`. WebUI Graph-tab Suggested-Merges panel + manual-alias panel + 5 new WebSocket message handlers.
- [x] **Phase 1.5 — user-identity columns + synthetic-self scoring** (schema v44): `users.real_name` (required), `users.preferred_address`, `users.identity_aliases` (newline-separated); `auth_user_identity_t` + `auth_db_get/set_user_identity()` helpers; system-prompt identity block via `build_identity_block` in `src/webui/webui_server.c`; directional Jaccard for short-name candidates against verbose synthetic; `user`-token allow-list reaching Stage 6 unconditionally; promote-existing-match path replacing always-INSERT seed; commit-mode focused scoring against the user-self entity specifically.
- [x] **Phase 2 — auto-fire-on-extraction + WebUI dot** (commit `9bfda37`): gate-timing fix moves the resolver from inline entity-upsert to a sweep after the relations loop (so `exclusive_relation_overlap` actually sees the relations); Stage 2 substring rescue gained reverse direction via SQL `instr()`; propose-all-in-band (each scored candidate ≥ `review_threshold` gets its own proposal row instead of winner-only); auto-promote `user_self` at extraction (`memory_db_entity_maybe_auto_promote_user_self()`) + lazy sweep on Settings save (`memory_db_entity_auto_promote_user_self_by_real_name()`); runtime config `[memory.entity_merge] enabled / auto_threshold (0.90) / review_threshold (0.50)`. CONFIG_CLAMP NaN/inf hardening as a codebase-wide side effect. WebUI memory-icon dot indicator (mirrors music-playing dot, pulses 0.5s out of phase, `prefers-reduced-motion` carve-out) + auto-route to Graph tab on first-open-while-pending.
- [x] **Longer-canonical follow-up** (commit `ed0067c`, 2026-05-12): when the inbound entity has more name tokens than the cascade winner AND both are person/pet/place type, AUTO swaps direction so the longer form becomes canonical and the shorter form becomes the alias (e.g. a fuller "Jonathan Smith" arriving second after "Jon" is already canonical). Five new helper predicates gate the swap (preserves single-level alias invariant via `entity_has_canonical_dependents` check). Applies in both `consider_auto_merge` and the propose-all-in-band loop. REVIEW-band direction preference is filed as a follow-up TODO.
- [x] **Bundle 2 — equivalence-class read-path aggregation** (commit `0e9ae1e`, 2026-05-13): three SELECTs (`stmt_memory_entity_get_by_name`, `stmt_memory_entity_search`, admin canonical-list) gain correlated subqueries summing `mention_count` / minning `first_seen` / maxing `last_seen` across `WHERE id = e.id OR canonical_id = e.id`. Surfaces the equivalence-class totals on canonical rows so soft-aliasing a high-mention row under a low-mention canonical (e.g. "Jon"(88) → "Jonathan Smith"(10)) renders as the full class total (98) instead of just the canonical's own row. ORDER BY asymmetry is deliberate: admin list sorts by class total (operator source-of-truth), `entity_search` sorts by own `mention_count` (indexable hot path). Same commit bumped `FACT_MAP_MAX` 32 → 128.

**Files:** `src/memory/memory_db_alias.c` (resolver + scoring + write paths), `src/memory/memory_extraction.c` (gate-timing sweep + auto-promote hooks), `src/memory/memory_callback.c` (`merge_entities` rewrite to alias_link), `src/auth/auth_db_settings.c` (identity helpers), `src/webui/webui_server.c` (`build_identity_block` identity block injection), `src/auth/admin_socket.c` (six admin opcodes), `www/js/ui/memory.js` + `memory_aliases.js` (UI), schema v43-v44 in `src/auth/auth_db_core.c`.

### Phase 6.9: Crash-recovery worker (April 2026) ✅ COMPLETE

Picks up conversations whose `last_extracted_msg_count < message_count` after a daemon crash mid-extraction or an extraction failure.

- [x] New `src/memory/memory_recovery.c/h` — dedicated background thread (`nice 10`), immediate startup pass + recurring rescan every `[memory.recovery] recurring_interval_seconds` (default 24 h)
- [x] Schema v39 adds `conversations.extraction_attempts` + `extraction_last_attempt_at`
- [x] Auto-reset of cap counter when conversation gets new activity (via `extraction_last_attempt_at < updated_at` clause)
- [x] Successful extraction clears counters atomically in the existing `memory_db_set_last_extracted` UPDATE
- [x] Image-strip via `strip_image_markers()` (replaces `[IMAGE:data:image/...]` blocks with `[image]`) so vision-heavy conversations don't blow the extraction prompt size
- [x] `RECOVERY_MIN_TEXT_CHARS = 50` threshold marks truly-empty conversations as up-to-date with no LLM call
- [x] Recovery-tagged session_id (`recovery_<conv_id>`) suppresses WebUI toasts on re-extraction
- [x] Live result: 112 of 117 stuck conversations cleared on first pass (980 messages extracted)

### Phase 6.10: Summary backfill + semantic summary adapter (May 2026) ✅ COMPLETE

Made conversation summaries usefully retrievable per-turn context. Three steps shipped together (Step 4 — full canonical re-extract — completed 2026-05-12 against the dev's 276-conv corpus, $2.19 actual Haiku spend).

- [x] **Step 1 — extraction prompt bump**: summary instruction from "1-2 sentence" to "3-5 sentences, ~1500 chars covering topics / key parameters / decisions / outcomes." New summaries land ~850 chars vs old ~287 p90.
- [x] **Step 2 — `dawn-admin memory summarize-missing`** (opcode `ADMIN_MSG_MEMORY_SUMMARIZE_MISSING = 0x90`): new module `src/memory/memory_summarize_missing.c` runs the canonical extraction prompt per conversation and stores summary only; shared image-strip + history-load helpers extracted to new `src/memory/memory_history_loader.c` so the new worker reuses recovery's logic. `MEMORY_EXTRACTION_PROMPT_TEMPLATE` promoted to `extern const char *` so backfill and live extraction share one source of truth. `SUMMARIZE_HARD_CAP = 1000` bounds admin-driven LLM spend.
- [x] **Step 3 — semantic summary search**: schema v45 adds `memory_summaries.embedding BLOB`; v46 NULLs `users.embeddings_model_id` so the recompute worker treats every user as stale and backfills on next boot. `memory_db_summary_search_semantic` runs a two-pass locked scan (pass 1 reads `(id, embedding)` only and ranks survivors via min-heap, top-K capped at `MEMORY_SUMMARY_SEMANTIC_TOPK_CAP = 16` for stack safety; pass 2 fetches full rows for survivors only). `memory_embeddings_embed_and_store_summary()` wires embed-at-create in both the live extractor and the summarize-missing backfill worker. Hybrid `summary_adapter` (`memory_focus_adapters.c`) merges keyword + semantic by summary id and re-ranks by max-score.
- [x] **Default tuning**: `focus_injection.top_k 8 → 12`, `source_weights.memory_summary 0.7 → 1.0`, `weight_recency` stays at 0.15 pending re-bench with summary-relevant probe (open tension documented in `config_defaults.c`).
- [x] **Memory-tool double-dip mitigation**: two-sentence nudge in the `--- TURN CONTEXT ---` open framing (`prompt_compose.c`) telling the LLM the focus block already contains its top hits; phrased as a preference, not a prohibition.
- [x] **Folded in**: paraphrase dedup gate at extraction (cosine ≥ 0.92 bumps existing fact's confidence + extends provenance); meta-fact extraction rule rejects "User asked X" / "User inquired Y" / "User requested Z" phrasings; `memory_db_facts_delete_by_patterns` + `cleanup-meta-facts` admin command; provenance-extend semantics in `memory_db_provenance.c` (same-conv widen / no-prov adopt / newer replace / older no-op).
- [x] **Step 4 — full re-extract** (2026-05-12): `dawn-admin memory reextract --user admin --confirm` against 276-conv corpus, 35 min wall-clock, $2.19 actual Haiku spend (higher than the original $0.50-1.00 guess — prompt + provenance + categories all grew per-extraction tokens). Updated cost estimate for future re-extracts: ~$2-3 on Haiku for ~280 convs / ~4500 facts.

### Phase 6.11: Embedding model swap + recompute worker (May 2026) ✅ COMPLETE

- [x] **Tech debt**: `memory_embeddings_embed_and_store_entity()` now calls `memory_embeddings_invalidate_entity_cache()` after update, matching the fact wrapper.
- [x] **ID-based extraction filter**: `conv_db_add_message_ex()` returns the insert row ID; `session_stamp_last_message_id()` plumbs IDs into in-memory history on all live paths; early-skip gate uses `conv_db_get_max_msg_id` vs `last_extracted_msg_id`; count-based cursor retained one release for back-compat.
- [x] **Background re-embed worker** (`src/memory/memory_embed_recompute.c`, schema v41): detects `model_id` mismatch via `system_metadata.embedding_model_id`, re-embeds facts then entities per user (marking `users.embeddings_model_id` only after both succeed), deferred chunks pass, `meta_set` fires only after chunks complete so interrupted passes restart cleanly. Live recompute: 2,183 embeddings re-indexed across 3 users + 243 document chunks on first boot.
- [x] **Model swap to bge-small-en-v1.5-int8**: LoCoMo **81.6%** (+7.9pp from 73.7%), cat-3 inference **64.4%** (+8.6pp), LongMemEval R@5 **97.0%** (+1.4pp), ConvoMem **99.0%** (unchanged).

### Phase 6.12: Cat-2 temporal extraction — Phase 1 anchor injection (May 2026) ✅ COMPLETE

Diagnosis localized cat-2 collapse to extraction-time date loss, not retrieval. Phase 1 ships L1 + L5 from the diagnosis.

- [x] Schema v42: `conversations.anchor_date INTEGER NOT NULL DEFAULT 0` (epoch seconds, literal-constant default for SQLite O(1) ALTER fast path)
- [x] Production `conv_db_create_*` writers populate at insert with `time(NULL)`
- [x] Bench harness overrides per-session via `parse_locomo_session_date()`
- [x] Extraction prompt prepends `Conversation anchor: YYYY-MM-DD` when set, instructs the LLM to resolve relative temporal phrases against it AND preserve the original phrase verbatim in `fact_text` (Case-9 protection)
- [x] New API: `conv_db_get_anchor_date()`, `conv_db_force_anchor_date_unsafe()` (bench-only, authorization-scoped), `ANCHOR_DATE_NONE` sentinel
- [x] **Measured lift (Haiku, n=1982)**: cat-2 recall_generation **0.022 → 0.321** (+29.9pp; projected 0.15-0.25, exceeded high end by +7pp); overall 0.208 → 0.279 (+7.1pp); bonus lift on cat-3 (+4.4pp) and cat-4 (+5.0pp)
- [x] Phase 2 (`event_when` structured-field extraction) re-scoped — see Active TODO

### Phase 6.13: Memory provenance — Phase A + Phase B (May 2026) ✅ COMPLETE

Source-linked recall — append verbatim conversation excerpts to retrieved facts so the caller can verify the extractor's paraphrase. Two phases.

- [x] **Phase A** (commit `e1a34de`): validate provenance triple plumbing through extraction; fix silent buffer truncation across 6 paths
- [x] **Phase B** (commit `4652891`): coverage extension to preferences / summaries / relations; dedup via `memory_source_dedup_set_t`; new module `src/memory/memory_db_provenance.c` (moved out of `memory_db.c` for line-budget); 4 batch source readers (`memory_db_facts_get_sources`, `_relations_get_sources`, `_summaries_get_sources`, `_prefs_get_sources`), each returns parallel `conv_id` / `msg_id_start` / `msg_id_end` arrays for any positive N, auto-chunked at 32 IDs per SQL pass; privacy JOIN against `conversations.is_private`
- [x] Tool integration: `with_source` param on `search` / `recent`; `[memory] source_budget_chars` config caps total verbatim attached per call; strbuf-based output retired earlier 8 KB cap

### Phase 6.14: Dynamic context injection — Phase 1 (May 2026) ✅ COMPLETE

Per-turn focus block ranks facts / entities / relations / summaries / document chunks / calendar events and prepends them to the LLM prompt.

- [x] New modules: `src/memory/memory_focus_adapters.c` + `memory_fact_search.c` + `focus_source.c` + `focus_recency.c` + `focus_candidate_helpers.c`
- [x] Header split: `memory_db.h` carved into `memory_db_entities.h` + `memory_db_embeddings.h` + `memory_db_provenance.h` + `memory_db_aliases.h`; umbrella `memory_db.h` re-exports via transitive include so external callers compile unchanged
- [x] Memory-tool double-dip mitigation: prompt nudge in `prompt_compose.c`
- [x] Phase 2 (DAWN background context — silent-observe events into a sibling memory store) deferred to its own design pass
- [x] Cross-encoder reranker on focus-injection candidates filed under Active TODO (addresses cosine over-inclusion on dominant tokens)

### Phase 6.15: LLM-based fact recategorization (April 2026) ✅ COMPLETE

`dawn-admin memory recategorize-all <username>` complements the embedding-centroid backfill (~15% assignment rate) by covering the remaining ~85% via per-fact LLM classification.

- [x] New module `src/memory/memory_recategorize.c`; batches of 25 general-category facts sent to the extraction LLM (reuses `extraction_provider` / `extraction_model` config)
- [x] Public `memory_extraction_parse_json()` helper (promoted from `extract_json_from_response`)
- [x] New DB helpers `memory_db_fact_list_general()` / `memory_db_fact_count_general()`
- [x] Admin socket opcode `ADMIN_MSG_MEMORY_RECATEGORIZE = 0x80`, fire-and-forget pattern; CAS for TOCTOU race; consecutive-failure abort (3 strikes); injection filter on fact text; `pthread_timedjoin_np` timeout
- [x] First production run: 99.8% classified (946/948 facts in ~80s)

### Phase 6.16: Phase 0 extraction-prompt redesign + Phase 1A entity-graph retrieval (May 2026, schema v47) ✅ COMPLETE

Coupled work — Phase 0 reshapes extraction to produce a dense relation graph; Phase 1A activates 1-hop entity-graph traversal at retrieval time so the dense graph becomes a default-on retrieval signal.

- [x] **Phase 0 — Extraction**: paired-output JSON schema (relations nested inside each fact, eliminating post-hoc text-matching); required `subject` field with precedence ladder (real_name → named entity → descriptor → "User"); two-tier predicate vocabulary (27 standard Schema.org/ConceptNet types + open snake_case custom); "Previously used relation types" prompt hint built from `memory_db_relation_distinct_predicates`; entities-loop-before-facts-loop parser refactor; legacy `find_fact_for_relation` text-matching retired
- [x] **Schema v47** — `memory_facts.subject_entity_id` FK column + idx_memory_facts_subject partial index; backfill from linked relations on first boot
- [x] **New modules** — `memory_predicate_dedup.c/h` (token-Jaccard canonicalizer, 0.66 floor) and `memory_graph_retrieval.c/h` (proper-noun parse → entity resolve → 1-hop walk + grounding bonus)
- [x] **Phase 1A — Retrieval**: `[memory.graph_retrieval]` config block (enabled=true / entity_grounding_bonus=0.40 / max_facts_per_query=30 / use_query_scoring=true) plumbed through all 7 config surfaces (defaults / parser / validator / env JSON + TOML writer / WebUI POST / settings schema / dawn.toml.example)
- [x] **Bench-validated** end-to-end on full LoCoMo (1982 QA, 10 convs, 2736-fact fresh corpus): internal `recall_reach` 0.7392 → 0.8733 (+13.4pp); per-category lift +8.3 to +19.0pp across all five categories simultaneously. `recall_reach` is internal-diagnostic only — for leader-comparable position see [STATE.md](STATE.md)
- [x] Smoke: 100% relation→fact linkage (was 17%); 98.4% subject_entity_id coverage on the fresh corpus

### Phase 6.17: BM25 keyword index (May 2026, schema v48) ✅ SHIPPED (opt-in)

Phase 1 of the Mem0 architectural parity plan — `docs/MEM0_ARCHITECTURAL_PARITY.md` in the dawn repo.

- [x] **Schema v48** — `memory_facts_fts` FTS5 contentless virtual table (`tokenize='unicode61 remove_diacritics 2'`, single `fact_stems` column); v48 migration block creates the table + backfills every existing `memory_facts` row; `v48_ok` guard tracks both CREATE and backfill so a transient backfill failure doesn't advertise a partial index as complete
- [x] **New modules** — `memory_bm25.c/h` (sigmoid normalization with query-length-adaptive midpoint + steepness across five tiers; adapted from `mem0/utils/scoring.py` under Apache-2.0 — see `NOTICE`) and `memory_stem.c/h` (Porter2 / libstemmer pre-pass)
- [x] **Sync on writes** — fact create / supersede / delete paths inside `memory_db_facts.c` keep `memory_facts_fts` in sync via explicit `INSERT` / `DELETE` (contentless table — app owns rows)
- [x] **Config** — `[memory] bm25_enabled` (default `false`); flipping to `true` routes the keyword channel of hybrid search through the FTS5 BM25 path instead of the legacy SQL LIKE + multi-token match-count path
- [x] **Default-off in v1** — shipped as a documented experiment ahead of the full end-to-end leader-comparable bench gate; expected to default-on once Phase 1 of the parity plan closes

### S4: Entity Graph ✅ COMPLETE

- [x] Entity types and relation types (`memory_entity_t`, `memory_relation_t`)
- [x] SQLite schema: `memory_entities` and `memory_relations` tables
- [x] 8 prepared statements for entity/relation CRUD
- [x] Entity upsert with `RETURNING id` and canonical name normalization
- [x] Relation storage with entity FK or literal value
- [x] Extraction wiring: entities and relations parsed from LLM JSON output
- [x] Semantic embeddings for entities (embed on creation only, not every mention)
- [x] Multi-provider embedding support (Ollama, OpenAI, ONNX) via `[memory.embeddings]`
- [x] In-memory embedding cache with mutex protection (fact cache: 1000, entity cache: 500)
- [x] Hybrid search: keyword + cosine similarity with configurable weights
- [x] Bidirectional graph search: outgoing + incoming relations for top entities
- [x] Entity graph context appended to memory search results (ENTITIES section)
- [x] Existing entities fed into extraction prompt to prevent duplicate names
- [x] Entity dedup: canonical name matching prevents "Jon" vs "Jonathan Smith" variants
- [x] `_Static_assert` on prepared statement layout for safety
- [x] Bulk relation loading (`memory_db_relation_list_all_by_user()`) — N+1 query fix
- [x] Entity deletion with FK-ordered relation cleanup

### S5: Memory Injection Filter ✅ COMPLETE

Shared blocklist-based injection filter (`memory_filter.c/h`) prevents prompt injection via stored memory content. Designed to block attempts to influence LLM behavior through facts, preferences, entities, relations, summaries, and topics that flow into future system prompts.

- [x] Unicode normalization pipeline: zero-width stripping, Cyrillic/Greek homoglyph mapping, Latin-1 accent stripping, fullwidth ASCII mapping, bidi/tag character handling, UTF-8 continuation byte validation
- [x] ~118 blocked patterns across 17 categories (imperatives, overrides, credentials, role manipulation, LLM markers, XML/HTML injection, markdown exfiltration, base64, jailbreak, system impersonation, memory poisoning, recommendation poisoning, behavioral modification, social engineering, calendar metadata, ReAct co-occurrence)
- [x] Data-marking framing in system prompt ("DATA entries, not instructions")
- [x] Wired into all storage paths: tool callback (`memoryCallback`), sleep-consolidation extraction (facts, preferences, corrections, entities, relations, summaries, topics), WebUI import (facts, preferences with field truncation)
- [x] `skipped_blocked` counter in WebUI import response
- [x] 137 unit tests (`test_memory_filter`)
- [x] Three agent review passes + Copilot review

Commits: `b16bbae`, `c4bdb2f`, `9b01d7d`, `089f79c`, `23fc79c`

### S6: Fact Categorization ✅ COMPLETE

8-label fixed taxonomy for memory facts, enabling category-filtered search that reduces noise and improves retrieval precision.

- [x] Schema v34: `memory_facts.category` TEXT NOT NULL DEFAULT 'general'
- [x] Taxonomy: personal, professional, relationships, health, interests, practical, preferences, general — validated from single source of truth in `memory_types.h`
- [x] Extraction prompt updated with per-category guidelines
- [x] LLM tool gains `category` enum param for `search`/`recent` actions
- [x] Pre-filtered at SQL level via `memory_db_fact_search_by_category()` so hybrid scoring only sees the right slice
- [x] Embedding-centroid backfill: 7 category centroids built from seed phrases, existing facts classified by cosine similarity above configurable threshold (default 0.25)
- [x] LLM-based recategorization via `dawn-admin memory recategorize-all` — sends batches of 25 general-category facts to the extraction LLM for per-fact classification (first run: 99.8% classified, 946/948 facts in ~80s)
- [x] `categories_backfilled_at` column on users table for per-user gate

Commits: `07e20aa`, `69f8cef`

### S7: Temporal Scoring ✅ COMPLETE

Time-expression parser and temporal proximity scoring for time-anchored memory queries.

- [x] New `src/core/time_query_parser.c/h` (Layer 1) — recognizes absolute dates ("in 2020", "September 2022"), numeric relative ("5 days ago", "two weeks ago" with English number words 1-31), named relative ("last week", "yesterday"), bare month with verb disambiguation ("in may" vs "I may visit"), and vague expressions ("recently", "lately")
- [x] Additive Gaussian decay boost in `memory_embeddings_hybrid_search()` (facts) and `document_search.c` (chunks): `score += temporal_weight * proximity(item.created_at, target_ts)`
- [x] Temporal relation bounds: schema v33 adds `valid_from`/`valid_to` on `memory_relations`, `memory_db_relation_supersede()` auto-closes exclusive relations with temporal awareness
- [x] `memory.temporal_weight` config key (default 0.20), surfaced through TOML, config_env JSON, WebUI settings
- [x] Schema v35: `document_chunks.created_at` with backfill from parent documents
- [x] ISO-8601 date recognizer (`YYYY-MM-DD` ± 1 day, `YYYY-MM` ± 15 days)
- [x] 78 parser unit tests (51 original + 27 follow-up)
- [x] Measured lift: LoCoMo overall 71.4%→73.7% (+2.3pp), cat-2 temporal 76.7%→79.9% (+3.2pp), cat-3 inference 51.5%→55.8% (+4.3pp); LongMemEval temporal-reasoning R@10 95.5%→97.7% (+2.3pp)

Commits: `15cf0d9`, `07e20aa`, `c8a0164`

### S8: Contradiction Detection ✅ COMPLETE

Entity-graph-driven fact contradiction detection. When an exclusive relation is superseded, the linked fact is automatically superseded too.

**Research context:** An embedding-based experiment (`bench_contradiction`, 215 labeled pairs across 5 labels and 15 semantic categories) proved that cosine similarity + Jaccard word-overlap cannot separate contradictions from paraphrases — best F1=0.642. Paraphrases scored higher cosine (0.858) than contradictions (0.765), and Jaccard added only +0.019 F1. This is consistent with published findings: SparseCL (Xu & Lin, ICML 2025) shows contradiction manifests in a sparse semantic subspace that vanilla cosine misses, and the negation-blindness problem in sentence embeddings is well-documented (arXiv:2504.00584).

**Solution:** Rather than fine-tuning embeddings, leverage the existing relation-graph exclusivity mechanism. The relation system already validates contradictions for structured data — the gap was propagating that knowledge to the fact layer.

- [x] `UPDATE...RETURNING fact_id` on the close-open statement — atomically retrieves the old relation's linked fact when closing an exclusive relation (no TOCTOU race)
- [x] `out_old_fact_id` output parameter on `memory_db_relation_supersede()` (NULL-safe, backward compatible)
- [x] `fact_map` in extraction pipeline tracks newly created facts for relation linkage
- [x] `find_fact_for_relation()` matches entity names in fact text via `strcasestr` (same-batch LLM output)
- [x] Expanded exclusive relations (5→12): `works_at`, `lives_in`, `married_to`, `attends_school`, `owns_vehicle`, `born_in`, `born_on`, `favorite_color`, `favorite_food`, `primary_language`, `email_is`, `phone_number_is`
- [x] Contradictory relation pairs (4): likes↔dislikes, enjoys↔hates, can↔cannot, is↔is_not — closes opposing relation on same (subject, object) pair
- [x] Extraction prompt expanded to 20 relation types matching all exclusive + contradictory types
- [x] `bench_contradiction` experiment binary preserved as reusable tool for future embedding experiments
- [x] 7 unit tests (14 assertions) covering exclusive supersede, legacy data, non-exclusive skip, idempotency, contradictory pairs, and different-object regression case
- [x] Architecture review: 0 critical, 1 high (SQL bug fixed), 4 medium (all fixed)

Commit: `cb3458b`

**Memory System Status: Phases 1-6.7 + S4-S8 Complete, RAG Complete (see `docs/RAG_DESIGN.md`)**

---

### RAG SYSTEM — ✅ IMPLEMENTED (see `docs/RAG_DESIGN.md`)

Phases 7-11 were implemented as a separate subsystem documented in `docs/RAG_DESIGN.md` (March 2026). Key implementation files: `src/tools/document_search.c`, `src/tools/document_db.c`, `src/tools/document_chunker.c`, `src/core/embedding_engine.c`, `src/webui/webui_doc_library.c`, `www/js/ui/doc-library.js`.

---

### FUTURE ENHANCEMENTS

### Phase 12: Speaker Identification

- [ ] sherpa-onnx integration
- [ ] Voice enrollment
- [ ] Per-utterance speaker identification
- [ ] Memory loading by identified speaker

### Phase 13: Per-User Document Storage — SHIPPED

- [x] WebUI file upload capability
- [x] Per-user document directories
- [x] Add `user_id` back to RAG tables
- [x] User-isolated document search
- [x] Storage quota management (admin global-flag toggle + all-users view)

### Phase 14: Advanced RAG

- [ ] Reranking after initial retrieval (6-7% accuracy improvement)
- [ ] Multimodal document indexing (images, diagrams from PDFs)
- [ ] Lightweight safety classifier for edge cases

---

**Summary (May 2026):**

| System                                              | Phases       | Status           |
| --------------------------------------------------- | ------------ | ---------------- |
| Memory base (extraction / retrieval)                | 1-6.7        | ✅ Complete      |
| Entity Graph                                        | S4           | ✅ Complete      |
| Injection Filter                                    | S5           | ✅ Complete      |
| Categorization                                      | S6           | ✅ Complete      |
| Temporal Scoring                                    | S7           | ✅ Complete      |
| Contradiction Detection                             | S8           | ✅ Complete      |
| Entity Merge — soft aliases + Phase 2               | 6.8          | ✅ Complete      |
| Crash-Recovery Worker                               | 6.9          | ✅ Complete      |
| Summary Semantic Adapter                            | 6.10         | ✅ Complete      |
| Embedding Recompute + bge-small swap                | 6.11         | ✅ Complete      |
| Cat-2 Temporal Anchor (Phase 1)                     | 6.12         | ✅ Complete      |
| Memory Provenance (A + B)                           | 6.13         | ✅ Complete      |
| Dynamic Context Injection (Phase 1)                 | 6.14         | ✅ Complete      |
| LLM Recategorization                                | 6.15         | ✅ Complete      |
| Phase 0 Extraction-Prompt Redesign (v47)            | parity-pre   | ✅ Complete      |
| Phase 1A Entity-Graph Retrieval (default-on)        | parity-pre   | ✅ Complete      |
| Phase 1 BM25 Keyword Index (v48, opt-in)            | parity-1     | ✅ Shipped       |
| RAG                                                 | 7-11         | ✅ Complete      |
| Per-User Docs                                       | 13           | ✅ Shipped       |
| Cat-2 Temporal Phase 2 (`event_when`)               | future       | Re-scoped        |
| Cross-Encoder Reranker                              | future       | Dead-lettered    |
| Dynamic Context Injection — Phase 2                 | future       | Future           |
| Speaker ID                                          | 12           | Future           |
| Advanced RAG                                        | 14           | Future           |

Forward-looking phases of the Mem0 architectural parity plan (lemmatization,
extraction-prompt overhaul, scoring refinements, re-evaluation of disabled
experimental code) live in `docs/MEM0_ARCHITECTURAL_PARITY.md` in the dawn
repo — not duplicated here.

---

## 12. File Structure

Cross-checked against `ls src/memory/`, `ls include/memory/`, and `ls src/core/focus/` at schema v48 (2026-05-16, post-cleanup). The May 2026 cleanup pass folded the monolithic `memory_db.c` (3,800 LOC) and `memory_db_alias.c` (3,800 LOC) into per-record-type modules and moved the trust-boundary focus helpers into a dedicated `src/core/focus/` subdirectory. The umbrella `memory_db.h` still re-exports the split headers so external callers compile unchanged. Six new modules — `memory_bm25.c`, `memory_stem.c`, `memory_score_floor.c`, `memory_source_dedup.c`, `memory_graph_retrieval.c`, `memory_predicate_dedup.c` — landed alongside Phase 0 / Phase 1A / Phase 1 BM25 / score-floor work.

```
include/memory/
├── memory_db.h                  # Umbrella header — re-exports the split headers below
├── memory_db_internal.h         # Cross-TU helpers shared by the six memory_db_*.c modules
├── memory_db_entities.h         # Entity CRUD, upsert, search
├── memory_db_embeddings.h       # Fact-embedding storage + cache invalidation
├── memory_db_aliases.h          # Entity-alias resolver cascade + alias_link/unlink
├── memory_db_alias_internal.h   # Cross-TU helpers shared by the four memory_db_alias_*.c modules
├── memory_db_provenance.h       # Phase B — source-linked recall batch readers
├── memory_db_admin.h            # Admin-only DB helpers (bulk delete, recategorize support)
├── memory_embeddings.h          # Semantic embedding API (multi-provider, hybrid search)
├── memory_embed_recompute.h     # Background re-embed worker (model_id mismatch + summary backfill)
├── memory_embed_tokenizer.h     # Shared WordPiece tokenizer (extracted from ONNX provider)
├── memory_extraction.h          # Sleep-consolidation extraction entry points
├── memory_fact_search.h         # Hybrid-search public surface (extracted from memory_embeddings.c)
├── memory_focus_adapters.h      # Per-turn focus block: facts/entities/relations/summaries/chunks/calendar
├── memory_filter.h              # Injection-pattern blocklist (~118 patterns + Unicode normalize)
├── memory_context.h             # Session-start context builder (legacy ~800-tok block)
├── memory_history_loader.h      # Shared message-history loader for recovery + summarize-missing
├── memory_maintenance.h         # Nightly decay orchestration API
├── memory_recategorize.h        # LLM-based fact-category recategorize admin path
├── memory_recovery.h            # Crash-recovery worker for unconsolidated sessions
├── memory_summarize_missing.h   # Backfill summary rows for legacy / failed extractions
├── memory_similarity.h          # Duplicate detection API (normalize, hash, Jaccard)
├── memory_bm25.h                # v48 BM25 sigmoid normalization (mem0-adapted)
├── memory_stem.h                # Porter2 (libstemmer) pre-pass for FTS5
├── memory_graph_retrieval.h     # Phase 1A — entity-graph 1-hop retrieval grounding
├── memory_predicate_dedup.h     # Token-Jaccard canonicalizer for relation predicates
├── memory_types.h               # Data structures (fact, pref, summary, entity, relation,
│                                #   provenance triple, fact-category taxonomy, return codes)
├── memory_callback_internal.h   # Internal types shared by memory_callback.c + provenance
└── contacts_db.h                # Contact CRUD API (find, add, update, delete, list)

include/core/
├── embedding_engine.h           # Shared embedding API (ONNX, Ollama, OpenAI providers)
├── time_query_parser.h          # Recognizes temporal expressions in queries (moved from src/tools)
├── iso8601.h                    # Canonical ISO 8601 parsing (consolidated April 2026)
├── strbuf.h                     # Growable string buffer used for memory tool output
└── focus/
    ├── focus_candidate_helpers.h  # Shared helpers across focus-block adapters
    ├── focus_recency.h            # Temporal-decay scoring for focus-block ranking
    ├── focus_source.h             # Source-type taxonomy (INTERNAL / EXTERNAL / USER_CONTENT)
    └── focus_dominant_token.h     # Reranker heuristic — over-inclusion dampener

include/tools/
├── document_db.h                # Document/chunk CRUD (RAG storage layer)
├── document_chunker.h           # Text chunking for embedding
├── document_search.h            # RAG semantic search tool
├── document_extract.h           # PDF/DOCX/HTML text extraction
├── document_index_pipeline.h    # Shared indexing pipeline (chunk, embed, store)
└── time_utils.h                 # parse_time_period() — h/m/d/w/y unit parsing

src/memory/
  # ── Storage (post-Phase-6a split of monolithic memory_db.c, May 2026) ─────────
├── memory_db.c                  # Core/shared SQLite plumbing, lock, prepared-stmt registry
├── memory_db_facts.c            # Fact CRUD, hybrid search public entry, subject_entity_id
├── memory_db_summaries.c        # Summary CRUD + window-list helpers (Bundle 3)
├── memory_db_entities.c         # Entity CRUD, upsert (with first_seen / last_seen overrides),
│                                #   equivalence-class read-side aggregation
├── memory_db_relations.c        # Relation CRUD, bitemporal validity, contradiction detection
│                                #   (EXCLUSIVE_RELATIONS[], CONTRADICTORY_PAIRS[],
│                                #   memory_db_relation_supersede)
├── memory_db_prefs.c            # Preference CRUD + window-list helpers
├── memory_db_admin.c            # Admin-only DB helpers (bulk delete by pattern, etc.)
├── memory_db_provenance.c       # Phase B — 4 batch source readers + privacy JOIN,
│                                #   auto-chunked at 32 IDs per SQL pass
  # ── Entity-merge alias surface (post-Phase-6b split of monolithic memory_db_alias.c) ─
├── memory_db_alias.c            # Shared cascade orchestration + public alias entry points
├── memory_db_alias_scorer.c     # Composite-score helpers (Jaccard, embedding, type, overlap)
├── memory_db_alias_cascade.c    # Six-stage resolver cascade + Phase 2 auto-merge gate
├── memory_db_alias_writes.c     # alias_link / alias_unlink / merge / hard-consolidate writes
  # ── Embedding plumbing ──────────────────────────────────────────────────────
├── memory_embeddings.c          # Embedding cache, hybrid search dispatch, cache invalidation
├── memory_embed_ollama.c        # Ollama embedding provider (/api/embed endpoint)
├── memory_embed_openai.c        # OpenAI embedding provider (/v1/embeddings endpoint)
├── memory_embed_onnx.c          # ONNX Runtime embedding provider (local)
├── memory_embed_recompute.c     # Background re-embed worker, v41 + v46 summary backfill
├── memory_embed_tokenizer.c     # WordPiece tokenizer shared with reranker investigation
  # ── Retrieval / scoring ─────────────────────────────────────────────────────
├── memory_fact_search.c         # Hybrid search public surface (extracted from memory_embeddings.c)
├── memory_bm25.c                # v48 BM25 sigmoid normalization (mem0-adapted, Apache-2.0)
├── memory_stem.c                # Porter2 (libstemmer) pre-pass feeding memory_facts_fts
├── memory_score_floor.c         # search_score_floor cut-off applied at memory_fact_search.c
├── memory_graph_retrieval.c     # Phase 1A — proper-noun parse → entity resolve → 1-hop walk
├── memory_predicate_dedup.c     # Token-Jaccard canonicalizer for extraction predicates
├── memory_focus_adapters.c      # Memory-specific focus-block adapters (fact / entity / relation /
│                                #   summary / chunk) — generic focus primitives now live in src/core/focus/
├── memory_source_dedup.c        # source_dedup_set_t — suppresses re-fetched provenance
  # ── Extraction / consolidation ──────────────────────────────────────────────
├── memory_extraction.c          # LLM-based extraction: facts, prefs, entities, relations,
│                                #   summaries, topics; subject-naming + anchor-aware prompt;
│                                #   Phase 2 gate sweep + auto-promote user_self hooks
├── memory_recategorize.c        # Per-fact LLM recategorization admin command
├── memory_recovery.c            # Crash-recovery worker (idle 1h, 24h scan, 5-min timeout)
├── memory_summarize_missing.c   # dawn-admin memory summarize-missing — opcode 0x90
├── memory_history_loader.c      # Shared image-strip + history-load for recovery + summarize-missing
  # ── Misc ────────────────────────────────────────────────────────────────────
├── memory_context.c             # Session-start context builder
├── memory_callback.c            # LLM tool callback dispatcher (search / remember / forget /
│                                #   recent + 4 contact actions + merge_entities → alias_link)
├── memory_filter.c              # Injection-pattern blocklist + Unicode normalization
├── memory_similarity.c          # Duplicate detection (Jaccard, FNV-1a hashing)
├── memory_maintenance.c         # Nightly decay orchestration
└── contacts_db.c                # Contact CRUD (find, add, update, delete, list)

src/core/
├── embedding_engine.c           # Provider-agnostic embedding + vector math
├── time_query_parser.c          # Layer 1 temporal-expression recognizer
├── iso8601.c                    # Canonical ISO 8601 parsing
├── strbuf.c                     # Growable string buffer (256 KiB cap, sticky-OOM)
└── focus/                       # Post-Phase-6c move — trust-boundary helpers leave src/memory/
    ├── focus_source.c           # Source-type tagging for trust-boundary filter routing
    ├── focus_recency.c          # Temporal-decay scoring inside the focus-block adapters
    ├── focus_dominant_token.c   # Reranker heuristic — over-inclusion dampener
    └── focus_candidate_helpers.c  # Shared helpers across focus-block adapters

src/tools/
├── document_db.c                # SQLite CRUD for documents and chunks
├── document_chunker.c           # Paragraph/sentence splitting with configurable overlap
├── document_search.c            # RAG search: cosine + keyword + temporal-proximity
├── document_extract.c           # PDF (MuPDF), DOCX (libzip+libxml2), HTML, plain text
├── document_index_tool.c        # LLM tool for URL-based document ingestion
├── document_index_pipeline.c    # SHA-256 dedup, chunking, embedding, DB storage
└── memory_tool.c                # Tool registry metadata + descriptor (11 params)

src/webui/
├── webui_memory.c               # WebUI memory endpoints (list, delete, stats, entities,
│                                #   import/export with filter pre-pass)
├── webui_contacts.c             # WebUI contact CRUD endpoints
└── ...

src/auth/
├── admin_socket.c               # dawn-admin socket — memory-entity dispatch handlers
│                                #   (consolidate / aliases / proposals / link-user-self / ...)
└── auth_db_settings.c           # User-identity helpers (v44 real_name / preferred_address / aliases)

www/js/ui/
├── memory.js                    # Memory viewer UI (tabs, search, entity graph, delete,
│                                #   memory-icon dot indicator + auto-route)
├── memory_aliases.js            # Alias panel + Suggested-Merges UI
└── ...

www/css/components/
├── memory.css                   # Memory viewer styles (entity badges, relations, tabs,
│                                #   dot indicator pulse + prefers-reduced-motion carve-out)
└── ...

benchmarks/
├── bench_retrieval.c            # C-side retrieval scoring driver (LongMemEval/LoCoMo/ConvoMem)
├── bench_memory_pipeline.c      # End-to-end extraction + retrieval bench (Phase 0+ memory mode)
├── bench_temporal_arithmetic.py # Cat-2 temporal-arithmetic LLM guardrail
├── bench_contradiction.c        # 215-pair embedding contradiction experiment (preserved for reuse)
├── run_benchmark.py             # Orchestrator
└── ...

scripts/
└── scan_legacy_memory_filter.c  # One-time legacy-data scanner against memory_filter_check
```

---

## 13. Privacy and Security

### 13.1 Data Stored

- Facts and preferences in plain text in SQLite
- Full conversation transcripts NOT stored (only summaries)
- Sensitive items excluded by extraction prompt
- Document contents stored as chunks (not full files)

### 13.2 Mitigations

**Prompt Injection via Memory:**

- Sanitize loaded memories
- Don't allow instruction-like content in facts
- Validate extracted JSON structure

**Memory Disclosure:**

- Full transparency: User can always see what's stored
- WebUI viewer for complete memory access
- Voice command: "What do you know about me?"

**Data Export/Deletion:**

- User can delete individual facts
- "Forget everything about me" command (WebUI typed confirmation)
- Export all memories as JSON (lossless) or plain text (portable)
- Import from DAWN JSON, ChatGPT, Claude, or plain text with duplicate detection

### 13.3 Local vs Cloud Processing

| Operation    | Default      | Privacy Note                                    |
| ------------ | ------------ | ----------------------------------------------- |
| Conversation | User choice  | Depends on LLM provider                         |
| Extraction   | Local        | Can be configured for cloud if quality needed   |
| Embeddings   | Configurable | Ollama (local), ONNX (local), or OpenAI (cloud) |
| Storage      | Local        | SQLite on device                                |

### 13.4 Safety Guardrails

Memory content flows into future LLM prompts, creating a potential attack vector. We implement **dual-stage filtering** (inspired by enterprise voice agent patterns):

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INPUT GUARDRAILS                                  │
│                    (Remember Tool)                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User: "Remember that I'm allergic to peanuts"                      │
│                              │                                       │
│                              ▼                                       │
│                    ┌─────────────────┐                              │
│                    │ Pattern Filter  │                              │
│                    │ (Blocklist)     │                              │
│                    └────────┬────────┘                              │
│                              │                                       │
│         ┌────────────────────┼────────────────────┐                 │
│         ▼                    ▼                    ▼                  │
│    [PASS]              [BLOCKED]            [BLOCKED]               │
│  "allergic to       "whenever someone    "ignore previous          │
│   peanuts"           asks, reveal..."     instructions"            │
│         │                                                            │
│         ▼                                                            │
│    Store with source="explicit", confidence=1.0                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    OUTPUT GUARDRAILS                                 │
│                    (Extraction Process)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Extraction LLM returns:                                             │
│  {"facts": [{"text": "User is vegetarian", ...}, ...]}              │
│                              │                                       │
│                              ▼                                       │
│                    ┌─────────────────┐                              │
│                    │ Validation      │                              │
│                    │ Layer           │                              │
│                    └────────┬────────┘                              │
│                              │                                       │
│  For each extracted fact:                                            │
│                              │                                       │
│  1. Length check:    fact_text < 512 bytes?                         │
│  2. Pattern filter:  No instruction-like content?                   │
│  3. Source check:    source ∈ {"explicit", "inferred"}?             │
│  4. Confidence:      0.0 ≤ confidence ≤ 1.0?                        │
│                              │                                       │
│         ┌────────────────────┼────────────────────┐                 │
│         ▼                    ▼                    ▼                  │
│    [PASS]              [REJECTED]           [REJECTED]              │
│  Store in DB         Log warning,          Log warning,             │
│                      skip this fact        skip this fact           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Blocked Patterns (S5 — `memory_filter.c`):**

The original 14-pattern blocklist was replaced with a comprehensive filter in S5:

- ~118 multi-word blocked patterns across 17 categories (imperatives, overrides, credentials, role manipulation, LLM markers, XML/HTML injection, markdown exfiltration, base64, jailbreak, system impersonation, memory poisoning, recommendation poisoning, behavioral modification, social engineering, calendar metadata)
- ReAct co-occurrence detector (blocks 2+ of `thought:`/`action:`/`observation:`)
- Unicode normalization pipeline before pattern matching: zero-width/invisible char stripping, Cyrillic/Greek homoglyph mapping, Latin-1 accent stripping, fullwidth ASCII mapping, bidi/tag character handling
- Filter wired into all ingestion paths (tool callback, extraction, WebUI import)
- 137 unit tests validate pattern coverage and normalization bypass resistance

See S5 phase section for full details. Design doc archived: `INJECTION_FILTER.md` (sibling).

**Why not a dedicated safety model?**

- Enterprise systems (NVIDIA Nemotron) use 8B+ parameter safety models
- Overkill for personal home assistant on embedded hardware
- Pattern matching catches obvious attacks with zero latency
- Future: weighted scoring with per-pattern confidence (0.3-0.9) and context modifiers for quoted/fictional text — deferred until binary blocking proves insufficient

**Future Enhancements (Phase 14+):**

- Reranking for RAG retrieval (6-7% accuracy improvement)
- Multimodal document indexing (images/diagrams)
- Multi-language injection filter (Korean/Japanese/Chinese patterns)
- LLM-based contradiction judgment for cases entity-graph analysis doesn't cover (quantity, subtle negation, preference shifts)

---

## 14. Testing Strategy

### 14.1 Unit Tests

- Storage CRUD operations (`test_document_db`, `test_memory_embeddings`)
- Embedding math: L2 norm, cosine similarity (`test_embedding_engine`)
- Text chunking: paragraph/sentence splitting, overlap (`test_document_chunker`)
- Decay calculations
- JSON extraction parsing
- Duplicate detection: FNV-1a hash, Jaccard similarity

### 14.2 Integration Tests

- End-to-end extraction from transcript
- Memory loading into context
- RAG retrieval accuracy
- Session lifecycle

### 14.3 Retrieval Benchmarks

A standalone benchmark harness (`benchmarks/`) measures retrieval quality against
three published academic benchmarks. The harness exercises DAWN's real embedding
engine and document search scoring via a C binary (`bench_retrieval`) driven by a
Python orchestrator (`run_benchmark.py`). The binary uses an in-memory SQLite
database — completely isolated from production data.

**Benchmarks supported:**

| Benchmark | Source | Size | Metric |
|-----------|--------|------|--------|
| LongMemEval | HuggingFace `xiaowu0162/longmemeval-cleaned` | 500 questions | Recall@K, NDCG@K |
| LoCoMo | GitHub `snap-research/locomo` | 10 convos, ~2000 QA pairs | Avg recall (fraction of evidence found) |
| ConvoMem | HuggingFace `Salesforce/ConvoMem` | 75K+ QA pairs | Partial recall (substring evidence match) |

**Modes:**

- **Raw**: Pure cosine similarity (no keyword boosting). Baseline comparison.
- **Hybrid**: Cosine similarity + keyword boosting (0.15 weight on top 50 candidates). DAWN's default production scoring path.

**Session-level results (April 2026, all-MiniLM-L6-v2 ONNX int8, full datasets):**

| Benchmark | Raw | Hybrid | Notes |
|-----------|-----|--------|-------|
| **LongMemEval R@5** | 96.0% | 97.2% | 500 questions, top-K=10 retrieved |
| **LongMemEval R@10** | 98.4% | 99.2% | Near-perfect at depth 10 |
| **LongMemEval NDCG@10** | 88.9% | 91.7% | Position-weighted relevance |
| **LoCoMo avg recall** | — | 73.7% | 1982 QA pairs, dialog granularity, top-K=10 (post-S6/S7) |
| **ConvoMem avg recall** | — | 99.0% | 100 items, per-message retrieval |

**Turn-level results (LongMemEval, matches academic evaluation methodology):**

Turn-level retrieval indexes each user turn as a separate document (~273 per question instead of ~48 sessions). Top-K=5, matching the RMM paper (Tan et al., ACL 2025, arxiv.org/abs/2503.08026). Methodology verified against the official LongMemEval evaluation code (github.com/xiaowu0162/LongMemEval).

| Scoring | R@1 | R@3 | R@5 | NDCG@5 | Evaluated | Model |
|---------|-----|-----|-----|--------|-----------|-------|
| **Official** (any turn from answer session) | 89.2% | 95.2% | **97.0%** | 92.5% | 500 questions | bge-small int8 + proper-noun boost |
| **Official** (MiniLM baseline, April 2026) | 85.4% | 93.2% | 95.6% | 89.8% | 500 questions | all-MiniLM-L6-v2 int8 |
| **Strict** (only `has_answer` turns) | 50.2% | 75.9% | **86.9%** | 69.4% | 428 questions (72 skipped) | all-MiniLM-L6-v2 int8 |

The official scoring is directly comparable to published baselines. The strict scoring evaluates retrieval against only the specific turns annotated as containing the answer (average 1.7 targets per question vs 11 for official).

**Comparison to published baselines (LongMemEval turn-level R@5, top-K=5):**

| System | R@5 | Model | Model Size | Method |
|--------|-----|-------|------------|--------|
| **DAWN hybrid** | **97.0%** | bge-small-en-v1.5 (int8) | 32MB | Cosine + keyword boost + proper-noun boost |
| **DAWN hybrid** | 95.6% | all-MiniLM-L6-v2 (int8) | 22MB | Cosine + keyword boost (April 2026 baseline) |
| RMM + GTE | 69.8% | GTE-Qwen2-7B | 7B | Cosine + RL-trained reranker |
| RAG + GTE | 62.4% | GTE-Qwen2-7B | 7B | Cosine only |
| RAG + Stella | 59.2% | Stella 1.5B | 1.5B | Cosine only |
| RAG + Contriever | 54.3% | Contriever | 110M | Cosine only |

DAWN's hybrid search exceeds the published SOTA (RMM + GTE) by 27.2 percentage points while using a model 200x smaller. Results verified deterministic across two independent hardware runs (Jetson Orin and Jetson AGX Orin — identical R@5 = 0.8692 on strict scoring).

**Why keyword boosting is effective at turn-level:**

At session level, documents are 500-2000 characters (all user turns concatenated) — keyword matching has moderate discriminative power since common words appear across many sessions. At turn level, documents are 50-200 characters (single messages) — keyword matching becomes highly discriminative because short documents either contain a query word or they don't. That binary signal on top of fuzzy cosine breaks ties the embedding model alone can't resolve. The effect amplifies as document granularity decreases.

**LoCoMo category breakdown and analysis (May 2026):**

LoCoMo QA pairs fall into 5 categories. Understanding which categories are hard and why shapes the improvement roadmap:

| Category | Description | MiniLM baseline | +proper-noun boost | +bge-small | Root cause of gap |
|----------|-------------|-----------------|-------------------|------------|-------------------|
| **1** | Profile/identity facts ("What is Caroline's relationship status?") | 64.8% | 64.8% | 69.1% | Fact stated once, often indirectly; semantic gap between question vocabulary and answer vocabulary |
| **2** | Temporal ("When did X happen?") | 79.7% | 80.5% | 85.6% | Temporal boost + proper-noun boost effective; most headroom already captured |
| **3** | Inference/counterfactual ("Would she enjoy X?") | 58.0% | 58.0% | 63.0% | Evidence never directly answers; requires reasoning across multiple facts; retrieval ceiling is fundamental |
| **4** | Event detail ("What did the race raise awareness for?") | 74.8% | 74.8% | 84.1% | Co-located with event; stronger model bridges vocabulary gap well |
| **5** | Consequence/reflection ("What did she realize after X?") | 78.9% | 78.9% | 86.8% | Usually stated in same session as event; strong lift from better model |

**Key findings from May 2026 ablation study:**

1. **Proper-noun boost (+1.0 weight on capitalized keyword matches):** +0.7pp overall, predominantly cat-2 (+0.8pp) and cat-3 (+2.2pp). Zero effect on cat-1 because every chunk starts with the speaker name — the name matches uniformly and doesn't discriminate. Implemented as `--proper-noun-boost` flag in `bench_retrieval` and passed through `run_benchmark.py`.

2. **Sentence-level chunking:** Zero effect on LoCoMo. LoCoMo dialog turns are already short single utterances — there is no multi-sentence dilution to remove. Hypothesis was valid but the data structure doesn't have the problem.

3. **Embedding model comparison** (all runs: hybrid + proper-noun-boost 1.0 + temporal-weight 0.20):

| Model | ONNX size | Per-embed (Jetson) | Overall | Cat-1 | Cat-2 | Cat-3 | Recommendation |
|-------|-----------|-------------------|---------|-------|-------|-------|----------------|
| all-MiniLM-L6-v2 int8 | 22MB | ~8ms | 74.4% | 64.8% | 80.5% | 58.0% | Current default |
| all-MiniLM-L12-v2 int8 ARM64 | 32MB | ~15ms | 73.5% | 64.4% | 79.3% | 56.4% | Worse — same family, depth doesn't help |
| bge-small-en-v1.5 fp32 | 127MB | ~35ms | 81.8% | 69.1% | 85.6% | 63.0% | Good but fp32 |
| **bge-small-en-v1.5 int8** | **32MB** | **~15ms** | **81.6%** | **69.3%** | **84.9%** | **64.4%** | **✅ Ship this** |
| bge-base-en-v1.5 fp32 | 416MB | ~117ms | 83.2% | 70.6% | 86.1% | 62.2% | Diminishing returns |
| bge-large-en-v1.5 fp32 | 1.3GB | ~396ms | 82.9% | 70.2% | 86.6% | 65.0% | Not practical on Jetson |

**bge-small int8 is the production recommendation:** Same 32MB footprint as current model, ~2x embedding latency (~15ms vs ~8ms), +7.2pp overall retrieval quality, +4.5pp cat-1, +6.4pp cat-3. Quantization loses only 0.2pp vs fp32 — essentially lossless. Source: `Xenova/bge-small-en-v1.5`, `onnx/model_int8.onnx`.

`DAWN_ONNX_MODEL` / `DAWN_ONNX_VOCAB` env vars now override the hardcoded model path in `memory_embed_onnx.c` for experimentation without recompilation.

**Lesson from MiniLM-L12 experiment:** Doubling depth within the same model family (L6→L12, identical 32MB int8) yields almost nothing (+0pp cat-1, flat overall). The quality gain is entirely about the model family (MiniLM → BGE), not depth within the family.

4. **Cat-1 root cause:** Profile facts are rarely stated directly ("I'm single") — they surface through indirect phrasing ("tough breakup", "challenging as a single parent"). The semantic gap between question vocabulary and evidence vocabulary is too large for MiniLM-L6 to bridge reliably. bge-small closes this gap meaningfully (+4.3pp). Further improvement likely requires either (a) a larger model, or (b) memory extraction at write time — extracted facts like `{entity: "Caroline", relation: "relationship_status", value: "single"}` would match the question perfectly regardless of model quality, since the benchmark measures retrieval over raw turns while DAWN searches over extracted facts in production.

5. **Cat-3 ceiling:** Inference/counterfactual questions have a structural retrieval ceiling. Even bge-large (fp32) only reaches 65%. The evidence is present in the indexed turns, but no single turn "answers" the question — the answer requires multi-fact synthesis. The benchmark metric (fraction of annotated evidence turns in top-K) undersells DAWN's production performance here since the extracted entity profile approach would sidestep the problem entirely.

**Benchmark harness fix (May 2026):** `run_benchmark.py` had a subprocess deadlock: Python's `select` + `TextIOWrapper.readline()` with `bufsize=1` on a pipe deadlocked because the kernel reports data ready but libc holds it in a full-buffering buffer when stdout is a pipe (not a TTY). Fixed by switching to raw `os.read()` in the orchestrator and adding `setvbuf(stdout, NULL, _IOLBF, 0)` at `bench_retrieval` startup. `fflush(output_stream)` also added to `common/src/logging.c` per-line to fix the same class of fragility for any future subprocess consumer of a DAWN binary.

**Important:** Always pass `--temporal-weight 0.20` when running LoCoMo benchmarks. The orchestrator defaults to 0.0 (disabled); omitting this flag gives pre-S7 baseline numbers and makes results look like a regression.

**Analysis:**

- **LongMemEval**: 97.0% R@5 with bge-small int8 + proper-noun boost (+1.4pp over MiniLM baseline). NDCG@5 92.5% (+2.7pp). The model swap and scoring improvements are additive with the existing keyword boost advantage over academic baselines.
- **LoCoMo**: 81.6% with bge-small int8 + proper-noun boost + temporal-weight 0.20 (+7.2pp over MiniLM baseline). Temporal weight sweep (0.20/0.25/0.30) confirms 0.20 is already optimal. Cat-3 at 64.4% remains the hard ceiling for retrieval-only approaches.
- **ConvoMem**: 99.0% — no change from model swap. Ceiling effects; unlikely to improve without dataset-specific tuning.
- **Top-K depth experiment (critical finding):** At top-K=15 vs top-K=10 with bge-small int8, overall recall jumps from 81.6% to **89.9%** (+8.3pp). Cat-3 alone goes 64.4% → **77.8%** (+13.4pp). The evidence for inference questions IS being retrieved — it falls at ranks 11-15, just outside the cutoff. This strongly motivates a cross-encoder reranker: retrieve top-20, rerank to top-10. Even moderate promotion of the correct answer from rank 12 to rank 8 would capture most of this gain without increasing the LLM context window.

**Identified gaps from benchmarking (status as of May 2026):**

1. ~~**No temporal scoring**~~ → ✅ **Resolved in S7.** `time_query_parser.c` recognizes temporal expressions; additive Gaussian decay boosts time-anchored queries. LoCoMo cat-2 lifted 76.7%→79.9%, cat-3 51.5%→55.8%.
2. ~~**No fact categorization**~~ → ✅ **Resolved in S6.** 8-label taxonomy with SQL-level pre-filtering via `memory_db_fact_search_by_category()`. LLM recategorization covers ~99.8% of existing facts.
3. ~~**No relation temporality**~~ → ✅ **Resolved in S7.** Schema v33 adds `valid_from`/`valid_to` on `memory_relations` with exclusive-relation auto-close temporal awareness.
4. ~~**Weak embedding model for cat-1 profile facts**~~ → ✅ **Benchmarked, ready to ship.** `bge-small-en-v1.5 int8` (32MB, ~15ms, `Xenova/bge-small-en-v1.5`) benchmarked at +7.2pp overall, +4.5pp cat-1. Drop-in swap: change `MODEL_PATH` in `memory_embed_onnx.c`, update default in `dawn.toml`. `DAWN_ONNX_MODEL` env var override available for zero-recompile switching.
5. **Cat-3 inference questions** → Top-K=15 experiment confirms evidence IS present at ranks 11-15 (+13.4pp cat-3 at depth 15 vs 10). Cross-encoder reranker (TODO.md) is the right fix: retrieve top-20, rerank to top-10. Projected gain: +8pp overall, +13pp cat-3. LLM-in-loop synthesis or entity-profile retrieval would additionally help for the remaining ceiling after reranking.

**Running benchmarks:** See `benchmarks/README.md` for dataset download instructions and usage. Both session and turn granularities supported via `--granularity` flag. Turn-level scoring supports both official and strict modes via `--turn-scoring`. Always pass `--temporal-weight 0.20` for LoCoMo.

### 14.4 Evaluation Metrics

- Extraction precision (correct facts extracted)
- Extraction recall (facts not missed)
- RAG retrieval relevance (measured via benchmarks above)
- User satisfaction (qualitative)

### 14.5 Implementation Notes

**Duplicate detection internals:**

Facts include a `normalized_hash` column (FNV-1a hash of normalized text) for O(1)
duplicate detection at insertion time. Normalization strips stopwords, lowercases,
and sorts words before hashing. Jaccard similarity (0.7 threshold) serves as a
fuzzy fallback when hash lookup misses near-duplicates.

**Extraction concurrency:**

The extraction pipeline limits concurrent extractions via `MAX_EXTRACTION_SLOTS`
to prevent runaway LLM calls when multiple sessions end simultaneously. The slot
count respects the `max_clients` configuration.

**Embedding backfill:**

On startup, facts without embeddings are queued for background embedding via a
backfill thread. This handles facts created before embeddings were enabled, or
facts that failed embedding on first attempt. Controlled by the
`embedding_backfill_on_startup` config flag.

---

## Appendix A: Comparison to Original Proposals

### From "Adaptive Preference Learning" Proposal

- **Kept:** Preference categories, confidence scores, explicit override detection, decay
- **Changed:** Batch extraction instead of real-time, broader scope (facts + summaries + RAG)
- **Dropped:** 8% token overhead claim (now 0% runtime overhead)

### From "Building Your Own JARVIS" Presentation

- **Kept:** Sleep consolidation metaphor, facts vs. summaries separation, vector DB concepts
- **Added:** Concrete storage schema, RAG integration, configuration system
- **Resolved:** Hard problems that were hand-waved (user ID, context budget, decay rates)

---

---

## Appendix B: WebUI Wireframes

### Memory Section (Implemented)

```
┌─────────────────────────────────────────────────────────────────────┐
│  ▼ My Memory                                                    [?] │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Stats: Facts: 23  |  Prefs: 5  |  Sums: 12  |  Entities: 18      │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ [Facts]  [Preferences]  [Summaries]  [Graph]                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  🔍 Search memories...                              [Filter ▼]     │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ ● User is vegetarian                              [✏️] [🗑️] │   │
│  │   Source: explicit  |  Confidence: 95%  |  Jan 15, 2026     │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ ● User's daughter is named Dawn                   [✏️] [🗑️] │   │
│  │   Source: explicit  |  Confidence: 92%  |  Jan 10, 2026     │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ ○ User prefers concise responses                  [✏️] [🗑️] │   │
│  │   Source: inferred  |  Confidence: 78%  |  Jan 18, 2026     │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ ○ User lives in Atlanta area                      [✏️] [🗑️] │   │
│  │   Source: inferred  |  Confidence: 65%  |  Jan 12, 2026     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Showing 4 of 23 facts                              [Load More]    │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                     │
│  [Forget Everything]                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

Legend:
  ● = High confidence (>80%) - green dot
  ○ = Medium confidence (50-80%) - yellow/gray dot
  [✏️] = Edit button (expands row for inline editing)
  [🗑️] = Delete button (with confirmation)
```

### Expanded Fact Row (Edit Mode)

```
┌─────────────────────────────────────────────────────────────────────┐
│ ● User is vegetarian                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Fact text:                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ User is vegetarian                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Source: explicit     Confidence: 95%     Created: Jan 15, 2026    │
│  Last accessed: Jan 22, 2026              Access count: 7          │
│                                                                     │
│                                         [Cancel]  [Save Changes]   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Preferences Tab

```
┌─────────────────────────────────────────────────────────────────────┐
│  [Facts]  [Preferences]  [Summaries]                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Verbosity                                                   │   │
│  │  ○ Prefers concise responses               Confidence: 78%  │   │
│  │     Reinforced 3 times                            [✏️] [🗑️] │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  Humor                                                       │   │
│  │  ○ Enjoys dry humor                        Confidence: 65%  │   │
│  │     Reinforced 2 times                            [✏️] [🗑️] │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  Technical Level                                             │   │
│  │  ● Expert level explanations               Confidence: 88%  │   │
│  │     Reinforced 5 times                            [✏️] [🗑️] │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Summaries Tab

```
┌─────────────────────────────────────────────────────────────────────┐
│  [Facts]  [Preferences]  [Summaries]                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Jan 22, 2026 - 3:42 PM                              [🗑️]   │   │
│  │  Discussed home automation setup for garage lights.         │   │
│  │  Topics: home automation, mqtt, lighting                    │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  Jan 21, 2026 - 10:15 AM                             [🗑️]   │   │
│  │  Helped debug Python script for data processing. User       │   │
│  │  was working on CSV parsing with pandas.                    │   │
│  │  Topics: python, debugging, data                            │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  Jan 18, 2026 - 7:30 PM                              [🗑️]   │   │
│  │  Discussed weekend plans and restaurant recommendations     │   │
│  │  for vegetarian options in Atlanta.                         │   │
│  │  Topics: restaurants, atlanta, weekend                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Showing 3 of 12 summaries (last 30 days)           [Load More]    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Graph Tab (Entities)

```
┌─────────────────────────────────────────────────────────────────────┐
│  [Facts]  [Preferences]  [Summaries]  [Graph]                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Jon                                           [person] [🗑️] │   │
│  │  Mentions: 12  |  First: Jan 10  |  Last: Mar 3             │   │
│  │  ▸ Show relations (3)                                       │   │
│  │    → lives_in: Atlanta                                      │   │
│  │    → works_on: The OASIS Project                            │   │
│  │    → owns: Buddy                                            │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  Buddy                                           [pet] [🗑️]  │   │
│  │  Mentions: 5  |  First: Jan 15  |  Last: Feb 28             │   │
│  │  ▸ Show relations (2)                                       │   │
│  │    → is_a: golden retriever                                 │   │
│  │    ← owned_by: Jon                                          │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  Atlanta                                       [place] [🗑️]  │   │
│  │  Mentions: 3  |  First: Jan 12  |  Last: Feb 20             │   │
│  │  ▸ Show relations (1)                                       │   │
│  │    → is_in: Georgia                                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### "Forget Everything" Confirmation

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ⚠️  Warning                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  You are about to delete ALL your memories:                         │
│                                                                     │
│    • 23 facts                                                       │
│    • 5 preferences                                                  │
│    • 12 conversation summaries                                      │
│                                                                     │
│  This action cannot be undone. DAWN will no longer remember        │
│  anything about you.                                                │
│                                                                     │
│  To confirm, type DELETE below:                                     │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│                              [Cancel]  [Delete Everything]          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### RAG Documents Section (Phase 11)

```
┌─────────────────────────────────────────────────────────────────────┐
│  ▼ Knowledge Base                                               [?] │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Stats: Documents: 8  |  Chunks: 347  |  Last indexed: 2h ago      │
│  Directory: ~/Documents/dawn-knowledge  (shared)                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 📄 home_network_setup.md                    47 chunks  [🗑️] │   │
│  │    Indexed: Jan 20, 2026                                    │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ 📕 garage_door_manual.pdf                   124 chunks [🗑️] │   │
│  │    Indexed: Jan 18, 2026                                    │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ 📄 recipes.txt                              23 chunks  [🗑️] │   │
│  │    Indexed: Jan 15, 2026                                    │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ ⚠️ meeting_notes.docx                       (pending)       │   │
│  │    ████████░░░░░░░░  Indexing... 52%                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  [Scan for New Files]                           [Re-index All]     │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                     │
│  To add documents, place files in:                                  │
│  ~/Documents/dawn-knowledge                                         │
│                                                                     │
│  Supported formats: .txt, .md, .pdf, .docx                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Empty State

```
┌─────────────────────────────────────────────────────────────────────┐
│  ▼ My Memory                                                    [?] │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                         📝                                          │
│                                                                     │
│         DAWN hasn't learned anything about you yet.                │
│                                                                     │
│   As you have conversations, DAWN will remember important facts    │
│   and preferences to personalize your experience.                  │
│                                                                     │
│   You can also tell DAWN to remember things:                       │
│   "Remember that I'm allergic to shellfish"                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Design Notes

**Placement:** Memory and Documents sections appear in the settings panel after "My Settings" and before "My Sessions", grouping all personal data together.

**Confidence Indicators:**

- Green dot (●): Confidence > 80%
- Yellow/gray dot (○): Confidence 50-80%
- Dim text: Confidence < 50% (rarely shown, usually pruned)

**Mobile Considerations:**

- Edit/delete buttons always visible on touch (no hover)
- Touch targets minimum 44x44px
- Stats stack vertically on narrow screens

**Accessibility:**

- All interactive elements have visible focus states
- Confidence colors paired with text labels ("Confidence: 78%")
- Progress bars have aria-live announcements
- Delete confirmations keyboard-navigable

_This document reflects finalized design decisions as of January 2026. Implementation should follow this specification._

---

## Appendix C: Semantic Memory Search — ✅ IMPLEMENTED

> **Note**: The embedding infrastructure described below was implemented as `src/core/embedding_engine.c` — a shared module used by both memory and RAG. The implementation uses a unified API rather than the separate `embeddings/` directory originally planned. Entity embeddings live in `src/memory/memory_embeddings.c`. WebUI configuration is integrated into the existing settings panel.

### C.1 Overview

The current memory system uses **keyword-based search** (SQL LIKE queries). This works well for exact matches but misses semantic relationships:

| Query                   | Stored Fact                          | Keyword Search              | Semantic Search                        |
| ----------------------- | ------------------------------------ | --------------------------- | -------------------------------------- |
| "What's my dog's name?" | "My pet Buddy is a golden retriever" | ❌ No match (no word "dog") | ✅ Match (dog ≈ pet, golden retriever) |
| "food allergies"        | "User is allergic to shellfish"      | ❌ No match                 | ✅ Match (allergies ≈ allergic)        |
| "daughter"              | "Dawn is the user's child"           | ❌ No match                 | ✅ Match (daughter ≈ child)            |

**Semantic search** uses **embeddings** (vector representations of meaning) to find conceptually similar content even when words differ.

### C.2 Hybrid Search Architecture

Combine keyword and semantic search for best results:

```
User Query: "What's my dog's name?"
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│ Keyword Search│       │ Embed Query   │
│ (SQL LIKE)    │       │ (API call)    │
└───────┬───────┘       └───────┬───────┘
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│ Results with  │       │ Cosine        │
│ BM25-style    │       │ Similarity    │
│ ranking       │       │ Search        │
└───────┬───────┘       └───────┬───────┘
        │                       │
        └───────────┬───────────┘
                    ▼
            ┌───────────────┐
            │ Merge Results │
            │ (weighted)    │
            │ 0.3 keyword + │
            │ 0.7 vector    │
            └───────┬───────┘
                    ▼
            Return top N results
```

**Benefits:**

- **Keywords** catch exact matches ("Buddy" finds "Buddy")
- **Vectors** catch semantic matches ("dog" finds "golden retriever")
- Together they're more robust than either alone

### C.3 Embedding Provider Configuration

Mirror the LLM provider pattern for consistency:

**dawn.toml:**

```toml
[embeddings]
type = "local"                    # "local" or "cloud"

[embeddings.local]
provider = "ollama"               # "ollama" or "llama_cpp"
endpoint = "http://127.0.0.1:11434"
model = "nomic-embed-text"        # 768 dimensions, ~275MB

[embeddings.cloud]
provider = "openai"               # "openai" (fallback or primary)
model = "text-embedding-3-small"  # 1536 dimensions
```

**Provider Options:**

| Provider      | Endpoint                       | Model                  | Dimensions | Notes                 |
| ------------- | ------------------------------ | ---------------------- | ---------- | --------------------- |
| **Ollama**    | `localhost:11434/api/embed`    | nomic-embed-text       | 768        | Recommended local     |
| **llama.cpp** | `localhost:8081/v1/embeddings` | nomic-embed-text       | 768        | OpenAI-compatible API |
| **OpenAI**    | `api.openai.com/v1/embeddings` | text-embedding-3-small | 1536       | Cloud fallback        |

**Local-First Philosophy:**

- Default to Ollama if available (same service as local LLM)
- Cloud embeddings as optional fallback
- Embedding cache reduces API calls significantly

### C.4 Embedding Cache

Avoid re-embedding identical text:

```sql
CREATE TABLE embedding_cache (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    content_hash TEXT NOT NULL UNIQUE,    -- SHA256 of text
    provider TEXT NOT NULL,                -- "ollama", "llama_cpp", "openai"
    model TEXT NOT NULL,                   -- Model name
    dimensions INTEGER NOT NULL,           -- Vector size
    embedding BLOB NOT NULL,               -- Float array
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_embedding_cache_hash ON embedding_cache(content_hash);
```

**Cache Strategy:**

1. Hash incoming text (SHA256)
2. Check cache: `SELECT embedding FROM embedding_cache WHERE content_hash = ?`
3. If hit → return cached embedding (zero latency)
4. If miss → call provider API, store result, return

**Expected hit rates:**

- Same fact accessed multiple times → 100% hit
- File re-indexed without changes → 100% hit
- Typical workload → 40-60% hit rate

### C.5 Database Schema Changes

Add embedding column to memory tables:

```sql
-- Add to memory_facts (Phase 1-4 already created this table)
ALTER TABLE memory_facts ADD COLUMN embedding BLOB;

-- New index for vector search (optional, for large datasets)
-- SQLite doesn't have native vector indexes; we'll do linear scan for small N
```

**Storage Cost:**

- nomic-embed-text: 768 floats × 4 bytes = 3,072 bytes per embedding
- 1,000 facts = ~3 MB of embeddings
- text-embedding-3-small: 1,536 floats × 4 bytes = 6,144 bytes per embedding

Trivial for modern storage.

### C.6 Cosine Similarity in C

```c
float embedding_cosine_similarity(const float *a, const float *b, int dims) {
    float dot = 0.0f, norm_a = 0.0f, norm_b = 0.0f;
    for (int i = 0; i < dims; i++) {
        dot += a[i] * b[i];
        norm_a += a[i] * a[i];
        norm_b += b[i] * b[i];
    }
    if (norm_a == 0.0f || norm_b == 0.0f) return 0.0f;
    return dot / (sqrtf(norm_a) * sqrtf(norm_b));
}
```

For ~1,000 facts with 768-dim vectors, linear scan takes <1ms on modern hardware.

### C.7 API Integration

**Ollama:**

```bash
curl http://localhost:11434/api/embed \
  -d '{"model": "nomic-embed-text", "input": "What is my dog named?"}'

# Response:
{"embeddings": [[0.123, -0.456, 0.789, ...]]}
```

**llama.cpp (OpenAI-compatible):**

```bash
curl http://localhost:8081/v1/embeddings \
  -d '{"input": "What is my dog named?"}'

# Response:
{"data": [{"embedding": [0.123, -0.456, 0.789, ...]}]}
```

**OpenAI:**

```bash
curl https://api.openai.com/v1/embeddings \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{"model": "text-embedding-3-small", "input": "What is my dog named?"}'

# Response:
{"data": [{"embedding": [0.123, -0.456, 0.789, ...]}]}
```

### C.8 WebUI Configuration

Add embeddings section to settings panel (similar to LLM settings):

```
┌─────────────────────────────────────────────────────────────────────┐
│  ▼ Embeddings                                                   [?] │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Type:     [Local ▼]              Provider:  [Ollama ▼]            │
│                                                                     │
│  Model:    [nomic-embed-text ▼]                                    │
│                                                                     │
│  Endpoint: [http://127.0.0.1:11434_______________________]         │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                     │
│  Hybrid Search:  [✓] Enabled                                       │
│                                                                     │
│  Weights:        Keyword [====30%====]  Vector [=======70%=======] │
│                                                                     │
│  Cache:          [✓] Enabled    Entries: 847    Hit rate: 62%     │
│                                                                     │
│                                            [Clear Cache]           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### C.9 Implementation Phases

#### Phase S1: Embedding Infrastructure (~1 week)

- [ ] Create `include/embeddings/embeddings.h` with provider abstraction
- [ ] Create `src/embeddings/embeddings.c` with provider implementations
- [ ] Implement Ollama provider (`/api/embed` endpoint)
- [ ] Implement llama.cpp provider (OpenAI-compatible `/v1/embeddings`)
- [ ] Implement OpenAI provider (cloud fallback)
- [ ] Add `[embeddings]` section to config parser
- [ ] Add embedding cache table and CRUD operations
- [ ] Unit tests for embedding generation and cache

#### Phase S2: Memory Integration (~1 week)

- [ ] Add `embedding BLOB` column to `memory_facts` table (migration)
- [ ] Generate embeddings on fact insert (remember tool + extraction)
- [ ] Implement `embedding_cosine_similarity()` function
- [ ] Implement hybrid search in `memory_search()`
- [ ] Update `memory_action_search()` to use hybrid search
- [ ] Integration tests for semantic search

#### Phase S3: WebUI Configuration (~0.5 week)

- [ ] Add embeddings section to settings schema
- [ ] Provider/model dropdowns with dynamic population
- [ ] Hybrid search weight sliders
- [ ] Cache statistics display
- [ ] Clear cache button

#### Phase S4: RAG Integration (~0.5 week)

- [ ] Use same embedding infrastructure for RAG chunks
- [ ] Unified provider configuration (memory + RAG share settings)
- [ ] Test cross-system embedding consistency

**Total Estimate: ~3 weeks**

### C.10 Configuration Reference

**Full dawn.toml embeddings section:**

```toml
[embeddings]
enabled = true
type = "local"                    # "local" or "cloud"

[embeddings.local]
provider = "ollama"               # "ollama" or "llama_cpp"
endpoint = "http://127.0.0.1:11434"
model = "nomic-embed-text"

[embeddings.cloud]
provider = "openai"
model = "text-embedding-3-small"

[embeddings.hybrid]
enabled = true
keyword_weight = 0.3
vector_weight = 0.7
min_similarity = 0.5              # Ignore results below this threshold

[embeddings.cache]
enabled = true
max_entries = 10000               # Prune oldest when exceeded
```

### C.11 Model Recommendations

| Use Case            | Model                  | Provider | Dims | Size   | Quality   |
| ------------------- | ---------------------- | -------- | ---- | ------ | --------- |
| **Default**         | nomic-embed-text       | Ollama   | 768  | 275 MB | Excellent |
| **RAM constrained** | all-minilm             | Ollama   | 384  | 46 MB  | Good      |
| **Max quality**     | mxbai-embed-large      | Ollama   | 1024 | 670 MB | Best      |
| **Cloud fallback**  | text-embedding-3-small | OpenAI   | 1536 | N/A    | Excellent |

`nomic-embed-text` outperforms OpenAI's text-embedding-ada-002 on most benchmarks while running locally.

### C.12 Running Two Models (LLM + Embeddings)

**Q: Can I run my LLM and embedding model simultaneously?**

**With Ollama (Recommended):**
Ollama manages model loading automatically. Both models can be pulled:

```bash
ollama pull llama3.2:3b        # LLM
ollama pull nomic-embed-text   # Embeddings
```

Ollama keeps recently-used models in memory and swaps as needed. On Jetson with 8GB+ RAM, both typically fit simultaneously.

**With llama.cpp:**
Run two server instances on different ports:

```bash
# Terminal 1: LLM
llama-server --model llama-3.2-3b.gguf --port 8080

# Terminal 2: Embeddings
llama-server --model nomic-embed-text.gguf --port 8081 --embedding
```

**VRAM Budget:**
| Model | VRAM |
|-------|------|
| Llama 3.2 3B Q4_K_M | ~2.0 GB |
| nomic-embed-text | ~0.3 GB |
| **Total** | ~2.3 GB |

Leaves plenty of headroom on Jetson Orin (8-16 GB unified memory).
