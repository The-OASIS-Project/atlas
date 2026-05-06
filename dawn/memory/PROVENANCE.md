# Provenance — Phase A Validation Result + Phase B Plan

**Status:** Phase A SHIPPED 2026-05-06.  Bench-validated overall `recall_generation` lift +12.51pp at budget=12288 (0.368 → 0.493) on LoCoMo.  Phase B (coverage extension) GREEN-LIT, deferred to next implementation cycle.  This doc is the as-shipped record for Phase A and the spec for Phase B.

**Date:** 2026-05-06.

## Phase A — Final Results

Measured on LoCoMo (10 conversations, 1982 questions), Haiku extraction + judge + generator + correctness, all 5 budget points run within the same session for clean within-run comparison:

| Budget | Overall gen | Δ baseline | cat-1 | cat-2 | cat-3 | cat-4 | cat-5 |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 (baseline) | 0.368 | — | 0.319 | 0.486 | 0.283 | 0.484 | 0.114 |
| 1024 | 0.404 | +3.53pp | 0.362 | 0.508 | 0.283 | 0.536 | 0.130 |
| 3072 (current default) | 0.445 | +7.67pp | 0.348 | 0.498 | 0.283 | 0.636 | 0.141 |
| 6144 | 0.464 | +9.54pp | 0.365 | 0.505 | 0.283 | 0.675 | 0.135 |
| **12288 (bench winner)** | **0.493** | **+12.51pp** | **0.404** | **0.511** | **0.348** | **0.713** | **0.152** |

**Reach + entailment unchanged at 0.834 / 0.420 across all 5 budgets** — confirms the lift is purely generator-side from verbatim context (same retrieval, same fact pool).

### Marginal-lift trajectory (NOT monotonically diminishing)

| Step | Δ |
|---|---|
| 0 → 1024 | +3.53pp |
| 1024 → 3072 | +4.14pp |
| 3072 → 6144 | +1.87pp |
| **6144 → 12288** | **+2.98pp** ← re-accelerated |

The non-diminishing 6144→12288 step suggests further headroom past 12288 is plausible.  Stop condition was "<0.5pp between adjacent values"; 12288 hasn't met it yet.  A single Pass 4 at 24576 would settle the operational sweet spot — listed as a follow-up in `STATE.md` short-term workstream.

### Surprise findings

1. **Cat-3 (multi-hop inference) is NOT provenance-immune** as previously concluded.  It stayed flat at 0.283 from budget=0 through 6144, then jumped to 0.348 (+6.5pp) at 12288.  Multi-hop questions are budget-bound: below ~12K chars of source per memory call, no benefit; above it, real lift.  This overturns the earlier "needs graph work, not retrieval improvements" framing from `LOCOMO_CAT3_PROFILING.md` for the answer-generation half of the cat-3 problem.  Retrieval-side `recall_reach` is still 0.689 (the original cat-3 ceiling); generation is what jumped.

2. **Cat-4 (knowledge updates) is the dominant beneficiary** — +22.9pp at 12288 (0.484 → 0.713).  Knowledge-update questions ("when did X change to Y") need verbatim conversation context to disambiguate temporal updates that get lost in summarized facts.

3. **Old fixed 8KB result buffer was silently truncating** — discovered mid-sweep.  Pass 1B at budget=6144 underperformed 3072 on conv 1 (0.508 vs 0.553) — the truncation signature.  Replaced with growable strbuf utility; the same conv at 12288 now lands 0.574.  Five additional truncation hazards in the same shape were folded into the same commit (see "Truncation cleanup" below).

## Phase A — What Was Shipped

