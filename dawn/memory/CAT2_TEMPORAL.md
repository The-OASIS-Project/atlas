# Cat-2 Temporal Extraction — Diagnosis & Phase 1 Result

**Status:** Phase 1 (L1 + L5) shipped 2026-05-05. Phase 2 (L2 / structured `event_when` field) re-scoped — see "Phase 1 result vs projection" below.

**Date:** 2026-05-05.

**TL;DR:** Cat-2 collapse was dominated by **lost dates at extraction time**, not retrieval / ranking / generator blindness. Two distinct failure modes drove ~95% of the loss: (1) **mode A** — relative phrases like "yesterday" / "this month" / "five years ago" couldn't be resolved because the extraction LLM never saw a conversation anchor date; (2) **mode C** — specific time-bounded events dropped during session-window condensation. **Phase 1 (anchor injection only) lifted cat-2 `recall_generation` from 0.022 → 0.321 (+29.9pp), exceeding the projected ~0.15-0.25 range and overall LoCoMo by +7.1pp (0.208 → 0.279).** L2's structured `event_when` field becomes a much smaller marginal win and is re-scoped accordingly.

---

## Phase 1 result vs projection

**Per-category `recall_generation` on LoCoMo, Haiku extraction + judge + generator + correctness (n=1982):**

| Category | May 4 baseline | L1 shipped | Δ | Projected |
|---|---|---|---|---|
| **cat-2 (temporal — target)** | **0.022** | **0.321** | **+29.9pp** | 0.15-0.25 (exceeded high end by +7pp) |
| cat-1 (single-hop) | 0.209 | 0.170 | −3.9pp | n/a (regression — investigate) |
| cat-3 (multi-hop inference) | 0.228 | 0.272 | +4.4pp | bonus |
| cat-4 (knowledge update) | 0.316 | 0.366 | +5.0pp | bonus |
| cat-5 (adversarial) | 0.135 | 0.155 | +2.0pp | bonus |
| **Overall** | **0.208** | **0.279** | **+7.1pp** | +5-10pp |

Cat-2 entailment also surged: 0.022 → 0.396 (+37.4pp). Reach effectively unchanged on cat-2 (0.771 → 0.762) — the lift is purely from extraction quality, not retrieval. Bench wall-clock 1h 54min, 272 fresh extractions, ~5900 LLM calls, 3 transient API errors (~0.05% — fail-closed; the +7.1pp is *despite* this noise).

**What Phase 1 actually shipped** (see the v42 schema migration commit and the diagnosis history below):
- `conversations.anchor_date INTEGER NOT NULL DEFAULT 0` (schema v42).
- Production writers populate at insert with `time(NULL)`; bench overrides per-session with `parse_locomo_session_date()`.
- Extraction prompt prepends `Conversation anchor: YYYY-MM-DD` when `anchor_date != ANCHOR_DATE_NONE`; the prompt instructs the LLM to resolve relative phrases against the anchor and preserve the original phrase verbatim in `fact_text`.
- `conv_db_get_anchor_date` (Layer 1) consumed by `memory_extraction.c` (Layer 2). `conv_db_force_anchor_date_unsafe(conv_id, user_id, anchor)` for bench overrides — authorization-scoped, OLOG_WARNING on every call.

**What changed in the Phase 2 calculus.** The diagnosis projected Phase 1+2 at cat-2 ~0.40-0.55. Phase 1 alone reached 0.321. The remaining ~10-20pp on cat-2 is the "mode C" event-coverage gap (events dropped during session condensation) — which the structured `event_when` field doesn't directly address; it just gives surviving events a structured place to land. L2 / L3 / L4 budget should be re-evaluated against (a) the cat-1 regression investigation, (b) whether mode C cases can be addressed by a smaller prompt-only event-coverage rev, and (c) whether the ~3-day Phase 2 spend is the highest-leverage memory work on the board now that cat-2 is no longer the bleeding edge.

**Cat-1 regression (−3.9pp) — known, not yet investigated.** Likely cause: anchor injection adds context that's irrelevant for atemporal factual recall. Worth a focused look (~½ day) before Phase 2 starts. Keep the lift; don't ship Phase 2 without understanding the trade.

