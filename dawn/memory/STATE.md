# Memory Subsystem — Current State

**Purpose:** living snapshot of where the memory subsystem stands. Read this at the start of any memory-focused work session before diving into individual atlas docs or `dawn/docs/TODO.md`. Refresh when a major shipment lands.

**Last updated:** 2026-05-06 (after Cat-2 Temporal Phase 1 + Phase 1.1 subject-naming fix + bench cache-invalidate fix + production cache-invalidate fold-in + Phase 2 Option B probe / closure + Provenance Phase A bench validation + budget sweep + strbuf truncation fix across 6 hazards).

---

## At a glance

The memory subsystem is in good shape. LongMemEval (97.0% R@5) and ConvoMem (99.0%) are essentially solved. LoCoMo `recall_generation` lifted twice this week: Phase 1 (anchor injection) 0.208 → 0.279 (+7.1pp), Phase 1.1 (subject-naming prompt fix) 0.279 → **0.371** (+9.2pp on top, +16.3pp total). Cat-1 single-hop also recovered through Phase 1.1 — went from baseline 0.209 → Phase 1 regression 0.170 → **0.316** (+10.7pp over baseline). The remaining benchmark alpha is on **cat-3 multi-hop inference** — graph-based competitors get +20-35pp on retrieval here and closing it likely requires architectural work, not parameter tuning. Cross-encoder reranking is **dead** (built, integrated, reverted — see RERANKER_INVESTIGATION.md).

Short-term focus: micro-bench promotion + legacy-data filter scan is what's left of Phase 1 consolidation (~agent 1 hr · api ~$2 · 2 ckpt). After that, the medium-term roadmap is provenance + injection-filter multi-language + speaker mis-attribution.

---

## Recently shipped (last ~6 weeks)

