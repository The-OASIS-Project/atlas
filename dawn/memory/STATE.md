# Memory Subsystem — Current State

**Purpose:** living snapshot of where the memory subsystem stands. Read this at the start of any memory-focused work session before diving into individual atlas docs or `dawn/docs/TODO.md`. Refresh when a major shipment lands.

**Last updated:** 2026-05-05 (after Cat-2 Temporal Phase 1 ship + atlas/dawn/memory/ reorg).

---

## At a glance

The memory subsystem is in good shape. LongMemEval (97.0% R@5) and ConvoMem (99.0%) are essentially solved. LoCoMo cat-2 (temporal) just got a structural fix that lifted `recall_generation` from 0.022 → 0.321 (+29.9pp) via conversation anchor injection. The remaining benchmark alpha is on **cat-3 multi-hop inference (64.4%)** — graph-based competitors get +20-35pp here and closing it likely requires architectural work, not parameter tuning. Cross-encoder reranking is **dead** (built, integrated, reverted — see RERANKER_INVESTIGATION.md).

Short-term focus: ~2 days of Phase 1 consolidation (cat-1 regression investigation, Phase 2 re-scope decision, micro-bench promotion, legacy-data filter scan) clears the cycle. After that, the medium-term roadmap is provenance + injection-filter multi-language + speaker mis-attribution.

---

## Recently shipped (last ~6 weeks)

| Shipment | Schema | Headline result | Atlas doc |
|---|---|---|---|
| **bge-small-int8 + recompute worker + ID-based extraction filter** (May 2026) | v41 | LoCoMo overall +7.9pp (73.7→81.6), LongMemEval R@5 +1.4pp (95.6→97.0), cat-3 +8.6pp. Live recompute of 2,183 embeddings + 243 chunks. CI 38→40. | [EMBEDDING_UPGRADE.md](EMBEDDING_UPGRADE.md) |
| **Cross-encoder reranker** (May 2026) | — | Built ms-marco-MiniLM-L-6-v2 int8 ONNX with CUDA EP, full integration, 5 config keys. **Reverted** — no net benefit on conversational data, marginal lift on LongMemEval at ~10× retrieval latency. Kept artifacts: WordPiece tokenizer, `rerank_shootout.py` test harness. Don't revisit unless new evidence. | [RERANKER_INVESTIGATION.md](RERANKER_INVESTIGATION.md) |
| **Cat-2 temporal Phase 1: conversation anchor injection** (May 2026) | v42 | Cat-2 `recall_generation` 0.022 → 0.321 (+29.9pp), overall +7.1pp, cat-3 +4.4pp & cat-4 +5.0pp bonus. Cat-1 regressed −3.9pp (tracked, not yet investigated). | [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md) |
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

| Category | Pre-Phase-1 | Post-Phase-1 |
|---|---|---|
| Overall | 0.208 | **0.279** |
| Cat-1 (single-hop) | 0.209 | 0.170 (regressed −3.9pp) |
| Cat-2 (temporal) | 0.022 | **0.321** |
| Cat-3 (multi-hop) | 0.228 | 0.272 |
| Cat-4 (knowledge update) | 0.316 | 0.366 |
| Cat-5 (adversarial) | 0.135 | 0.155 |

**Cat-2 has a structural ceiling around 0.60** — ~22% of LoCoMo cat-2 questions have non-date gold answers (mis-categorized as cat-2 in the dataset) that no temporal fix can address.

---

## Short-term workstream (Phase 1 consolidation, ~2 days)

In execution order:

| # | Item | Effort | Why this order |
|---|---|---|---|
| 1 | **Cat-1 regression investigation** | ~½ day | Phase 1's anchor line added context that's irrelevant for atemporal facts and cost cat-1 −3.9pp. Need root cause before deciding Phase 2 — is the anchor specifically hurting, or is the prompt too long now, or something else? |
| 2 | **Phase 2 re-scope decision** | ~½ day | Original projection: cat-2 → 0.40-0.55 from L1+L2 combined. Phase 1 alone hit 0.321; remaining gap is "mode C" (events dropped during session condensation) which `event_when` doesn't directly address. Decide: ship event_when anyway, do a smaller prompt-only event-coverage rev (folded L4), or move on to higher-leverage work. Settled spec for L2 lives in CAT2_TEMPORAL.md if pursued. |
| 3 | **Promote `/tmp/l1_microbench.py` → `benchmarks/bench_temporal_arithmetic.py`** | ~½ day | Permanent guardrail against future extraction-prompt regressions on LLM date arithmetic. Cheap insurance. |
| 4 | **Memory injection filter: pre-filter legacy data** | ~½ day | One-time scan of facts/entities/summaries stored before April 2026, run each through `memory_filter_check()`. Closes the only known filter coverage gap. Low priority but easy. |

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
