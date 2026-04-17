# DAWN Memory System Design

**Status:** Phases 1-6.7, S4 Complete - Core Memory, Decay, WebUI Viewer, Import/Export, Contacts WebUI, Entity Merge, Entity Graph, Embeddings
**Date:** January 2026
**Authors:** Kris Kersey, with input from community proposals
**Last Updated:** 2026-04-08

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
- **Phases 7-11 (RAG):** ✅ Implemented — see `docs/RAG_DESIGN.md` (separate system)
- **Retrieval Benchmarks:** ✅ Complete - LongMemEval turn-level 95.4% R@5 (25.6pp above published SOTA), session-level 97.2% R@5, ConvoMem 99.0% — see `benchmarks/README.md`

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
| **ONNX**             | all-MiniLM-L6-v2       | 384        | ~5ms   | Shares ONNX Runtime with Piper TTS |

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
│     "User's daughter is named Emma" → memory_facts                  │
│                                                                      │
│  2. Extract PREFERENCES                                              │
│     "User asked for shorter responses twice" → memory_preferences   │
│                                                                      │
│  3. Extract ENTITIES + RELATIONS                                     │
│     "Kris" (person), "Bruno" (pet) → memory_entities                │
│     "Kris" → owns → "Bruno" → memory_relations                     │
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

Using SQLite (already a DAWN dependency for auth).

### 6.1 Memory Facts Table

```sql
CREATE TABLE memory_facts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT NOT NULL,
    fact_text TEXT NOT NULL,           -- "User is allergic to shellfish"
    confidence REAL DEFAULT 1.0,        -- 0.0-1.0
    source TEXT DEFAULT 'inferred',     -- 'explicit', 'inferred'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_accessed TIMESTAMP,
    access_count INTEGER DEFAULT 0,
    superseded_by INTEGER,              -- FK to newer fact if corrected
    embedding BLOB,                     -- 384 floats for semantic search (optional)
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (superseded_by) REFERENCES memory_facts(id)
);

CREATE INDEX idx_memory_facts_user ON memory_facts(user_id);
CREATE INDEX idx_memory_facts_confidence ON memory_facts(user_id, confidence DESC);
```

### 6.2 Memory Preferences Table

```sql
CREATE TABLE memory_preferences (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT NOT NULL,
    category TEXT NOT NULL,             -- 'verbosity', 'humor', 'formality', 'detail_level'
    value TEXT NOT NULL,                -- "prefers concise responses"
    confidence REAL DEFAULT 0.5,
    source TEXT DEFAULT 'inferred',     -- 'explicit', 'inferred'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    reinforcement_count INTEGER DEFAULT 1,
    FOREIGN KEY (user_id) REFERENCES users(id),
    UNIQUE(user_id, category)
);
```

### 6.3 Memory Summaries Table

```sql
CREATE TABLE memory_summaries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT NOT NULL,
    session_id TEXT NOT NULL,
    summary TEXT NOT NULL,              -- "Discussed home automation setup..."
    topics TEXT,                        -- JSON array: ["home automation", "mqtt"]
    sentiment TEXT,                     -- 'positive', 'neutral', 'frustrated'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    message_count INTEGER,
    duration_seconds INTEGER,
    consolidated BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_memory_summaries_user ON memory_summaries(user_id, created_at DESC);
```

### 6.4 Memory Entities Table

```sql
CREATE TABLE memory_entities (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    name TEXT NOT NULL,                 -- Display name (e.g., "Kris Kersey")
    entity_type TEXT NOT NULL,          -- "person", "place", "pet", "project", "thing"
    canonical_name TEXT NOT NULL,       -- Lowercase normalized (e.g., "kris kersey")
    mention_count INTEGER DEFAULT 1,
    first_seen INTEGER NOT NULL,        -- Unix timestamp
    last_seen INTEGER NOT NULL,         -- Unix timestamp
    embedding BLOB,                     -- Float array for semantic search
    embedding_norm REAL,                -- Pre-computed L2 norm
    FOREIGN KEY (user_id) REFERENCES users(id),
    UNIQUE(user_id, canonical_name)
);
```

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
    confidence REAL DEFAULT 1.0,
    created_at INTEGER NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (subject_entity_id) REFERENCES memory_entities(id),
    FOREIGN KEY (object_entity_id) REFERENCES memory_entities(id),
    FOREIGN KEY (fact_id) REFERENCES memory_facts(id)
);

