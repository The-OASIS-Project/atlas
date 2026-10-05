# Large Tool Results: Bounded Views over Stored Results

**Status:** Shipped (2026-10-01) on DAWN branch `adaptive_dwell`.

| Phase | What | Commit |
|---|---|---|
| P1 | MCP result normalization; the neutralizer decodes escapes | `313f0826` |
| P2a | The view module (`llm_tool_view.c`) and its tests | `0faa9689` |
| P3 | The `tool_results` store (schema v97) and the `result_read` tool | `2035ee9d` |
| P2b | The tool loop's view stage | `9205b45d` |
| P4 follow-up | The header steers tallies to `count` / `distinct` | `bfbed7c8` |

The design was reviewed before any code was written (plan, architecture, security and efficiency
lenses: go, with changes), and again after each phase. This document records what was built. Where
the plan changed during the build, the section says what was built instead and why.

**Motivating case:** conversation 1554, 2026-09-29.
- A parallel batch of three codebase-graph MCP calls returned 205 KB (`cbm_get_architecture`
  149 KB, two searches of about 27 KB each).
- One tool iteration added 76K tokens to the request (47% → 85% of a 200K window).
- That forced a compaction, and the kept tail still carried the results (110K tokens on the next
  request).
- About 57% of the 149 KB was the same payload twice: the MCP client passed the raw JSON-RPC result
  through, `structuredContent` and its text copy both.

---

## 1. Goal and principles

The model must be able to get at any data a tool returned. Nothing is thrown away. Only the default
cost of a big result changes.

- **Never a blind cut.** Over budget, the model gets a *view*: the structure for JSON, the head and
  the tail for text. It carries a header with the total size and a handle.
- **The full result is kept** for the conversation's life and is readable through `result_read`.
- **Append-only.** A view is made once, when the result arrives, and it is what history and the
  stored row hold. Nothing is rewritten later, and nothing recomputes a view. (DAWN's conversation
  requests are append-only so providers can cache them; a view that changed later would break the
  cache.)
- **Budgets follow the model's window and the room left**, in characters (§6).
- **Steer, don't just shrink.** When the producing tool can narrow itself, the header says how.
- **Every byte that enters a request is neutralized last**, after every transform.

## 2. Prior art (research 2026-09-30; URLs checked then)

- **Claude Code** saves an MCP result over 25K tokens (`MAX_MCP_OUTPUT_TOKENS`; warns at 10K) to
  the session's `tool-results` directory and replaces it with the path. A tool may raise its own
  limit with `_meta["anthropic/maxResultSizeChars"]`, up to 500K characters.
  <https://code.claude.com/docs/en/mcp>
- **Cursor** writes long shell and MCP outputs to a file ("truncat[ion] … can lead to data loss"),
  and the agent `tail`s it, then reads more. <https://cursor.com/blog/dynamic-context-discovery>
- **OpenHands** middle-truncates and can save the full content with a reference. **Manus** keeps
  compression "restorable" (keep the URL or path, drop the body).
