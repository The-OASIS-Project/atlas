# Embedding Upgrade — bge-small Migration + Recompute Worker

**Status**: shipped (May 2026). Schema v41.

**What shipped:** model swap from `all-MiniLM-L6-v2-int8` (22MB, ~8ms/embed) → `bge-small-en-v1.5-int8` (32MB, ~15ms/embed). Both 384-dim. Benchmarking showed +7.9pp LoCoMo overall, +1.4pp LongMemEval R@5. Three supporting pieces shipped alongside: a tech-debt cache-invalidation fix, an ID-based extraction filter (replacing the count-based cursor), and a per-user embedding recompute worker (gates on `embeddings_model_id` mismatch, re-embeds facts → entities → document chunks per user).

**Cross-encoder reranker (originally Feature 2 of this plan):** investigated and reverted after empirical results showed no net benefit on conversational data and only marginal lift on LongMemEval at ~10× retrieval latency. The full investigation, integration shape, and decision rationale live in [RERANKER_INVESTIGATION.md](RERANKER_INVESTIGATION.md). Kept artifacts: shared WordPiece tokenizer (`memory_embed_tokenizer`) and the `rerank_shootout.py` test harness.

**Model files on disk** (`models/embeddings/`):
- `bge-small-en-v1.5-int8.onnx` (32MB) — current default
- `bge-small-en-v1.5.onnx` (127MB fp32) — benchmark reference, not used in production
- `bge-base-en-v1.5.onnx`, `bge-large-en-v1.5.onnx` — benchmark reference only
- `all-MiniLM-L6-v2-int8.onnx` (22MB) — previous default, kept for rollback
- `vocab.txt` — shared BERT WordPiece vocabulary, used by all models above

---

## Tech Debt: `memory_embeddings_embed_and_store_entity()` Missing Cache Invalidation

**Surfaced by**: architecture review of the original plan (May 2026).

**Location**: `src/memory/memory_embeddings.c`. The entity wrapper did not call `memory_embeddings_invalidate_entity_cache()` after updating an entity embedding. The parallel function for facts (`memory_embeddings_embed_and_store()`) did call its cache invalidation correctly.

**Impact**: any code path that used the entity wrapper to embed a new entity left `s_entity_cache` stale until something else invalidated it. Entity-based recall could return outdated vectors until the next cache-busting operation. Low-severity in practice (entity embeddings update infrequently) but a latent correctness bug.

**Fix**: one-line addition to `memory_embeddings_embed_and_store_entity()`:
```c
memory_embeddings_invalidate_entity_cache();
```
after the `memory_db_entity_update_embedding()` call. The recompute worker (Feature 1 below) explicitly calls this function anyway, sidestepping the bug — but the wrapper was fixed regardless so callers can trust it.

---

## Feature 3: ID-Based Extraction Filter

### Problem (pre-fix)

The extraction filter used a count-based cursor (`messages_seen > last_extracted`) where `messages_seen` incremented only for non-system messages, but `memory_db_set_last_extracted()` stored the conversation's total `message_count` including system + tool messages. On conversations with mixed role sequences, the two counters drifted and the final new message could be silently skipped on the next extraction pass.

The provenance work (schema v40, shipped May 2026) added `last_extracted_msg_id` as an ID-based cursor and used it correctly for provenance range computation. The extraction filter itself was not updated — it still used the old count-based logic.

### Design

#### 3.1 Plumb `msg.id` into history entries

For an ID-based filter to work, each in-memory message entry needs an `"id"` field. Two paths produce history entries:

- **Recovery path** (`memory_recovery.c::append_message_to_history`): added `json_object_object_add(entry, "id", json_object_new_int64(msg->id))`. The `msg->id` field is already in scope via `message_callback_t`.
- **Live session paths** (`session_manager.c` voice path and `webui_history.c` × 3 sites): `conv_db_add_message()` returns `int` (SUCCESS/FAILURE), not the row ID. The ID is obtained via `sqlite3_last_insert_rowid(s_db.db)` immediately after the call (requires `#define AUTH_DB_INTERNAL_ALLOWED` — both files already define this). The ID is stamped into the in-memory history entry before continuing.

