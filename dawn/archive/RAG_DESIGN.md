# RAG Document Search — Design Document

**Created**: March 9, 2026
**Status**: Implemented (March 11, 2026; admin features March 12, 2026)
**Dependencies**: Shared embedding engine (`core/embedding_engine.*`), document parsing (`webui_documents.c`)

### Implementation Notes

Core system implemented as designed with these additions/deviations:

- **`document_read` tool added** (not in original design): Paginated document reader for full-document retrieval. Three parameters: `document` (name), `start_chunk` (offset), `count` (page size, max 20). Complements `document_search` for targeted queries.
- **Two-pass search optimization**: Cosine similarity computed for all chunks first, keyword boosting applied only to the top 50 candidates (performance optimization for large corpora).
- **ARM NEON vectorized dot product** in `embedding_engine.c` for fast cosine similarity on Jetson/ARM.
- **Thread-safe embedding**: `embedding_engine_embed()` protected by pthread mutex.
- **`result_extended` mechanism**: Large tool results (>8KB) stored via heap pointer instead of truncating into the fixed buffer.
- **Per-user document limits**: Configurable `max_index_size_kb` and `max_indexed_documents` in `[documents]` section. Exposed in WebUI settings panel as advanced fields.
- **Admin document management** (March 12, 2026):
   - "All Users" toggle in Document Library stats bar (admin-only, server-enforced)
   - Username resolution via SQL JOIN on users table (`owner_name` in `document_t`)
   - Global visibility toggle per document (SVG globe icon, owner or admin)
   - `doc_library_toggle_global` WebSocket endpoint with ownership/admin check
   - Global upload checkbox below dropzone (admin-only)
   - Audit logging for admin cross-user delete and global toggle operations
   - `user_id` only sent in list responses when admin "All Users" view is active
- **Filesystem watch directory**: Designed but not yet implemented.
- **`embedding_provider_t` typedef** moved from `memory_embeddings.h` to `core/embedding_engine.h` for shared access.
- **`document_index` tool added** (March 26, 2026): LLM can autonomously download and index documents from URLs. Supports PDF, DOCX, HTML, plain text, markdown, and code files. Includes SSRF-safe redirect handling with DNS pinning, per-user rate limiting (5/min), FlareSolverr fallback for HTTP/2 errors and JS-rendered pages, filename sanitization, and audit logging. Extraction code refactored from `webui_documents.c` into shared `document_extract` module; indexing pipeline refactored from `webui_doc_library.c` into shared `document_index_pipeline` module. 81 unit tests in `test_document_extract.c`.
- **Three ingestion paths**: WebUI upload (manual), filesystem watch (not yet implemented), and LLM-driven URL indexing (`document_index` tool).

---

## Overview

Add persistent document indexing and semantic search to DAWN. Users upload documents (PDF, DOCX, TXT, MD) via the WebUI or drop them in a filesystem directory. DAWN chunks the text, generates embeddings, and stores them in SQLite. When the LLM calls the `document_search` tool, relevant chunks are retrieved via hybrid keyword + vector search and returned as tool results with source citations.

### Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Embedding infrastructure | Shared with memory system | No code duplication; extract common layer |
| Document organization | Flat per-user store, optional global flag | Collections deferred; keeps v1 simple |
| Ingestion paths | WebUI upload + filesystem watch directory | Filesystem = auto-global; WebUI = per-user with global checkbox |
| Context injection | Standard `tool_result` with `result_extended` | Fits existing tool loop; extended pointer handles large results cleanly |
| Result limits | Configurable `max_chunks` and `max_context_tokens` | Truncation notice in result tells LLM what was omitted |

---

## Architecture

### Shared Embedding Layer

The memory system already has production-grade embedding infrastructure. Rather than duplicating it, extract the reusable parts into a shared layer:

```
Current:
  memory_embeddings.c  →  embeds facts, searches facts, caches fact vectors

Target:
  embedding_engine.c   →  model loading, embed(), cosine(), cache management
  memory_embeddings.c  →  calls embedding_engine for fact-specific operations
  document_embeddings.c →  calls embedding_engine for chunk-specific operations
```

**Shared API (`include/core/embedding_engine.h`):**

```c
/// Initialize the embedding engine (model loading, provider selection).
/// Called once at startup. Memory and document systems both use this.
int embedding_engine_init(void);
void embedding_engine_cleanup(void);
bool embedding_engine_available(void);

/// Generate embedding for text. Thread-safe.
int embedding_engine_embed(const char *text, float *out_vec, int max_dims, int *out_dims);

/// Math utilities
float embedding_engine_cosine(const float *a, const float *b, int dims);
float embedding_engine_cosine_with_norms(const float *a, const float *b,
                                          int dims, float norm_a, float norm_b);
float embedding_engine_l2_norm(const float *vec, int dims);
```

