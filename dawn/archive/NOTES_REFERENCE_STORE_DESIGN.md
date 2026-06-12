# Notes / Reference-Text Store — Design & Implementation

**Status: SHIPPED (archived 2026-06-13).** A filing-cabinet (deterministic, exact-recall) store built on the document/RAG layer, plus surgical editing and full version history. This is the historical design + implementation record for the whole program; it was a working doc in `dawn/docs/` and graduated here once the program settled.

**Ship history (branch `notes_reference_store`, all on `main`):**
- `ffa7aee` — notes/reference store + hybrid lexical (BM25) search (schema **v61**): a "note" = single-chunk document whose `filename` IS the label; column-weighted FTS5 label/body + phrase bonus so exact-label queries rank #1 (the conv-809 fix). + WebUI Notes tab, the `document_manage` tool (save/list/delete + two-step delete), memory get-by-ID.
- `59730cb` — **Phase 9**: extraction guard (redacts note-filed text from session-end extraction) + memory→note bridge (a thin gloss fact pointing at the note so a fuzzy "what's my bio" routes deterministically to the verbatim note).
- `c697202` + `7687051` — A-items: UI polish (tab memory, read-only viewer, sticky panel, search-weight settings) + admin FTS rebuild (`dawn-admin memory rebuild-document-fts`, v61 recovery).
- `545d7fc` — **B1a + B2**: surgical note edit/append (find/replace, unique-match contract, JSON-object tail param) + memory↔doc one-home-per-datum guidance.
- `ce72d3f` — **B3 versioning (v62)** + **B1b multi-chunk document editing (v63)** + post-ship follow-ups (undo-a-delete surfaces, `recover`-as-undo, documents-tab History/Restore, save_text steering). Big-three reviewed, 78/78 CI, live-verified end-to-end (incl. LLM-driven undo of an edit on a multi-chunk document through the destructive re-chunk swap).

**Open:** only **B4 structured records** (a key-value/record type) remains — SHELVED (B1 surgical edit absorbed most of the "updates are clumsy" pain), plus the deferred "next-touch" engineering items in the B1–B3 plan section below. Everything else here is shipped.

## Live test (2026-06-11) — what passed

A ~35-minute manual shakedown against the real model (OpenRouter / Claude Sonnet 4.6) exercised every surface, all clean:
- `document_manage save_note` (3 bios, doc 18/19/20) — tail-param `label::text::body` encoding parsed correctly; `document_read` returned the **verbatim** text (the 809 fix).
- hybrid `document_search` on "bio" returned the bios; **overwrite-by-label** ("use Jay instead" → same label) → *"Updated (overwrote existing)"*, doc_id stable.
- `list`; **two-step delete both branches** — ask→cancel ("hang onto it") AND ask→`confirm_delete` ("do it" → Dum-E deleted).
- `save_text` made a **3-chunk document** (distinct from single-chunk notes) and `document_read` returned all 3 chunks.
- **WebUI**: `note_save` / `note_update` / delete on 'Flock Work' from the browser; PDF upload still indexes.

Late fix found by testing (not by static review): `document_manage` carried `TOOL_CAP_DANGEROUS` but no config/parser, so `validate_dangerous_tool` **rejected its registration** and the LLM never saw it. Fixed by adding a minimal `document_manage_config_t {bool enabled=true}` + parser + `.config_section="document_manage"` (mirrors `shutdown_tool`). **Lesson: any `TOOL_CAP_DANGEROUS` tool MUST supply a config struct + parser, `enabled` first.**

## Context — why this exists

Conversation 809 exposed a real gap: the user stored three canonical bios, then couldn't get them back cleanly. Two failures — (1) Friday narrated "let me go direct by ID" but there was no get-by-ID action, and (2) a literal-text search for the bios surfaced their near-twins and dropped the canonical entries below the top-K cutoff. Semantic recall buried the exact records under their semantic neighbors.

Root cause: DAWN's memory subsystem is **associative recall** ("what do you know about X?") — fuzzy, ranked, top-K by design. The user also needs a **reference filing cabinet** ("give me back EXACTLY what I filed under label Y") — deterministic, keyed, exact. No ranking tuning makes semantic recall deterministic.

**Solution:** build the filing cabinet on the **existing document/RAG store**. A "note" is a single-chunk document whose `filename` IS the user's label. The store gains a **column-weighted FTS5/BM25 lexical retrieval path** that is its **own candidate set** (not a boost on the semantic top-K), fused with semantic search, plus an ordered/contiguous-phrase bonus — so an exact-label query ranks the right record first.

## Developer decisions

- **No `source_type` enum** — note-ness is the derived property `num_chunks == 1` plus `filetype == "note"`; the whole-record property can't drift.
- **`filename` IS the label** — no new title column; FTS-weighted high so a label match dominates.
- **Schema split first** — v61 was the trigger the standing `auth_db_schema.c` split TODO named, so a pure-movement extraction (`auth_db_migrations.c`) landed before v61.
- **Single-chunk by construction** — notes bypass the chunker and reject over-cap text, so `num_chunks == 1` holds on every path.
- **LLM ingestion is a genuinely new capability** (not folding into the URL-based `document_index`): a dedicated `document_manage` tool gives the LLM save/list/delete, realizing the planned "LLM manages its own docs" direction and retiring the *LLM document delete tool* TODO. **Deletions require user approval** (server-side staged two-step, mirroring email trash).
- **Deterministic memory→note bridge** (planned) keeps canonical text OUT of semantic memory; a thin gloss with `note_doc_id` redirects fuzzy hits.

## Architecture

```
                 ┌───────────────────────── document_search (hybrid) ──────────────────────┐
   query ───►    │  semantic (cosine over all chunks)   ⊕   lexical (BM25, own candidates) │ ──► fused top-K
                 │              + ordered/contiguous phrase bonus + temporal                │
                 └─────────────────────────────────────────────────────────────────────────┘
   "my bio" ──► document_read (exact label) ──► verbatim note text
   save/list/delete ──► document_manage tool (LLM) | Notes tab (WebUI) ──► document_index_note / _text / delete_indexed
```

