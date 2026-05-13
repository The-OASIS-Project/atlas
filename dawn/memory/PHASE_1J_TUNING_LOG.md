# Phase 1j Focus-Injection Weight Tuning Log

**Status:** Closed — single iteration produced 11/11 across all providers.

**Date:** 2026-05-08.

**Bench:** `benchmarks/bench_focus_fix_rate.py` (Component 6 — end-to-end
fix-rate probe).  Multi-model 2-of-3 quorum across anthropic / openai /
local.  v2 fixtures (`benchmarks/focus_probe_cases.json` schema 2,
11 cases — 4 designed-FAIL, 6 designed-PASS, 1 negative-empty).

**Pre-tuning HEAD baseline** (Phase 1j.A lockdown 2026-05-08):
anthropic 7/11, openai 7/11, local 7/11.

**Post-tuning HEAD baseline** (committed in this phase):
anthropic 11/11, openai 11/11, local 11/11.

---

## Pathology table

The four designed-FAIL cases each map to a specific weight pathology:

| Case | Pathology | Right answer | Default rank (out of 11/10/10/10) |
|---|---|---|---|
| family_kids_competing_partial_matches | Semantic-noise dominates over high-importance right answer | imp=0.85, sem=0.40 fact ("Alice, Bob, Charlie") buried under sem=0.78 mid-importance noise | 11 (drops out of top_k=8) |
| stock_followup_off_topic_v2 | Same semantic-noise pathology, anaphoric query | imp=0.85, sem=0.40 fact about XYZCorp/ACMEHoldings | 10 |
| recency_collision_old_high_importance | Recency too aggressive — high-importance OLD fact loses to medium-importance recent facts | imp=0.85, rec=0.10 fact ("Petrova maiden name") | 10 |
| source_weight_bury_incorrectly | source_weight asymmetry — calendar (sw=0.6) loses +0.4 free per item to memory_facts (sw=1.0) | calendar event for holiday-weekend picnic | 10 |

---

## Iterations

| # | Adjustment | Pre fix-count (anth/oai/local) | Post fix-count | Outcome | Rationale |
|---|---|---|---|---|---|
| 1 | weight_importance: 0.2 → 1.0 + weight_recency: 0.3 → 0.15 | 7/7/7 | 11/11/11 | **LOCK** | All four FAIL pathologies addressed in one targeted change.  See "Why this single change works" below. |

**Stop reason:** primary goal exceeded by margin — target was ≥9/11 on
anthropic + openai and ≥7/11 on local; achieved 11/11 across all three.
Single paid `--recalibrate` run; remaining 4-run budget unused.

**Total API spend on Component 6 tuning:** ~$0.10 (one --recalibrate run).

---

## Why this single change works

Pre-dispatch protocol prescribed iterations table maps each pathology to a
specific knob, but the final analysis showed **all four pathologies share a
common cause:** the default `weight_importance = 0.2` was too low to let
high-importance items overcome the natural per-item dispersion in
`semantic_score` (typical noise items in the v2 fixtures span sem=0.50-0.80
within a single source).

With `w_sem = 1.0, w_imp = 0.2`, the importance dimension contributes at most
`0.2 * 1.0 = 0.2` to the final score, but the semantic dimension contributes
up to `1.0 * 1.0 = 1.0`.  An importance gap of 0.30 (the right-answer-vs-noise
gap in 3 of 4 FAIL cases) translates to only **+0.06** advantage on final
score, while a semantic gap of 0.40 (typical right-answer disadvantage when
the LLM-relevant fact has dry literal phrasing while noise has prose-rich
phrasing) translates to **+0.40** disadvantage.  Importance was effectively
suppressed.

Lifting `w_imp` to 1.0 makes importance and semantic equal-weighted.  The
0.30 importance gap now contributes **+0.30** to the right answer, more than
enough to overcome the 0.40 semantic gap, especially when paired with a
recency drop that prevents medium-importance recent items from
crowding the top.

Dropping `w_rec` from 0.3 to 0.15 specifically addresses the
`recency_collision_old_high_importance` case, where the right answer is
correctly identified as high-importance but old (rec=0.10), and the noise
items are medium-importance but recent (rec=0.55-0.70).  Halving `w_rec`
shrinks the recency advantage of the noise from `0.3 * 0.50 = 0.15` to
`0.15 * 0.50 = 0.075` — small enough that the importance gap dominates.

The fourth pathology (`source_weight_bury_incorrectly`, calendar burial)
also resolves under the same change: the calendar event's `imp=0.80`
generates `1.0 * 0.80 = 0.80` importance contribution under the new
weights, vs `0.2 * 0.80 = 0.16` under defaults.  This +0.64 boost on the
calendar event is more than enough to overcome the `(1.0 - 0.6) * 1.0 = 0.4`
source-weight-handicap, even before any source_weight rebalancing.

---

## Verification before paid recalibration

The targeted importance-gap check from the protocol was performed via a
non-LLM rank-probe helper that bumps `top_k` to 32 and reports the rank of
the expected-substring answer for every case under any candidate config.
At `w_imp = 1.0, w_rec = 0.15`:

| Case | HEAD rank | Candidate rank | In top_8? |
|---|---|---|---|
| family_kids_competing_partial_matches | 11 | 6 | YES |
| stock_followup_off_topic_v2 | 10 | 3 | YES |
| recency_collision_old_high_importance | 10 | 1 | YES |
| source_weight_bury_incorrectly | 10 | 3 | YES |
| temporal_recent_work_v2 | 5 | 1 | YES |
| entity_graph_traversal_v2 | 5 | 2 | YES |
| ambiguous_category_v2 | 5 | 3 | YES |
| source_weight_bury_correctly | 6 | 7 | YES |
| document_chunk_priority_v2 | 5 | 2 | YES |
| calendar_today_relevance_v2 | 2 | 1 | YES |
| negative_empty_unrelated | NEG | NEG | YES (no chess-related items surface) |

Zero regressions on the six designed-PASS cases; all four FAIL cases lift
into top_k=8.  The `source_weight_bury_correctly` case is the only one
that ranks slightly lower (6 → 7) under the new weights but still
comfortably inside top_8.

This pre-paid verification gave high confidence that the candidate
config would clear the ≥9/11 threshold on every provider, and the paid
run confirmed 11/11 across the board.

---

## Files updated

1. `src/config/config_defaults.c:320-322` — production daemon default
   weights (added Phase 1j comment block).
2. `dawn.toml.example:483-489` — example config rebalanced + Phase 1j
   note.
3. `benchmarks/bench_focus_fix_rate.py:80-95` — `HEAD_BASELINE_FIX_COUNT`
   updated to {anthropic: 11, openai: 11, local: 11} with recalibration
   note.
4. `benchmarks/bench_focus_fix_rate.py:93-95` — `DEFAULT_FOCUS_CONFIG`
   mirrors the new daemon defaults (the variable's role is to mirror
   `dawn.toml` defaults, not host tuning state).

The headline change is two lines:

```diff
-   config->memory.focus_injection.weight_recency = 0.3f;
-   config->memory.focus_injection.weight_importance = 0.2f;
+   config->memory.focus_injection.weight_recency = 0.15f;
+   config->memory.focus_injection.weight_importance = 1.0f;
```
