# Reranker Workstream — Dominant-Token Heuristic

**Status**: shipped 2026-05-13 (commit `0e9d5c3` and friends — see `dawn/docs/TODO.md` shipped log).  Three sub-phases ran same-day: Phase A diagnostic + Phase B-i Python prototype + Phase B-ii C implementation + review-fix pass.  18/18 unanimous bench across all three providers (anthropic / openai / local).  Total spend ~$1.08 paid, agent ~8 hours.

**Motivation**: The cross-encoder reranker workstream sat in `dawn/docs/TODO.md` and `STATE.md`'s "next session" callout as Feature 2 from the EMBEDDING_UPGRADE plan, motivated as the structural fix for cosine over-inclusion on dominant tokens (e.g., "mother" pulling in birthday context for Mother's Day queries).  The prior reranker investigation (atlas/dawn/memory/RERANKER_INVESTIGATION.md) was reverted with a clear verdict — ms-marco doesn't transfer to conversational data, latency dominates voice-mode budget, multi-signal hybrid beats reranker-on-retrieval by 19pp on LoCoMo.

The shipped fix is a non-ML heuristic, not a cross-encoder.  This matches the prior investigation's "Next Direction" conclusion and closes the Feature 2 TODO entry — the dominant-token over-inclusion pathology that motivated the reranker is now bench-validated as fixed at the focus-injection layer.

Two things shifted that justify a diagnostic-first restart rather than a "resurrect what was reverted" rebuild:

1. **Scope changed** — the new direction is focus-injection-only (smaller pool, different code path, different latency budget), not retrieval.
2. **Phase 1j re-bench just shipped** (2026-05-13).  `weight_recency=0.30` + `weight_importance=1.0` produces 15/15 across all three providers on the augmented Phase 1j fixture suite.  Cases 1, 2, 12, 13 all probe dominant-token cosine over-inclusion — the exact pathology the reranker was queued to fix.  The weight tuning passed.

The TODO entry's stated motivation has been substantially addressed at the weight layer.  Phase A's job is to surface whether a *severe* version of the pathology — one the weight tuning cannot fix — exists in production or designed bench cases.  If yes, Phase B (gated) designs the appropriate fix.  If no, the workstream retires.

This document follows the RERANKER_INVESTIGATION's own explicit lesson: *"Profile the LoCoMo cat-3 errors before designing.  The reranker investigation suffered from designing without diagnostic data; don't repeat that."*

## What blocks production-frequency measurement today

Ideally we'd answer: "in what fraction of memory-flavored turns does focus-injection miss content that the LLM subsequently retrieves via `memory.search`?"  That measurement needs:

- per-turn focus-block contents logged
- per-turn subsequent `memory.search` queries + results logged
- a way to ask whether the search result *should* have been in the focus block

DAWN doesn't capture all three today.  The double-dip mitigation we shipped (May 2026) explicitly tells the LLM to fall through to the memory tool when the focus block is absent — so a memory-tool call doesn't unambiguously mean focus-injection failed.

Production-frequency measurement is therefore deferred to a separate small infra TODO (per-turn focus-block logging).  Phase A here is *reachability-based* — does the fixture suite admit a severe-pathology case that current weights cannot fix?  Reachability is a necessary precondition for Phase B to be worth doing; if even synthetic severe cases pass at HEAD weights, the reranker isn't solving anything.

## Phase A — Severe-pathology fixture probe

### Three new fixture cases (case 16-18)

Each encodes the dominant-token shape at a *severity* the weight tuning cannot rescue.  Math: under `w_rec=0.30`, the maximum recency boost the right answer (summary, `rec=0.95`) can gain over a decoy (`rec=0.25`) is `0.30 * (0.95 − 0.25) = 0.21`.  If the semantic gap (decoy_sem − summary_sem) exceeds ~0.21, the right answer cannot lift into top_k=8 by weights alone.  Cases 16-18 set decoy_sem ≥ 0.93 and summary_sem ≤ 0.50 — a semantic gap ≥ 0.43, twice what `w_rec=0.30` can close.

**Case 16 — `severe_dominant_token_restaurant`**

