# Entity Merge / User-Identity Dedup — Design

**Status:** design only, not yet implemented. Untracked per project policy ("planned-but-unstarted work and working/scratch docs stay untracked").

**Date:** 2026-05-09. Authored after Phase 1 of Dynamic Context Injection closed and the live `/var/lib/dawn/auth.db` reextract surfaced persistent user-identity duplication as a per-tool-call user-visible defect.

---

## 1. Problem statement

The memory subsystem treats different surface forms of the same real-world entity as separate `memory_entities` rows. The primary user (user_id=1) currently has — among others — these duplicates in the live DB:

| id | name | entity_type | canonical_name | mention_count | outgoing relations |
|----|------|-------------|----------------|---------------|---------------------|
| 1162 | `user` | thing | `user` | 292 | 292 |
| 1230 | `Kris` | thing | `kris` | 28 | 59 |
| 1406 | `Kristopher Kersey` | person | `kristopher kersey` | 21 | 45 |
| 1735 | `kerseyfabrications` | thing | `kerseyfabrications` | 1 | — |
| 1747 | `kerseyfabrications@gmail.com` | thing | `kerseyfabrications@gmail.com` | 1 | — |

All five refer to the same person. Consequences observed in production:

1. **Relation duplication.** "Kris works_at FOO" and "Kristopher Kersey works_at FOO" are stored as two `memory_relations` rows with different `subject_entity_id` values. 396 outgoing relations are split across the three primary aliases (292 + 59 + 45) instead of consolidated under one canonical subject.
2. **Search fragmentation.** `memory_action_search` for "where does Kris work" hits one entity; "where does Kristopher work" misses (or vice versa).
3. **Confidence dilution.** Facts attached to different aliases never aggregate; nightly decay treats them independently.
4. **Contradiction-detection blind spot.** `memory_db_relation_supersede()` matches on `subject_entity_id` — `Kris works_at FOO` (open) and `Kristopher Kersey works_at BAR` (open) coexist forever because no `EXCLUSIVE_RELATIONS[]` row sees them as the same subject.
5. **Per-turn focus block.** Phase 1f's `memory_focus_adapters` lists each alias as an independent candidate, so a single conversational turn about "where does Kris work" can produce both "Kris → works_at → FOO" and "Kristopher Kersey → works_at → FOO" cards in the user-visible focus injection block. The user sees their own duplication every turn the focus block fires.

The same shape applies to non-user entities ("Mom" vs "Melanie", "the cat" vs "Sasha") but at lower volume — non-user entities tend to use a smaller vocabulary per conversation.

The reextract utility (`dawn-admin memory reextract`, May 9, 2026) does NOT fix this. Re-running extraction creates fresh duplicates because each conversation's surface forms get extracted as they appear, and the existing `memory_db_entity_upsert()` path has no canonical-form / merge logic — it's `INSERT … ON CONFLICT(user_id, canonical_name)`, so different canonical_names always create different rows.

---

## 2. Architecture choice (Q-A): equivalence-class with audit table

### Three options weighed

- **Canonical-form (one physical row per real-world entity, separate `memory_entity_aliases` for surface forms).** Cleanest model. Most expensive migration: every existing row that should collapse needs physical merge before the model is consistent. Hard-deletes lose per-alias `mention_count` semantics unless aggregated at merge time. Recovery from a wrong merge requires `memory_entity_aliases` to retain enough state to recreate a separate row, which means it's effectively the equivalence-class shape with extra steps.
- **Equivalence-class (`memory_entities.canonical_id` self-FK; NULL = self is canonical).** Cheapest migration: one nullable column add + one partial index. No physical row movement. Reversible by zeroing `canonical_id`. Order-independent reads via `COALESCE(canonical_id, id)` JOIN. Live-reads pay one indexed lookup per entity touched; the existing focus-adapter / search paths already load the entity scratch buffer in RAM, so resolution is sub-millisecond.
- **Alias-table-only (degenerate canonical-form: rows untouched, `memory_entity_aliases(alias_text, target_entity_id)` consulted before upsert).** Adds a third lookup path the resolver must consult. Doesn't naturally express "this row is canonical" because the canonical is implicit (rows that aren't anyone's `target_entity_id`). Doesn't compose with the existing `memory_db_entity_merge()` primitive.

### Choice: equivalence-class

Add a nullable `canonical_id INTEGER` column to `memory_entities`. NULL means "self is canonical." Non-NULL means "this row is a soft alias of `canonical_id`." All read paths resolve via `COALESCE(canonical_id, id)` and aggregate accordingly.

A separate **audit table** `memory_entity_aliases` tracks the soft-link history (when, why, evidence, link kind) but is NOT consulted by hot read paths. See §3 for its precise purpose.

### Two-tier model: soft-link vs hard-merge (point 1)

The design has two states:

| State | What changed | Reversibility | Read cost | Triggered by |
|-------|--------------|---------------|-----------|--------------|
| **Soft-linked** | `canonical_id` set on alias row; alias row physically present; relations still attached to alias `entity_id` | Trivial: zero `canonical_id`, alias is independent again | One indexed JOIN per entity touched | Auto-merge gate at extraction (composite ≥ auto threshold, default 0.90); manual `memory.merge_entities` LLM tool action; WebUI two-click; `dawn-admin memory entity merge` |
| **Hard-merged** | Existing `memory_db_entity_merge()` primitive ran: relations re-pointed (UPDATE subject_entity_id / object_entity_id), contacts re-pointed, source row physically deleted | Not reversible without `dawn-admin memory reextract` | No JOIN — single-table read | Operator-only: `dawn-admin memory entity consolidate` against a soft-linked alias |

**Why both?**

Keeping soft-only forever is tempting but loses on three grounds:

1. **Cache pressure.** `memory_embeddings.c` caches up to 500 entity embeddings (`ENTITY_CACHE_CAP`); `memory_focus_adapters.c` allocates a 256-entity scratch buffer (`ENTITY_EMBED_BUF_CAP`). Soft-only growth pollutes both pools with redundant rows that contribute no new signal — the alias's stored embedding is by definition near-cosine-1.0 to the canonical's. At scale (hundreds of conversations, dozens of soft aliases per major entity) this evicts useful entities.
2. **JOIN cost compounds.** Every hot read pays `COALESCE` lookup. Sub-ms today, but the focus block runs per-turn and the entity scratch buffer is rebuilt on cache miss; soft-only growth is unbounded as new conversations introduce new surface variants.
3. **Existing primitive is already proven.** `memory_db_entity_merge()` ships with transactional safety, dedup logic, and a known-good interaction with the embedding cache (modulo the in-scope cache-invalidation fix in §12). Discarding it for soft-only reverses prior work for no read-path benefit.

Keeping hard-only loses on graceful undo — auto-merge gates make mistakes (twin siblings; same-name colleagues; "Bob" the user vs "Bob" the manager) and a destructive-by-default policy means every wrong merge lands with no recovery short of full reextract.

**Trigger for soft → hard burn-in: operator-driven only. Never automatic.**

`dawn-admin memory entity consolidate [--all | --link-id <id> | --older-than <duration>]` runs the existing primitive against soft-linked pairs the operator has explicitly approved or aged-out. The audit row's `linked_at` timestamp gives operators a "this alias has been stable since 2026-04-15" signal for confidence in the consolidation.

The deliberate non-triggers:
- **No time-based auto-promote.** Even a six-month-old soft link can be wrong (the user happened to never disambiguate them). Burn-in must be a deliberate choice.
- **No confidence-based auto-promote.** Composite ≥ 0.95 doesn't justify destructive action without explicit consent — auto-merge gate already used composite to make the soft link.
- **Reextract auto-resets soft links** (per §10): the `memory_entity_aliases` table is one of the derived tables dropped by `dawn-admin memory reextract`, and `canonical_id` is nulled out across all entities for the user. This is consistent with reextract semantics and gives the operator a clean rebuild path if the alias graph drifts.

---

## 3. Schema deltas (Q-A continued; point 2 made explicit)

### `memory_entities` — additive

- **New column** `canonical_id INTEGER REFERENCES memory_entities(id) ON DELETE SET NULL` (NULL by default; FK self-references the entities table). NULL = "this row is canonical." Non-NULL = soft alias of `canonical_id`.
- **New column** `is_user_self INTEGER NOT NULL DEFAULT 0` (boolean). Used by `canonical_priority` and the seeded user-identity path (§7). Exactly one row per user_id has this flag = 1 once seeding has run.
- **New partial index** `idx_memory_entities_canonical ON memory_entities(canonical_id) WHERE canonical_id IS NOT NULL`. Used by the read-path JOIN (`SELECT * FROM memory_entities WHERE canonical_id = ?` to enumerate aliases of a canonical). Partial because the bulk of rows have NULL canonical_id and don't need to be in the index.
- **New partial unique index** `idx_memory_entities_user_self ON memory_entities(user_id) WHERE is_user_self = 1`. Schema-level invariant enforcing the "exactly one user-self row per user" rule. Prevents a malformed `link-user-self` invocation (e.g. operator runs the command twice with different `--username` matches resolving to different existing entities) from leaving two `is_user_self = 1` rows.

