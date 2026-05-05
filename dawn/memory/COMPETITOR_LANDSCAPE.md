# Memory Retrieval Competitor Landscape

**Status:** research notes. Frozen snapshot of published numbers and methodology observations from competing memory-retrieval systems on the three major benchmarks (LongMemEval, LoCoMo, ConvoMem). Pull from this doc when writing fresh DAWN positioning material; refresh competitor entries when new papers/posts publish.

**Last verified:** 2026-05-05.

This file intentionally contains **no DAWN-side numbers**. DAWN's measurements rot every memory shipment; competitor numbers don't. Keep them separate.

---

## Benchmark context

### LongMemEval

500 manually curated questions embedded in scalable chat histories. Tests information extraction, multi-session reasoning, temporal reasoning, knowledge updates, and abstention. Official metric: **R@K** — does the correct session/document appear in the top-K retrieved? Typically reported at K=5.

Sub-variants: LongMemEval-S (small), LongMemEval-M (medium). Different sizes have different difficulty curves and can confound cross-system comparison.

### LoCoMo

300+ turns per conversation, up to 35 sessions. Evaluates single-hop fact recall, temporal reasoning, multi-hop reasoning, and open-domain synthesis. Official metric is **F1**; recent systems report accuracy percentages. Five categories:

- cat-1: profile facts (single-hop)
- cat-2: temporal reasoning
- cat-3: multi-hop inference
- cat-4: knowledge update
- cat-5: adversarial / abstention

Granularity matters: dialog-level vs session-level retrieval produce materially different numbers on the same harness. See `LOCOMO_CAT3_PROFILING.md` for the granularity audit (sibling doc).

### ConvoMem

75,336 QA pairs across user facts, assistant facts, abstention, preferences, temporal changes, and implicit connections. 60% of test cases scatter evidence across 2-6 messages, requiring synthesis of distributed information. Conversations <150 turns. Official metric: average recall.

---

## Competitor comparison table

Numbers are what each system publishes — see Sources for primary references. Empty cells mean the system did not report that benchmark.

| System | LongMemEval R@5 | LoCoMo overall | ConvoMem avg | Notes |
|---|---|---|---|---|
| **Official baselines (papers)** | | | | |
| BM25 only | 93.8% | — | — | LongMemEval-S baseline (Wang et al. 2024) |
| Dense only (Contriever) | 72.3% | — | — | LongMemEval-M baseline |
| Dense + fact keys (Contriever) | 76.2% | — | — | LongMemEval-M baseline |
| BM25 + dense hybrid (MiniLM) | 95.2% | — | — | LongMemEval R@5 |
| GPT-4 (no retrieval) | — | 32.1 F1 | — | LoCoMo paper baseline |
| Full-context | — | — | 70-82% | ConvoMem baseline |
| Mem0 (RAG, <150 conv.) | — | — | 30-45% | ConvoMem |
| Human ceiling | — | 87.9 F1 | — | LoCoMo ground-truth ceiling |
| Long-context LLM (no memory) | — | 22-66pp Δ | — | Improvement over GPT-4 baseline (+32.1pp) |
| | | | | |
| **Academic systems** | | | | |
| SelRoute (query-aware routing) | 80.0% | — | — | LongMemEval-M |
| RMM (Reflective Memory Mgmt) | ~85-90% (est.) | — | — | "5%+ over baseline" — exact number not published |
| Observational Memory (Mastra) | — | — | — | Reports 94.87% **end-to-end QA**, not retrieval — not directly comparable |
| | | | | |
| **Commercial / OSS systems** | | | | |
| MemPalace | 96.6% | ~88.9% | — | **Methodology critiqued** — see "Methodology controversies" below |
| Hindsight (Gemini-3 + TEMPR) | 91.4% | 89.61% | — | Entity + temporal aware |
| ByteRover 2.0 | — | 92.2% | — | All-MiniLM embeddings + reranking |
| MemMachine v0.2 | — | 91.69% | — | Directed graph memory + cross-encoder |
| Mem0 (core) | 49.0% | 68.5% | — | Baseline retrieval or graph variant |
| Letta (MemGPT) | — | 74.0% | — | Simple filesystem + file search |
| SuperLocalMemory Mode C | — | 87.7% | — | Zero-LLM-call design |

### Hindsight cat-2 detail

Hindsight publishes per-category LoCoMo numbers; cat-2 (temporal): **83.8%**. Useful comparison point for any system claiming temporal-reasoning improvements.

### ByteRover cat-3 detail

ByteRover's published cat-3 (multi-hop inference): **85.1%**. Useful comparison for multi-hop inference work.

---

## Methodology controversies and caveats

### MemPalace audit (Vectorize, 2024)

The published 96.6% LongMemEval R@5 uses ChromaDB's built-in `all-MiniLM-L6-v2` embedding retrieval **without any MemPalace-specific architecture** — making it a pure embedding baseline rather than a memory-system benchmark. The "100%" sometimes claimed is a hand-coded patch on three specific questions.