Functions currently in `memory_embeddings.c` that move to `embedding_engine.c`:
- `memory_embeddings_init()` → `embedding_engine_init()` (model loading, provider selection)
- `memory_embeddings_embed()` → `embedding_engine_embed()`
- `memory_embeddings_cosine()` → `embedding_engine_cosine()`
- `memory_embeddings_l2_norm()` → `embedding_engine_l2_norm()`
- Provider dispatch (ONNX local, Ollama, OpenAI-compatible)

Functions that stay in `memory_embeddings.c`:
- `memory_embeddings_hybrid_search()` (fact-specific ranking)
- `memory_embeddings_embed_and_store()` (fact-specific storage)
- Fact/entity cache management
- Background backfill thread for facts

### Document-Specific Layer

**New file: `src/tools/document_search.c`** — tool registration, chunking, search

**New file: `src/tools/document_db.c`** — SQLite CRUD for documents and chunks

**New file: `include/tools/document_search.h`** — public API

---

## Database Schema

All tables in the existing `auth.db` database (shared handle via `auth_db`).

```sql
-- Document metadata
CREATE TABLE documents (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,                   -- NULL = global (filesystem-ingested)
    filename TEXT NOT NULL,
    filepath TEXT NOT NULL,            -- Original path (for re-indexing)
    filetype TEXT NOT NULL,            -- pdf, docx, txt, md
    file_hash TEXT NOT NULL,           -- SHA-256 of file content (dedup)
    num_chunks INTEGER NOT NULL,
    is_global INTEGER DEFAULT 0,       -- 1 = accessible to all users
    created_at INTEGER NOT NULL,
    FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Text chunks with embeddings
CREATE TABLE document_chunks (
    id INTEGER PRIMARY KEY,
    document_id INTEGER NOT NULL,
    chunk_index INTEGER NOT NULL,      -- Order within document
    text TEXT NOT NULL,
    embedding BLOB NOT NULL,           -- 384-dim float vector (1536 bytes)
    embedding_norm REAL NOT NULL,      -- Pre-computed L2 norm for fast cosine
    FOREIGN KEY(document_id) REFERENCES documents(id) ON DELETE CASCADE
);

CREATE INDEX idx_doc_chunks_doc ON document_chunks(document_id);
CREATE INDEX idx_documents_user ON documents(user_id);
CREATE INDEX idx_documents_hash ON documents(file_hash);
```

**Note**: The roadmap specified 768-dim embeddings. The actual model (`all-MiniLM-L6-v2-int8.onnx`) produces 384-dim vectors. Schema matches reality.

**No collections table.** Documents belong to a user (or are global). Collections can be added later as a `collection_id` FK if needed.

---

## Text Chunking

### Algorithm

```
1. Parse document → raw text (reuse existing parsers from webui_documents.c)
2. Split on paragraph boundaries (double newline)
3. If paragraph > max_chunk_tokens: split on sentence boundaries
4. Merge small consecutive chunks until target size reached
5. Add overlap (configurable, default 50 tokens) between chunks
```

### Configuration

```toml
[documents]
# Chunking
chunk_target_tokens = 500       # Target chunk size
chunk_max_tokens = 1000         # Hard maximum before forced split
chunk_overlap_tokens = 50       # Overlap between consecutive chunks

# Search
max_search_results = 5          # Default chunks returned per query
max_context_tokens = 2000       # Token budget for tool result

# Filesystem watch
watch_directory = ""            # Empty = disabled. e.g., "/home/user/dawn-docs"
watch_interval_sec = 60         # How often to scan for new/changed files
```

### Sentence Boundary Detection

The existing `sentence_buffer.c` handles streaming sentence boundaries for TTS. For batch chunking, implement a simpler function that splits on `.!?` followed by whitespace, respecting abbreviations (Mr., Dr., etc.) and decimal numbers. This is a separate function — don't couple it to the streaming TTS buffer.

---

## Tool Registration

Follows the Home Assistant pattern.

