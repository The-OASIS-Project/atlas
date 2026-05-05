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
| [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) | shipped (Phases 1–6.7 + S4 + 13) | Master design: entity graph, relations, facts, semantic embeddings, hybrid keyword + semantic search, contacts, entity merge, retrieval benchmarking. The reference architecture for everything else in this directory. |
| [INJECTION_FILTER.md](INJECTION_FILTER.md) | shipped (April 2026) | Shared blocklist module with Unicode normalization (homoglyphs, accents, fullwidth, invisible chars). ~118 patterns across 17 categories. Data-marking framing in extraction prompts. All-path coverage: LLM tool, extraction, WebUI import. 137 unit tests. |
| [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md) | Phase 1 shipped (May 2026); Phase 2 re-scoped | Cat-2 temporal extraction collapse on LoCoMo. Failure-mode taxonomy (A1/A2/B/C/D/E) and 10 case studies localized the gap to extraction-time date loss, not retrieval. **Phase 1 (anchor injection) lifted cat-2 `recall_generation` 0.022 → 0.321 (+29.9pp)** and overall LoCoMo by +7.1pp. Phase 2 (`event_when` field) re-scoped after Phase 1 exceeded the original L1+L2 mid-projection. Cat-1 regression (−3.9pp) tracked separately in `dawn/docs/TODO.md`. |
| [CAT2_TEMPORAL_INVESTIGATION_PLAN.md](CAT2_TEMPORAL_INVESTIGATION_PLAN.md) | procedure (companion to CAT2_TEMPORAL) | The diagnostic procedure that produced CAT2_TEMPORAL.md: how to identify reach=1, gen=0 cases on existing snapshots, pull dialog + extracted facts + relations, label A-E. Reusable for similar future failure-mode investigations. |
| [EMBEDDING_UPGRADE.md](EMBEDDING_UPGRADE.md) | shipped (May 2026, schema v41) | bge-small-en-v1.5-int8 model swap + tech-debt cache-invalidation fix + ID-based extraction filter + per-user embedding recompute worker. Lifted LoCoMo overall +7.9pp, LongMemEval R@5 +1.4pp. Live: 2,183 embeddings re-indexed across 3 users + 243 document chunks. CI grew 38 → 40. The cross-encoder reranker (originally Feature 2 of this plan) was investigated and reverted — see RERANKER_INVESTIGATION.md. |
| [RERANKER_INVESTIGATION.md](RERANKER_INVESTIGATION.md) | implemented then reverted (2026) | Cross-encoder reranker investigation: built ms-marco-MiniLM-L-6-v2 int8 ONNX with CUDA EP, integration across memory + RAG retrieval paths, 5 config keys. Reverted after empirical results and literature review showed no net benefit on conversational data and only marginal lift on LongMemEval at 10× latency. Kept artifacts: shared WordPiece tokenizer (`memory_embed_tokenizer`), `rerank_shootout.py` test harness. |
| [LOCOMO_CAT3_PROFILING.md](LOCOMO_CAT3_PROFILING.md) | shipped (May 2026) | LoCoMo cat-3 (multi-hop inference) failure-mode profiling. Session-neighbor boost (Tier 2 quick win — +3.0pp dialog overall, +20.0pp cat-3 in dialog granularity). Memory-pipeline bench mode (Tier 1, Phases 0/1/1.5) — end-to-end LoCoMo evaluation against extracted memory at production parity, `recall_reach` metric. Haiku 4.5 result: 0.742 overall / 0.646 cat-3. Frames retrieval vs answer-support distinction. |
| [COMPETITOR_LANDSCAPE.md](COMPETITOR_LANDSCAPE.md) | reference (last verified 2026-05-05) | Frozen snapshot of published numbers and methodology observations from competing memory-retrieval systems (Hindsight, ByteRover, MemMachine, MemPalace, Letta, Mem0, SelRoute, RMM, etc.) on LongMemEval, LoCoMo, and ConvoMem. Includes the MemPalace methodology audit and the retrieval-vs-end-to-end-QA distinction. **No DAWN numbers** — pull from this when writing publication material; DAWN measurements rot every shipment, competitor numbers don't. |

## In-flight (no working docs in dawn/docs/ today)

Active TODO items related to the memory subsystem live in `dawn/docs/TODO.md`:

- **Cat-1 regression from anchor injection** (Phase 1 follow-up — investigate before Phase 2 starts).
- **Phase 2: structured `event_when` field on facts** (re-scope first; original projection met by Phase 1 alone).
- **Memory injection filter: multi-language** (Korean / Japanese / Chinese pattern coverage; deferred Phase 4 from INJECTION_FILTER.md).
- **Memory injection filter: weighted scoring** (per-pattern weights, prompt-armor-style; deferred until binary blocking proves insufficient).
- **Pre-filter legacy data** (one-time migration scan to retroactively check facts/entities/summaries stored before the filter shipped; low priority).

## Conventions

- Filenames use the subsystem-relative form (drop the `MEMORY_` prefix that was redundant under the flat archive).
- Each doc opens with a status line (shipped / in-flight / reverted / planned) and the date(s) of major revisions.
- Cross-references between sibling docs in this directory use bare filenames; cross-references to the rest of atlas use `../archive/...`.
- When a planning doc in `dawn/docs/` describes shipped code, it migrates here.