### `memory_entity_aliases` — new audit/evidence table (point 2)

Audit log only. Not consulted by hot read paths. Holds the *why* and *when* of each soft-link, and survives even after a hard-merge deletes the source row.

```sql
CREATE TABLE memory_entity_aliases (
    id                     INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id                INTEGER NOT NULL,
    source_entity_id       INTEGER,                 -- NULL after hard merge (source row gone)
    target_entity_id       INTEGER NOT NULL,        -- canonical entity at link time
    source_canonical_name  TEXT NOT NULL,           -- preserved at link time so split can verify
    target_canonical_name  TEXT NOT NULL,           -- snapshot for forensics
    link_kind              TEXT NOT NULL,           -- 'soft' | 'hard'
    reason                 TEXT NOT NULL,           -- 'auto-merge' | 'operator' | 'seeded'
    composite_score        REAL,                    -- NULL for 'seeded'/'operator-forced' merges
    evidence_json          TEXT,                    -- {name_jaccard, embedding_cosine, exclusive_overlap, contact_overlap, type_match, applied_bonuses[]}
    linked_at              INTEGER NOT NULL,
    consolidated_at        INTEGER,                 -- when soft → hard fired (NULL while still soft)
    unlinked_at            INTEGER,                 -- when split fired (NULL = active)
    unlink_reason          TEXT,                    -- 'split-by-operator' | 'reextract-reset' | NULL
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (source_entity_id) REFERENCES memory_entities(id) ON DELETE SET NULL,
    FOREIGN KEY (target_entity_id) REFERENCES memory_entities(id) ON DELETE SET NULL
);

CREATE INDEX idx_memory_entity_aliases_user_target
    ON memory_entity_aliases(user_id, target_entity_id) WHERE unlinked_at IS NULL;
```

`source_entity_id` is `ON DELETE SET NULL` because hard merge deletes the source row but the audit row must survive. `source_canonical_name` is the snapshot that lets the audit stay self-describing after the row is gone.

**Read paths that consume `memory_entity_aliases` (cold paths only):**
- WebUI Graph tab "show aliases for canonical": `SELECT … WHERE target_entity_id = ? AND unlinked_at IS NULL`
- `dawn-admin memory entity history <entity_id>`: full timeline including unlinked rows
- `dawn-admin memory entity split <link_id>`: load + validate the link before unlinking

**Read paths that do NOT consume it (hot paths):**
- Extraction-time alias resolver (uses `canonical_id` on `memory_entities` only)
- `memory_action_search`, `append_graph_context`, focus-adapter entity recall (all use `COALESCE(canonical_id, id)` JOIN against `memory_entities`)

### `memory_entity_merge_proposals` — new table for the mid-confidence band

The review band (default 0.70–0.90, first-ship) needs a queue surface (operator review). Adding a fourth `link_kind` to `memory_entity_aliases` (`'proposed'`) conflates two different lifecycles, so a separate table is cleaner:

```sql
CREATE TABLE memory_entity_merge_proposals (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id           INTEGER NOT NULL,
    source_entity_id  INTEGER NOT NULL,
    target_entity_id  INTEGER NOT NULL,
    composite_score   REAL NOT NULL,
    evidence_json     TEXT NOT NULL,
    proposed_at       INTEGER NOT NULL,
    resolved_at       INTEGER,                       -- NULL = still pending
    resolution        TEXT,                          -- 'approved' | 'rejected' | NULL
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (source_entity_id) REFERENCES memory_entities(id) ON DELETE CASCADE,
    FOREIGN KEY (target_entity_id) REFERENCES memory_entities(id) ON DELETE CASCADE
);

CREATE INDEX idx_merge_proposals_pending ON memory_entity_merge_proposals(user_id, proposed_at)
    WHERE resolved_at IS NULL;
```

Approving a proposal calls the soft-link path (writes a `memory_entity_aliases` row + sets `canonical_id`); rejecting just stamps `resolved_at`/`resolution`. The proposal table is a transient queue and rows can be aged out periodically — it's not authoritative for anything.

### Migration

One schema migration (next version after v42 → v43):

1. `ALTER TABLE memory_entities ADD COLUMN canonical_id INTEGER REFERENCES memory_entities(id) ON DELETE SET NULL` (literal-constant default, SQLite O(1) ALTER fast path).
2. `ALTER TABLE memory_entities ADD COLUMN is_user_self INTEGER NOT NULL DEFAULT 0`.
3. `CREATE INDEX idx_memory_entities_canonical … WHERE canonical_id IS NOT NULL`.
4. `CREATE UNIQUE INDEX idx_memory_entities_user_self ON memory_entities(user_id) WHERE is_user_self = 1`.
5. `CREATE TABLE memory_entity_aliases (…)` + index.
6. `CREATE TABLE memory_entity_merge_proposals (…)` + index.

No backfill of existing rows in the migration itself — soft-link backfill runs as the operator-invoked `dawn-admin memory entity link-user-self` and `--scan-all` paths (§7, §13), so the migration is a pure schema change with O(1) cost per ALTER.

---

## 4. Extraction-time integration

### Where the resolver hooks in

In `src/memory/memory_extraction.c::process_extraction_response()`, around line 649 (where `memory_make_canonical_name(ent_name, …)` produces the canonical and `memory_db_entity_upsert()` is called), insert a new resolver call:

```c
char canonical[MEMORY_ENTITY_NAME_MAX];
memory_make_canonical_name(ent_name, canonical, sizeof(canonical));

/* New: alias-aware resolution before upsert. */
int64_t resolved_id = 0;
memory_alias_resolve_t resolution = { 0 };
int rc = memory_db_entity_resolve_alias(user_id, ent_name, ent_type, canonical,
                                        &resolved_id, &resolution);
if (rc == MEMORY_DB_SUCCESS && resolved_id > 0) {
   /* Existing entity (canonical or alias) — bump mention_count, no new row. */
   memory_db_entity_bump_mention(user_id, resolved_id);
   eid = resolved_id;
   was_created = false;
} else {
   /* Fall through to existing upsert path. */
   memory_db_entity_upsert(user_id, ent_name, ent_type, canonical, &was_created, &eid);
}

/* New: when a fresh row was created, run the auto-merge gate against existing entities. */
if (was_created && eid > 0) {
   memory_alias_evaluate_t eval = { 0 };
   memory_db_entity_consider_auto_merge(user_id, eid, &eval);
   /* eval populates either:
    *   - eval.linked_canonical_id (auto-merge fired; canonical_id was set + audit row written)
    *   - eval.proposal_id (mid-confidence; proposal queued for review)
    *   - eval is empty (rejected / no candidates)
    */
}
```

The `_resolve_alias` function does all the work in §6. The `_consider_auto_merge` function runs the full composite scorer against existing entities and either soft-links (composite ≥ auto threshold), queues a proposal (review band), or no-ops. Threshold defaults are configurable per §7; first-ship values are conservative (0.90 / 0.70) pending Phase 2 corpus calibration.

The local `entity_map` cache inside `process_extraction_response` (the per-call dedup at `memory_extraction.c:609`) keeps working as-is — it's a within-extraction-call dedup. The resolver supersedes the cross-conversation dedup gap that today produces fresh `Kris`/`Kristopher Kersey` duplicates from each new conversation.

### Embedding generation order

Today: `memory_db_entity_upsert` returns `was_created=true`, then `memory_embeddings_embed_and_store_entity()` fires (`memory_extraction.c:664`).

After: same logic. If the resolver short-circuited to an existing entity (`was_created=false`), no new embedding is generated — the existing one is reused. If a fresh row is created and *then* auto-merge soft-links it, the embedding generation still happens (it costs nothing extra). The alias's stored embedding is **disk-only after soft-link** — the cache loader filter (§15) excludes aliases, so future resolver cosine calls compare new surface forms against canonical embeddings only. The disk-resident alias embedding remains as audit/completeness data and is reachable by the per-call Stage 4 single-fetch path if a future workstream ever needs it; it does not feed the production cache.

### Privacy / injection filter

Resolver calls happen AFTER `memory_filter_check(ent_name)` (existing). Filtered names never reach the resolver. The audit table stores `source_canonical_name` which has already passed the filter, so injection content can't land in audit rows.

---

## 5. Search-time integration

### `memory_action_search` and `append_graph_context`

Three places in `src/memory/memory_callback.c` need alias-aware lookup:

