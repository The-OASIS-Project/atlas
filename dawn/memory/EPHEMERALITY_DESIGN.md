# Memory Ephemerality / Fact-Lifecycle Axis (C3) — Design + Ship Record

**Status:** **Phase 1 SHIPPED + LIVE-VALIDATED 2026-06-05.** Commits `b30a16c` (schema v58 +
retrieval guard + prune + extraction tagging + config + WebUI panel) and `3912310` (memory-tool
anti-bluff + object-form). `expire_enabled` defaults **off**; the one remaining gate before
default-on is the B1-style recall-neutrality bench (§7). Architecture-reviewed 2026-06-04;
implemented + validated on the real DB across two live test sessions 2026-06-05. Atlas ship record
(moved here from `dawn/docs/` on completion). Drafted from the C1 diagnosis; §1–§9 are the original
design rationale, the Implementation note below is the as-shipped record (supersedes §4/§5 where they
differ).

## Implementation note (2026-06-05) — what actually shipped (supersedes §4/§5 where they differ)

Tagging is **dual-path and AI-decided.** Deterministic date inference was explicitly rejected by the
dev — too prone to misflagging without a judge in the loop. Both tagging paths put a model in the
middle:

1. **Extraction (dominant path)** — the session-end extraction LLM emits an optional per-fact
   `expires_at` for transient dated facts. The prompt teaches it to tag forecasts,
   scheduled-not-yet-happened events, and — the key fix after the first live test missed every movie
   — **instantaneous point-events** (a movie/appointment on a day gets NO relation `valid_to` but DOES
   get a fact `expires_at`). Applied on fresh creates AND **adopted on paraphrase/LIKE merges**: a
   transient fact often arrives first via the `remember` tool (or a prior session) and is re-stated at
   extraction, so the merge path is exactly where it gets reinforced — skipping it would leave the
   fact immortal (the precise failure the live test surfaced). Only ever SETs from an inbound
   AI-emitted date, never clears. *(My first pass gated this to fresh creates only; that was backwards
   for the common case and was corrected to adopt-on-merge.)*
2. **`remember` tool** — array elements may be a plain string (durable) or
   `{"text": "...", "expires_at": "YYYY-MM-DD"}` (transient); sets expiry on store. The tool
   description also carries an **anti-bluff** clause — the model was acknowledging "keep in mind"
   requests in text without calling the tool (commit `3912310`).

**Retrieval guard** (§5's original call-site list was inaccurate — it named maintenance/write paths
like `memory_embeddings.c:345` category backfill, `memory_db.c` stats/decay/prune,
`memory_embed_recompute.c:234`, all of which must KEEP seeing expired facts):

- **Keyword/BM25/list SQL statements** (`auth_db_statements.c`: `fact_list`, `fact_search`,
  `fact_search_since`, `fact_search_bm25`, `_bm25_since`, `fact_list_since`,
  `fact_list_window_asc/desc`) carry `AND (expires_at IS NULL OR expires_at >= ?N)`, guard bound as
  the **highest-numbered param** (numbered params → no existing bind index shifts), value
  `expire_enabled ? time(NULL) : 0` (`fact_expiry_guard_now()`) → disabling admits everything
  instantly, no row mutation.