- **Codex CLI** does a token-limited head+tail cut (`tool_output_token_limit`). An open issue
  (#14206) reports that it corrupts JSON, and asks for spill-to-file with an artifact id, a preview,
  and ranged or grep reads.
- **Anthropic's tool-design guidance:** pagination, filters, a concise/detailed format, and
  truncation messages that say how to ask better.
- **SWE-agent** (Table 3):
  - capped search with "write a more specific query" solved 18.0%, against 12.0% for iterating
    over the matches;
  - a 100-line window solved 18.0%, against 12.7% for the whole file.
- **Not adopted:**
  - programmatic tool calling (needs a sandbox; DAWN runs none);
  - mid-conversation clearing of old results (a history edit; it belongs only at the compaction
    seam);
  - LLM summaries of results (JetBrains: plain masking matches summarization at half the cost).

Vendors that cut treat JSON as text. Vendors that spill (Claude Code, Cursor) give no structure
preview. What this design adds:
- a valid-JSON structure view;
- a JSON-path retrieval tool, since there's no shell to `grep` or `tail` with;
- a budget from the window and the room left, where every vendor's cap is fixed.

No published study compared structure views with head+tail cuts. The P4 A/B (§12) is our evidence.

## 3. Where it sits

```
tool callback → llm_tools_execute (llm_tools.c): result[8192] / result_extended (unbounded)
tool loop (llm_tool_loop.c) → measures the batch budget, sets the session's read budget
   → llm_tools_execute_all(calls, results, finish, ud)
        finish = the view stage (llm_tool_views_apply.c): for the batch, store what's over its
        share, replace it with its view, then finish each result (neutralize + mask, header,
        tell the WebUI what the model sees)
   → llm_tools_prepare_followup → append to history → persist the tool turn
```

- **The stage** (`src/llm/llm_tool_views_apply.c`, Layer 2) is the batch's *finish callback*:
  `llm_tools_execute_all(calls, results, finish, ud)` runs `finish` on the raw batch once every
  call has returned. That is the one place with the whole batch and the turn's model, window and
  calibration. The pure arithmetic (budget, split, header, read share) lives in
  `src/llm/llm_tool_views_plan.c`.
- **The finish contract is structural.** A result that the finish callback leaves unfinished is
  finished by `llm_tools_execute_all` itself (and logged), so nothing leaves it raw, whoever calls
  it. `llm_tools_finish_result` neutralizes and masks, puts DAWN's header on, marks the result
  finished (once only) and tells the WebUI what the model sees; the "thinking" state follows the
  announcements. A call outside a batch (a plan's steps) is neutralized and finished on every path,
  error paths included. `tests/test_llm_tools_finish.c` holds the contract.
- **Left alone by the stage:** results marked `skip_followup` (spoken directly), `!should_respond`
  (handled by the tool itself), vision results, and tools flagged `result_whole` in their metadata
  (`render_visual`: the WebUI renders the markup from the full text). The exemption is keyed on the
  tool, never on the content: a web page containing `<dawn-visual` gets no exemption. The WebUI's
  visual transcript entry (and the visual saved with the reply) is keyed the same way.
- **Neutralization happens on the final content**, the whole result when it's small, the view when
  it's large. Two gains:
  - It runs after every transform. A JSON view re-serializes, and json-c decodes `\uXXXX`, which
    could revive an imitation the neutralizer never saw.
  - It costs O(budget), not O(input). (At design time the neutralizer ran at ~31 MB/s with 13 bytes
    of scratch per input byte.)
- **The completion notify** (`notify_tool_execution`, the WebUI's tool pill) happens when a result
  is finished, so it shows what the model saw.
- **Downstream,** `tool_result_content()` returns the view to history, the stored message row, the
  event log and replay. Reload replays the stored row as it is.
- **Where the view goes:** into `result[]` if it fits in 8 KB (`result_extended` freed), otherwise
  it is `result_extended`.
- **Out of memory while viewing:** a fixed notice ("Tool result too large to show; call the tool
  again with narrower arguments.") replaces the result. Never the whole result.
- **Out of scope for this work** (filed separately, §13): `plan_executor.c` steps and scheduled
  briefing steps, which call tools outside the loop; `url_fetch` narrowing parameters; secret
  redaction of stored results.

## 4. MCP results: unwrap in the client (P1)

`src/tools/mcp_result.c` (`mcp_result_text`) builds the normalized result from the parsed JSON-RPC
`result` tree that `mcp_client.c` already holds, with no second parse. Before this, the client
serialized the whole tree.

- **`structuredContent` present (and not null):** it is the result, and the text parts (the spec's
  mirrored copy) are dropped.
- **Otherwise, `content` parts in order,** separated by newlines (a part that renders nothing adds
  no separator):
  - `text`: the text, as json-c already unescaped it. The view stage's single parse decides whether
    that is JSON.
  - `resource` (embedded) **with `text`:** the text is inlined like a text part.
  - `resource` with a `blob`, and `resource_link`: a one-line inert reference (name, mime, URI
    shown, never fetched; line-neutralized, length-capped; for a blob, its decoded size). Base64 is
    never decoded to text.
  - `image` and `audio`: a one-line reference (mime, bytes). The MCP path has no vision route;
    adding one is a separate feature.
  - A part with no `type`, or a type this doesn't know: the part's JSON, as sent.
- **A result shape it doesn't know** (no `structuredContent`, no non-empty `content` array): the
  whole result's JSON, as sent.
- **`isError`** marks a tool error only when it is a JSON boolean `true` (`"false"` as a string
  doesn't count).
- **Serialized** with `JSON_C_TO_STRING_PLAIN | JSON_C_TO_STRING_NOSLASHESCAPE`. The default
  escapes `/`, which cost 2.4% on the code-graph server's paths, and spacing adds about 8%.
- **The code-graph path scrub** (`code_project_namemap_scrub`, in `mcp_bridge_tool.c`) runs after
  normalization, on the final text.
- **At registration**, `mcp_bridge_tool.c` reads `_meta["anthropic/maxResultSizeChars"]` from the
  server's `tools/list` entry (a positive JSON integer) and sets the tool's `max_result_chars`
  (§6), capped at 500,000.
- **Measured on the code-graph server:** 149,489 B became 64,814 B (−57%). The 205 KB batch became
  about 90 KB before any view.

## 5. The view (`src/llm/llm_tool_view.c`, pure)

Text in, budget in bytes, path prefix in, view out.
- **Deterministic:** the same input and budget give the same bytes. Views are stored and replayed,
  never recomputed; determinism is a tested property, not something the design relies on.
- **Every render writes into a buffer capped at the budget**, so a failed attempt costs O(budget),
  not O(input).
- A budget below 256 bytes is raised to it. A NUL byte is shown as a space (a view is a C string).

**JSON mode** runs when the first non-space byte is `{` or `[`, the text is at most
`LLM_TOOL_VIEW_JSON_MAX_BYTES` (4 MB), and it parses as **strict** JSON whose numbers json-c holds
exactly. Text with NaN, Infinity, single quotes, comments, an integer past 64 bits, a double past
its range, or a number like `1.` gets the text view instead (json-c accepts some of these, or clamps
silently; a view must never show a changed value). The same rule (`llm_tool_view_parse`) is what
`result_read` uses (§8). json-c's nesting limit is 32.

- **Objects:** the first `keys` keys, then a `"N more keys at <path>"` marker under the key `"..."`
  (or `"...#2"`, `"...#3"`, so it never repeats a real key). A key longer than 64 chars shows its
  head, its length and an FNV-1a hash: `<head>...(N chars, #1a2b3c4d)`, which `result_read` resolves
  back to the key.
- **Arrays:** the first `items` items and the last one, with one marker string between them:
  `"... 324 more items at $.results[3:327], each {id:int, name:str, file:str}"`. The item shape
  comes from at most 32 evenly spaced samples (up to 8 keys; else the item types, such as
  `str|null`), computed at each render. An elision is made only when what it hides outweighs its
  marker; fewer than 4 items or keys are shown, not elided.
- **Too deep:** `"...[N items at <path>]"` or `"...{N keys at <path>}"`.
- **Strings:** the head, then `"...(12,345 chars at $.nodes[0].body)"`. A string is cut only when
  at least 48 bytes past its head.
- **Values keep their source text.** Numbers are emitted as sent (only an integer `-0` shows as
  `0`); json-c would otherwise re-render doubles. A view never changes an id.
- **Paths** use `$.name` for identifier keys and `$["…"]` (a JSON string, escaped, one line) for any
  other key, so untrusted key names never appear bare inside DAWN's marker text. The grammar lives
  in one module (`llm_tool_view_path.c`), so a path copied from a view always parses.
- **The shape: tighten, then loosen.** Rendering starts at items 5, strings 400, keys 20, depth 16.
  - *Tightening* (the shape overflows the budget): halve strings, then items, then keys, then depth,
    down to the skeleton (items 1, strings 40, keys 8, depth 3). If even the skeleton overflows,
    the view falls back to text mode, which always fits, so the ladder ends.
  - *Loosening* (it fits with more than half the budget unused): grow, one at a time, each dimension
    that cut something, doubling it, keeping each step that still fits, up to the `_MAX` limits
    (items 4,096, strings 16,000, keys 1,024, depth 32). A step that overflows, or a dimension that
    cuts nothing, is done. The number of renders is small in practice (each 0.08–0.13 ms on the
    Jetson at design time) but no fixed bound is promised.
- **Subtrees** (`result_read path`) render with the path prefix, so markers name paths from the
  original root. Slices of an array number their items from the original indices.

**Text mode:**
- Numbered lines (`cat -n` style): about ⅔ head and the rest tail at line boundaries, with
  `... 1,204 lines (88,112 chars) omitted: lines 41-1244 ...` between them.
- A line too long to fit (minified JSON too big for JSON mode) is cut by characters, UTF-8 safe:
  ` ... N chars omitted ... `.

**Header**, framed with the conversation's tag like DAWN's other frames (`llm_tool_views_header`).
For a JSON view with `result_read` offered:
```
[Tool result shortened (<tag>): 64,814 chars, shown as a view.
 Full result: [tool-result trs_<id>]. result_read: count / distinct ($.a, field) tally a whole array in one call; path ($.a[10:20]), grep (text) and read (lines) return parts.
 The tool takes "limit": a narrower call may be all you need.]
```
- **A text view** offers only `Full result: [tool-result trs_<id>]. result_read: read (lines),
  grep (text).`
- **A result too big to store whole** (§7) adds: "It was too big to keep whole: its first and last
  parts are kept."
- **Without `result_read`** (it isn't in this request's tool set: a conversation whose tool set was
  frozen before it existed, a no-tools turn, a surface that withholds it), there is no handle and
  the line reads "Call the tool again with narrower arguments for more."
- **`result_read`'s own over-budget answer** (`is_read`) says "Read a smaller part: a deeper path,
  or a narrower pattern; to tally an array, count / distinct ($.a, field)." (JSON) or "Read a
  smaller part: a line range, or a narrower pattern." (text), and never a narrowing line.
- **The narrowing line** appears only when the tool's schema has a parameter from a fixed list
  (`limit`, `max_results`, `count`, `page`, `offset`, `query`, `filter`, `fields`). It names that
  parameter only. **The header never names the tool**: an MCP tool's name is its server's choice,
  and so is untrusted text.
- **When the full header doesn't fit** its room (640 bytes), a minimal one keeps "shortened", the
  size and the handle: `[Tool result shortened (<tag>): N chars. Full result: [tool-result trs_…].]`.
- **The neutralizer learns the frame.** "[Tool result shortened" in any spelling joins its defuse
  set, and the header goes on *after* neutralizing, so untrusted content can't forge a header. The
  bare handle token `[tool-result trs_…]` is not defused: a summary must keep it (§9), and a forged
  one can only name a result the reader's scope already allows or refuses.
- **Changed from the plan: the omission markers are not in the defuse set.** The final content (the
  view, its markers included) is neutralized last, so defusing the markers would defuse the view's
  own. The residual risk is integrity only: a forged marker in a result can misstate what was left
  out (as any untrusted text can lie), or name a handle, which `result_read`'s scope refuses unless
  it belongs to the reader's own conversation.

## 6. Budgets

**In characters, never tokens (the maintainer's decision, 2026-10-01, after the P2b live test).**
The first build priced the batch in tokens and turned them into characters with the conversation's
calibration factor. That factor describes the whole request, mostly prose (4+ characters per token);
tool results are often JSON (2.2 per token on Claude, 3.7 on GPT-5.6). Live, a "24K-token" budget
let a 72K-character batch through on Sonnet and added 32K tokens, and a session whose factor had
drifted let a 115K one through on another model. A character budget gives the same result the same
view on every model, and its error is one-sided: a view shortened more than needed costs a
`result_read`, never data.

- **Per iteration (the batch):**
  `B = max(min(REF_CHARS × (window / REF_WINDOW)^k, ROOM × WORST_CHARS_PER_TOKEN), FLOOR_CHARS)`
  - A curve through two anchors: `REF_CHARS` 48,000 at `REF_WINDOW` 1M tokens, `SMALL_CHARS` 3,000
    at `SMALL_WINDOW` 16K, so `k = ln(48000/3000) / ln(1M/16K) ≈ 0.67`. Sub-linear, so a small
    window still gets a usable view. The window is capped at `REF_WINDOW`, and an unknown window
    gets `REF_CHARS`. About 16K characters at 200K, 12K at 128K, 4.8K at 32K.
  - `ROOM` = the compaction hard threshold × window − the request's calibrated tokens (the estimate
    compaction uses: it measures how full the request is, not how a result reads) − the assistant
    `tool_calls` message (at the worst density) − a reply reserve (the model's max output, capped
    at 8K). `WORST_CHARS_PER_TOKEN` is 2: dense JSON on the densest tokenizer.
  - `FLOOR_CHARS` (1,500) keeps even a full context's view large enough for the header, so the
    data stays reachable. What follows is the compaction seam's job.
- **Constants** in `include/llm/llm_tool_views.h`. The two anchors are the levers (cost against
  completeness, like Claude Code's `MAX_MCP_OUTPUT_TOKENS`). P4 kept them as constants (§12).
- The header reports a result's size in characters only.
- **The split** (`llm_tool_views_split`) is water-filling: results under the fair share of what's
  left pass whole, which raises the fair share of the rest, repeated until none does; the rest share
  what's left in proportion to their size. Each viewed result keeps 640 bytes of its share for its
  header.
- **Per tool,** `tool_metadata_t` gained three fields:
  - `max_result_chars` (default 0): the tool asks to be shown whole up to that size, as Claude Code
    reads `_meta["anthropic/maxResultSizeChars"]`. A result within its ask is served first in the
    split, smallest first, whole while it fits the batch budget; one that doesn't fit, or is larger
    than its ask, is split like any other. The budget is never exceeded.
  - `result_no_store`: the result is viewed but never stored. `result_read` sets it, so it can't
    hand out a handle to its own answer.
  - `result_whole`: the stage leaves the result alone (§3).
- **The read budget.** Before the batch runs, the loop sets the session's read budget to one call's
  fair share of the batch budget less the 640-byte header room, at least 2,048
  (`llm_tool_views_read_chars`), and resets it after. A `result_read` answer is built to that size,
  so it isn't viewed and stored a second time. Outside a batch (unset), the fallback is 32,000.
- **A result under its share** costs one size check.

## 7. The store: a dedicated table (schema v97)

DAWN's existing blob store couldn't serve this. Its sweep runs a per-candidate `messages LIKE` scan
(measured at ~50–75 ms per candidate, under the global database lock), and every live tool result
would be a candidate forever. Its caps, retention, LRU and dedup are table-wide, so they would reach
document originals. Tool results got their own table, with lifetime by foreign key rather than text
search. The DDL is one macro (`AUTH_DB_TOOL_RESULTS_SCHEMA_SQL`, in `include/auth/auth_db_internal.h`)
shared by the base schema and the v97 migration:

```sql
CREATE TABLE tool_results (
   id              TEXT PRIMARY KEY,          -- 'trs_' + 12 base62, validated on every read
   user_id         INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
   conversation_id INTEGER REFERENCES conversations(id) ON DELETE CASCADE,  -- NULL until bound
   session_key     TEXT,                      -- the storing session, while unbound
   tool_name       TEXT NOT NULL,
   tool_call_id    TEXT,
   content_kind    INTEGER NOT NULL,          -- json / text
   chars           INTEGER NOT NULL,
   bytes           INTEGER NOT NULL,
   created_at      INTEGER NOT NULL,
   body            BLOB NOT NULL              -- last: a metadata read never touches its pages
);
CREATE INDEX idx_tool_results_conv ON tool_results(conversation_id, created_at, bytes);
CREATE INDEX idx_tool_results_user ON tool_results(user_id, created_at, bytes);

CREATE TABLE tool_results_usage (            -- bytes held per scope, kept by triggers
   scope INTEGER NOT NULL,                    -- 0 conversation, 1 user, 2 the whole table
   key   INTEGER NOT NULL,                    -- the conversation id, the user id, or 0
   bytes INTEGER NOT NULL,
   PRIMARY KEY (scope, key)
) WITHOUT ROWID;
-- triggers: AFTER INSERT (add), AFTER DELETE (subtract; drop a zero row),
--           AFTER UPDATE OF conversation_id (move the conversation total on a bind)
```

- **`body` is the last column**, so the metadata reads (scope checks, cap bookkeeping) stop before
  the overflow pages a large body occupies.
- **`tool_results_usage`** makes each cap check one primary-key lookup instead of a sum over the
  rows. Its triggers fire on every delete, including the ones a foreign-key cascade makes, so the
  totals stay right when a conversation or account goes.
- **The indexes** are age-ordered per conversation and per user (with `bytes` carried), so eviction
  walks the oldest rows (by `created_at`) without touching bodies, and the unbound reclaim reads the
  conversation index's NULL range.

**Life:**
- Deleting a conversation or an account removes its results in the same transaction. No sweep, no
  marker coupling, no files to leak.
- Unbound rows (`conversation_id` NULL) are reclaimed after a 24-hour grace, whether or not a
  session still lists them; every store runs the reclaim. They also go first when the user's cap
  evicts.
- A voice conversation lives in memory until its idle save. The save's retries are re-armed every
  60 s and bounded by the session's life, and the idle timeout is at most 60 minutes, so 24 hours is
  more than 20× the longest wait.

**Binding** (refined in the P3 review): a result is stored with the turn's conversation id when it
has one. Otherwise it is NULL, and its handle goes into the session's minted set together with the
turn that stored it. This is the same model DAWN uses for the facts a turn saves before it has a
conversation (`session_history.c`). Under the session's `history_mutex`:
- a running turn's entries stay with it through a context reset (the user starts another chat
  mid-turn);
- when the turn learns its conversation (`session_turn_set_conversation`,
  `session_bind_created_conversation`), its own entries are bound, and the live history's
  ended-turn entries only when that history became the conversation;
- a turn that ends without one hands its entries to the live history when it wrote there, else
  drops them (no one can read them; the reclaim takes the rows);
- a reset of the live history drops the ended turns' entries; the voice save binds them as its last
  step (a failure before that deletes the conversation, and the cascade would take bound results
  with it; a bind failure fails the save, which retries).

Binding sets a row's conversation only when its user owns that conversation (checked in the
`UPDATE`), then enforces the conversation's cap.

**Who stores, and when.** `tool_result_store_put` takes the user from
`session_effective_user_id()` and reads the conversation itself, under `history_mutex` (the
calling turn's, else the live history's). A thread that isn't the running turn's (a cancelled
turn's tool still running) stores nothing. Nothing is stored unless `result_read` is offered and
enabled in that request (`llm_tools_request_offers`, decided as execution decides it, from the
surface); without it the header says to call the tool again. `result_read`'s own answers are viewed
but never stored.

**Scope (the maintainer's decision, 2026-09-30): the loaded conversation only.** `result_read`
answers when either of two things holds:
- the row is unbound and the reading turn stored it, or an ended turn stored it into the live
  history the reader is on (this covers unsaved voice before its idle save). The minted set never
  admits a bound row: a turn left running on another conversation can't make that conversation's
  results readable here;
- the row's `conversation_id` is the turn's own conversation, and its user is the turn's effective
  user.

No lineage: a continuation can't read its parent's results, and says so ("call the tool again"). A
NULL or 0 conversation never matches another. Private conversations get no special handling: their
results are stored and readable under the same conversation scope, and go with the conversation.
Searching old conversations' results is a separate future feature; the table's shape (an FTS index
is one migration) leaves room for it.

**Deletion (verified at P3):**
- `PRAGMA foreign_keys=ON` is set when the database opens.
- Conversation deletes are hard `DELETE FROM conversations` in a savepoint (the user and admin
  paths, and child jobs with their parent), the same cascade that removes messages.
- `ON DELETE CASCADE` on `conversation_id` and `user_id` removes the results in the same
  transaction.

**Caps:**
- `TOOL_RESULT_STORE_MAX_BYTES` per result (16 MB): above it, a head+tail copy is stored (about
  8 MB of each end, cut at a line and a UTF-8 boundary, with an "N bytes omitted" marker between),
  stored as the **text** kind so no JSON-mode read runs on it, and the header says "its first and
  last parts are kept."
- Per conversation (256 MB), per user (1 GB) and for the whole table (4 GB), evicting within this
  table only, oldest first:
  - at the conversation cap, that conversation's oldest;
  - at the user cap, that user's unbound rows first, then their oldest;
  - at the global cap, the oldest results of the user holding the most bytes, so one heavy user
    can't push out everyone else's, then anyone's in insertion order.
  Each eviction is logged with the cap it enforced and that cap's key (conversation or user).
- No store for an effective user ≤ 0 (guests, a session-less MQTT call). Their view carries no
  handle.
- A store failure never loses the view: it becomes a view without a handle.

**At rest:** the body is the normalized, scrubbed result with the conversation's tag masked. It is
not neutralized; whatever a read returns is neutralized when that answer is finished (§3). Before
this, `messages.content` held whole results (up to 705 KB seen), neutralized. This moved the bulk
out of `messages`, shrinking rows and reloads, keeps the tag off disk, and applies neutralizer
improvements to old results.

## 8. The retrieval tool: `result_read`

`src/tools/result_read_tool.c` (Layer 3, read-only, the `document_read` pattern) with its pure
operations in `src/tools/result_read_ops.c`. It reads through the store module
(`src/core/tool_result_store.c`, Layer 2). The name avoids the API's `tool_result` block type.

It fails closed:
- It answers only inside the model's own tool calls (`llm_tools_executing()`), in a command-context
  session whose effective user is > 0. It never falls back to the default voice user. An MQTT
  message naming a session, which can invoke tools directly, is refused.
- Every refusal (unknown id, malformed handle, another user's, another conversation's, evicted,
  guest) gets one message: "That result isn't available; call the tool again." The answer reveals
  nothing about results the caller can't read.
- A store or database failure gets a different message, "Couldn't read that result just now; try
  again.", since retrying can help there.

| action | args | returns |
|---|---|---|
| `read` | `handle`, `from`, `to` (1-based lines; clamped, ordered) | numbered lines; an over-budget slice is itself viewed |
| `path` | `handle`, `path` | the value, viewed with the path prefix |
| `grep` | `handle`, `pattern` (literal, ≤200 chars, one line, case-insensitive), `context` (0–3, text only) | JSON: `$.results[123].file: "…"` lines from a walk of the tree (each path feeds `path`); text: numbered lines, a line up to 400 bytes whole, else a 200-byte window around each match. At most 50 matches, then "... N more; narrow the pattern, or read a path" |
| `count` | `handle`, `path` | items of an array, keys of an object, or characters of a string |
| `distinct` | `handle`, `path`, `field` | values of `field` across the array's items with their counts, at most 50 values, each shown at most 200 bytes |

**Path grammar** (`llm_tool_view_path.c`, the one module views write paths with): `$`, `.name`
(letters, digits and `_`), `["key"]` (any key, as a JSON string; a cut key as the view shows it),
`[i]` (negative counts from the end), and `[i:j]` (either end optional) **only as the last
segment**. Paths are at most 512 bytes and 32 segments, indices overflow-checked. **Changed from
the plan:** the `*` wildcard was dropped; it is refused with a hint ("use distinct for a field's
values, or grep"), since `distinct` and `grep` answer what a wildcard would, in one bounded call.

- **Every answer** is shaped by the view to the read budget (§6), UTF-8 repaired, and then finished
  like any result in the batch: neutralized and tag-masked. A P2b test gates that path, since the
  stored bodies are masked but not neutralized.
- **Parse through the view's rules.** A stored JSON result is parsed as the view parses it
  (`llm_tool_view_parse`): strict JSON whose numbers json-c holds exactly. Text with NaN, Infinity,
  single quotes, comments or an integer past 64 bits is treated as text by `path`, `count` and
  `distinct` too, so no read ever shows a changed value. Only rows of the JSON kind take JSON-mode
  reads.
- **Tree cache.** A json-c tree is about 17× its text (measured: 4 MB of JSON rows, ~69 MB of
  tree), so the cache is capped by estimated tree memory, not text size:
  - 32 slots across the daemon, at most 2 per session, under a 64 MB cap of estimated tree memory
    (17 × text bytes); only results whose text is ≤ 1 MB go in these.
  - One dedicated large slot, outside that cap, holds one tree of a result between 1 MB and the
    4 MB JSON-view limit, so its turn's follow-ups read it without parsing again.
  - The view stage seeds the cache with the tree it just parsed (`llm_tool_view_ex` hands it back),
    so the model's first follow-up parses nothing.
  - A result whose tree isn't cached is re-parsed for that call (~100–300 ms per 4 MB on the Jetson,
    transient). Those large parses are serialized by a mutex, one at a time across the daemon, so
    parallel reads can't multiply the memory.
  - The cache lock covers only find, pin, insert and release (a leaf lock); a tree in use is pinned
    to its one reader. The cache is dropped at turn end and at `session_free` (a research round,
    which runs no turn, keeps its trees until the job ends).
- **Available on every surface and in background jobs,** and on the deep-research read-only
  allowlist (`url_fetch` results are large too). A job's handles belong to the job's conversation;
  a re-engagement turn in the parent conversation can't read them, which is intended.
- **Logged:** each read (`result_read: <action> <id> (<tool>, N bytes) -> M bytes`) and each
  refusal, with the session and user.

## 9. Compaction

A summarized range must not lose the handles in it. The L1 and L2 summarizer prompts gained one
instruction: keep every `[tool-result trs_...]` handle exactly as written, with a phrase on what it
held. This went in with P2b, when handles first appeared: it is a functional need, not a style
change. The mechanical L3 summary keeps a handle only if it falls within the 80-character snippets
it copies, so L3 is best-effort.

## 10. Security summary

- Retrieved and viewed text is untrusted. It is neutralized last, after every transform, and
  tag-masked. The neutralizer decodes what a model could read through before it matches
  imitations: JSON `\uXXXX`, `\UXXXXXXXX`, surrogate pairs, C short escapes (`\n`, `\/`, …) and
  HTML character references (numeric, hex and named).
- The header frame is in the defuse set and goes on after neutralizing; the omission markers aren't
  (§5: integrity only).
- Handles are scoped to the loaded conversation (or the storing turn before a voice save) and its
  user, fail closed, and give one uniform answer. Ids are validated fixed-length tokens in their
  own namespace (`trs_`), so they can't be confused with document blobs (`blb_`).
- `result_read` runs only inside the model's own tool calls; nothing outside the tool loop can read
  a stored result.
- Resource links are inert. Resource use is bounded: the JSON-mode size, the per-result,
  conversation, user and global caps, the grep and path bounds, and the tree cache caps.
- Parity notes (no worse than before):
  - Access revoked after a result was stored (an MCP server's per-call checks) doesn't reach it,
    the same as the full row that history used to hold.
  - Secrets other than the tag are as exposed as before; redaction of stored results is a separate
    item.

## 11. Phases (one release; each phase is part of the final design, and the stage never ran without the store)

1. **P1: MCP normalization and the neutralizer** (`313f0826`).
   - Work: §4, plus teaching the neutralizer's shadow to decode JSON escapes (since extended to the
     forms listed in §10). This closed a pre-existing gap: the model could decode an escaped
     `[Operator note` that the neutralizer passed through.
   - Gate: tests on the MCP shapes and the escape cases; a live code-graph call logging unescaped
     JSON at about 43% of its old size.
2. **P2a: the view module and its tests** (`0faa9689`). No wiring into the loop.
   - Tests: deep, wide and heterogeneous JSON; key caps; huge strings; big-int and float fidelity;
     JSON inside text; one huge line; UTF-8 cuts; determinism; the skeleton; the text fallback;
     path prefixes; quoted keys.
   - Gate: conversation 1554's result rendered at 24K, 8K and 1.5K, checked by eye.
3. **P3: the store and `result_read`** (`2035ee9d`, schema v97).
   - Work: the table, the store module, the minted set, binding at the voice save, the caps, the
     tool, the tree cache, the research allowlist.
   - Gate: scope tests (own conversation; another conversation refused; a continuation refused;
     foreign user; unknown id; guest; same-session unbound; other-session unbound refused); cascade
     on conversation and account delete; `EXPLAIN QUERY PLAN` uses the indexes.
4. **P2b: the loop stage** (`9205b45d`).
   - Work: budgets and water-filling; the exemptions; neutralization on the final content; the
     finish contract (§3); `max_result_chars`, `result_no_store`, `result_whole` and the MCP
     `_meta`; the conditional header; guest and store-failure views; the per-turn read budget;
     seeding the tree cache; the store's routine statements kept prepared; logging.
   - Carried from the P3 review and done: `result_read`'s answers go through the stage's
     neutralize + mask (tested); the read budget replaces P3's constant; `result_no_store` is
     metadata, never a name compare in the stage; `put` takes the user and conversation itself
     and refuses a thread that isn't the running turn's.
   - Gate: conversation 1554's batch replayed under budget with three handles; a voice turn reads
     its own handle before the save; reload replays identical bytes; the Claude path's
     `input_transformations` stay empty. Live-verified on Luna, Sonnet and a voice read before the
     idle save.
   - The budget moved from tokens to characters after the live test (§6).
5. **P4: measure** (§12). Done 2026-10-01; the header change it led to is `bfbed7c8`.

## 12. Measurement

**Logged** (bytes, not tokens):
- each view: `Tool view: <tool> N bytes -> M bytes, json|text, share S of B, <handle or "no handle">`;
- each batch large enough to matter (over a quarter of its budget): its size, budget, window, room
  and how many results were over their share;
- each `result_read` call and refusal (§8), and each eviction.

**Questions** it answers: how often views fire; how often the model reads more, and in how many
calls (many small reads mean the view is too small); how many evictions.

### P4 results (2026-10-01): controlled A/B

**Method.** One scripted harness drove the live daemon over the WebUI socket as a dedicated
non-admin user, with that user's memory wiped before every run and every run's conversation private
(so no run could learn from another; a first attempt without that isolation was discarded, as was a
before-side pass contaminated by the startup memory-recovery pass). Side A was the tree just before
P2b (`2a334147`, whole results in context), side B the P2b build, same schema and database. Three
tasks whose answers sit in the part a view leaves out, three runs each, graded against ground truth
computed from the stored full results, with an independent judge confirming the grades:
- `arch_deep`: counts from entries 160–727 of a 728-entry `file_tree` (JSON view), Sonnet 5.5 and
  GPT-5.6 Luna.
- `search_breadth`: counts across 200 search results (JSON view), same two models.
- `page_rice`: a sentence and its citation numbers from the omitted middle of a 24K-char page (text
  view on a 200K window, 16K-char budget), Sonnet 4.5.

**Accuracy (correct / partial / honest fail / wrong, of 3; A → B).**

| Task | Model | A (whole) | B (view + reads) |
|---|---|---|---|
| arch_deep | Sonnet 5.5 | 3/0/0/0 | 3/0/0/0 |
| arch_deep | Luna | 1/2/0/0 | 0/3/0/0 |
| search_breadth | Sonnet 5.5 | 3/0/0/0 | 2/0/1/0 |
| search_breadth | Luna | 0/0/0/3 | 0/0/0/3 |
| page_rice | Sonnet 4.5 | 3/0/0/0 | 3/0/0/0 |

Views did not cost accuracy. Claude models recognized every view as partial and read what they
needed unprompted (grep, distinct, path, small line reads); every B answer was correct or an honest
"couldn't verify". Luna's misses are the same on both sides: it counts a directory as under itself,
and it can't tally 200 rows even with all of them in context.

**Cost (median per run, A → B).** The batch's own growth fell everywhere (Sonnet arch 28.8K → 11.0K
tokens). But a single question answered only by the hidden part is the worst case: each read is
another request carrying the conversation. Uncached input: Sonnet arch 32.0K → 16.7K, Sonnet 4.5
page 10.5K → 9.7K, Luna roughly flat, Sonnet search 26.1K → 48.4K (8 reads to tally 200 rows).
Totals including cached input rose 2–4× (cached reads dominate). The savings this test does not
count accrue on every later turn (a view instead of the whole result) and in context headroom (the
batch that motivated this added 76K tokens and forced a compaction).

**Decisions.**
- The anchors stay (48K chars at 1M, 3K at 16K). No task here would have been helped by raising
  them: a tally over 200 rows needs the whole array or an aggregate, whatever the view's size.
- The header now points tallies at `count` / `distinct` ("tally a whole array in one call"), in the
  stored-result header and in a JSON read's own view, to turn the 8-read tally into one call
  (`bfbed7c8`).
- They stay constants; settings only if real use shows a need.
- A multi-turn A/B (the later-turn savings) was skipped by decision (the maintainer, 2026-10-01).

## 13. Filed alongside (pre-existing, found in review)

- **Scheduled briefing steps skipped `neutralize_result`** (an injection gap). *Closed:* the
  scheduler now neutralizes each briefing step's result, and a single-tool briefing's, with
  `llm_context_neutralize_owned` before it reaches the prompt (`6c933f4c`, the append-only requests
  work). Those steps still run outside the loop, so they get no views.
- **WebUI account delete didn't purge image or document-original files** (the admin socket did).
  *Closed:* both paths call one purge helper (`conv_images_purge_user`) before the account goes.
- **`url_fetch` has no narrowing parameter** (`max_chars`, a section), and it is the largest text
  producer. *Open.* It also still cuts every page at 24,000 characters inside the tool, before the
  view stage sees it, so the rest of a long page is lost with no handle. With views in place the
  tool can return the whole extracted text and let the stage store it.
- **`llm_context_neutralize` could fast-path text with no trigger characters** (it ran at ~31 MB/s
  on everything). *Open;* worth more now that a batch's results are neutralized one after another
  on the loop thread.

## 14. Decisions (the maintainer, 2026-09-30, unless dated otherwise)

1. The summarizer keeps handles: shipped with P2b (§9).
2. The pre-existing gaps in §13 weren't needed by this design (the store is a table with cascades,
   not files; scheduled steps are out of scope). They were filed as short-term follow-ups. The
   escape-decoding neutralizer gap stayed in P1: it was cheap, and security.
3. The budget's anchors stay constants (confirmed by P4, 2026-10-01).
4. Results are readable only in the conversation that stored them (§7).
5. Budgets are in characters, never tokens (2026-10-01, §6).