```c
static const treg_param_t doc_search_params[] = {
   {
       .name = "query",
       .description = "The search query — what you want to find in the user's documents",
       .type = TOOL_PARAM_TYPE_STRING,
       .required = true,
       .maps_to = TOOL_MAPS_TO_VALUE,
   },
};

static const tool_metadata_t doc_search_metadata = {
   .name = "document_search",
   .device_string = "document search",
   .description = "Search the user's uploaded documents for relevant information. "
                  "Use this when the user asks about content they've previously uploaded "
                  "(PDFs, manuals, notes, etc.). Returns relevant excerpts with source "
                  "citations. Do NOT use this for general web searches.",
   .params = doc_search_params,
   .param_count = 1,
   .device_type = TOOL_DEVICE_TYPE_GETTER,
   .capabilities = 0,  /* No network needed — local embeddings + SQLite */
   .is_getter = true,
   .config = &s_doc_config,
   .config_size = sizeof(s_doc_config),
   .config_parser = doc_parse_config,
   .config_writer = doc_write_config,
   .config_section = "documents",
   .is_available = doc_search_is_available,
   .init = doc_search_init,
   .cleanup = doc_search_cleanup,
   .callback = doc_search_callback,
};
```

### Availability

`is_available` returns true when `embedding_engine_available()` is true AND at least one document exists for the current user (or globally). If no documents are indexed, the tool doesn't appear in the LLM's tool list — no point searching an empty store.

---

## Search Flow

```
1. LLM calls document_search(query="wiring specifications")
2. Callback:
   a. Embed the query via embedding_engine_embed()
   b. Load all chunk embeddings for this user + global docs
   c. Compute cosine similarity for each chunk
   d. Keyword boost: if query terms appear literally in chunk text, boost score
   e. Rank by combined score, take top max_search_results
   f. Format results with source citations
   g. If total tokens > max_context_tokens, truncate and add notice
   h. Return via result_extended (see below)
3. LLM receives chunks, incorporates into response with citations
```

### Result Format

```
DOCUMENT SEARCH RESULTS (showing 3 of 7 matches, 1847 tokens):

[1] (score: 0.89) manual.pdf, page 12:
The wiring harness connects to the main distribution panel via a 30-amp
breaker. Use 10-gauge THHN wire for runs up to 50 feet...

[2] (score: 0.84) manual.pdf, page 14:
For 240V installations, a dedicated circuit with appropriate wire gauge
is required. Consult local electrical codes for minimum requirements...

[3] (score: 0.71) project-notes.txt:
Remember to check NEC 2023 Table 310.16 for ampacity ratings before
selecting wire gauge for the workshop circuit...

[4 additional matches omitted — 1,203 tokens over budget. User can ask for more detail.]
```

When truncated, the notice tells the LLM both the count omitted and why, so it can inform the user or request more.

---

## Extended Tool Results (`result_extended`)

### Problem

`tool_result_t.result` is a fixed `char[8192]` buffer. All tool callbacks return `char*` which is copied via `safe_strncpy(result->result, cb_result, LLM_TOOLS_RESULT_LEN)`. If the callback returns more than 8192 bytes, the tail is **silently truncated** — the LLM receives a result that may end mid-sentence with no indication of data loss.

This currently affects `search_tool.c` (allocates 12288, formats into it, truncated to 8192 at copy) and will affect `document_search` (chunks can easily exceed 8192).

### Solution

Add an optional `result_extended` pointer to `tool_result_t`:

```c
typedef struct {
   char tool_call_id[LLM_TOOLS_ID_LEN];
   char result[LLM_TOOLS_RESULT_LEN];    /* Fixed buffer (default path) */
   char *result_extended;                 /* Malloc'd large result (preferred if non-NULL) */
   bool success;
   bool skip_followup;
   bool should_respond;
   char *vision_image;
   size_t vision_image_size;
} tool_result_t;
```

**Rules:**
- If `result_extended != NULL`, the formatting functions (`llm_tools_add_results_openai`, `llm_tools_add_results_claude`) use it instead of `result[]`
- Caller (tool loop) frees `result_extended` after formatting
- Tools that don't need it leave it NULL — zero change to existing tools
- The fixed `result[]` buffer remains for simple tools (calculator, datetime, etc.)

### Changes Required

| File | Change |
|------|--------|
| `include/llm/llm_tools.h` | Add `char *result_extended` to `tool_result_t` |
| `src/llm/llm_tools.c` | In `llm_tools_add_results_openai()` and `llm_tools_add_results_claude()`: use `result_extended ?: result` as content source |
| `src/llm/llm_tools.c` | In `llm_tools_execute_from_treg()`: if callback returns a string longer than `LLM_TOOLS_RESULT_LEN`, store in `result_extended` instead of truncating. Copy first 8K to `result[]` as fallback. |
| `src/llm/llm_tool_loop.c` | Free `result_extended` after history append |
| `src/tools/search_tool.c` | Remove `SEARCH_RESULT_BUFFER_SIZE`; tool now returns its full result without worrying about truncation |

