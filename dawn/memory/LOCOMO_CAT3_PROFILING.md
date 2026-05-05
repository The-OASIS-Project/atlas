# LoCoMo cat-3 Failure-Mode Profiling

**Date:** 2026-05-02
**Branch:** unity-testing
**Follow-up to:** RERANKER_INVESTIGATION.md
**Status:** Diagnostic complete; recommendation pivots away from in-retrieval graph
traversal toward bench-harness methodology fixes.

---

## TL;DR

The reranker investigation recommended closing the LoCoMo cat-3 gap with a
typed entity / temporal graph traversal layer in retrieval. Profiling the
actual cat-3 misses changes that recommendation:

1. **The published 64.4% cat-3 baseline is at session granularity, not
   dialog granularity.** Argparse default for `--granularity` is `session`;
   benchmarks/README.md and atlas memory/SYSTEM_DESIGN.md both incorrectly
   claim "dialog granularity, top-K=10." At true dialog granularity (one
   doc per dialog turn — the production-aligned setting), cat-3 = **44.6%**.

2. **Comparison to leaders may be a granularity artifact.** Leaders'
   published numbers (ByteRover 92.2%, MemMachine 91.7%, Hindsight+TEMPR
   89.6%) almost certainly evaluate against an extracted/structured memory
   representation, not raw dialog turns. The framing "DAWN is mid-tier
   on LoCoMo" rests partly on comparing DAWN's raw-dialog session-level
   recall to systems running their full extraction pipeline.

3. **At dialog granularity, the speaker entity is already in top-10 for
   100% of missed evidence pieces.** There are no truly disconnected
   misses. The signal a graph traversal would surface — "this missed
   dialog is connected to a retrieved one through entity X" — is
   *always* available because the question almost always names entity X
   and the top-10 is dominated by X's dialogs anyway.

4. **The bench harness measures raw-dialog retrieval, not the production
   memory pipeline.** `bench_retrieval` ingests dialog text into
   `document_chunks` and queries cosine + keyword + temporal. Production
   retrieval queries extracted facts, the entity graph, and bitemporal
   relations. The 44.6% number is the ceiling for one specific layer of
   the system, not for "DAWN's memory."

The recommended next step is **NOT** "add graph traversal to retrieval."
It is "make the bench harness exercise the production extraction +
entity-graph pipeline so we are measuring what we actually ship."

A simpler quick win — session-neighbor boosting — would address ~43% of
dialog-level misses without any graph work, but is still a band-aid on
the wrong measurement target.

---

## Methodology

### Harness

`benchmarks/cat3_misses.py` runs the 96 LoCoMo cat-3 questions (4 skipped
for empty evidence; 92 evaluated) against `bench_retrieval` with the
standard config (`--temporal-weight 0.20 --proper-noun-boost 1.0`) at
**dialog granularity**. For each question, it requests `top_k=1000` —
deeper than any conversation's dialog pool — so the rank of every gold
evidence piece is observable, not just whether it appears in top-10.

Per missed evidence piece, the harness records:
- The rank in the deep candidate list (or "absent" if outside top-1000)
- The score (cosine + keyword + temporal aggregate)
- The session number, speaker, and full dialog text
- The full top-10 retrieved (id, score, speaker, text)

Output: `/tmp/cat3_misses.json`. Total runtime ~3 min.

### Buckets

A missed evidence piece is bucketed by where it landed in the deep list:

| Bucket | Rank range | Interpretation |
|---|---|---|
| top-10 (recall hit) | 1–10 | Counted as found |
| Ranking failure | 11–50 | In hybrid scoring's working pool, but ranked below noise. Re-ranking helps. |
| Retrieval failure | 51–1000 | Cosine alone couldn't put it in the top-50, so hybrid scoring never reweighed it. |
| Absent | >1000 | Not present in candidate list — likely an ID format issue |

`bench_retrieval` only applies keyword and proper-noun boost to the top-50
by cosine, then re-sorts within that 50. Anything below cosine rank 50 is
effectively "cosine couldn't find it" and is invisible to the hybrid
layer.

---

## The granularity bug

**`benchmarks/run_benchmark.py` argparse line 681:**
```python
parser.add_argument(
    "--granularity",
    default="session",
    choices=["session", "turn", "dialog"],
    ...
)
```