Per the Vectorize audit, the honest MemPalace number on full LoCoMo at top_k=10 is approximately **88.9%**, not the ~100% sometimes claimed.

GitHub issues #29 and #214 on the MemPalace repo independently note the methodology divergence from the published QA leaderboard.

**When citing MemPalace:** specify which of the three numbers (96.6% R@5 / ~100% patched / 88.9% audited) is being referenced and the methodology assumed.

### "Retrieval" vs "end-to-end QA" — different metrics

Some systems publish **retrieval R@K** (did we find the right evidence in top-K?). Others publish **end-to-end QA accuracy** (did we generate a correct answer?).

- LongMemEval and ConvoMem are typically reported as retrieval metrics.
- LoCoMo papers vary; original is F1 on generated answers, recent systems report accuracy.
- Observational Memory's 94.87% is end-to-end QA on GPT-4o-mini — this is **not directly comparable** to retrieval R@5 numbers.

These metrics are complementary but not interchangeable. Cite the metric explicitly.

### LoCoMo granularity (dialog vs session)

The LoCoMo benchmark can be run at dialog granularity (one doc per turn) or session granularity (one doc per session). Same harness, materially different numbers. Most published systems do not specify granularity. See sibling `LOCOMO_CAT3_PROFILING.md` for the audit and the resulting clarification of which baselines are session-level.

### ConvoMem conversation-length scaling

ConvoMem explicitly notes simple full-context approaches achieve 70-82% on histories <150 conversations. Mem0 systems achieve 30-45% on that same regime, then RAG approaches dominate beyond ~150 conversations.

ConvoMem numbers are **not directly comparable to LoCoMo** because LoCoMo's 300-2000+ turn conversations are much longer.

### LongMemEval author observation (Wang et al. 2024)

> "Expanded keys with extracted user facts greatly facilitate both memory recall (4% higher) and downstream QA (5% higher)."

Useful framing for any system claiming temporal/proper-noun/named-entity boosts as incremental improvements over vanilla semantic search.

### Cross-encoder reranking — empirical caveat

Multiple published systems (ByteRover, MemMachine) report cross-encoder reranking as part of their LoCoMo improvements. **Independent investigation in DAWN** (see sibling `RERANKER_INVESTIGATION.md`) found that cross-encoder reranking added ~10× retrieval latency without net benefit on conversational data and only marginal lift on LongMemEval — was implemented and reverted.

This finding does not necessarily contradict ByteRover/MemMachine — their reranker may be paired with other architectural pieces that compound the lift, or the lift may be specific to LoCoMo's structure. Worth flagging when comparing reranking claims across systems.

---

## Sources

| Source | URL |
|---|---|
| LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory (ICLR 2025), Wang et al. 2024 | https://arxiv.org/abs/2410.10813 |
| LongMemEval GitHub (official benchmark + leaderboard) | https://github.com/xiaowu0162/LongMemEval |
| LoCoMo: Evaluating Very Long-Term Conversational Memory of LLM Agents, Maharana et al. | https://arxiv.org/abs/2402.17753 |
| ConvoMem Benchmark: Why Your First 150 Conversations Don't Need RAG, Pakhomov et al. 2025 | https://arxiv.org/abs/2511.10523 |
| Letta MemGPT 74% LoCoMo result (blog) | https://www.letta.com/blog/benchmarking-ai-agent-memory |
| Hindsight is 20/20: Building Agent Memory that Retains, Recalls, and Reflects | https://arxiv.org/abs/2512.12818 |
| ByteRover Benchmarking: 92.2% LoCoMo | https://www.byterover.dev/blog/benchmark-ai-agent-memory |
| MemMachine: Ground-Truth-Preserving Memory for Agents | https://arxiv.org/abs/2604.04853 |
| MemPalace Benchmarks Debunked — methodology audit (Vectorize) | https://vectorize.io/articles/mempalace-benchmarks |
| Observational Memory: 95% on LongMemEval (Mastra) | https://mastra.ai/research/observational-memory |
| Reflective Memory Management for Long-term Conversational Understanding | https://arxiv.org/abs/2503.08026 |
| SelRoute: Query-Type-Aware Routing for Long-Term Conversational Memory Retrieval | https://arxiv.org/abs/2604.02431 |

---

## Maintenance protocol

This file is meant to be a **near-static reference** — competitor numbers don't change with DAWN's evolution. Update only when:

1. A competitor publishes a new headline number on a tracked benchmark.
2. A new methodology critique surfaces (audit, blog post, paper note).
3. A new system reaches publication threshold (~commercial release or peer-reviewed paper) and is positioned in the same space.

When updating, also bump the "Last verified" date at the top.

Do **not** add DAWN-side numbers here. DAWN positioning belongs in publication drafts (composed at publish time from current measurements) or in `SYSTEM_DESIGN.md` §14.3 (frozen historical snapshots tied to specific shipments).