Sessions with `conv_id == 0` (voice-only, no persistence) omit the `"id"` field entirely; the filter treats missing or zero id as "always include."

#### 3.2 Filter rewrite

The new filter at `memory_extraction.c`:

```c
int64_t last_msg_id = 0;
memory_db_get_last_extracted_msg_id(conversation_id, &last_msg_id);

struct json_object *filtered = json_object_new_array();
for (size_t i = 0; i < arr_len; i++) {
   struct json_object *msg = json_object_array_get_idx(conversation_history, i);
   struct json_object *role_obj, *id_obj;
   if (!json_object_object_get_ex(msg, "role", &role_obj)) continue;
   if (strcmp(json_object_get_string(role_obj), "system") == 0) continue;
   int64_t msg_id = 0;
   if (json_object_object_get_ex(msg, "id", &id_obj))
      msg_id = json_object_get_int64(id_obj);
   if (msg_id == 0 || msg_id > last_msg_id)
      json_object_array_add(filtered, json_object_get(msg));
}
```

#### 3.3 Early-skip gate

Replaced `message_count <= last_extracted` with an ID-based gate:

```c
int64_t last_msg_id = 0;
memory_db_get_last_extracted_msg_id(conversation_id, &last_msg_id);
int64_t max_msg_id = 0;
conv_db_get_max_msg_id(conversation_id, user_id, &max_msg_id);
if (max_msg_id > 0 && max_msg_id <= last_msg_id) {
   LOG_INFO("memory_extraction: skipping — no new messages (max_id=%lld, last=%lld)",
            (long long)max_msg_id, (long long)last_msg_id);
   return SUCCESS;
}
```

The `last_extracted_msg_count` write in `memory_db_set_last_extracted` was kept for one release cycle for back-compat verification — to be removed in a follow-up once the ID cursor is proven stable.

### Test coverage

Four regression tests in `tests/test_memory_extraction.c`:
1. `[system, user, assistant, tool, assistant]` + one new user message → exactly the new user message extracted.
2. `[user, user, user]` + one new user → no skip.
3. Two sequential extractions → second's window starts after first's `msg_id_end`, no overlap and no gap.
4. Voice-only path (`conv_id == 0`, no `"id"` field on any history entry) → all non-system messages included unconditionally.

### Files modified

| File | Action |
|---|---|
| `src/memory/memory_recovery.c` | Add `"id"` field to history entries in `append_message_to_history` |
| `src/core/session_manager.c` | Stamp message ID into in-memory history entry after `conv_db_add_message` |
| `src/webui/webui_history.c` | Same — 3 call sites |
| `src/memory/memory_extraction.c` | Replace count-based filter + early-skip with ID-based equivalents |
| `tests/test_memory_extraction.c` | New regression tests |

---

## Feature 1: Embedding Recomputation on Model Change

### Problem (pre-fix)

When the daemon restarted with a new embedding model, all stored embeddings were produced by the previous model. Cosine similarity between a new-model query vector and an old-model stored vector produces meaningless results. Both models being 384-dim meant nothing crashed, but retrieval quality degraded silently and permanently. DAWN had no mechanism to detect or recover from this.

Embeddings live as `embedding BLOB` columns on the data tables (not in separate embedding tables):
- `memory_facts.embedding` — 384 × float32 per fact
- `memory_entities.embedding` — 384 × float32 per entity
- `document_chunks.embedding` — 384 × float32 per RAG chunk

The pre-existing `backfill_thread_fn` in `memory_embeddings.c` only handled the case where `embedding IS NULL OR length(embedding)/4 != ?` — a same-dimension model swap was invisible to its selector.

### Design

#### 1.1 Model identity: single system row

Rather than adding `model_hash` columns to every row, store a single model identity string in a new `system_metadata` key/value table:

```sql
CREATE TABLE IF NOT EXISTS system_metadata (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
```

On startup, compare `system_metadata.embedding_model_id` against `g_config.memory.model_id`. If they differ, flag all users for recomputation.

Identity string set via a new config key:
```toml
[memory.embeddings]
model_id = "bge-small-en-v1.5-int8"  # bump this when changing MODEL_PATH
```

