# Dynamic Context Injection — Phase 1

**Phase 1 of Dynamic Context Injection** (April-May 2026, closed 2026-05-09).

This document is the historical record of Phase 1: what shipped, what didn't, the architectural decisions worth preserving, and the load-bearing invariants future contributors should preserve. Companion docs: [STATE.md](STATE.md) for current memory-subsystem posture, [CAT2_TEMPORAL.md](CAT2_TEMPORAL.md) and [EMBEDDING_UPGRADE.md](EMBEDDING_UPGRADE.md) for adjacent context.

---

## North star

DAWN's memory was already strong on retrieval (LongMemEval 97.0% R@5, ConvoMem 99.0%, LoCoMo 81.6%) and reasonably strong on extraction (Cat-2 79.9% post-anchor-injection). But the assistant *felt* less competent than the numbers implied because the LLM never saw the right facts on the right turns.

The diagnostic instance was sharp: ask DAWN about your stocks, and the response would be off-topic ("you've been evaluating GPUs for local inference") because the focus block — DAWN's only auto-injection of memory at session start — was time-bounded (recent summaries ≤ 30 days, top-N facts ≥ 0.5 confidence) and did not refresh per-turn. The user's full memory was queryable via tool calls but never appeared automatically; the assistant could find facts only if the LLM thought to call `memory_action_search`. The structured graph DAWN spent months building (entities, bitemporal relations, contradiction supersedes) was effectively pull-only.

Phase 1 closes that gap. Per-turn focus injection composes a ranked + filter-checked + dedup'd block from registered adapters and prepends it to the LLM system prompt every user turn. The user can see what was injected via a trust-tiered transcript surface gated behind the existing debug toggle. Tuning lifted the synthetic Component 6 fix-rate from 7/11 to 11/11; real-DB validation against the developer's 6+-week-old store (after a fresh re-extraction under the current pipeline) showed kids/wife/mother/stocks all surfacing correctly on family + stock queries.

---

## Architectural foundations

### Three-layer context model

```
┌──────────────────────────────────────────────────────┐
│ Layer 1 — Persistent (session-start, refresh-driven) │
│   Preferences, top-N high-confidence facts, recent   │
│   summaries.  Built by `memory_build_context`.       │
│   Refresh: settings change, tool change, satellite   │
│   rebind, HUD discovery.                             │
├──────────────────────────────────────────────────────┤
│ Layer 2 — Per-turn focus (NEW, Phase 1)              │
│   8 candidates max, ranked + filter-checked +        │
│   dedup'd, scored across 4 weighted contributions.   │
│   Composed by `focus_compose` → `build_focus_block`. │
│   Refresh: every user turn.                          │
├──────────────────────────────────────────────────────┤
│ Layer 3 — Pull-only via tool calls                   │
│   Full memory_action_search, document RAG, calendar  │
│   range queries, email search.  LLM-initiated.       │
└──────────────────────────────────────────────────────┘
```

Phase 1 added Layer 2. Layer 1 and Layer 3 unchanged.

### Focus framework + adapter pattern

`include/memory/focus_source.h` defines a typed adapter contract. Each candidate context source registers a `focus_source_adapter_t` at daemon init:

```c
typedef struct {
   const char *source_id;          // stable static string
   focus_source_type_t source_type; // INTERNAL | EXTERNAL | USER_CONTENT
   bool requires_embedding;         // skip if query_embedding == NULL
   focus_source_query_fn query;     // adapter callback
} focus_source_adapter_t;
```

Adapters return `focus_candidate_t[]` with three score components (semantic / recency / importance) plus provenance. The framework then:

1. Runs unconditional `memory_filter_check()` on every candidate's text — defense in depth, even if the adapter pre-filters
2. Computes `final_score = w_sem*sem + w_rec*rec + w_imp*imp + w_src*source_weight(source_id)`
3. Ranks descending, ties broken by `item_timestamp`
4. Drops below `min_score`, trims to `top_k`, truncates to `focus_budget_tokens`
5. Returns the surviving set

