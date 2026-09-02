# Memory Citation Signal — Design & Implementation Plan

**Status:** Phase -1 / 0 / 1 / 2 **SHIPPED** (2026-08 → 2026-09). Retained design doc (graduated to atlas 2026-09-02).
**Shipped (as of 2026-08-21):**
- Phase -1: legacy `<command>` transport retired (`6e856c0`).
- Phase 0: unified `llm_response_finalize` (`08eff27`).
- Phase 1: `[M#]` injection + `<cited>` capture + `memory_citation_audit` table (`455228b`) + injection-precision analyzer (`benchmarks/citation_audit_summary.py`).
- Live-strip seam + score capture + gold-highlight frame (committed 2026-08-21): a **separate** stateful `text_filter_cited_tags` (in `src/core/text_filter.c`) wired into `webui_send_stream_delta` — **NOT** folded into the `text_filter_command_tags` seam as §2/§6 planned. Reason: with `<command>` retired, the default (native-tools) stream path runs the `cmd_tag_filter_bypass` branch, which does **no** tag filtering, so `<cited>` needed its own filter on that branch (covers live send + the `conv_stream_append` replay ring, both native and legacy). Also added: `injected_scores` on the audit table (schema v78) for a data-driven injection floor, and the `context_citations` WS frame (+ `item_id` on `context_injection`) that gold-highlights cited rows in the WebUI Context panel.
- **Phase 2** (citation-only confidence reinforcement) **BUILT & INERT 2026-09-01** (§5, §9, §9a). Reinforce hook in `memory_citation_capture` (after the audit write); `citation_reinforcement_boost` (float, default **0.0**) gated on `citation_enabled && boost > 0`; 1 h cooldown on the new **`last_cited` column (schema v83)** — NOT `last_accessed` (the recall path stamps that at injection, seconds before the cite → a shared cooldown would no-op; the original §9 instruction was a latent bug, corrected in §9a). Facts-only v1; reinforces cites from BOTH focus and tool-sourced surfaces. Saturation telemetry (pinned fraction + cited-vs-uncited confidence drift) added to the audit analyzer. **Ships inert; enable only after the legacy device-state cleanup** — a fact that survived from before the extraction guard reinforcing off a stale value would entrench it via the confidence→rank loop.
- **Extraction transient-state guard (shipped alongside Phase 2, 2026-09-01):** the extraction prompt (`memory_extraction.c`) now rejects transient device/system state (battery %, on/off/locked status, live counts, current time) as durable facts — the root cause of the stale-citation failure the 2026-09-01 luna/haiku recall probe surfaced (both models cited a stale front-door-lock fact instead of a live tool call). Legacy cleanup of pre-guard facts done via `scripts/find_transient_state_facts.py` (read-only finder) + curated removal.

**Validation (historical):** finalizer census GREEN (§13). Phase -1 full-teardown scope GREEN (§13b).
**Origin:** Mined from *"Reflective Memory Management"* (RMM, arXiv 2503.08026, ACL 2025). We take **only** its citation signal — the label-free readout of which injected memories the generator used — and discard the online-RL reranker (heavy; needs autograd/per-user training; DAWN already dead-lettered cross-encoder rerankers). Lifecycle feature, not retrieval-tuning.

---

## 1. Two things this delivers

0. **Retire the legacy `<command>` transport (Phase -1 — do this FIRST).** Investigation (2026-08-19) confirmed the `<command>`-tag path is **dormant in every config Kris runs** (all `[llm.tools].mode = "native"`), carries **zero** unique device capability (native tool-calling hits the same executor for every provider incl. local), is a **security gap** (bypasses the research-context allowlist the native path enforces), and is the finalizer's single hardest problem (the recursive reinject loop exists *only* to serve it). Removing it first shrinks Phase 0 dramatically. See §2.5.
1. **A unified response finalizer (Phase 0 — now much smaller, post-`<command>`).** With `<command>` gone, every surface's completion handling collapses to the same simple shape: strip residual tags → clean text → persist/deliver. Phase 0 centralizes that into **one** `llm_response_finalize` and, in the same seam migration, deletes the now-dead command-execute code. The recursive-loop trap is gone; the finalizer is **strip + citation + clean-text only**. Makes citation (and every future tag) a **one-place** change. Build once — implementing citation extraction twice is wasted work.
2. **The citation signal itself (Phases 1-2).** Inject memories tagged `[M# type]`, have the LLM emit `<cited>M1,M7</cited>`, the finalizer strips it from all output/persist and writes the citation data to a **dedicated audit table** (§7), then reinforces cited facts' confidence.

**Hard requirement:** voice-first parity — identical on local mic, WebUI, DAP2 Tier-1 satellites, messaging, and background jobs. The finalizer is what makes parity structural instead of copy-pasted.

**Non-goals:** no reranker, no RL, no retrieval-weight change. Only fact **confidence** moves (coupled to rank — §8, accepted).

---

## 2. Why a finalizer (the 5-sink problem, reframed)

A completed LLM response can be spoken, displayed, sent, or stored at **five** places, each with its own tag handling today:

| Sink | Location | Streams partials? |
|------|----------|-------------------|
| (a) Local voice / Piper | `dawn_tts_sentence_callback` `dawn.c:459-470` + full cleaner `:2672-2702` | **yes** (sentence-by-sentence to TTS) |
| (b) WebUI browser | `text_filter.c` state machine via `combined_chunk_callback` `session_manager_llm.c:421` | **yes** (token deltas) |
| (c) Tier-1 satellite (own Piper) | **streams** via `session_text_chunk_callback` → `webui_send_stream_delta` → the **same** `text_filter_command_tags` seam as (b) (`webui_server.c:2434`); `strip_command_tags()` `webui_satellite.c:227` covers the final text + server persist | **yes** (token deltas — CORRECTED; local_tts=true routes `sentence_cb=NULL` → chunk deltas) |
| (d) Messaging outbound | `messaging_engine_inbound.c:~648` — **no strip today** | no (final text) |
| (e) History reload (WebUI) | `transcript.js:242-282` regex cleaner | no (persisted text) |