- **A note** = `documents` row, `filetype='note'`, `num_chunks=1`, `filename`=label; one `document_chunks` row; one `document_chunks_fts` row (label_stems + body_stems).
- **FTS5** `document_chunks_fts(label_stems, body_stems)` — contentless, app-synced, mirrors `memory_facts_fts`. Global-IDF caveat carried (per-user safety is the search JOIN + `(user_id=? OR is_global=1)` filter).
- **Hybrid fusion:** `hybrid = vec_w·cosine + kw_w·bm25_norm + phrase_w·phrase_bonus (+ temporal_w·proximity)`. Lexical is an independent candidate set; a chunk present in only one channel still scores. Phrase bonus computed over `union(semantic top-N, ALL lexical hits)` so an exact-label note with weak cosine still gets it.

## Schema (v61)

- `document_chunks_fts` virtual table (two weighted columns, contentless), in `SCHEMA_SQL` for fresh installs + a migration block (create + backfill every existing chunk: stem parent filename → label_stems, chunk text → body_stems) modeled on the v48 BM25 backfill. Partial-backfill-advances-version policy carried.
- `memory_facts.note_doc_id INTEGER DEFAULT NULL` (FK→documents ON DELETE SET NULL in fresh installs; plain `ADD COLUMN` on upgrade, matching the v47 precedent) — for the memory→note bridge.
- `AUTH_DB_SCHEMA_VERSION` 60 → 61.

## Config (`[documents]`)

`fts_label_weight` (3.0), `fts_body_weight` (1.0), `hybrid_keyword_weight` (0.3), `hybrid_vector_weight` (0.7), `phrase_bonus_weight` (0.25), `search_min_score` (0.3). Defaults match the memory subsystem's keyword/vector balance; expect to tune `fts_label_weight`/`search_min_score` during search-quality validation.

## Tool surface

- **`document_search`** — now hybrid lexical+semantic; description covers exact-label retrieval.
- **`document_read`** — verbatim by label (the deterministic exact-get); description steers "my <label>" here.
- **`document_manage`** (new) — `save_note` (overwrite-on-same-label via exact `COLLATE NOCASE` lookup), `save_text` (general chunked doc), `list`, `delete` (stages + asks approval), `confirm_delete` (executes). `TOOL_CAP_DANGEROUS`; `is_global` always false.
- **memory `get`** action (bundled) — exact fetch by ID, closes the 809 bluff.

## Lock & correctness invariants

- **Stem outside the auth_db lock** (leaf-lock rule) everywhere — create, edit, delete, backfill. The `_locked` FTS helpers take pre-stemmed strings.
- **Contentless FTS5 has no FK cascade** — `document_db_delete_indexed` runs an FTS `'delete'` pre-pass (re-deriving stems from live rows) before the document delete, in one transaction, so no orphan FTS rows bias IDF.
- **Stable-id note edit** — `document_db_note_update`, gated `filetype=='note' && num_chunks==1 && owner`, swaps chunk text+embedding, rebuilds FTS rows, updates filename+hash in one transaction; doc_id/is_global stay stable.

## Testing

- `tests/test_document_search_bm25.c` (the **809 reproduction**, 6 tests): exact-label ranks #1 over near-twins; other labels discriminate; body terms retrieve; stable-id edit keeps doc_id + swaps FTS (old label gone); orphan-free indexed delete; exact-label lookup rejects substrings ("Bio" ≠ "Public Bio").
- `test_document_db` extended (FTS sync via `memory_stem`/`memory_bm25`); `test_memory_bm25` updated for the promoted `memory_bm25_build_match_expr`.
- Full CI: **74/74 green**, zero warnings, format clean.

## Implementation status

| Phase | Status |
|---|---|
| 0a `auth_db_migrations.c` split | ✅ shipped |
| 0b memory `get`-by-ID | ✅ shipped |
| 1 schema v61 (FTS + backfill + note_doc_id) | ✅ shipped, SQL validated on live-DB copy |
| 2 FTS sync helpers + `delete_indexed` | ✅ shipped + tested |
| 3 prepared statements | ✅ shipped (bound-param bm25 verified) |
| 4 hybrid search + phrase bonus | ✅ shipped + **809 test passes** |
| 5 note create / edit / delete | ✅ shipped + tested |
| 6 tool descriptions + `document_manage` tool | ✅ shipped (staged-delete approval) |
| 8 config knobs | ✅ shipped |
| UI WebUI Notes tab | ✅ shipped |

### WebUI Notes tab (shipped)

`Documents | Notes` tabs + a search box in the Document Library popover (`scope`/`query` added to `doc_library_list`, server-side scope via `document_db_list_filtered`, label/body BM25 search). Inline `.note-editor` (create/edit, label ≤80 + text ≤4000 with live counter, validation, dirty-confirm on cancel/ESC). NOTE badge (accent cyan), pencil-edit on hover; note bodies are returned in the list (single-chunk, small) so edit pre-fills with no extra round-trip. New `doc_library_note_save`/`note_update` WS opcodes (`webui_doc_library.c`); existing delete now routes through `document_db_delete_indexed` (no FTS orphans) and keeps the client confirm-modal (user approval). `.dawn-input` primitive extracted (the search box is its second consumer); tablist roving-tabindex + ←/→/Home/End; 480px bottom-sheet inherited.

## Post-ship hardening — doc/tool-loop fixes (2026-06-12, live-verified)

Live testing of Phase 9 surfaced four *pre-existing* bugs in the document/tool-loop path (not Phase 9 code) plus an ID-targeting gap.  All implemented + live-verified on the real DB; folded into the same uncommitted branch.