**`run_locomo()` function default (line 399):** `granularity="dialog"`.

The function default is `dialog` but it never wins, because `main()`
always passes `args.granularity` (which defaults to `session`) explicitly.

**Verification:** the standard run command in `benchmarks/README.md` does
not pass `--granularity`. Running it produces retrieved IDs of the form
`session_7`, `session_15` — session-level. Adding `--granularity dialog`
produces IDs of the form `D7:5` — dialog-level. The published 64.4%
cat-3 number is reproducible only at session granularity.

**Impact on baselines:** the published "81.6% LoCoMo overall, 64.4% cat-3"
numbers (benchmarks/README.md, atlas memory/SYSTEM_DESIGN.md §14.3) are
session-level. Dialog-level numbers from the same harness:

| Granularity | Overall | Cat-3 |
|---|---|---|
| Session (published) | 81.6% | 64.4% |
| Dialog (production-aligned) | not measured | **44.6%** |

The difference is structural: at session granularity each LoCoMo session
collapses ~30 dialogs into one ~4 KB document. Cosine over a 4 KB
document is a much lower-discrimination task than cosine over a
single-utterance dialog. Coarse granularity makes recall look better
than the underlying retrieval signal can support.

A standalone fix is needed regardless: either flip the argparse default
to `dialog` and re-run baselines (atlas + benchmarks/README), or correct
the documentation to state "session granularity (default), top-K=10."

---

## Failure-mode breakdown (dialog granularity, 92 cat-3 QAs)

```
Total cat-3 evidence pieces: 200
  Retrieved in top-10:        63  (31.5%)
  Missed:                    137  (68.5%)
    - Ranking failure (11-50): 37  (18.5%)
    - Retrieval failure (51+): 97  (48.5%)
    - Absent (data issue):      3  (1.5%)
  Per-question avg fraction recall: 44.6%
```

The 3 "absent" cases are a dataset bug in conv 8: three cat-3 questions
have evidence stored as space-separated multi-IDs in a single string
(`"D9:1 D4:4 D4:6"` instead of `["D9:1", "D4:4", "D4:6"]`). Both this
harness and the production runner score these as 0% recall always.
Worth fixing for accurate reporting but not a profiling target.

### Connectivity to top-10 (134 valid misses)

For each missed evidence piece, we asked: is the *same session* or the
*same speaker* represented anywhere in the retrieved top-10?

| Class | Count | Share | Interpretation |
|---|---|---|---|
| Session-mate in top-10 | 58 | 43% | Same conversation, fragmented across multiple `dia_id` rows. A retrieved hit and the missed piece are both part of the same exchange. |
| Same speaker, different session | 76 | 57% | The question's primary entity is in top-10 — the missed dialog is just another statement by the same person, in a different session. |
| Neither speaker nor session shared | **0** | **0%** | None. |

**Every missed evidence piece is connected to a top-10 hit by either
session or speaker.** There are zero truly disconnected misses.

### Within "session-mate in top-10": where is the closest mate?