**Why not SHA-256 of the model file**: SHA-256 of a 32MB file costs 50-100ms on cold eMMC at every startup. A human-readable model ID string is sufficient — the realistic threat model is an operator changing `MODEL_PATH`; they can also bump `model_id`. The existing dim probe in `embedding_engine_init()` catches the dimension-mismatch case independently.

#### 1.2 Per-user gate (mirrors existing pattern)

Added a column to `users` (mirroring the established `categories_backfilled_at` pattern):

```sql
ALTER TABLE users ADD COLUMN embeddings_model_id TEXT DEFAULT NULL;
```

A user's embeddings are stale when `users.embeddings_model_id != g_config.memory.model_id` (or IS NULL). Scopes work per-user, avoids cross-user cache thrash.

Schema migration: **v41** — `system_metadata` table creation + `users.embeddings_model_id` column. Wrapped in `BEGIN IMMEDIATE` / `COMMIT` per v40 precedent. No changes to `memory_facts`, `memory_entities`, or `document_chunks`.

#### 1.3 Recompute worker

New module `src/memory/memory_embed_recompute.c` / `include/memory/memory_embed_recompute.h`. Public API:

```c
int  memory_embed_recompute_start(void);
void memory_embed_recompute_stop(void);
bool memory_embed_recompute_in_progress(void);
int  memory_embed_recompute_status(int64_t *done_out, int64_t *total_out);
```

Worker thread (nice 10) processes per user in priority order:
1. `memory_facts` — most critical for memory recall quality
2. `memory_entities` — entity graph lookups
3. `document_chunks` — RAG; may be very large, runs as a separate lower-priority pass after facts/entities complete

Per-batch loop:
1. SELECT 50 rows for one user.
2. Re-embed text via `embedding_engine_embed()`.
3. Update `embedding BLOB` via `memory_db_fact_update_embedding()` / `memory_db_entity_update_embedding()`.
4. **Explicitly call `memory_embeddings_invalidate_cache()` after each fact batch and `memory_embeddings_invalidate_entity_cache()` after each entity batch.** The DB helpers do NOT call these functions — only the higher-level `memory_embeddings_embed_and_store[_entity]()` wrappers do. The worker invokes the invalidate functions directly between batches so concurrent searches see fresh vectors.
5. Sleep 100ms between batches.
6. On completion of **both** the facts pass and the entities pass: UPDATE `users.embeddings_model_id = g_config.memory.model_id`. A crash between the two passes leaves the user's row NULL, triggering a full recompute on next start — correct behaviour.

#### 1.4 Coordination with the existing backfill thread

The new recompute worker gates on `embeddings_model_id` mismatch; the existing backfill gates on NULL/wrong-dim. Mutually exclusive by trigger condition. On startup:
1. Check `system_metadata.embedding_model_id` vs config.
2. If mismatch → launch recompute worker (skips backfill for that user until complete).
3. If match → launch backfill as before (NULL/wrong-dim only).

#### 1.5 Lock ordering

Worker holds the embed mutex (for embedding) and the auth_db mutex (for batch updates). These are leaf-equivalent; never held simultaneously:

```
acquire embed_mutex → embed → release embed_mutex
acquire auth_db_mutex → UPDATE → release auth_db_mutex
```

Auth_db is a leaf lock per ARCHITECTURE.md ("copy data out, release the lock, then continue"). This matches the pattern in `memory_embed_onnx.c` — the ONNX `Run()` call happens outside any auth_db lock.

#### 1.6 Document chunks: deferred second pass

`document_chunks` may have thousands of rows (10k+ rows × 15ms/embed ≈ 2.5+ min background time). Runs as a separate `memory_embed_recompute_chunks_start()` launched after the facts/entities pass completes. Lower priority; tolerable quality degradation during transition since RAG search falls back gracefully when cosine scores are low.

Progress tracking accepts the simpler approach of **always re-running until complete**. Because `users.embeddings_model_id` is updated only after facts + entities finish (not after chunks), and the chunk worker starts fresh on each daemon restart until done, there is no risk of marking a user as fully current while chunks are still stale. If the chunk pass is interrupted, it restarts from the beginning on next launch — processing is idempotent.

#### 1.7 Shutdown ordering

