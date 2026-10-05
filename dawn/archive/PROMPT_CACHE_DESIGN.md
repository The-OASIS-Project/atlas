# Prompt Caching in DAWN: One Append-Only Request per Conversation

**Status:** Shipped (2026-10-05) on DAWN branch `adaptive_dwell`; P0–P4 complete.

DAWN is a voice-first assistant daemon that talks to Anthropic, OpenAI (Responses and Chat
Completions), OpenRouter, Gemini (through its OpenAI-compatible endpoint) and local llama.cpp/Ollama.
Until September 2026 it cached only the static front of each request: the tool schemas and the
system prompt. Everything after that, including the conversation history and each tool-loop
iteration, was re-sent at full price on every call, and the logs reported "cache hit" on every turn
while most of the prompt was billed uncached. The same request layout also caused a hard correctness
failure on Anthropic accounts created after 2026-08-31. The fix was one design: every conversation
becomes a single append-only request whose earlier bytes never change. The system prompt and the tool
set are frozen when a conversation starts. Each turn's retrieved context, operator instructions and
tool changes are appended where they happen. Signed reasoning is stored and replayed verbatim.
Compaction runs only between turns. A per-call cache monitor reports what the provider read against
what it should have read. This document records the evidence, the measured vendor behavior, the final
architecture, and the decisions behind it.

---

## 1. The problem and the evidence

### 1.1 What was being re-billed

A study of 34 daemon logs (2026-09-21 to 09-25) paired 135 Anthropic calls cleanly:

| Metric | Value |
|---|---|
| Calls with any cache read | 80% |
| Full misses | 3% |
| Prompt tokens read from cache | 32% |
| Prompt tokens written to cache | 7% |
| Prompt tokens **uncached** (full price) | **62%** |
| Input-cost saving vs. no caching at all | ~27% |

The cache read stayed pinned at the size of the static prefix (≈40K tokens) while the prompt grew.
Coverage fell with session length:

| Prompt size | Calls | Read / prompt | Uncached / prompt | Median uncached tokens |
|---|---|---|---|---|
| 20–50K | 34 | 50% | 25% | 7.3K |
| 50–100K | 54 | 46% | 46% | 26K |
| 100–160K | 31 | 30% | 67% | 80K |
| >160K | 15 | 13% | 85% | 221K |

Short voice exchanges looked healthy because the prefix was most of the prompt (the production voice
median is ~2.4K tokens). Long tool-heavy and research sessions were where the money went. Modeling
each warm call as "read the previous prompt, write the growth, leave a 3–10K tail uncached" predicted
a 42–55% cut in Anthropic input cost.

### 1.2 Why: the per-turn context sat in front of the history

Each turn DAWN rendered a volatile block (`[system_time]`, memory/document/calendar retrieval,
device notices, satellite and channel hints) and put it in a second system message, ahead of the
history. On Anthropic it became `system[1]`. On the Chat Completions family it was merged into the
front system message. Prompt caches are prefix matches, so a change at the front invalidates
everything after it:

- **Anthropic** cached up to its breakpoints, and both breakpoints sat on static content (last tool,
  `system[0]`). The history was never inside a cached span.
- **Chat Completions providers and local llama.cpp** saw a different front every turn, so automatic
  prefix caching and KV reuse stopped at the system prompt. Locally this showed up as latency rather
  than cost: the history was re-processed every turn.
- **Mid-turn compaction** rewrote history in the middle of a tool round, and its fallback emptied old
  tool results in place. Each of these is a history edit.
- **Event-driven prompt rewrites** edited `messages[0]` in place: tool toggles, a component going
  online or offline (which rewrote every session's prompt at once), persona and settings changes, and
  satellite context.

It was never caught because a documented premise was wrong. An earlier OpenAI fix stated that
Anthropic `cache_control` made placement "position-independent". Anthropic's documentation says the
opposite: caching is a byte-prefix match in `tools → system → messages` order, and *"any byte change
anywhere in the prefix invalidates everything after it."* The existing instrumentation checked whether
the static prefix drifted, not what fraction of each prompt was uncached. Nothing failed and no answer
was wrong; only cost accounting showed it.

### 1.3 The correctness bug: binding mismatches on new Anthropic accounts

On Claude Opus 5.5 and Fable 5.1 (and Mythos 5.1), a replayed `thinking` block's `signature` binds
the exact prefix it was produced under: top-level `system`, the `tools` set and every earlier message.
For accounts created on or after 2026-08-31 00:00 UTC, replaying a block whose prefix changed is a
**400 `invalid_request_error`**, decided before any output. Older accounts only record the mismatch,
which is why the problem was invisible on the maintainer's own account.

DAWN replayed signed thinking from tool-use turns and also made every edit class the vendor lists as
invalidating: the re-rendered `system[1]` (`system_rerendered`), stripped broadcasts, keep-tail
compaction, an asynchronous compaction merge, and emptying old tool results. On 2026-09-25 the
maintainer's account was opted into enforcement (the documented way an older organization sees what a
new one sees) and two captures of DAWN's exact request shape were replayed with
`prefix_mismatch_behavior: "error"`:

