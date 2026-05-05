# Cross-Encoder Reranker Investigation

**Status**: Investigated, dropped May 2026.
**Author**: Implementation by Claude Code (Sonnet 4.6 + Opus 4.7); decisions by Kris Kersey.

## Summary

We implemented a cross-encoder reranker stage on top of DAWN's bi-encoder
retrieval pipeline (bge-small-en-v1.5-int8 + keyword + temporal +
proper-noun boost), then **dropped it** after empirical results and a
literature review showed it provides no net benefit on our primary
workload (LoCoMo conversational dialog) and at best a marginal lift on
LongMemEval at 10× the latency.

The full implementation reached production-quality (clean ONNX +
CUDA execution-provider integration, comprehensive review-pass fixes,
36→41 unit tests passing) before being reverted.  This document captures
what we tried, what we measured, and what the literature says, so we
don't repeat the investigation in six months.

## Original Premise

The original embedding-upgrade plan (now archived as `EMBEDDING_UPGRADE.md`, sibling) §Feature 2 noted that LoCoMo recall jumps
from 81.6 % at top-K=10 to 89.9 % at top-K=15 — meaning the right
evidence sat at ranks 11–15 for ~8 % of questions.  The plan inferred that
a cross-encoder reranker could pull those items into top-10 by scoring
(query, candidate) pairs jointly, recovering most of that 8 pp.

The plan **did not test any reranker model before committing to the
design**.  ms-marco-MiniLM-L-6-v2 was selected because it had a
pre-quantized int8 ONNX export from the same Xenova source as our
bi-encoder, was small (22 MB), and used a compatible WordPiece
tokenizer.

## What We Built

| Piece | Lines | Status |
|---|---|---|
| `memory_embed_tokenizer.{c,h}` — extracted shared WordPiece tokenizer with refcount | ~400 | **Kept** (clean refactor — bi-encoder shares it now, future BERT-tokenizer consumers can too) |
| `memory_embed_rerank.{c,h}` — ONNX cross-encoder module with CUDA EP, batched inference, mutex-protected session | ~350 | Reverted |
| Integration in `document_search.c` (RAG path) and `memory_embeddings_hybrid_search` (memory facts path) | ~120 | Reverted |
| `memory_db_fact_get_texts_by_ids` — bulk fact-text fetcher for the rerank pass | ~60 | Reverted |
| Config plumbing (5 keys: enabled / model_path / candidate_pool / threads / use_gpu) across `dawn_config.h`, defaults, parser, validate, env, webui_config, dawn.toml.example, settings/schema.js | ~80 | Reverted |
| `tests/test_embed_rerank.{c,stub}` — Unity tests including ONNX ranking-sanity probe | ~280 | Deleted |
| `tests/test_embed_tokenizer.c` — 8 CI tests for the shared tokenizer | ~230 | **Kept** |
| `benchmarks/rerank_shootout.py` — Python harness for testing arbitrary cross-encoder models without C-side tokenizer porting | ~270 | **Kept** (future test harness) |
| Bench wiring in `bench_retrieval.c` and `run_benchmark.py` (`--reranker`, `--reranker-pool`, `--reranker-model` flags) | ~70 | Reverted |

CI suite went 36 → 41 tests during the investigation; reverts to 37
(test_embed_tokenizer is the new permanent addition).

## Results

### Initial integration (default flow: hybrid scoring → rerank top-20 → return top-10)

LoCoMo conversation 1, GPU reranker (ms-marco-MiniLM-L-6-v2 int8):

| | Baseline (no rerank) | With reranker | Δ |
|---|---|---|---|
| recall@10 overall | 0.843 | 0.666 | **−0.177** |
| cat 1 (profile) | 0.682 | 0.683 | +0.001 |
| cat 2 (temporal) | 0.919 | 0.784 | −0.135 |
| cat 3 (inference) | 0.758 | 0.545 | −0.213 |
| cat 4 (single-hop) | 0.871 | 0.629 | −0.242 |
| cat 5 (adversarial) | 0.872 | 0.638 | −0.234 |

Severe regression.  The reranker was *overriding good rankings produced
by the keyword + temporal + proper-noun-boost signals* with worse ones.

### Python shootout: integration-bug isolation + model comparison

To isolate the integration bug from the model choice, we wrote a Python
harness that runs bge-small bi-encoder for first-stage retrieval, then
reranks the top-50 with each candidate cross-encoder.  Pure-cosine
baseline (no keyword / temporal / proper-noun boost), so any movement is
pure reranker contribution.

LoCoMo conversation 1, pure-cosine baseline = **0.620 recall@10**.