Recompute worker stopped in the same block as `memory_recovery_stop()` in `src/dawn.c`, **before** `auth_db_close()`. A long embed step (15ms) may be in flight; the stop flag is checked between batches, not inside an embed call.

#### 1.8 Config

```toml
[memory.embeddings]
model_id = "bge-small-en-v1.5-int8"      # bump when changing MODEL_PATH
recompute_on_model_change = true          # default true
recompute_batch_size = 50                 # rows per batch
recompute_batch_sleep_ms = 100            # sleep between batches
```

`recompute_on_model_change = false` suppresses the worker (ops escape hatch; logs a warning). If `embedding_backfill_on_startup = false`, recompute is also suppressed for consistency.

#### 1.9 Graceful degradation during transition

While recomputation runs, search continues against mixed-model embeddings. Results are degraded but not broken. Log lines on start ("Embedding model changed: re-indexing N facts for user X") and on completion are sufficient for v1. Admin socket extension (`memory_embed_recompute_status`) is a future enhancement.

### Live result

First-boot recompute on production:
- 2,183 embeddings re-indexed across 3 users (facts + entities)
- 243 document chunks re-indexed in the deferred pass
- Architecture review surfaced and fixed two HIGH bugs: chunk pass ordering, partial-failure mark-complete

CI test suite grew 38 → 40 (new `test_memory_extraction` and `test_embed_recompute`, 14 new assertions).

### Files created or modified

| File | Action |
|---|---|
| `include/memory/memory_embed_recompute.h` | Create |
| `src/memory/memory_embed_recompute.c` | Create (`#define AUTH_DB_INTERNAL_ALLOWED` for custom SELECTs) |
| `src/auth/auth_db_core.c` | Schema v41: `system_metadata` + `users.embeddings_model_id` |
| `include/config/dawn_config.h` | Add `model_id[64]`, `recompute_on_model_change`, `recompute_batch_size`, `recompute_batch_sleep_ms` |
| `src/config/config_defaults.c` | Defaults |
| `src/config/config_parser.c` | TOML parsing |
| `src/config/config_validate.c` | `VALIDATE_RANGE_INT` for batch_size, sleep_ms |
| `src/dawn.c` | Start recompute worker after `embedding_engine_init()`; stop before `auth_db_close()` |
| `tests/test_embed_recompute.c` | Unity tests |
| `www/js/ui/settings.js` | `SETTINGS_SCHEMA` entries |

---

## Benchmark result (May 2026)

After the model swap + Features 1 and 3 shipped together:

| Benchmark | Before | After | Δ |
|---|---|---|---|
| LoCoMo overall | 73.7% | 81.6% | +7.9pp |
| LoCoMo cat-1 (profile facts) | 64.4% | 69.3% | +4.9pp |
| LoCoMo cat-2 (temporal) | 79.9% | 84.9% | +5.0pp |
| LoCoMo cat-3 (inference) | 55.8% | 64.4% | +8.6pp |
| LongMemEval R@5 | 95.6% | 97.0% | +1.4pp |
| ConvoMem average | 99.0% | 99.0% | unchanged (ceiling) |

The `recall_reach`-style metrics above are from the dialog/session retrieval path. They are independent of the memory-pipeline `recall_generation` metric tracked separately in `CAT2_TEMPORAL.md` (which lifted Phase 1 anchor injection from 0.022 → 0.321 on cat-2 specifically).

---

## Design rationale (resolved questions)

1. **Per-row `model_hash` columns?** No. Dropped in favour of the per-user gate on `users.embeddings_model_id`. No changes to `memory_facts`, `memory_entities`, or `document_chunks` schema.
2. **Defer document_chunks?** Yes. Separate lower-priority pass after facts/entities complete. Idempotent restart on interrupt.
3. **SHA-256 the model file at startup?** No. Cost 50-100ms on cold eMMC for 32MB file at every startup. Human-readable `model_id` string is sufficient — operator-controlled, dim-mismatch caught independently.
4. **Cross-encoder reranker?** Investigated and reverted — see [RERANKER_INVESTIGATION.md](RERANKER_INVESTIGATION.md). Was Feature 2 of the original plan; outcome captured separately so this doc stays focused on what shipped.