CREATE INDEX idx_memory_relations_subject ON memory_relations(user_id, subject_entity_id);
CREATE INDEX idx_memory_relations_object ON memory_relations(user_id, object_entity_id);
```

### 6.6 RAG Documents Table

```sql
CREATE TABLE rag_documents (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_path TEXT NOT NULL UNIQUE,
    file_name TEXT NOT NULL,
    file_type TEXT NOT NULL,            -- 'txt', 'md', 'pdf', 'docx'
    file_hash TEXT NOT NULL,            -- SHA256 for change detection
    indexed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    chunk_count INTEGER DEFAULT 0
);
```

### 6.7 RAG Chunks Table

```sql
CREATE TABLE rag_chunks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    document_id INTEGER NOT NULL,
    chunk_index INTEGER NOT NULL,       -- Order within document
    chunk_text TEXT NOT NULL,
    embedding BLOB NOT NULL,            -- 384 floats
    token_count INTEGER,
    FOREIGN KEY (document_id) REFERENCES rag_documents(id) ON DELETE CASCADE
);

CREATE INDEX idx_rag_chunks_document ON rag_chunks(document_id);
```

**Shared Documents Model:** RAG documents are shared across all users - this is household-level knowledge (manuals, recipes, reference docs). Admin manages files manually via SSH/SFTP. No per-user isolation in v1.

**Future Enhancement:** Per-user document storage with WebUI file upload (see Phase 12+).

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
| ONNX     | all-MiniLM-L6-v2       | 384        | ~5ms   | Local, shares ONNX Runtime with TTS |

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
     Examples: "User is vegetarian", "User's daughter is named Emma"
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
    {"name": "Kris", "type": "person"},
    {"name": "Bruno", "type": "pet"},
    {"name": "Atlanta", "type": "place"}
  ],
  "relations": [
    {"subject": "Kris", "relation": "lives_in", "object": "Atlanta"},
    {"subject": "Bruno", "relation": "is_a", "object": "golden retriever"}
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
- "What did we decide about the garage?"

**Tool Definition:**

```json
{ "device": "memory", "action": "search", "value": "daughter name" }
```

**With optional date filtering:**

```json
{ "device": "memory", "action": "search", "value": "garage", "date": "2026-01-16" }
```

```json
{
   "device": "memory",
   "action": "search",
   "value": "",
   "date_from": "2026-01-13",
   "date_to": "2026-01-17"
}
```

**Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `value` | Yes | Keywords to search (can be empty if using date filter) |
| `date` | No | Specific date (YYYY-MM-DD) |
| `date_from` | No | Start of date range |
| `date_to` | No | End of date range |

The LLM translates natural language ("last Thursday") to date parameters.

**Design Principles:**

- **Unified search**: Single tool searches across facts, preferences, and summaries
- **Keyword-based**: Search value should be small keywords, not full sentences
- **Date-aware**: Can filter by date for "when did we discuss X?" queries
- **LLM interprets results**: Return matching items, let the LLM sort out relevance

**Search Implementation:**

```c
typedef struct {
   const char *keywords;      // Can be empty
   const char *date;          // NULL or "YYYY-MM-DD"
   const char *date_from;     // NULL or "YYYY-MM-DD"
   const char *date_to;       // NULL or "YYYY-MM-DD"
} memory_search_params_t;

typedef struct {
   memory_fact_t *facts;
   int fact_count;
   memory_preference_t *preferences;
   int preference_count;
   memory_summary_t *summaries;
   int summary_count;
} memory_search_result_t;

// Search across all memory tables with optional date filtering
memory_search_result_t *memory_search(const char *user_id, const memory_search_params_t *params);
```

**Search Strategy:**

1. Tokenize keywords (split on spaces) - skip if empty
2. Search `memory_facts.fact_text` for keyword matches (LIKE '%keyword%')
3. Search `memory_preferences.value` for keyword matches
4. Search `memory_summaries.summary` and `memory_summaries.topics` for keyword matches
5. Apply date filter to summaries (and facts if `created_at` matches)
6. Optionally: Use embeddings for semantic search if keyword search returns few results
7. Return top N results from each category, sorted by relevance/confidence/date

**Search Response Format:**

```json
{
   "facts": [
      { "text": "User's daughter is named Emma", "confidence": 0.95, "created": "2026-01-10" },
      {
         "text": "Decided to use Zigbee for garage automation",
         "confidence": 0.9,
         "created": "2026-01-16"
      }
   ],
   "preferences": [],
   "summaries": [
      {
         "summary": "Discussed garage automation. Decided Zigbee over Z-Wave for better range.",
         "date": "2026-01-16",
         "topics": ["home automation", "zigbee"]
      }
   ]
}
```

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

Handles time-based queries when the user wants to see what's been learned recently without specific keywords.

**Tool Definition:**

```json
{ "device": "memory", "action": "recent", "value": "24h" }
```

**Supported Time Periods:**

- Minutes: `30m`, `60m`
- Hours: `1h`, `24h`, `48h`
- Days: `1d`, `7d`, `30d`
- Weeks: `1w`, `2w`

**Design Principles:**

- **No keywords required**: Returns all memories within the time window
- **Discovery-oriented**: For "what have you learned about me lately?"
- **Reusable parser**: `parse_time_period()` in `include/tools/time_utils.h` available to other tools

**Recent Implementation:**

```c
// Parse human-readable time period into seconds
// Located in include/tools/time_utils.h for reuse
time_t parse_time_period(const char *period);  // "24h" -> 86400, "7d" -> 604800