1. **`save_text`/`save_note` clipped at 4 KB.**  The text buffer in `document_manage_tool.c` was sized to the per-*chunk* `DOC_CHUNK_TEXT_MAX` (4096), so a multi-chunk `save_text` was silently truncated mid-content (Friday's 5127-char doc lost its task table + tail).  Fix: `DOCMGMT_SAVE_TEXT_MAX` (16384, `_Static_assert`'d to `LLM_TOOLS_ARGS_LEN`).  **Verified**: re-save stored the full doc — final chunk grew 427→1472 chars, ends clean instead of `…| Pendi`.
2. **`save_text` duplicated on every re-save.**  `do_save_text` had no overwrite-by-label (unlike `do_save_note`), so re-saving made a second document.  Fix: a bounded sweep deletes every same-named non-note doc the caller owns, then re-indexes — converges to one + self-heals existing dups.  **Verified**: result `Updated (overwrote existing)`, one doc remains.
3. **Duplicate-tool-call guard blocked user-requested re-reads.**  `DUPLICATE_CHECK_LOOKBACK=10` spanned prior turns, so "reread that doc" was rejected as a loop.  Fix: scope the scan to the current turn (after the last real user message; Claude tool-result messages don't count).  Unit-tested (`tests/test_llm_dup_check.c`, both providers).
4. **Over-long tool args truncated + executed silently.**  Args over `LLM_TOOLS_ARGS_LEN` were clipped (storing corrupt data).  Fix: `tool_call_t.args_truncated` set in all four arg-assembly paths (OpenAI/Claude × parse/stream, incl. the streaming drop-on-overflow); the executor refuses with an actionable error.
5. **`document_read` by ID + `document_manage delete` by ID.**  `document_read` resolved by name only (always returned the newest of same-named docs), so the LLM couldn't compare or target one.  Added an optional owner-scoped `id` param to `document_read` (precedence over name); `delete`-by-id already existed and `list` already prints `[id N]` — descriptions now point at it.  **Verified**: Friday compared 22 vs 24 by id and deleted 22.

## Unfinished / future work — the program backlog (tracked here, NOT in TODO.md)

### A. Carried follow-ups (from the original notes-store ship)
- ~~**Usability (3 items, dev-requested):** (1) Document Library **remember the active tab**; (2) clicking a note **name** opens a **read-only view** with action buttons; (3) **sticky panel** (drop outside-click auto-close, mirror the music panel).~~ **SHIPPED** (branch `notes_reference_store`): (1) `state.scope` persists to `localStorage` (`dawn_doc_library_scope`), restored on `open()`; (2) note name is a `role="button"` opening an inline read-only `.note-viewer` (Close/Edit/Delete), keyboard-accessible, nested-Esc-aware; (3) outside-click handler removed — closes only on ×/Esc or a competing top-right header panel (memory + scheduler now both call `DawnDocLibrary.close()` on open; the top-right trio stays mutually exclusive, music coexists at bottom-right). Because the panel is now sticky (coexists with the Settings panel instead of closing), its z-index dropped 1000→997 (matching the music panel) so opening Settings slides out *over* it — memory/scheduler stay at 1000 since they close on the settings-button outside-click.
- ~~**Surface `[documents]` hybrid-search weights** in the WebUI `SETTINGS_SCHEMA`.~~ **SHIPPED** (same change): the 6 weights (`fts_label_weight`/`fts_body_weight`/`hybrid_keyword_weight`/`hybrid_vector_weight`/`phrase_bonus_weight`/`search_min_score`) added to the Documents settings section (advanced). Full backend round-trip wired — they were missing from `webui_config.c` (set), `config_env.c` JSON getter + TOML writer (only the parser + defaults existed).
- ~~**Phase 7 — admin FTS rebuild:** sibling `rebuild_document_fts` pass (recovery path for a partial v61 backfill; the migration auto-backfill covers the normal case).~~ **SHIPPED** (branch `notes_reference_store`): `document_db_rebuild_fts(int *count_out)` in `document_db.c` (chose this home over `memory_embed_recompute.c` — it's a one-shot synchronous op, not a background model-swap worker, so it belongs next to the FTS helpers + `delete_indexed` it reuses). Clears the contentless index with `'delete-all'` (no need for each row's original stems — the partial-backfill case may never have written them), then keyset-cursors every live chunk one-per-lock-cycle (stem outside the leaf lock, `document_db_chunk_index_fts` for the locked insert). Also cleans FTS orphans left by `delete_indexed`'s OOM plain-delete fallback (the index is rebuilt to exactly the live chunk set). Admin command `dawn-admin memory rebuild-document-fts` (global, no user — the FTS index spans users; opcode `0x92` → `handle_memory_rebuild_document_fts`). Tests: `test_rebuild_fts_reindexes_all` + `test_rebuild_fts_scopes_to_live_chunks` in `test_document_search_bm25.c` (77/77 CI).

### B. Friday's tool-experience critique (2026-06-12) — assessed, prioritized
Friday's own assessment after using the tools, ranked by impact:

1. **No surgical editing (highest value — build next).**  Editing one field forces read→reconstruct-in-context→re-save the whole doc — error-prone (it *caused* the truncation incident).  Proposed primitive: a **find/replace edit** (and **append**) on `document_manage` — the LLM sends only the diff (`find`/`replace`), the server loads the doc, applies it, re-chunks + re-embeds.  Implementation note: multi-chunk docs currently store only the (overlapping) chunks, not the canonical full text, so clean find/replace needs full-text storage added (a small schema change).  Phases naturally: **notes first** (single chunk = the full text, trivial), **docs second** (after full-text storage).  Mitigates B-4 below.
2. **Memory ↔ doc drift (policy, partly fixed).**  A living status (e.g. a badge state) can land in *both* a memory fact and a doc and drift.  Phase 9's extraction guard already stops *note-filed* text from being re-mined; the rest is a **one-home-per-datum** guidance rule (living status → doc/record; ambient facts → memory) to reinforce in tool descriptions.  Not a feature.
3. **No versioning (deferred).**  Overwrite is destructive.  Cheap future mitigation when overwrite is next touched: keep the prior version soft-archived for a retention window (mirrors email-trash).  Low priority.
4. **Structured records vs flat markdown — SHELVED (2026-06-12).**  A key-value / record type for living structured data (task lists, status tables) would make updates reliable — but it's a whole new subsystem (schema/CRUD/query/UI/retrieval-integration).  Surgical edit (B1) absorbs most of the "updates are clumsy" pain, so B4 is **parked until surgical edit proves insufficient** — revisit only if, after B1 ships, the LLM still struggles to reliably update living structured data.  Also tracked as SHELVED in `docs/TODO.md`.

### C. Bookkeeping / housekeeping
- **Atlas graduation:** this doc moves to `atlas/dawn/` (likely `archive/` or a new `documents/` area) once the program settles; the Phase 9 design + the bridge belong in `atlas/dawn/memory/` cross-refs.
- **Test gaps (deliberate):** `#4 args_truncated` has no dedicated unit (regression-covered via `test_streaming`/`test_claude_format`); `#1`/`#2`/read-by-id verified live, no unit tests (would need the full tool-registry/user-context harness).
- **Live-DB remediation (one-time):** forget the leaked-bio facts now covered by glosses — see the session notes (`memory forget` via the daemon, never raw SQL — contentless FTS + cache).

## B1–B3 implementation plan (2026-06-12, master-plan-reviewed) — surgical edit, drift policy, versioning

Planned, unstarted.  Grounding verified against the tree: `document_manage` actions live in `src/tools/document_manage_tool.c` (456 ln); the param flattening allows **exactly one tail (read-to-end) field** per call (`tool_registry.h` — "at most one `tool_param_extract_custom_tail`"); notes are **single-chunk** (`num_chunks==1`, the chunk text IS the full note); multi-chunk docs store only the (overlapping) chunks, **no canonical full text**; `document_note_update()` already does re-embed + FTS rebuild + stable-id swap; schema is at **v61**.

**Master-plan-reviewer pass (2026-06-12) folded in.**  Two HIGH gaps fixed (silent-corruption encoding → JSON tail; extraction-guard not extended to the new content args); B3 gains delete-archival + snapshot-once-at-pipeline-layer; **B3 reordered BEFORE B1b** (B3 = v62, B1b = v63) so the versioning safety net is live before the whole-doc rewrite lands.  Confirmed-correct calls (kept): unique-match contract, gloss-skip-on-edit (gloss text is label-derived + content-independent), separate `append`, no-backfill, B4 shelved.

### B1a — surgical edit for NOTES (find/replace + append) — NO schema change — IMPLEMENTED (Phase 1, branch `notes_reference_store`)
The high-value piece; covers the truncation-incident root cause (it was a bio = a note).  **Shipped 2026-06-12 with B2; 78/78 CI, build + format clean.**  Notes vs. the plan below: (1) `change` is a STRING param carrying the JSON object (the scheduler `details` pattern — DAWN has no OBJECT param type; still tail-safe + JSON-escaped, so the `::` corruption vector is closed); (2) the pure find/replace + append logic was extracted to `src/tools/document_edit_ops.c` (`docmgmt_find_replace_once` / `docmgmt_append_text`) so the contract is unit-testable without the storage layer (`tests/test_document_manage.c`, 10 cases incl. `::`-content); (3) the guard fix needed **two** edits, not one — the collector (`guard_collect_tool`) AND the structural redactor (`guard_redact_save_input`, which only blanked the `text` field) both had to learn `edit`/`append`; the test caught the missing second one (`+2` cases in `test_memory_note_guard`); (4) fixed a pre-existing `param_count`=5/4-entries off-by-one in the tool metadata as a side effect.  Below is the as-designed spec.

- **Tool surface:** `document_manage` enum 5→7: add `edit` and `append`.
  - `edit`: `label` (base = which note) + **`change` — a JSON-object tail param** `{"find": "...", "replace": "..."}`.
  - `append`: `label` (base), `text` (custom **tail**).
  - A distinct `append` (vs. "edit with empty find") reads clearer to the LLM and dodges find/replace ambiguity.
- **Encoding — JSON-object tail (REVISED per review, was the HIGH gap):** the original flat plan (`find` as a non-tail custom) is **silent-corruption-unsafe** — the extractor truncates a `find` value at the first embedded `::` (e.g. `find="Deadline:: June 10"` → extracts `"Deadline"`), and if the truncated string is unique the unique-match gate *passes* and applies the wrong edit with **zero diagnostics** (and the server can't even detect it — the `::` is gone by construction).  `::` is real in notes (Obsidian `Key:: value`, any saved `std::` code snippet; this deployment is coding-assistant-adjacent).  Fix: carry the edit as ONE terminal JSON object (`TOOL_PARAM_TYPE_*` already supports object/array tails — music_tool, scheduler `steps[]`; the serializer emits the JSON repr, `tool_param_extract_custom_tail` reads it intact, callback does one `json_tokener_parse`).  JSON escaping makes both fields arbitrary-content-safe and **extends to a future multi-edit array** (the SOTA direction).  Mitigate the known nested-JSON LLM-trip (the scheduler `steps[]` key confusion) with a strong description + tolerant key handling — trivial for a 2-key object.  `append`'s `text` stays a plain tail (arbitrary-safe already).
- **Match contract (mirror Anthropic `str_replace` — confirmed SOTA):** `find` must match **exactly once** as a substring.  Error texts matter as much as the contract (Friday is Claude-family, RLHF-trained on this):
  - 0 matches → `"text not found — use document_read to get the exact current text and copy it verbatim"`.
  - >1 → `"found N times — include more surrounding context to make it unique"`.
  - No silent wrong-edits.  Description carries **read-before-edit** discipline ("use document_read first; copy `find` exactly").  Optional future `replace_all`; omit v1.
- **append:** `new = old + "\n" + text` (newline-join only if `old` doesn't already end in one).  Reject empty `text`.  Append to a nonexistent note → error ("no note 'X' — use save_note to create it"), never auto-create (G10).  Over-cap → error stating remaining capacity + suggest split / save_text (G7 — also the first telemetry signal for un-shelving B4).
- **Extraction-guard extension (REVISED per review, was the second HIGH gap):** `edit`/`append` carry note content in their `change.replace` / `text` args.  Without extending the Phase 9 guard, that content rides the conversation into session-end extraction and gets re-mined into memory facts — re-opening the exact leak Phase 9 closed.  Add both actions + their content-arg field names to **both** guard collector sites (`src/memory/memory_note_guard.c:171` and `:426`), with a unit case per provider shape (OpenAI `tool_calls[].function.arguments` + Claude `content[].tool_use.input`).
- **Concurrency (G5):** edit normalizes read→modify→write; serialize it with a tool-local mutex (the `s_pending_mutex` pattern is already in the file).  Tool-vs-WebUI residual is documented-acceptable for single-user; B1b upgrades to CAS-by-hash.
- **Server flow (`do_edit` / `do_append`):** resolve note by `label` via `document_db_find_by_label_exact(user, label, true, &doc)` → gate owner + `filetype=="note"` + `num_chunks==1` (M-5); read the single chunk (`document_db_chunk_read`); apply edit/append → `new_text`; reject if over the note cap; reuse `document_note_update(user, doc.id, label, new_text, len, &res)` (stable doc_id/is_global/gloss-pointer); `LOG_INFO` the mutation (doc id, label, find length, occurrence count — **never content**, G9); return a message that **names what changed**.  Gloss unchanged (label same; gloss text is content-independent) → no bridge call (reviewer-confirmed correct).
- **Tests:** unique-find replace (doc_id stable, new searchable, old phrase gone); **`::`-laden find round-trips through the JSON tail**; not-found → error + unchanged; ambiguous → error + unchanged; append newline-join + empty-text reject + nonexistent-note reject; edit on multi-chunk/non-note → rejected; over-cap → rejected; **both provider-shape guard cases**.
- **Files:** `src/tools/document_manage_tool.c` + description + `src/memory/memory_note_guard.c` (guard extension) + a `tests/test_document_manage.c`.
- **Risk:** Low–Med (JSON-tail encoding removes the corruption vector; unique-match + M-5 gate + guard extension contain the rest). No schema, no migration.

### B2 — memory↔doc drift: one-home-per-datum guidance — NO schema, NO new code path
Phase 9's extraction guard (now extended to edit/append in B1a) stops note-filed text being re-mined.  Residual is **prompt guidance**: living/verbatim text the user will update or retrieve exactly → a **note/document** (editable, exact); ambient facts learned in passing → **memory** (auto-extracted).  Reinforce in the `document_manage` save AND **edit/append** descriptions ("to update living data, edit the note — don't re-remember it") + a one-liner in the memory `remember` description.  **Risk:** Very low (prompt-only; keep the guidance scoped to *verbatim/living* text; quick live sanity check).  Bundles with B1a.

### B3 — versioning: soft-archive prior version on destructive mutation — v62 (REORDERED before B1b)
- **Goal:** overwrite/edit/**delete** is destructive; keep the pre-change version recoverable for a retention window (mirrors email-trash + retention).
- **Schema v62:** `document_versions(id, document_id, user_id, filename, text, archived_at)` — snapshot the live content BEFORE any destructive mutation.  Config `[documents] version_retention_days` (default 14); a sweep drops older rows **plus a per-doc cap (keep last ~10)** so an edit loop can't balloon storage inside the window.
- **Snapshot ONCE, at the pipeline/DB layer (REVISED per review):** the original plan listed both `do_edit` (tool) AND `document_note_update` (pipeline) — `do_edit` calls `document_note_update`, so that double-snaps.  Snapshot at the lower layer (`document_note_update` + the delete core + B1b's chunk-replace), in the **same transaction** as the swap — one site covers tool + WebUI callers for free.
- **Write points (incl. DELETE — was missing):** overwrite (`document_note_update` / save_text overwrite-sweep), edit/append (B1, via the same pipeline call), and **delete** (`do_confirm_delete` → `document_db_delete_indexed`, + the WebUI delete path) — delete is the highest-stakes destruction and the two-step confirm doesn't stop user error, so it archives most of all.
- **Restore:** v1 = WebUI Notes-tab restore only; defer an LLM `list_versions`/`restore_version` (two-step) surface.  Retention sweep piggybacks on an existing periodic task (memory decay sweep / scheduler tick).
- **Why before B1b:** zero dependency on it; protects note overwrites/edits sooner; and B1b's whole-doc destructive rewrite — the scariest mutation in the program — lands with the safety net **already live**, not in the same diff.
- **Risk:** Med (schema + sweep + restore UX + bounded storage).

### B1b — surgical edit for MULTI-CHUNK documents — v63 (full-text storage)
- **Why:** multi-chunk docs keep only overlapping chunks; clean find/replace needs the canonical full text (overlap makes chunk-reconstruction lossy → reject any "rebuild from chunks" migration).
- **Schema v63:** new side table `document_full_text(document_id PK, text)` (keeps the `documents` row lean; full_text can be tens of KB; LEFT JOIN only when editing).
- **Backfill: NONE.**  Populate full_text on ingest going forward; an edit of a doc lacking full_text returns "this document predates editable storage — **re-saving it via save_text upgrades it on the spot**" (the overwrite-sweep + ingest-writes-full_text makes that a single tool call when the model has the content, not a user chore).  Notes never need full_text — B1a covers them.
- **Ingest:** `document_index_text` writes full_text in the same transaction as the chunks.
- **Edit flow (lock discipline spelled out, G6):** read full_text (locked) → compute new full_text + re-chunk + stems + embeddings **(all unlocked — stemming/embedding must NOT run under the auth_db leaf lock, the bug `document_db_rebuild_fts`'s one-per-lock-cycle dance avoids)** → replace all chunks + FTS rows + full_text in a **single write transaction**.  Heavier (re-embeds all chunks) but edits are rare.  Drop the `num_chunks==1` gate when full_text exists (keep owner gate).
- **Concurrency:** upgrade B1a's tool-local mutex to **CAS-by-hash** (pass expected-old-hash into the update; B1b's race window is seconds of embedding I/O, not microseconds).
- **Risk:** Med–High (migration + whole-doc re-chunk/re-embed + no-clean-backfill).  Own commit, lands with B3's versioning already protecting it.

### Sequencing (REVISED per review — B3 before B1b)
1. ~~**Phase 1 — B1a + guard extension + B2**~~ **SHIPPED** (commit `545d7fc`).
2. ~~**Phase 2 — B3 as v62**~~ **IMPLEMENTED 2026-06-13 (uncommitted, branch `notes_reference_store`).**  `document_versions` (no-FK-to-documents so versions outlive deletion; FK user_id→users CASCADE) + `version_archive_locked` snapshot-once at the DB layer inside `document_db_note_update` / `document_db_delete_indexed` (covers overwrite/edit/append/delete) + retention: config `version_retention_days` (14) age-sweep in `auth_maintenance` + per-doc cap `version_keep_per_doc` (10, config) at write time + WebUI version list/restore (`doc_library_version_list`/`_restore` WS, History button in the note viewer).  5 unit tests.  big-three-reviewed.
3. ~~**Phase 3 — B1b as v63**~~ **IMPLEMENTED 2026-06-13 (uncommitted).**  `document_full_text` side table (FK→documents CASCADE; written on ingest for non-global docs, NOT for single-chunk notes); `document_doc_update` pipeline (re-chunk + re-embed outside the leaf lock → `document_db_doc_replace` atomic swap: archive old → FTS-delete old → DELETE+INSERT chunks → UPDATE num_chunks/hash → upsert full_text, one txn, doc_id stable); `do_edit`/`do_append` accept multi-chunk docs via `load_editable_text` (pre-v63 docs → "re-save to enable"); no backfill.  3 unit tests incl. the destructive-swap.  big-three-reviewed (0 Critical/exploitable-High; fixes applied: per-doc-cap→config, skip full_text for globals, multi-chunk restore routes through doc_update, id>0 guard).  **Deferred follow-ups** (all defer-acceptable, in the overnight memory note): cache prepared stmts (cold path), extract versioning cluster from `document_db.c` (now 1544 ln) + the 4-copy stem-loop helper + `doc_replace` 12-param→struct on next touch, delete-path snapshot-failure contract, per-user version cap.
4. **B4** — shelved (above).

Phases 1–2 are mandatory and fully cover the high-value corpus (notes).  Phase 3 is the natural pause point if priorities shift.

### Post-ship follow-ups (2026-06-13, from live testing + Friday's tool-experience re-assessment) — IMPLEMENTED + live-verified
- **Undo-a-delete made reachable (the gap live testing exposed):** B3 archived deletes but had no surface to restore them (the History button is viewer-only; the LLM had no restore action).  Added: backend `document_db_version_list_deleted` (newest surviving version per now-deleted document_id, owner-scoped) + WebUI **"Recently deleted"** footer button (lists + Restore, reuses the restore handler's re-create branch) + LLM `document_manage` actions **`list_deleted`** and **`recover`**.
- **`recover` = full UNDO, not just delete-recovery:** for an item that STILL EXISTS, `recover <label>` restores its newest archived version in place (undo the last edit/overwrite via `document_note_update`/`document_doc_update`; the restore is itself snapshotted, so repeating toggles undo↔redo); for a deleted item it re-creates from the snapshot (note, or document if too long for one note).  Live-verified end-to-end incl. on a multi-chunk document (the destructive re-chunk swap rolled an edit back correctly).
- **Versioning made discoverable to the LLM:** the tool-level description now states edits/overwrites/deletes are auto-version-snapshotted and recoverable — versioning was previously invisible (automatic), which read as "unknown" to the assistant.
- **Documents-tab History/Restore:** the read-only viewer now opens for documents too (clicking a doc name), with version History + Restore (inline Edit is note-only); item-name click affordance generalized to notes + docs.
- **save_text vs save_note steering:** sharpened the action descriptions (save_note = SHORT, save_text = longer/multi-part) and the "note too long" errors now name `save_text` — live testing showed the model looping on `save_note` for long content.  Delete-confirm messages (LLM + WebUI) no longer claim "cannot be undone" — they point to `recover` / "Recently deleted."
- **Friday's wishlist scorecard after this round:** #1 surgical edit ✅, #2 append ✅, #3 versioning ✅ (was ⚠️ "unknown"), #4 memory↔doc drift = B2 description guidance only (partial, by design), #5 structured records = B4 (shelved).
- Tests: `document_db_version_list_deleted` + the undo round-trip added (18 cases in `test_document_search_bm25.c`).  **78/78 CI, 0 warnings, format clean.**

**Deferred "next-touch" items** (from the big-three review, all defer-acceptable — bundle on the next edit of these files): cache prepared statements in the versioning/full-text paths (cold path); extract the versioning + full-text + doc-replace cluster out of `document_db.c` (now ~1.6k lines) and the 4-copy stem-chunk-bodies helper; `document_db_doc_replace` 12-param → struct; decide the delete-path snapshot-failure contract (currently best-effort); per-user version-count cap (multi-user only); LLM-facing undo for multi-chunk-doc DELETE re-creates as a single doc via `document_index_text` (note→doc type coercion edge).

## Phase 9 design — extraction guard + memory→note bridge

**Status:** IMPLEMENTED + big-three-reviewed 2026-06-11 (master-plan review folded into the design; then architecture/efficiency/security review of the diff — 1 critical + 2 security-medium + others all fixed). All four phases shipped on branch `notes_reference_store`, 76/76 CI, build + format clean; live 809 re-run pending the developer. Two independent halves: the **guard** stops new leaks (ships standalone), the **bridge** makes a fuzzy "what's my bio" route deterministically to the note.

**Post-implementation review fixes (2026-06-11):** (C1, critical) the new `s_cache.note_doc_ids` flag array wasn't set on the incremental `cache_append_locked` path and was `malloc`'d — a mid-session appended fact could read garbage and be misread as a gloss → silently dropped from paraphrase-dedup; fixed with `calloc` + explicit `=0` on append.  (Sec-M1) `memory_db_fact_set_note_doc_id` now ownership-validates the linked document via an `EXISTS` subquery (mirrors `set_subject_entity`) so a gloss can't link to another user's note (the schema FK can't be user-scoped); the bridge rolls back the gloss if the link is refused.  (Sec-M2) the gloss text is built from the user-controlled label and reaches the LLM context bypassing the extraction injection filter, so the bridge now gates the label through `memory_filter_check`.  (Eff) the guard pre-normalizes each filed-body needle once at collect time (was re-normalizing per haystack) and the redact loop does one find per iteration.  (Misc) `failed` counter in the backfill summary; named gloss-confidence constant; aligned `do_save_note`'s message buffer.  Tests: `test_memory_note_guard` (8), `test_memory_note_bridge` (6, incl. cross-user-link rejection).

### Why
Conv 809's live log proved the leak: session-end extraction re-mined the filed bios into semantic memory facts (7872/7873/7877; 7873 ≈ the full bio). That's double-storage + drift — edit the note, the fact goes stale. The reference text must live ONLY in the note store; memory keeps a thin pointer.

### Part A — Extraction guard (Phase 1, standalone leak fix)
Hook at the extraction entry-filter loop (`memory_trigger_extraction` ~`memory_extraction.c:1995`), on JSON message objects before serialization. Three mechanisms:

1. **Collect filed bodies (both provider shapes — reviewer G3).** Scan assistant messages for `document_manage` save_note/save_text tool calls and pull each filed `text`. History is provider-shaped: OpenAI `tool_calls[].function.arguments` (JSON string) vs Claude `content[].type=="tool_use"` `.input` object. Must parse both — the dev's daily driver is Claude/OpenRouter, where an OpenAI-only parser finds nothing and the leak persists. Collect over the **full** history; redact only within the filtered window (G12).
2. **Redact filed bodies by exact normalized-whitespace substring.** Tool-call args: replace the `text` value with `"[filed to notes store]"` (deterministic — we own the string). Message `content`: exact-containment match of a filed body → replace that span with `[filed to notes store: 'Label']`, leaving surrounding instruction text and OTHER facts in the same turn intact. Exact-containment = zero false positives; no similarity threshold to mis-tune (sealed decision — do not re-litigate as fuzzy matching). Match at line/sentence granularity to catch "LLM trimmed one line" filings (G13).
3. **Redact `document_read`/`document_search` tool RESULTS unconditionally (reviewer G6).** A note read echoes the body into a *later* extraction window where the save tool-call (the redaction source) is absent → leak recurs on every read-after-save. Replace stored-doc/note result payloads with `"[stored document/note content elided — available verbatim via document_read]"`. No matching needed — it's our own deterministic tool echo of already-stored text; the user/assistant *discussion* of the content stays minable.

**Copy-don't-mutate (reviewer G9):** extraction messages are live refs into the session's conversation history (`extraction_message_strip_images` is explicit about not mutating them). Redact on fresh copies — fold into the strip-images copy pass. **Config:** `[memory] note_extraction_guard` bool, default ON. Entirely Layer 2.

**Known residual (documented):** a *dictated* note whose ASR transcript differs from the LLM-cleaned filed text won't be an exact substring of the user turn → that copy can still be mined. Typed/pasted (the 809 case) fully covered.

### Part B — Bridge
- **B1 gloss upsert (reviewer G7).** Shared memory-layer helper, **upsert keyed on `note_doc_id`**, called from all FOUR mutation sites: tool save (new + overwrite), WebUI save, WebUI update — AFTER a successful save, OUTSIDE any DB lock (embedding does I/O). Flow: `memory_db_fact_create` → embed → `memory_db_fact_update_embedding` → `memory_db_fact_set_note_doc_id` (new ad-hoc setter, `fact_set_expires_at` precedent). Canonical body NEVER enters `memory_facts`; v1 embeds the SENTENCE only (label-based recall; generic-label weakness is a future tunable — embed label+body-prefix while keeping fact_text the sentence). `save_text` multi-chunk docs are guard-protected but get NO gloss (RAG hybrid search covers them; deliberate scope cut).
- **B2 resolution — self-directing gloss text (reviewer G5; supersedes the formatter breadcrumb).** Facts surface both via the `memory.search` tool AND the per-turn focus-injection context. Rather than a formatter hook that only covers search, the gloss `fact_text` IS the directive: `"User's bio is saved verbatim as note 'Public Bio' — retrieve with document_read before quoting."` Every surface carries it for free. No inline body (a ~4000-char note overflows the 3072-char `source_budget` and could truncate mid-bio — sealed rejection). No Layer 2→3 reach. Settle the label quoting/escaping convention once (labels can contain quotes; must match what `document_read` expects).
- **B3 lifecycle.** DELETE: look up the gloss `fact_id` by `note_doc_id` BEFORE the doc delete (the FK `ON DELETE SET NULL` nulls the pointer after), then call the existing, battle-tested `memory_db_fact_delete` (handles FTS + cache invalidation correctly) — hook in the two delete callers, which have `user_id`; NOT inside `document_db_delete_indexed`'s transaction (reviewer G2: a `_locked` helper there can't stem outside the lock or invalidate the cache → orphaned FTS row, the exact bug the 3-phase delete dance exists to prevent). Atomicity dropped — the crash window self-heals (an orphan gloss with a NULLed pointer becomes prune-eligible, G2a). RENAME: gloss-update = **delete + create** (reviewer G8: `fact_text` is immutable for FTS contentless maintenance; no in-place UPDATE machinery exists). Makes upsert trivial too.
- **B4 dedup/prune exemptions.** Add `AND note_doc_id IS NULL` to: prune_low_confidence DELETE, prune_stale DELETE, bulk delete_by_patterns, `stmt_memory_fact_find_similar` (reviewer G4 — used by relation-supersede old-fact lookup; a LIKE hit could otherwise `superseded_by` a gloss → bridge dead silently), and confidence-**decay** UPDATE (reviewer G10 — a rarely-read bio gloss would otherwise decay to the floor, degrading the exact fact whose job is reliable routing). **The paraphrase-dedup exemption does NOT go in the cache-load SELECT (reviewer G1 — CRITICAL).** That SELECT (`stmt_memory_fact_get_embeddings`) feeds the single shared `s_cache` that serves BOTH the dedup gate AND the semantic retrieval channel; filtering glosses there removes them from cosine search entirely (and ships undetected — BM25 still finds "bio", masking the dead channel). Instead: add `note_doc_id` to the cache projection + an `is_gloss` flag array on `s_cache`; have `memory_embeddings_nearest_fact` (the dedup consumer) skip flagged entries — glosses stay retrievable, never merge. NOT exempted (correct): `memory_action_forget` by ID (explicit user delete of a gloss is fine — the note survives), prune on already-orphaned glosses (the self-heal).

### Files
- NEW `src/memory/memory_note_bridge.{c,h}` (Layer 2) — guard (A) + gloss upsert/delete/rename (B1/B3) + self-directing-text builder (B2).
- `memory_db_facts.c` — `memory_db_fact_set_note_doc_id` setter; `note_doc_id IS NULL` on prune/find_similar; `memory_db.c` decay exemption.
- `memory_embeddings.c` / `auth_db_statements.c` — `note_doc_id` into cache projection + `s_cache` gloss flag + `nearest_fact` skip (G1).
- `memory_extraction.c` — call the guard at the entry copy/filter pass.
- `document_manage_tool.c` + `webui_doc_library.c` — call the upsert helper after save (4 sites); look up + `memory_db_fact_delete` the gloss before note delete.
- Config: `dawn_config.h` / `config_defaults.c` / `config_parser.c` / `dawn.toml.example` — `note_extraction_guard`.
- Admin backfill: `admin_socket_memory.c` (+ `dawn-admin`) — one-time pass that creates missing glosses over existing notes and forgets the known leaked facts (dev chose the admin-socket pass over a manual step).

### Phasing (each builds + tests green; GPL header on new files; 3-space/100-col/K&R; SUCCESS/FAILURE no negatives)
1. **Guard** — Part A: both provider shapes (G3), tool-result redaction (G6), copy-don't-mutate (G9), full-history collect (G12), line-granular match (G13). Unit: one synthetic convo per provider shape (save_note tool call + user msg with body + an unrelated "moving to Denver" fact) → body redacted both spots, Denver survives; plus a `document_read`-result-redaction case. Whitespace-normalizer gets its own tab/newline/multi-space unit cases. Independently shippable — closes the leak completely incl. cross-window.
2. **Setter + exemptions + cache flag** — B1 setter; B4 exemptions incl. find_similar + decay; cache keeps glosses, `nearest_fact` skips them (G1). No behavior change until glosses exist. Unit: cache loads a gloss, hybrid_search returns it, nearest_fact skips it.
3. **Gloss upsert + lifecycle** — upsert at all four mutation sites (G7); delete via lookup-then-`memory_db_fact_delete` (G2); rename as delete+create (G8); self-directing fact_text (G5). Tests: gloss created w/ note_doc_id + embedded; deleted w/ note (no orphan); rename updates text; two similar-label notes → two un-merged glosses; overwrite-no-duplicate; pre-existing-note-overwrite-creates-gloss.
4. **Admin backfill + remediation + live 809 re-run** — backfill glosses over existing notes + forget 7872/7873/7877 (reviewer G11 — none of the other phases touch the existing damage, and the 809 bios predate the gloss path). Live test amended with a **semantic-channel-only** retrieval assertion (so a G1-style dead-channel can't hide behind BM25) and a focus-injection surface check (gloss + directive appear in "Context for this turn").

**Ordering note:** the guard-first window (notes filed, not mined, not yet glossed) is acceptable — the shipped hybrid `document_search`/`document_read` + the focus-injection document-chunk channel still route "what's my bio" to the note; only the memory-*fact* discovery path is briefly absent.

## WebUI design notes (for the Notes tab)

One popover, `Documents | Notes` tabs + a search row spanning both (`scope: documents|notes|all`); the primary action morphs per tab (upload dropzone vs "+ New note"). Inline slide-down `.note-editor` (NOT a modal — avoids nested focus traps), pure `.dawn-form`: Label (≤80 chars, becomes filename) + Text (≤4000 chars, live counter). Edit via a pencil icon on hover (rows are `cursor:default`); label editable with a rename warning. Search drives the hybrid backend; results reuse item rows + a snippet line with `<mark>` on lexical hits and a `.dawn-badge.muted` "similar" tag for semantic-only. NOTE badge in accent cyan. Dedicated `note_*` WS opcodes (edit/rename/single-chunk validation differ from upload); list/search piggyback on `doc_library_list` with `scope`+`query`. Bottom-sheet at 480px; tablist roving-tabindex + ←/→/Home/End. Extract the `.dawn-input` primitive (this search box is its second consumer).