**Key insight (CORRECTED by validation): there are TWO live-strip seams, not "(a)+(b) only".** Voice has its own sentence-level strip (`dawn.c:459-470`); WebUI browser **and** Tier-1/Tier-2 satellites all stream through the **one shared** `text_filter_command_tags` chunk seam. So live strips = **voice sentence-cb + one shared `text_filter`** (adding `<cited>` there covers browser + both satellite tiers at once). (d) messaging, (e) reload, and the background paths (jobs/reinvoke) consume **final** text only → they take the finalizer's canonical clean output and add no new strip site. Net: **2 live-strip seams + 1 shared finalizer.** (Post Phase -1, the command-strips in this table become **defensive-only** — nothing is instructed to emit `<command>` — and the strip logic folds into the finalizer.)

---

## 2.5 Phase -1 — Full teardown of the legacy `<command>` transport (do this FIRST)

**Decision (Kris, 2026-08-19): full removal, not coerce-and-keep.** Don't leave the wiring around a dead executor. Ripping the whole path out is *simpler* than the half-measure — with the `command_tags` mode value **gone**, there is nothing to coerce (a stale value becomes an unknown key → warn+ignore), and with the parse/execute callers **deleted**, the hallucinated-tag execution bug (below) deletes itself. Both the "defense-in-depth coercion" and "guard the hallucination" problems from v4 are **MOOT under full removal** — they only existed because v4 kept the machinery.

**Verdict basis (3-agent investigation):** `<command>` is cruft — all configs `native`, native reaches every provider incl. local (no fallback), same `command_execute*`/MQTT/OCP executor, and the legacy path bypasses the research allowlist (`llm_tools.c:1010-1015`). Only lost capability = device control for a local model with no tool-use chat template (a 2026 non-scenario).

### What is REMOVED (full teardown)

1. **Emit (prompt):** `LEGACY_RULES_CORE` (`llm_command_parser.c:107-126`), `generate_command_tag_instructions()` (`:300-390`) + its sole caller `build_combined_entry()` (`:413-452`), and the `command_tags` branch of `build_system_instructions_to_buffer()` (`:523-556`) + the cosmetic `command_tags`-mode `OLOG_INFO` lines (`:585-589`, `:736-742`, `:782-788`).
2. **Parse/execute (functions + all 7 seam callers):** delete `parse_llm_response_for_commands` (`llm_command_parser.c:883`) and `webui_process_commands` + its recursive loop (`webui_text_processing.c:95`), and every caller — voice `dawn.c:2667`; WebUI text `:579/:611`; satellite `webui_satellite.c:316/:333`; **WebUI audio `webui_audio.c:1275` (seam #7)**. This closes the hallucinated-`<command>`-executes-in-native bug by construction.
3. **The `[llm.tools].mode` config — collapse it (VALIDATED §13b; big touch set → its own sub-phase -1b).** `command_tags` and `disabled` both go as *user* values. **The internal per-call tool-suppression is preserved on a mechanism that already exists** — validation found `llm_tools_suppress_push()/pop()` → thread-local `tl_suppress_count`, checked FIRST in `llm_tools_enabled()` (`llm_tools.c:2258-2269`), already used by `llm_context.c`/`search_summarizer.c`. The 5 internal `tool_mode="disabled"` callers (`scheduler.c:712`, `llm_context.c:1210`, `memory_extraction.c:1665/2253`, `llm_silent_observe.c:267`) migrate onto a per-call `suppress_tools` bool (or `suppress_push/pop`) — so `disabled` need not survive as an enum value at all. (`switch_llm_tool.c:263` is a *propagation* site, not a suppressor.) **End-state (DECIDED, Kris 2026-08-19: option ii — boolean `[llm.tools] enabled`):** drops both dead values, keeps a clean single global on/off master switch, and `native_enabled` already maps bool↔mode (`config_parser.c:580-582`) so it's the natural landing. **Touch set is LARGE** (validation §5): config struct/default/parse/env/JSON+TOML serialize/validate, WebUI POST + per-session override, the prompt-builder branches, the resolution path, the per-conversation `tools_mode` DB column, and the UI (`www/index.html:1907`, `schema.js:693-703`, `llm.js` several, CSS). **DB-column handling (DECIDED): leave `tools_mode` DEAD — stop reading/writing it (drop the per-session override + the hydrate at `webui_server.c:3211`), do NOT drop the column in this commit.** The actual `ALTER`/rebuild is a trivial separate follow-up (DAWN's standard "strip legacy columns after a release cycle" pattern) — keeping the delicate schema change OUT of the bundled deletion commit is what makes bundling safe. Removing the mode *key* itself is clean (unknown-key → warn+ignore).
4. **WebUI mode selector:** remove BOTH dropdowns (`www/js/ui/settings/schema.js:698` + `www/index.html:1909`), the per-session `tool_mode` override accept (`webui_message_dispatch.c:534-549` → `build_remote_prompt_for_mode` `:806-834`), and the stored per-conversation `tools_mode` handling (DB col `auth_db.h:1065`, restore `webui_server.c:3211`) — ignore/drop the column on read.
5. **Vestigial tool-metadata / registry machinery (VALIDATED §13b, item a — small + clean).** The tool layer is **mostly shared**, so this is a short list, not a rewrite. REMOVABLE (sole consumer was the `<command>` path): the **`is_getter`** field (only reader `llm_command_parser.c:433` — delete field + all `.is_getter =` initializers) and **`tool_registry_foreach_enabled()`** (sole caller `llm_command_parser.c:542`), plus the emit funcs `generate_command_tag_instructions`/`build_combined_entry`/`combined_build_ctx_t`/`LEGACY_RULES_CORE`. FREE dead-code cleanup surfaced: **`default_local`** (zero readers tree-wide), **`tool_registry_foreach_with_capability`** (zero callers), `tool_registry_count_variations*` (test-only), and trim the now-purposeless alias-hash branch inside `tool_registry_find` (keep the function). **DO NOT REMOVE (shared, looked-legacy-but-aren't):** the whole `device_type` action-word subsystem (BOOLEAN/ANALOG/GETTER… — it's **direct-regex core**), `aliases` arrays, `device_string`, `device_map`, `mqtt_only`, `sync_wait`, `skip_followup`. `TOOL_DEVELOPMENT_GUIDE.md` edits: drop the `is_getter`/`default_local` rows, re-attribute Device Types to the direct-regex path, remove `foreach_enabled` from the API reference.
6. **Docs:** `TOOL_DEVELOPMENT_GUIDE.md`, `docs/arch/command-processing.md`, `LLM_INTEGRATION_GUIDE.md`, `WEBSOCKET_PROTOCOL.md`, `ARCHITECTURE.md:749-753`, `docs/arch/subsystems/llm.md`, `CAPABILITY_MASK_DESIGN.md`, the `memory_note_guard.c:648-651` comment.