1. **`memory_embeddings_entity_search()`** at `memory_callback.c:286`: today returns `entity_ids[]` of cosine top-K. The cache underneath (`s_entity_cache` in `memory_embeddings.c`) is rebuilt from `memory_db_entity_get_embeddings`, which does NOT today filter by `canonical_id`. After change: the cache loader includes only canonical entities (`WHERE canonical_id IS NULL`); aliases are dropped from the cache. The cosine search therefore returns canonical IDs natively; no JOIN needed.
2. **`memory_db_entity_search()`** keyword fallback at `memory_callback.c:300`: existing LIKE-on-canonical_name search must resolve `canonical_id` at the result level — wrap with `COALESCE(canonical_id, id)` and DISTINCT. New SQL pattern:
   ```sql
   SELECT DISTINCT COALESCE(e.canonical_id, e.id) AS resolved_id
   FROM memory_entities e
   WHERE e.user_id = ? AND e.canonical_name LIKE ?
   ```
   followed by a second SELECT to fetch the canonical row(s) by `resolved_id`. Two-step pattern keeps the `_search` API shape unchanged.
3. **`memory_db_relation_list_by_subject()` and friends**: relations stay attached to their original (alias) `entity_id`. Listing relations for a canonical must UNION across the equivalence class. New helper `memory_db_relation_list_by_subject_class(user_id, canonical_id, …)` does:
   ```sql
   SELECT … FROM memory_relations
   WHERE user_id = ? AND subject_entity_id IN (
      SELECT id FROM memory_entities WHERE id = ? OR canonical_id = ?
   )
   ```
   The `IN` subquery is small (typically 1–4 IDs per canonical for the dev's cluster) and the partial index on `canonical_id` makes it cheap. `_list_by_subject_at`, `_list_by_object`, and the bulk `_list_all_by_user` get sibling helpers.

### Focus block (`memory_focus_adapters.c`)

The entity adapter (`memory_focus_adapters.c:s_entity_scratch`) loads entities + embeddings into a 256-row scratch buffer. After the change: the loader filters `WHERE canonical_id IS NULL` so aliases are absent from the scratch buffer entirely. This is the single biggest user-visible win — the per-turn focus block stops showing "Kris" and "Kristopher Kersey" as separate cards.

The relation adapter follows the same path: relations are listed against canonical entity_ids only (using `_list_by_subject_class`). Aliases' relations surface through their canonical's listing.

The fact adapter is unaffected — facts have no entity_id FK, only soft references via fact_text.

### Cost discipline (per Q-H)

Hot-path cost summary (after change):

| Path | Today | After |
|------|-------|-------|
| Extraction-time alias resolver | (didn't exist) | ~10–20ms typical (§6 cascade) |
| Search-time entity cosine | <1ms (cache) | <1ms (cache; aliases pre-filtered out at cache load) |
| Search-time entity keyword | ~1–3ms (LIKE) | ~1–3ms (same LIKE + DISTINCT canonical) |
| Focus block entity recall | <1ms | <1ms (same; aliases absent from scratch) |
| Focus block relation recall | ~2–5ms | ~2–5ms (sibling helper with `IN (canonical, alias_ids)`) |
| Relation listing (single subject) | ~1–2ms | ~1–2ms (same) |

The only meaningful new cost is the extraction-time resolver — which happens at session-end, not on the conversational hot path, and is amortized across the whole extraction call.

---

## 6. Resolver implementation (point 3)

`memory_db_entity_resolve_alias(user_id, name, entity_type, canonical_name, &out_id, &out_resolution)` runs a six-stage cascade:

| Stage | Cost | Drops candidates that fail | Output |
|-------|------|---------------------------|--------|
| 1. Exact canonical_name match | <0.5ms (existing indexed lookup) | — | Returns immediately if hit; canonical resolved via JOIN |
| 2. Token-Jaccard candidate generation | 1–3ms (LIKE per token + caps at 32) | name_jaccard < 0.30 | Up to 32 raw candidates |
| 3. Type filter | <0.1ms (in-RAM) | type mismatch when both sides non-`thing` | ≤ 32 typed candidates |
| 4. Embedding cosine | 2–5ms (8 cosine ops × 384 dims) | embedding_cosine < 0.50 | ≤ 8 surviving candidates |
| 5. Exclusive-relation + contact overlap | 3–10ms (3 × bounded SELECTs) | n/a (signal addition, not filter) | Top 3 candidates with full evidence |
| 6. Composite score + threshold band | <0.1ms | composite < review threshold → reject | 0 or 1 winner |

**Stage 1**: if `memory_make_canonical_name(name)` matches an existing `canonical_name`, return that entity's `COALESCE(canonical_id, id)`. This is the fast path for repeat mentions of the same surface form — the dominant case after the first conversation establishes an entity.

**Stage 2**: split `canonical_name` on whitespace and ASCII punctuation; for each token of length ≥ 2, run `memory_db_entity_search(user_id, token, …)` (existing LIKE-based keyword search). Aggregate up to 32 unique candidate IDs. Compute `name_jaccard = |tokens_a ∩ tokens_b| / |tokens_a ∪ tokens_b|` per candidate; drop those below 0.30. The 0.30 floor catches "Kris"/"Kristopher Kersey" (Jaccard = 1/2 = 0.50) but rejects accidental token sharing ("Kris" vs "Krispy Kreme" → 0.33; passes Stage 2 but Stage 3 catches the type/content mismatch in later stages).

**Stage 3**: drop candidates whose `entity_type` mismatches the inbound entity's type, EXCEPT when one side is `thing`. The `thing` carve-out exists because the live DB has `Kris` extracted as `thing` and `Kristopher Kersey` as `person` — the LLM is inconsistent about person/thing classification, and a strict type match would block the obvious correct merge. `thing` matches everything; person/place/pet/org must agree.

**Stage 4**: embed the inbound `canonical_name` via the existing engine (one call, ~2ms on bge-small-en-v1.5-int8). The embedding is reused for the new entity's `embedding` column if a fresh row ends up created. Cosine is computed against each Stage-3 survivor's stored embedding. The candidate pool is RAM-resident: the resolver reuses the existing `s_entity_cache` in `memory_embeddings.c` (already loaded for `memory_embeddings_entity_search`). No DB hit per candidate. Drop candidates with cosine < 0.50.

**Stage 5**: for the top 3 by partial-composite (`name_jaccard + clamped_cosine`), fetch:
   - Open exclusive relations: `SELECT relation, object_entity_id, object_value FROM memory_relations WHERE subject_entity_id = ? AND valid_to IS NULL AND relation IN (works_at, lives_in, …)`. The partial index `idx_memory_relations_subject_open` makes this cheap.
   - Contacts: `SELECT field_type, value FROM contacts WHERE entity_id = ?`.
   - Overlap is computed against the same fetches for the inbound entity (which exists by now if a fresh row was created in the parent `process_extraction_response` flow; the resolver is called either before upsert or after a fresh upsert depending on the integration point — see §4).

**Stage 6**: the composite formula in §7 fires; threshold band routes the result.

**Caching strategy**: the resolver is stateless across calls. The candidate embedding pool is the existing entity cache (already RAM-resident, loaded once per user with mutex protection). The Stage-2 keyword search is uncached (one LIKE per token, sub-ms). The Stage-5 SELECTs are uncached but bounded (≤ 3 candidates × 2 SELECTs).

**Concrete budget per resolver call**: ~10–20ms typical; ~30ms worst case (32 Stage-2 candidates surviving to Stage 4). Well within the tens-of-ms budget for extraction-time. The dominant component is the embedding call in Stage 4 — already part of the extraction path for new entities; the resolver makes that embedding do double duty.

---

## 7. Composite score formula (point 5)

### Veto

If both entities have non-`thing` `entity_type` AND the types differ → composite is forced to 0. (Example: a `person` and a `place` never merge regardless of name similarity.)

### Weighted sum (post-veto)

```
composite =
  0.30 × name_jaccard
+ 0.30 × max(0, embedding_cosine)
+ 0.25 × exclusive_relation_overlap
+ 0.10 × contact_field_overlap
+ 0.05 × type_match
```

Sum of weights = 1.0; composite ∈ [0, 1].

| Signal | Definition | Range |
|--------|------------|-------|
| `name_jaccard` | Token Jaccard on whitespace-split canonical_name | [0, 1] |
| `embedding_cosine` | Cosine similarity on entity-name embeddings (current model: bge-small-en-v1.5-int8, 384d) | [-1, 1], clamped to [0, 1] |
| `exclusive_relation_overlap` | 1.0 if any open exclusive relation (`works_at`, `lives_in`, `married_to`, …) shares `object_entity_id` (or canonical-equal `object_value`); 0.5 if any non-exclusive relation shares object; 0 otherwise | {0, 0.5, 1.0} |
| `contact_field_overlap` | 1.0 if any contact `email`/`phone`/`address` shared after normalization; 0 otherwise | {0, 1} |
| `type_match` | 1.0 if both types equal AND both non-`thing`; 0 otherwise. (`thing` ≠ contribution; not a penalty.) | {0, 1} |

### Additive bonuses (capped at 1.0)

Applied AFTER the weighted sum:

- **`name_substring_bonus = +0.10`** when one canonical_name is a strict substring of the other (e.g. "kris" ⊂ "kristopher kersey"). The bonus boosts cases where one name is a strict substring of another, on top of any token Jaccard signal — both lighting on shared-token evidence is intentional, rewarding the case where two name signals agree.
- **`user_self_bonus = +0.20`** when one side is the seeded user-self entity (`is_user_self = 1`) AND the other has user-identity signal: matches `users.username`, OR an existing seeded alias substring, OR an `email_is` open exclusive relation pointing at a known account email.

The user-self bonus is what lets `user` (id 1162, type=thing, mention_count=292) cluster with the seeded `Kristopher Kersey` (canonical, type=person) despite the otherwise-weak signal between a generic noun ("user") and a proper name. Without it, the developer's existing 292-relation `user` row would never auto-link.

### Threshold bands

| Composite | Action | Where it lands |
|-----------|--------|----------------|
| ≥ 0.90 | Auto soft-merge | `memory_entities.canonical_id` set + `memory_entity_aliases` row inserted (`reason='auto-merge'`) |
| 0.70 ≤ x < 0.90 | Queue for review | `memory_entity_merge_proposals` row inserted; no DB mutation on entities |
| < 0.70 | Reject | Debug-log only (`OLOG_DEBUG`) |

**Initial thresholds are deliberately conservative** (0.90 / 0.70 vs the originally-considered 0.85 / 0.65). Phase 1 has the WebUI surface but no auto-merge gate — the conservative defaults only matter when Phase 2 turns it on. Coming in tight and loosening after Phase 2 calibration against the dev's 2,379-row corpus is safer than coming in loose and discovering the false-positive rate at the user's expense. Loosening to 0.85 / 0.65 (or wherever the corpus calibration lands) is a Phase 2 deliverable.

### Defaults are configurable

`[memory.entity_merge]` config section (added to `dawn.toml`):

```toml
[memory.entity_merge]
enabled = true
auto_merge_threshold = 0.90
review_threshold = 0.70
weight_name_jaccard = 0.30
weight_embedding_cosine = 0.30
weight_exclusive_relation_overlap = 0.25
weight_contact_field_overlap = 0.10
weight_type_match = 0.05
bonus_name_substring = 0.10
bonus_user_self = 0.20
```

Calibrate against the developer's 2,379-row corpus during Phase 2 (auto-merge gate). Ship defaults as above; adjust if precision/recall drifts.

---

## 8. User-identity seeding (Q-C; point 6)

Two paths:

### Path A — New users (forward-going)

`auth_db_user_create()` calls a new `memory_db_seed_user_self_entity(user_id, username, persona_description)` after the user row is committed:

1. Compute display name: `display_name_from(username, persona_description)` — first token of `persona_description` if present and human-readable, else a beautified `username` (strip leading numerics, capitalize first letter), else fall back to the raw username.
2. Run the resolver against existing entities for the new user. Empty for fresh accounts; this is just defensive consistency.
3. INSERT `memory_entities (user_id, name=display_name, entity_type='person', canonical_name=canonical(display_name), is_user_self=1, mention_count=0, first_seen=now())`.
4. Insert seed audit rows in `memory_entity_aliases` for known surface forms:
   - `username` (always present)
   - First token of `persona_description` if it differs from `username`
   - Account email if the account was created with one (via a hook from oauth/contacts when an email lands later)
   These seed rows have `link_kind='soft'`, `reason='seeded'`, `composite_score=NULL`. Note: they don't set `canonical_id` on any other entity (no other entities exist yet to link); they're just authoritative aliases-of-record. They become live `canonical_id` links the first time the resolver matches the surface form to a fresh entity (which then gets the soft link applied retroactively in `_consider_auto_merge`).

The `is_user_self = 1` flag is what `canonical_priority` (§9) reads to give the seeded entity top priority during canonical selection.

### Path B — Existing clusters (one-time backfill for the dev's DB)

A new admin command `dawn-admin memory entity link-user-self [--username <u>] [--dry-run]`:

1. **Find or create the user-self canonical.** Search existing entities for the best resolver match against `(username, persona_description)`:
   - If any existing entity scores composite ≥ auto-merge threshold (default 0.90, with the user-self bonus computed against itself, equivalent to the entity-as-canonical case), USE that entity as canonical and set `is_user_self = 1` on it. This is the dev's expected case — `Kristopher Kersey` (id=1406, type=person, 21 mentions) should match strongly enough to be promoted to user-self.
   - Else, create a fresh seed entity per Path A.
2. **Run resolver across all entities** for that user_id, treating the user-self as an always-present additional candidate (with `user_self_bonus` applied).
3. **For each candidate scoring ≥ auto-merge threshold**: insert a soft link (set `canonical_id` + audit row, `reason='operator'` if dry-run was confirmed).
4. **For candidates in the review band**: enqueue in `memory_entity_merge_proposals` for the operator to review later via WebUI.
5. **Print report**: for the dev, expected output is something like:
   ```
   user-self canonical: Kristopher Kersey (id=1406)

   Auto-merged (composite ≥ 0.90):
     1230 'Kris' (thing)              → composite 0.91 (substring + works_at-overlap + user_self_bonus)
     1162 'user' (thing)              → composite 0.95 (email_is overlap + 292-mention authority + user_self_bonus)
     1747 'kerseyfabrications@gmail.com' (thing) → composite 0.92 (email_is exclusive-relation match + user_self_bonus)

   Queued for review (0.70 ≤ composite < 0.90):
     1735 'kerseyfabrications' (thing) → composite 0.78 (substring + lacks exclusive-relation evidence)

   Below threshold (no action):
     (none)
   ```
6. **Dry-run mode** prints the report without writing. Operator can review and re-run without `--dry-run`.

`dawn-admin memory entity split <link_id>` reverses any individual mistake.

---

## 9. Order-independence (invariant 9; point 4)

The resolver's binding decision must be a pure function of the cluster's members, independent of conversation visit order. Three deterministic mechanisms enforce this:

### Canonical selection rule

When the resolver finds multiple candidates above the auto-merge threshold (rare in practice — usually only the canonical-side's existing alias chain), the canonical is determined by lexicographic comparison of `canonical_priority`:

```
canonical_priority(entity) = (
  is_user_self,             -- bool: entity has is_user_self=1
  type_specificity,         -- person=4, place=3, pet=3, org=3, thing=1
  -mention_count,           -- higher mentions win (negate for max-then-min ordering)
  first_seen,               -- older = more established
  id                        -- absolute deterministic tiebreak
)
```

For the dev's cluster after seeding: `Kristopher Kersey` wins via `is_user_self=1` first criterion. No other entity has that flag for user_id=1, so the comparison short-circuits.

### Resolver runs against full entity set, not visit-state

The resolver loads ALL existing entities for the user from cache before scoring. It doesn't matter whether the new mention "Krys" arrives before or after "Kris" and "Kristopher Kersey" exist as rows — the canonical it picks is determined by `canonical_priority` against the existing rows visible at resolver time. If the existing rows haven't been linked yet (e.g. a previous extraction lacked sufficient evidence), then "Krys" lands as a new row and contributes to the cluster; the next resolver invocation that has all three plus their relations evaluates the cluster identically regardless of which one arrived last.

### Tiebreaks are stable across reextract

`id` (assigned at row creation, never reused) is the absolute fallback. `first_seen` is preserved by the existing primitive on hard merge (`MIN(first_seen, ?)`). `mention_count` is monotonically increasing within a session and aggregated on merge. Reextract drops derived tables and rebuilds, but the canonical chosen by the rebuild is determined by the same comparison rule against the same conversation evidence — so re-running the extract produces an isomorphic cluster shape. The specific `id` values may differ between reextract runs (rows get fresh AUTOINCREMENT), but within a single run the canonical is stable.

### What's deliberately NOT order-dependent

- Conversation *processing* order in reextract — `dawn-admin memory reextract` walks `conversations` in `id` order, but the resolver decisions don't depend on which conversation it sees first because they're scored against the entity set as it exists.
- Extraction batch boundaries — auto-merge runs per-entity inside `process_extraction_response`, and the candidate pool is the full live entity table (cache), not just the current extraction's locally-built `entity_map`.

The one remaining dependency: an auto-merge can only fire when sufficient signal exists. A cluster that's split because evidence accumulated slowly will eventually link once an exclusive-relation overlap or sufficient name signal lands. This isn't an order-dependence bug — it's the auto-merge gate intentionally being conservative until evidence supports the link.

---

## 10. Relation re-pointing semantics on merge (Q-D)

### Soft-link state (`canonical_id` set, source row present)

Relations stay attached to their original (alias) `entity_id`. No re-pointing. Read paths use `_list_by_subject_class` (§5) to UNION across the equivalence class.

**Why no re-point on soft?** Reversibility. Splitting a soft link is just zeroing `canonical_id`; the alias's relations are already on its row. Re-pointing on soft would make split require reverse-re-pointing, which loses the original `subject_entity_id` history.

### Hard-merge state (existing primitive runs)

The existing `memory_db_entity_merge()` primitive at `memory_db.c:2549` already handles this — it re-points `subject_entity_id` and `object_entity_id`, deletes self-refs, and dedups via `ROW_NUMBER() OVER (PARTITION BY subject_entity_id, relation, object_entity_id, COALESCE(object_value, ''))`.

What the primitive does NOT do today:
- **Collapse duplicate exclusive relations.** If A has `works_at = FOO` (open) and B has `works_at = BAR` (open) and both get re-pointed to canonical, both rows survive the dedup window-function (they have different `object_entity_id`s). After the merge, the canonical has two open `works_at` rows — violating `EXCLUSIVE_RELATIONS[]` semantics.

### New step: post-merge exclusive reconciliation

Add a step inside the existing `memory_db_entity_merge()` transaction (or as a follow-on in the same `BEGIN IMMEDIATE` block):

```c
/* For each exclusive relation type, if multiple open rows exist on
 * target after dedup, keep the most recent (highest valid_from, fall
 * back to highest confidence, then lowest id) and supersede the others. */
for (int i = 0; i < EXCLUSIVE_RELATIONS_COUNT; i++) {
   const char *rel = EXCLUSIVE_RELATIONS[i];
   /* SELECT id, valid_from, confidence FROM memory_relations
    *  WHERE user_id = ? AND subject_entity_id = ? AND relation = ?
    *    AND valid_to IS NULL
    *  ORDER BY COALESCE(valid_from, 0) DESC, confidence DESC, id ASC */
   /* If COUNT > 1, set valid_to = max(predecessors' valid_from, now)
    * on every row except the first. */
}
```

This ships as part of the entity-merge primitive update. It's a same-transaction operation; rollback semantics are preserved.

### Non-exclusive duplicates after re-pointing

The existing primitive's dedup handles exact duplicates (same `subject`, `relation`, `object`, `object_value`). It picks highest confidence. That's correct for non-exclusive relations and stays as-is.

### Fact-text references

`memory_facts.fact_text` references entities by string name, not FK. Rewriting "Kris works at FOO" → "Kristopher Kersey works at FOO" is lossy (loses user's original phrasing). **No-op: do not rewrite.** Search relies on the entity-graph merge to bridge during retrieval — the query "where does Kris work" hits `memory_embeddings_entity_search` which now resolves to canonical and surfaces facts attached to the canonical's relations. Fact text retains its original phrasing for provenance fidelity.

---

## 11. Contradiction-merge interaction (Q-E)

If A has `works_at = FOO` (open) and B has `works_at = BAR` (open) and they merge, the post-merge reconciliation step from §10 keeps the most-recent-by-`valid_from` and supersedes the older. This is exactly the `memory_db_relation_supersede()` semantic, applied retroactively after re-pointing.

**Policy:** auto-supersede the older. The most recent valid_from is the most authoritative open relation.

**Confidence check:** if both rows have NULL `valid_from` (no temporal anchor), fall back to higher `confidence`. If both confidences are equal, keep the one with the lower `id` (deterministic).

**Edge case — both open with NULL valid_from and equal confidence:** keep both is wrong (violates EXCLUSIVE_RELATIONS); supersede the higher `id`. Log a WARNING with both relations + the merge_id so the operator can investigate (this case shouldn't happen with proper extraction but is a real risk if the extraction LLM emitted contradictory facts in the same session).

The reconciliation runs inside the merge transaction, so rollback is atomic.

For soft-merge (no re-point), there's no contradiction interaction because relations stay on alias rows. Reads from the equivalence class will surface BOTH `works_at` open relations to the LLM — this is acceptable for the soft state because the operator hasn't yet confirmed the merge; it's better to surface the ambiguity than silently pick a winner. The auto-merge gate at extraction time DOES NOT touch existing relations; it only sets `canonical_id`.

---

## 12. Embedding-cache invalidation (point 7; in-scope fix)

### Current bug

`memory_db_entity_merge()` at `memory_db.c:2549` does NOT call `memory_embeddings_invalidate_entity_cache()` after a successful collapse. The cache (`s_entity_cache` in `memory_embeddings.c`) keeps stale embeddings for the deleted source entity_id; subsequent `memory_embeddings_entity_search()` may return phantom IDs that no longer exist. The `merge_entities` LLM tool action at `memory_callback.c:1219` doesn't invalidate either, but that's downstream of the primitive.

### Fix scope

Folded into Phase 1 of this workstream's first PR (per the "fold-in supersedence fixes" pattern).

### Where the invalidation fires

`memory_embeddings_invalidate_entity_cache()` is an atomic dirty-bit flip (`atomic_store(&s_entity_cache.dirty, true)`). It does not acquire `auth_db` mutex, does not acquire the `s_entity_cache.mutex`, and cannot self-deadlock. Safe to call any time.

**Three call sites get the invalidate:**

1. **`memory_db_entity_merge()`** (existing primitive) — insert after the COMMIT-success `AUTH_DB_UNLOCK()` and before the success `OLOG_INFO`, in the non-`merge_fail` path. Single line:
   ```c
   memory_embeddings_invalidate_entity_cache();
   ```
   The rollback path (label `merge_fail`) does NOT need the invalidate — `ROLLBACK` reverses the in-progress UPDATE and the cache state is consistent with the pre-merge DB.
2. **New `memory_db_entity_alias_link()`** (soft-link path) — same pattern: invalidate after `AUTH_DB_UNLOCK()`, before success log.
3. **New `memory_db_entity_alias_unlink()`** (split path) — same pattern.

### Why post-`AUTH_DB_UNLOCK()` and not pre

Post-unlock means readers can see the dirty flag immediately. Pre-unlock would also work (the dirty-bit flip is decoupled from the auth_db lock), but post-unlock follows the "release lock as soon as possible, do non-DB work after" pattern from `ARCHITECTURE.md` § Mutex Lock Ordering Hierarchy.

### Fact cache

`memory_embeddings_invalidate_cache()` (the *fact* cache) does NOT need invalidation on entity merge. Facts are unchanged by the merge — their `fact_text` is untouched, their FK references via `memory_relations.fact_id` may move when relations re-point, but the fact rows themselves are stable. No fact embeddings need re-loading.

---

## 13. Reversibility / audit (Q-F)

### Soft-undo (`dawn-admin memory entity split <link_id>`)

For a `memory_entity_aliases` row with `link_kind='soft'` and `unlinked_at IS NULL`:

1. Validate the link still exists and is active.
2. UPDATE `memory_entities SET canonical_id = NULL WHERE id = source_entity_id`.
3. UPDATE `memory_entity_aliases SET unlinked_at = strftime('%s','now'), unlink_reason = 'split-by-operator' WHERE id = ?`.
4. Invalidate entity embedding cache.

Trivial. The alias is independent again, with its original relations and its preserved `canonical_name`. Read paths immediately surface it as a separate entity.

For a `link_kind='hard'` row: refuse with `Error: this alias was hard-merged; the source entity row no longer exists. Use 'dawn-admin memory reextract' to fully rebuild the entity graph.`

### Hard-undo

Not supported short of `dawn-admin memory reextract`. The reextract utility already drops the `memory_entities`, `memory_relations`, and `memory_facts` tables and rebuilds from `conversations`/`messages`. Add `memory_entity_aliases` and `memory_entity_merge_proposals` to the list of tables dropped by reextract — they're derived state.

### Audit evidence

Every `memory_entity_aliases` row carries:
- `composite_score` (the float that triggered the band)
- `evidence_json` — JSON blob with all five signals + applied bonuses + the candidate's `entity_type`/`mention_count` snapshots at link time

Example evidence_json:

```json
{
   "name_jaccard": 0.50,
   "embedding_cosine": 0.78,
   "exclusive_relation_overlap": 1.0,
   "contact_field_overlap": 0.0,
   "type_match": 0.0,
   "applied_bonuses": ["name_substring", "user_self"],
   "source_snapshot": {"entity_type": "thing", "mention_count": 28, "first_seen": 1745000000},
   "target_snapshot": {"entity_type": "person", "mention_count": 21, "first_seen": 1741000000},
   "shared_evidence": [
      {"signal": "exclusive_relation_overlap", "relation": "works_at", "object": "Adtech Global Solutions"}
   ]
}
```

This is what `dawn-admin memory entity history <entity_id>` and the WebUI "why was this merged?" affordance render.

---

## 14. UX surface (Q-G; point 8)

### Phase 1 — WebUI: alias visibility

**New WebSocket message types** (DAWN's existing memory operations are WebSocket-based; matches `webui_memory.c` patterns):

1. **`get_entity_aliases`** (request) → **`entity_aliases_response`** (response):
   - Request: `{type: "get_entity_aliases", entity_id: 1406}`
   - Response: `{type: "entity_aliases_response", entity_id: 1406, aliases: [{alias_entity_id: 1230, name: "Kris", canonical_name: "kris", entity_type: "thing", mention_count: 28, link_id: 42, linked_at: 1746000000, link_kind: "soft", composite_score: 0.91}, ...]}`
   - Used by Graph tab on entity-card expand to show nested aliases.

2. **`split_entity_alias`** (request) → **`split_entity_alias_response`**:
   - Request: `{type: "split_entity_alias", link_id: 42}`
   - Response: `{type: "split_entity_alias_response", link_id: 42, success: true, restored_entity_id: 1230}` (or `success: false, error: "..."` for hard-merge / not-found / not-owned)

3. **`propose_entity_merge_preview`** (request) → **`propose_entity_merge_preview_response`**:
   - Request: `{type: "propose_entity_merge_preview", source_entity_id: 1230, target_entity_id: 1406}`
   - Response: `{type: "propose_entity_merge_preview_response", composite_score: 0.91, evidence: {…}, would_auto_merge: true, would_queue_for_review: false, would_reject: false}`
   - Used by the WebUI to show a confidence badge on the existing two-click merge UI before the user clicks confirm.

4. **`list_pending_merge_proposals`** (request) → **`pending_merge_proposals_response`**:
   - Returns rows from `memory_entity_merge_proposals WHERE resolved_at IS NULL` for the current user. Powers the new "Suggested merges" panel.

5. **`resolve_merge_proposal`** (request) → **`resolve_merge_proposal_response`**:
   - Request: `{type: "resolve_merge_proposal", proposal_id: 17, resolution: "approved" | "rejected"}`
   - On `approved`: writes the soft link (canonical_id + audit row), stamps `resolved_at`/`resolution`.
   - On `rejected`: stamps `resolved_at`/`resolution`, no entity changes.

**REST endpoints** (parity with existing `GET /api/memory/{facts,preferences,...}`):

- `GET /api/memory/entities/:id/aliases`
- `GET /api/memory/merge-proposals` (pending only by default; `?resolved=1` for history)
- `DELETE /api/memory/entity-aliases/:link_id` (alias for split; same as the WebSocket message; for HTTP-tooling parity)

**WebUI Graph tab changes**:
- Each canonical entity card gets a new "aliases (N)" badge when N > 0.
- Expanding an alias row shows: alias name + type + mention_count + linked_at + composite_score + a "split" button.
- The existing two-click manual merge UI gains a confidence badge powered by `propose_entity_merge_preview`.
- New "Suggested merges" panel above the entity list lists pending proposals; each has approve/reject buttons.

**Behavior change to existing `merge_entities` LLM tool action and WebUI two-click**: both default to **soft merge** (set `canonical_id` + audit row, source row present). Hard merge is operator-only via `dawn-admin memory entity consolidate`. The WebUI surface explains this in the merge confirm dialog: "The source entity will be marked as an alias of the target. You can split it later. Use `dawn-admin memory entity consolidate` to make this permanent."

**Post-Phase-2 purpose of the LLM `merge_entities` action**: once Phase 2's auto-merge gate is live, the resolver handles high-composite cases automatically at extraction time. The LLM tool action remains valuable for cases the gate didn't fire on — e.g. the resolver scored a pair in the 0.70–0.90 review band and queued a proposal, but the LLM has additional context from the current conversation that establishes equivalence (a user explicitly says "Kris and Kristopher are the same person"). The LLM action **bypasses the proposal queue** with explicit LLM judgment, soft-linking directly. This positions the action as the LLM's escape hatch for mid-confidence cases the operator hasn't reviewed yet.

The LLM tool action's response text changes: `"Linked 'Kris' as an alias of 'Kristopher Kersey'. The 'Kris' entity is now a soft alias and will be resolved to 'Kristopher Kersey' on lookup. Use 'split' to undo."`

### Phase 2 — `dawn-admin memory entity` subcommands

```
dawn-admin memory entity list [--user <id>] [--include-aliases]
dawn-admin memory entity merge <source_canonical> <target_canonical> [--soft|--hard]
dawn-admin memory entity split <link_id>
dawn-admin memory entity aliases <canonical_or_id>
dawn-admin memory entity link-user-self [--username <u>] [--dry-run]
dawn-admin memory entity propose-merges [--threshold <f>] [--dry-run]
dawn-admin memory entity consolidate [--all | --link-id <id> | --older-than <duration>] [--dry-run]
dawn-admin memory entity history <entity_id_or_canonical>
```

`--soft` (default) writes the `canonical_id` link. `--hard` runs the existing primitive directly (skipping the soft state). `consolidate` promotes existing soft links to hard.

`propose-merges` runs the resolver across all entity pairs offline, prints a sorted-by-composite report, and (without `--dry-run`) inserts proposals for the review band and auto-merges the ≥auto-threshold band (per configured thresholds; first-ship 0.90 / 0.70). This is the bulk-backfill path for retroactive cleanup that doesn't go through the user-self special case.

### Phase 3 — Settings / config exposure

`SETTINGS_SCHEMA` (`www/js/ui/settings.js`) gains `entity_merge` group: enable toggle + the seven config knobs from §7. Mirrored in `dawn.toml` via `[memory.entity_merge]`.

### Phase 4 — Extraction-time logs

`OLOG_INFO("memory_extraction: alias auto-merged %s (id=%lld) → %s (id=%lld), composite=%.2f, ...")`.

`OLOG_INFO("memory_extraction: merge proposal queued: %s → %s, composite=%.2f")`.

Both are observability-only; existing log pipeline.

### Contacts coupling (out of scope, per brief)

`contacts_db` links to entities by name match in extraction-time logic, NOT by `entity_id` foreign key. Adding a hard FK is **out of scope for this design** (per brief §G). What IS in scope: ensuring alias resolution helps the existing name-match — and it does, automatically. When `contacts_db.contacts.entity_id` is populated by name match against `memory_entities`, the resolver's exact-canonical_name lookup (Stage 1) returns the canonical even if extraction stored the contact under an alias name. The contact row's `entity_id` therefore points at the canonical from the start. Existing behavior is preserved; no contacts_db changes.

---

## 15. Performance & indexes (Q-H)

### Hot-path query plans

**Extraction-time alias resolver** (§6 cascade):
- Stage 1: `SELECT id, canonical_id FROM memory_entities WHERE user_id = ? AND canonical_name = ?` — uses existing `idx_memory_entities_user` + UNIQUE constraint on `(user_id, canonical_name)`. Single row lookup, sub-ms.
- Stage 2: per-token `SELECT id, name, entity_type FROM memory_entities WHERE user_id = ? AND canonical_name LIKE ?% LIMIT 32` — uses `idx_memory_entities_user` + LIKE index scan. Up to 4 token loops × ≤32 rows. 1–3ms.
- Stage 4: cosine over RAM-resident `s_entity_cache` (already loaded). No DB.
- Stage 5: per-candidate `SELECT relation, object_entity_id, object_value FROM memory_relations WHERE user_id = ? AND subject_entity_id = ? AND valid_to IS NULL AND relation IN (...)` — uses `idx_memory_relations_subject_open` (partial index on `valid_to IS NULL`). 3 candidates × ≤12 relations each. <5ms.
- Plus `SELECT field_type, value FROM contacts WHERE user_id = ? AND entity_id = ?` — uses `idx_contacts_user_entity` (existing). <1ms per candidate.

**Search-time entity recall** (`append_graph_context`):
- `memory_embeddings_entity_search`: cache lookup, no DB. <1ms.
- After cache rebuild filters `WHERE canonical_id IS NULL`, only canonical rows land in the cache — same shape, same cost.

**Search-time relation listing**:
- `memory_db_relation_list_by_subject_class`: `SELECT … WHERE subject_entity_id IN (SELECT id FROM memory_entities WHERE id = ? OR canonical_id = ?)`. Inner SELECT uses the new `idx_memory_entities_canonical` partial index. Outer uses existing `idx_memory_relations_subject`. 2-step but indexed; ≤5ms in practice.
- **Implementation must verify** via `EXPLAIN QUERY PLAN` that the partial index is consulted on the inner SELECT. SQLite's planner generally handles `IN (subquery)` correctly with partial indexes, but `OR` inside the subquery can defeat index use depending on planner version. If the plan shows a full scan, rewrite as `UNION ALL`: `SELECT id FROM memory_entities WHERE id = ? UNION ALL SELECT id FROM memory_entities WHERE canonical_id = ?`. Same row set, planner-friendly.

**Focus block entity recall**:
- Reads the same `s_entity_cache`. The 256-entity scratch buffer in focus adapters is rebuilt from the cache; aliases pre-filtered out. Zero new DB hits per turn.

### Indexes

**New:**
- `idx_memory_entities_canonical ON memory_entities(canonical_id) WHERE canonical_id IS NOT NULL` — partial; supports `WHERE canonical_id = ?` lookups (alias enumeration).
- `idx_memory_entity_aliases_user_target ON memory_entity_aliases(user_id, target_entity_id) WHERE unlinked_at IS NULL` — partial; supports the WebUI `get_entity_aliases` query.
- `idx_merge_proposals_pending ON memory_entity_merge_proposals(user_id, proposed_at) WHERE resolved_at IS NULL` — partial; supports `list_pending_merge_proposals`.

**Existing indexes that already cover new query patterns:**
- `idx_memory_entities_user`: covers the user-scoped scans in Stages 1–2.
- `idx_memory_relations_subject_open`: already partial on `valid_to IS NULL`, covers Stage 5 exclusive-relation lookup.

### Embedding-cache contract (point 7 reiterated)

`memory_embeddings_invalidate_entity_cache()` is an atomic dirty-bit flip — no auth_db lock involvement, no self-deadlock risk. Fires after `AUTH_DB_UNLOCK()` in three places: existing `memory_db_entity_merge()` (in-scope fix), new `memory_db_entity_alias_link()`, new `memory_db_entity_alias_unlink()`. The fact cache (`memory_embeddings_invalidate_cache()`) is NOT invalidated by entity merge — facts are unchanged.

The cache loader (`entity_cache_load` in `memory_embeddings.c:664`) gets a small change: filter `WHERE canonical_id IS NULL` so aliases never enter the cache. The `s_entity_scratch` buffer in `memory_focus_adapters.c` shares the same source query (`memory_db_entity_get_embeddings`); inheriting the filter is automatic if we update `memory_db_entity_get_embeddings` itself, OR per-caller if we add a flag. Specify: update the underlying `memory_db_entity_get_embeddings` to default to canonical-only (with an optional `include_aliases` parameter for the future Graph tab "show all rows" view).

### TSan / lock-discipline notes

The resolver runs inside `process_extraction_response`, which holds no auth_db lock at call time (it's between LLM call and the DB writes). Each resolver SQL runs through `AUTH_DB_LOCK_OR_FAIL()` / `AUTH_DB_UNLOCK()` independently — no nesting. The cosine math reads from `s_entity_cache.mutex` (separate from auth_db). Lock ordering: auth_db (leaf, per-statement), then s_entity_cache.mutex (for cosine), then s_entity_scratch_mutex (focus adapters). No new ordering dependencies.

---

## 16. Operator interface — `dawn-admin memory entity` subcommand shapes

Detailed in §14 Phase 2. The wire protocol matches existing `dawn-admin` patterns:
- New `ADMIN_MSG_MEMORY_ENTITY_*` opcodes in the 0x83+ range (after `ADMIN_MSG_MEMORY_REEXTRACT_STATUS = 0x82`).
- Binary payload ≤240 bytes per message, fits within the 256-byte cap.
- Status responses use `ADMIN_MSG_MEMORY_ENTITY_*_STATUS` and stream over the existing detached-thread pattern from `memory_db_admin.c`.

The authoritative subcommand signatures live in §14. The implementation lands in a new `src/memory/memory_db_admin.c` extension (`memory_db_admin_entity_*`) and a new `dawn-admin/cmd_memory_entity.c` for the client side.

---

## 17. Phasing

Effort estimates use the project's `agent ~Xh · api ~$Y · N ckpt` convention from feedback memory.

### Phase 1 — Schema + soft-merge plumbing + WebUI alias view + cache-invalidate fix (smallest user-visible win)

Ship together because they compose: schema is a prerequisite, soft-merge plumbing is the read/write path, WebUI surface makes it visible, cache-invalidate fix folds in.

Includes:
- v43 migration: `canonical_id`, `is_user_self`, `memory_entity_aliases`, `memory_entity_merge_proposals`, indexes (including the partial unique index `idx_memory_entities_user_self ON memory_entities(user_id) WHERE is_user_self = 1` to enforce one-self-per-user)
- New `memory_db_entity_resolve_alias`, `memory_db_entity_alias_link`, `memory_db_entity_alias_unlink`, `memory_db_entity_relation_list_by_subject_class` (+ `_at` and `_by_object` siblings) — implemented in a **new `src/memory/memory_db_alias.c` + `include/memory/memory_db_aliases.h`** following the v42 split pattern that produced `memory_db_entities.h` / `memory_db_embeddings.h`. Keeps `memory_db.c` from crossing the soft size limit
- Cache-invalidate fix in existing `memory_db_entity_merge()` and the cache loader filter (`canonical_id IS NULL`)
- **Reextract drop-list update**: `memory_entity_aliases` + `memory_entity_merge_proposals` added to `memory_db_admin.c`'s reset list now (not Phase 3) — closes the Phase 1 → Phase 3 window where reextract would leave orphaned audit rows pointing at deleted target entities
- 5 new WebSocket message types + 3 REST endpoints
- WebUI Graph tab updates (alias-nested entity cards, confidence badge, suggested-merges panel)
- `merge_entities` LLM tool action becomes soft by default; response text updated
- `dawn-admin memory entity merge|split|aliases|history|list|link-user-self` subcommands (manual paths only — no auto-merge yet)
- Unity tests for resolver cascade (Stages 1–6 individually, composite scoring, threshold bands, canonical_priority lexicographic comparison, embedding cache invalidation roundtrip)

**Effort: agent ~12h · api ~$1 · 5 ckpt.** No auto-merge gate yet — the operator drives every link via WebUI or admin command. This is the minimum viable user-visible win: the dev runs `dawn-admin memory entity link-user-self --dry-run` to see the proposed aliases for the existing 21+28+292-relation cluster, runs without `--dry-run` to apply them, and the focus block immediately stops surfacing duplicate entities.

### Phase 2 — Auto-merge gate at extraction

Includes:
- `_consider_auto_merge` integrated in `process_extraction_response`
- `propose-merges` admin command
- `[memory.entity_merge]` config + `SETTINGS_SCHEMA` exposure
- Threshold calibration against the dev's corpus (run Phase 1 backfill, observe outcomes, adjust default weights if precision/recall drifts)
- Unity tests for the full extraction integration, mid-confidence proposal queue lifecycle

**Effort: agent ~6h · api ~$1.50 · 3 ckpt.** API spend dominated by re-running extraction across the dev's 272 conversations to validate auto-merge precision (~$1 per full reextract on Haiku, plus calibration runs).

#### Implementation notes (shipped 2026-05-12, commit `9bfda37`)

Phase 2 landed in one commit covering the full gate surface — auto-merge at extraction, runtime config, WebUI proposal indicator, and operator review flow. The shipped behavior diverged from the original §17 outline in five places worth recording for future workstreams (Phase 3 hard-merge, threshold recalibration, cross-encoder reranker, anchor-aware resolver).

1. **Propose-all-in-band was an addition not in the original spec.** Original spec described single-winner-per-source: cascade picks the best Stage-6 candidate, routes to AUTO/REVIEW/no-op once. Shipped behavior: `cascade_internal` now optionally populates an array of every Stage-6-scored candidate, sorted by composite DESC. `consider_auto_merge` iterates that list — AUTO short-circuits when the top winner reaches `auto_threshold` (source becomes a soft alias, sole outcome), otherwise EVERY candidate at or above `review_threshold` produces a separate proposal row. Rationale: false-positive cost is one operator click in the WebUI; false-miss cost is a duplicate entity that survives extraction and persists. Live-validated on the dev's cluster — Kristopher Kersey legitimately matched both Kris (correct, accepted) and Shelley Kersey (shared "kersey" token, false positive, rejected). Both surfaced as expected; the prior winner-only path would have hidden one or the other depending on which composite ranked higher.

2. **Gate timing moved from inline-during-entity-upsert to post-relations sweep.** Original spec wired the gate inline during the entity loop in `process_extraction_response`. Shipped behavior: new `apply_phase2_merge_gate()` helper in `memory_extraction.c` tracks `fresh_entity_ids[ENTITY_MAP_MAX]` of newly-created entities and runs the cascade AFTER the relations loop completes. Reason: inline scoring saw `exclusive_relation_overlap = 0` for every candidate because relations had not yet been processed — Stage 2 substring rescue found `Kris ⊂ Kristopher Kersey` as a candidate, but composite stayed below threshold without the dominant relation-overlap signal. Broadcast to clients was also coalesced to a single fire-at-end (was per-proposal, which thrashed the DB lock and the WebUI memory-icon dot animation when multiple proposals fired in one extraction).

3. **Two auto-promote helpers replaced the CLI-only flow.** Original spec made `dawn-admin memory entity link-user-self` the required first-run setup step (§8 Path A / Path B). Shipped behavior: `memory_db_entity_maybe_auto_promote_user_self()` fires inline at extraction when a fresh entity's canonical_name matches `users.real_name` AND no user_self anchor exists yet for that user; `memory_db_entity_auto_promote_user_self_by_real_name()` runs as a sweep on Settings save to catch the case where `real_name` is set AFTER matching entities already exist in the DB. Both check exact canonical-form match against `real_name` + each `identity_alias` line. The CLI tool remains useful for heavy cluster cleanup and edge cases (manual link of a name variant that doesn't pass the exact-match gate) but is no longer required for first-run UX. To keep the per-entity cost flat, the `find_user_self_id` lookup is lifted to once-per-extraction via a cached `user_self_already_exists` flag — the inline auto-promote check only acquires the DB lock once per extraction in steady state, not once per fresh entity.

4. **Stage 2 gained reverse-direction substring rescue.** Original §6 described forward substring rescue only (inbound name as substring of an existing canonical). Shipped adds `stage2_reverse_substring_candidates()` using SQL `instr(?, canonical_name) > 0` to find shorter existing canonicals whose names appear inside a long-form inbound. Closes the symmetric gap where the per-token LIKE never finds "Kris" when searching "kristopher kersey" — the existing forward path catches the inbound short form, but a fresh long-form needs the reverse direction to find the existing short form. Bounded with `length(canonical_name) <= length(inbound)` SQL pre-filter to skip rows that can't possibly be a substring, and skipped entirely when the inbound is < 8 characters (the reverse direction can't usefully match anything when the inbound itself is a short form).

5. **CONFIG_CLAMP NaN hardening (codebase-wide side-effect).** Runtime thresholds are exposed via the `[memory.entity_merge]` config keys `enabled` / `auto_threshold` (0.90 default) / `review_threshold` (0.50 default, down from the compile-time 0.70 used during Phase 1 — the shipped propose-all-in-band path makes review-band recall more valuable than precision since the operator is gating). The `CONFIG_CLAMP` macro was rewritten codebase-wide to catch NaN / +inf / -inf — IEEE 754 NaN compares false to everything, so the old `(val < lo || val > hi)` form silently let NaN through every validator. The new negated-range form `!((val) >= (lo) && (val) <= (hi))` rejects NaN as out-of-range. Side-effect security improvement reaches every config validator in the project, not just entity_merge.

**Deferred follow-up: prefer longer canonical name for person/pet/place.** Today's cascade picks the existing canonical as the merge target regardless of which name form is "fuller" — direction is determined incidentally by extraction order (whoever was inserted first wins canonical, whoever arrives second becomes alias). For person/pet/place entity types the intuition is "longer wins" (Kristopher Kersey canonical, Kris alias). Doesn't apply uniformly to `org` (acronyms like IBM are often the canonical form in common usage) or `thing`. Fix shape: in `memory_db_entity_consider_auto_merge`, after the cascade picks `winner_id` (target) and we know `entity_id` (source), if both are person/pet/place AND source has strictly more whole-word tokens than target AND target has no dependents (no aliases pointing at it, no exclusive open relations as subject), swap source↔target before `alias_link`. The "no dependents" guard avoids orphaning a subtree when the existing canonical is the head of an equivalence class — Phase 3 hard consolidate handles relation migration when that's wanted. The user-self entity already gets "longer wins" via the `users.real_name` + auto-promote anchor, so this fix mainly affects spouse / kids / friends / family clusters where no real_name anchor exists.

### Phase 3 — Hard-merge consolidation + bulk backfill

Includes:
- `dawn-admin memory entity consolidate` (soft → hard burn-in, operator-driven)
- `dawn-admin memory entity propose-merges --apply` (offline scan across all entity pairs, not just user-self)
- Documentation pass (atlas `SYSTEM_DESIGN.md` schema section bump)

**Effort: agent ~4h · api ~$0.10 · 2 ckpt.** No new model calls; mostly plumbing + atlas update.

### Total

**agent ~22h · api ~$2.60 · 10 ckpt** across all three phases. The brief's "~2-3 days" estimate (in `STATE.md` medium-term workstream table) lines up roughly with Phase 1 + Phase 2 in agent-hours; Phase 3 is post-validation polish.

---

## 18. Open questions / follow-ups

These are items the design DOES NOT decide; deferred or adjacent to entity-merge.

### Deferred to follow-up TODO

1. **Anchor-aware resolver** — should the resolver weight evidence from the conversation's `anchor_date` window? Two relations with the same `object_entity_id` but different `valid_from` epochs are weaker evidence of merge than two co-temporal relations. Current design ignores temporal coincidence. Probable +0.05 weight on `exclusive_relation_overlap` when both rows have `valid_from` within ±90 days.

2. **Cross-user alias contamination** — `is_user_self = 1` is per-user, but the resolver's stage-2 LIKE search is correctly scoped to `user_id`. What about multi-user satellites where two users mention the same external entity (e.g. both say "my coworker Bob")? The user-id scoping correctly keeps these separate — Bob-of-user-A and Bob-of-user-B are independent rows, no cross-contamination risk. Documented for clarity.

3. **Reranker interaction** — RERANKER_INVESTIGATION.md closed cross-encoder reranking as dead. Entity-merge doesn't depend on reranking; if reranking is ever revisited, alias resolution at retrieval still works (canonical IDs surface; reranker scores them).

4. **Embedding-only merges** — when both candidates have only name evidence (no relations, no contacts) and embedding cosine alone fires, the composite is `0.30 × jaccard + 0.30 × cosine ≤ 0.60` even at perfect cosine = 1.0 unless name_jaccard also lifts. This intentionally suppresses embedding-only merges (false-positive prone for similar-meaning generic nouns). Confirm by testing against the dev's corpus.

5. **Canonicalization-rule formalization** — `memory_make_canonical_name` does ASCII-lowercase + UTF-8 passthrough + trailing-space trim. It does NOT normalize accents (`Mélanie` ≠ `melanie`) or strip honorifics (`Mr. Bob` ≠ `bob`). The resolver's name_jaccard inherits this gap. If non-Latin user identities become common, revisit canonicalization first.

### Adjacent issues spotted

6. **`merge_entities` LLM tool action is currently hard-merge.** The behavior change to soft-by-default in §14 affects an LLM-callable tool. The LLM may have past behavior expectations about it. The tool-description string in the metadata needs updating to reflect "creates a soft alias" instead of "permanently merges."

7. **`contacts_db` contact-to-entity FK as a future workstream.** Per brief, out of scope for this design. But the seeding path could naturally add `contacts_db` entries for the seed user-self entity (registered email, registered phone if any), populating the contact_field_overlap signal for the auto-merge gate. Concrete enough to be a separate ~2h follow-up.

8. **Reextract drops alias state — migration UX.** When the dev next runs `dawn-admin memory reextract`, the alias state is dropped along with the entities and the dev re-runs `link-user-self`. The reextract output should mention this prompt-style: "Aliases: 4 soft links dropped. Run `dawn-admin memory entity link-user-self` to rebuild." Enhancement to `memory_db_admin.c` reextract status text.

9. **Atlas SYSTEM_DESIGN.md drift** — `atlas/dawn/memory/SYSTEM_DESIGN.md` §6.4 doesn't mention `canonical_id`/`is_user_self` after this lands. Update is part of Phase 3 as noted, but it's also one of the cohesion-debt items in `dawn/docs/TODO.md`. Bump that item explicitly when atlas updates ship.

10. **Operator visibility into auto-merge log volume** — once Phase 2 fires, every extraction call may produce 0–N `OLOG_INFO("memory_extraction: alias auto-merged …")` lines. For a busy multi-user deployment this could be log-noisy. Rate-limit or aggregate? Defer until a deployment surfaces it as noise.

---

## 19. Out of scope (per brief — explicit)

- **Implementation code.** This doc is design only. `.c`/`.h`/migration SQL files do not land here.
- **Memory-subsystem redesign.** Entity merge is a localized fix to the entity surface. Summary structure, fact storage, document RAG — all unchanged.
- **Cross-encoder reranking, contradiction LLM judgment, multi-tenant deployment hardening.** Adjacent or future work; not gating entity-merge.
- **Hard `entity_id` FK on `contacts_db`.** Per brief; alias resolution helps the existing name-match lookup, no schema change.
- **Multi-language canonical normalization.** Open-question follow-up #5; not a blocker for the dev's English-only corpus.

---

*End of design doc. Draft date: 2026-05-09. Author: design-lead agent. Routes through `architecture-reviewer` per coordination protocol.*