Six adapters registered in production at Phase 1 close: memory_fact, memory_entity, memory_relation, memory_summary, document_chunk, calendar_event. Email focus adapter deferred to v2 (gated on building an email message cache; current email subsystem is live-network-only).

### Parallel `score_breakdowns` array on `focus_compose_result_t`

Phase 1g-i needed the four already-weighted contributions (semantic / recency / importance / source) plus `final_score` and `applied_source_weight` per candidate for the WebUI score-breakdown popover. The naive shape — extending `focus_candidate_t` — would have violated the adapter contract: adapters allocate that struct, framework doesn't own it. Adding fields would silently grow every adapter's `calloc` and have the framework write into adapter-allocated memory.

Resolution: parallel array on `focus_compose_result_t`:

```c
typedef struct {
   float semantic_contribution;
   float recency_contribution;
   float importance_contribution;
   float source_contribution;
   float final_score;
   float applied_source_weight;
} focus_score_breakdown_t;

typedef struct {
   focus_candidate_t *candidates;             // adapter-owned shape
   int candidate_count;
   focus_score_breakdown_t *score_breakdowns; // framework-owned, parallel
   focus_filter_rejection_t rejections[MAX_FOCUS_SOURCES];
   int rejection_count;
} focus_compose_result_t;
```

`apply_dedup_locked` compacts both arrays in lockstep when admitting/dropping candidates. A latent 1f score-uplift bug (reading `c->semantic_score` as the uplift base instead of the rank-determining composite) was fixed in the same diff: the bench now reads `result->score_breakdowns[i].final_score`.

### Cross-turn dedup state

`session_t.injected_set` is a 256-entry fixed-array LRU tracking last-injected turn + last score per `(source_id, item_id)`. On each turn, candidates are admitted iff:

```
(turns_since_last > recent_window_turns AND current_score >= min_score)
   OR (current_score >= prior.last_score * score_uplift_factor)
```

Strict `>` (NOT `>=`): at `recent_window_turns = 0`, time-based dedup is effectively disabled and uplift becomes the only re-injection path. Documented in `dawn.toml.example`.

The injected_set shares `session->history_mutex` (no new lock). 8 SESSION_START clear sites (6 webui + 2 in session_manager). Per-session warning gates (`eviction_logged_once`, `all_suppressed_logged_once`) substitute for a kill-switch knob — operationally observable without a config flag.

### Three-gate WebSocket broadcast routing

`webui_broadcast_context_injection` filters candidate connections by three gates:

```c
if (conn->session->type != SESSION_TYPE_WEBUI) continue;        // gate 1
if (conn->auth_user_id != user_id) continue;                    // gate 2
if (webui_get_active_conversation_id(conn->session) != conv_id) // gate 3
   continue;
```

Gate 1 is mandatory because `dawn_build_prompt` runs for every session type (LOCAL / DAP / DAP2 / WEBUI); only WEBUI sessions have a `client_data` cast-able to `ws_connection_t *`. Gate 2 + gate 3 together implement per-`(user, conversation)` fan-out — two browser tabs on the same conversation both receive the event; a different conversation in another tab does not. A different user's tab does not.

**Multi-tab semantics matter.** If only the dispatching tab received the event, the second tab's transcript would silently desync from the first — visible asymmetry that breaks the trust-transparency invariant.

### Pre-flight registry scan optimization

When focus injection is enabled but no WebUI session is active for the user (e.g., user logged out mid-conversation, or local-session-only), the broadcast helper builds a full JSON payload, then iterates the connection registry, finds zero matches, and discards the JSON. Wasted allocation per turn.

