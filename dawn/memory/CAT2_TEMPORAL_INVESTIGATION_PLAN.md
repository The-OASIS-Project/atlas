# Cat-2 Temporal Extraction — Investigation Plan

**Status:** working doc, untracked. Do not commit until investigation produces
a concrete fix plan that's been turned into a design doc.

**Owner:** next session. Self-contained — does not depend on the in-progress
sweep work continuing.

## Context (the part that doesn't change)

The May 2026 four-model LoCoMo sweep (full 10 convs, 1982 QA pairs) showed
that cat-2 (temporal — "when did X happen?") `recall_generation` sits at
0.02-0.03 across **every** extraction model:

| Model | reach | ent | gen |
|---|---|---|---|
| `claude-haiku-4-5` | 0.771 | 0.022 | 0.022 |
| `claude-sonnet-4-6` | 0.773 | 0.037 | 0.028 |
| `claude-opus-4-6` | 0.659 | 0.025 | 0.028 |
| `local:Qwen3.6-35B-A3B` | 0.569 | 0.025 | 0.016 |

**Reach is healthy** (~0.77 for Haiku/Sonnet — comparable to historical 84.9%
cat-2 reach over raw dialog chunks). The right facts are being retrieved.
But generation and entailment are catastrophic — facts don't contain the
date information needed to answer "when".

**Therefore:** the bottleneck is between the retrieval boundary and answer
synthesis. Either (a) extraction is stripping dates from facts, or (b) dates
are being stored in a structured field (e.g., `valid_from`/`valid_to`) that
the LLM doesn't see when it goes to generate, or (c) some combination.

Fixing this is the highest-leverage single piece of memory work on the
board. Cat-2 from 0.025 → 0.20 lifts overall `recall_generation` by roughly
+3.5 pp (cat-2 is ~20% of LoCoMo). Larger absolute lift than any model swap.

## Why this needs investigation, not implementation

We have a hypothesis but no evidence about *where* in the pipeline dates are
lost. The pipeline has at least four candidate failure points:

1. **Extraction prompt** — does it explicitly tell the LLM to preserve dates?
2. **Extraction LLM behavior** — even with the prompt, does the model emit
   dates in the fact text?
3. **Schema / storage** — `memory_facts.fact_text` vs
   `memory_relations.valid_from`/`valid_to` — does extraction populate the
   bitemporal fields, and for which kinds of utterances?
4. **Generation prompt** — when the generator sees retrieved facts, does it
   see the dates? (The bench `AnswerGenerator` shows facts as their
   `fact_text` only; bitemporal fields would need explicit inclusion.)

Without knowing which layer is dropping dates, it's easy to fix the wrong
thing. The investigation gives us a one-page diagnosis pinpointing the layer.

## Investigation procedure

All artifacts needed are already on disk under `benchmarks/snapshots/`:

- `<hash>.db` — per-(model, conv) extraction snapshots (raw SQLite,
  contains `memory_facts`, `memory_relations`, `messages`, `conversations`,
  full provenance)
- `<hash>.json` — dia_id ↔ msg_id maps
- `judgements.json` / `generations.json` / `correctness.json` — full LLM
  verdicts with raw responses

The Haiku-extraction snapshots are the right starting point (it's the
production-default model and has the cleanest signal).

### Step 1: Identify the worst cat-2 questions

```python
# pseudocode
import json
locomo = json.load(open("~/datasets/locomo/data/locomo10.json"))
gens = json.load(open("benchmarks/snapshots/generations.json"))
corrs = json.load(open("benchmarks/snapshots/correctness.json"))

# Find all cat-2 questions where the Haiku extraction snapshot's
# generation got NO from the correctness judge.
# Use JudgeCache.make_key shape to look up by (model, version, q, gold, fact_texts).
# The fact_texts dimension is the per-snapshot retrieved facts — pull from
# the snapshot DB by replaying the orchestrator query path.
```

Goal: shortlist of ~10 cat-2 questions with reach=1 (gold dia_ids covered)
and gen=0 (judge said wrong answer). Those are the cleanest cases —
retrieval succeeded, synthesis didn't.

### Step 2: For each shortlisted question, pull three things

For each gold `dia_id` mentioned in the question's evidence:

1. **The raw dialog turn** from the LoCoMo dataset: speaker, text. Does
   it actually mention a date? What format? ("last Tuesday", "in 2018",
   "two weeks ago", "August 5th"?)