// Get memories created within time window
char *memory_action_recent(int user_id, const char *period);
```

**Recent Response Format:**

```
RECENT FACTS:
- User prefers dark mode (explicit, 12 hours ago)
- Working on home automation project (inferred, 2 days ago)

RECENT CONVERSATIONS:
- [3 hours ago] Discussed memory system implementation...
  Topics: memory, dawn, sqlite

Total: 2 facts, 1 conversations
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

Combines two duplicate entities, reassigning all relations and contacts from source to target.

**Tool Definition:**

```json
{ "device": "memory", "action": "merge_entities", "value": "{\"source_name\":\"Kris\",\"target_name\":\"Kris Kersey\"}" }
```

**Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `source_name` | Yes | Entity to merge FROM (will be deleted) |
| `target_name` | Yes | Entity to merge INTO (survives) |

**Merge Behavior:**

1. Resolve both names to entity IDs via canonical name lookup
2. BEGIN IMMEDIATE transaction
3. Reassign all relations (subject + object references) from source to target
4. Reassign all contacts from source to target
5. Delete self-referencing relations created by reassignment
6. Deduplicate relations and contacts via ROW_NUMBER() window functions
7. Absorb mention count and time range from source
8. Delete source entity
9. COMMIT (or ROLLBACK on any error)

#### 9.1.7 Entity Graph Context

Entity graph context is automatically appended to memory search results — it is not a separate tool action. When a search returns facts, the system also queries for related entities and appends an ENTITIES section showing entity names, types, and their relationships. This provides the LLM with graph context without requiring a separate tool call.

#### 9.1.8 LLM Prompt Addition

```
For MEMORY:
- To recall: <command>{"device":"memory","action":"search","value":"keywords"}</command>
  Use short keywords (1-3 words). Returns matching facts and conversation summaries.
  Optional: Add "date":"YYYY-MM-DD" or "date_from"/"date_to" for time-based queries.
  Example: {"device":"memory","action":"search","value":"","date":"2026-01-16"}
- To see recent: <command>{"device":"memory","action":"recent","value":"24h"}</command>
  Use for time-based discovery: "24h", "7d", "1w", "30m", etc.
  Example: {"device":"memory","action":"recent","value":"1w"}
- To store: <command>{"device":"memory","action":"remember","value":"the fact"}</command>
  Use when user shares personal info or asks you to remember something.
  Phrase facts to be self-describing (include context in the text).
  Respond naturally: "I'll remember that."
```

#### 9.1.9 When to Use Each

| User Says                                | LLM Action                                               |
| ---------------------------------------- | -------------------------------------------------------- |
| "I'm vegetarian"                         | Call `remember` with "User is vegetarian"                |
| "Remember that I hate cilantro"          | Call `remember` with "User hates cilantro"               |
| "What do you know about me?"             | Call `search` with broad keywords or list injected facts |
| "Do you remember my daughter's name?"    | Call `search` with "daughter name"                       |
| "What did we talk about last Thursday?"  | Call `search` with date filter (convert to YYYY-MM-DD)   |
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
│   - Has two children: Emma and Jack                                  │
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

### Phase 6.7: Entity Merge ✅ COMPLETE

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
- [x] Entity dedup: canonical name matching prevents "Kris" vs "Kris Kersey" variants
- [x] `_Static_assert` on prepared statement layout for safety
- [x] Bulk relation loading (`memory_db_relation_list_all_by_user()`) — N+1 query fix
- [x] Entity deletion with FK-ordered relation cleanup

**Memory System Status: Phases 1-6.7 + S4/S4b Complete, RAG Complete (see `docs/RAG_DESIGN.md`)**

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

**Summary:**

| System         | Phases | Effort           |
| -------------- | ------ | ---------------- |
| Memory         | 1-6    | ~7-8 weeks       |
| RAG            | 7-11   | ~5 weeks         |
| Speaker ID     | 12     | Future           |
| Per-User Docs  | 13     | Future           |
| Advanced RAG   | 14     | Future           |
| **Total (v1)** |        | **~12-13 weeks** |