| Shipment | Schema | Headline result | Atlas doc |
|---|---|---|---|
| **bge-small-int8 + recompute worker + ID-based extraction filter** (May 2026) | v41 | LoCoMo overall +7.9pp (73.7→81.6), LongMemEval R@5 +1.4pp (95.6→97.0), cat-3 +8.6pp. Live recompute of 2,183 embeddings + 243 chunks. CI 38→40. | [EMBEDDING_UPGRADE.md](EMBEDDING_UPGRADE.md) |
| **Cross-encoder reranker** (May 2026) | — | Built ms-marco-MiniLM-L-6-v2 int8 ONNX with CUDA EP, full integration, 5 config keys. **Reverted** — no net benefit on conversational data, marginal lift on LongMemEval at ~10× retrieval latency. Kept artifacts: WordPiece tokenizer, `rerank_shootout.py` test harness. Don't revisit unless new evidence. | [RERANKER_INVESTIGATION.md](RERANKER_INVESTIGATION.md) |
| **Cat-2 temporal Phase 1: conversation anchor injection** (May 2026) | v42 | Cat-2 `recall_generation` 0.022 → 0.321 (+29.9pp), overall +7.1pp, cat-3 +4.4pp & cat-4 +5.0pp bonus. Cat-1 regressed −3.9pp — root-caused to subject-name elision in Phase 1.1 below. | [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md) |
| **Cat-2 temporal Phase 1.1: subject-naming prompt fix + bench/prod cache-invalidate** (May 2026) | — | Tightened the Phase 1 "preserve original phrase" rule to event-bearing facts only AND added an explicit subject-naming requirement. Lifted overall `recall_generation` 0.279 → **0.371** (+9.2pp on top of Phase 1, +16.3pp over baseline); cat-1 recovered to **0.316** (+10.7pp over baseline); cat-2 lifted further to **0.492** (+17.1pp on top of Phase 1); cat-3/4 also +5.4 / +10.7pp. Validation uncovered a stale-cache bug in `bench_memory_pipeline.c` (snapshot_load + reset_memory weren't invalidating the in-memory embedding cache, causing cache-HIT replays to give false-zero recall on every conv past the first); fix folded in as `memory_embeddings_invalidate_all()` helper. Same gap existed in production `memory_db_delete_user_memories` (WebUI "delete my memories" path) — also fixed. | [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md) Phase 1.1 |
| **Cat-2 temporal Phase 2 Option B probe — closed, no ship** (May 2026) | — | Tested L4 (event-coverage prompt rev) alone before committing to L2's schema work. Cat-2 lifted +2.8pp (0.492 → 0.520) but cat-1 regressed −2.9pp (0.316 → 0.287) due to top-K crowding; overall +0.3pp = wash. Reverted prompt change. **Phase 2 closed**: full L2+L3+L4 spec preserved as frozen reference but not pursued — C wouldn't fix the cat-1 crowding mechanism (its boost is query-temporal-conditional; cat-1 has no temporal expression to trigger it) and the residual cat-2 gap is mostly the dataset's structural ceiling. Re-open if `max_facts` ever raises or query-aware top-K filtering is added. | [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md) Phase 2 Option B |
| **Provenance Phase A bench validation + budget sweep + strbuf truncation fix** (May 2026) | — | Wired production-faithful `query_memory_callback` bench command (calls real `memory_action_search` path); added `source_budget` parameter to action functions instead of mutating a global; ran 5-point sweep (budget = 0, 1024, 3072, 6144, 12288) on all 10 LoCoMo conversations. **Overall `recall_generation` lifts +12.51pp at budget=12288 (0.368 → 0.493); +9.54pp at 6144; +7.67pp at 3072 (current default).** Cat-4 (knowledge updates) is the dominant beneficiary — +22.9pp at 12288 (0.484 → 0.713). **Cat-3 finally lifted at 12288 (0.283 → 0.348, +6.5pp)** after staying flat at 0/1024/3072/6144 — overturning earlier "cat-3 is provenance-immune" claim. Multi-hop questions ARE budget-bound; below ~12K chars no benefit, above it real lift. Marginal 6144→12288 = +2.98pp (re-accelerated, not diminishing) — suggests further headroom unsampled (24576 budget worth a future probe). Reach + entailment unchanged at 0.834 / 0.420 across all 5 budgets (sanity check: pure generator-side lift). **Mid-sweep discovery: production memory_action_search was using a fixed 8KB `result` buffer with `offset += snprintf` accumulation, silently truncating responses at higher source budgets** — that's why 6144 dropped vs 3072 on some convs in the original Pass 1B. Built a growable `strbuf_t` utility (`include/core/strbuf.h` + `src/core/strbuf.c`, doubling realloc, 256 KiB max_cap, sticky OOM, 18 unit tests) and refactored memory_callback + memory_context to use it. A code scout surfaced 5 more truncation hazards in the same shape — all folded in (`webui_memory.c` export, `webui_music_handlers.c` queue, `tools/scheduler_tool.c` events, `tools/calendar_tool.c` 5 handlers, `memory_context.c` system-prompt assembly was the highest-leverage fix since it bites every session not every tool call). Three agent reviews (architecture, security, coding-standards) post-implementation found 0 Critical / 4 High / 10 Medium / 9 Low — all HIGHs and budget-cluster MEDIUMs folded in same commit. Production now reads `[memory] source_budget_chars` from dawn.toml (default 3072 unchanged; deployments tune up for higher answer quality at the cost of more LLM input tokens). Phase B (coverage extension to relations/summaries/preferences + new `memory_db_provenance.c` module) GREEN-LIT, deferred to next session. | [PROVENANCE.md](PROVENANCE.md) |
| **Memory injection filter** (April 2026) | — | `memory_filter.c/h` Layer 2 module, ~118 patterns / 17 categories, Unicode normalization (homoglyphs / Latin-1 / fullwidth / invisible chars / UTF-8 validation), data-marking framing in system prompt, 137 unit tests. All ingestion paths gated. | [INJECTION_FILTER.md](INJECTION_FILTER.md) |
| **LoCoMo cat-3 profiling + session-neighbor boost** (May 2026) | — | Identified that retrieval-vs-answer-support framing matters; bench-only +3.0pp dialog overall / +20.0pp cat-3 boost. Production retrieval unchanged. Documented bench harness measures `document_chunks` not `memory_facts` — motivated the memory-pipeline bench mode. | [LOCOMO_CAT3_PROFILING.md](LOCOMO_CAT3_PROFILING.md) |

Plus subsystem hygiene this session: docs reorganized into this directory (was flat under `atlas/dawn/archive/`), stale plan docs deleted from `dawn/docs/`, COMPETITOR_LANDSCAPE.md extracted as a frozen reference for publication material.

For the full chronological shipped list (memory + everything else) see `dawn/docs/TODO.md` Shipped section.

---

## Benchmark position

Two distinct metrics — keep them separate, both matter:

### Retrieval reach (R@K / `recall_reach`)

"Does retrieval surface the right evidence?" — comparable to most published competitor numbers.

| Metric | DAWN | Position |
|---|---|---|
| LongMemEval R@5 | **97.0%** | Leads or matches all published systems. MemPalace's 96.6% is methodology-critiqued. |
| LoCoMo overall | **81.6%** | Solid mid-tier. **8-15pp behind** ByteRover (92.2%, cross-encoder), MemMachine (91.69%, graph + cross-encoder), Hindsight (89.61%). Above Letta 74.0%, Mem0 68.5%. Below human ceiling 87.9. |
| ConvoMem avg | **99.0%** | Ceiling-level. No higher published. |
| LoCoMo cat-2 (temporal reach) | 84.9% | Competitive with Hindsight 83.8%. April 2026 temporal-query parser is paying. |
| LoCoMo cat-3 (multi-hop) | 64.4% | **The gap.** Graph competitors exceed by 20-35pp. |

Numbers reflect post-bge-small-swap baseline (May 2026). For competitor citations and methodology caveats see [COMPETITOR_LANDSCAPE.md](COMPETITOR_LANDSCAPE.md). For DAWN's per-benchmark detail tables see [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) §14.3.

### Generation accuracy (`recall_generation`)

"Does the LLM produce the right answer given retrieved facts?" — production-faithful, not directly comparable to retrieval-only competitor numbers.

| Category | Pre-Phase-1 | Phase 1 | **Phase 1.1 (current)** |
|---|---|---|---|
| Overall | 0.208 | 0.279 | **0.371** |
| Cat-1 (single-hop) | 0.209 | 0.170 (regressed) | **0.316** (recovered + lifted) |
| Cat-2 (temporal) | 0.022 | 0.321 | **0.492** |
| Cat-3 (multi-hop) | 0.228 | 0.272 | **0.326** |
| Cat-4 (knowledge update) | 0.316 | 0.366 | **0.473** |
| Cat-5 (adversarial) | 0.135 | 0.155 | 0.135 (within noise) |

**Cat-2 has a structural ceiling around 0.60** — ~22% of LoCoMo cat-2 questions have non-date gold answers (mis-categorized as cat-2 in the dataset) that no temporal fix can address. Phase 1.1's 0.492 puts the system at ~82% of that ceiling.

---

## Short-term workstream

Phase 1 / 1.1 / 2 cycle is closed. Provenance Phase A is closed (validation + sweep + strbuf truncation fix all done). Remaining short-term items:

| # | Item | Effort | Why this order |
|---|---|---|---|
| 1 | ~~**Cat-1 regression investigation**~~ | DONE (Phase 1.1) | Root cause was subject-name elision; recovered cat-1 to +10.7pp over original baseline. |
| 2 | ~~**Phase 2 re-scope decision**~~ | DONE (Option B probe) | Tested L4 prompt-only rev: +2.8pp cat-2 / −2.9pp cat-1 / +0.3pp overall. Reverted; Phase 2 closed. |
| 3 | ~~**Provenance Phase A: validation + budget sweep + strbuf fix**~~ | DONE | Final overall +12.51pp at budget=12288 (0.368 → 0.493); cat-3 finally lifted +6.5pp at 12288 (was thought provenance-immune); 6 truncation hazards fixed via new strbuf utility. Phase B green-lit. |
| 4 | **Provenance Phase B: coverage extension + dedup + new module** | agent ~3-4 hr · api ~$8 · 2 ckpt | New `memory_db_provenance.c` module with batch readers for relations/summaries/preferences mirroring `_facts_get_sources` (with N-cap fail-closed fix); `source_dedup_set_t` struct + helpers in memory_callback; defense-in-depth privacy hop in `conv_db_get_messages_by_range`; ~12 new unit tests + 2 integration tests. Plan: `atlas/dawn/memory/PROVENANCE.md` — already review-cleared by 3 agents this cycle. **Production default already exposed via `[memory] source_budget_chars` so deployments can set 12288 without waiting on B.** |
| 5 | **Probe budget=24576** (Phase A follow-up) | agent ~1 hr · api ~$8 · 1 ckpt | The 6144→12288 marginal lift (+2.98pp) was non-diminishing — actually re-accelerated vs 3072→6144 (+1.87pp). Suggests headroom past 12288; cat-3 in particular jumped only at 12288 and may keep climbing. Stop condition: <0.5pp marginal lift vs 12288. Cheap to settle the operational sweet spot. |
| 6 | **Promote `/tmp/l1_microbench.py` → `benchmarks/bench_temporal_arithmetic.py`** | agent ~30m · 1 ckpt | Permanent guardrail against future extraction-prompt regressions on LLM date arithmetic. Cheap insurance. |
| 7 | **Memory injection filter: pre-filter legacy data** | agent ~30m · 1 ckpt | One-time scan of facts/entities/summaries stored before April 2026, run each through `memory_filter_check()`. Closes the only known filter coverage gap. Low priority but easy. |

**For the next session: load `atlas/dawn/memory/STATE.md` + `atlas/dawn/memory/PROVENANCE.md` and pick up Phase B implementation.** The 5 sweep result JSONs at `/tmp/prov_pass{1a,1b,2a,2b,3_strbuf}_*.json` capture the validated numbers if needed.

---

## Medium-term workstreams (active TODO items)

Priority-ordered by leverage × tractability:

| Item | Effort | Notes |
|---|---|---|
| **Memory injection filter: multi-language** | ~2 days | Korean / Japanese / Chinese injection patterns currently bypass the normalizer. prompt-guard `high.yaml` has ready-to-adopt patterns. Independent of Phase 2. |
| **Memory provenance / source-linked recall** | ~3 days | Schema v40 added the columns; extraction has the IDs but doesn't record them. `recall` tool gains `with_source` for verbatim-snippet-alongside-fact. Likely +X on cat-3 inference; broadly improves verifiability. Composes with the future MCP server. |
| **Phase 2 (`event_when` field)** | 2-3 days, **gated on item #2 in short-term** | Re-scoped per Phase 1 result. Spec settled in CAT2_TEMPORAL.md if pursued. |
| **Speaker mis-attribution** | ~2-3 days | Cat-2 diagnosis surfaced this as a separate workstream. Extraction prompt biases toward conversation's first speaker; assistant-turn events get mis-attributed (Case 10: Jolene's pendant became Deborah's). Orthogonal to temporal work. |
| **Memory injection filter: weighted scoring** | ~3 days | Per-pattern weights, prompt-armor approach. Deferred until binary blocking proves insufficient. Not active yet. |
| **LLM-based contradiction judgment** | ~2 days | Inline LLM call for contradictions the entity-graph can't catch (quantity changes, subtle negation). Gated on cost/latency becoming practical for per-fact calls. |

---

## Long-term direction (architectural / strategic)

The biggest remaining benchmark gap is **LoCoMo cat-3 multi-hop inference (64.4%)**. Graph-based competitors get +20-35pp here. Closing this likely requires:

1. **Relation-aware retrieval** — query → entity match → graph traversal, then score the connected sub-graph. The cat-3 profiling analysis showed 100% of misses share an entity with retrieved top-10 — the gap is *ranking within an entity's dialogs*, not crossing between entities. Substantial architectural shift; weeks of work. See LOCOMO_CAT3_PROFILING.md "Outcome" section.

2. **Standalone memory/retrieval library** — extract `embedding_engine` + `memory_db` + `document_db` + similarity into `libdawn_memory.so` + Python bindings. Enables the **DAWN memory MCP server** (see TODO Coding Assistant section) and external benchmarking. Composes with provenance work above. ~1-2 weeks for v1.

3. **Cross-encoder reranking** — **dead.** Don't revisit. Investigation already happened, no net benefit on conversational data.

---

## Maintenance protocol

Update this doc when:

1. A memory-subsystem shipment lands (add row to "Recently shipped," update benchmark numbers if measured).
2. A major workstream completes or re-scopes (move between "Short-term" / "Medium-term" / "Long-term," update rationale).
3. A new long-term direction emerges or an old one is closed off.

Bump the "Last updated" date at the top.

For sources of authority (when the doc and another doc disagree, the other wins):

- **Per-benchmark numbers**: SYSTEM_DESIGN.md §14.3 (DAWN) and COMPETITOR_LANDSCAPE.md (competitors).
- **Active TODO**: `dawn/docs/TODO.md` (gitignored, dev-maintained).
- **Per-feature design**: the individual archive doc (CAT2_TEMPORAL.md, EMBEDDING_UPGRADE.md, etc.).

This doc is the *synthesis*, not the source. Refresh from the sources when something feels stale.