- **DAWN's layout:** the tool-loop iterations of turn 1 returned 200. The user's second turn returned
  **400**: *"Invalid `signature` in `thinking` block … The `system` prompt differs from the one this
  block was created with"* (diagnosed as `system_changed`, pattern `system_rerendered`).
- **Append-only layout** (system frozen, per-turn context appended and left in place): all requests
  200, nothing dropped.

So every new-account user on Opus 5.5 or Fable 5.1 would hit a hard 400 on the first turn after any
tool use. Two related defects made it worse:

- **"Thinking disabled" did not disable thinking.** DAWN's shipped default omitted the `thinking`
  field. On Sonnet 5, Opus 5, Opus 5.5 and Fable 5.x, an omitted field runs adaptive thinking at the
  model's default effort, with the display defaulting to `"omitted"`. Blocks arrive as zero bytes of
  text plus a signature; 48 of 63 thinking blocks in the logs were like that. DAWN dropped such blocks
  on replay (a `predecessor_missing` edit), and default-config users paid for medium-effort thinking
  they believed was off. On Opus 5.5 and Fable 5.1, `thinking.type: "disabled"` is itself a 400.
- **Block fidelity.** The streaming parser kept one thinking buffer and one signature buffer per
  response. A response with two thinking blocks would be replayed as one block with concatenated
  signatures, which Anthropic treats as tampering (a 400 on every account, which no binding setting
  can rescue). `redacted_thinking` was not handled, block order was not preserved, and the final text
  turn was stored as a plain string without its reasoning.

The vendor's own statement settled the design: *"the same patterns keep the prompt cache warm."* The
cache fix and the correctness fix are one design.

---

## 2. The principle: one append-only request per conversation

**Invariant:** the serialized bytes of call *k* are an exact byte-prefix of call *k+1*, across turns
and across tool-loop iterations, except at a small set of declared boundaries.

| Mechanism | What it does | Why |
|---|---|---|
| **Frozen prefix** | The system prompt is rendered once per conversation and stored in a content-addressed `prompt_prefixes` table; `conversations.prefix_hash` points at it. Reloads replay the stored bytes. | A re-render after a code or settings change would otherwise move the front of the prompt. Storing bytes (not just a hash) lets a later persona change reach old conversations as an appended instruction instead of a boundary. |
| **Surface-neutral stable prompt** | Anything that varies by surface or time (tool availability, room/area, channel, voice/ASR hints, the headless-job directive, notices, memory with relative dates) moved out of the system prompt. | A conversation can continue on another surface (WebUI, voice, messaging, a background job) without changing its prefix. |
| **Appended turn context** | Each turn's `[system_time]`, retrieved items and notices become a kind-marked `turn_context` part in the question's user message, persisted as a hidden row and never cleared or rebuilt. Canonical order: `[memory][turn_context][question]`, then any operator notes. | Anthropic's documented append-only form; nothing earlier ever changes. |
| **Appended instructions and directives** | A changed stable section is appended as an `instruction` message; the surface's directive set is appended whenever it differs from the newest one. On models that accept mid-conversation system messages these render as `role: system`; elsewhere as an in-band `[Operator note]`. | Replaces every in-place prompt writer. Changes are detected at dispatch by comparing section hashes, so unknown writers are caught too. |
| **Tool set frozen by value** | Tool definitions (`{name, description, parameters}`, sanitized, canonical-hashed) are frozen with the prefix, covering every tool registered for local or remote surfaces. A tool not allowed in the current context stays advertised and is refused at execution. A real change becomes a `tool_change` row. | Toggling a tool or a device going offline no longer touches the prefix. The security gate stays at execution. |
| **Compaction at the turn seam** | Summaries are made asynchronously after a turn ends and applied at the start of the next one, as a folded `summary` part. Never mid tool round. | Mid-round compaction edits a prefix that signed blocks are bound to. A compaction is one declared boundary. |
| **Turn blocks replayed verbatim** | Each assistant turn is stored as an ordered list of canonical blocks (`text`, `tool_call`, `reasoning{carrier, format, model, native}`) in `messages.llm_blocks`, and replayed exactly. | Signatures and encrypted reasoning are bound to bytes. A reload must reproduce the prefix they were minted under. |
| **Images from history only** | An image is rendered from its persisted `[IMAGE:id]` form with its real MIME type, on live turns and reloads alike; tool-captured images are stored on their tool row. | Images were serialized three different ways (one hard-coded JPEG, one duplicate), so any non-JPEG turn poisoned later blocks. |
| **Incremental focus** | A turn sends only retrieved items that are new or changed. Items the retained context already shows get a `[still relevant: M3, M7]` reference line. Item handles (`[M#]`) are stable for the conversation's lifetime. | Retained context would otherwise grow by one full retrieval block per turn, and per-turn `[M1]` numbering would collide across retained turns. |