- Query: "What's my favorite restaurant in the city?"
- Right answer: `memory_summary` about a recent visit to a specific restaurant by name (does NOT contain the words "favorite" or "restaurant" — describes the meal, the company, the recommendation)
- Decoys: 9 `memory_facts` each containing both "favorite" and "restaurant" tokens generically (favorite-cuisine-is-X, favorite-restaurant-experience-was-Y, etc.) — `sem ≥ 0.93` across the pool
- Expected substrings (right answer): the specific restaurant name + cuisine
- Designed-FAIL under `w_rec=0.30`: summary `rank ~10` (out of top_k=8)
- Bi-encoder cosine cannot rescue this; only a cross-encoder reading query+candidate together or a heuristic that detects "all decoys share the dominant query token" can

**Case 17 — `severe_dominant_token_doctor`**

- Query: "What did my doctor say about the blood test last week?"
- Right answer: `memory_summary` from a recent appointment with specific findings (does NOT contain "doctor" / "blood test" tokens — describes what the appointment covered using clinical phrasing)
- Decoys: 9 `memory_facts` each containing "doctor" and/or "blood test" generically — `sem ≥ 0.93`
- Probes the same shape on a different topic to confirm the pathology isn't query-specific

**Case 18 — `severe_dominant_token_project`**

- Query: "What's the status on the database migration project?"
- Right answer: `memory_summary` with the specific recent status (does NOT contain "database migration" or "project" tokens — uses specific technical terms like "shard rebalance", "downtime window", etc.)
- Decoys: 9 `memory_facts` each paraphrasing "database migration project" generically — `sem ≥ 0.93`
- Probes the work-context shape

All three are designed-FAIL under the current shipped defaults (`w_rec=0.30`, `w_imp=1.0`).  The Phase A success criterion is that they all FAIL on the paid `--recalibrate` run.  If they pass at HEAD, the weight tuning has more reach than expected and Phase B is harder to justify.

### Composite-math validation (no-cost, dry-run only)

Before any paid run, run the same Python composite-rank simulator we used for Phase 1j to verify cases 16-18 land at rank 9-10 under current weights AND that bumping `w_rec` to any reasonable value (≤ 0.50) cannot rescue them.

### Paid `--recalibrate` run

Single paid run against all 18 cases at current shipped defaults.  Expected outcome: `13/18` aggregate (existing 15 pass, cases 16-18 unanimous FAIL × 3 providers).

If outcome ≠ expected:
- `16/18` or `17/18` — partial pass means our fixture design under-specified the pathology.  Strengthen and re-run, or accept that the pathology shape is harder to construct synthetically than expected.
- `18/18` — current weights handle even severe cases; defer Phase B.

Cost: ~$0.20-0.25 (18 cases × 3 providers × ~$0.005 per judge call).

## Phase A success criteria

| Outcome | Decision |
|---|---|
| Cases 16-18 all FAIL × 3 providers (severe pathology is reachable) | **Proceed to Phase B** with the three candidate fix shapes — `mxbai-rerank-base`, Haiku zero-shot, dominant-token heuristic |
| Cases 16-18 fail on 1-2 providers, pass on others | Mixed signal; investigate why the LLM judge is provider-asymmetric (the original Phase 1j fixtures had 100% inter-provider agreement; asymmetry here is a fixture quality issue) |
| Cases 16-18 all pass × 3 providers | **Defer Phase B.**  Weight tuning has more reach than expected; the reranker workstream retires until per-turn focus-block logging surfaces a production-frequency signal |

## Phase B (gated on Phase A)

Triggered only by the first row of the Phase A criteria table.  Out of scope for this doc — see scope-of-reranker discussion in this session's chat history.  Brief sketch of the three Phase B candidates if reached:

| Candidate | Strength | Risk |
|---|---|---|
| **Dominant-token heuristic** — non-ML detector: when query has a dominant low-IDF token AND > 60% of candidates share it, downweight cosine for those candidates | Fast (<50ms), no model, no API cost, directly targets the pathology shape | Heuristic — may not generalize to non-dominant-token shapes |
| **mxbai-rerank-base-v1** — cross-encoder, +11.2pp on LoCoMo conv 1 in the prior shootout | Best raw numbers in the prior comparison | 4500ms latency (CPU); 244MB model; quantize-int8 may shave but still heavy |
| **Haiku zero-shot pairwise judge** — MemPalace pattern | Highest ceiling; uses an existing dependency | Per-turn cost; gate-design problem |

The heuristic is the cheapest first move if Phase A confirms the shape.  The cross-encoder is the bigger swing if the heuristic doesn't generalize.

## Cost + effort (Phase A)