### Backward Compatibility

- `memset(result, 0, sizeof(*result))` already zeros the struct — `result_extended` starts NULL
- Existing tools return strings < 8192 bytes — no change in behavior
- `result[]` is still populated (truncated copy) as a safety net for any code that reads it directly

### Fix for Current Silent Truncation

Until `result_extended` is implemented, add a truncation marker in `llm_tools_execute_from_treg()`:

```c
if (cb_result && strlen(cb_result) >= LLM_TOOLS_RESULT_LEN) {
   safe_strncpy(result->result, cb_result, LLM_TOOLS_RESULT_LEN);
   /* Overwrite last 60 bytes with truncation notice */
   const char *notice = "\n\n[Result truncated — original was longer]";
   size_t nlen = strlen(notice);
   memcpy(result->result + LLM_TOOLS_RESULT_LEN - nlen - 1, notice, nlen + 1);
   LOG_WARNING("Tool result truncated: %zu bytes → %d", strlen(cb_result), LLM_TOOLS_RESULT_LEN);
}
```

This is a stopgap until `result_extended` is implemented.

---

## Ingestion Paths

### Path 1: WebUI Upload

User uploads a document via the WebUI. Reuse the existing upload infrastructure from `webui_documents.c` (which currently handles one-shot document injection into conversations).

**Flow:**
1. User uploads file via new "Document Library" panel in WebUI
2. Server receives file, validates type and size
3. Parse text (MuPDF for PDF, libzip+libxml2 for DOCX, direct read for TXT/MD)
4. Chunk text
5. Generate embeddings for each chunk (can be slow — run async, show progress)
6. Store document + chunks + embeddings in SQLite
7. SHA-256 hash of file content prevents duplicate uploads
8. User can toggle "Global" checkbox — sets `is_global = 1`

**File size limit**: Configurable, default 10MB. Configurable in `[documents] max_file_size_mb`.

### Path 2: Filesystem Watch Directory

Daemon monitors a configured directory for new/changed files. All filesystem-ingested documents are automatically global (no user association).

**Flow:**
1. On startup and every `watch_interval_sec`, scan `watch_directory` recursively
2. For each file: compute SHA-256 hash
3. If hash not in `documents` table → ingest (parse, chunk, embed, store with `user_id = NULL, is_global = 1`)
4. If hash matches existing document → skip
5. If file is deleted → optionally remove from index (configurable: `watch_auto_delete = false`)

**Supported extensions**: `.pdf`, `.docx`, `.txt`, `.md`

**Note**: Filesystem ingestion runs on a background thread. Embedding generation is CPU-intensive (~50ms per chunk on Jetson with ONNX) — a 100-chunk document takes ~5 seconds.

---

## WebUI

### Document Library Panel

New panel in WebUI (similar to Memory Viewer):

- **Upload area**: Drag-and-drop or file picker
- **Document list**: Filename, type, chunk count, date uploaded, global toggle
- **Delete button**: Removes document + all chunks
- **Status indicators**: "Indexing..." with progress for large documents
- **Storage stats**: Total documents, total chunks, estimated embedding storage size

### Admin View

Admins see all documents (all users + global). Regular users see only their own + global documents.

---

## Access Control

```
Query: "Find chunks relevant to this query"
  → Return chunks from:
     1. Documents where user_id = current_user_id
     2. Documents where is_global = 1
  → Never return chunks from another user's private documents
```

SQL:
```sql
SELECT c.text, c.embedding, c.embedding_norm, d.filename, d.filetype, c.chunk_index
FROM document_chunks c
JOIN documents d ON c.document_id = d.id
WHERE d.user_id = ? OR d.is_global = 1
```

---

## Configuration

### dawn.toml

```toml
[documents]
# Chunking
chunk_target_tokens = 500
chunk_max_tokens = 1000
chunk_overlap_tokens = 50

# Search
max_search_results = 5
max_context_tokens = 2000

# Filesystem watch (empty = disabled)
watch_directory = ""
watch_interval_sec = 60
watch_auto_delete = false

# Limits
max_file_size_mb = 10
max_documents_per_user = 100
max_total_chunks = 10000
```

### WebUI Settings

Expose `max_search_results`, `max_context_tokens`, `watch_directory`, and `max_file_size_mb` in the admin settings panel.

---

## Implementation Plan

### Week 1: Core Infrastructure