**Declared boundaries** are the only places the prefix may change: adopting a pre-existing
conversation (`adopted`), a tool-set change that cannot be appended (`tool_set_changed`), and a
compaction. A boundary drops earlier reasoning in memory and raises a persisted
`conversations.reasoning_floor_msg_id`; the replay read never returns reasoning at or below the floor,
so a reload cannot re-insert a block whose prefix is gone.

**Invariant R3:** everything appended to a persisted conversation's live history is persisted. Tool-loop
hints, reinvoke envelopes, captured images and kind rows all became rows, and CI checks live-history /
DB-replay parity after every turn. A message that existed only in memory would make every reload a
cache miss and, on enforced accounts, a dropped block.

Kind rows (`turn_context`, `memory`, `directive`, `instruction`, `envelope`, `tool_change`, loop notes)
are filtered in SQL from every display, search, export and admin read; only the replay read returns
them. The WebUI and the Aurora client needed no changes. Memory extraction excludes them, or DAWN would
re-learn its own retrievals.

Main modules: `session_prefix.c` (the seam: freeze, append, boundaries, save), `prefix_in_force.c` and
`prefix_message.c` (what is in force, the one prefix renderer), `prefix_tools.c` (tool changes),
`session_compaction.c` and `llm_compaction.c` (seam compaction), `llm_turn_blocks*.c` and
`llm_history_rows.c` (canonical blocks and their rows), `focus_incremental.c` (incremental focus),
`llm_claude_route.c` / `llm_claude_betas.c` (Anthropic routes and betas), `llm_cache_monitor.c`
(telemetry and alarms).

---

## 3. Per-provider design and measured behavior

### 3.1 Anthropic (first party)

**Breakpoints.** Three of the four allowed: the last tool, `system[0]` (which caches tools + system),
and a moving breakpoint **B** on the last block of the last user message. With an append-only history
B caches both the tool loop within a turn (iteration *i* writes, *i+1* reads) and the next turn. The
lookback window (validated against the docs) is 20 positions per breakpoint, where a run of
`tool_use` or `tool_result` blocks counts as one; DAWN appends at most ~4 positions per iteration with
an iteration cap of 8, so no intermediate breakpoint was needed.

**Minimum cacheable prefix** (silent below it; no error, just no write): 512 tokens on Opus 5 / 5.5
and Fable 5 / 5.1; 1024 on Opus 4.8, Sonnet 5, Sonnet 4.6; 2048 on Opus 4.7; 4096 on Opus 4.6 and
Haiku 4.5. The value is not monotonic across generations, so it is per-model data.

**Pricing** (verified 2026-09-27): reads 0.1× input, except 0.05× on Opus 5.5 and 0.025× on Fable 5.1;
writes 1.25× (5-minute TTL) or 2× (1-hour TTL). These are `[cache_pricing.*]` rows in `models.toml`.

**TTL rules** (validated): a read refreshes the timer at no cost; lifetime runs from request start to
next request start, so generation time counts against it. Under 5 minutes between requests, 5m is
strictly cheaper; 5–60 minutes is the only range where 1h pays; over an hour neither helps.

**Binding controls.** Every Claude request carries the beta `thinking-binding-controls-2026-08-01` and
`thinking.block_binding.prefix_mismatch_behavior`. The daemon always sends `drop_block`; only the test
harness sends `error`. `input_transformations` is parsed from `message_start`, the final
`message_delta` and non-streaming bodies (`llm_claude_binding.c`). A `prefix_binding_mismatch` entry
is a harness bug and raises an alarm; a `model_binding_mismatch` after a model switch is expected and
unbilled. Probe results (2026-09-27): Opus 5.5 enforces (an edited replay is caught as
`blocks_modified` / `first_message_rewritten`); Opus 4.8 and Sonnet 5 accept the controls but do not
enforce; on Haiku 4.5, `disabled` plus `block_binding` is a 400 (`Extra inputs are not permitted`), so
`block_binding` is attached only under adaptive or enabled thinking.