Phase 1j.B added `any_session_matches(user_id, conv_id_or_zero)` — a cheap pre-flight scan under the registry mutex. If no recipient matches, return early before JSON construction. Applied to `webui_broadcast_silent_observation` and `webui_broadcast_context_injection`. **Deliberately NOT applied** to `scheduler_broadcast_notification` because that function inserts a `missed_notif` DB row inside the registry mutex critical section when `sent == 0` — a naive pre-flight skip would silently drop missed-alarm rows.

### Body-class CSS gate (Phase 1h)

Originally the focus-injection block + filter-warning chip rendered on every turn whenever the feature was enabled. Real-use feedback: noisy. Per-turn block on every conversation turn fatigues the eye even when the surfaced items are good.

Resolution: hide the focus-injection UI by default; reveal under the existing `body.debug-mode` class (toggled by the existing `#debug-btn` at `index.html:247`). Same toggle that already gates tool-call-detail rendering. Pure CSS-only render gate — JS unchanged, retroactive on already-rendered blocks (CSS recomputes), no live-update wiring needed.

`#warning-chip-rail` is shared with silent-observe's chip, which stays always-on (Phase 0 contract). Only `#context-injection-warning-chip-anchor` is gated.

### Trust-tiered defaults

`focus_source_type_t` enum drives default expansion:

| `source_type` | UI default | Rationale |
|---|---|---|
| `internal` (memory facts/entities/relations/summaries) | Collapsed under `dawn-badge.muted` cluster summary "N internal items" | High-trust, low-novelty — DAWN extracted these from the user's own conversations |
| `external` (document chunks, calendar events) | One-line preview with native `<details><summary>` disclosure | Medium-trust — documents the user uploaded, calendar the user maintains |
| `user-content` (email body, future feeds) | One-line preview, monospace (`font-family: 'IBM Plex Mono'`) | **Designated potentially-malicious** — content the user did not author, may contain prompt injection. Monospace signals "treat as data, not formatted text" |

Email adapter is deferred to v2; the `user-content` tier is wired and ready.

---

## Phase-by-phase ledger

### Phase 0 — Silent-Acknowledge Primitive (April 2026)

Foundation for both Phase 1 and the future Phase 2 background-context workstream. New `LLM_CALL_SILENT_OBSERVE` call mode + audit-log tagging + a persistent UI counter / peek popup with a filter-match warning chip. Phase 1's WebUI block reuses Phase 0's chip-rail mount and dismissal semantics.

### Phase 1a — API audit + cohesion-debt split + legacy filter scan (May 2026)

Pre-implementation audit verified the rev-2 source manifest against HEAD before adapters built. Header split: `memory_db.h` umbrella into `memory_db_entities.h` + `memory_db_embeddings.h` + `memory_types.h`. Re-exports preserve external compile compatibility. Legacy filter scan ran against the production DB — found 21 false-positive matches in legacy data, all benign-but-flagged ("you should", "system prompt" in technical content); coverage gap closed without auto-cleanup.

### Phase 1b — Source adapter framework (May 2026)

`focus_source.{c,h}` defines the adapter contract + framework-owned ranking + filter-on-retrieval. ~600 LOC at Layer 2. No adapter implementations yet — adapters land in 1c/1d.

### Phase 1c — Memory adapters (facts, entities, relations, summaries) (May 2026)

Four memory-side adapters wrapping `memory_fact_search` (extracted into its own file `memory_fact_search.{c,h}`), `memory_db_entity_search`, `memory_db_relation_list_by_subject_at`, `memory_db_summary_search_since`. Defense-in-depth user_id post-check fold-in: `memory_db_fact_get` was not user-scoped at the SQL layer.

### Phase 1d — Document + calendar adapters (May 2026)

`document_focus_adapter.{c}` over `document_db_chunk_search_load`; `calendar_focus_adapter.{c}` over `calendar_db_occurrences_*`. Email focus adapter intended for 1d but deferred to v2 after API audit revealed: `email_db` has only an `email_accounts` table — no message-level cache. The current email subsystem fetches messages live via IMAP / Gmail REST per-call. Building a focus adapter against that path would violate the cache-only invariant the bench architecture depends on. v2 prerequisite: an `email_messages` table with retention policy + double-write from `email_service_recent` callers.

