# Phase 2 — Entity-Graph Retrieval: Query-Scored Graph Candidates + 2-hop Traversal

**Status**: **Step 1 SHIPPED 2026-05-14** as part of a broader retrieval consolidation. Step 2 (2-hop) — scaffolding stripped after architectural review; design preserved in §Step 2 below for re-introduction if a multi-hop benchmark surfaces genuine test cases.
**Pre-conditions**: Phase 0 + Phase 1A shipped (commits `0cdca08` + `4f8f122`).
**Last updated**: 2026-05-14

## What shipped (TL;DR)

1. **Unified retrieval primitive** `memory_search_execute()` in `memory_fact_search.{c,h}`. Single source of truth for the production pipeline (hybrid keyword + cosine → entity-graph candidate expansion → Step 1 query-scoring → score-based merge → search-score floor). Both `memory_action_search` and the bench's `handle_query_memory` now call it. Closes a ~270-LOC parallel implementation that had been drifting between bench and production.
2. **Phase 2 Step 1 — query-aware scoring of graph candidates**. New primitive `memory_embeddings_rescore_against_query()` with sentinel contract for un-scoreable facts. Replaces the legacy flat `entity_grounding_bonus` (still available via `use_query_scoring=false` for ablation). Bench-measured **marginal lift ~0pp on the corrected bench** — the prior "+1.1pp" was a measurement artifact of the bench/production drift.
3. **Score-based competitive merge** replaces "append if room" in `memory_action_search`. Allows high-scoring graph candidates to displace weak hybrid tail entries (the old logic locked graph out whenever hybrid saturated 10 slots).
4. **Bench parity fix** — bench was missing the production `search_score_floor` and using cosine-only retrieval (no keyword pre-filter). Both fixed via the unified primitive. **This surfaces ~4.4pp of production-vs-bench measurement gap** (LoCoMo overall: bench 0.8696 → 0.9132 reflecting what production actually does, not a code lift).
5. **Step 2 scaffolding stripped**. `memory_graph_expand_n_hop`, `max_hops`/`hop_2_decay` config, `--max-hops` bench CLI — all removed (~270 LOC). Diagnostic script + bench command preserved.
6. **Five reviewer findings applied**: sentinel hardening (security M-2), pre-computed embedding through `_with_emb` variant (embedded HIGH-1, eliminates duplicate ONNX inference per call), `MEMORY_GRAPH_INTERMEDIATE_CAP` promoted to header (arch M-1), `entity_bonus` knob defaults to 0.0 with deprecation note (audit CRITICAL-1), six new unit tests pinning the rescore contract.

### Measurement (against cached LoCoMo corpus, deterministic)