| Reranker model | Final | Δ vs cosine baseline | Latency (CPU) | Size | Tokenizer |
|---|---|---|---|---|---|
| ms-marco-MiniLM-L-6-v2 | 0.709 | +0.089 | 582 ms | 22 MB | WordPiece |
| ms-marco-MiniLM-L-12-v2 | 0.725 | +0.104 | 1100 ms | 33 MB | WordPiece |
| jina-reranker-v1-tiny-en | 0.663 | +0.043 | 540 ms | 33 MB | BPE |
| **mxbai-rerank-base-v1** | **0.732** | **+0.112** | 4500 ms | 244 MB | SentencePiece |

Per-category Δ for the two best models:

| cat | ms-marco-L-12 Δ | mxbai-base Δ |
|---|---|---|
| 1 (profile) | +0.190 | +0.237 |
| 2 (temporal) | +0.027 | +0.054 |
| 3 (inference) | +0.182 | +0.091 |
| 4 (single-hop) | +0.107 | +0.064 |
| 5 (adversarial) | +0.085 | +0.149 |

**Every cross-encoder lifted the pure-cosine baseline.**  The earlier
LoCoMo regression was indeed an integration bug — keyword + temporal +
proper-noun boost were doing similar work to the reranker and the
reranker was overriding their already-good rankings.

### Integration fix: skip hybrid pre-scoring when reranker active

We modified `document_search.c` and `memory_embeddings_hybrid_search` to
skip keyword / temporal / proper-noun boost when the reranker is active,
feeding it pure-cosine candidates instead.

LoCoMo conv 1, GPU reranker (L-12, pool=50) **after** the integration
fix:

| Setup | recall@10 |
|---|---|
| Baseline: no reranker, full hybrid scoring (cosine + keyword + temporal + proper-noun) | **0.843** |
| Pure cosine (no boost, no reranker) | 0.652 |
| Pure cosine + reranker (the new path) | 0.656 |

**The reranker provides essentially no lift over pure cosine in our
actual stack** — and the combined hybrid scoring without reranker beats
both by 19pp.

### LongMemEval subset (n=20, turn granularity, official scoring)

| metric | Baseline | Reranker | Δ |
|---|---|---|---|
| R@1 | 0.900 | 1.000 | +0.100 |
| R@3 | 0.950 | 1.000 | +0.050 |
| R@5 | 0.950 | 1.000 | +0.050 |
| NDCG@1 | 0.900 | 1.000 | +0.100 |
| NDCG@3 | 0.921 | 0.988 | +0.067 |
| NDCG@5 | 0.902 | 0.974 | +0.072 |

LongMemEval queries are more web-search-like (questions about facts in
prior sessions), and the reranker's MS MARCO training transfers better
here.  +5 pp R@5 on the subset.  Modest gain; not validated at full
scale before the decision to drop.

### ConvoMem

99.0 % avg recall both with and without reranker — ceiling effect, no
discriminative power.

## Why ms-marco Doesn't Help LoCoMo

ms-marco-MiniLM-L-6-v2 (and the L-12 variant we tested) were trained
on the MS MARCO Passage Ranking dataset: ~500 K tuples of (search query,
web passage, relevance label).  The training data looks like:

```
query:  "where is the eiffel tower located"
passage: "The Eiffel Tower is a wrought-iron lattice tower on the
          Champ de Mars in Paris, France..."
```

LoCoMo data looks like:

```
query:   "When did Sam break his ankle?"
passage: "Sam said, \"I had a horrible time at the dentist last Tuesday.\""
```

Different query distribution (conversational temporal vs. web search),
different passage format (`Speaker said, "..."` quoted dialog vs. web
prose), different relevance signals.  The cross-encoder's learned scoring
priors don't transfer.

Bi-encoder cosine is more domain-agnostic — embedding distance survives
the format shift better than learned ranking patterns.

## Latency

Even with the GPU execution provider attached and `IntraOpNumThreads`
pinned to 1 to prevent CPU oversubscription, the reranker added **~600
ms per query on LoCoMo and ~3 s per query on LongMemEval** at pool=20.
Pool=50 (the value that actually delivered the LongMemEval lift) doubles
that.

For comparison, the existing hybrid bi-encoder path runs at **~58
ms/query**.  Voice-mode TTS first-token budget is ~200 ms; the reranker
would dominate it.

## Literature Review

The literature **confirms** our finding.  Recent (2023–2026) work on
conversational and personal-memory retrieval converges on multi-signal
hybrid scoring rather than pre-trained cross-encoder reranking:

1. **MS-Shift** (Thawani et al.) documents 10–20 pp drops when MS MARCO
   models are applied to non-web-search query distributions.