**Beta handling.** Betas are derived from what the body carries. A rejected beta is turned off and the
request retried once, but rejection matching is deliberately narrow (the two exact error shapes the
API returns) and recorded per model: an enforced account's binding-mismatch 400 also names
`block_binding`, and treating it as a rejection would disable the one control protecting that
account. Retries are bounded and clear their marks, because a leaked retry mark once risked re-sending
an earlier request and running its tools twice.

**Cache diagnostics** (beta `cache-diagnosis-2026-04-07`, first party only). A probe found the real
contract differs from the published one: a request's fingerprint is stored only when it carries the
header *and* a `diagnostics` object (`{}` when there is no previous id); otherwise every lookup returns
`previous_message_not_found`. The result is in `message_start.message.diagnostics.cache_miss_reason`.
**TTL expiry is not diagnosed:** after a 330-second idle with an identical prefix, the cache missed and
`diagnostics` was `null`. Diagnostics name prefix changes only, so DAWN identifies TTL misses itself
from the request gap.

**Mid-conversation system messages** work on Opus 5 / 5.5, Opus 4.8, Fable 5 / 5.1 and Mythos 5 / 5.1
(listed in `models.toml [mid_system]`), not on Sonnet 5 or Haiku 4.5 (a 400). They must follow a user
message. Other models get an `[Operator note]` text block.

**Inline tool changes** (beta `inline-tools-2026-09-15`, first party, models in `[inline_tools]`): a
new or changed tool definition is appended as a `tool_addition` block in a system message, with no
boundary and no reasoning lost. Live: with a code-graph MCP server down at boot and up later, Opus 5.5
received 8 definitions in place, the API accepted them, and the next turn read exactly the expected
amount; a model without inline support took one declared boundary (0% once, 92% the next turn). A
removal emits nothing: the definition stays and execution refuses.

**Thinking config.** A per-model capability table (`models.toml [thinking.*]`, resolved by
`llm_capabilities.c`) replaced four name heuristics. Every request sends an explicit config; a model
that cannot disable thinking gets adaptive at its lowest effort, and the UI says so. Two old "history
clamps", which turned thinking off when history held `tool_use` without a thinking block, were
removed: every currently served Claude model returned 200 in each case they guarded, and the clamp was
turning reasoning off for the rest of a conversation whenever adaptive thinking legitimately skipped a
tool call. Utility calls (compaction, extraction, scheduler) resolve to the cheapest legal setting
instead of a hard-coded `disabled`.

**Image parity.** Images replay byte-identical: no re-encode or downscale on replay. Captured images
sit inside their `tool_result`, sized at ingestion (≤1568 px long edge, ≤1.5 MB stored); a capture
over a model's per-request limit is refused rather than evicting earlier images in place.

### 3.2 OpenRouter, Anthropic models (P1.6)

OpenRouter `anthropic/*` models moved from DAWN's Chat Completions mirror to OpenRouter's Anthropic
Messages endpoint (`POST /api/v1/messages`) and go through the Claude formatter. Probed 2026-10-02 and
2026-10-05 on Opus 5.5:

- **Forwarded:** signed thinking, verbatim replay, binding controls (an edited replay is reported as
  `prefix_binding_mismatch`; `error` mode returns the same 400 as first party), cache markers,
  mid-conversation `role: system`, `session_id`. Usage comes back in Anthropic's shape plus OpenRouter
  cost fields.
- **Different:** `input_transformations` is **omitted** when nothing was dropped (first party sends
  `[]`), so absence is read as empty. `diagnostics` is accepted but nothing comes back, so it is not
  sent. **`tool_addition` is rejected** by OpenRouter's own validator, so tool changes on this route
  always take the folded path (a declared boundary), chosen by route, never by waiting for a 400. An
  unknown beta is ignored.
- **Pinning:** `provider: {order: ["anthropic"], allow_fallbacks: false}`, because caches and binding
  are per platform and a silent switch to Bedrock or Vertex is a full miss. `session_id:
  "dawn-conv-<id>"` asks for sticky routing.
- **Streaming quirks:** OpenRouter commits its 200 early and keeps the line alive, so an overload can
  arrive as an in-stream `error` event. A Claude stream that reports `error`, or ends without
  `message_stop`, now fails the call instead of saving a truncated answer (a latent first-party bug
  too). Upstream error text is read from `metadata.raw`.