### What is KEPT (do NOT touch)

- **`command_executor.c`** (`command_execute`/`_json`/`_mqtt_direct`/`_sync`) — shared by native + direct-regex. Provably separable (validation: legacy paths are strictly callers; no shared mutable global).
- **The native tool loop** — the survivor.
- **The internal per-call tool-suppression mechanism** that `disabled` currently provides (memory-extraction/silent-observe rely on it) — survives even if the user-facing `disabled` mode goes.
- **`text_filter.c` strip state machine** — kept as the finalizer's defensive strip; `test_text_filter.c` guards it (stays green).
- **The direct-regex path** (`try_tool_registry_match` → `command_execute_json`, `dawn.c:3803`; `processing_mode` ∈ direct_only/llm_only/direct_first): **VALIDATED NOT vestigial — recommend KEEP.** It matches ASR **input** against compile-time tool patterns *before* any LLM call (opposite direction from the retired path, which parsed LLM **output** tags — no shared code, `dawn.c:3837` confirms JSON `<command>`-matching was already removed). It delivers value native can't: `direct_only` = **offline / zero-LLM** actuation; `direct_first` = **latency** (skip the round-trip on common commands). Fully independent of the `<command>` transport; do NOT sweep it into this teardown.

### Sequencing (DECIDED: one bundled Phase -1)