2. **M2A** (2025), **Beyond Dialogue Time** (2025), **MemPalace** (2025)
   all achieve SOTA on conversational benchmarks via dense + sparse +
   temporal hybrid scoring.  None use cross-encoder rerankers.
3. **AFR-Rank** (2025) names "reranker noise override" as a documented
   failure mode: cross-encoders trained on pairwise ranking loss don't
   preserve the orderings that good multi-signal scoring produces — they
   confidently override correct top-1 results to lower ranks.  This is
   exactly the 17 pp regression we measured before the integration fix.
4. **Jina Reranker v3** (2024) shows that *late-interaction*
   (ColBERT-style token-level scoring) outperforms full cross-encoder
   reranking on general retrieval, but no conversational benchmarks.
5. **MemPalace** demonstrates that *zero-shot LLM reranking* (Claude
   Haiku in their setup) can lift retrieval to 100 % on LongMemEval, but
   at one LLM API call per query — cost-prohibitive for voice.

## Where DAWN Sits in the Landscape (May 2026)

| Benchmark | DAWN | Best published | Method |
|---|---|---|---|
| LongMemEval R@5 | **97.0 %** | MemPalace 96.6 %, BM25 93.8 %, ours leads | bge-small + hybrid |
| LongMemEval NDCG@5 | **92.5 %** | (others rarely report NDCG) | leading on quality |
| LoCoMo overall | 81.6 % | ByteRover 92.2 %, MemMachine 91.7 %, Hindsight+TEMPR 89.6 % | mid-tier, 8–15 pp behind |
| LoCoMo cat-3 (inference) | 64.4 % | ByteRover 85.1 %, Hindsight 70.8 % | **biggest gap** |
| ConvoMem | **99.0 %** | (no published exceeds; baseline 70–82 %) | ceiling |

DAWN leads on LongMemEval and ConvoMem.  The LoCoMo gap is concentrated
in **cat-3 (inference)** — questions requiring multi-hop reasoning
("Did Sam ever mention his sister?  What did she do?").  The leaders
here use **structured entity / temporal graphs**, not better
cross-encoders.

## Decision

**Drop the reranker.  Keep the tokenizer extraction (legitimate
refactor) and the Python shootout harness (future model testing).**

The path to closing the LoCoMo cat-3 gap is structured memory — entity
graphs and temporal reasoning structures — which is **orthogonal** to
ranking quality.  See `STRUCTURED_MEMORY_DESIGN.md` (TBD) for the
follow-up direction.

## Things We'd Try Differently

If we revisit reranking later, the literature points to these
alternatives that have published evidence on conversational data:

1. **Zero-shot LLM-as-listwise-reranker** with a fast model (Haiku,
   Gemini Flash).  MemPalace shows this works.  Need careful cost/latency
   gating and probably a "rerank only if confidence is low" trigger.
2. **ColBERT-style late interaction** (Jina-ColBERT-v2 or similar).
   Token-level scoring is more expressive than mean-pooled bi-encoder
   without the full cross-attention cost.  Different model + tokenizer
   architecture; significant work.
3. **Domain fine-tuning** of a small cross-encoder on labeled
   conversational QA pairs.  Out of scope without labeled data; a future
   project once we have user feedback signal we can mine.

## What We Kept Beyond the Tokenizer Refactor

- **GPU execution provider integration pattern** — the
  `OrtSessionOptionsAppendExecutionProvider_CUDA` invocation, lazy
  fallback to CPU on attach failure, and `IntraOpNumThreads=1` pinning to
  prevent CPU oversubscription on hybrid CPU/GPU graphs.  Lives in the
  reverted `memory_embed_rerank.c` (now deleted) but the recipe is here
  if we want to GPU-accelerate the bi-encoder.

- **`benchmarks/rerank_shootout.py`** — Python harness that pulls
  candidates from any bi-encoder (ours or another), then reranks with any
  HF model regardless of tokenizer (BPE, SentencePiece, WordPiece).
  Useful for testing future reranker candidates without touching C.

## Files in this Investigation