| Config | Overall | cat-3 |
|---|---|---|
| Pre-Phase-0 baseline (May 11) | 0.7392 | 0.643 |
| Phase 0 + 1A shipped baseline | 0.8696 (this run's resample) | 0.740 |
| **This commit, Step 1 OFF** (legacy flat scoring) | **0.9132** | 0.795 |
| **This commit, Step 1 ON** (default, query-scored) | **0.9137** | 0.795 |

The ~+4.4pp overall and ~+5.5pp cat-3 from "shipped baseline" to "this commit" is the **bench finally measuring production**, not a retrieval lift. Step 1's marginal contribution on the corrected bench is ~0pp. Step 1 still ships because the structural improvements (sentinel contract, score-based merge, single retrieval path) are real wins independent of the measured lift.

## Scope revision history

> **Scope revision note (May 14, 2026)**: this design was originally scoped around 2-hop traversal as the primary lever (based on the Phase 2A reachability diagnostic showing 43.8% additional cat-3 reach at 2 hops). During Phase 2C implementation we discovered that the 2-hop ceiling lift does NOT pass through the production merge layer because the graph layer uses a **flat entity-grounding score** disconnected from query relevance. The 43.8% diagnostic was measured at cap=500; production caps at 10 with hybrid hits taking most slots by score. See §Phase 2B Findings for the full investigation. The revised scope leads with **query-aware scoring of graph candidates** (Step 1).
>
> **Further finding from cat-3 miss inspection**: the 21 cat-3 misses marked "reached only at 2-hop" turned out to all be cases where the gold dialog's *own speaker* matches the question's seed entity (e.g., gold is John's own dialog answering a John question). These should be 1-hop reachable; they appear at 2-hop only because extraction linked the relevant fact to a peripheral entity rather than the speaker. **LoCoMo cat-3 does not actually contain genuine multi-hop graph-traversal test cases** in the architectural sense — the category name "inference" doesn't imply graph distance. Step 2 production scaffolding was stripped after this finding; the design below is preserved for re-introduction if/when a benchmark with true multi-hop questions (HotpotQA, 2WikiMultiHop, MuSiQue, or a synthetic test set) is added to the bench harness.
>
> **Bench-vs-production drift finding (May 14, 2026)**: while applying audit recommendations, the unified-primitive extraction revealed the bench had been measuring a different retrieval pipeline than production — cosine-only (no keyword pre-filter), no `search_score_floor`. The previously reported "Phase 0+1A delivered +13.4pp" was measured on the weaker bench pipeline; production has been at ~0.91 overall the whole time. This is meaningful context for any future cross-benchmark comparison.

## Baseline

| Metric | Pre-Phase-0 (May 11) | Phase 0+1A (May 13) | Δ |
|---|---|---|---|
| LoCoMo overall recall_reach | 0.7392 | **0.8733** | +13.4pp |
| cat-1 | 0.598 | 0.788 | +19.0pp |
| cat-2 | 0.771 | 0.888 | +11.7pp |
| **cat-3 (inference, primary target)** | 0.643 | **0.806** | +16.3pp |
| cat-4 | 0.785 | 0.931 | +14.6pp |
| cat-5 | 0.740 | 0.823 | +8.3pp |

**Competitive position**: cat-3 at 80.6% sits 4.5pp below ByteRover (85.1%) and 11.4pp below MemMachine (~92%). Phase 2 targets the remaining gap.

## Goals

Close the remaining cat-3 gap using the substrate Phase 0 + 1A established:
- 100% fact↔relation pairing on new extractions
- `memory_facts.subject_entity_id` FK populated (~98-100% on fresh corpus)
- `memory_relations.valid_from` / `valid_to` populated by Phase 0 extraction prompt
- `memory_graph_retrieval` module wired into `memory_action_search` with 1-hop fan-out from query-resolved seed entities

Two candidate improvements, both made tractable by the above:

1. **2-hop entity-graph traversal**: extend `memory_graph_expand_fact_linked` from 1 hop to 2, with hop-2 decay, edge filtering, cycle prevention. Handles inference queries that require traversal through an intermediary entity ("what does Alice's spouse do for work?").

2. **Bitemporal filter at query time**: detect query intent (present-tense → filter `valid_to IS NULL`; historical → `as_of_ts`; all-time → no filter) and apply at retrieval. Tool already has `as_of` / `include_historical` params — this is auto-detection plus default-filter.

Whether both ship together or separately depends on Phase 2A's evidence.

---

## Phase 2A — Pre-flight diagnostic

**Goal**: let evidence pick scope. We don't know whether the remaining cat-3 gap lives in unreachable evidence, mis-ranked reachable evidence, or temporal-validity confusion. Diagnosing first prevents us from building 2-hop traversal if the real bottleneck is somewhere else.

**Why diagnostic-first**: same rigor as Phase 0. Last session's diagnostic flipped the direction from "implement Phase 1A workaround" to "fix extraction at the root." Phase 2 deserves the same evidence-first approach.

### Step 2A.1 — Generate fresh dense-graph snapshot corpus

Phase 0 bench used `--no-cache`; snapshots are gone. Run again with caching enabled.

```bash
cd /home/jetson/code/The-OASIS-Project/dawn

python3 benchmarks/run_benchmark.py \
    --binary ./build-debug/tests/bench_retrieval \
    --benchmark locomo \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --memory-pipeline \
    --cache-dir ./benchmarks/snapshots_phase2_base \
    --output ./benchmarks/bench_phase2_baseline.json
```

Expected: reproduces the May 13 numbers ± noise (~$2.50, ~60 min). Persisted snapshots feed every subsequent diagnostic.

### Step 2A.2 — Capture cat-3 misses on dense graph

```bash
python3 benchmarks/cat3_misses_memory.py \
    --binary ./build-debug/tests/bench_retrieval \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --cache-dir ./benchmarks/snapshots_phase2_base \
    --output /tmp/cat3_misses_phase2_base.json
```

This re-uses the cached corpus (no LLM cost). Outputs per-query miss records — which gold dialogs were not retrieved in top-10 for each missed cat-3 question.

### Step 2A.3 — Reachability profile on dense graph

```bash
python3 benchmarks/cat3_graph_reachability.py \
    --binary ./build-debug/tests/bench_retrieval \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --misses-input /tmp/cat3_misses_phase2_base.json \
    --cache-dir ./benchmarks/snapshots_phase2_base \
    --output /tmp/cat3_reachability_phase2_base.json
```

Classifies each missed evidence dialog by graph distance from the question's seed entities. Pre-Phase-0 measurement (sparse graph): 4% reachable at rank-51+ via 1-hop. With Phase 0's dense graph, expect substantially higher.

### Outputs to inspect

Per missed evidence dialog, classify:

| Class | What it means | Phase 2 path |
|---|---|---|
| **0-hop reachable** | Evidence is at the seed entity itself, but retrieval failed | Ranking issue, not graph issue |
| **1-hop reachable** | Evidence is one relation away from a seed, but Phase 1A didn't surface it | Phase 1A scoring tune, not 2-hop |
| **2-hop reachable** | Evidence is two relations away | **2-hop traversal helps** |
| **3+ hop reachable** | Multi-step inference chain | Beyond Phase 2 scope |
| **Unreachable** | No graph path from any seed | Entity-resolution or extraction failure — separate bug |
| **Temporally filtered** | Reachable but timed-out by an unstated time window | **Bitemporal filter helps** |
| **Entity-resolution failure** | Seed extraction missed the actual referent | Entity-merge / seed-extraction issue |

### Step 2A.4 — Latency baseline

Capture per-query timing on the baseline bench run (already emitted in bench stdout). Phase 2's 2-hop traversal adds cost; we need to know what the budget looks like before tuning.

### Decision matrix (2A → 2B exit)

| Reachability profile | Recommended Phase 2 scope |
|---|---|
| ≥30% of missed evidence is 2-hop reachable AND ≤10% temporally filtered | **2-hop only** (defer bitemporal) |
| ≥15% temporally filtered AND 2-hop reachability <20% | **Bitemporal only** (defer 2-hop) |
| Both signals strong (≥20% each) | **Both, sequenced**: bitemporal first (lower-risk, lower-effort), then 2-hop |
| Neither signal strong | **Pivot** — likely entity-resolution or ranking. Cancel Phase 2 as currently scoped; investigate dominant miss class. |

### Estimated cost

- Snapshot corpus: ~$2.50 Haiku, ~60 min wall
- Diagnostic scripts: $0, ~5-10 min
- **Total Phase 2A: ~$2.50, ~70 min**

---

## Phase 2B — Scope lock (May 14, 2026)

### Original decision (superseded — see §Findings below)

The initial scope-lock based on Phase 2A's 43.8% additional reachability at 2-hops led us to commit to "ship 2-hop entity-graph traversal, defer bitemporal." The implementation (Phase 2C) revealed why this was premature.

### Decision (revised): two steps — query-scored graph candidates first, 2-hop deferred

### Evidence (from 2A.3 + the 2-hop reachability extension)

Pre-flight bench reproduced May 13 within run-to-run variance (overall 0.870 vs 0.873; cat-3 0.740 vs 0.806 — see Variance note below). 48 cat-3 evidence pieces were missed; reachability profile:

| Class | Count | % | Phase 2 implication |
|---|---|---|---|
| Reached at 1-hop (Phase 1A already has it) | 21 | 43.8% | Ranking issue, not reachability — reranker territory |
| **Reached ONLY at 2-hop** | **21** | **43.8%** | **Phase 2 ceiling lift** — gold is one hop further than Phase 1A walks |
| Unreachable even at 2-hop | 6 | 12.5% | Phase 3+ territory; out of scope |

Per rank-bucket structure:

| Bucket | 1-hop reach | +2-hop adds | Unreachable at 2-hop |
|---|---|---|---|
| Rank 11-50 (n=40) | 50% | 37.5% | 12.5% |
| **Rank 51+ (n=8)** | 12.5% | **75%** | 12.5% |

Deep-ranked misses (rank 51+) are dominated by 2-hop need — exactly the failure mode 2-hop traversal is designed for. The 43.8% additional reach at the 2-hop layer is comfortably above the design doc's ≥30% threshold for "2-hop only" scope. Decision matrix triggers cleanly on this row.

### Findings from 2C investigation (May 14, 2026) — why the original scope was wrong

Phase 2C implementation completed the 2-hop machinery (config plumbing, `memory_graph_expand_n_hop`, bench CLI flag, call-site wiring). Smoke ablation against the same cached corpus produced **byte-identical results** for Phase 2 ON (max_hops=2) and OFF (max_hops=1):

| Config | Overall recall_reach | cat-3 recall_reach |
|---|---|---|
| max_hops=1 (Phase 1A behavior, Phase 2 OFF) | 0.8696 | 0.740 |
| max_hops=2 (Phase 2 ON, original scope) | 0.8696 | 0.740 |

Root cause: **the 2-hop reachability ceiling does not pass through the production merge layer.** Three reinforcing reasons:

1. **The diagnostic ran at cap=500. Production runs at cap=10.** Phase 2A's 43.8% additional reach was measured with a deep candidate pool (BENCH_MP_FACT_LIST_CAP). `memory_action_search` truncates to a working array of `facts[10]` after merging hybrid hits with graph candidates.

2. **Graph candidates score flat at `entity_grounding_bonus` (0.4) regardless of query content.** Looking at `memory_graph_extract_seed_entities`, the seed extractor finds capitalized proper-noun tokens only — non-proper-noun query content ("patriotic", "open to moving", "what job") is completely invisible. The graph layer then returns *every fact connected to the resolved entities* with a flat 0.4 score. For a high-degree seed like John (64 1-hop facts), 60 of those are topically irrelevant to the question.

3. **Hybrid hits saturate the 10-slot pool by score.** Hybrid cosine + keyword scores typically land in 0.5–0.7. Graph candidates score 0.4 (hop-1) or 0.24 (hop-2 after decay). Sort-by-score + truncate-to-10 keeps mostly hybrid hits; graph candidates fight for the bottom slots that get cut. Hop-2 facts at 0.24 lose to hop-1 at 0.4 lose to hybrid at 0.5+.

The 43.8% "ceiling lift" the diagnostic measured is real but unusable: it represents facts the graph CAN reach if the candidate pool is wide enough, NOT facts that actually surface to the LLM under current production constraints. Treating it as a guaranteed lift was a misread of what the diagnostic measured.

### Why the user's intuition (query-aware scoring) is the right fix

Phase 2A asked "how far does the graph reach?" — the answer is "far enough." The unanswered question is "how relevant is what the graph reaches?" That signal is currently MISSING from graph search.

The graph layer today: **"Give me everything about John"** — equally weighted, no quality filter, no semantic relationship to the actual question.

The graph layer should be: **"Bound the candidate set to facts touching John (cheap entity-graph pre-filter), then rank those candidates by how well they answer the question (same hybrid scoring hybrid retrieval already uses)."**

Three reasons this is the right fix, not a workaround:

1. **It uses signal we already have.** The query already gets embedded for hybrid cosine; we already do keyword overlap scoring. Applying the same scoring to graph candidates is structurally identical to what hybrid retrieval does, just bounded to a smaller candidate set.

2. **It fixes Phase 1A as well as Phase 2.** Today's Phase 1A is hampered by the same flat-score issue. Query-scoring graph candidates lifts both 1-hop and 2-hop walks.

3. **It makes 2-hop usable, not noisy.** A 2-hop walk widens the candidate pool. Without query-relevance scoring, wider = noisier. With it, wider = more rescue opportunities for query-relevant facts that hybrid alone missed.

This is also exactly what the user's principle of "principled path over easy path" (`feedback_no_easy_path.md`) demands: not "bump the cap to push more candidates through," but "fix why graph candidates are low-signal in the first place."

### Revised scope: two steps, sequenced by measurement

| Step | What | Status | Gate |
|---|---|---|---|
| **2C-Step 1** | Query-aware scoring of graph candidates (cosine + keyword applied to entity-bounded fact pool, with small entity bonus as tiebreaker) | **Primary work** | — |
| **2C-Step 2** | 2-hop traversal default-on (expand candidate pool to include 1-hop entity neighbors) | **Deferred** | Step 1 ships and Step 1 ablation shows it doesn't already capture most of the available lift |

The 2-hop infrastructure from this session's 2C work (config knobs, `memory_graph_expand_n_hop`, bench `--max-hops` flag, diagnostic script) **remains in tree** as foundation. Default `max_hops` reverts to **1** until Step 1 lands and Step 2 measurement justifies bumping.

### Why bitemporal is deferred

The 2-hop reachability diagnostic does NOT distinguish "temporally filtered" misses as a separate class. To know whether bitemporal filtering helps, we'd need a separate diagnostic that classifies each miss by whether a `valid_to` filter would have changed its rank. **No data on this signal yet.** Adding bitemporal blind would risk the false-negative case (filter excludes a fact the user still wants) without measured benefit.

Plan: defer to **Phase 2.5** once Phase 2 ships. Same diagnostic-first discipline.

### Why reranker is deferred

Reranker only addresses the 43.8% "reached at 1-hop, ranked too low" bucket. **2-hop addresses a different bucket** (43.8% "not reached at all by Phase 1A"). Both are real, but 2-hop has identical upside AND opens new territory (rank-51+ misses where 75% become reachable). Reranker is complementary follow-up after Phase 2 ships.

### Why not 3+ hop

Only 12.5% of misses (6 pieces) would remain unreachable after 2-hop. Implementing 3-hop traversal for that 12.5% has poor ROI given:
- Cost scales exponentially with hop depth (entity neighbors grow ~10× per hop on hub entities)
- Hub-entity blowup risk multiplies
- The 12.5% may not all be 3-hop reachable either (some may be unreachable at any depth)

Out of scope. Re-visit only if 2D ships strong and the remaining gap to MemMachine (~7-11pp) needs more.

### Locked Phase 2 surface (revised)

**Step 1 — Query-aware graph scoring** (primary work):
- **Refactor `memory_action_search`** to score graph candidates against the query using hybrid logic (cosine + keyword overlap), with a small `graph_entity_bonus` as tiebreaker (distinguishes graph hits from pure-cosine hits without overwhelming hybrid signal)
- Either: extract a shared `memory_score_facts_against_query()` helper that both `memory_fact_search_hybrid` and the graph layer call, OR call hybrid scoring on the merged candidate pool (hybrid_top_K_ids ∪ graph_candidate_ids) and let one ranker handle everything
- **`memory_graph_expand_*`** evolves to return entity-bounded candidate fact IDs WITHOUT scoring (or scoring becomes a small bonus only). Scoring responsibility shifts entirely to the unified scoring path
- **Config `[memory.graph_retrieval]` gains**:
  - `entity_bonus` (float, default 0.10, range [0, 1]) — small additive bonus applied to graph candidates on top of their hybrid score. Distinguishes graph-rescued candidates from pure-cosine hits but doesn't dominate
- **`entity_grounding_bonus`** (existing, 0.40 default) — retained as a fallback for legacy 1-hop-only call sites if any persist; can be deprecated post-Step-1

**Step 2 — 2-hop traversal** (deferred; foundation already in tree):
- **`memory_graph_expand_n_hop`** — already implemented this session, gated behind config
- **Config `[memory.graph_retrieval]` keeps**:
  - `max_hops` (int, **default reverted to 1** until Step 1 ships; will revisit defaulting to 2 after Step 2 measurement)
  - `hop_2_decay` (float, default 0.6) — kept for when Step 2 activates
- **Bench CLI `--max-hops`** flag retained for Step 2 ablation when ready

**Cross-cutting**: bench CLI gains a Step 1 ablation knob (e.g., `--graph-query-scoring on|off`) so the same binary supports config-toggle ablation for Step 1 in addition to Step 2's `--max-hops`.

### What 2A did NOT answer (deferred follow-up diagnostics)

1. **Question 2 from the prior session** ("would gold-seed extraction help the 6 unreachable misses?") is left open. The 6-piece bucket is too small to drive Phase 2 scope; that question feeds **Phase 1A seed-extraction tuning** as a separate workstream if needed.
2. **2-hop predicate composition**: which relation predicates show up on productive 2-hop paths? Deferred until live telemetry on Phase 2 production accumulates — initial implementation walks ALL predicates (no edge whitelist).

### Variance note (informational, not blocking)

This run's overall recall_reach was 0.870 vs May 13's 0.873 — within noise. But cat-3 swung 6.6pp (0.740 vs 0.806). Same code, same dataset, different Haiku extraction run. Phase 0 has higher run-to-run variance on cat-3 specifically because cat-3 depends on which specific relations get extracted. The Phase 2 ablation will run Phase 2 ON and Phase 2 OFF against the **same** cached corpus (this run's snapshots), so extraction variance is controlled out and we measure the marginal Phase 2 contribution cleanly. The variance question is independent and will be addressed via n=3 runs for any future single-number reporting.

---

## Phase 2C — Implementation

### Note on Phase 1B obsolescence

The `memory_graph_retrieval.h` doc comment names "Phase 1B" as planned successor: structured-relation synthesis for relations that exist as graph edges without an associated fact (Phase 1A walks fact-linked relations only — ~40% of relations at the dev's pre-Phase-0 scale). Phase 0's paired-output schema guarantees ~100% fact↔relation linkage on new extractions, so Phase 1B's motivation is largely obsolete for new memory. For the dev's existing corpus, Phase 1B's gap is real but resolves on production re-extract (deferred until memory work complete per current strategic decision).

**Phase 2 builds on Phase 1A directly; Phase 1B is retired as a separate workstream.**

### Step 1 — Query-aware graph candidate scoring (primary work)

The core insight: graph search currently uses a flat `entity_grounding_bonus` score that has **nothing to do with the query content**. For "Would John be considered a patriotic person?" all 64 facts touching John get score 0.4 — the topic-irrelevant ones (John's allergies, John's morning routine) score identically to the topic-relevant ones (John joined the military). Hybrid scoring against the actual query is the signal that's missing.

#### Algorithm

```
Inputs: query_text, user_id

1. EXTRACT seeds via existing memory_graph_extract_seed_entities(query_text)
   → seed_entity_ids[]  (capitalized proper-noun resolution against memory_entities)

2. EXPAND candidates via existing memory_graph_expand_n_hop(seeds, max_hops)
   → graph_fact_ids[]  (entity-bounded pool, default max_hops=1 until Step 2 activates)

3. SCORE every graph candidate against query
   For each fact_id in graph_fact_ids:
     hybrid_score = kw_weight * keyword_overlap(query, fact.text)
                  + vec_weight * cosine(query_embedding, fact_embedding)
     graph_score  = hybrid_score + entity_bonus    // small additive

4. MERGE with hybrid pool
   union(hybrid_top_K_fact_ids, graph_fact_ids)
   re-sort by score descending
   truncate to N (current 10-slot working array)
```

The graph layer becomes a **candidate-pool widener**, not a parallel scorer. All scoring runs through one path. The `entity_bonus` (default 0.10) is a small additive that distinguishes graph-rescued candidates from pure-cosine candidates — enough to break a tie between two equally-scored facts, not enough to override semantic relevance.

#### Why this works (and the flat-score version didn't)

Under flat scoring, the 64 John-facts compete on irrelevance. Under hybrid scoring:
- John's military service fact: cosine to "patriotic" ≈ 0.5, score becomes 0.5 + 0.1 = 0.6
- John's allergy fact: cosine to "patriotic" ≈ 0.2, score becomes 0.2 + 0.1 = 0.3

Now the military fact competes meaningfully with hybrid's top-10 (which scored 0.5-0.7). The allergy fact correctly stays out. The graph layer's value of "rescuing entity-grounded facts hybrid missed" survives — but only entity-grounded facts that ARE topically relevant get rescued.

#### Module changes

| File | Change |
|---|---|
| `src/memory/memory_callback.c` | Rework graph merge: get graph candidate fact_ids, score each via hybrid logic, merge into unified sorted pool. Either inline the scoring (small change, ~30 LOC) OR extract a shared `memory_score_facts_against_query()` helper used by both pure-hybrid and graph paths |
| `src/memory/memory_graph_retrieval.c` | `memory_graph_expand_n_hop` semantically returns "candidate fact IDs"; the `out_scores` field becomes either zero-filled (caller scores) or carries the small `entity_bonus` so caller doesn't need to know which facts came from the graph layer. Choose based on which keeps `memory_action_search` simpler |
| `include/config/dawn_config.h` | Add `entity_bonus` (float). Mark `entity_grounding_bonus` as legacy (still parsed but unused once Step 1 lands; deprecate after Step 2) |
| `src/config/{config_defaults,config_parser,config_env,config_validate}.c` | Plumb `entity_bonus` (default 0.10, range [0, 1]) through the standard 7 surfaces |
| `src/webui/webui_config.c` + `www/js/ui/settings/schema.js` | Settings panel entry |
| `benchmarks/bench_retrieval.c` + `benchmarks/run_benchmark.py` | New CLI flag `--graph-query-scoring on\|off` for Step 1 config-toggle ablation. ON applies query-scoring; OFF reverts to flat `entity_grounding_bonus` (matches pre-Step-1 behavior) |

#### Reuse opportunities

`memory_fact_search_hybrid` already does cosine + keyword scoring on full-corpus candidates. The scoring math is centralized in `memory_embeddings.c`. The cleanest refactor is to expose the **scoring function as a primitive** that operates on a caller-supplied fact ID list, then have hybrid call it (against the embedding-search candidate list) and the graph layer call it (against the graph-expansion candidate list).

Proposed primitive (sketch):

```c
/* Score a caller-supplied list of fact IDs against the query.
 * Uses the same kw + cosine + temporal scoring as memory_fact_search_hybrid
 * but operates on a bounded candidate set rather than the full corpus. */
int memory_score_facts_against_query(int user_id,
                                     const char *query_text,
                                     const int64_t *candidate_fact_ids,
                                     int candidate_count,
                                     memory_fact_t *out_facts,
                                     float *out_scores,
                                     int max,
                                     int *out_count);
```

#### Unit tests

New `tests/test_memory_graph_query_scoring.c` (Unity), or extension of `test_memory_graph_traversal.c`:

1. Topic-irrelevant graph candidate (e.g., "John has cat allergies") + topic-relevant query ("Would John be patriotic?") → low score
2. Topic-relevant graph candidate ("John joined the military") + same query → high score (close to or above weak hybrid hits)
3. Tied query relevance: graph candidate vs identical-text non-graph candidate → graph wins by `entity_bonus`
4. `entity_bonus=0`: graph candidates score identically to pure hybrid (back-compat ablation)
5. Empty graph candidate list → SUCCESS, no error
6. Score floor still applies: weak query-relevance graph candidates correctly drop below the floor
7. Same fact in both hybrid and graph pools after dedup → keeps the higher of the two scores

#### Phase 2C-Step 1 ablation knob

Bench CLI: `--graph-query-scoring on|off`
- `on` (default after Step 1 ships): query-aware scoring (this section's algorithm)
- `off`: flat `entity_grounding_bonus` (pre-Step-1 behavior, today's production)

The same binary supports both configurations. Step 1 ON vs OFF on the same cached corpus measures Step 1's marginal contribution cleanly.

### Step 2 — 2-hop traversal (deferred)

**Status**: foundation in tree (this session's Phase 2C work). Gated to `max_hops=1` by default until Step 1 ships and measurement confirms 2-hop adds further lift beyond what query-scoring captures.

#### What's already done (this session, uncommitted)

| Surface | Status |
|---|---|
| `memory_graph_expand_n_hop` in `memory_graph_retrieval.{c,h}` | Implemented, cycle prevention, hop-2 decay, dedup max-merge |
| `[memory.graph_retrieval]` config keys `max_hops`, `hop_2_decay` | Plumbed through defaults / parser / env / validator / webui / schema / dawn.toml.example |
| `memory_action_search` call site | Updated to use `memory_graph_expand_n_hop` reading `max_hops` from config |
| Bench CLI `--max-hops` | Implemented |
| Bench command `query_graph_two_hop` | Added for diagnostic use (cat3_graph_2hop_reachability.py) |

#### Adjustment for revised scope

- **Revert `max_hops` default from 2 to 1** in `src/config/config_defaults.c` and `dawn.toml.example` until Step 1 measurement validates Step 2's marginal value
- Keep all other Step 2 code intact — it's the foundation Step 2 ablation will exercise

#### Step 2 unit tests (when activated)

New `tests/test_memory_graph_traversal.c` (Unity):

1. Single seed, 1-hop returns same as Phase 1A wrapper
2. Single seed, 2-hop returns hop-1 ∪ hop-2 facts deduplicated
3. Cycle: A→B→A — visited set prevents re-traversal, count returns sane
4. Hub entity: seed with 100+ neighbors respects per-hop neighbor cap
5. Empty seed set: SUCCESS, count=0
6. Decay: hop-2 score = hop-1 score × `hop_2_decay` (under flat-score legacy path)
7. Max-merge: fact appearing at both hops gets hop-1 score
8. NULL guards on output arrays
9. Capacity exhausted at hop-1 — hop-2 not attempted; count == max
10. `max_hops` out of range gets clamped (defensive)

Defer writing these until Step 2 activates; the current 2-hop machinery is only exercised under explicit config override during Phase 2D ablation.

### Bitemporal filter at query time (still deferred to Phase 2.5)

Phase 2A's reachability diagnostic didn't classify temporally-filtered misses. Until a separate diagnostic measures whether bitemporal helps, no implementation. Plan: revisit after Step 1 + Step 2 ship, with a fresh diagnostic that distinguishes "this miss would be rescued by bitemporal filter" as a class.

---

## Phase 2D — Post-flight measurement (two-step ablation)

Phase 2 ships in two stages, each with its own measurement gate. Both run against the **same cached corpus** at `./benchmarks/snapshots_phase2_base/` — no re-extraction.

### Step 1 ablation (after query-scored graph candidates land)

**Step 1 ON** (production default):
```bash
python3 benchmarks/run_benchmark.py \
    --binary ./build-debug/tests/bench_retrieval \
    --benchmark locomo \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --memory-pipeline \
    --cache-dir ./benchmarks/snapshots_phase2_base \
    --graph-query-scoring on \
    --max-hops 1 \
    --output ./benchmarks/bench_step1_on.json
```

**Step 1 OFF** (pre-Step-1 behavior, flat scoring):
```bash
python3 benchmarks/run_benchmark.py \
    --binary ./build-debug/tests/bench_retrieval \
    --benchmark locomo \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --memory-pipeline \
    --cache-dir ./benchmarks/snapshots_phase2_base \
    --graph-query-scoring off \
    --max-hops 1 \
    --output ./benchmarks/bench_step1_off.json
```

**Sanity check**: Step 1 OFF must reproduce 0.8696 overall / 0.740 cat-3 (today's measured baseline). If not, config toggle isn't fully un-wiring; escalate to git-revert.

**Ship/measure decision for Step 1**:

| Outcome | Action |
|---|---|
| cat-3 ≥ +5pp marginal AND no category regresses >1pp | **Ship Step 1; proceed to Step 2 measurement** |
| cat-3 +3-5pp marginal | **Ship Step 1 with caveat; still proceed to Step 2** |
| cat-3 < 3pp marginal | **Hold Step 1; investigate why query-scoring didn't deliver** (likely indicates a deeper retrieval issue) |
| Any category regresses >1pp | **Hold; diagnose** |
| Step 1 OFF ≠ pre-Step-1 baseline ±0.5pp | **Escalate to git-revert** |

### Step 2 ablation (after Step 1 ships, only if Step 1 didn't capture everything)

Conditional gate: if Step 1 lands at >85% of the 43.8% reachability ceiling, Step 2 ROI is low and we **may skip**. If Step 1 lands at <60% of the ceiling, Step 2 has meaningful headroom.

**Step 2 ON** (with Step 1 also on):
```bash
python3 benchmarks/run_benchmark.py \
    --binary ./build-debug/tests/bench_retrieval \
    --benchmark locomo \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --memory-pipeline \
    --cache-dir ./benchmarks/snapshots_phase2_base \
    --graph-query-scoring on \
    --max-hops 2 \
    --output ./benchmarks/bench_step2_on.json
```

**Step 2 OFF** (Step 1 still on, hop-1 only):
```bash
python3 benchmarks/run_benchmark.py \
    --binary ./build-debug/tests/bench_retrieval \
    --benchmark locomo \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --memory-pipeline \
    --cache-dir ./benchmarks/snapshots_phase2_base \
    --graph-query-scoring on \
    --max-hops 1 \
    --output ./benchmarks/bench_step2_off.json
```

Now the ablation measures the **marginal contribution of 2-hop on top of query-scored graph** — i.e., the structural improvement we couldn't measure before because flat scoring blocked it.

**Ship/measure decision for Step 2**:

| Outcome | Action |
|---|---|
| Marginal cat-3 lift ≥ +3pp AND no regression | **Ship Step 2** (default `max_hops` becomes 2) |
| Marginal cat-3 lift +1-3pp | **Ship with caveat** — confirms 2-hop has value but reranker work (separate item) may be higher ROI |
| Marginal cat-3 lift < 1pp | **Hold Step 2** — Step 1 captured the available reachable lift; expand candidate pool isn't the constraint |
| Latency cost > 2× retrieval time | **Hold pending hop-2 latency tuning** |

### Comparisons (apply to both step ablations)

1. **Per-category recall_reach** — full breakdown (cat-1/2/3/4/5)
2. **Cat-3 per-query before/after** — did 2A's identified misses actually move from rank-11-50 to top-10?
3. **Regression check** — non-cat-3 categories within ±1pp of baseline
4. **Latency** — bench wall-clock + per-query timing (Step 1 adds embedding lookups on graph candidates; Step 2 adds 2-hop neighbor walks)

### Estimated cost (revised)

- Step 1 implementation + ablation: ~half-day agent + ~$0 bench (cache hits, ~70 min)
- Step 2 ablation (foundation already in tree): ~$0 bench (cache hits, ~70 min)
- **Total Phase 2D: $0 bench, ~2.5 hours wall**

---

## Risk surfaces & mitigations

| Risk | Mitigation |
|---|---|
| **Step 1: query-scoring adds noise on cat-3 queries with bad seeds** | Step 1 is an additive improvement over flat scoring; worst case it scores graph candidates identically to hybrid (when query embedding is uninformative), which is no worse than not having graph rescue at all |
| **Step 1: entity_bonus too high → graph candidates dominate over hybrid** | Default 0.10; bench-tunable; ablation isolates `entity_bonus=0` (pure hybrid) as a sanity check |
| **Step 1: scoring N graph candidates adds latency** | Each candidate is one embedding cache lookup + one cosine + keyword overlap; N≤30 typically; ~5-10ms overhead per query. Within budget |
| **Step 2 (2-hop) pulls in noise displacing correct hop-1 items** | Step 1's query-scoring filters by relevance, not entity match. Step 2's wider candidate pool is rescued only when query-relevant. Hub-entity neighbor cap (`MEMORY_GRAPH_HOP_NEIGHBORS_PER_SEED=64`) bounds blowup |
| **2-hop latency exceeds budget** | Measured against Step 1 baseline; if 2-hop turns out >2× slower at no extra recall, hold Step 2 |
| **Step 1 doesn't deliver expected lift** (e.g., <3pp cat-3 marginal) | Indicates a deeper retrieval issue — likely hybrid scoring itself is the constraint (kw_weight, vec_weight, score floor). Re-diagnose before going further |
| **Subject-FK NULL on old facts blocks future graph traversal off facts** | Deferred TODO from Phase 0 — entity-merge propagates FK on alias. Not blocking Phase 2 because Phase 2 walks `memory_relations.subject_id` / `object_id`, not `memory_facts.subject_entity_id` |
| **Phase 1A's seed extraction misses query intent** | Existing limitation. Step 1 mitigates by also weighting graph candidates by query semantics — even if seed extraction picks a sub-optimal entity, the irrelevant facts under it get filtered by query score. If seed extraction misses the entity entirely, no graph rescue happens (separate workstream) |

---

## Open questions

### Step 1 design choices (to resolve during implementation)

1. **`entity_bonus` magnitude**: 0.10 is the starting guess. Too high = graph candidates dominate even when topically irrelevant; too low = graph rescue value disappears. Bench-tune in Step 1 ablation.
2. **Refactor vs inline**: extract `memory_score_facts_against_query()` as a shared primitive between hybrid + graph paths, or inline the scoring in the graph-merge code path? Shared primitive is cleaner but a larger change. Decide based on code-review feedback during implementation.
3. **Score legacy `entity_grounding_bonus`**: deprecate immediately or maintain as a config-toggle fallback? Recommendation: keep parsed/clamped for one release, log deprecation warning, remove next cycle.

### Step 2 questions (only relevant if Step 2 ships)

4. **Hop-2 decay factor**: 0.6 was the original guess for flat-score Phase 2. Under Step 1's query-scoring path, decay may want to be different (or 0.0 = no decay, let query relevance be the only differentiator).
5. **Edge whitelist composition**: which relation predicates carry signal at hop-2? Defer until 2-hop production telemetry exists.
6. **Asymmetric edges**: `(Alice, parent_of, Bob)` — does 2-hop want to traverse `(Bob, parent_of⁻¹, Alice)` automatically? Initial answer: NO, only follow stored edges. Inverse-relation synthesis is a separate workstream if it becomes a bottleneck.
7. **Memory tool descriptor update**: does the LLM benefit from knowing the tool walks 2 hops? Decide if/when Step 2 ships.

---

## Follow-up TODOs (defer until Phase 2 ships)

1. **Entity-merge propagates `subject_entity_id`** — gated on first FK reader. Phase 2 doesn't read it (walks relations), so Phase 2 doesn't trigger this. Defer.
2. **v48 NOT NULL migration on `subject_entity_id`** — gated on phase0 summary log NULL rate stabilizing near zero. Independent of Phase 2.
3. **Predicate-dedup stem-strip pass** — gated on telemetry showing morphological duplicates accumulating. Independent of Phase 2.
4. **Inverse-relation synthesis** — `(Alice, parent_of, Bob)` ⇒ `(Bob, child_of, Alice)` derived edge — gated on 2A surfacing this as a miss class. May obsolete the 2-hop "symmetry" workaround.
5. **3+ hop traversal** — explicitly out of scope. Revisit only if 2D shows substantial unreached evidence at 3 hops AND latency budget permits.

---

## References

- Phase 0 design: `docs/PHASE_0_EXTRACTION_PROMPT_DRAFT.md` (untracked)
- Phase 0 bench result: `benchmarks/bench_phase0_2026-05-13.json`
- Phase 1A module: `src/memory/memory_graph_retrieval.{c,h}`
- Phase 2A 1-hop diagnostic: `benchmarks/cat3_graph_reachability.py`
- Phase 2A 2-hop diagnostic (new, this phase): `benchmarks/cat3_graph_2hop_reachability.py`
- Phase 2A bench result + snapshot corpus: `benchmarks/bench_phase2_baseline.json` + `benchmarks/snapshots_phase2_base/`
- LoCoMo cat-3 profiling (sparse-graph era): `atlas/dawn/memory/LOCOMO_CAT3_PROFILING.md`
- Memory subsystem state: `atlas/dawn/memory/STATE.md`
- Atlas memory design: `atlas/dawn/memory/SYSTEM_DESIGN.md` (note: schema sections are pre-Phase-0)

---

## Workflow checklist

### Completed (May 14, 2026 — shipped this commit)

**Phase 2A diagnostic**:
- [x] Generate snapshot corpus (cached at `benchmarks/snapshots_phase2_base/`)
- [x] Capture cat-3 misses on dense graph (`/tmp/cat3_misses_phase2_base.json`)
- [x] Reachability profile (1-hop and 2-hop variants)

**Phase 2B scope analysis**: original 2-hop scope abandoned after Phase 2C investigation revealed (a) the 43.8% ceiling lift doesn't pass through the production merge layer, and (b) LoCoMo cat-3 doesn't contain genuine multi-hop test cases.

**Phase 2C Step 1 implementation**:
- [x] New scoring primitive `memory_embeddings_rescore_against_query()` with sentinel contract
- [x] Score-based competitive merge in `memory_action_search`
- [x] Pre-computed embedding (Shape B) — eliminates duplicate ONNX inference per call
- [x] `use_query_scoring` config knob (default true)
- [x] `entity_bonus` config knob (default 0.0 — audit-validated; >0 acts as asymmetric thumb-on-scale and regresses)
- [x] Bench CLI `--graph-query-scoring on|off` and `--entity-bonus <float>` for ablation
- [x] Six new Unity tests pinning the rescore contract (NULL safety, sentinel semantics, sentinel-distinct-from-valid invariant)

**Phase 2C Step 2 strip (post-architectural review)**:
- [x] `memory_graph_expand_n_hop` + helpers removed (~270 LOC)
- [x] `max_hops` + `hop_2_decay` config keys removed from all 7 plumbing surfaces
- [x] `--max-hops` bench CLI removed
- [x] Diagnostic preserved: `benchmarks/cat3_graph_2hop_reachability.py` + bench `query_graph_two_hop` command

**Unified retrieval primitive (consolidation)**:
- [x] `memory_search_execute()` in `memory_fact_search.{c,h}`
- [x] `memory_action_search` non-category branch delegates to it
- [x] Bench `handle_query_memory` delegates to it
- [x] Bench now applies `search_score_floor` (audit HIGH-1 closed)
- [x] Bench uses production retrieval shape (audit MEDIUM-1 closed)

**Cleanup**:
- [x] Sentinel-check pattern uses `MEMORY_EMBEDDINGS_RESCORE_SENTINEL` macro (security M-2)
- [x] `MEMORY_GRAPH_INTERMEDIATE_CAP` promoted to `memory_graph_retrieval.h` (arch M-1)
- [x] Five reviews passed (architecture × 2 / embedded / security / coding-standards)
- [x] `format_code.sh --check --changed` clean
- [x] All 58 CI tests pass
- [x] ConvoMem sanity (0.990 unchanged), LongMemEval skipped (same raw-cosine path; unaffected)

### Deferred to follow-up workstreams

- [ ] LoCoMo entailment benchmark (`--gen-and-judge`) for true leader-comparable comparison vs ByteRover / MemMachine
- [ ] Step 2 (2-hop) — re-introduce only if a multi-hop benchmark (HotpotQA / 2WikiMultiHop / MuSiQue / synthetic) is added
- [ ] Phase 2.5 (bitemporal at query time) — pending separate diagnostic
- [ ] Embedded-review follow-ups: permutation-index sort in merge (saves struct copies), reader/writer lock on s_cache (when multi-session)
- [ ] `memory_callback.c` size cleanup (still >1500 lines after this commit's extraction, file-level cohesion concern)

### Post-commit

- [ ] Update atlas `STATE.md` (separate atlas repo commit)
- [ ] Update local `docs/TODO.md` (gitignored)