Full teardown is **one reviewed commit** (Kris: bundle -1a+-1b — splitting only bought double-testing; the test overlap is just "native tools still work"). Scope of the single commit: transport removal (emit + parse + execute across 7 seams) + tool-metadata vestigial cleanup + `[llm.tools].mode`→bool collapse + internal-suppression refactor (5 callers → `suppress_tools`/`suppress_push`) + UI dropdown removal + docs. **The one thing kept OUT of the bundle is the `tools_mode` column DROP** (left dead; §2.5 #3) — that isolates the only delicate, least-reversible piece. It's a large but coherent commit ("retire the legacy command transport + its config"); the big-three review handles it. Phase -1 deletes the execute callers at the 7 seams; Phase 0 later re-touches them to add the finalizer (deliberate delete-then-add across two commits). **Gate: native device control + direct-regex both still actuate; a config with a stale `mode`/`tools_mode` boots (key ignored, warned); internal tool-suppression still works for memory extraction / silent-observe / compaction; §13b green.**

---

## 3. Phase 0 — the unified finalizer (post-`<command>`)

**New:** `llm_response_finalize(session_t *session, const char *raw_response, response_final_t *out)` (proposed module `src/core/llm_response_finalize.c`, Layer-2 core beside `text_input_dispatch.c`). Every response-completion path calls it once at its completion seam. It:

1. **strips** all non-user-facing tags (`<command>`, `<cited>`, `<end_of_turn>`) → the **canonical clean text** in `out`, which every surface persists + delivers
2. extracts `<cited>M#…</cited>` → validates against the per-turn stash (§6), writes the audit row (§7), reinforces cited facts (Phase 2)

**Post-Phase-(-1) simplification: there is no command handling left to reconcile.** In the original v3 plan the finalizer had to carefully *avoid* the divergent command executors (voice's synchronous `command_execute_json` vs the WebUI/satellite agentic reinject loop). Phase -1 **deletes** those legacy executors outright, so the entire problem evaporates: the finalizer never touches command execution because `<command>` execution no longer exists. Native tool-calling and the direct-regex path still actuate devices, but they do so on their own paths, entirely outside the finalizer. So the finalizer is unambiguously **strip (defensive residual tags) + citation + canonical-clean-text only** — no execute path, no policy flag, no recursive loop to leave upstream. Its value: collapse the (now defensive-only) strip copies into one, the single citation seam, and one canonical clean-text producer. The v3 "biggest risk" (a reviewer folding the recursive loop into the finalizer) is **gone** — there is no loop.

**Surfaces to migrate (the census — VALIDATED §13):**

| # | Path | Completion seam (verified) | Session | Threading | Notes |
|---|------|------------------------------|---------|-----------|-------|
| 1 | Local voice | ONE clean block `dawn.c:2633-2731` (`response_text` under `llm_mutex`) | `session_get_local()` LOCAL | main loop, no lock held | migration DELETES the `parse_llm_response_for_commands` call (dead post -1); folds inline strip into finalizer |
| 2 | WebUI session | **SCATTERED**: raw history-add in `llm_call_finalize` `session_manager_llm.c:302`; extract/strip/persist in worker `webui_text_processing.c:561-681` | `session` (auth'd) | session/lws, no lock held | migration DELETES the recursive `webui_process_commands` loop (dead post -1), adds finalizer |
| 3 | Tier-1 satellite | worker `webui_satellite.c:296-355` (via `session_llm_call_..._no_add`) | `conn->session` DAP2 | worker, no lock held | same loop deleted; **streams partials** via shared `text_filter` |
| 4 | Messaging | ONE clean point `messaging_engine_inbound.c:533` | `session` MESSAGING forever-conv | persistent messaging worker, no lock held | no command handling today (strip = leak fix) |
| 5 | Background jobs | ONE point `job_worker.c:222`, persist `:292` | pooled JOB session | detached worker, no lock held (pool mutex dropped pre-callout) | no command handling; focus stash DOES populate |
| 6 | Reinvoke | **TWO seams**: live `job_reinvoke.c:468`, detached `:572` | live viewer WEBUI session / detached pool session | detached, no lock held | client-saved branch (`:496-501`) → relies on the live stream strip, not just persist |
| 7 | **WebUI audio-input worker** | `webui_audio.c:1275` (execute) + `:866` (strip) — `audio_worker_thread` (starts `:1104`), DISTINCT from the #2 text worker | `session` (auth'd) | audio worker, no lock held | **MISSED by the original census (validation catch).** Live `webui_process_commands` caller; must migrate/delete or it orphans a tag executor |

**Live strips retained** (only to prevent mid-stream leakage): the voice sentence-buffer (`dawn.c:459-470`) and the **one shared** `text_filter_command_tags` seam that browser + Tier-1/Tier-2 all stream through. Both already strip `<command>`; they gain `<cited>` + the orphan-closer failsafe in Phase 1.

**Scope honesty — this is a down-payment, NOT the big refactor.** Phase 0 unifies *response finalization* (raw → clean text + citation). It does **not** touch the frame *routing / capability-matrix* addressing model, or the command executor's recursive loop — both stay put. Migrate call sites incrementally, test after each.

**System-wide behavior change to gate (validation finding):** ALL paths keep RAW `<command>` tags in the **in-memory history that feeds the next LLM turn** (voice `dawn.c:2722`; WebUI/satellite `session_manager_llm.c:302`) — only WebUI's *durable* conv_db row is stripped today. Making the finalizer's clean text the canonical persisted form therefore changes what the model sees on subsequent turns on **every** surface. This is the "voice-persists-raw bug" — but it's system-wide, and it's a deliberate, gated change: Phase-0 before/after parity tests must assert on **stored** text, not just spoken text.

---

## 4. Research findings (verified) that ground the citation half

1. **Focus injection reinforces NOTHING today** — the only `+0.05` is the explicit `recall` tool (`memory_callback.c:707`). → Decision 1 = citation-only (§5).
2. **Candidates freed before the LLM runs** (`focus_result_free`, `build_focus_block.c:481`) → the ordinal→item_id map must be stashed in session state during composition.
3. **Voice shares the WebUI prompt path** (`session_dispatch_user_turn` → `dawn_build_prompt` → `build_focus_block`), so `[M#]` injection lands on voice free — `#ifdef ENABLE_WEBUI` only (§10).
4. Each `focus_candidate_t` already carries `item_id` = `"fact:123"` (`focus_candidate_helpers.c:124`) → `[M1] → fact_id` is a prefix-strip.

---

## 5. Decision 1 — RESOLVED: citation-only reinforcement

Finding #4.1 inverts the surface+cite split: with no existing every-turn reinforcement to redistribute, a surface award becomes net-new reinforcement **for proximity not use** — a positive feedback loop on `confidence` (= the fact's `importance` term, `focus_source.c:185`) that saturates/entrenches the head of the distribution. **Option A (citation-only):** reinforce only cited facts; uncited injected facts unchanged from today. Award: single `citation_reinforcement_boost` = **0.05** (a time-to-saturation knob, not a discrimination knob — §8/G9).

---

## 6. Citation plumbing (Phases 1-2)

| # | Change | Location | Notes |
|---|--------|----------|-------|
| 1 | Render `[M# type]` | `build_focus_block.c:411-415` | `[M%d %s]` — keep the type label (don't delete context the model conditions on). |
| 2 | Stash ordinal→item_id | new `struct session` field; populate in render loop under `history_mutex` | survives the free (#4.2). Runs on shared dispatch path → LOCAL+WebUI free. |
| 2b | **Clear stash at dispatch entry** | `session_dispatch_user_turn` `session_manager.c:2320`, unconditional | `build_focus_block` short-circuits before render on empty/zero-candidate/dedup-emptied turns (`:330-349`); without a dispatch-entry clear a short-circuited turn inherits turn N−1's map and a stale `<cited>` false-validates. Empty map ⇒ all citations drop. |
| 3 | Static cite instruction | new footer in `build_stable_segment` `webui_auth_helpers.c:444-479` | cached prefix → no cache bust; no few-shot echo needed. |
| 4 | Parse + validate + audit + reinforce | **inside the finalizer (§3)** | one place, all 6 surfaces. Mirror `parse_llm_response_for_commands` minus the executor; validate ordinals against the stash, facts-only reinforce. Copy the ≤12-entry map out under `history_mutex`, release, then write auth_db (lock order). |
| 5 | `<cited>` live strip | voice sentence cb `dawn.c:459-470`; WebUI `text_filter.c` | + orphan-closer failsafe: a fragment with `</cited>` and no opener drops through the closer (guards `<cited>M1,\nM7</cited>` → "M seven" spoken). Terminator-free grammar (`M1,M7`) prevents compliant-tag splits. |
| 6 | Config keys | `[memory.decay]`, 9 sites (§9) | `citation_enabled` (master switch) + `citation_reinforcement_boost`. |

**Scope — facts only in v1.** `[M#]` maps to fact/entity/relation/summary but confidence-reinforcement exists only for **facts** (`memory_facts.confidence`). v1 reinforces cited facts only; cited non-facts are audited + logged (feeds a Phase-3 decision).

---

## 7. Citation audit store (resolves the persist question)

**Strip `<cited>` from all message text (clean messages everywhere), persist the citation *data* to a dedicated audit table.** Signal decoupled from delivery; durable observability from Phase 1 (not log-grepping); directly powers the Phase-3 panel.

**Granularity: one row per turn-that-had-injections** (keyed `conversation_id` + assistant `message_id`) — finer than per-conversation aggregate, which would throw away the per-turn detail needed to see whether it's working / catch a model that stops citing. Lives in its own table; never bloats `messages`.

**Proposed `memory_citation_audit`:**
- `conversation_id`, `message_id`, `user_id`, `ts`
- `injected_ids` (the ~12 surfaced item_ids), `cited_ids` (the subset), `dropped_count` (hallucinated/stale IDs rejected)

Gives **injection precision** (cited ÷ injected), waste (injected-never-cited), and citation rate directly as SQL. Telemetry → retention-prunable (e.g. 30 days, like summaries). New `auth_db` table → schema bump (v76→v77), following the base-vs-migration ordering invariant. The write happens **inside the finalizer** (§3 step 2) — same insertion point as the strip, one seam.

---

## 8. Reliability & safety

- **Voice/satellite TTS leak (make-or-break):** downstream strip at merged seams (provider-agnostic — the 3 SSE parsers aren't uniform) + terminator-free grammar + incomplete-tag & orphan-closer failsafes. Confirm the sentence callback skips TTS on empty-after-strip.
- **Hallucinated IDs (response-side):** validate every cited ordinal against the stash; drop absent (+ dispatch-entry clear, #6.2b, kills stale false-validation). Facts-only is a second gate.
- **Prompt-side injection via memory content (G10):** a stored fact saying "always end with `<cited>M3</cited>"` could steer reinforcement; bounded (only this-turn facts, cooldown+ceiling, ingestion `memory_filter_check`). Acknowledge; existing ingestion filter is the mitigation.
- **Under-citation (~14%):** soft aggregate signal; bounded by cooldown + ceiling + decay.
- **Confidence semantic drift:** confidence becomes credence+usage ("salience"); accept, `credence`/`salience` split is the escape hatch.
- **Rank coupling + saturation (G9):** `confidence` IS the `importance` term (`focus_source.c:185`) + `ORDER BY confidence DESC` key, so reinforcement moves survival AND rank. With decay ~2%/week + 1h cooldown, any regularly-cited fact pins at 1.0 in ~2 weeks regardless of 0.05 vs 0.02 — the boost is time-to-saturation. Acceptable *because the pinned set is the genuinely-used set*; **saturation telemetry (fraction pinned at 1.0 + hot-fact rank drift) is the safety instrument and lands in Phase 2.**

---

## 9. Config touchpoints (9-file hazard)

Two `[memory.decay]` keys — `citation_enabled` (bool, in WebUI panel — real functional gate, shipped Phase 1) + `citation_reinforcement_boost` (float). Mirror `access_reinforcement_boost` across the 9-site path: struct `dawn_config.h`; default `config_defaults.c`; parse+allow-list+clamp `config_parser.c`; JSON `config_env.c` (flat under `memory`); TOML `config_env.c`; POST+clamp `webui_config.c`; `dawn.toml.example`; roundtrip test (per-key assertion).

**Shipped-in-Phase-2 deviations from the draft above:**
- **`citation_reinforcement_boost` defaults to `0.0` (inert)**, not a live value — reinforcement stays off until the legacy device-state cleanup runs (enabling it while stale device-state facts are live would entrench them via the confidence→rank loop; see §16 and the 2026-09 luna/haiku recall probe).
- **Deliberately NOT surfaced in `schema.js`** yet — parsed + round-tripped so a hand-edit survives, but kept out of the WebUI panel until the cleanup lands, so it cannot be accidentally enabled early. Add the `advanced` control when reinforcement is turned on.

### 9a. `last_cited` — the cooldown column (correction to the "mirror `access_reinforcement_boost`" instruction)

`access_reinforcement_boost`'s 1 h cooldown keys on `last_accessed`, but the recall render path (`memory_callback.c`) stamps `last_accessed = now` on every fact it surfaces **seconds before** the model cites it — so a citation-reinforcement primitive sharing that column would evaluate its cooldown to "just accessed" and **never bump** the tool-cited facts this feature exists to reinforce. A dedicated **`last_cited INTEGER` column on `memory_facts` (schema v83)** is therefore required as the citation cooldown key. The reinforce statement gates in its `WHERE` (`last_cited IS NULL OR now - last_cited > 3600`) so the row is touched only when eligible — the hour is measured from the last *bump*, not the last attempt, and a never-cited fact bumps on its first citation. Confidence is ceilinged at `MIN(1.0, confidence + boost)`; `(id, user_id)` filter for CWE-639.

---

## 10. WEBUI-gating caveat

Focus-block + builder path is `#ifdef ENABLE_WEBUI`; non-WebUI builds use the static prompt (no focus block) → feature inert there, same gating as jobs/research. Parity holds within WebUI-compiled (shipped) builds. Note: the **finalizer** (Phase 0) is *not* memory-specific and could live outside that gate — validation should confirm whether Phase 0 can be WEBUI-agnostic even if the citation half isn't.

**Phase 2 placement (shipped):** reinforcement is NOT `#ifdef ENABLE_WEBUI`-gated. It lives in `memory_citation_capture` (`src/memory/memory_citation.c`, compiled unconditionally), right after the audit write — same Layer-2 module, so no core→memory upward call and the copy-under-`history_mutex`-then-write-auth_db-leaf discipline is inherited from the existing capture path. It self-gates on `citation_enabled && citation_reinforcement_boost > 0.0`. This matters because the **tool-sourced** citation path (`memory_callback.c`) fires on voice/local (`conv=0`) turns too — a WEBUI ifdef would have wrongly disabled reinforcement there. Only the *focus* `[M#]` half is WebUI-origin.

---

## 11. Phasing

- **Phase -1 — full teardown of `<command>` + mode-config collapse (one bundled commit; §2.5).** Delete emit (`LEGACY_RULES_CORE`/generator/`build_combined_entry`) + parse/execute (`parse_llm_response_for_commands`/`webui_process_commands`) across the 7 seams; tool-metadata vestigial cleanup (`is_getter`, `foreach_enabled`, dead-code); collapse `[llm.tools].mode`→boolean `enabled`; migrate the 5 internal `disabled` callers to `suppress_tools`/`suppress_push`; remove both UI dropdowns + the per-session override; leave the `tools_mode` DB column dead (no drop). Keep `command_executor.c`, native, and the direct-regex path. Own reviewed commit (big-three). **Gate: native device control + direct-regex "set volume to 5" both still actuate; internal tool-suppression still works (memory extraction / silent-observe / compaction get no tools); a stale `mode`/`tools_mode` boots clean (ignored + warned); §13b green.**
- **Phase 0 — unified finalizer (now much smaller).** Build `llm_response_finalize` (strip + clean-text; no command handling — it's gone). Migrate all **7** completion seams incrementally (test after each) — the original 6 **plus the WebUI audio-input worker (§3 #7)** — **deleting the now-dead legacy execute callers as you go** (one touch per seam). Persist canonical clean text uniformly. No citation yet. Own reviewed commit. **Gate: all 7 surfaces behave identically to today for plain responses; stored-text tests pass; §13 green.**
- **Phase 1 — citation plumbing + audit, LOG/AUDIT-only, master-switch-gated.** `[M# type]` render, dispatch-entry stash clear + populate, static instruction, `<cited>` live strips (both) + failsafes, parse+validate **in the finalizer**, write the audit table. No reinforcement. All gated by `citation_enabled`. **Land + unit-test the strips before enabling the instruction.** Gate (≥1 cycle): zero leaks any surface; per-provider citation rate from the audit table above a chosen bar; low/stable dropped-ID rate.
- **Phase 2 — reinforcement.** Option A, 0.05, facts only, cooldown untouched. Add saturation telemetry (the safety instrument).
- **Phase 3 (optional).** `item_id` broadcast + WebUI panel highlight (reads the audit table); award tuning; entities/relations decision from Phase-1 audit; bench overlap-oracle.

---

## 12. Testing

Unit: `<cited>` parse (well-formed/malformed/stale/empty-stash/count-cap/terminator-free), orphan-closer fragment never reaches TTS, each strip site vs synthetic tagged responses, empty-after-strip skips TTS, config roundtrip per-key. Integration/live: a voice + Tier-1 + messaging + job turn each with a citation — nothing spoken/sent/persisted contains the tag; audit rows written correctly. Bench: citation rate + injection precision from the audit table. Format + `test_config_roundtrip` + WS-send-funnel guards.

---

## 13. Validation status — GREEN (3-agent census, 2026-08-19)

**Verdict: FEASIBLE. Build-ready pending the design corrections above (already folded in) + Kris's sign-off.**

- [x] **Census (interactive) — voice/WebUI/satellite:** all have a usable `session_t` + user_id + reachable completion seam. Corrections applied: Tier-1 **streams partials** (was mis-modeled as final-only) via the shared `text_filter` seam; WebUI seam is **scattered** across `llm_call_finalize` + the text worker; voice is bespoke (not through `session_llm_call`).
- [x] **Census (background/channel) — messaging/jobs/reinvoke:** all have a `session_t` (MESSAGING forever-conv / pooled JOB / viewer-or-pool). **Threading CLEAN** — every seam is lock-free at completion; copy-stash-out-under-`history_mutex`-then-`auth_db`-leaf satisfies ordering from the persistent messaging worker and both detached workers. Reinvoke has **two** seams + a client-saved branch. None execute `<command>` today.
- [x] **Feasibility — cross-cutting:** one signature `(session_t*, raw, out*)` serves all 6 (no path lacks a session). `<command>` *execution* is unified (`command_execute_json`) but *extract/dispatch* is duplicated with divergent semantics (voice synchronous vs WebUI/satellite agentic loop) → **finalizer excludes command execution entirely** (§3 correction). Lock ordering clean from all threads. **Phase 0 CAN be WEBUI-agnostic** (Layer-2 core beside `text_input_dispatch.c`, the precedent; only citation *population* stays `#ifdef ENABLE_WEBUI`, stash simply empty otherwise).
- [x] **Biggest risk (named, mitigated):** a reviewer/implementer trying to fold the WebUI recursive command loop *into* the finalizer. Mitigation: the contract explicitly leaves the loop upstream; finalizer = strip + citation + clean-text only.
- [x] **No blockers.** Two wrong premises corrected (Tier-1 streaming; the raw-in-memory-history change is system-wide, not voice-only) — both affect scoping/tests, not feasibility.

## 13b. Phase -1 teardown validation — GREEN (removal inventory + teardown scope, 2026-08-19)

**cbm code-graph cross-check (reindexed, 2026-08-19):** CALLS-graph confirms the caller sets are closed — `webui_process_commands` = 3 callers (graph **independently found** the audio seam `webui_audio.c:1275`), `parse_llm_response_for_commands` = 1 (dawn.c:2667), `tool_registry_foreach_enabled` = 1 (emit branch), `tool_registry_foreach_with_capability` = 0 (dead). No native fn calls a legacy parse fn (keep-set separable). `search_code` confirms `is_getter` has 1 read (`build_combined_entry:433`, deleted anyway) and `default_local` 0 reads. All 5 internal `disabled` setters + a 6th (`handle_json_message:542`, part of the per-session override already being removed) confirmed; `suppress_push/pop` home is live. Graph `USAGE` doesn't model C field access → field claims rest on `search_code`, which is correct for that.


**Verdict so far: FEASIBLE, external contracts SAFE, keep-set provably separable. Full-teardown decision (Kris) makes the v4 coercion + hallucination-guard items MOOT (removed, not kept). Two new scoping targets opened by the teardown — validating before ready.**

- [x] **Emit inventory COMPLETE.** The 3 spec sites are the only syntax-teachers (`build_combined_entry`/`generate_command_tag_instructions` orphan cleanly; `tool_instructions/`, persona, localization all clear). **Gap fixed in §2.5:** the builder has no fallback else → add unknown→native.
- [x] **Config surface — INCOMPLETE as v3-written; corrected.** Three uncoerced ingest points the v3 list missed: env `DAWN_LLM_TOOLS_MODE` (`config_env.c:211`, applied after parse), stored per-conversation `tools_mode` (`webui_server.c:3211`, DB `auth_db.h:1065`), second dropdown (`www/index.html:1909`). Coercion must be **defense-in-depth** because `llm_tools.c:2384` + the builder gap turn a stale value into **silent total tool-capability loss**. Round-trip field-drop hazard CLEAR (serializers unconditional).
- [x] **Execute-caller inventory — one ORPHAN found: seam #7** (`webui_audio.c:1275`), a live `webui_process_commands` caller the 6-seam census missed. Now census row #7. Otherwise closed (voice `dawn.c:2667`; text `:579/:611`; satellite `:316/:333`). No callers in messaging/jobs/reinvoke.
- [x] **KEEP-set provably separable — CONFIRMED.** Legacy = strictly callers of shared services (`command_executor.c`, `command_router`), one-way edges, no shared mutable global. Native loop has zero legacy refs. Direct-regex independent (gated on `command_processing_mode`, a different axis). Defensive strip (`text_filter.c`, `strip_command_tags`) pure + lexically separable from every execute call.
- [x] **Blast-radius:** external contracts **NONE broken** (OCP/MQTT device traffic reproduced by the native path; satellites never saw the tag). Tests: `test_text_filter.c` guards the KEPT strip (stays green — don't delete); no test asserts emit/execute. Docs stale-only (list in §2.5 refs). www/js strips go dark harmlessly; both mode dropdowns must lose the option.
- [x] **Hallucination-window → it's a PRE-EXISTING live bug, not a transient gap.** Callers fire on `strstr` with no tool-mode gate → hallucinated `<command>` executes in today's native config. Guard is effectively **required** (4 sites incl. #7). Recommendation in §2.5: small standalone neuter now, or Phase 0 caller-deletion closes it. **Caveat:** the *live* `text_filter` strip is bypassed under native (`cmd_tag_filter_bypass = llm_tools_enabled`), so the finalizer's defensive strip is **final-text-only** under native — a stray tag still streams raw mid-stream (pre-existing). **Under full removal this whole item is moot — the callers are deleted, not guarded.**

### New teardown-scope targets — GREEN (2-agent, 2026-08-19)

- [x] **(a) Tool-metadata / registry vestigial scoping — DONE.** Tool layer is mostly shared; removable list is small: `is_getter` field, `tool_registry_foreach_enabled()`, the emit funcs, + free dead-code (`default_local`, `tool_registry_foreach_with_capability`, test-only variation-counters, the `tool_registry_find` alias branch). KEEP the `device_type`/action-word subsystem (direct-regex core), `aliases`, `device_string`, `device_map`, `mqtt_only`, `sync_wait`, `skip_followup`. Details in §2.5 #5.
- [x] **(b) `[llm.tools].mode` / `disabled` teardown — DONE.** Collapsible. Internal suppression has a ready home (`suppress_push/pop` / a per-call `suppress_tools` bool) — `disabled` need not survive as an enum. **DECIDED: option ii (boolean `[llm.tools] enabled`)**. Touch set is LARGE (config + DB `tools_mode` column + UI) but **bundled into the single Phase -1 commit** (Kris); the `tools_mode` column is left dead (not dropped) to keep the delicate schema change out of the bundle. Full touch set in §2.5 #3.
- [x] **Adjacent — direct-regex/`processing_mode` is NOT vestigial.** Different direction (ASR input, pre-LLM) from the retired path, no shared code; delivers offline (`direct_only`) + latency (`direct_first`) value native can't. **Recommend KEEP** — not part of this teardown. (Any stale `direct_only` ignore-word branch is a separate future look.)

---

## 14. Open decisions

1. **RESOLVED:** finalizer-first (Phase 0). Decision 1 = Option A. Award 0.05. Facts-only v1. Persist = strip-from-messages + audit table (§7). Log/audit-only gate = one cycle with kill-switch.
2. **RESOLVED by §13:** feasibility confirmed, no blockers. Finalizer excludes command execution (recursive loop stays upstream); threading clean; Phase 0 WEBUI-agnostic. Two premises corrected (Tier-1 streaming, system-wide raw-history change).
3. **RESOLVED (full teardown, not coerce):** retire `<command>` (Phase -1) before the finalizer. `[llm.tools].mode` → **boolean `enabled`** (`disabled`+`command_tags` both dropped as user values; internal suppression preserved on the existing `suppress_push/pop` path). **One bundled Phase -1 commit** (transport + mode-collapse + metadata); `tools_mode` DB column left **dead** (drop in a later trivial migration) — that isolation is what makes bundling safe. **Keep** direct-regex/`processing_mode` (offline + latency; not vestigial).
4. **RESOLVED:** dropping the tool-less-local escape hatch accepted (2026 non-scenario). System-wide clean-text-in-history change intended (mostly evaporates with `<command>` gone; residual `<end_of_turn>`/`<cited>`).

**Nothing open. Build-ready — awaiting only "go".** First unit of work: the bundled **Phase -1**.

---

## 15. Phase -1 build order (implementation kickoff)

One bundled commit. Build + `./format_code.sh --changed` + relevant unit tests after **each** step so the tree always compiles; re-query cbm (`project: var-lib-dawn-source-dawn`) after each deletion to prove no caller dangles. **Never `git add/commit/push`** — suggest the command, Kris runs it.

1. **Emit removal** — delete `LEGACY_RULES_CORE` (`llm_command_parser.c:107-126`), `generate_command_tag_instructions` (`:300-390`), `build_combined_entry`/`combined_build_ctx_t` (`:397-452`), and the `command_tags` branch of `build_system_instructions_to_buffer` (`:523-556`) + the cosmetic mode `OLOG_INFO` lines. After: nothing teaches `<command>`. Build.
2. **Execute-caller removal (the 7 seams + functions)** — delete the `parse_llm_response_for_commands` call (`dawn.c:2667`) + the function (`:883`); delete `webui_process_commands` (`webui_text_processing.c:95`) + its 3 callers (`text_worker_thread` `:611`, `satellite_worker_thread` `webui_satellite.c:333`, `audio_worker_thread` `webui_audio.c:1275`) and the recursive re-call sites (`:579`/`:316`). **This closes the hallucinated-`<command>`-executes bug.** KEEP `strip_command_tags`/`text_filter_command_tags` (defensive). **Verify: native device control + direct-regex "set volume to 5" still actuate.**
3. **Tool-metadata cleanup** — remove `is_getter` (field + every `.is_getter =` initializer; sole reader was `build_combined_entry:433`, now gone), `tool_registry_foreach_enabled` (sole caller was the emit branch), and dead code: `default_local` (0 readers), `tool_registry_foreach_with_capability` (0 callers), `tool_registry_count_variations*` (test-only), trim the alias-hash branch inside `tool_registry_find` (keep the fn). Build + `test_tool_registry`.
4. **Mode → boolean `enabled`** — (a) add a per-call `suppress_tools` bool (or reuse `suppress_push/pop`) and migrate the 5 internal `disabled` setters (`scheduler.c:712`, `llm_context.c:1210`, `memory_extraction.c:1665/2253`, `llm_silent_observe.c:267`); **verify memory-extraction / silent-observe / compaction still get NO tools.** (b) replace `[llm.tools].mode` string with bool `enabled` across the config touch set (struct/default/parse/env/JSON+TOML serialize/validate/webui POST — `native_enabled` maps naturally). (c) remove the per-session `tool_mode` override (`webui_message_dispatch.c:534-549`, `build_remote_prompt_for_mode`). (d) leave the `tools_mode` DB column **dead** — stop reading/writing (drop the hydrate `webui_server.c:3211` + override persist); **no column drop.** (e) remove both UI dropdowns (`schema.js:693-703`, `index.html:1907`) + `llm.js` refs + CSS.
5. **Docs + test** — update `TOOL_DEVELOPMENT_GUIDE.md` (drop `is_getter`/`default_local` rows, re-attribute Device Types to direct-regex, drop `foreach_enabled` from API ref), `docs/arch/command-processing.md`, `LLM_INTEGRATION_GUIDE.md`, `WEBSOCKET_PROTOCOL.md`, `ARCHITECTURE.md:749-753`, `llm.md`; update `test_config_roundtrip.c` for the bool.
6. **Gate before commit** — native + direct-regex actuate; internal suppression works; a stale `mode`/`tools_mode` boots clean (ignored+warned); format `--check`; big-three review (arch/efficiency/security) on the diff; then Phase 0.

---

## 16. Post-ship decisions — audit read (2026-08-26)

First real read of `memory_citation_audit` on the live DB (`benchmarks/citation_audit_summary.py`), after Phases -1/0/1 shipped and the OpenAI Responses prompt-cache reorder landed. Decisions below are Kris's; revisit **tomorrow evening (2026-08-27)**.

**Data snapshot (live auth.db):**

| Window | Turns | Cite rate | Recall-turn precision | Dropped ordinals |
|---|---|---|---|---|
| All time | 171 | 29.2% | 34.7% | 20 |
| Last 3d | 86 | 46.5% | 36.9% | 11 |
| Last 1d | 44 | **47.7%** | 22.8% | **0** |

- **Tool-sourced (Option B):** tool-cite precision 36.4% (67/184), dropped-tool **0** — clean and comparable to focus cites.
- **Dropped ordinals are episodic, not systemic:** all 20 come from **3 turns** (Aug 20 ×4, Aug 21 ×5, Aug 24 ×11 — that last turn emitted 19 ordinals, 8 valid + 11 bogus). Last full day (Aug 25, 44 turns) spotless. Keep `dropped=0` as the Phase-2 gate; it isn't drifting.
- **`injected_scores` (v78) does NOT separate cited from uncited.** cited final_score median **2.531** vs uncited **2.513** (0.018 gap on a 2.3–3.0 range); per-kind it's worse — for **facts**, uncited median (2.595) is *higher* than cited (2.579). Floor sweep cuts uncited only at ~1:1 with cited (floor 2.533 → 49% cited kept / 47% uncited cut).

**Decisions:**

1. **Item 1 (compliance salience fix) — NOT building. Monitor only.** The prompt-cache reorder moved the volatile focus block (the `[M#]` items) out of `instructions` to sit immediately before the question — structurally the same "put the cite reminder next to the `[M#]` items" fix this item proposed. Compliance rose ~18% → ~47% in the recent window as a side effect. No dedicated salience work; keep watching the rate.
2. **Item 2 (injection-floor from `injected_scores`) — ABANDONED (off the table).** Preliminary data kills the premise: `final_score` doesn't discriminate cited vs uncited (facts anti-correlate slightly), so a score floor can't shave the ~90% injection waste without dropping real cites at the same rate. Keep the `injected_scores` column + the analyzer for monitoring, but no floor-fitting work. Reopen only if a *different* discriminator surfaces.
3. **Phase 2 (reinforcement) — GREEN-LIT, hold ~a day.** Proceed with the +0.05 citation-only award, **reinforcing cites from BOTH focus AND tool-sourced (Option B) surfaces** (not focus-facts-only as the original §5/§6 draft assumed — tool-cite precision is clean and comparable). Facts-only in v1 stands. **Hold ~a day before building** — luna is newly in rotation and its wordiness is still being balanced, which confounds cite behavior; let the recent-window baseline settle. Revisit **2026-08-27 evening.**