| Component | File(s) |
|---|---|
| Growable string-builder utility | `include/core/strbuf.h`, `src/core/strbuf.c` |
| 18-test Unity coverage | `tests/test_strbuf.c` |
| memory_callback refactored to strbuf, OOM-bail, source_budget param, 32 KiB clamp | `src/memory/memory_callback.c` (+ `include/memory/memory_callback_internal.h`) |
| memory_context.c API change (caller-buffer → malloc'd return) + MIN_TOKEN_BUDGET floor (defense-in-depth on prompt-injection framing) | `src/memory/memory_context.c` (+ `include/memory/memory_context.h`) |
| `[memory] source_budget_chars` exposed in dawn.toml (default 3072 unchanged; deployments can tune to 12288 for the +12.51pp lift) | `include/config/dawn_config.h`, `src/config/config_defaults.c`, `_parser.c`, `_env.c` |
| Bench `query_memory_callback` handler + `#error DAWN_DAEMON_BUILD` guard + BENCH_MP_USER_ID contract block | `benchmarks/bench_memory_pipeline.c` |
| `--with-source` / `--source-budget` CLI flags + production-faithful generator path | `benchmarks/run_benchmark.py` |

### Truncation cleanup (folded into same commit per "fold in fixes when superseded" policy)

A code scout surfaced the same `(buf, buf_size, offset += snprintf)` truncation hazard in 5 other LLM/WebUI response paths.  All converted to strbuf with explicit fallback error strings on OOM:

| File | Path | Why this severity |
|---|---|---|
| `src/memory/memory_context.c` | per-session LLM system prompt assembly | HIGH — bites every session, not every tool call |
| `src/webui/webui_memory.c` | user export endpoint (Markdown text) | MEDIUM |
| `src/tools/scheduler_tool.c` | active events list to LLM | MEDIUM |
| `src/tools/calendar_tool.c` | calendars/today/range/next/search (5 handlers + 3 helpers) | MEDIUM |
| `src/webui/webui_music_handlers.c` | queue list to WebUI | MEDIUM |

### Three-agent post-implementation review (architecture / security / coding-standards)

0 Critical / 4 High / 10 Medium / 9 Low findings.  All HIGHs and budget-cluster MEDIUMs folded into the same commit:
- HIGH: stale "1 MiB" comments updated to "256 KiB" (3 sites)
- HIGH: `RESULT_BUF_SIZE` in scheduler_tool.c gained clarifying comment that the unbounded list path is now strbuf
- HIGH: `MEMORY_SOURCE_BUDGET_CHARS` invariant comment rewritten with bench data + dawn.toml pointer
- MEDIUM: `source_budget` clamped at 32 KiB inside `memory_action_search`/`_recent` (defense-in-depth)
- MEDIUM: `MIN_TOKEN_BUDGET = 250` floor in `memory_context.c` so prompt-injection-defense framing always fits
- MEDIUM: `memory_action_recent` summaries loop now bails on `strbuf_oom` (mirrors search pattern)
- MEDIUM: `[memory] source_budget_chars` exposed as config (deployable bench winner)

Remaining MEDIUMs and LOWs (cosmetic/consistency items: error-string voice, `_sb` suffix convention, missing test for memory_context elision, etc.) tracked in TODO.md for a follow-up pass.

## Context (historical)

Memory provenance shipped in commit `86e47d6` (May 1, 2026, schema v40). It added back-links from extracted memories to the source `(conversation_id, msg_id_start, msg_id_end)` and a `with_source` parameter on the `recall`/`search`/`recent` LLM tool that returns verbatim source excerpts alongside the extracted fact text.

What was *not* shipped:
1. **Validation that `with_source=true` actually lifts benchmark scores** — particularly cat-3 (multi-hop inference, currently 0.326 `recall_generation`). The TODO entry projected "likely lift on cat-3" but no measurement was ever run.
2. **Source rendering for non-fact records.** Schema columns and writers exist on `memory_relations`, `memory_summaries`, `memory_preferences`, but `memory_callback` only renders source excerpts for facts. Reader APIs for the other three types are missing.
3. **Source dedup within a single response** — when N facts share the same `(conv_id, msg_start, msg_end)` (common since session-window extraction produces dozens of facts per range), `conv_db_get_messages_by_range` is called N times.

This plan validates first, then extends coverage if validation succeeds.

## Phase A — Bench validation (production-faithful path)

**Goal:** Measure whether enabling `with_source=true` in the production memory retrieval path lifts LoCoMo `recall_generation`.

**Approach:** Production-faithful — pull the actual `memory_callback`'s `memory_action_search()` path into the bench, not a re-implementation. The bench's generator must consume what voice DAWN users actually consume.

### A.1 — Bench command for production memory path

New `bench_memory_pipeline.c` command:

```
query_memory_callback {
  query: string,
  with_source: bool,
  source_budget: int (0 = use default 3072, >0 = override per-call)
}
→ {
  status: "ok",
  text: string  // The exact formatted output memory_action_search() produces
}
```

**Implementation: pass budget as a function parameter, not a process-global.** Add a `source_budget` parameter to `memory_action_search()` (and `_recent()`); when 0, use the existing `MEMORY_SOURCE_BUDGET_CHARS = 3072` default. Bench binary is the only caller that passes a non-zero override. This eliminates the global-mutation thread-safety concern (Embedded H1, Arch H1, Sec M2 all fold into this single change) and removes the need for a setter / `_unsafe` naming entirely.

The bench command:
- Lives only in `bench_memory_pipeline` (separate binary, not linked into `dawn` daemon).
- File-level `#error` guard: `#ifdef DAWN_DAEMON_BUILD #error "..." #endif` (Sec M3).
- Calls `memory_action_search(BENCH_MP_USER_ID, query, 0, NULL, 0, false, with_source, source_budget)` directly.

**Pre-flight verification (Arch H2):** before Pass 1, confirm that `memory_action_search` and any function it transitively calls never re-derive user_id via `get_current_user_id()` (which would reach a session context the bench doesn't have). Audit covers: `memory_action_search`, `multi_token_fact_search`, `memory_db_*` callees, `append_graph_context`. Verification step is a grep + dry-run trace; takes ~5 minutes. If a re-derivation path exists, parameterize it before any bench measurement.

**Layer hierarchy note (Arch M1):** bench is harness scope, not part of the layered application; calling Layer 2 (`memory_callback`) directly is the established pattern (see existing `memory_db_facts_get_sources` / `conv_db_force_anchor_date_unsafe` callouts in `bench_memory_pipeline.c`).

### A.2 — Generator template change + cache key bumps

Bench's `GENERATOR_USER_TEMPLATE` (run_benchmark.py:531) currently formats numbered facts. With production-faithful retrieval, the bench gets pre-formatted memory text that already includes facts + source excerpts + entity graph block. New template:

```
Memory tool output:
{memory_text}

Question: {question}
```

The orchestrator uses `query_memory_callback` instead of `query_memory` when `--with-source` is set.

**Bump all three cache keys (Arch M3)**, not just generator: `GENERATOR_PROMPT_VERSION`, `ENTAILMENT_PROMPT_VERSION`, and `CORRECTNESS_PROMPT_VERSION`. The retrieved text changes (now contains pre-formatted memory output instead of fact list), so judge / generator / correctness keys all need to diverge from old cached entries. Alternatively, include `with_source=bool` in each cache key's input — equivalent and cleaner if multiple budget values share the same prompt versions.

### A.3 — Source budget sweep ("intelligent probe")

`MEMORY_SOURCE_BUDGET_CHARS` (currently 3072) controls how many chars of source-excerpt text get included per memory tool response. Higher budget = more verbatim context but more tokens consumed by the generator (which has 200 max output tokens, 8KB tool result buffer, and a finite top-K of facts).

**Char budget heuristic (L1):** budget values are documented in chars, not tokens. ASCII English ≈ 4 chars/token; conversation text with code/JSON can be ~3 chars/token. Source excerpts are LoCoMo conversation prose (mostly English, ~4 chars/token). Plan numbers below assume that ratio.

**Pass 1 — gate (Embedded M, save ~60% if no lift):** 2 budget values
- 0 (`with_source=false` — baseline)
- 3072 (current default, ~768 tokens)

Cost: 2 passes × ~$8 = ~$16, ~4hr wall.

**Decision after Pass 1:**
- 3072 lifts overall `recall_generation` ≥1pp vs 0 → proceed to Pass 2
- 3072 ≤ 0 baseline → stop, declare provenance neutral/harmful in this generator config; ship is feature-complete as-is
- 3072 within ±1pp of 0 → optional Pass 2 with two extra values (1024, 6144) to confirm flat-line before declaring neutral

**Pass 2 — refine (gated):** 2 more budget values, shape depends on Pass 1 trajectory
- If 0 → 3072 lifts: add 1024 (cheaper midpoint) + 6144 (does it keep climbing?)
- If 6144 still lifting: optionally add 9216 in Pass 3

Each additional pass: ~$8, ~2hr.

**Total cost (worst case):** 5 passes (0, 1024, 3072, 6144, 9216) × ~$8 = ~$40 + ~10hr wall. Best case (no lift): 2 passes = ~$16.

**Stop condition:** declare a winner once two adjacent values differ by <0.5pp on overall `recall_generation`.

### A.4 — Verifiable success criteria for Phase A

**All deltas measured against the same-run Pass 1 baseline (`with_source=false`, budget=0)**, not against historical numbers (Arch L3) — controls for any judge-model drift between runs.

| Metric | Threshold | Decision |
|---|---|---|
| Cat-3 `recall_generation` | ≥+3pp vs same-run baseline (current shipped: 0.326 → ≥0.356) | proceed to Phase B |
| Overall `recall_generation` | ≥+2pp vs same-run baseline (current shipped: 0.371 → ≥0.391) | proceed to Phase B |
| Any category regresses | >1pp drop vs same-run baseline at chosen budget | investigate before B; may abort |
| Source budget winner | identified to within ±1024 chars | sufficient resolution |

If none of the lift criteria are met, declare provenance feature-complete as shipped (the LLM-tool-API value is still real for human verifiability use cases, even if it doesn't lift bench numbers). Update SYSTEM_DESIGN.md / STATE.md, skip Phase B.

### A.5 — Cost summary for Phase A

`agent ~1-2hr active · api ~$16 (best case) – ~$40 (worst case) · 1-3 ckpt depending on sweep depth`

## Phase B (gated on Phase A) — Coverage extension + dedup

**Goal:** Source-link relations / summaries / preferences in the same memory tool response. Dedup repeated source fetches.

### B.1 — Batch reader APIs (in new module)

**Module location (Arch M4):** create `src/memory/memory_db_provenance.c` + `include/memory/memory_db_provenance.h`. Move `memory_db_facts_get_sources` and `memory_db_fact_get_source` from `memory_db.c` into the new module along with the three new sibling functions. Pre-empts `memory_db.c` line-count pressure (currently near limit per MEMORY.md) and groups the four `_get_sources` functions in one file.

Three new functions mirroring `memory_db_facts_get_sources`:

```c
int memory_db_relations_get_sources(int user_id,
                                    const int64_t *relation_ids, int n,
                                    int64_t *out_conv_ids,
                                    int64_t *out_starts,
                                    int64_t *out_ends);

int memory_db_summaries_get_sources(int user_id,
                                    const int64_t *summary_ids, int n,
                                    int64_t *out_conv_ids,
                                    int64_t *out_starts,
                                    int64_t *out_ends);

int memory_db_prefs_get_sources(int user_id,
                                const int64_t *pref_ids, int n,
                                int64_t *out_conv_ids,
                                int64_t *out_starts,
                                int64_t *out_ends);
```

Each: single SQL query, IN clause on IDs, JOIN on `conversations.is_private = 0` (privacy invariant identical to fact path). One `AUTH_DB_LOCK` cycle per call.

**Fail-closed on truncation (Sec H2):** the existing `memory_db_facts_get_sources` snprintf-based IN-clause builder silently truncates if N records exceed the 1024-byte sql buffer. The new module **must** either (a) reject `n > MAX_PROVENANCE_BATCH` (computed to fit the sql buffer with safety margin, ~50 IDs at 20 chars each) returning `MEMORY_DB_FAILURE`, or (b) build the SQL on the heap so truncation is impossible. Strong preference for (a) — explicit cap, callers know their own list sizes (top-K is bounded). Add a `static_assert` tying buffer size to `MAX_PROVENANCE_BATCH`. Update existing `_facts_get_sources` to use the same cap (consistency + closes the existing bug).

**Unit tests:** for each of the four record types — round-trip, NULL-prov returns NOT_FOUND, private-conv filter, **N=64 truncation rejection** (Sec H2 explicit test).

### B.2 — Source rendering for all four record types

Refactor `append_source_excerpt` from fact-specific to record-type-generic:

```c
static int append_source_excerpt_from_range(int user_id,
                                            int64_t conv_id, int64_t start, int64_t end,
                                            char *buf, size_t buf_size,
                                            size_t *offset,
                                            int *budget_remaining,
                                            source_dedup_set_t *seen);
```

Callers (one per record type) batch-fetch sources using the new `_get_sources` APIs, then iterate calling the renamed helper. The previous `append_source_excerpt` becomes a thin wrapper that does the single-fact lookup + delegates.

**Defense-in-depth privacy hop (Sec H1):** the range-generic helper trusts upstream callers to have already filtered private conversations. To prevent any future caller from accidentally rendering private content via this path, **add `AND c.is_private = 0` to the SQL in `conv_db_get_messages_by_range`** (currently in `auth_db_conv.c:1710`). This is a one-line change that makes the privacy filter independent of upstream caller correctness. Add a unit test in `test_conv_db` that creates a private conversation, calls `_get_messages_by_range` directly, and asserts no rows returned.

### B.3 — Source dedup

`source_dedup_set_t` is a file-private struct in `memory_callback.c` (Arch M2):

```c
#define SOURCE_DEDUP_CAP 24  /* top-K facts ≤10 + summaries/relations/prefs ~ 14 */

typedef struct {
   int64_t conv_ids[SOURCE_DEDUP_CAP];
   int64_t starts[SOURCE_DEDUP_CAP];
   int64_t ends[SOURCE_DEDUP_CAP];
   int count;
} source_dedup_set_t;

static bool dedup_seen(source_dedup_set_t *set, int64_t conv, int64_t s, int64_t e);
static void dedup_add(source_dedup_set_t *set, int64_t conv, int64_t s, int64_t e);
```

Initialize with `count=0` at the start of each `memory_action_search` / `_recent` call. Before fetching messages for a record's source, check `dedup_seen()`; if hit, skip with a single line like `[source: same as above]` instead of refetching. After successful fetch, `dedup_add()`.

**Dedup must operate post-batch-filter only (Sec M1):** the `(conv_id, start, end)` triples populating the dedup set come exclusively from the new `_get_sources` batch APIs (which return only non-private rows via the JOIN). Single-fact `_get_source` callers must NOT populate the dedup set. This prevents the leak shape where a private fact's range gets stored in the set and a later non-private fact sharing the same range emits `[source: same as above]`, betraying the existence of the private upstream fact.

**Test coverage (Sec M1):** integration test that mixes one private + one public fact sharing identical `(conv_id, range)`, queries with `with_source=true`, asserts no `[source: same as above]` line appears in the output.

### B.4 — Verifiable success criteria for Phase B

| Metric | Target | How measured |
|---|---|---|
| New unit tests on batch APIs | 12 (4 per record type: roundtrip, NULL-prov returns NOT_FOUND, private-conv filter, **N=64 truncation rejection**) | `ctest -L ci -R memory_provenance` |
| Defense-in-depth test | `conv_db_get_messages_by_range` against private conv returns 0 rows | New test in `test_conv_db` |
| Dedup integration test (Arch L2) | Construct a query returning 5 facts from same `(conv_id, range)`, instrument call count via a counter exposed in `source_dedup_set_t.count`, assert exactly 1 distinct source-fetch | New `tests/test_memory_source_dedup.c` |
| Privacy dedup integration test (Sec M1) | Mix 1 private + 1 public fact sharing identical `(conv_id, range)`, assert no `[source: same as above]` in tool output | Same test file |
| Coverage | All 4 record types render source excerpts when `with_source=true` | Manual QA + integration tests above |
| Re-run Phase A bench at chosen budget | Overall `recall_generation` should not regress vs the Phase A winner; may improve marginally | Bench rerun (`agent ~30m · api ~$8 · 1 ckpt`) |

### B.5 — Cost summary for Phase B

`agent ~3-4hr · api ~$10 · 2 ckpt`

## Combined verifiable goals

| Outcome | How verified |
|---|---|
| Production-faithful bench validates provenance | Phase A measurement; Pass criteria in §A.4 |
| Optimal source budget identified | Sweep result + sweep-stop condition met |
| All four record types source-linked when requested | Phase B integration test + ctest |
| No unintentional source-fetch duplication | Phase B dedup test |
| No regression on cat-1 / cat-2 / cat-4 / cat-5 | Phase A measurement (each category checked) |
| No regression on existing 41 CI tests | `ctest -L ci` |
| Format clean | `./format_code.sh --check --changed` |

## Risk inventory (post-review)

| Risk | Likelihood | Mitigation |
|---|---|---|
| Source excerpts crowd top-K for cat-1 (the same problem that bit Phase 2 Option B) | Medium | Sweep includes 0 budget (baseline) — if cat-1 regresses across all non-zero budgets, declare neutral and ship what we have without changing the production default |
| Privacy filter accidentally bypassed in new batch APIs | Low (caught by tests) | Mirror the JOIN pattern from `_facts_get_sources` exactly; defense-in-depth via `conv_db_get_messages_by_range` privacy filter (Sec H1); explicit private-conv unit test per record type |
| Source dedup leaks private-source existence via `[source: same as above]` (Sec M1) | Resolved | Dedup operates only on post-batch-filter rows; explicit integration test with mixed private+public same-range facts |
| Batch IN-clause SQL truncation silently drops rows (Sec H2) | Resolved | Cap `n` at `MAX_PROVENANCE_BATCH` with explicit `MEMORY_DB_FAILURE` return; `static_assert` ties cap to buffer size; N=64 unit test |
| Source budget sweep wall-clock cost runs out of API quota mid-sweep | Low | Each pass independent + cached; can resume next session. 2-pass gate front-loads abort |
| Bench exposes admin user (BENCH_MP_USER_ID=1) data via `query_memory_callback` if linked into daemon (Sec M3) | Resolved | `#error` guard on `DAWN_DAEMON_BUILD`; bench is separate executable; pre-flight verification step |
| Pre-existing TOCTOU between owner check and privacy check on `_fact_get_source` (Sec L1) | Pre-existing, accepted | Preserve the existing comment; do not collapse into single SQL JOIN that masks the limitation |
| Source excerpts may surface pre-injection-filter (pre-April-2026) prompt injection payloads from old conversations (Sec L2) | Out of scope; tracked separately | Existing TODO item "Memory injection filter: pre-filter legacy data" addresses |
| Cache key collisions across sweep passes | Resolved | Bump all three cache prompt versions (Arch M3); cache key inputs include `with_source` flag |

## Rollback plan

- Phase A: no code change ships unless Phase B happens. Prompt revert is a no-op.
- Phase B: the new batch APIs and source-rendering changes are additive. If a regression is caught post-merge, revert is a single git-revert; schema unchanged.

## Review summary

Plan was reviewed in parallel by `architecture-reviewer`, `security-auditor`, `embedded-efficiency-reviewer` on 2026-05-05.

**Strongest fold-in: pass `source_budget` as a parameter on `memory_action_search` instead of mutating a global.** Single change closes Embedded H1 (thread-safety), Architecture H1 (setter naming + multi-threaded reader), and Security M2 (compile-fence) simultaneously. No setter, no `_unsafe` naming needed.

| Finding | Severity | Status |
|---|---|---|
| Setter thread-safety / global mutation | Embedded H1 / Arch H1 / Sec M2 | ✅ Folded — replaced with parameter |
| Range-generic helper privacy hop | Sec H1 | ✅ Folded — `is_private` filter added to `conv_db_get_messages_by_range` |
| Batch IN-clause silent truncation | Sec H2 | ✅ Folded — fail-closed cap + `static_assert` + N=64 test |
| `memory_action_search` re-deriving user_id | Arch H2 | ✅ Folded — explicit pre-flight verification step |
| Source dedup leaks private-source existence | Sec M1 | ✅ Folded — dedup post-batch-filter only + explicit test |
| Bench command linked into daemon | Sec M3 | ✅ Folded — `#error` guard + separate-binary verification |
| New module `memory_db_provenance.c` | Arch M4 | ✅ Folded |
| Bump all three cache prompt versions | Arch M3 | ✅ Folded |
| `source_dedup_set_t` as proper struct + helpers, cap 24 | Arch M2 | ✅ Folded |
| 2-pass sweep gate to save API spend | Embedded M | ✅ Folded — Pass 1 = {0, 3072} only |
| Layer hierarchy doc | Arch M1 | ✅ Folded |
| Char vs token heuristic | Arch L1 | ✅ Folded — explicit ratio note |
| Dedup count assertion mechanism | Arch L2 | ✅ Folded — counter exposed in `source_dedup_set_t.count` |
| Within-run baseline for "no regression" | Arch L3 | ✅ Folded — same-run baseline, not historical |
| TOCTOU comment preservation | Sec L1 | ✅ Folded |
| Pre-injection-filter excerpt risk | Sec L2 | Out of scope; tracked under existing TODO |

## Next steps

1. ✅ Plan reviewed by three agents and revised.
2. Implement Phase A (bench wiring + 2-pass sweep gate). Pre-flight: verify `memory_action_search` does not re-derive user_id mid-call.
3. Run Pass 1 (budget=0, budget=3072). Decide on Pass 2 per the gate criteria.
4. If Phase A meets success criteria, proceed to Phase B implementation.
5. Move this doc to `atlas/dawn/memory/PROVENANCE_VALIDATION.md` once Phase A measurement lands and Phase B decision is settled.