---

## Original diagnosis follows. Failure-mode taxonomy and pre-Phase-1 case studies kept for reference.

**Pre-Phase-1 TL;DR (historical):** Cat-2 collapse is dominated by **lost dates at extraction time**, not by retrieval, ranking, or generator-side blindness. Two distinct failure modes drive ~95% of the loss: (1) **mode A** — relative phrases like "yesterday" / "this month" / "five years ago" can't be resolved because the extraction LLM never sees a conversation anchor date; (2) **mode C** — specific time-bounded events are dropped entirely during session-window condensation. Mode D (bitemporal captured but generator blind) is real but small (~0.6% of cat-2). The proposed fix is a two-part change in the extraction prompt and a small generator-side change.

---

## Method

Source: 10 Haiku-extraction snapshots from the May 4 2026 four-model sweep (`benchmarks/snapshots/`, mapped to `claude:claude-haiku-4-5` via `benchmarks/.claude/benchmark_logs/sweep_20260504_130448.log`). 321 cat-2 questions analyzed across 10 LoCoMo conversations.

Two passes:

1. **Statistical pass.** For every cat-2 question, mapped gold dia_ids → msg_ids via the snapshot's `dia_id`/`msg_id` map, pulled all facts whose `(source_msg_id_start, source_msg_id_end)` covers the gold msg_id, and checked whether the gold answer appears (normalized) in any covering fact's `fact_text`.
2. **Per-event pass.** For 10 sampled cases, ignored the provenance-window heuristic and instead grep'd the *entire* snapshot's `memory_facts` + `memory_relations` for the question's event keywords (e.g. "console" + "Jolene"). This is sharper because session-window extraction produces dozens of facts per range — most are unrelated to any one gold dia_id.

The per-event pass changed the distribution materially. The statistical pass overcounts mode-A because unrelated facts in the same window often happen to contain dates. The per-event pass shows that **the specific event the question asks about is frequently absent from the facts entirely** — mode C is much larger than the statistical pass suggested.

---

## Distribution (per-event pass; corrected)

| Mode | Description | Estimated share | Driver |
|---|---|---|---|
| **C** (event lost) | Question's specific event not present in any fact / relation | **~50%** | Extraction condenses ~30-60-turn sessions into ~10-30 facts. Specific time-bounded events drop. |
| **A1** (relative, no anchor) | Dialog says "yesterday" / "this month" / "five years ago"; extraction either drops the date or preserves the relative phrase verbatim | **~30%** | Extraction LLM has no anchor date. Generator can't resolve "this month" → "July 2023" either. |
| **A2** (session-implicit) | Dialog has no temporal phrase; gold derives from session metadata | **~15%** | Session date never reaches the extraction LLM. |
| **D** (bitemporal blind) | `memory_relations.valid_from`/`valid_to` populated, generator never sees them | **~1%** | Confirmed at the schema level (generator prompt only includes `fact_text`); rare in cat-2 because the relation predicate whitelist is stative-only. |
| **E** (reach=0) | Gold dia_id has no covering fact at all | **~1%** | 320 of 321 cat-2 questions had at least one covering fact, so this is rounding-error. |
| **B** (date in fact, generator misparse) | Gold appears verbatim in fact, but generator says wrong | **~3%** | Mostly cases where the fact has the date as a relative phrase. |

(Shares are approximate — mode C / A1 split depends on per-case judgment about whether a related-but-incomplete fact "covers" the event. The 10 hand-labeled cases below give the cleanest picture.)

---

## 10 cases analyzed

For each: question (Q), gold answer, evidence dia_id and the dialog turn it points to, the relevant facts found anywhere in the snapshot (event-keyword search, not provenance-window), and the assigned mode.