| Day | Task |
|-----|------|
| 1 | Extract `embedding_engine.c` from `memory_embeddings.c`. Update memory system to call shared layer. Verify existing memory tests pass. |
| 2 | Create `document_db.c` — SQLite schema, CRUD operations (create document, create chunks, query chunks by user, delete document). |
| 3 | Implement text chunking — paragraph split, sentence split, overlap, merge small chunks. Unit test with sample documents. |
| 4 | Implement `document_search.c` — tool registration, embedding query, cosine similarity search, result formatting with token budget and truncation notice. |
| 5 | Implement `result_extended` in `llm_tools.h` / `llm_tools.c`. Fix silent truncation in search tool. |

### Week 2: Integration + WebUI

| Day | Task |
|-----|------|
| 1 | WebUI upload endpoint — reuse `webui_documents.c` upload path, add persistent storage path. |
| 2 | WebUI Document Library panel — upload, list, delete, global toggle. |
| 3 | Filesystem watch directory — background thread, hash-based change detection, auto-ingest. |
| 4 | Integration testing — upload PDF, ask questions, verify citations. Test with multiple users, global docs, token truncation. |
| 5 | Configuration — TOML section, WebUI settings fields, CMake feature guard (`DAWN_ENABLE_DOCUMENT_SEARCH`). |

---

## File Manifest

### New Files

| File | Purpose | Est. Lines |
|------|---------|------------|
| `include/core/embedding_engine.h` | Shared embedding API | ~60 |
| `src/core/embedding_engine.c` | Model loading, embed(), cosine(), provider dispatch | ~400 (extracted from memory_embeddings.c) |
| `include/tools/document_search.h` | Document search public API | ~30 |
| `src/tools/document_search.c` | Tool registration, chunking, search callback | ~500 |
| `src/tools/document_db.c` | SQLite CRUD for documents and chunks | ~300 |
| `include/tools/document_db.h` | Document DB public API | ~60 |
| `www/js/ui/documents-library.js` | WebUI document library panel | ~300 |
| `www/css/components/documents-library.css` | Document library styles | ~100 |

### Modified Files

| File | Change |
|------|--------|
| `src/memory/memory_embeddings.c` | Remove embedding engine code, call `embedding_engine_*` instead |
| `include/memory/memory_embeddings.h` | Remove engine-level functions (keep memory-specific API) |
| `include/llm/llm_tools.h` | Add `result_extended` to `tool_result_t` |
| `src/llm/llm_tools.c` | Use `result_extended` in formatting functions, fix silent truncation |
| `src/llm/llm_tool_loop.c` | Free `result_extended` after use |
| `src/tools/tools_init.c` | Register `document_search` tool |
| `cmake/DawnTools.cmake` | Add `DAWN_ENABLE_DOCUMENT_SEARCH` option |
| `www/index.html` | Add document library panel container |
| `src/webui/webui_http.c` | Add `/api/documents/library/*` endpoints |

---

## Testing

### Unit Tests

| Test | Assertions | What |
|------|-----------|------|
| `test_document_chunker` | 34 (10 tests) | Paragraph split, sentence split, overlap, merge, edge cases (empty doc, single sentence, very long paragraph) |
| `test_document_db` | 65 (14 tests) | CRUD, user isolation, global flag, hash dedup, cascade delete, find_by_name, chunk read, pagination, update_global, invalid params |
| `test_embedding_engine` | 24 (9 tests) | L2 norm (basic + edge), cosine similarity (identical, orthogonal, opposite, pre-computed norms, edge cases), 384-dim vectors, uninitialized state |

All 123 assertions passing (2026-03-12). Build: `make -C build-debug test_document_chunker test_document_db test_embedding_engine`

### Integration Tests

| Test | What |
|------|------|
| Upload PDF → search → get relevant chunks | End-to-end with source citations |
| Upload as user A → search as user B → no results | User isolation |
| Upload as global → search as any user → results | Global access |
| Upload duplicate (same hash) → rejected | Dedup |
| Token truncation → LLM sees notice | Result formatting |
| Filesystem watch → auto-ingest → searchable | Background ingestion |

---

## Future Considerations (Not in v1)

- **Named collections** — group documents by topic, search within a collection
- **Re-indexing** — re-chunk and re-embed when model or chunk settings change
- **OCR** — extract text from scanned PDFs (Tesseract)
- **Image extraction** — charts and figures from documents (ties into vision)
- **Incremental updates** — append pages to existing document without full re-index
- **FAISS index** — for collections with 10k+ chunks where brute-force cosine is too slow