| Path | Disposition |
|---|---|
| `include/memory/memory_embed_tokenizer.h` | Kept |
| `src/memory/memory_embed_tokenizer.c` | Kept |
| `tests/test_embed_tokenizer.c` | Kept (CI) |
| `benchmarks/rerank_shootout.py` | Kept |
| `include/memory/memory_embed_rerank.h` | **Delete** |
| `src/memory/memory_embed_rerank.c` | **Delete** |
| `tests/test_embed_rerank.c` | **Delete** |
| `tests/test_embed_rerank_stub.c` | **Delete** |
| `models/embeddings/ms-marco-MiniLM-L-6-v2-int8.onnx` (22 MB) | **Delete** |
| `models/embeddings/ms-marco-MiniLM-L-12-v2-int8.onnx` (33 MB) | **Delete** |
| `models/embeddings/ms-marco-L-12-vocab.txt` (duplicate of vocab.txt) | **Delete** |
| `models/embeddings/jina-reranker-v1-tiny-en-int8.onnx` (33 MB) | **Delete** |
| `models/embeddings/jina-tokenizer.json` | **Delete** |
| `models/embeddings/jina-vocab.txt` (15 bytes "Entry not found") | **Delete** |
| `models/embeddings/mxbai-rerank-base-v1-quantized.onnx` (244 MB) | **Delete** |
| `models/embeddings/mxbai-tokenizer.json` | **Delete** |

## Next Direction: Structured Entity / Temporal Graph Retrieval

The path to closing the LoCoMo gap is **structured memory**, not better
ranking.  ByteRover (92.2 % overall, 85.1 % cat-3), MemMachine (91.7 %),
and Hindsight+TEMPR (89.6 %) all use typed entity graphs with temporal
edges and traverse them at query time — not embeddings + reranking.

### What DAWN already has

- `memory_entities` table — typed entity nodes
- `memory_relations` table — `(subject, relation, object)` triples
- Bitemporal validity on relations (`valid_from`, `valid_to`) — added April 2026
- Entity resolution / canonicalization at extraction time
- Relation-driven fact contradiction (April 2026) — exclusive relations auto-supersede

### What's missing

Retrieval doesn't traverse the graph.  `memory_callback`'s recall path
runs hybrid keyword + cosine search over `memory_facts`; the entity
graph is currently used only for the entity-recall block in the system
prompt, not for finding facts in response to multi-hop questions.

### Sketch of the design

1. **Query classifier** — small LLM call or rule-based, tags each query
   as `lookup | temporal | multihop | profile`.  Routes to specialized
   retrieval.  Heuristic could trigger on keywords like "when",
   "after", "did X also", proper-noun count, etc.

2. **Graph traversal retrieval** — when the query mentions entities
   (proper nouns or matched canonical names), do a 1-2 hop expansion in
   `memory_relations` from those seed entities, collect connected
   entities and their associated facts, return that subgraph as the
   candidate set.  Apply semantic ranking only within the subgraph.

3. **Bitemporal filter** — when the query has temporal expressions
   (parser already detects this), filter `memory_relations` by
   validity window before expansion.

4. **Reasoning-type-aware response formatting** — multi-hop queries
   should return the path (e.g., "Alice → works_at → Acme →
   located_in → Boston") so the LLM can reason about the chain
   explicitly.

### Estimated effort

| Piece | Days |
|---|---|
| Query classifier (heuristic) | 0.5 |
| Graph traversal in `memory_db.c` | 3-5 |
| Bitemporal filter integration | 1 |
| Response formatting + system prompt update | 1 |
| Benchmark + tune | 2-3 |
| **Total** | **~2 weeks** |

(Same order of magnitude as the reranker work; with much higher
ceiling.)

### Risks

- **Coverage bound** — if the LLM extraction layer doesn't extract a
  relation, traversal can't find it.  Cat-3 lift is ultimately bounded
  by extraction quality.  Worth a parallel pass on extraction prompt
  tuning.
- **Cross-session entity canonicalization** — works today within a
  session.  Multi-session reasoning depends on it working across time.
  Likely fine but worth validating on LoCoMo's specific cases.
- **Inferential cat-3 questions** ("what was Alice's mood when she
  said X?") aren't pure retrieval — they need LLM reasoning over the
  retrieved subgraph.  Same problem the reranker had.  ByteRover
  likely closes this gap with prompt engineering on the LLM consuming
  the structured retrieval output, not retrieval itself.

### Recommended next step (not yet started)

Profile the LoCoMo cat-3 errors **before** designing.  Pull the questions
we miss, classify the failure mode (extraction failure vs. retrieval
failure vs. ranking failure vs. LLM reasoning failure), then design with
eyes open.  ~1 day.  The reranker investigation suffered from designing
without diagnostic data; don't repeat that.

## References

- DAWN bench harness: `benchmarks/run_benchmark.py`, `benchmarks/bench_retrieval.c`
- Python shootout: `benchmarks/rerank_shootout.py`
- LongMemEval (ICLR 2025): https://arxiv.org/abs/2410.10813
- LoCoMo (Maharana et al., 2024): https://arxiv.org/abs/2402.17753
- MemPalace benchmarks (2025): https://www.mempalace.tech/benchmarks
- AFR-Rank (2025): https://www.sciencedirect.com/science/article/abs/pii/S0306457325001736