### Case 1 — conv 7 — **Mode C**
- **Q:** When did Jolene gift her partner a new console?  **Gold:** 17 August, 2023
- **Dialog [D19:2]** (session 12, 19 Aug 2023): *"I bought a console for my partner as a gift on the 17th and it's so much fun…"*
- **Facts on Jolene + console:** only `id=1634 "Jolene has a partner with whom she plays console games"`. The gift event itself — and the date — was not extracted.
- **Mode:** C. Date "the 17th" needs anchor (= August 2023 from session) to resolve. Both the event and the date were lost.

### Case 2 — conv 1 — **Mode A1**
- **Q:** When was Jon in Paris?  **Gold:** 28 January 2023
- **Dialog [D2:4]** (session 2, 29 Jan 2023): *"I've been to Paris yesterday!"*
- **Fact:** `id=201 "Jon recently traveled to Paris"`. "Yesterday" was lost; "recently" replaces it.
- **Mode:** A1. Anchor would let extraction emit "Jon traveled to Paris on 2023-01-28".

### Case 3 — conv 0 — **Mode A1**
- **Q:** When is Caroline going to the transgender conference?  **Gold:** July 2023
- **Dialog [D5:13]** (session 1, 3 Jul 2023): *"I'm going to a transgender conference this month."*
- **Fact:** `id=39 "Caroline is planning to attend a transgender conference this month"` — the relative phrase *survived verbatim*.
- **Mode:** A1. Generator sees "this month" with no anchor → can't answer "July 2023". This is the cleanest example of the leak: extraction did its job (preserved the temporal phrase), the system just has no anchor anywhere in the pipeline.

### Case 4 — conv 8 — **Mode A1 + partial C**
- **Q:** When will Evan and his partner have their honeymoon in Canada?  **Gold:** February 2024
- **Dialog [D23:23]** (session 1, 6 Jan 2024): *"We're off to Canada next month for our honeymoon."*
- **Facts:** Multiple "Evan recently traveled to Canada" facts; no honeymoon-with-date fact.
- **Mode:** A1. Even if the honeymoon were extracted with "next month", anchor (= January 2024) would resolve it to "February 2024".

### Case 5 — conv 4 — **Mode C**
- **Q:** What year did John start surfing?  **Gold:** 2018
- **Dialog [D3:27]** (session 4, 16 Jul 2023): *"I started surfing five years ago…"*
- **Facts on surfing + John:** **none found** in the entire snapshot.
- **Mode:** C. Surfing dropped during condensation. With anchor, "five years ago" from a 2023 session would resolve to 2018, but the topic itself was lost.

### Case 6 — conv 3 — **Mode C**
- **Q:** When is Joanna going to make Nate's ice cream for her family?  **Gold:** Weekend of 24 June, 2022
- **Dialog [D16:11]** (session 10, 24 Jun 2022): *"I'm going to make it for my family this weekend - can't wait!"*
- **Facts on ice cream:** several about recipes / diets, but no "Joanna making ice cream this weekend" fact.
- **Mode:** C. The event was condensed away; "this weekend" in dialog would have needed anchor anyway.

### Case 7 — conv 3 — **Mode A2**
- **Q:** When did Joanna start writing her third screenplay?  **Gold:** May 2022
- **Dialog [D12:13/14]** (session 7, 20 May 2022): conversation about a personal screenplay; **no date phrase in dialog at all**.
- **Facts:** screenplays are extracted (`id=571 "Completed first full screenplay last Friday"`, etc.) but none with a "started in May 2022" timestamp.
- **Mode:** A2. Gold derives entirely from session date. Anchor would let extraction emit "Joanna was working on her third screenplay around 2022-05-20".

### Case 8 — conv 3 — **Mode A2 (mild)**
- **Q:** Where did Joanna travel to in July 2022?  **Gold:** Woodhaven
- **Dialog [D17:4]** (session 11, 10 Jul 2022): *"I went to Woodhaven, a small town in the Midwest…"*
- **Fact:** `id=692 "Joanna visited Woodhaven, a small town in the Midwest for creative research"` (no date).
- **Relation:** `(Joanna -[visited]-> Woodhaven) [vf=0, vt=0]`.
- **Mode:** A2 in spirit. The "in July 2022" is a question-side filter, not strictly required in the fact. Generator can still answer "Woodhaven" if retrieval surfaces this fact, but if Joanna had multiple visits the temporal constraint would be needed to disambiguate. Demonstrates that even a clean visit-fact has no `valid_from`.