Shared scaffolding: `focus_recency.{c,h}` (Gaussian recency decay), `focus_candidate_helpers.{c,h}` (allocation + cleanup discipline used by all four adapters).

### Phase 1e — Single prompt builder collapse + Layer 4 placement (May 2026)

`build_focus_block` at Layer 4 in `src/webui/build_focus_block.c` is the sole consumer of `focus_compose`. Composes the ranked result into a multi-line `[<source_id>] <text>\n` block prepended to the system prompt under `--- TURN CONTEXT ---` markers. Adapter `register_all` wired at daemon init.

### Phase 1f — Cross-turn dedup state (April-May 2026)

256-entry `injected_set_t` per session (see "Cross-turn dedup state" above). 14 dedup tests + TSan concurrent stress (4 PER_TURN + 2 SESSION_START × 100 iter). Asymmetric `_locked` mutex API. Once-per-session warning gates substitute for kill-switch.

### Phase 1g-i — WebSocket `context_injection` event + score breakdown (May 2026)

Wire format flat at root (NOT under `payload` like silent_observation — consumer foot-gun pinned). Parallel `score_breakdowns` array on `focus_compose_result_t`. Three-gate broadcast routing. Latent 1f score-uplift correctness fix folded same diff: `apply_dedup_locked` reads `result->score_breakdowns[i].final_score` (the rank-determining composite), not `c->semantic_score` (raw cosine). New `session_get_last_user_msg_id` accessor for stable `turn_id`.

### Phase 1g-ii — WebUI consumer (May 2026)

`www/js/ui/context-injection.js` (713 LOC after fix-pass). Trust-tiered transcript blocks. Filter-match warning chip (statically mounted in `index.html` after H5 fix-pass; lazy `<body>` mount initially). "Tell me more" modal with score-breakdown bar chart + provenance back-link via `DawnHistory.loadConversation` (H1 fix corrected the non-existent `requestSelectConversation`). 50-block FIFO transcript cap (H3). Reset wired on conversation-switch AND `handleContextCompacted` (H2 stale-provenance race). DOMPurify on every untrusted string field. Document-level Esc handler + Tab focus-trap on modal (C1 a11y blocker). Empty-state collapses to a single muted line on every empty turn (H4 — full block was visual chaff).

Four-agent review: 1 Critical (modal keyboard trap) + 7 Highs + 6 Mediums folded in fix-pass before commit.

### Phase 1h — Body-class CSS gate (May 2026)

Pure CSS-only render gate (~33 LOC). Hooks into existing `#debug-btn` toggle. Two new dedup knobs (`focus_dedup_recent_window_turns`, `focus_dedup_score_uplift_factor`) added to settings panel `focus_injection_settings` group. No C touched.

### Phase 1i — Bench probe scaffolding (May 2026)

**1i.A:** Multi-model harness factored to `multi_model_probe.py` (`bench_speaker_attribution.py` shrinks 1621 → 1456 LOC, under soft warning).

**1i.B:** `tests/test_focus_probe.c` — 7 deterministic Unity tests covering recency / importance / source-weight / dedup-suppress / dedup-uplift / budget force-keep-first-only / combo. Fake-adapter pattern via `focus_register_source` + programmable seed pool. CI 49 → 50.

**1i.C:** `benchmarks/bench_focus_pipeline.c` (909 LOC) wraps `build_focus_block` against fake-adapter synthetic memory pool. FNV-1a + xorshift deterministic 384-dim embeddings (`semantic_override` opt-out). Multi-model Python probe `bench_focus_fix_rate.py` (465 LOC) with `--dry-run` / `--recalibrate` / `--cases` flags. 10 synthetic cases — no real PII (architectural invariant C2). Fix-pass remediation: `dedup_suppressed` log-capture, `broadcast_fired` field for `conv_id <= 0` disambiguation, JSON config field clamps, Y2K38 `int64_t` swap.