- **Two by-id retrieval materializations** that feed the LLM focus block are guarded in C via
  `memory_db_fact_expiry_hidden()` (mirrors the SQL guard): the vector-only hit in
  `memory_fact_search.c`, and the entity-graph 1-hop expansion in `memory_graph_retrieval.c`
  (`memory_graph_expand_fact_linked` — graph_retrieval defaults ON; a scheduled event reachable via a
  relation triple is exactly the C3 target, so it's load-bearing). The second site was missed in the
  first pass and caught by the architecture review — **any future id→fact retrieval materialization
  must call the predicate.** The fact-embedding cache is shared with extraction-time dedup, so it is
  intentionally NOT filtered. `memory_db_fact_get` carries `expires_at` (projection col 10); all
  other fact readers default it to 0.
- **Maintenance paths NOT guarded** (correct): dedup hash/like lookups, embedding recompute,
  category backfill, prune DELETEs, by-id fetch.

**Storage / prune / config:** schema **v58** `memory_facts.expires_at` + partial index `WHERE
expires_at IS NOT NULL` (migration verified on the live 1985-row DB — column nullable, existing rows
untouched). Setter `memory_db_fact_set_expires_at()`; nightly `memory_db_fact_prune_expired()` after
`prune_superseded`, gated. Config `[memory] expire_enabled` (off) / `expire_grace_days` (7) /
`prune_expired_days` (30), surfaced in the WebUI Memory panel. Tests: `test_memory_provenance`
(guard hide/restore + prune + predicate).

**Live validation (2026-06-05):** tool-path expiry fired (`remember` object form → `memory_callback:
remember set expires_at`), extraction tagged movies/festival/appointments (+7d grace),
**merge-adoption** tagged plain-string tool facts at session end, durable facts (preferences,
identity) left NULL, the multi-day trip used its **end date**, and the anti-bluff + object-form
nudges both confirmed live (Friday called `remember` unprompted and reached for the object form on
the clearly-dated fact).

**Known residual (deliberately NOT auto-fixed, per the no-misflagging-without-a-judge rule):**
near-duplicate asymmetry — when extraction creates a near-dup (cosine just below the 0.92 merge
threshold) of a plain-string tool fact instead of merging, the un-tagged twin can outlive the
expiring one. Left to the AI's existing `find_duplicates` / `forget --replaced_by` cleanup (which
Friday does spontaneously). An automated expiry-propagation pass was considered and **rejected** — it
reintroduces exactly the misflag surface C3 deliberately avoids.

**Next:** the B1-style recall-neutrality bench (Mem0 protocol, `expire_enabled` on) before flipping
default-on; then the deferred legacy back-fill (§6). Phase 2 (durability class) only if a fuzzy tail
remains.

---

## 1. The problem (from the C1 diagnosis, live DB, user 1, 2015 active facts)

The memory corpus does **not** bloat from duplication — the 0.92 extraction gate works and there
is no safe automated dedup (see Track C / C1). It bloats from **transient facts that never
expire**:

- **4.8% weather/forecast** facts ("Sugar Hill forecast for 2026-04-12: clear, 83.5°F").
- **17.5% date-pinned** facts, many of them **stale event-states** ("RayNeo glasses arriving
  tomorrow" — later made moot by "received them Saturday", but stored as two facts).

These carry an **implicit expiry**: a forecast for a past date, or a "happening tomorrow" that has
since happened, has near-zero forward value. Yet nothing removes them.

## 2. What already exists (and why it doesn't catch this)

- **Confidence decay** (`[memory] decay_*`, nightly): a weekly multiplier per `source` class —
  `inferred` 0.95/wk (floor 0.0), `explicit` 0.98/wk (floor 0.50), `preference` 0.97/wk
  (floor 0.40). Lowers `confidence` over time.
- **Prune paths** (`memory_db_facts.c`): `prune_stale(user, stale_days, min_confidence)` deletes
  facts below a confidence threshold past an age; `prune_superseded(user, retention_days)`
  hard-deletes superseded rows past a retention window.
- **Nightly orchestration**: `memory_run_nightly_decay()` (`memory_maintenance.c`), single-caller
  from the auth-maintenance thread, hour-gated with a ~20h re-entry guard; per user it runs
  decay → prune_low_confidence → prune_superseded → prune_old_summaries.
- **`source`** is the only lifecycle axis. Assigned at extraction (`memory_extraction.c`; default
  `inferred`, `explicit` for corrections at 0.9 confidence).
- **Bitemporal `valid_to`** exists on **relations** (not facts); a relation leaves retrieval once
  `valid_to` passes. Facts have **no TTL**.

**Why transient facts survive:** weather/event facts get stored high-confidence (`explicit` or
high `inferred`), so decay never drives them below the prune-stale floor — **immortal by
classification accident**.

**Category is not the answer.** `category` is already retrieval-dead (category-free search shipped)
and noisy (77 cross-category clusters in the C1 inventory). It is a *topic* label, orthogonal to
*lifecycle*. C3 does not fix lifecycle via category.

## 3. Design space

A prerequisite fork (raised in review): **could a transient "happening tomorrow" just be modeled
as a bounded *relation* (`valid_to`) instead of a new fact axis?** Partly — a scheduled event with
clear subject/predicate/object *can* be a bounded relation, which already leaves retrieval when
`valid_to` passes. But most of the measured bloat is **fact-shaped, not relation-shaped**: a
weather forecast ("Sugar Hill, 83.5°F, clear") has no subject→predicate→object triple, and the
extractor stores it as a free-text fact. So facts need their **own** expiry axis; the relation
`valid_to` path covers only the minority that are cleanly relational. §4 keeps both: crisp
scheduled *events* may route to bounded relations; forecast/snapshot *facts* get `expires_at`.

### Option 1 — Durability class at extraction (`ephemeral | durable`)
Extraction tags each fact; ephemeral facts get an aggressive decay class so `prune_stale` reaps
them.
- **+** Reuses decay+prune; one enum. **−** Coarse (no notion of *when*); a forecast for next week
  would start decaying while still useful.

### Option 2 — Expiry timestamp (`expires_at` on facts)
Extraction sets `expires_at` = referenced-date + grace; expired facts leave retrieval and are
later hard-pruned.
- **+** Precise; directly attacks the measured date-pinned bulk; the conversation **anchor already
  resolves relative→absolute dates** at extraction, so the hard half is built. **−** Needs the
  extractor to emit an expiry; undated transient facts uncovered.

### Option 3 — Hybrid (recommended): `expires_at` for dated facts + a durability class for the rest
Dated transient facts get a precise `expires_at`; fuzzy transient facts get the ephemeral decay
class; durable facts untouched.
- **+** Covers crisp-dated (precise) and fuzzy tail (decay). **−** Two mechanisms.

### Option 4 — Just tune the existing decay
Rejected: transient facts are stored high-confidence/`explicit`, so without reclassification they
never reach the floor, and blanket-aggressive decay would also reap durable `inferred` facts.

## 4. Recommendation — phased Option 3, expiry first

**Phase 1 — `expires_at` for explicitly-dated transient facts (high-leverage, low-risk).**

- Extraction, when a fact is **about a specific date and loses value after it** (a forecast; a
  "happening on/at <date>" with no lasting record value), emits `expires_at` = that date + a
  **grace window** (proposed 7 days, configurable). Historical *records* ("received the glasses on
  5/31") are durable → no expiry. Crisp scheduled *events* with a clean triple may instead route to
  a bounded **relation** (`valid_to`) — extraction already supports that path.
- **Soft phase = a retrieval guard, not a write, and not a review queue.** A fact is "hidden" the
  moment `expires_at < now` via an `(expires_at IS NULL OR expires_at >= now)` predicate in
  retrieval — **no row mutation, fully reversible** (raise/clear `expires_at`). Expired facts do
  **NOT** reuse `superseded_by` (a self-FK to a *replacement* fact — can't carry an expiry
  sentinel; see §5). **The soft window's value is a systemic rollout safety margin, NOT per-fact
  human review** (you cannot browse thousands of soft-hidden facts and shouldn't try): if the
  classifier proves too aggressive, you catch it via the recall check / "where did my facts go",
  flip `expire_enabled=off`, and everything still in-window returns instantly. **Do not build a
  soft-pruned review surface** — it's the wrong tool at scale. The soft window is *free* (it's just
  the gap between the retrieval guard and the delete), so the cost of keeping it is ~nil.
- **Hard phase** = a dedicated nightly `prune_expired` DELETE (`expires_at IS NOT NULL AND
  expires_at < now - prune_expired_days`), modeled on the `prune_superseded` *pattern* but its
  **own prepared statement on its own predicate**. Set `prune_expired_days=0` to collapse to
  hard-expire-on-reference-date if the buffer is ever judged pointless.

**Phase 2 — durability class for fuzzy transient facts.** Add `ephemeral` as a decay class (fast
multiplier, floor 0) so `prune_stale` reaps undated transient facts. Pursue only if Phase 1 leaves
a visible fuzzy tail.

**Category:** keep as a **vestigial soft label** — do not invest in making it authoritative. This
closes **B3**: document `remember`'s `category=general` default as intended. *Reopen trigger:* only
if a future LLM-facing use of category emerges.

## 5. Schema + code touch points (Phase 1) — corrected per review

- **Schema** (`auth_db_schema.c`): `ALTER TABLE memory_facts ADD COLUMN expires_at INTEGER DEFAULT
  NULL;` — nullable, O(1), version-gated. (`expires_at` already precedented on `sessions`,
  `messaging_link_codes` — consistent.) Partial index `... WHERE expires_at IS NOT NULL` keeps the
  nightly scan and retrieval guard cheap.
- **Retrieval guard — THE load-bearing change (was understated as "no retrieval change").** Every
  query that currently filters `superseded_by IS NULL` must also gate `(expires_at IS NULL OR
  expires_at >= now)`. Known call sites to update: `memory_embeddings.c:345`,
  `memory_db.c:283-285,438,520,539`, `memory_embed_recompute.c:234`, plus the focus-adapter fact
  paths. Enumerate + sweep these together; this is the real Phase-1 retrieval work.
- **Extraction** (`memory_extraction.c`): optional per-fact `expires_at` (or `ttl_days`); the
  prompt teaches "set ONLY for facts that lose value after a date — forecasts, scheduled-not-yet-
  happened; NOT durable records," and **must disambiguate fact-`expires_at` from relation-
  `valid_to`** (the prompt already carries `valid_from`/`valid_to`). Reuse the anchor for
  relative→absolute resolution.
- **Prune** (`memory_db_facts.c`): new `memory_db_fact_prune_expired(user, retention_days, *count)`
  — a leaf-lock single DELETE mirroring the `prune_superseded` shape. **Lock discipline:** holds
  only the auth_db leaf lock for one DELETE, copies nothing out, acquires no embedding-cache or
  stemmer lock inside the critical section (matches the sibling prune paths). **Carries the same
  known FTS5-orphan debt** as `prune_superseded`/`prune_stale` (bulk DELETE doesn't fire
  `fts5_delete_fact_stems_locked`); close all three together when pruning is exercised in
  production.
- **Nightly wiring** (`memory_maintenance.c`): insert the expire hard-prune into
  `memory_run_nightly_decay()`'s per-user loop at a **defined position** — proposed *after*
  `prune_superseded`, *before* summary prune; idempotent under the existing 20h guard (a DELETE of
  already-deleted rows is a no-op). Specify ordering explicitly so it doesn't shift which rows
  `prune_stale` sees.
- **Config** (`[memory]`): `expire_enabled` (default **off** until validated), `expire_grace_days`
  (default **7**), `prune_expired_days` (default **30**; `0` = hard-expire on reference date) —
  named to parallel the existing `prune_superseded_days` / `prune_stale_days` family.

## 6. Migration + expectation calibration

No back-fill for Phase 1: existing rows default `NULL` (durable), nothing breaks. **But the
measured 17.5%/4.8% bloat is in the *legacy* rows, which Phase 1 does not touch** — only new
extractions get `expires_at`. So **Phase 1 is preventive: it stops the bleeding; the existing pool
only shrinks via the deferred one-time pass or as legacy facts age out.**

**Legacy back-fill (decided — run AFTER Phase 1, deferred):** a one-time, **read-only-then-
dev-approved** script (same shape as the C1 diagnostic) flags clearly-dated/weather legacy facts →
dev reviews the list → batch-sets `expires_at`, after which the **normal nightly prune drains
them** (reuses Phase 1 machinery; no separate delete logic). Note: `find_duplicates`/the C4 merge
do **NOT** address this pool — legacy transient facts are *distinct*, not duplicates, so Friday's
dedup tools can't surface them; the bulk identification is far cheaper as a script than as
per-fact LLM `forget` calls. Out of scope for Phase 1; sequence it once expiry semantics are live.

## 7. Risks + mitigations

| Risk | Mitigation |
|---|---|
| Expiring a fact with lingering value (a forecast referenced for planning) | Grace window + **soft (retrieval-guard) before hard-prune**, fully recoverable; ship `expire_enabled=off`, validate on the dev's DB first |
| **Recency-boost interaction** — a transient fact is created exactly when it's about to expire, so `temporal_weight`/`focus_recency_decay_uniform` (`memory_focus_adapters.c:272`) *boosts* it into the top of retrieval during its least-useful pre-grace window | The retrieval guard hides it at `expires_at`, but during grace it's both present AND recency-boosted; the recall check must specifically watch for transient facts crowding the focus block pre-expiry |
| Extractor mis-tags durable history as ephemeral, or confuses fact-`expires_at` with relation-`valid_to` | Conservative prompt (durable unless *clearly* forecast/pending); explicit prompt disambiguation vs `valid_to`; default `expires_at = NULL` |
| **Extraction cost** — a new per-fact expiry decision adds tokens to every session-end extraction and a new confusion axis | Acknowledged standing cost; conservative prompt + the off-by-default gate; revisit if extraction latency/cost moves |
| Recall regression (a benchmarked answer relied on an expired fact) | **B1-style paired recall check** (LongMemEval/LoCoMo, Mem0 protocol) before enabling; expiry must be recall-neutral |
| Nightly scan cost at scale | Partial index on `expires_at IS NOT NULL`; the transient set is a minority |

## 8. Decisions — RESOLVED by the dev 2026-06-04

1. **Expiry-first vs durability-first** → ✅ **expiry-first** (both eventually; order doesn't
   matter much).
2. **Scheduled-events as bounded relations vs expiring facts** → ✅ **the split** (clean triples →
   relation `valid_to`; free-text forecasts/snapshots → fact `expires_at`). Dev: "interesting to
   see the expiring facts in practice" — so Phase 1 should make expired/expiring facts easy to
   eyeball in dev/logs (not a user surface — a debug/admin count + sample is enough).
3. **Soft-then-hard vs hard-on-expiry** → ✅ **soft-then-hard, reframed**: the soft window is a
   *free systemic safety margin*, not a review queue — no review surface is built (§4). Dev's
   concern ("can't catch soft prunes across thousands of facts") is answered by *not needing to*;
   `prune_expired_days=0` collapses it to hard-expire if ever judged pointless.
4. **Windows** → ✅ **grace 7d** (visible in recall 1 week past the reference date), **prune_expired
   30d** (recoverable buffer before hard-delete). Default `expire_enabled=off` until the recall
   check passes.
5. **Category** → ✅ **vestigial** ("if the label no longer has value or is used"). Closes **B3** —
   document `remember`'s `category=general` default as intended; reopen only if a future LLM-facing
   use of category emerges.
6. **Legacy back-fill** → ✅ **deferred one-time dev-reviewed script, post-Phase-1** (§6). NOT
   Friday's dedup tools (legacy transient facts aren't duplicates) and NOT per-fact LLM forget
   (too slow/expensive); a script does the bulk flagging, dev approves, nightly prune drains.

## 9. Effort (indicative, agent terms)

- Phase 1 (schema + retrieval-guard sweep + extraction field + `prune_expired` + nightly wiring +
  config + B1 recall check): agent ~1.5–2d · api ~$1–2 (recall check) · ~4 ckpt (schema, retrieval
  sweep, extraction, prune+nightly). *(Bumped from the first draft after review surfaced the
  retrieval-guard call-site sweep as real, not "no change.")*
- Phase 2 (durability class): agent ~half-day · only if a fuzzy tail remains.

---

## Appendix — architecture review disposition (2026-06-04)

Reviewed before finalizing. Findings and how they were folded in:
- **Critical — soft-hide can't reuse `superseded_by`** (self-FK to a replacement, no sentinel):
  *fixed* — soft phase is now a non-writing `expires_at` retrieval guard; hard phase is a dedicated
  `prune_expired` statement; the retrieval call-site sweep is now stated as the load-bearing work
  (§4, §5).
- **High — recency-boost interaction / fact-`expires_at` vs relation-`valid_to` / nightly ordering:**
  *folded in* (§3 fork, §5 ordering, §7 risk rows).
- **Medium/Low — FTS5-orphan debt, leaf-lock statement, extraction cost, preventive-not-retroactive
  calibration, config naming, B3 reopen-trigger:** *folded in* (§5, §6, §7, §4).

Verdict after fixes: the recommendation (phased Option 3, expiry-first, soft-then-hard) survived
pressure-testing; the proposal is sound to hand to the dev for direction.