Memory and RAG can be developed in parallel by different contributors, or sequentially. Memory should be prioritized as it delivers core "assistant that knows you" value.

---

## 12. File Structure

```
include/memory/
├── memory_db.h            # CRUD for facts, prefs, summaries, entities, relations, entity merge
├── memory_embeddings.h    # Semantic embedding API (multi-provider, hybrid search)
├── memory_context.h       # Context building for LLM system prompt
├── memory_maintenance.h   # Nightly decay orchestration API
├── memory_similarity.h    # Duplicate detection API (normalize, hash, Jaccard)
├── memory_types.h         # Data structures (fact, pref, summary, entity, relation)
└── contacts_db.h          # Contact CRUD API (find, add, update, delete, list)

include/core/
└── embedding_engine.h     # Shared embedding API (ONNX, Ollama, OpenAI providers)

include/tools/
├── document_db.h          # Document/chunk CRUD (RAG storage layer)
├── document_chunker.h     # Text chunking for embedding
├── document_search.h      # RAG semantic search tool
├── document_extract.h     # PDF/DOCX/HTML text extraction
└── document_index_pipeline.h  # Shared indexing pipeline (chunk, embed, store)

src/memory/
├── memory_db.c            # SQLite CRUD, entity upsert, relation create, entity merge
├── memory_embeddings.c    # Embedding cache, hybrid search, cache invalidation
├── memory_embed_ollama.c  # Ollama embedding provider (/api/embed endpoint)
├── memory_embed_openai.c  # OpenAI embedding provider (/v1/embeddings endpoint)
├── memory_embed_onnx.c    # ONNX Runtime embedding provider (local, WordPiece tokenizer)
├── memory_context.c       # Context building for LLM system prompt
├── memory_callback.c      # LLM tool callback (9 actions: search/remember/forget/recent + contacts + merge)
├── memory_extraction.c    # LLM-based extraction: facts, prefs, entities, relations (threaded)
├── memory_maintenance.c   # Nightly decay orchestration
├── memory_similarity.c    # Duplicate detection (Jaccard, FNV-1a hashing)
└── contacts_db.c          # Contact CRUD (find, add, update, delete, list)

src/webui/
├── webui_memory.c         # WebUI memory endpoints (list, delete, stats, entities)
└── ...

www/js/ui/
├── memory.js              # Memory viewer UI (tabs, search, entity graph, delete)
└── ...

www/css/components/
├── memory.css             # Memory viewer styles (entity badges, relations, tabs)
└── ...

src/core/
└── embedding_engine.c     # Provider-agnostic embedding + vector math (shared by memory + RAG)

src/tools/
├── document_db.c          # SQLite CRUD for documents and chunks
├── document_chunker.c     # Paragraph/sentence splitting with configurable overlap
├── document_search.c      # RAG search: cosine similarity + keyword boosting
├── document_extract.c     # PDF (MuPDF), DOCX (libzip+libxml2), HTML, plain text
├── document_index_tool.c  # LLM tool for URL-based document ingestion
└── document_index_pipeline.c  # SHA-256 dedup, chunking, embedding, DB storage
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

**Blocked Patterns (Hardcoded v1):**

```c
const char *MEMORY_BLOCKED_PATTERNS[] = {
   "whenever", "always", "you should", "you must",
   "ignore", "forget", "disregard", "pretend",
   "act as if", "system prompt", "instructions",
   "password", "api key", "token", "secret",
   NULL
};
```

**Why not a dedicated safety model?**

- Enterprise systems (NVIDIA Nemotron) use 8B+ parameter safety models
- Overkill for personal home assistant on embedded hardware
- Pattern matching catches obvious attacks with zero latency
- Future enhancement: lightweight local safety classifier if needed

**Future Enhancements (Phase 14+):**

- Reranking for RAG retrieval (6-7% accuracy improvement)
- Multimodal document indexing (images/diagrams)
- Lightweight safety classifier for edge cases

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
| **LoCoMo avg recall** | — | 71.3% | 1982 QA pairs, dialog granularity, top-K=10 |
| **ConvoMem avg recall** | — | 99.0% | 100 items, per-message retrieval |

**Turn-level results (LongMemEval, matches academic evaluation methodology):**

Turn-level retrieval indexes each user turn as a separate document (~273 per question instead of ~48 sessions). Top-K=5, matching the RMM paper (Tan et al., ACL 2025, arxiv.org/abs/2503.08026). Methodology verified against the official LongMemEval evaluation code (github.com/xiaowu0162/LongMemEval).

| Scoring | R@1 | R@3 | R@5 | NDCG@5 | Evaluated |
|---------|-----|-----|-----|--------|-----------|
| **Official** (any turn from answer session) | 85.4% | 93.2% | **95.4%** | 89.7% | 500 questions |
| **Strict** (only `has_answer` turns) | 50.2% | 75.9% | **86.9%** | 69.4% | 428 questions (72 skipped) |

The official scoring is directly comparable to published baselines. The strict scoring evaluates retrieval against only the specific turns annotated as containing the answer (average 1.7 targets per question vs 11 for official).

**Comparison to published baselines (LongMemEval turn-level R@5, top-K=5):**

| System | R@5 | Model | Model Size | Method |
|--------|-----|-------|------------|--------|
| **DAWN hybrid** | **95.4%** | all-MiniLM-L6-v2 (int8) | 22M | Cosine + keyword boost |
| RMM + GTE | 69.8% | GTE-Qwen2-7B | 7B | Cosine + RL-trained reranker |
| RAG + GTE | 62.4% | GTE-Qwen2-7B | 7B | Cosine only |
| RAG + Stella | 59.2% | Stella 1.5B | 1.5B | Cosine only |
| RAG + Contriever | 54.3% | Contriever | 110M | Cosine only |

DAWN's hybrid search exceeds the published SOTA (RMM + GTE) by 25.6 percentage points while using a model 300x smaller. Results verified deterministic across two independent hardware runs (Jetson Orin and Jetson AGX Orin — identical R@5 = 0.8692 on strict scoring).

**Why keyword boosting is effective at turn-level:**

At session level, documents are 500-2000 characters (all user turns concatenated) — keyword matching has moderate discriminative power since common words appear across many sessions. At turn level, documents are 50-200 characters (single messages) — keyword matching becomes highly discriminative because short documents either contain a query word or they don't. That binary signal on top of fuzzy cosine breaks ties the embedding model alone can't resolve. The effect amplifies as document granularity decreases.

**Analysis:**

- **LongMemEval**: Strong results across granularities. The ~30 lines of keyword boost code in `document_search.c:107-175` provides the majority of the performance advantage over academic baselines.
- **LoCoMo**: Temporal inference (category 3) is the weakest area at 51.4% recall. This category asks questions like "what happened a week ago?" — DAWN currently has no timestamp-aware scoring. Category 2 (temporal facts) scores 76.4%, and category 5 (adversarial) scores 76.0%.
- **ConvoMem**: 99.0% recall on per-message retrieval demonstrates strong fine-grained search quality. The keyword boost helps surface exact-match evidence.

**Identified gaps from benchmarking:**

1. **No temporal scoring**: Queries with relative time references ("last week", "a month ago") are scored purely by semantic similarity. Adding date-aware boosting would improve LoCoMo temporal categories significantly.
2. **No fact categorization**: All facts are stored flat without domain/topic metadata. Filtering by category before scoring (analogous to searching within a specific topic) improves retrieval by up to 34% in academic benchmarks.
3. **No relation temporality**: Entity relations lack `valid_from`/`valid_to` fields, preventing "as of date X" queries.

**Running benchmarks:** See `benchmarks/README.md` for dataset download instructions and usage. Both session and turn granularities supported via `--granularity` flag. Turn-level scoring supports both official and strict modes via `--turn-scoring`.

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
│  │ ● User's daughter is named Emma                   [✏️] [🗑️] │   │
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
│  │  Kris                                          [person] [🗑️] │   │
│  │  Mentions: 12  |  First: Jan 10  |  Last: Mar 3             │   │
│  │  ▸ Show relations (3)                                       │   │
│  │    → lives_in: Atlanta                                      │   │
│  │    → works_on: The OASIS Project                            │   │
│  │    → owns: Bruno                                            │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  Bruno                                           [pet] [🗑️]  │   │
│  │  Mentions: 5  |  First: Jan 15  |  Last: Feb 28             │   │
│  │  ▸ Show relations (2)                                       │   │
│  │    → is_a: golden retriever                                 │   │
│  │    ← owned_by: Kris                                         │   │
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
| "What's my dog's name?" | "My pet Bruno is a golden retriever" | ❌ No match (no word "dog") | ✅ Match (dog ≈ pet, golden retriever) |
| "food allergies"        | "User is allergic to shellfish"      | ❌ No match                 | ✅ Match (allergies ≈ allergic)        |
| "daughter"              | "Emma is the user's child"           | ❌ No match                 | ✅ Match (daughter ≈ child)            |

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

- **Keywords** catch exact matches ("Bruno" finds "Bruno")
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