### Phase 1j — Fixture hardening + weight tuning (May 2026)

**1j.A:** v1 cases (2-3 candidates each, saturated 10/10 baseline) replaced with v2: 11 cases at 9-11 candidates each, partial-match competition with `semantic_override` + `recency_override` pinned per-seed. 4 designed-FAIL + 6 designed-PASS + 1 negative-empty. Recalibrated baseline 7/11 across all three providers — within design's 4-8/10 movable window.

**1j.B:** Single targeted iteration. `weight_importance` 0.2 → 1.0 + `weight_recency` 0.3 → 0.15. Component 6 fix-rate **7/11 → 11/11** across anthropic / openai / local. LoCoMo retrieval byte-identical pre/post (zero regression — `bench_retrieval` doesn't exercise focus_compose). Common cause behind all four designed-FAIL pathologies (semantic-noise ×2, recency-collision, source-weight-burial): `w_imp = 0.2` was systematically too low to let high-importance items overcome the natural per-item dispersion in `semantic_score`. Pre-flight registry scan landed in same diff. ~$0.10 spend (4 of 5 paid-recalibrate budget unused). **Closes Phase 1.**

---

## Reextract utility — permanent infrastructure shipped same day

After 1j.B closed Phase 1, manual real-DB testing surfaced an unexpected outcome: the synthetic 11/11 win didn't generalize. The user's family query surfaced meta-facts (`User asked 'what is the LLM?'`) and tangential items (genealogy, calendars) instead of kids names; the entity graph showed duplicate `Jon` / `Jonathan Smith` person entities with parallel `works_at Netsurion` relations.

Root cause: the user's DB carried derived state from before April 2026 quality improvements (subject-naming fix, anchor injection, categorization, injection filter, current importance heuristics). Embeddings had been recomputed under bge-small-int8, but the underlying fact text + importance values + extraction-prompt artifacts were locked in from when each fact was originally extracted.

The fix was not Phase 1 weight tuning. It was a clean derivative-state reset under the current pipeline.

`dawn-admin memory reextract --user <name>` ships this as first-class admin infrastructure:

- **Drops 5 derived tables** (facts / preferences / summaries / entities / relations) for the user in a single `BEGIN IMMEDIATE` transaction
- **Resets per-user backfill flags** (`users.categories_backfilled_at = 0`, `users.embeddings_model_id = NULL`) so the post-extraction backfill workers re-run cleanly
- **Atomic backup before reset** via `auth_db_backup()` (existing `/var/lib/dawn/` + `/tmp/` allowlist)
- **Detached recovery-worker spawn** matching `memory_recategorize_start`'s pattern — admin handler returns immediately, worker grinds in the background
- **Concurrency guards** refuse if `memory_recategorize_is_running()` or `memory_extraction_in_progress(user_id)` is true
- **Cost estimator** uses the real `EXTRACTION_PROMPT_TEMPLATE` size (~4744 bytes, exposed via `memory_extraction_get_template_size_chars()` — not a hardcoded guess)
- **Default `--dry-run`** shows the plan + cost estimate; `--confirm` required for execution

Wire protocol: `ADMIN_MSG_MEMORY_REEXTRACT = 0x81`, `ADMIN_MSG_MEMORY_REEXTRACT_STATUS = 0x82`. Binary payload ≤240 bytes within the 256-byte cap (length-prefixed strings + flags + max-cost-micros).

Live-validated against the developer's `/var/lib/dawn/auth.db`: 272 conversations / 3052 messages / ~$1 actual spend on Haiku, ~80 minute wall-clock. After completion, real-DB family + stock queries surfaced kids names + NVDA/AMD/MRVL stock-tracking facts on the first focus-injected turn. Phase 1's tuning generalized cleanly to clean derivative state.

Same-day hot patch fixed the cost estimator: it had been querying the *pre-reset* eligible-for-recovery state (conversations where `last_extracted_msg_count < message_count`, ~2 convs / $0.005) instead of the *post-reset* eligibility predicate (every non-private conv with `message_count >= 2`, ~272 convs / ~$1). Misled the user into confirming with a 100x-low estimate. Estimator now mirrors the post-reset predicate exactly.

The reextract utility is **permanent infrastructure**. Future major memory-subsystem upgrades (extraction-prompt revs, schema additions that need backfill, importance-heuristic improvements) can use it to surface a controlled reset path. Conversations + messages are source of truth; everything derived is regenerable.

Carried debt: cost-spent in `reextract-status` reports "not tracked" because the recovery worker doesn't tag its LLM calls in `auth_db_metrics` with an operation label. Fix shape: add `operation_tag TEXT` to `session_metrics`, plumb a tag like `"reextract_<user>_<timestamp>"` through the recovery worker's LLM call path. ~1-2 hours, deferred.

---

## Architectural invariants (PIN — preserved across Phase 1, future contributors should preserve)

1. **DAWN's persistent memory is local-only.** Cloud LLMs see transient per-turn snapshots; no provider-hosted persistent storage features (Anthropic Files API, OpenAI Assistants threads, fine-tuning datasets, provider memory features). The 30-day inference retention typical of Anthropic/OpenAI is acceptable; persistent indexed storage in a provider's database is not. Future contributors: bump against this line if you propose violating it.

2. **Conversations + messages are source of truth.** Everything derived (facts, entities, relations, summaries, preferences, embeddings) is regenerable. `memory_db_admin.c::memory_db_reset_derived` operationalizes this: drops derived state, leaves source untouched. Schema additions or extraction-pipeline changes that violate this invariant should land alongside an explicit migration story.

3. **Adapter struct contract is sacrosanct.** Adapters allocate `focus_candidate_t` arrays. The framework owns parallel arrays on `focus_compose_result_t` (e.g., `score_breakdowns`). Don't extend `focus_candidate_t` to carry framework-computed values.

4. **Filter enforcement at retrieval is unconditional.** The framework runs `memory_filter_check()` on every adapter-returned candidate's text before ranking. Adapters MAY pre-filter, but correctness must not depend on adapter discipline. Defense in depth across the three gates: extraction-time, storage-time, retrieval-time.

5. **Synthetic only in benches.** Bench fixtures (`focus_probe_cases.json`) MUST contain only synthetic content shaped like real failures, never real names / tickers / projects from a developer's actual memory. Cloud LLMs see bench fixtures during `--recalibrate`; real PII would cross the privacy boundary.

6. **Per-`(user, conv)` broadcast fan-out, never per-session.** Multi-tab semantics depend on this. Single-session broadcast silently desyncs sibling tabs.

7. **Pre-flight registry scan applies to side-effect-free broadcasters only.** `scheduler_broadcast_notification` inserts `missed_notif` DB rows when no recipient matches; naive optimization would drop missed-alarm rows. Verify side-effect-freedom before applying the optimization to any future broadcaster.

---

## Out-of-scope at Phase 1 close

Items considered, explicitly deferred:

- **Phase 1k — real-DB structured probe.** Originally planned as the post-tuning measurement against real failure shapes. Manual smoke after `dawn-admin memory reextract` proved sufficient — kids/wife/mother/stocks all surface correctly. Probe stays optional, unbuilt. If future drift becomes observable, the probe shape is documented (extend `bench_focus_fix_rate.py` with a `--real-db --local-only` mode that reads `auth.db` directly, queries from gitignored personal-failure file, sends to local Qwen for verdict).

- **Email focus adapter.** Deferred to v2; gated on building an email message cache (schema migration + IMAP/Gmail polling worker + retention policy + double-write from existing `email_service_recent` / `_search` callers). Roughly 3-4 days of dedicated work. The `user-content` source_type tier is wired and ready to receive email content once the cache exists.

- **`dedup_should_suppress` v3 case.** Component 6 v2 fixtures don't include a "same item from prior turn → should NOT surface" case because the bench's stdin schema doesn't yet plumb `session_state` (prior-injection set). The reserved field is in the wire format; extension would add `session_state.injected_set` array and the bench would seed that into the synthetic session_t before invoking `build_focus_block`. ~½ day of work; defer until needed.

- **Bench-wiring focus injection through `bench_memory_pipeline.c`.** The existing LoCoMo bench (`bench_memory_pipeline.c` + `run_benchmark.py`) measures `memory_action_search` retrieval, not focus injection. Building a focus-aware LoCoMo harness would let us measure how focus injection affects answer quality across the full LoCoMo dataset. ~1-2 days of new bench infrastructure for marginal value over Component 6 — deferred.

- **Phase 2 — DAWN background context.** Silent-observe events fed into a sibling memory store (separate from the user's facts/entities), per-day conversation_id, nightly compaction. Sketch preserved below ([Phase 2 — DAWN background context (sketch)](#phase-2--dawn-background-context-sketch)); full design pass deferred.

---

## Outstanding work surfaced post-close

Real-DB validation surfaced three classes of imperfection that are NOT focus-injection bugs but downstream-of-extraction issues:

1. **Meta-facts in memory.** Extraction creates personal-facts entries that are interaction-pattern artifacts: `User asked 'what is the LLM?'`, `User requests news and political information from their personal assistant`, `User inquired about gold price forecasts`. Surface noise on retrieval. Tracked in [STATE.md](STATE.md) as **"Extraction-prompt audit"** short-term item. ~½ day investigation + variable fix.

2. **Paraphrase duplicates.** Hash-based dedup catches identical text only; semantic paraphrases co-exist (`User has 3 children` + `Jon has 3 kids`, two `User's mother's birthday is February 16th` entries with different annotations). Tracked in [STATE.md](STATE.md) as **"Paraphrase dedup at extraction"** short-term item. ~1 day. Embedding-similarity gate before storage — if cosine > threshold (e.g., 0.92) to existing fact, merge or skip.

3. **Entity duplication.** `Jon` and `Jonathan Smith` stored as separate person entities; `user` (thing) and the canonical person entity coexist with parallel relations. User-visible at every memory tool call. **Highest-priority remaining cleanup** because it compounds across every retrieval. Tracked in [STATE.md](STATE.md) as **"Entity merge / user-identity dedup"** — promoted to top of medium-term queue 2026-05-09. ~2-3 days. Name-similarity + type-match merge pass at extraction time + retroactive merge tool exposed via `dawn-admin`.

4. **Filter false-positives on documents** (separable, longstanding). Per-turn focus injection runs the injection filter against doc_chunk text every turn; technical documents containing `'system prompt'`, `'api key'`, `'base64,'`, etc. as legitimate content trigger warning chips on every turn. Tracked in [STATE.md](STATE.md) as **"Memory injection filter: weighted scoring"**. The chip surface makes this user-visible in a way it wasn't before; promotion-pressure has increased.

---

## Phase 2 — DAWN background context (sketch)

Preliminary design preserved from the rev-3 working draft (folded in here at that
draft's retirement). The full design pass is **deferred to its own session**; the
decisions below are resolved starting points, not a final spec.

**Architecture:** a dedicated per-day `conversation_id` for DAWN's own observations
(or a sibling table — choice deferred, but lean toward a sibling table per
architecture review L3, to avoid widening the conversations table).

**Logged events** (filter-gated for user-influenceable inputs per security review H2):

| Event | User-influenceable? | Filter check required |
|---|---|---|
| Notification fired | yes (SMS, email body) | yes |
| Calendar event passed | yes (CalDAV title/description) | yes |
| Scheduled task executed | yes (user-set title/notes) | yes |
| New conversation started | indirect | yes |
| Conversation compacted | indirect (LLM summary) | yes |
| MQTT events of significance | broker-trusted; treat as user-controlled | yes |
| Music track played / playlist changed | partially (tag metadata) | yes |
| Errors observed | system-generated | no |
| Satellite registered / went offline | partially (registration name) | yes |
| HUD discovery | partially (component name) | yes |

Every user-influenceable event passes through `memory_filter_check()` before the
silent-observe call. Filter-match events are still logged (with rejection metadata)
— the offending text is replaced by a placeholder so the event is *recorded* for
DAWN's awareness without polluting context.

**Compaction:** nightly at 3am local time, **idle-gated**. "Idle" across all surfaces:

- WebUI sessions: no user message in the last 30 minutes
- Tier 1 satellites: not currently recording / playing audio
- Tier 2 satellites: not in an active push-to-talk session
- MQTT: no user-attributed activity (commands from a satellite or local mic)
- Voice always-on: no wake-word-followed-by-speech in the last 30 minutes

If any surface is active at 3am, compaction defers to the next 30-minute idle window
(re-check on a 5-minute timer, hard upper bound 4am local — past that, defer the
whole compaction to next night). Configurable via `[memory.dawn_background]
compaction_idle_minutes = 30`.

The LCM substrate handles the context-rolling: yesterday's full background-context
conversation is summarized, and the summary seeds the new day's conversation
("Yesterday: [compacted bullet list]"). The old day's full context is retained in the
`summary_nodes` DAG for drill-down via `context_expand`.

**Surfacing into active chats (Layer 3 / anticipatory):** background context becomes
one source for the Phase 1 framework. Adapter shape:

- `requires_embedding = false`; query input is current time + recent-turn embedding
- emits candidates for time-relevant items (calendar event approaching, scheduled task firing)
- multi-chat dedup via a `surfaced_items_set`
- anticipatory firing also triggers `PROMPT_REFRESH_SESSION_START` (Layer 1 invalidation, per architecture review H3)

**UI surface — DAWN Background Context Viewer** (per UI review M1): a two-pane layout
matching the existing memory panel — left pane a vertical timeline (today expanded;
older collapsed by date with a `summary_nodes` preview, plus a filter chip bar and
search), right pane the detail view of the selected observation (provenance link,
"open source conversation", reusing `memory-source-modal` styling). Filed as its own
top-level rail icon (sibling to memory) — a viewing-not-configuring surface.

---

## Sources

- Rev 3 design doc (`dawn/docs/DYNAMIC_CONTEXT_INJECTION_DESIGN.md`) — working / scratch document, untracked per project policy. Its Phase-1 reality is captured in this atlas record and its Phase-2 sketch is folded in above; the draft is retired (removed at Phase 1 close).
- Per-phase architecture-reviewer pre-dispatch reviews — caught Criticals on every phase prompt before worker dispatch (verified value across 1c, 1d, 1e, 1f, 1g-i, 1g-ii, 1h, 1i, 1j.A, 1j.B, reextract-utility). The pattern is now standard practice for any non-trivial worker dispatch in this codebase.
- Four-agent end-of-phase reviews (architecture / embedded / security / coding-standards, plus ui-design-architect for UI-touching phases). Findings + remediation captured in commit messages.
- Phase 1g-ii fix-pass commit — the most-comprehensive single fix-pass in the workstream (1 Critical + 7 Highs + 6 Mediums folded). Reference for the discipline of "code-review before paid validation" + "fix-pass before commit" rhythm.
- Real-DB validation transcript (Test A, post-reextract) — kids/wife/mother surfacing correctly on family query; NVDA/AMD/MRVL on stock query. Qualitative evidence Phase 1's tuning generalizes to clean derivative state.
