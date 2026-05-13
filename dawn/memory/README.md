# DAWN Memory Subsystem — Atlas

Historical design documentation for the persistent memory subsystem in [DAWN](https://github.com/The-OASIS-Project/dawn). The memory subsystem extracts, stores, retrieves, and reasons over user-specific facts, preferences, entities, and relations across conversations.

This subdirectory groups memory-specific design docs and investigations. Sibling subsystems live elsewhere:
- **RAG** (`document_chunks` corpus, web/uploaded documents): [`../archive/RAG_DESIGN.md`](../archive/RAG_DESIGN.md). Shares `embedding_engine.c` and `iso8601.c` with memory but is otherwise independent.
- **Compaction / context summarization** (`memory_summaries`, conversation-history compaction): currently lives in `dawn/docs/LCM_DESIGN.md` until archived.

## Active code (in dawn repo)

The production source of truth is `dawn/src/memory/` and `dawn/include/memory/`. Architectural overview: `dawn/docs/arch/subsystems/memory.md`. Operational benchmark harness: `dawn/benchmarks/run_benchmark.py` + `dawn/benchmarks/bench_memory_pipeline.c`.

## Documents in this directory

| Document | Status | Summary |
|---|---|---|
| [STATE.md](STATE.md) | living snapshot (refreshed on each major shipment) | **Read this first.** Current state of the memory subsystem: recently shipped + benchmark position + short/medium/long-term workstreams in priority order. Synthesis layer over the design docs in this directory and `dawn/docs/TODO.md`. |
| [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) | shipped (Phases 1–6.15 + S4–S8) | Master design: entity graph, relations, facts, semantic embeddings, hybrid keyword + semantic search, contacts, soft + hard entity merge, retrieval benchmarking, summary semantic search, recovery worker, provenance, dynamic context injection, recategorization. Schema v46. The reference architecture for everything else in this directory. |
| [ENTITY_MERGE_DESIGN.md](ENTITY_MERGE_DESIGN.md) | shipped (May 2026, Phases 1 + 1.5 + 2, schema v43–v44) | Equivalence-class soft-merge surface resolving the user-identity duplicate problem. Six-stage resolver cascade + composite scorer + dual-band routing (auto 0.90 / review 0.50) + Phase 2 gate firing at extraction. Auto-promote `user_self` against `users.real_name`. Reversible alias_link / alias_unlink. WebUI Suggested-Merges panel + memory-icon dot indicator. Document captures the design from 2026-05-09 + the fold-ins through 2026-05-13 (longer-canonical swap, Bundle 2 equivalence-class read aggregation). |
| [INJECTION_FILTER.md](INJECTION_FILTER.md) | shipped (April 2026) | Shared blocklist module with Unicode normalization (homoglyphs, accents, fullwidth, invisible chars). ~118 patterns across 17 categories. Data-marking framing in extraction prompts. All-path coverage: LLM tool, extraction, WebUI import. 137 unit tests. |
| [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md) | Phase 1 shipped (May 2026); Phase 2 re-scoped | Cat-2 temporal extraction collapse on LoCoMo. Failure-mode taxonomy (A1/A2/B/C/D/E) and 10 case studies localized the gap to extraction-time date loss, not retrieval. **Phase 1 (anchor injection) lifted cat-2 `recall_generation` 0.022 → 0.321 (+29.9pp)** and overall LoCoMo by +7.1pp. Phase 2 (`event_when` field) re-scoped after Phase 1 exceeded the original L1+L2 mid-projection. Cat-1 regression (−3.9pp) tracked separately in `dawn/docs/TODO.md`. |
| [CAT2_TEMPORAL_INVESTIGATION_PLAN.md](CAT2_TEMPORAL_INVESTIGATION_PLAN.md) | procedure (companion to CAT2_TEMPORAL) | The diagnostic procedure that produced CAT2_TEMPORAL.md: how to identify reach=1, gen=0 cases on existing snapshots, pull dialog + extracted facts + relations, label A-E. Reusable for similar future failure-mode investigations. |
| [EMBEDDING_UPGRADE.md](EMBEDDING_UPGRADE.md) | shipped (May 2026, schema v41) | bge-small-en-v1.5-int8 model swap + tech-debt cache-invalidation fix + ID-based extraction filter + per-user embedding recompute worker. Lifted LoCoMo overall +7.9pp, LongMemEval R@5 +1.4pp. Live: 2,183 embeddings re-indexed across 3 users + 243 document chunks. CI grew 38 → 40. The cross-encoder reranker (originally Feature 2 of this plan) was investigated and reverted — see RERANKER_INVESTIGATION.md. |
| [DYNAMIC_CONTEXT_INJECTION.md](DYNAMIC_CONTEXT_INJECTION.md) | Phase 1 shipped (May 2026, closed 2026-05-09); Phase 0 + Phase 2 future | Per-turn focus block ranks facts / entities / relations / summaries / document chunks / calendar events and prepends them to the LLM prompt via the single prompt-builder hook. Source adapter framework with feature-vector contract + framework-owned privacy enforcement. Header-only split of `memory_db.h` shipped as Phase 1a prerequisite. Memory-tool double-dip mitigation via prompt nudge. Phase 0 (silent-observe primitive) + Phase 2 (DAWN background context) sketched but deferred. |
| [PHASE_1J_TUNING_LOG.md](PHASE_1J_TUNING_LOG.md) | closed (May 2026) | Focus-injection weight tuning log. Single-iteration change (`weight_importance` 0.2 → 1.0, `weight_recency` 0.3 → 0.15) produced 11/11 fix-rate across all three providers (anthropic / openai / local) on the Component 6 fix-rate probe. Production defaults shipped at `src/config/config_defaults.c:320-322`. ~$0.10 paid spend. |
| [PROVENANCE.md](PROVENANCE.md) | shipped (Phase A + Phase B, 2026-05-06) | As-shipped record for memory provenance: `(conv_id, msg_id_start, msg_id_end)` triples on facts / relations / summaries / preferences (schema v40 + v42); `with_source=true` on the memory tool returns verbatim conversation excerpts. Phase A bench-validated +12.51pp overall `recall_generation` lift at `source_budget=12288` on LoCoMo (12288 is the production sweet spot). Phase B extended coverage from facts-only to all four record types with `memory_source_dedup_set_t` dedup. |
| [PROVENANCE_VALIDATION.md](PROVENANCE_VALIDATION.md) | shipped (commits `e1a34de` + `4652891`) | The agent-reviewed plan that drove the provenance Phase A + Phase B implementation. Captures the architectural fold-ins (parameter-not-global `source_budget`, defense-in-depth privacy hop, batch IN-clause cap with `static_assert`, dedup-post-batch-filter privacy guarantee, separate-binary bench `#error` guard) that hardened the shipped surface. Companion to PROVENANCE.md (as-shipped record). |
| [RERANKER_INVESTIGATION.md](RERANKER_INVESTIGATION.md) | implemented then reverted (2026) | Cross-encoder reranker investigation: built ms-marco-MiniLM-L-6-v2 int8 ONNX with CUDA EP, integration across memory + RAG retrieval paths, 5 config keys. Reverted after empirical results and literature review showed no net benefit on conversational data and only marginal lift on LongMemEval at 10× latency. Kept artifacts: shared WordPiece tokenizer (`memory_embed_tokenizer`), `rerank_shootout.py` test harness. |
| [LOCOMO_CAT3_PROFILING.md](LOCOMO_CAT3_PROFILING.md) | shipped (May 2026) | LoCoMo cat-3 (multi-hop inference) failure-mode profiling. Session-neighbor boost (Tier 2 quick win — +3.0pp dialog overall, +20.0pp cat-3 in dialog granularity). Memory-pipeline bench mode (Tier 1, Phases 0/1/1.5) — end-to-end LoCoMo evaluation against extracted memory at production parity, `recall_reach` metric. Haiku 4.5 result: 0.742 overall / 0.646 cat-3. Frames retrieval vs answer-support distinction. |
| [COMPETITOR_LANDSCAPE.md](COMPETITOR_LANDSCAPE.md) | reference (last verified 2026-05-05) | Frozen snapshot of published numbers and methodology observations from competing memory-retrieval systems (Hindsight, ByteRover, MemMachine, MemPalace, Letta, Mem0, SelRoute, RMM, etc.) on LongMemEval, LoCoMo, and ConvoMem. Includes the MemPalace methodology audit and the retrieval-vs-end-to-end-QA distinction. **No DAWN numbers** — pull from this when writing publication material; DAWN measurements rot every shipment, competitor numbers don't. |

## In-flight (working docs in `dawn/docs/`)

Two design docs for unstarted memory-adjacent workstreams currently live under `dawn/docs/` per the design-doc-commit policy (planned-but-unstarted stays untracked):

- **`dawn/docs/BENCH_PARALLEL_SWEEP_DESIGN.md`** — Model-level parallelism for `--sweep-extraction-models`. Cache subdirs + retry/backoff + ThreadPoolExecutor + per-thread logs. ~3 hr agent effort; pays off after next 4-model sweep.
- **`dawn/docs/SPEAKER_IDENTIFICATION_PLAN.md`** — sherpa-onnx integration for per-user voice identification. Research complete; flagged as memory-system prerequisite for multi-user households (Phase 12 in SYSTEM_DESIGN.md).

## Active TODO

Active TODO items related to the memory subsystem live in `dawn/docs/TODO.md`. As of 2026-05-13:

- **Phase 1j re-bench with summary-relevant probe** (memory) — resolve the `weight_recency 0.15 vs 0.3` tension between Phase 1j bench evidence and live observation. Add summary-relevant probe cases first.
- **Cat-1 regression from anchor injection** (Phase 1 follow-up — investigate before cat-2 Phase 2 starts).
- **Cat-2 Phase 2: structured `event_when` field on facts** (re-scope first; original projection met by Phase 1 alone).
- **Cross-encoder reranker on focus-injection candidates** (Feature 2 from EMBEDDING_UPGRADE — addresses cosine over-inclusion on dominant tokens).
- **Dynamic Context Injection Phase 0** (silent-observe primitive) and **Phase 2** (DAWN background context) — see DYNAMIC_CONTEXT_INJECTION.md §"Phase 0" and §"Phase 2".
- **"I don't remember" gate** — explicit threshold-floor rejection in the memory tool when retrieval scores all fall below a configured floor. Partially addressed by Step 3 double-dip mitigation; remaining gap is the floor itself.
- **Speaker mis-attribution** — extraction prompt biases toward conversation's first speaker; assistant-turn events get mis-attributed. Correctness > everything else on the memory list. Note 2026-05-08: Attempt #2 closed without full-bench lift; deferred.
- **Memory injection filter: multi-language** (Korean / Japanese / Chinese pattern coverage; deferred Phase 4 from INJECTION_FILTER.md).
- **Memory injection filter: weighted scoring** (per-pattern weights, prompt-armor-style; deferred until binary blocking proves insufficient).
- **Memory injection filter: pre-filter legacy data** (one-time migration scan; low priority since filter now gates all live ingestion paths).

## Conventions

- Filenames use the subsystem-relative form (drop the `MEMORY_` prefix that was redundant under the flat archive).
- Each doc opens with a status line (shipped / in-flight / reverted / planned) and the date(s) of major revisions.
- Cross-references between sibling docs in this directory use bare filenames; cross-references to the rest of atlas use `../archive/...`.
- When a planning doc in `dawn/docs/` describes shipped code, it migrates here.