2. **The corresponding extracted fact(s)** from the Haiku snapshot DB,
   joined via provenance:
   ```sql
   SELECT f.id, f.fact_text, f.category,
          fs.conv_id, fs.msg_start, fs.msg_end
   FROM memory_facts f
   LEFT JOIN memory_fact_sources fs ON fs.fact_id = f.id
   WHERE fs.msg_start <= ? AND fs.msg_end >= ?  -- the dia_id's msg_id
   ```
   Does the fact text contain the date? In what form?
3. **Any matching `memory_relations` entries** with `valid_from` /
   `valid_to` populated:
   ```sql
   SELECT subject, predicate, object, valid_from, valid_to
   FROM memory_relations
   WHERE subject IN (...entities for this fact...)
   ```
   Are bitemporal bounds being captured?

### Step 3: Categorize the failure mode

For each of the ~10 cases, label which layer dropped the date:

- **A.** Dialog has no date → not actually a fix opportunity (dataset issue)
- **B.** Dialog has date, fact text has date, but it's in a free-form
  string the LLM can't reliably parse at generation time
- **C.** Dialog has date, fact text *missing* the date entirely → extraction
  prompt issue
- **D.** Dialog has date, fact text missing it, but `memory_relations.
  valid_from`/`valid_to` populated → bitemporal capture works, but the
  generator's prompt doesn't include those fields
- **E.** Dialog has date, fact + relation both missing it → extraction is
  fully losing the temporal signal at the LLM level

The distribution across A-E tells us where to fix.

### Step 4: Inspect the extraction prompt

Open `src/memory/memory_extraction.c` (or wherever the extraction prompt
template lives) and check:

- Does the prompt explicitly mention dates / time / "when did this happen"?
- Does it show example facts with date phrasing?
- Is there a structured field for date in the JSON schema, or is everything
  free text?
- The April 2026 "Temporal relation fields" entry in TODO mentioned that
  "extraction prompt teaches the LLM to emit `valid_from`/`valid_to`
  ISO-8601 date fields when the conversation mentions time bounds." Confirm
  this is current and check whether the LLM actually does it on cat-2
  utterances.

### Step 5: Inspect the generation prompt (bench-side)

In `benchmarks/run_benchmark.py` look at `GENERATOR_USER_TEMPLATE`. It
shows the model only `fact_text` lines — no `valid_from`/`valid_to`,
no entity timeline. If extraction is correctly populating bitemporal
fields but the generator doesn't see them, that's a bench-side fix that
might also exist in production retrieval contexts.

## Output of the investigation

A short doc (`CAT2_TEMPORAL.md`, sibling to this one) with:

1. The 10 cases analyzed, labeled A-E
2. Distribution: e.g., "8 of 10 are mode C (extraction drops date entirely),
   2 are mode D (bitemporal captured, generator doesn't see it)"
3. Per-mode fix proposal:
   - **C** → revise extraction prompt to require explicit date preservation
     in fact text (or in a new field), with examples
   - **D** → expand the generator prompt to include bitemporal fields next
     to fact text, and verify production retrieval (memory_callback) does
     the same
   - **E** → larger redesign needed, may involve a separate "temporal fact"
     extraction pass
4. Recommended phased fix plan with effort estimates

## Estimated effort

- Investigation (steps 1-5 + writeup): **half a day, ~4 hours**
- The fix plan that follows: probably 3-5 days but contingent on what
  step 3 shows.

## Why not just guess and patch the prompt

Cheap to do, but the LLM sweep tells us specifically that prompt-following
strength is *not* the issue (Sonnet and Opus didn't help). If the fix is
"add a sentence about dates to the prompt," the sweep results would have
shown bigger Claude tiers helping. They didn't. That's evidence the issue
is structural — prompt edits alone may not move the needle. We need to know
what the failure mode actually is.

## Pre-baked queries (drop-in for the next session)

```bash
# Quick smoke: count cat-2 questions, count cat-2 questions with gen=0
python3 -c "
import json
d = json.load(open('~/datasets/locomo/data/locomo10.json'))
n_cat2 = sum(1 for c in d for qa in c.get('qa', []) if str(qa.get('category')) == '2')
print(f'cat-2 questions in dataset: {n_cat2}')
"

# Pull a Haiku snapshot for inspection (any *.db belongs to one model+conv;
# match against extracted_provider in metadata or just open and check facts
# table size for sanity)
ls -lt benchmarks/snapshots/*.db | head -5
sqlite3 benchmarks/snapshots/<hash>.db \
   "SELECT id, fact_text, category FROM memory_facts WHERE category='general' LIMIT 5"
```

## Do not start the implementation until this investigation runs

If the next session is tempted to just edit the extraction prompt and
re-run the sweep — that costs ~$5 and a few hours per attempt. The
investigation costs nothing and tells you whether the fix should target
the prompt, the schema, or the generator-side prompt.
