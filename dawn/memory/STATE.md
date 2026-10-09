# Memory Subsystem — Current State

Read this before starting memory work. It holds where DAWN's memory stands against published
systems, why work paused where it did, what has already been tried and failed, and what's next.
History lives in git and in `dawn/docs/DONE.md`; design detail lives in the docs linked below.

**North star:** JARVIS/FRIDAY minus the sci-fi. The assistant knows the user, brings up the right
context as the conversation moves, and never makes things up. Benchmarks serve that; they don't
define it.

**Last updated:** 2026-10-09.

## Where it stands

LoCoMo `recall_generation`, leader-comparable: **0.7689** (2026-10-08), with Claude Haiku 5.5
as `[memory] extraction_model` and `extraction_effort = "medium"` (the setting arrived in dawn
PR #38). That is the recommended setup, not the shipped default: a fresh install extracts with a
local `qwen2.5:7b`, effort off, which hasn't been benchmarked.

Haiku 4.5 extraction scored 0.7279 on the same protocol. Per-effort scores, categories, cost
and timing: dawn `benchmarks/README.md` § Claude Haiku 5.5 extraction.

| System | LoCoMo | vs DAWN |
|---|---|---|
| ByteRover 2.0 | 92.2% | +15.3pp |
| MemMachine v0.2 | 91.2% | +14.3pp |
| Hindsight (Backboard) | 90.0% | +13.1pp |
| Hindsight (Gemini-3) | 89.6% | +12.7pp |
| **DAWN** | **76.9%** | — |
| Letta | 74.0% | −2.9pp |
| Mem0 (self-reported) | 68.5% | −8.4pp |

Competitor sources: [COMPETITOR_LANDSCAPE.md](COMPETITOR_LANDSCAPE.md).

### Which numbers can be compared

Only `recall_generation` under the Mem0 protocol is comparable to published systems: generator
and judge `gpt-4o-mini`, `--prompt-style mem0`, `--with-source`, `--exclude-categories 5`,
n=1536. The bench prints `LEADER-COMPARABLE: YES|NO` and writes `leader_comparable` into the
results JSON; check it before quoting a number anywhere public. Run command: dawn
`benchmarks/README.md` § Comparing to published systems.

`recall_reach` (does top-K retrieval cover the gold dialog IDs) is an internal regression check.
It is not what ByteRover, MemMachine, Hindsight or Mem0 publish. An earlier version of this file
compared the two; that was wrong. Current internal numbers: LoCoMo reach 91.7%, LongMemEval R@5
97.0%, ConvoMem 99.0%. LongMemEval R@5 is the one reach figure others also publish.

## Why retrieval work is paused

Retrieval tuning stopped on 2026-05-17 at 0.7324 (Haiku 4.5 extraction). Profiling the wrong
answers showed **52.8% of them were never in the memory text at all**: the facts weren't
extracted. No retrieval change can fix that, and the three retrieval-side candidates queued at
the time each measured at about 1pp of possible gain or less (see dead letters below).

The Haiku 5.5 result confirms it. Changing only the extraction model and its reasoning effort,
with retrieval untouched, moved generation +4.1pp, mostly on temporal questions (cat-2 0.676 →
0.779). Effort off scores lower than Haiku 4.5, so the reasoning is doing the work, not the
model. Cat-3 multi-hop (0.533) is still the weakest category.

How the May baseline was reached (BM25, extraction prompt v2, specificity rules):
[MEM0_ARCHITECTURAL_PARITY.md](MEM0_ARCHITECTURAL_PARITY.md).

## Dead letters: don't re-propose without new evidence

Each was measured or researched and lost. Reopen only with evidence that changes the reason.

| Idea | Why it's dead |
|---|---|
| Cross-encoder reranker as the main lever | OMEGA removed theirs and gained; best published gain is small. A dominant-token heuristic shipped instead ([RERANKER_INVESTIGATION.md](RERANKER_INVESTIGATION.md)). |
| RRF parallel-channel retrieval | −3.7pp, then −0.58pp on re-test after BM25. Code stays behind `rrf_enabled` (off). |
| Mem0 scoring weights 0.40/0.60, over-fetch 40, spread-attenuated entity boost | −0.72pp, isolated to the weight shift; the boost duplicates the entity graph's 1-hop expansion. All reverted. |
| Mem0 no-echo extraction rule | Cat-4 −2.4pp; DAWN's source excerpts already separate user from assistant. |
| Side-mention completeness extraction rule | −0.78pp, cat-3 −6.5pp. |
| Per-question-type generation prompts | Leaders use one template; would break the comparable protocol. |
| Entity-graph traversal past 1–2 hops | Literature and plateau evidence against it. |
| Shorter summaries in memory text | −1.3pp; answers sit mid-summary (median char 606), so any cap that saves space loses them. |
| Temporal query-parser expansion | 86% of cat-2 misses weren't in memory; ceiling +1.3pp overall. |
| Question decomposition before retrieval | Same not-in-memory diagnosis; ~1–3pp ceiling for a large change. |

## What's next

Extraction recall is the open problem; the candidates (per-turn extraction, speaker
attribution) and the rest of the memory backlog are in `dawn/docs/TODO.md`.

## Keeping this current

When a benchmark number moves or a direction opens or closes, edit the section it belongs to and
change the date. Don't add history here; that's what git and DONE.md are for. How the subsystem
works: [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md).