- **Route vs. provider.** A request's route (wire format) is now separate from its provider (key,
  pricing, metrics): one predicate, `llm_uses_anthropic_messages()`, selects the wire, requiring the
  `openrouter.ai` host; usage books under OpenRouter; capability lookups use the Anthropic model id.
  Slugs with a `:variant` suffix take the same route.
- **Blocks across routes.** Probes showed blocks minted on either route replay on the other in `error`
  mode. DAWN still keeps them apart deliberately: a reasoning block's carrier is the endpoint host plus
  a keyed tag of the API key, and that probe's conditions cannot be assumed for every account.

Live after shipping: signed thinking on every turn, no drops, reads of 84–97%, and after a reload from
the database the next turn read 61,472 of the previous 61,474 tokens.

### 3.3 OpenAI: Responses vs. Chat Completions

- **Responses** (the precedent: an earlier reorder took cross-turn caching from 9% to 99%) keeps the
  current turn's context as a user item before the question, reasoning items replayed in original
  order next to their `function_call`. Direct OpenAI on Responses read the whole prior prompt across
  turns (27,080 / 27,083).
- **OpenAI drops previous-turn reasoning items**, so a turn whose tail was reasoning-heavy has that
  tail re-billed once. A partial read on OpenAI is therefore never treated as a defect.
- **Chat Completions, reasoning models.** In live use through OpenRouter (`openai/gpt-5.5`), tool
  iterations read 83–92% and later turns 95–99%; the one large cross-turn miss (to the first question)
  followed a reasoning-heavy turn, consistent with the reasoning drop above. Small probes that added
  under ~1K tokens per step read only to the first user message, but that was likely the increment
  being too small to cache, not a property of the route. OpenAI's own guidance is that Responses caches
  reasoning models better than Chat Completions; DAWN routes GPT-5.4 and later to Responses (a name
  rule, also because those models reject reasoning effort with tools on Chat Completions).
- `prompt_cache_key` (a content-neutral `dawn-conv-<id>` value) is sent on Chat Completions only to
  `api.openai.com` and `openrouter.ai`, because a strict compatible server can reject unknown fields.
- The forced-text final call keeps its tools and sends `tool_choice: "none"` instead of removing the
  tools, which had changed the front of the prompt.
- An encrypted reasoning item is bound to the model and organization that produced it, so it replays
  only to the same carrier (host + key tag) and model.

### 3.4 Gemini

Gemini 3.x showed no implicit caching: Gemini 3.1 Pro via OpenRouter read 0 tokens on identical
repeats. An earlier claim that tool definitions block implicit caching turned out to conflate the
*explicit*-cache rule and was not verified; DAWN's own logs showed 8,793 cached tokens on a Gemini 2.5
tool-loop call. The realistic target was "stop defeating it" (the fold below). Gemini thought
signatures are kept per tool call and resent across Gemini model switches, as Google advises. Explicit
caching (`cachedContent`, or `extra_body.google.cached_content` through the compatibility layer) is
future work.

### 3.5 Local llama.cpp / Ollama

On a local server the cost is latency (~482 tok/s prompt processing on the reference Jetson), and the
cache is the KV cache of a server slot. The Chat Completions family folds the turn's context and any
operator notes into the question as **one** user message and keeps one leading system message. That
keeps strict alternation and suits strict Jinja templates, which reject a second system message (the
reason DAWN merges leading system messages). No `id_slot` pinning: recent llama.cpp similarity-based
slot selection and `--cache-ram` already limit slot thrash.

**The Qwen 3.6 template trap.** The embedded Qwen 3.6 template drops `<think>` content from assistant
messages before the latest question unless `preserve_thinking` is set, so a tool-call message renders
differently once the next question arrives and the KV cache stops there. Sending
`chat_template_kwargs.preserve_thinking: true` fixed it: a probe's second turn re-processed **843
tokens without it and 21 with it**. In the daemon, turns after a tool turn then read everything before
the new question (42,075 / 43,116; 43,302 / 43,525; 52,885 / 53,236). Ollama reports no cache reuse,
so only llama.cpp is judged by the monitor.

---

## 4. Observability

**P0 telemetry.** Every provider path fills one usage record at the token-producing site, with the
provider taken from the endpoint (never guessed from the model name), the call kind (turn, tool
iteration, job, research, extraction, compaction, briefing, observe), the conversation, separate
hashes of tools, system, model and thinking config, and the gap since the previous call on the same
key. It produces one grep-stable line per call:

```
LLM cache: provider=claude model=claude-sonnet-5-5 kind=tool_iter conv=1705 iter=1 prompt=50816
  read=46551 write=4263 uncached=2 coverage=91% expected=46551 state=warm gap=5.6s tools=… system=…
  thinking=adaptive/low images=0 miss=-/0 drops=- output=…
```

and one row in `llm_usage_log` (90-day retention, written through a ring the main loop flushes),
including binding-drop counts. The celebratory "cache hit (90% savings!)" lines were demoted to debug.
P0 also fixed accounting defects: tokens booked to the wrong provider, cache writes not parsed on Chat
Completions, llama.cpp `timings.cache_n` parsed but unused, non-streaming Claude reporting no usage,
flat read discounts wrong for every current model, and a missing `models.toml` row that ran Opus 5.5
on a 200K default window (compaction fired near 170K instead of ~850K).

**Expected read.** For Anthropic, within a conversation, `expected_read(i+1) = read(i) + write(i)`.
No token-per-byte conversion is needed. For local llama.cpp, expected is the previous prompt on that
key, unless another key ran on the server in between.

**Cache states** (P3), carried in the log line, in `llm_usage_log.cache_state` and on the WebUI
`metrics_update` frame: `first`, `ttl`, `tools`, `system`, `model`, `thinking`, `rewritten`,
`shared`, `images`, `warm`, `warm_miss` (and `untracked` on the frame for a call that isn't judged).

**Known events that are not misses:** a compaction apply, a forget's withdrawal, a turn rollback, a
history reset (a research round, a new chat) are all `rewritten`. A TTL gap is measured from the
request start, so a long reply plus a pause reads as `ttl`. `tool_choice` is part of the tools hash.
A conversation's first image is a cold state (`images`); later image calls stay `warm`, because only
the newest message carries the new image. A P1.4 inline tool change stays `warm` (it is appended
after the cached prefix, and excluding it would hide a regression).

**Warm-miss rules:**

| Provider | Fires when |
|---|---|
| Anthropic (first party), local llama.cpp | On a warm call, read < 0.9 × expected, **or** read + write = 0 on a prompt of 4096+ tokens (the largest minimum cacheable size; catches a model that ignores the markers, where expected would stay 0) |
| OpenAI (Responses and Chat Completions), OpenRouter's OpenAI and Anthropic models | No expected read; three consecutive warm calls on one key with read = 0 and prompt ≥ 1024. A partial read is never flagged (OpenAI drops earlier reasoning by design; OpenRouter routes Anthropic best-effort without a pinned key). Binding drops are still reported |
| Gemini, other OpenRouter vendors, Ollama, side calls, the local microphone | Not judged (`untracked`) |

Warnings are rate-limited to one per (provider, model) per 10 minutes and include the conversation,
expected, read, gap and Anthropic's `cache_miss_reason` when present. Local misses log at INFO.

**Dropped-reasoning alarm.** A binding drop logs a rate-limited warning and, once per conversation,
sends admins a persistent WebUI toast in plain language ("Earlier reasoning was dropped in a
conversation. This is a DAWN bug: please report it with today's log."). The LLM worker only queues
the alert; the main loop's one-second flush, holding no locks, delivers it through a weak hook the
WebUI overrides.

**`dawn-admin cache stats [--since 24h|7d] [--provider X]`** is the first reader of
`llm_usage_log`: per provider, model and kind, calls, prompt tokens, coverage (read / prompt), input
tokens' worth saved (list-price multipliers; writes count against), warm misses and binding drops.
One indexed `GROUP BY`; no rollup table. (`models.toml` holds cache multipliers, not per-token prices,
so the figures are in tokens, not money.)

**Thresholds are constants, not settings.** θ = 0.9, the 10-minute rate limit and the streak of 3 are
harness internals with no meaning to a user; a setting would only let someone hide a real regression.
The planned `[llm.cache_monitor]` config section, an admin panel, a footer coverage figure, a briefing
digest and daily rollups were all dropped as answering questions `cache stats` already answers.

---

## 5. Phases and outcomes

Commit hashes are durable (the repository merges with merge commits). Phases ran in this order on one
branch, with no intermediate release, so no user ever ran a partial state.

| Phase | What shipped | Commits | Measured result |
|---|---|---|---|
| P0 | Per-call telemetry, `llm_usage_log`, cache pricing in `models.toml`, Anthropic diagnostics, accounting fixes; an A/B harness and baseline (FRIDAY suite + citation cite-rate on Opus 5.5 and Sonnet 5) | `096a7f95`, `d52607f4` | Baseline FRIDAY 114/122 (Opus), 113/122 (Sonnet); found the missing Opus 5.5 context-window row |
| P1.0 + P1.1a/b | Binding controls on every Claude request (`drop_block` in the daemon), `input_transformations` parsing, narrow beta-rejection handling; capability table, explicit thinking config on every provider, history clamps removed, session effort honored everywhere, capability data to the UIs | `c6543f22`, `04722301` | Self-test on Opus 5.5: WIRED; "disabled" now honest, no omitted thinking field |
| P1.1c | Ordered canonical turn blocks, per-provider capture and replay (Claude, Responses, Chat Completions/OpenRouter/Gemini), carrier-bound reasoning | `3813bf4f`, `8140e35b`, `9c4de845`, `80d88435` | Multiple, empty and redacted thinking blocks replayed verbatim |
| P1.1d | `messages.llm_blocks` persistence and reload; legacy voice rows scrubbed; carrier tag as a keyed BLAKE2b | `849bd3d1` | 200-turn reload with 8 KB thinking each: 20 ms vs 1 ms without blocks |
| P1.2a + P1.7 | Frozen prefix, kind rows, appended instructions/directives, tool-set freeze, stable `[M#]` handles, signature persistence through reload | `6c933f4c` | Cache 97–98%; byte-identical rebuild after restart; the confirmed 400 eliminated |
| P1.3 | Compaction only at the turn seam, async summary applied at the next seam, one transaction for summary, watermark, floor and the turn's rows; mid-turn overflow valve | `8f948fb5`, `de634c23` | Live: 81.7% of 2.39M prompt tokens read from cache; prefix stayed cached across the compaction boundary |
| P1.4 + P1.5 | Tool definitions frozen by value, `tool_change` rows (inline on first-party capable models, folded elsewhere); captures persisted, image parity, no JPEG fallback | `04e6d506` | Restart + reload: first turn read 51,698 / 53,695; inline tool addition with no boundary |
| P1.2b | Incremental focus derived from what the retained context shows; reference lines; neutralizer hardened against forged item lines | `d0eeeb9e` | Context chars/turn 867 → 764; warm reads 98–99%; judge pass unchanged at 91.7% |
| P1.6 | OpenRouter Anthropic models on OpenRouter's Messages endpoint; Claude non-streaming and in-stream error fixes | `89ca406b`, `59dde09d` | Reads 84–97%; reload read 61,472 / 61,474 |
| P2 | Chat Completions family: `prompt_cache_key`, Qwen `preserve_thinking`; most of P2 (the fold, forced-text tools) had already landed in P1.2a | `ffad87cc` | Local post-tool turns read everything before the new question; OpenAI via OpenRouter 95–99% after low-reasoning turns |
| P3 | Cache states, warm-miss detectors, dropped-reasoning toast, `dawn-admin cache stats` | `8ed2552d` | Live: first → warm on every call; 8-day stats showed every binding drop predated P1.3/P1.1d |
| P4 | Closed with no code after measurement (see §6) | — | — |

---

## 6. Decisions and the reasons

**`drop_block` always in the daemon; `error` only in tests; no setting.** A user's turn should
degrade (one dropped block, a cache miss) rather than fail outright if a harness bug slips through, and
every drop raises an alarm. `error` belongs in CI layout tests and probes, where an edit must fail
loudly. A knob would be an internal no user would know to look for, and could only be used to break
things. The live suite runs through the daemon on `drop_block` and asserts
`input_transformations == []`, which is equally strict.

**Store the prefix bytes, not only a hash.** Hash-only would make every persona or settings change a
reasoning-stripping boundary on the next reload of every conversation. A content-addressed table costs
one row per distinct prompt.

**Keep-tail compaction, made valid,** rather than server-side compaction or summary-only: summarize the
older range, keep the recent tail, strip earlier reasoning at the boundary, and run only at the seam.
The summary is untrusted text (it may quote tool content) and is neutralized and framed as such.

**Static 5-minute TTL.** Eight days of Claude conversation calls showed 11 TTL expiries (~670K tokens
rewritten) against 4.7M written; a 1-hour TTL everywhere would add ~3.5M token-equivalents. The shared
tools entry (~38K tokens, its own breakpoint), simulated over 631 calls, needed 61 writes at 1.25×
(5m) vs. 37 at 2× (1h): break-even. TTL never adapts per request.

**Pre-warm skipped.** A `max_tokens: 0` request on wake-word detection would write the cache while ASR
runs. A probe on a 60K prompt measured time to first token: Sonnet 5 cold 1.35 s / warm 0.85 s; Opus
5.5 cold 3.21 s / warm 1.47 s. The gain is real for Claude voice users; the maintainer deferred it.

**Summarizer A/B deferred.** A retain-list summarizer prompt and a plain-text transcript input affect
compaction quality, not caching.

**Fork-prefix reuse is measure-only.** The vendor recommends side calls copy the parent's system,
tools and model. DAWN's compaction suppresses tools and resolves thinking to the cheapest setting, so
it can never share the parent prefix.

**`clear_at` not used.** The mid-conversation `clear_at: "next_user_message"` system message costs no
input tokens after its turn, but it is a limited beta, unavailable on Sonnet 5, and clears at the first
`tool_result`-only message, so it would need re-appending every iteration. Retained text plus
incremental focus bounds growth instead.

**Per-message effort not pursued.** Effort changes reset the cache, but they are rare manual toggles.

**No config section and no rollups in P3** (see §4).

**`context_overflow_midturn` not tracked.** P1.3 ships a valve (past the hard threshold mid-turn, a
foreground turn gets a no-tools final answer; a background job ends the turn and continues in up to 8
continuation turns, each compacting at its seam). A context-length rejection that still happens fails
the turn visibly, and a failed call has no usage record to tag.

**Route chosen by data, never by a 400.** Inline tool changes, mid-conversation system messages and
the OpenRouter route are decided from `models.toml` capability lists and the endpoint host, so behavior
is deterministic and testable offline.

**Reasoning artifacts are never translated.** Text and tool calls translate exactly between providers;
signed or encrypted reasoning is replayed only to its own carrier, format and model family, and foreign
reasoning is omitted, never serialized as text. A→B→A keeps A's blocks.

**OpenRouter gets no expected read**, only the zero-read streak, until a run on an ordinary OpenRouter
key (OpenRouter's own Anthropic account pool, not a pinned key) shows that `session_id` sticky routing
holds; the maintainer's setup could not demonstrate it.

---

## 7. Lessons for similar systems

1. **Caching is a byte-prefix match on every major provider.** Anything that varies per turn belongs at
   the end of the request, never in the system prompt. Breakpoints do not make placement irrelevant.
2. **Treat the request as an append-only log** and enumerate the few places it may change (new
   conversation, compaction, tool-set changes you cannot append). Everything else is appended.
3. **Measure coverage, not hit rate.** "Any cache read" was true on 80% of calls while 62% of tokens
   were uncached. Log read, write, uncached and an expected read per call, keyed at the source;
   reconstructing pairs from interleaved logs breaks under concurrency.
4. **Store what the model returned, exactly.** Signed and encrypted reasoning, block order, empty
   blocks and image bytes must survive a reload byte for byte. A daemon restart is the test.
5. **Everything in the live history must be persisted,** including hints and notices you consider
   transient; otherwise every reload is a miss.
6. **Verify vendor contracts by probe.** The diagnostics contract, OpenRouter's omitted
   `input_transformations`, Chat Completions caching only to the first question, and the Qwen template
   behavior all differed from what documentation or intuition suggested.
7. **Test with the strictest account settings your users may have.** New accounts were enforced before
   the maintainer's; the vendor says so explicitly.
8. **Keep capabilities as data** (thinking modes, mid-conversation system support, inline tools,
   cache pricing, minimum sizes), with a compiled-in fallback copy so a stale config file cannot break
   requests.
9. **Do not over-build alarms.** One well-defined detector per provider family, with known events
   excluded first, beats a set of tunable floors nobody tunes.

---

## 8. Open and future work

- Native Gemini API with explicit `cachedContent` (also needed for visible thinking).
- Voice pre-warm on wake-word detection (numbers in §6).
- Summarizer prompt A/B and a plain-text summarizer input.
- Confirm OpenRouter sticky routing on a non-pinned account before judging its expected reads.
- OpenRouter's Responses endpoint for `openai/*` models (untested).
- Withdrawing forgotten items from stored compaction summaries; bounding memory-extraction input to the
  extraction model's window.

---

## Sources

- Anthropic, Prompt caching: https://platform.claude.com/docs/en/build-with-claude/prompt-caching
  (prefix order, breakpoints, lookback, minimum sizes, TTL, preserved-thinking binding, cache
  diagnostics, mid-conversation system messages and tool changes).
- Other vendor behavior in this document (OpenRouter's Messages route, OpenAI Responses and Chat
  Completions, Gemini, llama.cpp/Qwen templates) comes from DAWN's own probes and daemon telemetry,
  dated where cited.