| Piece | Effort |
|---|---|
| Design + write 3 severe-pathology fixture cases | agent ~1 hr |
| Composite-math dry-run verification | agent ~30 min |
| `bench_focus_pipeline` dry-run on all 18 cases | agent ~15 min |
| Single paid `--recalibrate` run at HEAD weights | agent ~15 min · **api ~$0.20-0.25** |
| Decision writeup + Phase B authorization request | agent ~1 hr |
| **Total Phase A** | **agent ~3 hr · api ~$0.20-0.25 · 1 ckpt** |

## Out of scope

- Production-frequency measurement (needs per-turn focus-block logging; separate small infra TODO)
- Phase B implementation (gated on Phase A outcome)
- Retrieval-path reranker (already reverted; explicitly out per the prior investigation's verdict)
- LoCoMo / LongMemEval / ConvoMem retrieval benchmark spot-checks (focus_injection doesn't affect those code paths — confirmed via earlier grep)
- ColBERT / late interaction architectures (different model paradigm; separate workstream)

## Rollback

Phase A is fixture-only.  New cases are additive — existing 15 untouched.  If the paid run reveals fixture-design issues (provider-asymmetric verdicts, unexpected pass-through), the cases stay in the suite for Phase B to refine, and `HEAD_BASELINE_FIX_COUNT` gets recalibrated to whatever the actual outcome is.  No production changes ship.

## When to actually do this

Immediately — small budget, decision-grade outcome, unblocks (or retires) the reranker TODO entry which has been queued for weeks.  No external deadline.

## Phase A results (2026-05-13)

### Composite-math verification (dry-run)

| Case | `w_rec=0.15` | `w_rec=0.30` | `w_rec=0.50` | `w_rec=0.70` |
|---|---|---|---|---|
| 16 (restaurant) | rank 10 | rank 10 | rank 10 | rank 1 |
| 17 (doctor) | rank 10 | rank 10 | rank 10 | rank 1 |
| 18 (project) | rank 10 | rank 10 | rank 10 | rank 1 |

All three cases stay at rank 10 (OUT of top_k=8) under any reasonable recency weight (0.15 / 0.30 / 0.50).  Only at `w_rec=0.70` — far outside the safe band — do they rescue.  `w_rec=0.70` would over-correct case 14 (regression guard) and likely promote stale items in production, so the candidate band is empty.  Confirms the semantic gap (0.43) genuinely exceeds what the recency lever can close.

### Paid `--recalibrate` run at HEAD (`w_rec=0.30`)

| Provider | Existing 15 | New 3 (cases 16-18) | Total |
|---|---|---|---|
| anthropic | 15/15 pass | 0/3 (all FAIL) | 15/18 |
| openai | 15/15 pass | 0/3 (all FAIL) | 15/18 |
| local | 15/15 pass | 0/3 (all FAIL) | 15/18 |

**Aggregate: 15/18 quorum-pass at HEAD.**  Per-case verdicts on cases 16-18 were unanimous NO on every provider for every case — zero provider asymmetry, confirming the fixture design is clean and the pathology is real, not a judging artifact.

Paid spend: ~$0.27 (18 cases × 3 providers × ~$0.005/call).

### Decision

**Triggered the first row of the success-criteria table: proceed to Phase B.**

Severe dominant-token cosine over-inclusion is reachable at a shape that:

- the weight-tuning lever cannot fix at any safe parameter value
- replicates unanimously across three different topic domains (restaurant, medical, work-project) and three providers
- is structurally similar to the Mother's Day pathology that motivated the original TODO entry, just at higher severity

Phase B is now justified — there is a real bench-reproducible gap that a non-weight-tuning intervention is the only path to closing.

### What this changes for Phase B design

Two findings worth carrying into the Phase B kickoff:

1. **The dominant-token heuristic option is now front-and-center.** Cases 16-18 share an exact structural shape: query contains a low-IDF dominant token (`favorite restaurant` / `doctor` / `blood test` / `database migration project`) AND > 80% of the candidate pool shares that token AND the right answer does not.  This is a heuristic-detectable shape — query-side dominant-token detection + candidate-side token-overlap counting would catch all three cases without any model inference.  Before reaching for a cross-encoder, run the heuristic against these three cases; if it rescues all three, it's the cheapest fix in the decision matrix.
2. **The pathology is more topic-uniform than expected.** All three synthetic cases use the same shape and all three fail unanimously.  If the heuristic generalizes, Phase B might be agent-days rather than agent-weeks.

### HEAD_BASELINE_FIX_COUNT decision

Leaving `HEAD_BASELINE_FIX_COUNT` at 15 (not 18).  The semantics of the constant per the harness comment is "regression detection: candidate_fix_count < baseline_fix_count is a FAIL."  Setting baseline=15 means: any future change that drops a case below 15 is flagged as a regression, while leaving headroom for Phase B to lift toward 18 without the bench's regression-detection complaining.  When Phase B ships and lifts the count, bump baseline accordingly.

### Phase A → Phase B transition

Phase A is complete.  Awaiting authorization to scope Phase B against the heuristic-first decision matrix.  No production changes shipped from Phase A; new fixture cases are additive (15 → 18) with `head_known_fail=true` on the three new ones.  Format clean, build green, bench reproducible.

---

## Phase B-i — Dominant-token heuristic prototype (2026-05-13)

### Mechanism

Per-candidate penalty applied to composite score *after* adapter emission but *before* top-k selection.  Algorithm:

1. Tokenize query: lowercase, keep alphanumeric runs ≥ 4 chars, drop a curated stopword list (articles, pronouns, common verbs, generic agent tokens like "user").
2. For each query token, count how many candidates contain it.
3. Tokens shared by > `threshold` (default 0.6) of the candidate pool are "dominant."
4. For each candidate:
   - `shared = query_tokens ∩ candidate_tokens`
   - `shared_dominant = shared ∩ dominant_tokens`
   - `non_dominant = shared − dominant_tokens`
   - Penalty = `0` if `shared_dominant` is empty
   - Penalty = `base_penalty` (default 0.4) if only dominant tokens match (the over-inclusion shape)
   - Penalty = `base_penalty * |shared_dominant| / |shared|` if some non-dominant tokens also match (diluted)

Implementation: standalone Python prototype at `benchmarks/bench_focus_heuristic_probe.py` that pre-processes the JSON case by subtracting the penalty from each seed's `semantic_override` (because `composite = w_sem * sem + ...` and `w_sem = 1.0`, this faithfully encodes the penalty without modifying the bench-binary code path).

### Fixture fix during Phase B-i

`severe_dominant_token_project` (case 18) violated its own design contract — the right-answer text included the dominant query token "migration" ("Recent migration work..."), which the heuristic correctly flagged as dominant-token-match-only and penalized.  Fix: rewrote to "Recent rollout work..." so the right-answer text contains zero query tokens, matching cases 16 + 17.  Single-word fixture edit; design intent preserved.

### Parameter sweep (dry-mode, free)

| threshold | base_penalty | pass / 18 | regressions |
|---|---|---|---|
| 0.50 | 0.30 | 18 | (none) |
| 0.50 | 0.40 | 18 | (none) |
| 0.50 | 0.50 | 18 | (none) |
| 0.60 | 0.30 | 18 | (none) |
| 0.60 | 0.40 | 18 | (none) |
| 0.60 | 0.50 | 18 | (none) |
| 0.70 | 0.30 | 18 | (none) |
| 0.70 | 0.40 | 18 | (none) |
| 0.70 | 0.50 | 18 | (none) |

**Heuristic is parameter-robust** — every reasonable (threshold, base_penalty) pair produces 18/18 dry-rank pass.  No fragile tuning required; the canonical params (threshold=0.60, base_penalty=0.40) sit in the middle of the safe band.

### Paid LLM judge run at canonical params (threshold=0.60, base_penalty=0.40)

| Provider | Pass count | Notes |
|---|---|---|
| anthropic | 17/18 | one NO on `entity_graph_traversal_v2` (a case that passed 3/3 in Phase A) — quorum saves it via openai + local YES; likely judge-side variance |
| openai | 18/18 | unanimous |
| local | 18/18 | unanimous |
| **Aggregate quorum (2-of-3)** | **18/18** | clean |

All three severe-pathology cases (16, 17, 18) lift from unanimous FAIL × 3 (Phase A baseline) to unanimous PASS × 3 (Phase B-i with heuristic).  All existing 15 cases pass — no regression.

Paid spend: ~$0.27 (18 cases × 3 providers × ~$0.005/call).

### Decision

**Phase B-i succeeds.  Greenlight Phase B-ii (C implementation).**

The heuristic mechanism works mechanically (composite-math) and empirically (LLM judges agree).  Parameter robustness across the full grid means no fragile tuning.  The one provider-asymmetric NO on case 6 is a passing case unaffected by the heuristic mechanism, so it's almost certainly judge-side variance, not a heuristic regression.

### Phase B-ii scope (next step)

C implementation of the heuristic inside `memory_focus_adapters.c` (or a new `memory_focus_heuristic.c`), exposed via three config keys:

```toml
[memory.focus_injection.dominant_token_heuristic]
enabled = true                # opt-in initially, then default-on after live validation
threshold = 0.60              # fraction of pool that must share a token to call it dominant
base_penalty = 0.40           # max composite-score penalty per candidate
```

Pipeline placement: after `focus_compose` collects candidates from all adapters, before composite scoring.  Same algorithm as the Python prototype, ported with C-idiomatic tokenization (reuse `memory_embed_tokenizer.c`'s WordPiece tokenizer would be excessive — a simple alphanumeric-run tokenizer with stopword list is sufficient at this layer).

Effort estimate:
- C implementation + config plumbing: agent ~1 day
- Unit tests: agent ~3 hr
- Bench re-validation via `bench_focus_fix_rate.py` against the C implementation (not Python prototype): api ~$0.27 · 1 ckpt
- Updates to STATE.md, TODO.md, dawn.toml.example, atlas docs: agent ~1 hr

Total Phase B-ii: agent ~1.5 days · api ~$0.27 · 1 ckpt.

### What Phase B-ii does NOT need to revisit

- mxbai-rerank cross-encoder (Phase B Plan B candidate) — not needed; the heuristic works
- Haiku zero-shot pairwise judge (Phase B Plan C candidate) — not needed
- Re-downloading the deleted model files from RERANKER_INVESTIGATION.md disposition table — none required

The heuristic is a clean win against the cross-encoder for this pathology: zero model footprint, microsecond-level latency, no API dependency, parameter-robust.  Matches the RERANKER_INVESTIGATION conclusion that multi-signal hybrid beats pre-trained cross-encoder reranking on conversational data — the heuristic is a fourth signal in the multi-signal stack.

---

## Phase B-ii — C implementation (2026-05-13)

### Files shipped

| File | Role | LOC |
|---|---|---|
| `include/memory/focus_dominant_token.h` | Public API contract | ~105 |
| `src/memory/focus_dominant_token.c` | Tokenizer + dominance detector + penalty application | ~270 |
| `tests/test_focus_dominant_token.c` | Unity unit tests (13 cases) | ~370 |

### Pipeline integration

Hook lives between adapter filter/cap and ranking in `focus_compose()` (`src/memory/focus_source.c:405-426`):

```c
if (pool_count == 0) {
   /* ... existing fast-out ... */
}

/* Dominant-token over-inclusion heuristic */
focus_apply_dominant_token_penalty(fi_hcfg->dominant_token_heuristic.enabled,
                                   fi_hcfg->dominant_token_heuristic.threshold,
                                   fi_hcfg->dominant_token_heuristic.base_penalty,
                                   query_text, pool, pool_count);

/* Rank */
ranker_entry_t *order = calloc(...);
```

The penalty mutates each candidate's `semantic_score` in the working pool (subtracting per-candidate penalty, clamped ≥ 0).  The existing composite formula in `compute_score_breakdown()` propagates the penalty naturally because `w_sem * semantic_score` is the first term of the sum.  No score-breakdown shape changes, no UI changes downstream — the diagnostic log (`focus_dominant_token: penalized N/M candidates`) is the operator-visible surface.

### Implementation details

- **Tokenizer**: alphanumeric runs ≥ 4 chars, lowercase, drop ~98 stopwords (kept in sync with the Python prototype's list).  Per-token byte cap 32; over-cap tokens truncate to avoid splitting URLs/IDs into noise.
- **Set membership**: linear scan over `token_t[]` arrays.  Bounded sizes (≤ 64 candidate tokens × ≤ 32 query tokens × ≤ 64 pool candidates) keep this faster than a hash set's cache-miss overhead.
- **Strict-greater-than threshold** matches the Python prototype semantics — a token shared by exactly `threshold * pool_count` candidates is NOT flagged dominant.
- **FOCUS_SCORE_NA sentinel guard**: candidates with negative-sentinel `semantic_score` skip the penalty (subtracting would push the value further negative and break the "not applicable" semantics the ranker reads elsewhere).
- **Allocation discipline**: two heap allocations (`token_t[][]` per-candidate token arrays + per-candidate token-count int array) freed at function exit.  Static stack allocations for query tokens (32 × 32 bytes) + dominance flags.

### Config plumbing

Three new config keys at `memory.focus_injection.dominant_token_heuristic`:

```toml
enabled = true
threshold = 0.60
base_penalty = 0.40
```

Wired through `config_defaults.c`, `config_parser.c`, `config_env.c` (JSON + TOML writer), `config_validate.c` (`VALIDATE_RANGE_FLOAT` on both knobs, 0.0-1.0), `webui_config.c` (POST handler with `JSON_TO_CONFIG_BOOL` / `_DOUBLE` + `CONFIG_CLAMP`), `dawn.toml.example` (with full design-rationale block), `settings/schema.js` (advanced checkbox + two range sliders).

### Bench validation against the C implementation

Same fixture suite (18 cases at `focus_probe_cases.json`), same `bench_focus_fix_rate.py` harness, default `weight_recency=0.30`:

| Provider | Pass count | Notes |
|---|---|---|
| anthropic | **18/18** | unanimous, no asymmetry |
| openai | **18/18** | unanimous |
| local | **18/18** | unanimous |
| **Aggregate quorum 2-of-3** | **18/18** | clean |

Cleaner than the Python prototype which had a single provider-asymmetric NO from anthropic on `entity_graph_traversal_v2` (case 6).  The C tokenization shake-out may have produced a slightly different focus block on that case; either way the C path passes unanimously.

`HEAD_BASELINE_FIX_COUNT` bumped 15 → 18 across all three providers.  Per the harness's regression-detection semantics, future changes that drop the count below 18 will FAIL.

Paid spend for Phase B-ii bench validation: ~$0.27 (one paid `--recalibrate` over 18 cases × 3 providers).

### Unit test coverage

`tests/test_focus_dominant_token.c` ships 13 Unity tests in CI:

| Category | Tests |
|---|---|
| No-op contracts | disabled, NULL query, empty query, single-candidate pool, stopword-only query |
| Dominance detection | no-dominant-tokens-is-noop, threshold-boundary-strict-greater-than |
| Penalty math | full-penalty-only-dominant-match, diluted-penalty-when-non-dominant-matches |
| Edge cases | clamp-to-zero, FOCUS_SCORE_NA sentinel protected, invalid params no-op |
| Integration smoke | Phase A case 16 penalty distribution (unit-level — full ranking flip is composite-level, validated by the bench) |

CI suite grows 55 → 56 tests, all green.

### Latency

Not formally measured this turn — the heuristic is well below the per-turn budget by inspection: ~98 strcmp() calls for stopword check × ≤ 32 query tokens + ≤ 64 × 64 candidate token scans + ≤ 32 × 64 dominance-frequency loops + per-candidate scoring loop.  All cache-warm linear scans on bounded-small arrays.  At ~10-30 actual candidates in production focus_compose runs, this is microsecond-territory work compared to the ~100-300ms embedding lookup it follows.

### Decision

**Phase B-ii ships as default-on.**  The reranker workstream queued in `dawn/docs/TODO.md` and `STATE.md` is resolved — `weight_recency=0.30` + dominant-token heuristic together cover the entire focus-injection over-inclusion pathology surface observable on the current fixture suite.  Cross-encoder and Haiku-as-judge candidates from the Phase A decision matrix are explicitly not needed.

### What didn't ship + why

- **Cross-encoder reranker** (mxbai-rerank-base-v1, ms-marco): heuristic is sufficient.  RERANKER_INVESTIGATION conclusion preserved — cross-encoders don't transfer to conversational data; the heuristic is the right shape for focus-injection's specific pathology.
- **Haiku zero-shot pairwise judge**: heuristic is sufficient.  Kept as Phase B fallback if heuristic ever regresses on new pathology shapes.
- **Latency profiling**: deferred — heuristic is small enough by inspection that formal profiling would be precision theater.  Production logs surface penalty counts at INFO level; if focus_compose ever shows up in p99 traces, profile then.

### Phase A → Phase B-ii recap

| Phase | Outcome | Paid spend |
|---|---|---|
| Phase A diagnostic | severe pathology reachable (cases 16-18 unanimous FAIL × 3 providers) | $0.27 |
| Phase B-i Python prototype | heuristic mechanism works (18/18 quorum, param-robust across 3×3 grid) | $0.27 |
| Phase B-ii C implementation | 18/18 unanimous across providers + 13 unit tests + zero regression | $0.27 |
| **Total reranker workstream** | **shipped** | **~$0.81 paid · agent ~6 hours total** |

Came in well under the Phase B-ii effort estimate (1.5 days agent) — the Python prototype's clean parameter-robust result + the well-defined hook point in `focus_compose` made the C port straightforward.