### Case 9 — conv 1 — **Mode C (gold-is-relative special case)**
- **Q:** When did Gina get her tattoo?  **Gold:** A few years ago
- **Dialog [D5:15]** (session 5, 8 Feb 2023): *"Got the tattoo a few years ago…"*
- **Facts on tattoo:** **none found**.
- **Mode:** C. Gold here happens to be the relative phrase verbatim — if extraction had simply preserved "Gina got her tattoo a few years ago" as a fact, the generator would have answered correctly. The event was just dropped.

### Case 10 — conv 7 — **Mode C + speaker mis-attribution**
- **Q:** When did Jolene's mom gift her a pendant?  **Gold:** in 2010
- **Dialog [D1:8]** (session 4, 23 Jan 2023): Jolene says *"This pendant reminds me of my mother, she gave it to me in 2010 in Paris."*
- **Fact:** `id=1616 "Deborah owns a pendant that reminds her of her mother"`. Two failures: (a) date "in 2010" dropped despite being absolute and explicit, (b) attributed to **Deborah** (the conversation's user) instead of Jolene (the assistant).
- **Mode:** C primarily; the speaker mis-attribution is a separate bug worth flagging.

---

## Bonus finding: speaker bias toward the conversation's "user"

LoCoMo conversations have two speakers. The bench's `run_locomo_memory` (run_benchmark.py:1219) assigns the first speaker `role=user` and the rest `role=assistant`. The extraction prompt (`memory_extraction.c:52`) opens with *"Analyze this conversation and extract user information"*. The LLM treats the user-role speaker as the canonical subject and underweights — or mis-attributes — events from the assistant-role speaker.

Case 10 is a clean example: Jolene's pendant (assistant turn) became a fact about Deborah (user). The cat-2 collapse is partly inflated by this — questions about the assistant-role speaker have a worse base extraction signal before any temporal issue is even considered.

This was not in the original A-E taxonomy. It's orthogonal to the temporal fix but worth tracking.

---

## What this means for the fix

The user's instinct — adding a temporal field with optional absolute or relative-resolved-against-anchor date — is the right destination, but the diagnosis sharpens the priority order:

| Layer | What it fixes | Modes addressed | Effort |
|---|---|---|---|
| **L1: Conversation anchor in extraction prompt** | LLM gets `Conversation anchor: YYYY-MM-DD HH:MM` line; can resolve "yesterday" / "next month" / "five years ago" against it | A1, A2 (~45% of cat-2) | ~½ day |
| **L2: Per-fact `event_when` (epoch INT64) + `event_when_precision` (enum)** | Schema v42 migration; extraction prompt asks LLM to populate it on time-bounded facts; `memory_embeddings.c:909` rewires from `created_at` to `event_when` | A1, A2, C (~80% of cat-2) | ~2-3 days (schema migration, prompt rev, extraction code, validation, scorer rewire) |
| **L3: Both bench AND production prompts include `event_when`** | Bench `GENERATOR_USER_TEMPLATE` renders `<text> (when: 2023-07-15)`; **production `memory_callback.c` does the same** so voice DAWN benefits | Closes the loop for L1+L2 in both contexts | ~½ day bench + ~½ day production |
| **L4: Extraction prompt event-coverage emphasis** | Add "for any specific event the speaker did or plans to do, emit a dedicated fact with a temporal field" | C (~50% of cat-2) | Folded into L2 prompt rev |
| **L5: Bench session-date plumbing via `conversations.anchor_date` column** | Add `conversations.anchor_date INTEGER NULL` (epoch seconds). Production `conv_db_create_*` populates with `time(NULL)` at insert; bench `engine.conv_create()` accepts and stores `session_X_date_time` (parsed via `parse_locomo_session_date()`). Extraction reads from the conversation row. *No* per-call param pollution, *no* `messages.created_at` override, no `add_message` protocol change | Required for benchmarking L1-L2 to actually work on LoCoMo | ~½ day |
| **L6: Generator sees relations + bitemporal fields** | `query_memory` returns matched relations; generator template renders them | D (~1% of cat-2) | ~1 day; trivial lift, do as a bundle item |

**Recommended phasing:**

- **Phase 1** (L1 + L5): plumb anchor date into extraction. Cheapest possible fix, addresses A1/A2. Estimate **~1 day** plus a half-day micro-benchmark (see Phase 1 prerequisites below). Re-run sweep, measure cat-2 lift.
- **Phase 2** (L2 + L3 + L4): add structured `event_when` field, prompt for it, generator surfaces it. Addresses C as well. Estimate **~3 days** plus Haiku-only re-extraction (~$1-2, ~1-2 hours). Full four-model re-sweep (~$5-15) only on positive Haiku lift.
- **Phase 3** (L6, optional): generator-side relations. Small absolute lift — bundle with another change. Estimate **~1 day**.

**Expected lift, very rough:**
- Phase 1 alone: cat-2 `recall_generation` 0.022 → ~0.15-0.25 (most A-mode cases become tractable; C still missing).
- Phase 1+2: cat-2 → ~0.40-0.55. The remaining gap is C-mode events that aren't extracted at all even with the new prompt — those are extraction-coverage problems beyond temporal.
- Phase 1+2 lifts overall LoCoMo `recall_generation` by ~5-10pp.

**Cat-2 has a structural ceiling around 0.60.** The L1 micro-benchmark sample showed ~22% of cat-2 questions have non-date gold answers — the question contains a temporal modifier ("what was Dave doing in October 2023") but the answer is not a date ("attending a car show"). These are functionally cat-1 / cat-3 questions mis-labeled as cat-2 in the dataset and are uninfluenceable by *any* temporal fix. Multiplied by the ~0.77 reach floor on cat-2, the realistic upper bound is `(1 − 0.22) × 0.77 ≈ 0.60`. Phase targets above should be read against that ceiling, not against 1.0.

### Phase 1 prerequisites (architecture-review fixes)

Both prerequisites have been cleared.

1. **L1 LLM-arithmetic micro-benchmark — RESULT: clears threshold (82.1% correct, 2.6% hallucination).**

   Ran 50 sampled cat-2 questions through `claude-haiku-4-5` with the anchor date injected and a strict ISO-output prompt. Out of 50 responses:
   - **100%** emit a parseable `DATE:` line (perfect output discipline).
   - **11** had non-date gold answers (cat-2 questions phrased as "what happened in May" / "where was X in July" — the question contains a temporal modifier but the answer is not a date). These are excluded from arithmetic scoring.
   - **39** had date-bearing gold. Of those:

     | Outcome | Count | Share |
     |---|---|---|
     | Exact ISO match | 27 | 69.2% |
     | Phrase-equivalent match (e.g. emitted `2023-07-14` for gold "the Friday before 15 July 2023") | 5 | 12.8% |
     | **Combined arithmetic correct** | **32** | **82.1%** |
     | Abstained with `DATE: UNKNOWN` (preferred over hallucinating) | 6 | 15.4% |
     | Genuine arithmetic failure (parseable but wrong) | 1 | 2.6% |

     The single hallucination: anchor 2023-08-02, gold "August 2023", emitted `2022-08`. Likely picked up an earlier "last August" reference from the dialog. **Mitigation: sanity-check that the emitted date is within ±2 years of the anchor; reject otherwise** (store NULL `event_when`, preserve relative phrase verbatim in `fact_text`). Catches the observed failure plus near-anchor-year confusion; legitimate "thirty years ago" claims fall back to phrase-only preservation, which is acceptable. ±2y was chosen over ±5y because LoCoMo's date span is ~5 years and anchor-year confusion is the dominant failure shape; over ±10y because tighter is cheaper to tune later if a real-world deployment shows different patterns. Implemented as a one-line check at extraction time using `iso8601_parse_date_utc()` plus an arithmetic delta against `conversations.anchor_date`.

   Architecture-review threshold was "<70% → L1 needs a different shape." 82.1% comfortably clears it. Abstain-vs-hallucinate ratio of 6:1 means failures are conservative (UNKNOWN), not deceptive — a structurally sound failure mode for downstream `event_when` storage where NULL is well-defined.

   Script: `/tmp/l1_microbench.py` + result `/tmp/l1_microbench.json` — kept locally, not checked in.

2. **L5 plumbing mechanism — `conversations.anchor_date` column.** Add `conversations.anchor_date INTEGER NULL` (epoch seconds) under the same v42 migration that L2 already requires. Production `conv_db_create_*` populates it with `time(NULL)` at insert time (one place — every existing call site is unchanged). Bench `engine.conv_create()` accepts an optional anchor parameter and stores the parsed `session_X_date_time`. Extraction (`memory_extraction.c`) reads `anchor_date` from the conversation row when building the prompt.

   This was originally specified as an optional parameter on `memory_trigger_extraction()` per the architecture-review picks. The column-on-conversation shape is cleaner: every production call site stays unchanged (no redundant `time(NULL)` pollution), multi-session conversations keep the anchor pinned to creation time rather than recomputing per-extraction, and the column is reusable by future features (e.g. session-relative reasoning, recurrence detection) without further plumbing. Same migration cost as the parameter approach (zero — folded into v42); ~30 extra lines of column-read code in extraction.

### L3 production parallel — NOT bench-only

The bench-side change (`benchmarks/run_benchmark.py` `GENERATOR_USER_TEMPLATE` ~line 521) does not by itself reach voice DAWN users. Production retrieval renders facts in `src/memory/memory_callback.c` (the `memory.recall` LLM tool callback). L3 must include both. Approximate effort: ~½ day bench + ~½ day production + verification that production tool-output token budget still fits under model limits with the new "(when: …)" suffix per fact.

### Phase 2: temporal-field semantics (resolves architecture-review C1)

DAWN already has two temporal fields on memory: `memory_facts.created_at` (today consumed by `memory_embeddings.c:909` for query-time temporal-proximity scoring) and `memory_relations.valid_from`/`valid_to` (bitemporal validity of stative relations). Adding `event_when` makes a third. The interaction must be settled before the schema migration is designed.

**Decision:**

| Field | Meaning post-L2 | Used for |
|---|---|---|
| `memory_facts.created_at` | Row insertion timestamp (unchanged from today). | Pruning, decay, "recently learned" UI ordering. **No longer used by temporal-query scoring.** |
| `memory_facts.event_when` (new) + `event_when_precision` (new enum: `year`/`month`/`day`/`datetime`) | Logical time of the event the fact describes. NULL = no temporal info known (atemporal preferences, extraction failures, and "no date in source utterance" all collapse here). | Temporal-proximity scoring in `memory_embeddings.c:909`. |
| `memory_relations.valid_from` / `valid_to` | Bitemporal validity of stative relations (unchanged). | `memory_db_relation_supersede` auto-close; `as_of` queries via the LLM tool. |

**Storage shape:** `event_when INTEGER NULL` (epoch seconds, mirrors `valid_from`); `event_when_precision TEXT NULL` constrained to {year, month, day, datetime}. NULL on both = "no info." Reuses `iso8601_parse_date_utc()` for validation/parse; explicitly out of the no-negative-returns rule per the convention already documented for `valid_from`.

**Phrase-preservation policy (required, not optional).** The L2 extraction prompt revision MUST preserve the original temporal phrase in `fact_text` whether or not `event_when` is populated. Today's extraction implicitly preserves "yesterday" / "this month" / "recently" / "a few years ago" in fact text; with `event_when` added, there is a real risk the prompt rev says "put dates only in `event_when`" and *removes* the phrase from fact text. That would be a regression — diagnosis Case 9 succeeds *only* because phrase preservation works today (gold = "a few years ago" verbatim). Make the policy explicit in the new prompt: *"Preserve the original temporal phrase in `fact_text`. Additionally, populate `event_when` with the resolved ISO date when determinable; leave NULL otherwise."* Both fields complement each other; neither replaces the other.

**Scorer rewire:** `memory_embeddings.c:909` switches from `created_at` to `event_when`. Facts with NULL `event_when` get **zero** temporal boost — additive, no penalty. Mirrors the April 2026 design intent ("additive so undated items forfeit the bonus rather than being penalized"). Net effects:
- Production: removes the silent-overload bug (`created_at` was never the right field for temporal-query proximity; it was extraction wall-clock).
- Bench: combined with L5 anchor plumbing, the bench finally scores against logical event time instead of bench run time.

**Schema migration:** v41 → v42. Standard pattern in `auth_db_core.c`. Initial backfill is NULL on all existing rows; LLM-driven batch backfill is a separate optional follow-up under the same Phase 2 budget.

**Known design limits (accepted, not fixed in Phase 2):**

- *Redundancy with `valid_from`*: when extraction creates a fact and a paired exclusive relation in the same pass (e.g., "I started at Google in 2018"), `event_when` and the relation's `valid_from` carry the same date. Mitigation: in `process_extraction_response` (`memory_extraction.c`), when `find_fact_for_relation` matches a fact to a relation with `valid_from`, copy `valid_from` → `event_when` rather than trusting the LLM to emit them consistently. Documented divergence is acceptable; silent divergence is not.
- *Ranged events*: `event_when` is single-point. "I lived in Tokyo 2018-2020" stores `event_when=2018-01-01, precision=year` and the end is preserved only in `fact_text`. Adding `event_when_end` is a future enhancement if data shows it matters.
- *Query-type asymmetry*: "what happened last week" (event time, served by `event_when`) works post-fix; "what did we discuss last week" (statement time) loses the temporal signal on the fact scorer. The latter is better served by conversation-history search anyway. Documented limit.

### Speaker mis-attribution (separate workstream — not in temporal fix budget)

Not blocked by temporal work, not part of the L1-L6 effort total. Tracked separately. The extraction prompt's "extract user information" framing biases toward the conversation's first speaker; on LoCoMo (and any multi-party conversation) this drops or mis-attributes events from the non-canonical speaker. Likely needs the extraction prompt to stop treating the conversation as having a single canonical subject. ~2-3 days, separate cycle.

### Phase 1 success measurement

Before declaring Phase 1 done, re-classify a 30-case sample of post-fix cat-2 misses using the same A1/A2/B/C/D/E taxonomy as this diagnosis. Goal: confirm A-mode share dropped (the population L1 was supposed to address) and identify whether C-mode share grew or stayed put as a fraction of the new misses. This becomes the input to Phase 2 priority.

---

## What we ruled out

- **Retrieval is not the problem.** 320 of 321 cat-2 questions have at least one covering fact (effective reach is ~99% by this measure; the bench's 0.77 number reflects a stricter metric).
- **Bigger Claude tier doesn't help.** Sonnet, Opus, and local Qwen all sit at 0.02-0.03 cat-2 `recall_generation`. The schema is the ceiling.
- **The relation bitemporal slot is not where dates are landing.** Only 69 of 321 cases (~21%) have any relation with `valid_from`/`valid_to` populated, and even those typically aren't the cat-2 event in question — they're stative relations like `works_at`. Mode D is rare.

---

## Caveats

- The "reach=1, gen=0" filter from the plan was not strictly applied. The bench's `correctness.json` cache key is hashed against the actual retrieved fact list, which would have required replaying retrieval on each snapshot. Given that ~320 of 321 cat-2 questions have effective reach of 1, sampling broadly across cat-2 was sufficient for failure-mode classification.
- Mode-share percentages above are approximate. The bright line between "C — event lost" and "A1 — present in fact, anchor missing" depends on whether a related-but-different fact counts as covering the event. The 10 hand-labeled cases are the source of truth; the 321-question statistical pass is calibration.
- Snapshots reflect the May 2026 production-default extraction (`claude:claude-haiku-4-5`, embedding `bge-small-en-v1.5-int8`, schema v41). Re-extraction on a model+prompt change invalidates these conclusions.