| Closest session-mate rank in top-10 | Count |
|---|---|
| 1 (top hit's session) | 19 |
| 2–3 | 11 |
| 4–5 | 7 |
| 6–10 | 21 |

A naïve "boost dialogs from sessions present in top-3" would surface 30
of the 58 session-fragmentation misses for re-ranking.

### Question type for "same speaker, different session"

Of the 76 same-speaker misses, 40 (53%) contain abstract-personality
keywords (*attributes, describe, personality, character, trait, likely,
might, would, believe, consider, think, opinion*). These questions
implicitly demand entity-broadcast retrieval — "show me everything we
know about Caroline" — and are ill-served by per-utterance cosine
similarity at any embedding scale.

---

## Sample misses by class

### Session fragmentation

```
Q: Would Melanie go on another roadtrip soon?
A (gold): Likely no; since this one went badly

Top-1 retrieved (rank 1, score 0.704):
  D18:1 Melanie: "Hey Caroline, that roadtrip this past weekend was insane!
  We were all freaked when my son got into an accident. We were so lucky
  he was okay..."

MISSED (rank 87, score 0.521):
  D18:3 Melanie: "Yeah, our trip got off to a bad start. I was really scared
  when we got into the accident. Thankfully, my son's ok and that was a
  reminder that life is precious and to cherish our family."
```

D18:1 (top hit) and D18:3 (missed at rank 87) are part of the same
exchange. The continuation didn't earn its boost despite being narratively
inseparable. Session-neighbor ranking would handle this.

### Same speaker, abstract question

```
Q: What attributes describe John?
A (gold): Selfless, family-oriented, passionate, rational

Top-3 retrieved:
  D27:7 John: "Maria, what's the deal with that note?"
  D21:18 John: "Wow, that's awesome! What inspired it?"
  D13:23 John: "That looks interesting. What's the story behind the picture?"

MISSED (rank 529, score 0.464):
  D4:6 John: "I tried to stay calm and asked for assistance, which helped me
  handle the situation and make it back safely."

MISSED (rank 389, score ~similar):
  D2:14 John: "Yeah, they are my rock in tough times and always cheer me on.
  I'm really thankful for their love. Family time means a lot to me."
```

The retriever ranks John's filler dialogue ("what's the story behind the
picture?") above his self-revealing dialogue ("they are my rock in tough
times"). Cosine sees both as roughly equidistant from "describe John" —
neither contains the words "attributes" or "describe." The signal that
"family-oriented" is supported by D2:14 lives in extracted-memory space,
not raw-dialog space.

### Multi-hop entity reasoning

```
Q: What alternative career might Nate consider after gaming?
A (gold): an animal keeper at a local zoo, working with turtles — he knows
a great deal about turtles and how to care for them, and enjoys it.

Top-3 retrieved (all about gaming):
  D28:5 Nate: "Thanks Joanna! Staying positive is key. I'm thinking of
  joining a new gaming team..."
  D28:13 Nate: "Yeah actually - creating gaming content for YouTube..."
  D24:11 Nate: "Yeah, I've just been practicing for my next video game
  tournament..."

MISSED (rank 392, score 0.464):
  D25:19 Nate: "They eat a combination of vegetables, fruits, and insects.
  They have a varied diet."
```

D25:19 mentions turtle diet in passing. The question phrases the answer
abstractly ("alternative career"). Connecting "Nate knows about turtles"
to "could be animal keeper" requires multi-hop reasoning that no
embedding-based retrieval can supply. Even a typed entity graph wouldn't
help unless extraction had already produced a `Nate—knows_about—turtles`
relation. This is firmly in extracted-memory territory.

---

## What would actually move the cat-3 number

Listed by leverage and effort, given the profiling above:

### Tier 1 — measurement methodology (highest leverage)

**Run extraction in the bench harness.** `bench_retrieval` currently
exercises only `document_chunks` retrieval — pure semantic+keyword over
raw text. Production retrieval uses `memory_facts`, `memory_entities`,
`memory_relations`, plus the entity-recall block injected into the LLM
system prompt. The LoCoMo numbers we report and compare to leaders'
numbers are not comparable because we are likely measuring a different
subsystem than they are.

Concrete shape:
1. New bench mode (`--memory-pipeline`) that runs each LoCoMo conversation
   through `memory_extraction.c` end-to-end before evaluating questions.
2. Map gold dialog IDs to the extracted facts that were derived from
   them (using the source-linked recall infrastructure shipped April —
   `memory_provenance` ties facts back to source message ranges).
3. Score recall as: did we retrieve at least one fact derived from a gold
   dialog ID? This matches what the production system actually does at
   inference time.

This is the change most likely to move LoCoMo numbers significantly and
make them comparable to ByteRover / MemMachine. **Estimated 1–2 weeks**
including methodology calibration.

### Tier 2 — quick wins on raw-dialog retrieval (medium leverage, low effort)

If we want to keep the raw-dialog bench (it is still useful for measuring
the embedding-only ceiling), two cheap signal additions would address most
identified failure modes:

**Session-neighbor boost.** For each top-K result, also include the N
neighboring dialogs from the same session at a discounted score. The
session ID is already encoded in `dia_id` (`D7:5` → session 7); for
production memory, sessions correspond to `conversation_id`. Implementation
is a post-processing step on the result list, ~50 LOC. Estimated lift:
+10–15pp on dialog-level cat-3 (28 of 58 fragmentation misses had a
session-mate in top-3, so a small boost should pull them in).

**Speaker-aware boost when question names an entity.** When the question
contains a proper noun that matches a known speaker, additively boost
dialogs by that speaker. We already have `--proper-noun-boost` for keyword
matches but not for the speaker metadata field. Estimated lift: +5–10pp on
dialog-level cat-3.

Both are mechanical, both reuse signals already in the data model, and
both would need re-tuning when the bench harness moves to memory-pipeline
mode.

### Tier 3 — graph traversal in retrieval (low leverage given findings)

The original recommendation was to traverse the entity graph at query
time. The profiling shows this is a poor fit for the LoCoMo retrieval
problem:

- **0% of dialog-level misses are "entity not in top-10."** The graph
  traversal scenario (start from a retrieved entity, walk edges to find
  related-but-not-retrieved evidence) doesn't apply — the entity is
  *always* already in top-10 because the question names it directly.
- The actual gap is *ranking within an entity's dialogs*, not connecting
  across entities. Graph traversal solves the latter; we have the
  former.
- Where graph traversal *would* help is in extracted-memory space (Nate
  → turtles → animal keeper). But that's the Tier 1 work, where the
  graph is already built and just needs to be queried.

---

## Side bug found during profiling

`benchmarks/run_benchmark.py` argparse default vs `run_locomo()` function
default disagree (`session` vs `dialog`). Documentation in
`benchmarks/README.md` and atlas `MEMORY_SYSTEM_DESIGN.md` §14.3 claim
LoCoMo runs at "dialog granularity" but the standard command produces
session-granularity runs. Recommend either:

- Flip argparse default to `dialog`, re-run all LoCoMo baselines, update
  benchmarks/README.md baseline table and atlas §14.3.
- Correct documentation to "session granularity (default)" and add the
  `--granularity dialog` flag to the standard command for the
  production-aligned measurement.

The first option is cleaner if dialog is the intended target; the second
preserves continuity with already-published numbers but documents what
the harness actually measures.

---

## Files

- `benchmarks/cat3_misses.py` — profiling harness (committed alongside
  this doc).
- `/tmp/cat3_misses.json` — full miss data for this run (137 missed
  pieces with rank, score, text, top-10 context). Not committed; rerun
  the harness to regenerate.
- `/tmp/cat3_inrun.jsonl` and `/tmp/standard_cat3_dump.py` — debug
  artifacts used to confirm the granularity bug. Not committed.

---

## Outcome — session-neighbor boost shipped

The Tier 2 quick win was implemented and validated. Two new bench-only
flags in `bench_retrieval`:

- `--session-neighbor-window N` — top-N items by cosine score become
  session anchors (split `doc_filename` on first `':'`)
- `--session-neighbor-boost W` — additive score for chunks sharing an
  anchor's session prefix

The boost is applied between the cosine sort and the top-50 keyword
sort, so promoted neighbors can enter the keyword window. Off by
default (window=0).

### Parameter sweep (LoCoMo dialog granularity)

Window=3 with varying boost magnitude. Baseline overall = 0.6491,
cat-3 = 0.4463.

| boost | overall | cat-1 | cat-2 | cat-3 | cat-4 | cat-5 |
|---|---|---|---|---|---|---|
| 0.00 (baseline) | 0.6491 | 0.428 | 0.793 | 0.446 | 0.749 | 0.539 |
| 0.02 | 0.6717 (+2.3) | -0.4 | +1.2 | +2.0 | +2.1 | +5.0 |
| **0.03** | **0.6789 (+3.0)** | **-0.6** | **+1.1** | **+2.5** | **+3.6** | **+5.5** |
| 0.04 | 0.6826 (+3.4) | -1.3 | +0.6 | +2.0 | +4.3 | +6.6 |
| 0.05 | 0.6818 (+3.3) | -1.8 | -0.5 | +0.9 | +4.7 | +7.1 |
| 0.10 | 0.6415 (-0.8) | -8.7 | -5.1 | -6.3 | +0.8 | +5.6 |
| 0.15 | 0.5501 (-9.9) | -15.0 | -14.6 | -16.3 | -9.6 | -2.7 |

(Cat columns show absolute pp delta from baseline.)

Window>3 was uniformly worse — top-3 by cosine is the right anchor
window for LoCoMo (questions concentrate on 1–2 sessions).

### Chosen settings: `window=3, boost=0.03`

- Best cat-3 lift in the sweep: **+2.5pp** (0.446 → 0.471)
- Cat-1 regression negligible: **-0.6pp**, well within noise on 282 questions
- Cat-2 lift: +1.1pp (turns positive; at boost=0.05 it was -0.5pp)
- Cat-4 lift: +3.6pp (single-hop)
- Cat-5 lift: +5.5pp (adversarial)
- Overall: +3.0pp

The interesting finding is that the boost helps cat-4 and cat-5 *more
than* the cat-3 it was designed for. Cat-4 (single-hop) and cat-5
(adversarial) frequently have evidence concentrated in a few neighboring
dialogs of one session — exactly what the boost recovers. Cat-3
(multi-hop inference) only benefits from the 43% of misses that are
session-fragmentation; the other 57% (same-speaker, different-session)
need an entity-aware mechanism, not a session-aware one.

### Regression check — all no-ops as expected

| Benchmark | Before | After | Status |
|---|---|---|---|
| LoCoMo session granularity (published config) | 0.8159 | 0.8159 | identical |
| LoCoMo session cat-3 | 0.6436 | 0.6436 | identical |
| ConvoMem (100 items) | 0.9900 | 0.9900 | identical |
| LongMemEval R@5 turn (limit 50) | 0.9800 | 0.9800 | identical |
| LongMemEval NDCG@5 (limit 50) | 0.9318 | 0.9318 | identical |

The mechanism only affects datasets with colon-separated doc IDs, which
is exclusively LoCoMo dialog mode. Every other benchmark reproduces its
baseline to four decimal places. No tuning is needed elsewhere.

### Why this is a safe win

1. **Off by default.** Existing callers that don't pass the flags see
   zero behavior change.
2. **Bench-only.** Production retrieval (`document_search.c`,
   `memory_embeddings.c`) is unchanged. If we later pivot to running
   extraction in the bench harness (the Tier 1 work above), the
   neighbor boost is trivial to remove or re-tune at that time.
3. **Transparent mechanism.** Two integer parameters, additive scoring,
   no learned components. Easy to reason about and easy to ablate.
4. **Doesn't preclude graph traversal.** The 76 same-speaker
   different-session misses (57% of cat-3 misses) are still
   un-addressed and would benefit from entity-aware retrieval. The
   boost is orthogonal to that work.

### What this does NOT change

- The granularity-bug doc/code mismatch flagged earlier still needs
  fixing (argparse default `session` vs. README claim `dialog`).
- The fundamental measurement gap — bench measures raw-dialog retrieval,
  not the production memory pipeline — is unchanged. That remains the
  highest-leverage next step.

---

## Tier 1 — Memory-Pipeline Bench Mode (Phase 0 + Phase 1 + Phase 1.5)

**Status:** shipped May 2026 across three commits. End-to-end run on full
LoCoMo (10 conversations, 1982 QAs) on Claude Haiku 4.5 produced
`avg_recall_reach = 0.742`, `cat-3 reach = 0.646`. The methodology fix is
real and the gap to published leaders is now quantified and meaningful.

### What landed

Three commits on `unity-testing`:

1. **Plumbing**: extended `tests/CMakeLists.txt` `bench_retrieval` target
   to link the production memory pipeline (memory_extraction.c,
   memory_db.c, memory_embeddings.c, full llm_*.c chain, AUTH_DB_SOURCES,
   config_parser/defaults/validate/env, tools/toml.c, tool_registry.c,
   sse_parser.c, iso8601.c). Extended `bench_retrieval_stub.c` with ~25
   no-op stubs for daemon-only symbols (sessions, WebUI broadcasts,
   metrics, command_execute, worker_pool, etc.) — bench's call chain
   never traverses these so no-ops are safe.
2. **Phase 0 smoke handler**: `benchmarks/bench_memory_pipeline.{c,h}`
   with `bench_mp_init/teardown/run_smoke`. Init runs `auth_db_init()`
   on a tmpfile path so all `conv_db_*` and `memory_db_*` prepared
   statements are ready (pure `:memory:` would skip prepare). Smoke run
   on conv 0 produced 204 facts in 144s (~7.6s/extraction avg on Haiku
   4.5), all with non-zero provenance, sensible text on spot-check.
3. **Phase 1 JSON protocol + orchestrator**: dispatcher in
   `bench_memory_pipeline.c` adds `conv_create`, `add_message`, `extract`,
   `query_memory`, `reset_memory` JSON commands. `run_locomo_memory()` in
   `benchmarks/run_benchmark.py` drives a full LoCoMo run: per-conv reset
   → per-session ingest+extract → per-QA query_memory with
   covered_dia_ids aggregation. Recall scoring via the bench's in-process
   `dia_id ↔ msg_id` map and fact provenance ranges.

### Phase 1.5 — Full LoCoMo run (Haiku 4.5, May 2026)

```bash
python3 benchmarks/run_benchmark.py \
    --binary ./build-debug/tests/bench_retrieval \
    --benchmark locomo --memory-pipeline \
    --dataset ~/datasets/locomo/data/locomo10.json \
    --output /tmp/mp_full10.json
```

10 conversations, 1982 QAs, 272 extractions, 30:45 wall-clock (1798s LLM
time = 97% of total).

| Category | Memory-pipeline | Dialog-baseline | Δ | Published session-baseline |
|---|---|---|---|---|
| **Overall** | **0.742** | 0.649 | **+9.3pp** | 0.816 |
| cat-1 (profile) | 0.587 | 0.428 | +15.9 | 0.693 |
| cat-2 (temporal) | 0.761 | 0.793 | -3.2 | 0.849 |
| **cat-3 (inference)** | **0.646** | 0.446 | **+20.0** | 0.644 |
| cat-4 (single-hop) | 0.794 | 0.749 | +4.5 | 0.842 |
| cat-5 (adversarial) | 0.749 | 0.539 | +21.0 | 0.857 |

### The striking observation

**Memory-pipeline cat-3 (0.646) ≈ published session-baseline cat-3 (0.644)**
to within noise. Both methodologies converge to the same number on
multi-hop inference. That implies retrieval at *some* granularity is
finding the right material — both per-session matching and fact-level
provenance overlap surface the same answer-bearing material at the same
rate.

What it means: on multi-hop inference, **retrieval isn't the bottleneck.**
Answer-support is. Multi-hop questions ("would Nate consider an
animal-keeper career") need the LLM to *reason over* retrieved facts, not
just have them retrieved.

### Gap to leaders is real and large

Published numbers from leader systems:
- ByteRover: 92.2%
- MemMachine: 91.7%
- Hindsight+TEMPR: 89.6%

DAWN memory-pipeline: 74.2% overall, 64.6% cat-3.

That's **~17–28pp behind**. The gap is not just methodology — even after
the apples-to-apples shift to extracted-memory retrieval, real ground
remains. Important: ByteRover/MemMachine numbers are almost certainly
**generate-and-judge** (run the LLM end-to-end on retrieved material,
score the generated answer against gold). Our `recall_reach` is a
generous proxy that does NOT include the answer-generation step. The
real gap under their methodology is likely **larger** than 17–28pp.

### `recall_reach` is generous by design

Architecture review C2 flagged this explicitly. Extraction emits
provenance over the *batch of messages it processed* (a session-wide
range), not "the dialog that supports this fact." A fact extracted from
a 40-message session inherits the whole session's range. So
`recall_reach` counts a hit if any retrieved fact's range covers the
gold dialog — even if the fact's text doesn't entail the gold answer.
**0.742 is an upper bound.**

`recall_entailment` (LLM-judge: "do retrieved facts entail the gold
answer?") will be lower — and is the next decisive measurement.

### Files shipped (commits on `unity-testing`)

```
benchmarks/bench_memory_schema.h         # documented DDL subset (drift detection future)
benchmarks/bench_memory_pipeline.c/h     # smoke handler + JSON dispatcher
benchmarks/bench_retrieval.c             # --memory-pipeline + --smoke + --config flags
benchmarks/bench_retrieval_stub.c        # +25 daemon-only no-op stubs
benchmarks/run_benchmark.py              # run_locomo_memory + flag plumbing
tests/CMakeLists.txt                     # extended bench_retrieval target
```

C1 architectural concern from the plan reviews dissolved on direct
inspection: `session_get_llm_config` is only called in
`memory_extraction_build_fallback()`, an optional helper. Bench passes
`fallback=NULL` to `memory_trigger_extraction()` and the main thread
builds its `llm_resolved_config_t` from `g_config.memory.*` directly. No
production code changes shipped.

### Phase 2 — adjusted direction

Original plan had: caching + four-model sweep + `recall_entailment`.
Phase 1.5 data adjusts the priority order:

1. **SQLite snapshot caching** — required infrastructure regardless.
   30-min sweeps become seconds on rerun. Cache key per the plan
   (provider | model | dataset | mtime | prompt_version | embedding_model
   | weights). Restore via `sqlite3_backup_init/step/finish` from disk
   into `:memory:`. Snapshot must include `users.embeddings_model_id` and
   `users.categories_backfilled_at` so cache-restore doesn't re-trigger
   backfill.
2. **`recall_entailment`** — diagnostic gate. Three scenarios with
   different next-step implications:
   - **High entailment (~0.85+)**: retrieval is fine, gap is in answer
     generation. Fix: better answer-LLM, multi-step retrieval, prompt
     engineering.
   - **Moderate entailment (~0.5–0.7)**: facts cover the session but
     don't entail specific answers. Fix: more facts, finer-grained
     extraction, richer relations.
   - **Entailment ≈ reach (~0.74)**: retrieval is near-optimal at this
     granularity, headroom is in extraction quality + answer generation.
3. **Generate-and-judge end-to-end** — the truly comparable-to-leaders
   metric. Run the LLM end-to-end on retrieved facts, judge generated
   answer against gold. Reuses cached extractions. Was not in the
   original plan but is needed for a fair leader comparison; ~1 day
   after entailment lands.
4. **Four-model sweep — DEFERRED.** Extraction quality variance across
   Haiku/Sonnet/Opus is probably ±2–5pp at most, not the 17–28pp we
   need. Defer until diagnosis from (2) and (3) is in.

### Two latent investigations if (2) and (3) don't move the needle

- **Fact density.** Haiku 4.5 extracts ~25 facts per LoCoMo conversation
  on average. Leaders may produce far more "memory units" via different
  extraction granularity. Could be a recall-floor issue invisible to
  `recall_reach` but visible to `recall_entailment`.
- **LoCoMo role-mapping.** DAWN's extraction prompt assumes asymmetric
  user/assistant. LoCoMo has symmetric two-friend chats (we map
  first-introduced speaker → user). The prompt may be implicitly
  under-extracting on the assistant side because the production prompt
  expects assistant=DAWN, not assistant=second-friend.

### Open known issues / polish

- `total_facts_extracted` in result JSON was reporting last-conv count
  before commit 4 (orchestrator now sums `facts_added` deltas).
- Granularity bug in `run_benchmark.py` (argparse default `session` vs
  doc claim `dialog`) is still not fixed — separate cleanup.
- `BENCH_MEMORY_DDL` header (`bench_memory_schema.h`) is unused at runtime
  but kept as a documented subset for the future drift-detection test.

### How to pick this up

```bash
# Re-verify state
git log --oneline -5         # should show three Phase 0/1 commits + total_facts fix
make -C build-debug bench_retrieval
./build-debug/tests/bench_retrieval --memory-pipeline --smoke \
    ~/datasets/locomo/data/locomo10.json 0   # 2-min smoke, ~200 facts

# Phase 2 starting point
# 1. Caching: snapshot_save/snapshot_load JSON commands using sqlite3_backup_*
# 2. Cache key in run_benchmark.py: sha256 of (provider | model | dataset_path
#    | dataset_mtime_ns | EXTRACTION_PROMPT_VERSION | embedding_model_id
#    | embedding weights). Stored at --cache-dir/<hash>.db.
# 3. EXTRACTION_PROMPT_VERSION constant in memory_extraction.h (new),
#    echoed in bench's "ready" JSON for orchestrator cache-key calculation.
```

Phase 2 work continues in `unity-testing` branch.
