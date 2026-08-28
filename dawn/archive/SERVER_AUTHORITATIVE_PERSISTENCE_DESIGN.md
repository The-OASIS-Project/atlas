# Server-Authoritative Conversation Persistence + Multi-Viewer Fan-Out

**Status:** Planned, unstarted (untracked working doc). Author: Friday. 2026-08-26.
**Reviewed** by master-plan-reviewer + architecture-reviewer + correctness-reviewer (2026-08-26); all
findings folded in below. Verdict: revise-then-ship; **Model A confirmed by all three**; direction and
layering sound; the load-bearing additions are the client adopt-map (§6a), content fidelity (§6c),
double-broadcaster reconciliation (§6b), and the accept-and-drop backstop (§9). Re-sliced into **three
phases** — a zero-risk Phase 0 that fixes the incident before any save-contract change (§7).

**SOTA note (master-plan-reviewer):** this is the canonical single-authoritative-writer + append-broadcast
+ client-side correlation-nonce pattern (Slack `client_msg_id`, Discord `nonce`, Matrix `txn_id`);
`stream_id` is the nonce, "adopt `message_id` onto the already-rendered bubble" is the standard
reconciliation. CRDTs are not applicable (no multi-master/offline edit; the DB rowid is the total order).
The *anomalous* design is the outgoing one (browser as writer-of-record).
**Motivating incident:** Kris had two WebUI clients (stock www = session 1, Aurora = another session)
open on conv 1204. Job 1207's `reinvoke_parent` re-engagement streamed to session 1 only; Aurora got
nothing — not live, and nothing persisted for it to refetch. Root cause: the re-engagement picks ONE
viewer and streams to it, and on the foreground path the **browser client-saves** the reply — so a
second viewer of the same conversation is never told.

This is the tracked "WebUI frame delivery: addressing model is the debt" item (docs/TODO.md), now with a
concrete forcing function.

---

## 1. The contract we are retiring

Today a foreground WebUI turn is persisted by the **browser**, not the server:

- The assistant reply streams live (`stream_start`/`stream_delta`/`stream_end`).
- At completion the browser's `finalizeStream()` (`www/js/ui/streaming.js:427`) calls
  `DawnHistory.saveMessage()` → sends a `save_message` WS frame → `handle_save_message`
  (`webui_message_dispatch.c`) writes the row.
- The server only persists server-side in two fallbacks: the turn finished while the client viewed a
  DIFFERENT conversation (`backgrounded`) or the client disconnected mid-turn (`client_gone`)
  — `webui_text_processing.c:388-405` for normal turns, `job_reinvoke.c:484-522` for reinvokes.

The reinvoke code names it: *"trusting the browser to save the foreground-streamed reply … If a future
client refactor breaks that save, a fired-but-unsaved re-engagement is lost"* (`job_reinvoke.c:496-501`).

**Consequences of a client-owned save:**
1. **Single-viewer:** only the one streamed client writes the row; other viewers of the same conversation
   see nothing until a manual reload (the incident).
2. **Fragile:** the save rides the client's liveness; a disconnect between `stream_end` and the save loses
   the reply.
3. **Two persistence models** to reason about (client-save foreground vs server-save background).
4. **A live TOCTOU race today** (master-plan-reviewer): the client decides to save at `stream_end` time,
   but the server decides `backgrounded` at persist time (`webui_text_processing.c:391`). A view-switch in
   that window → **both save (dup row)**, or switching back → **neither saves (lost reply)**. Model A
   eliminates this whole class.
5. **The client-save is the SOLE carrier of two fields** the server does not currently persist — the final
   answer's **reasoning** (E3 "AI thought" panel) and the **`pending_visual`** interleave. Retiring it
   naively silently regresses every reasoning panel and every `render_visual` turn on reload (§6c). This is
   the single most important thing the naive plan missed.

**Goal:** make the **server the single authoritative writer** for every conversation turn, and **fan the
persisted message out** to every session viewing that conversation. Retire the browser client-save.

---

## 2. Surface inventory — who participates in the client-saved contract?

Kris's explicit ask: check EVERY client surface, since we had a contract for clients to send saves back.
Verified in code:

| Surface | Client-saves today? | Change needed |
|---|---|---|
| **Stock www browser** | **YES** — `finalizeStream()` → `save_message` | **Retire** the client-save; render `message_appended` inline; honor stand-down signal. |
| **Aurora browser** (WWFD) | **YES** — same contract | **Retire** (WWFD owns Aurora's half; server contract identical). |
| **Tier-1 RPi satellite** (`dawn_satellite/`) | **NO** — grep shows only identity/OTA/UI-pref persistence, never `save_message`/conversation writes | **None.** Already server-authoritative. Verify no regression. |
| **Tier-2 ESP32** (`dawn_satellite_arduino/`) | **NO** — raw PCM; server does ASR/LLM/TTS | **None.** Already server-authoritative. |
| **Local mic** (`SESSION_TYPE_LOCAL`) | **NO** — server owns the turn | **None.** |
| **Messaging channels** (Telegram/Slack/Discord/SMS) | **NO** — server-persisted forever-conversations | **None.** |

**Key insight:** only the two WebUI browsers client-save. **Satellites/local/messaging already run the
target model** — they are the working proof that server-authoritative persistence is sound. This is a
much smaller surface than "all clients."

**Note the WebUI voice paths explicitly** (master §5): browser **push-to-talk** (`webui_server.c:3472`)
and **always-on** (`webui_always_on.c:733/790`) both funnel into the same `webui_text_processing.c` worker,
so Phase 2 covers them *by construction* — but they are distinct user-visible surfaces whose streamed
replies are client-saved today, so the test matrix must exercise a **voice** turn with two viewers, not
just a typed one.

---

## 3. Machinery that already exists (reuse, don't invent)

- **`server_saved` flag, end to end.** `send_transcript_impl(…, server_saved, …)`
  (`webui_send.c:398-413`) stamps `server_saved:true` on a frame; the browser skips its client-save when
  set (`dawn.js:213-216`: *"Skip server_saved messages (server persisted to DB, client save would
  dupe)"*). Already used for **user-message echoes** (the server persists the user msg and echoes it
  `server_saved=true` so the browser doesn't double-save — `webui_audio.c:1250`,
  `text_input_dispatch.c:96`). It does **not yet** cover the streamed assistant reply — that's the gap.
- **`message_appended` frame** — `webui_broadcast_message_appended(user, conv, msg_id, role, text)`
  (`webui_broadcasts.c:716`) already carries the persisted body + `message_id` and fans to the user's
  sessions. **Currently a browser no-op** (`dawn.js`: *"the browser already has it from the stream, so
  this is a no-op"*) — it exists for event-only consumers (the Phase-5 TUI). Repurposing it for
  cross-viewer delivery is additive.
- **Fan-out iteration pattern** — the per-conversation broadcast loop (e.g. `context_citations`,
  `webui_broadcasts.c:1187`) already filters `SESSION_TYPE_WEBUI && auth_user_id && active_conv == conv`.
- **`webui_find_reinvoke_viewer`** — picks a live WEBUI viewer of the parent conv (structurally WEBUI-only).

---

## 4. The ordering constraint (why `stream_end` can't carry `message_id`)

`stream_end` is emitted from **inside** the LLM orchestration (`session_manager_llm.c`), i.e. during
`core_text_input_dispatch`. The worker persists the reply **after** dispatch returns
(`job_reinvoke.c:486` / `webui_text_processing.c:394`). So **at `stream_end` time the persisted
`message_id` does not exist yet** — and the server does not yet know the persist will succeed. Any dedup
key or "server saved it" *confirmation* keyed on `stream_end` is therefore impossible without reordering
persist ahead of the in-dispatch stream_end (a deep change to the shared stream path — out of scope).

Two design options fall out of this constraint (§5). This is the **central decision for the reviewers.**

---

## 5. Two models for the "stand down from client-save" signal

Because the confirmation can't ride `stream_end`, the origin browser (which streamed the reply) needs
another way to know "the server owns this save, don't client-save it."

### Model A — Intent flag on `stream_end` (server promises to persist)

- For an **always-persist** turn, the server stamps `server_saved:true` (an *intent*, not a confirmation)
  onto `stream_end`. The browser sees it and **does not** client-save.
- The server then persists post-dispatch and fans out `message_appended`. If the post-dispatch persist
  **fails**, the server owns the recovery (retry / error notice) — the browser will not fall back.
- **Pros:** simplest client (one boolean, no timers); true single-writer; matches the existing
  `server_saved` semantics already used for user echoes.
- **Cons:** a server-persist failure after a `server_saved:true` promise loses the reply from the DB
  (still on screen live). Mitigation: local SQLite write is highly reliable; wrap with a retry + a
  server-side error frame; the risk is strictly smaller than today's client-disconnect-loses-save.

### Model B — Reconcile via `message_appended` (browser defers, times out to fallback)

- The browser holds the finalized stream in a **pending-save** state instead of saving immediately.
- If a `message_appended` whose `stream_id` matches the stream arrives → the server saved it → **discard
  the pending client-save**, adopt its `message_id`.
- If a timeout elapses with no such frame (old server) → **commit the client-save** (fallback).
- **Pros:** robust to server-persist failure; degrades cleanly against an old server with no version gate.
- **Cons:** client complexity (pending state + timer); a dup-row window if the timeout races a slow
  broadcast; harder to reason about.

### Model C — final-persist hook inside `llm_call_finalize` (the principled end-state, DEFERRED)

A `session_set_final_persist_hook`-style seam firing inside `llm_call_finalize` *before*
`webui_send_stream_end` — same in-dispatch, same-thread, WebUI-out-of-core pattern as
`session_set_tool_persist_hook`. Then `stream_end` carries a real `message_id` as a **confirmation**, not a
promise, retiring the intent/confirmation ambiguity entirely. **Cost (why deferred):** the cancel check
(`session_manager_llm.c:285`) and supersede check (`webui_text_processing.c:337`) run *after* finalize, so
persisting-in-finalize changes their semantics; and the reasoning/visual assembly (§6c) would have to move
in-dispatch too. Not needed for correctness — `message_appended` closes the confirmation loop within
milliseconds. **Record as the future shape** if a client ever needs `message_id` synchronously (fork/edit
anchoring); do not build now.

### Decision: **Model A** (all three reviewers concur)

Model A is right, and not narrowly. Model B is a **dual-writer anti-pattern**: a timeout-armed client
fallback makes the write-of-record ambiguous, and `queue_response` **drops the oldest frame under
backpressure** (documented) — so a dropped/slow `message_appended` past the timeout yields the client
saving anyway → **two rows, two `message_id`s, no DB dedup to collapse them**, with an *un-tunable* window
(broadcast latency is unbounded, infinite on a dropped frame). Model A's only exposure is a *local SQLite
insert failing after the promise* — disk-full/corruption territory, strictly smaller than today's routine
disconnect + TOCTOU (§1) exposures, and the reply is still on screen with a server error frame.

**Two framing corrections the reviewers required (folded in):**
- **Do not overload `server_saved`.** Today it is a post-write *confirmation* (user echoes). Model A's
  pre-write *intent* is a different meaning — use a **distinct field, `will_persist:true`, on the final
  `stream_end`** (never on `reason=tool_iteration`). One field, one meaning each.
- **State the failure policy honestly:** best-effort persist with a **bounded retry (2–3) + a loud server
  error frame + accepted DB-row loss on near-catastrophic local-write failure** — NOT "retry closes the
  gap." The reply stays on screen; the user is told. Do not over-engineer a journal.

> `message_appended` gains a **`stream_id`** field regardless of model — it is how the origin correlates
> (§6a). Forward-compat note (architecture): `stream_id`/`conversation_id` on `message_appended` are
> **envelope/routing metadata**; write the correlation to be *liftable* to the envelope when the tracked
> addressing-model refactor lands, not baked deeper into payload parsing.

---

## 6. The mechanism (Model A)

Server, per turn:
1. **Always persist** the assistant reply server-side — one `conv_db_add_message_with_tools` call, the
   sole writer (remove the "foreground → client saves" branch). **Must carry full fidelity — §6c.**
2. Stamp **`will_persist:true` on the FINAL `stream_end`** (never `reason=tool_iteration`), only when the
   fallback-gate condition holds (`turn_conv > 0 && turn_user_id > 0`; conv-less transient turns get no
   promise and no persist). The streamed viewer stands down on it. **Stamping mechanism (2026-08-27):** a
   per-turn `session->will_persist_turn` bool, cleared at turn start, set true by the persist-owning caller
   *before* dispatch (Phase 1: the reinvoke worker; Phase 2: `webui_text_processing`).
   `webui_send_stream_end` reads it and adds the field only when `reason != "tool_iteration"` — signature
   stays stable, every non-opted surface's `stream_end` is unchanged. (Distinct field from `server_saved`
   per §5: intent, not confirmation.)
3. **Fan out `message_appended`** `{conversation_id, message_id, role, text, reasoning, stream_id}` —
   **user-scoped** (all the user's `SESSION_TYPE_WEBUI` sessions, `browsers_only=true`; **do NOT add a
   server-side active-conv filter** — the active/inline vs non-active/badge branch is a CLIENT decision).

### 6a. Client adopt-map — the load-bearing reconciliation (correctness CRITICAL)

`finalizeStream()` (`streaming.js:527`) nulls `streamingState.streamId` and the bubble element, so by the
time the post-dispatch `message_appended` arrives the origin has **nothing to correlate against** — a naive
"match streamId" fails and the origin **double-renders every reply**. Required client change (www + Aurora):
- On finalize, retain a small **`(conversation_id, stream_id) → bubble`** map (a tiny ring of the last few
  finalized turns) that SURVIVES `finalizeStream`.
- Correlation key is **`(conversation_id, stream_id)`**, not `stream_id` alone (per-session counter). Use
  the turn's **LAST** stream_id — `stream_id` is bumped per **tool-iteration**, and the final answer
  bubble carries the last iteration's id.
- On `message_appended`: if `conversation_id == active` **and** `(conv, stream_id)` matches a finalized
  bubble → **adopt** `message_id` onto that existing bubble (idempotent, do NOT render anew) and stand down
  from saving. Else if `conversation_id == active` → **render inline** (non-origin viewer) . Else →
  existing unread/jobs-card behavior. **DOM dedup rule:** a bubble already carrying a `message_id` is never
  re-rendered; the origin bubble (no id yet) is matched by `(conv, stream_id)` and adopts the id once.
- The **resumed-viewer sub-case:** a viewer that switched in mid-stream got `stream_resume`
  (`webui_history.c:1164`) and became a live streamer with its own `streamingState.streamId` → it hits the
  SAME correlation path and needs the same adopt-map (it is not a pure "non-origin inline" viewer).

### 6b. Single active-view channel — reconcile the existing double-broadcaster (correctness HIGH)

The persist path **already** fires `conversation_messages_appended`, whose active-view handler does a
**full `requestLoadConversation` refetch** (`history.js:1007`). Adding inline `message_appended` on top →
the active viewer inline-renders AND full-refetches → scroll-jump/double-handling.
- Make **`message_appended` the SOLE active-view delivery channel.** **Chosen option (master R3):
  RETIRE `conversation_messages_appended` on the `message_appended` paths** — the §6a else-branch already
  does non-active unread marking, so nothing is lost, and messaging (which never emits `message_appended`)
  keeps its refetch signal unchanged. (Do NOT use the per-viewer-suppress variant — it would require the
  server-side active-conv filter §6 step 3 forbids.) One row → one active-view signal.
- **Phase-0 exception:** in Phase 0 the client-save still writes and the server does not yet suppress
  `conversation_messages_appended`; Phase 0's incident fix rides the R1-stamped `message_appended` inline
  render. The retire-the-refetch step lands with each save flip (Phase 1 reinvoke, Phase 2 normal).

### 6c. Content fidelity — the server persist must equal what the client-save wrote (architecture HIGH, master G1/G2)

The client-save assembles two things the server currently drops (every server-side persist passes `NULL`
reasoning and bare `final_response`):
- **Reasoning (E3):** the tool loop builds `reasoning_json` per iteration (`llm_tool_loop.c:838`) but the
  FINAL iteration's reasoning is never exported past dispatch. **Fix:** stash the final reasoning on the
  session inside the loop, read it post-dispatch in the worker (same pattern as `stream_conversation_id`),
  pass it to the persist call, and include it in the `message_appended` payload so a non-origin viewer
  renders the panel too. This also fixes today's latent loss on backgrounded/client-gone rows.
- **`pending_visual`:** `handle_save_message` clears `session->pending_visual` because the client already
  baked the `<dawn-visual>` interleave into the body (`webui_history.c:1784`). **Fix:** the server appends
  (and clears) `pending_visual` onto the persisted assistant row. Position fidelity drops from inline to
  **appended** — an accepted, explicit tradeoff.

### 6d. Reinvoke picker prefers a TTS viewer (Kris's TTS invariant)

`webui_find_reinvoke_viewer` **prefers a `tts_enabled` viewer** (scan tts-on first, fall back to any), so
the TTS user is the one that streams + hears audio. `message_appended` is text-only, so a text viewer
never gets audio and at most one client ever hears the reply. (The ≤1-audio + only-enabled invariant
already holds; this just makes the audio land on the client that wants it.)

### 6e. stream_id sourcing + emitter-site stamp discipline (master G7 / R2)

`session->current_stream_id` (atomic monotonic uint) is readable post-dispatch; the turn queue guarantees
no next turn has advanced it before the worker reads it. **Stamp `stream_id=0` ("no stream") when the turn
did not stream** so a stale id can't false-correlate. Every `message_appended` emitter must follow this —
enumerated (master R2), because a missed real-id stamp double-renders for a *watching* viewer:

| Emitter | stream_id to stamp |
|---|---|
| WebUI text worker (`webui_text_processing.c`) | real `current_stream_id` |
| Live reinvoke (`job_reinvoke.c` live branch) | real `current_stream_id` |
| **`job_worker.c:303`** (a user *watching* a job streams its answer) | **real `current_stream_id`** — 0 would double-render for the watcher |
| `handle_save_message` (Phase-0 transitional, via R1) | the client-sent `streamId` |
| Detached reinvoke / research / non-streamed transcript-fallback turn | `0` |
| Messaging | n/a — emits `conversation_messages_appended` only, never `message_appended` (verified) |

---

## 7. Staged rollout (three phases — re-sliced per master-plan-reviewer)

**Key insight:** `message_appended` **already fires on every persist path today** (including after a
client-save — `webui_history.c:1806`); the browsers just no-op it. So the *delivery/render* half can be
built and shipped **before any save-contract change** — and that alone fixes the motivating incident. This
decouples "fix the incident" (additive, zero dup risk) from "retire the client-save" (the risky part).

- **Phase 0 — enablers + client render (ADDITIVE, zero contract change). Fixes the incident.**
  - Server: add `stream_id` (+ `reasoning`) to `webui_broadcast_message_appended` /
    `conv_event_notify_message_appended` (stamp per §6e); export the final-iteration reasoning
    stash (§6c-G1); add the `pending_visual` append helper (§6c-G2). No save-path behavior change.
  - **R1 (both reviewers, MUST-FIX — else Phase 0 double-renders the origin on every foreground turn):**
    in Phase 0 the *client-save* is still the writer, so a foreground turn's `message_appended` is emitted
    by `handle_save_message` — which has **no `stream_id` in scope and would stamp 0**, so the origin's
    real-id bubble fails the §6a match and re-renders its own reply. Fix: the **client sends its `streamId`
    in the `save_message` payload** (available at `onSaveMessage`/`streaming.js:517`, *before* the `:527`
    reset), and `handle_save_message` **echoes it** into the `message_appended` it emits. Now the origin
    matches `(conv, streamId)` → adopts/skips; a true non-origin has no such entry → renders inline. A
    **transitional field**, deleted in Phase 2 with the rest of the client-save.
  - Clients (www + Aurora, lockstep): implement the `message_appended` handler — the §6a adopt-map
    (correlate `(conv, stream_id)`, adopt `message_id`, DOM-idempotent dedup), non-origin inline render,
    and the §6b `conversation_messages_appended` reconciliation; **stamp `data-message-id` on every bubble
    the reload/`addEntry` path builds** (§6a — prerequisite for appended-vs-reload dedup).
  - **Result:** a second viewer (Aurora on conv 1204) renders the pushed body regardless of who saved —
    the incident is fixed — with the save contract untouched, independently testable with two browsers
    today. *(agent ~half-day · 1-2 ckpt)*
- **Phase 1 — reinvoke path flips to server-authoritative.**
  **REPLACE** (not add alongside — else the server self-dups) the conditional
  `if(client_gone||backgrounded) persist` with an unconditional persist on both live-reinvoke branches
  (single writer); `will_persist` on the final `stream_end`; `mark_fired` gated purely on persist success
  (delete the "trusting the browser" LOAD-BEARING comment, `job_reinvoke.c:507`); picker-prefers-TTS (§6d);
  `browsers_only=true` (§8). **The promise-break handling (G4, §9) MUST land here, not Phase 2** — the
  moment `will_persist` is stamped on the reinvoke, a cancel/supersede during its finalize breaks the
  promise live (correctness R2). Live-verify the §10 matrix. **Reasoning export (§6c-G1) DEFERRED to
  Phase 2** (2026-08-27): reinvoke persist keeps `reasoning=NULL` as today's backgrounded/client-gone
  persist already does → no regression (Phase 1 doesn't newly drop a panel that survived before), and the
  reviewers scoped the fidelity gate to Phase 2. Phase 1 stays: persist-flip + `will_persist` +
  cancel-stash (G4) + picker-prefers-TTS + `browsers_only` + retire-the-refetch. *(agent ~half-day · 1-2 ckpt)*
- **Phase 2 — normal foreground turns flip (the hot path).**
  Always-persist in `webui_text_processing.c` **with reasoning + visuals attached** (§6c); `will_persist`
  on the final `stream_end` **and** stamp `server_saved` on the **non-streamed transcript fallback**
  (`session_manager_llm.c:258` — G3); resolve the **cancel-at-buzzer + supersede** promise-breaks (G4,
  §9); bounded persist retry + error frame; **delete** the client-save call in both browsers; flip
  `handle_save_message`'s assistant branch to **accept-and-drop** (§9) — keep the handler forever as the
  stale-client sink. *(agent ~1d · 2 ckpt)*

  **Phase-2 decision — cancelled-partial handling (surfaced 2026-08-27 live test, dev-agreed keep-it).**
  A mid-stream Stop on a normal turn persists the partial (today via the browser client-save) and it is a
  legitimate part of the transcript — **keep it, do not strip.** But two things need deciding as one
  coherent change when Phase 2 takes over the cancel save: (1) **it is currently fed to the model as if
  complete** — a partial can end mid-structure (observed: a reply ending on `### My read for today's` with
  nothing after), so the next turn tries to *finish the thought* instead of re-answering (observed as an
  unexpectedly broad "briefing rerun" on a "try again"). Persist + feed it with an **interrupted/truncation
  marker** so the model re-answers rather than seamlessly resuming. (2) **memory↔DB inconsistency:** the
  cancel path returns before `session_add_message` (`session_manager_llm.c` cancel branch), so the partial
  reaches the DB (client-save) but NOT the live in-memory session history → whether the next turn *sees* it
  is non-deterministic (warm session vs reload-from-DB). If we keep the partial, it must land in memory and
  DB the same way. Both are orthogonal to Phase 1 (which only touched the reinvoke path + the shared cancel
  branch, byte-identical for normal turns); Phase 2 is the natural home because it moves the normal-turn
  cancel save server-side. Not a Phase-1 fix.

Server + both browsers land in lockstep within each phase; WWFD lands Aurora's half each time.

---

## 8. Satellites / local — verification, not change

- The stream + TTS path is structurally WEBUI-only (`webui_find_reinvoke_viewer` filters
  `SESSION_TYPE_WEBUI`) — a satellite/local can never be the streamed viewer.
- `message_appended` currently goes out `browsers_only=false`, so it technically reaches satellites, which
  ignore the unknown frame (harmless). **Tidy-up:** send it `browsers_only=true` so it is strictly WEBUI —
  matches the frame-delivery capability matrix (a satellite renders no transcript).
- Reinvokes with a DAP2/local/messaging **parent** are already downgraded to `notify` upstream
  (`job_tool.c:103`, `job_dispatch.c:61-69`) — they never enter the live-stream path.
- **Test obligation:** confirm a Tier-1 satellite turn and a local-mic turn still persist + render exactly
  as today (they already server-persist; this change must not perturb them).

---

## 9. Risks & rollback

| Risk | Severity | Mitigation |
|---|---|---|
| **Rollout-skew dup row — PERMANENT, not transient** (correctness HIGH): `finalizeStream` saves *unconditionally* (doesn't consult a flag), so an old browser + new always-persist server → server row + client row, **two `message_id`s**, no DB dedup to collapse | High | **Structural backstop (master G6, chosen over a DB content-dedup):** after Phase 2, flip `handle_save_message`'s **assistant** branch to **accept-and-drop** — server-authoritative means the server *refuses* client writes (user-role + job-conv saves already accept-and-drop, `webui_history.c:1733/1750`). Permanent protection against any stale/lagging client (incl. Aurora behind a deploy). During Phase 1 the residual is one narrow case (stale client foreground-viewing a reinvoke) on a 2-client personal deployment — acceptable without a DB guard. |
| **Model A persist-failure** re-streams a visible dup (reinvoke retry) or silently loses (normal turn) | Medium | Reinvoke: the monitor retry is idempotent **once the §6a adopt-map recognizes a re-delivered reply**. Normal turn: bounded retry (2–3) + loud error frame; on hard local-write failure the reply stays on screen, DB-row loss accepted (disk-full territory, strictly < today's disconnect loss). |
| **Promise-break after `will_persist`** (master G4): cancel-at-buzzer frees a completed response *after* `stream_end` (`session_manager_llm.c:285`); post-dispatch supersede frees one (`webui_text_processing.c:337`) | Medium | **Persist-even-when-cancelled/superseded-if-complete** (lands in **Phase 1**). **Placement (correctness R3): the two are asymmetric.** *Supersede*: the response is in the WebUI worker's hand → fix locally. *Cancel*: the response is freed **inside dispatch** which returns NULL (`session_manager_llm.c:285-288`), so the WebUI worker never sees it → the persist-if-complete must happen at the **shared Layer-2 core cancel site**. **RESOLVED MECHANISM (2026-08-27) — a session stash, NOT "return the response on cancel":** `llm_call_finalize`'s cancel branch, when the response is complete, **finalizes it and stashes it on `session->cancelled_final_response` instead of `free`, and still returns NULL** — the return contract is unchanged, so voice/local/DAP2/messaging/research (which read only the return value and legitimately abort on cancel — e.g. wake-word barge-in) are byte-for-byte unaffected. Only the **persist-owning** caller opts in: the reinvoke worker, on `response==NULL`, reads-and-takes the stash and persists it (→ fan-out + `mark_fired`) exactly as it would a normal completion. Cleared at turn start (master-R5 lifetime rule) so a stale stash never persists onto the wrong row. This is the same "core stashes a per-turn artifact, the post-dispatch worker reads it" shape as the reasoning stash (§6c-G1) and `stream_conversation_id` — not a bolt-on; the rejected "return the completed response on cancel" option would silently flip 6 surfaces from abort-on-cancel to deliver-on-cancel. **Phase-1 scope = the CANCEL half only:** `reinvoke_turn_entry` checks `cancel_requested`, never `REQUEST_SUPERSEDED` (the turn queue serializes it, no generation bump), so the *supersede* half is genuinely Phase 2's (normal-turn) concern. Disposition (persist the completed row — it belongs to the conversation, matches today, where the browser client-saves it) is agreed. |
| **Hot-path blast radius** (Phase 2) | High | Phase 0 proves render/dedup with no contract risk; Phase 1 proves the save flip on the low-frequency path; Phase 2 rides both — but is **NOT** "a mechanical apply" (see below). |
| **Content-fidelity regression** (architecture HIGH): reinvoke replies are plain text, so Phase 1 does **not** exercise reasoning/visual/tool-turn persist | High | §6c makes the server persist reasoning + visuals; **Phase 2 gets its own fidelity gate** — a turn *with reasoning and a visual* must persist + reload byte-identical to the client-save. Do not describe Phase 2 as mechanical. |
| **Double-render from the existing `conversation_messages_appended`** refetch racing inline `message_appended` | Medium | §6b: `message_appended` is the sole active-view channel; suppress the refetch signal for active viewers on always-persist paths. |
| **Efficiency** (correctness deferred): `message_appended` fans the full assistant body to *all* the user's sessions regardless of active conv | Low | Fine at single-user scale; the `any_session_matches` pre-flight already short-circuits an empty pool. Revisit only if multi-user bandwidth shows up. |

**Rollback:** revert the phase; the client-saved contract is restored. Keep the client-save code
**dead-but-present** through Phase 1, delete in Phase 2 — but keep the `handle_save_message` **handler**
forever as the accept-and-drop stale-client sink. Update the now-false LOAD-BEARING comments
(`job_reinvoke.c:507-512`, `webui_text_processing.c:384`) in the same phase that flips each behavior
(correctness LOW).

---

## 10. Test plan

- **Two browsers, same conv, both text:** a turn (typed or reinvoke) appears in BOTH, exactly one DB row,
  survives reload in both.
- **Two browsers, one text + one TTS:** the TTS one streams + speaks; the text one gets text inline; audio
  reaches ONLY the TTS one; one DB row.
- **Reinvoke into a backgrounded/closed viewer:** server persists + fan-out; on reopen the reply is there.
- **Satellite + browser on the "same" user:** satellite turn persists + renders as today; no dup; browser
  unaffected.
- **Disconnect mid-turn:** server owns the save (Model A) — reply persists without the client.
- **Dup-row assertion:** every scenario asserts exactly one `messages` row per assistant turn.

---

## 11. Resolved questions (reviewer rulings)

| # | Question | Ruling |
|---|---|---|
| Q1 | Model A vs B vs C | **Model A** — all three concur. B is a dual-writer anti-pattern with an un-tunable dup window under frame-drop. C (final-persist hook) is the principled end-state, **deferred** (§5). |
| Q2 | Defensive DB dedup? | **No DB content-dedup.** Instead flip `handle_save_message`'s assistant branch to **accept-and-drop** post-Phase-2 (structural, permanent, precedented) — §9. |
| Q3 | `stream_id` sourcing | **Cleanly recoverable** — atomic `session->current_stream_id` read post-dispatch under turn-queue protection; key = `(conversation_id, stream_id)`; stamp 0 for non-streamed; **client must retain the finalized id** (§6a/§6e). |
| Q4 | Delete vs dead client-save | Dead-but-present through Phase 1, **delete in Phase 2**; keep the `handle_save_message` handler forever as the accept-and-drop sink. |
| Q5 | Turns NOT to promise | Conv-less/user-less turns + empty responses (existing guard). **Plus three misses now covered:** the non-streamed **transcript fallback** (G3, stamp `server_saved` there), and **cancel-at-buzzer + supersede** (G4 — persist-if-complete). Private convs persist normally (`is_private` gates memory extraction, not message rows). |

## 12. Test plan additions (from review)

Beyond §10: a **voice turn (PTT + always-on) with two viewers** (master §5 — both funnel through the
Phase-2 worker but are distinct surfaces); a **turn with reasoning + a `render_visual`** persisted
server-side then reloaded, asserted byte-identical to the client-saved row (architecture HIGH fidelity
gate) — **run this with a LOCAL model** so the fidelity gate catches an inline-`<thinking>`-strip delta the
client does today but server canonicalization may not (master R4); a **cancel-at-buzzer** and a
**supersede** turn, asserting the completed reply persists once; a **stale-client (old bundle) + new
server** run asserting the accept-and-drop backstop yields exactly one row.

---

## 12b. Live-test findings (Phase 0, 2026-08-26) + correlation-key decision

Two-browser test (www + Aurora on conv 1204) confirmed the core: assistant replies + context
fan out to both, one DB row, origin adopts. Four gaps surfaced; dispositions:

1. **User message not mirrored** (non-origin sees a reply with no question) — **fix in Phase 0** (§12c).
2. **Context** — working, mirrors to both. ✓
3. **Tool-use only on the origin live** (present on reload — persisted) — **→ SHIPPED, Phase-3 §12h**
   (ephemeral `tool_step` fan; reload already covered correctness, so this was a live-coherence nicety).
4. **TTS doesn't reach a non-origin TTS viewer when a text client originates** — **→ SHIPPED, Phase-4 §12i**
   (multi-target TTS, synth-once-fan-per-format). NB: the §6d "prefer a TTS viewer" picker is a **reinvoke**
   mechanism (the server picks which viewer to stream to); a **normal** turn's origin is fixed (whoever
   typed), so there is no picker — the shipped answer fans synthesized audio to every TTS-enabled WEBUI
   viewer of the conversation.

### Correlation-key decision — `message_id`, NOT a client nonce

The question "is a client-generated nonce the right correlation key?" resolved to **no**. The true key is
the server's DB `message_id`; the only reason any *bridge* is needed is the window where a client rendered
a bubble before it knew the `message_id`. That window is **asymmetric**:
- **User messages: no bridge needed.** They are not streamed, and the server ALREADY persists every user
  turn before the origin renders it (from a server transcript echo, not optimistically —
  `text_input_dispatch.c:99-109`). So the `message_id` exists at render time → the origin's bubble carries
  it → fan-out dedups on `message_id` directly. A client nonce would invent a second namespace for a case
  the DB id already covers — a bolt-on.
- **Assistant replies: `stream_id` bridge, and only here.** The Q4 ordering (streamed before persist)
  means the bubble exists before the id does, so a transitional transport-scoped key is unavoidable —
  scoped to exactly this case, and retired entirely if Model C (persist-before-stream_end) ever lands.

**Decision:** unify correlation on `message_id`; keep `stream_id` solely as the streamed-assistant bridge;
**introduce no client nonce.** This composes forward: the user message is already server-persisted (Phase 2
alignment), `message_id` is the natural field to lift into the envelope refactor, and there is no nonce
lifecycle to manage.

### 12c. User-message fan-out (Phase 0 addition)

Server: after persisting the user turn (already done), also **fan out `message_appended(role=user, msg_id,
text)`** to all WEBUI viewers of the conversation, and **carry `msg_id` in the origin's user transcript
echo** (`webui_send_transcript_ex` gains a message_id) so the origin stamps `data-message-id` on its user
bubble. Client: the transcript handler stamps the user bubble's `data-message-id` from the echo; the
existing `handleMessageAppended` then dedups the origin's copy on `message_id` and renders the non-origin's
inline — no handler change, because it is already role-agnostic and keyed on `message_id`.

## 12d. Phase-0 implementation review (big-three + correctness, 2026-08-26)

Ran architecture / security / efficiency / correctness on the full Phase-0 diff (server + www).
Verdict: sound, additive, merge-with-fixes. **0 Critical / 0 High-that-survived** — the fixes below landed:

- **Resumed-viewer double-render (arch HIGH-1, real):** `handleStreamResume` set `streamId` but not
  `conversationId`, so a viewer that switched into a conversation mid-stream finalized with a null conv →
  the adopt-map entry was dropped → it re-rendered the reply it just streamed. Fixed: set
  `conversationId` in `handleStreamResume`; null it in `resetStreamingStateSilently`. (Correctness had
  called this case working — it verified the `stream_id`/`sr_sid` half but missed the conv half; the adopt
  needs both.)
- **Stamp-after-await race (arch M1 + correctness MEDIUM, real):** the `data-message-id` stamp targeted
  `transcript.lastElementChild` *after* an `await` — for an image-bearing message the render parks on the
  image fetch, so the fan-out dedup ran first (double-render), and a concurrent render could cross-stamp
  the wrong node. Fixed: `addNormalEntry` stamps the bubble it creates **synchronously at creation**
  (before the image await); removed the `lastElementChild` guess. Subsumes the reasoning-only early-return
  drop.
- **Cross-session stale-ring false-adopt (correctness LOW):** an empty-reply turn recorded a null-id ring
  entry that a later same-numbered stream from another session could false-adopt. Fixed: gate
  `recordFinalized` on non-empty content.
- **Doxygen drift (correctness LOW):** folded the `message_id` note into the hook's `@param` block, dropped
  the stale `persisted_to_db` line.

**Deferred/tracked (not Phase-0 blockers):** `browsers_only=true` flip (→ Phase 1, §8); extract the two
user-fanout sites into one helper; the reinvoke `conversation_messages_appended`+`message_appended`
double-signal (→ Phase 1 §6b retire-the-refetch); full-body fan-out to idle-on-other-conv tabs (efficiency
M1 — the conv-scoped body-vs-signal split, the addressing refactor); `client_data`-read-without-sync (the
new fan-out/TTS-begin reads join tracked background-jobs item (d)). **Process (arch M2):** keep the
reinvoke-TTS seam a SEPARATE commit from Phase-0 so a bisect can separate a fan-out regression from a
TTS-frame regression.

## 12e. Phase-1 implementation + review (2026-08-27)

Shipped the reinvoke flip: `will_persist_turn` intent flag (session field, armed by the reinvoke
worker around dispatch, stamped on the final `stream_end` via `webui_send_stream_end`); the cancel-at-
buzzer stash (`session->cancelled_final_response`, §9/G4 cancel half); unconditional server-persist in
both live + detached reinvoke branches; `mark_fired` on persist success; §6d picker-prefers-TTS;
`browsers_only=true` on `message_appended`; retired the `conversation_messages_appended` refetch on both
reinvoke paths (§6b). www client: stand down from the client-save on a `will_persist` final stream_end,
still record the adopt-map entry. Two now-dead weak seams retired (`job_reinvoke_notify_conv_appended`,
`webui_session_active_conversation`).

**Reviewed** architecture / security / efficiency / correctness on the full diff. **0 Critical / 0 High.**
Efficiency: neutral on the hot path (will_persist_turn short-circuits the strcmp; no new hot-path alloc;
net-negative work on the reinvoke path — one fewer broadcast + a client refetch eliminated). Architecture:
clean, net debt removed. Security: safe on memory-safety + authz axes. Fixes applied:

- **MEDIUM (correctness) — non-streaming reinvoke double-row:** a reinvoke on a non-streaming provider
  delivers via the `webui_send_transcript` fallback (not `stream_end`), which carried `server_saved=false`
  → the client client-saved while the server also persisted. **Fix:** the fallback now stamps
  `server_saved = will_persist_turn` (§13 G3's sanctioned "server_saved-as-promise" acceptable-fallback;
  the client's existing `!server_saved` gate honors it, no client change).
- **MEDIUM (correctness) — `will_persist` over-promised on error/empty:** the stamp gate was
  `reason != tool_iteration`, so an error/transient_error final `stream_end` still promised persistence →
  the browser dropped its streamed partial while the server saved nothing. **Fix:** the gate is now
  `will_persist_turn && reason == "complete" && stream_had_content` — promise only when a row will actually
  be written. Preserves cancel-at-buzzer (completed stream → "complete" + content → stamped → cancel frees
  → stash → persist); a non-streamed turn promises via the transcript fallback's `server_saved` instead.
- **arch LOW — structural turn-start reset, applied at the correct site:** the reviewer suggested resetting
  `will_persist_turn` in `llm_call_prepare`, but prepare runs INSIDE dispatch, AFTER the reinvoke arms the
  flag → that would clobber the arm and silently reintroduce the dup. Applied instead in
  `session_begin_turn_flags` (the turn-start hook every WEBUI turn path runs BEFORE arming), so a leaked
  promise can never reach the next turn without clobbering the current arm.
- **security LOW — degraded-OOM stash:** the finalize-alloc-fail branch now strips the tag grammar in place
  before stashing, so no raw `<cited>`/`<command>` markers reach the DB/browser under OOM.

**Re-verified** (correctness, post-fix): all four CLOSED, no net-new defects. **One documented LOW
residual:** `stream_end`'s `will_persist` is computed on the RAW (pre-strip) response, so a reinvoke reply
that streams real chunks but strips to *entirely* empty markup (a body that is wholly
`<cited>…</cited>`/`<command>…</command>`/whitespace) still stamps the promise, the browser stands down,
and the worker then persists nothing (`reply[0]=='\0'` → monitor retry). Left as-is: the trigger is
near-impossible for a prose take, the only content "lost" is markup that would have been stripped anyway,
and the airtight fix = stamp on the FINALIZED emptiness, which needs the persist-before-`stream_end`
reorder the design deferred to Model C (§4/§5). Not worth it on this branch.

**Deferred (unchanged from below / by design):** the version-skew dup (stale client that ignores
`will_persist` + always-persist server → two rows) is covered by the Phase-2 `handle_save_message`
accept-and-drop backstop (§9); on this 2-client personal deployment the residual is one narrow case and
client+server ship lockstep. The reasoning-export (§6c-G1) and supersede half of G4 remain Phase 2.

## 12f. Phase-2 implementation + review (2026-08-27)

Implemented the hot-path flip in four bisectable slices on one branch (plan:
`~/.claude/plans/mossy-plotting-hinton.md`, master-plan + arch + correctness reviewed pre-build):

- **2a** — the shared **`webui_persist_final_answer`** weak seam (proto in `conv_event.h`, strong def in
  `webui_broadcasts.c`): splice accumulated `pending_visual` + `final_reasoning_json`, ONE row, image-retention
  promotion, id-stamp, ONE fan-out, bounded DB retry. Plus the final-answer **reasoning stash**
  (`session->final_reasoning_json`, `tool_loop_stash_final_reasoning` at every text-returning path, cleared in
  `llm_call_prepare`) and **visual accumulate** (multi-`render_visual` turns keep all, not just the last).
  Adopted first in the existing backgrounded/client-gone path — which **fixed a latent bug** (that path
  dropped reasoning + visuals).
- **2b-i** — text-worker flip: arm `will_persist_turn`, unconditional persist via the helper, the **G4 exit
  invariant** (cancel-stash consumed inside the `!response` branch; cancelled-then-superseded handled in the
  supersede branch), bounded retry + error frame.
- **2b-ii** — voice-worker flip (`webui_audio.c`): **H1 pre-dispatch captures** (no `conn` deref in the
  disconnect-surviving tail), the assistant persist the voice path never had, G4 exits, the stale-visual
  clear it lacked — plus flipping `handle_save_message`'s assistant fall-through to a **pure accept-and-drop**
  (the permanent stale-client backstop; net −90 lines).
- **2c** — retired the dead www client-save (`streaming.js`), relocating the finalized-thinking reset.

**Scope correction (design doc §2/§7 understated it):** voice turns client-save their reply too, so
`webui_audio.c` had to be flipped or accept-and-drop would silently drop every voice reply. **Satellite is
out of scope** — `satellite_worker` does zero conv-DB writes. Reinvoke was deliberately **left on its inline
persist** (its own follow-up commit — a bisect must isolate it from the hot-path flip; it already reaches the
seam with no include change).

**Reviewed** correctness / architecture / security / efficiency on the full diff. **0 Critical / 0 High / 0
Medium.** Architecture: MERGE-AS-IS (the weak seam is "textbook"; duplication genuinely killed). Security:
net memory-safety + authz *improvement* (H1 UAF fixed; DB ownership-gate blocks cross-user writes; no XSS
surface — nothing client-supplied is persisted). Efficiency: neutral on the streaming hot path. Fixes folded:

- **correctness LOW — voice `pending_visual` clear** now under `tools_mutex` (was the one unlocked access).
- **correctness LOW — promise ⇔ persist** hardened: the `will_persist` stream_end gate now also requires
  `stream_conversation_id > 0`, so a conv-less turn (or one whose conversation creation failed mid-dispatch)
  can't stand the browser down for a save the server won't make.
- **arch LOW ×2** — docstring/comment accuracy (three paths adopt the helper today, reinvoke next commit;
  accept-and-drop is assistant-only in practice).

**Behavior changes to note (both correct, both intended):** (1) the empty-completion "Hmm…" fallback and
(2) an **errored** mid-stream partial are no longer persisted (they were client-saved before) — the server
has no complete reply to write on either, so they're display-only/live-DOM-only now. **Deferred:** the
efficiency micro-win (fold the two per-turn `session_get_for_reconnect` lookups into one — next-touch); the
security LOW (the pre-existing `conn`-field-read race in the voice capture — folds into the tracked
atomic-`client_data` item, and this diff *shrinks* its window). **Live-test §10+§12 matrix owed** (esp. the
local-model reasoning+visual fidelity gate on a STREAMING model, two-viewer voice, Stop+immediate-reask,
image-retention asserted directly, stale-client accept-and-drop).

## 12g. Reinvoke helper-adoption + efficiency fold (2026-08-28)

Two follow-up commits after the Phase-2 ship, both on `job_reinvoke.c` / `llm_tool_loop.c`:

- **Reinvoke → shared helper.** Both reinvoke persist sites (live `reinvoke_turn_entry`, detached
  `reinvoke_run_detached`) now call `webui_persist_final_answer` instead of their own inline
  `conv_db_add_message_with_tools` + `conv_event_notify_message_appended`. Reinvoke replies gain
  final-answer reasoning + accumulated visual + reply-body image-retention — the last of the four persist
  producers to adopt the seam, so the hot write path is now single-sourced everywhere. `mark_fired` still
  gates on a durable insert (invariant C3). **Correctness-reviewed; one MEDIUM folded in:** the live path
  now *consumes* `pending_visual`, so it needs the same turn-start `tools_mutex` clear the text/voice workers
  carry — else a prior interrupted `render_visual` on that viewer's session could splice a stale chart onto a
  job reply and durably persist it. Added. (LOW: corrected a comment that claimed a detached session's
  `current_stream_id` is always 0 — a job session bumps it on first streamed content; benign, since with no
  live browser viewer there is no matching client adopt entry.)
- **Stash-lookup fold (Phase-2 efficiency LOW, deferred in §12f).** The two per-turn stash helpers
  (`tool_loop_stash_finish_reason` + `tool_loop_stash_final_reasoning`) merged into one
  **`tool_loop_stash_final`** doing a single `session_get_for_reconnect` per final-answer return (was two).
  Skips the lookup entirely when neither a finish reason nor reasoning JSON is present. Behavior identical.

**Still owed:** the §10/§12 live-test matrix (esp. the reasoning+visual fidelity gate on a STREAMING model —
now also exercising the reinvoke path's new reasoning/visual fidelity), and the tracked security-LOW
`conn`-field-read race (atomic-`client_data` item).

## 12h. Phase-3 — live tool-step cross-viewer fan (#3, commit `fa5e8bc`, 2026-08-28)

Closes §12b item 3: a second viewer of a conversation now sees a turn's `tool_call`/`tool_result` steps
LIVE, not only on reload. **Ephemeral by design** — the fan emits a live `tool_step` frame ONLY; it writes
NOTHING to `conversation_events` (the messages table already persists tool steps, so reload rebuilds the
full trail). A bystander's live view is thus a subset of its own reload — no new durable state, no
double-render.

- **Server excludes the ORIGIN from recipients** (`webui_broadcast_tool_step`, gated on
  `session_id != origin_session_id`), so the client needs no load-bearing origin-suppression — the
  tool-iteration `stream_end` clears the streaming flag BEFORE the step emits, so a client-side "am I
  streaming" self-check is deterministically false by the time the frame arrives (master-plan caught this
  pre-build).
- **Cost-gate:** `webui_user_has_other_browser` (user-scoped v1) skips serialization entirely for the common
  single-viewer case. Conv-scoping is deferred to the atomic-`client_data` fix (it would deref the racy
  `client_data` via `webui_get_active_conversation_id`).
- **Seam:** `conv_event_tool_step_fanout` → weak `webui_broadcast_tool_step` (Layer-2 → Layer-4, same weak
  pattern as `conv_event_emit`). The tool loop branches `ev_observe` (jobs/durable, writes
  `conversation_events`) vs `fan_ephemeral = ev_conv>0 && !ev_observe` (interactive, live-only).
- **Client:** `streaming.js handleToolStep` (active-conv check only) renders via `DawnTranscript.addDebug`
  (escaped, live == reload) + auto-scroll-to-follow; `dawn.js` routes `tool_step`. Also fixed a pre-existing
  PTT-voice `events_observable`-stale bug (`webui_audio.c`).
- Reviewed master-plan + arch (pre-build) + correctness + security (post-impl); 0 crit/high/med. Live-verified.
- **Accepted v1 gaps:** inter-iteration assistant narration ("let me search…") is persisted but not fanned
  live (bystander live view is a subset of reload — symmetric on both clients; closing it = the
  `ws_response_t` addressing-model refactor). Post-execution emit → call+result arrive batched.

## 12i. Phase-4 — multi-target TTS (#4, 2026-08-28, live-verified; commit pending)

Closes §12b item 4: a text- or voice-originated turn's synthesized speech now plays on every TTS-enabled
**WEBUI browser** viewing the conversation (the origin included iff its own TTS is on), not just the origin.
The opposite addressing cell from #3 — #3 EXCLUDES the origin and is browsers-only-minus-self; #4 INCLUDES
the origin and fans to all speaker viewers.

**Load-bearing invariant:** synthesis (`text_to_speech_to_pcm`, global `tts_mutex`) runs ONCE per sentence
regardless of recipient count; the 22k→48k resample and the Opus encode also run once each and are shared
across all recipients of that format (≤2 encodes; non-Opus browsers reuse the 48k buffer). Synthesis load
is flat vs viewer count — the whole point.

- **UAF-safe fan (master-plan R1):** the fan queues each recipient's audio UNDER `s_conn_registry_mutex`,
  re-validating membership against the LIVE registry, so a session freed mid-synth is never queued —
  mirroring `broadcast_json_to_user_ex` (queue-under-registry-lock ⇔ session-destroy purges the queue under
  the same lock). Synth/encode happen with all locks released, between a phase-1 format scan and the phase-2
  send. (My first draft snapshotted then sent after unlock — a real UAF; the reviewer corrected it.)
- **Arm (arch HIGH-2):** `origin.tts_enabled || webui_audio_has_other_speaker(...)`. Keeping the origin flag
  as an independent sufficient condition preserves origin-speaks on a fresh VOICE turn (conv is bound
  mid-turn, so a purely conv-scoped arm would find nobody and silence the origin's own reply).
- **Bracket:** the full per-connection `state:speaking` (per sentence) / `state:idle` (turn end, fanned from
  each worker's teardown funnel `text_worker_end`/`audio_worker_end`) reaches the SAME recipient set as the
  audio, so a bystander's reactor brackets cleanly (WWFD caught the missing-idle stuck-speaking gap). A light
  state frame (no per-sentence metrics/LLM-config lookup) keeps the send off the per-session locks under the
  registry lock.
- **Seams:** `for_each_user_conn` (webui_internal.h) — the generalized registry walk that also retired the
  two duplicated existence walks (`any_session_matches`, `webui_user_has_other_browser`);
  `broadcast_json_to_user_ex` deliberately left (different shape, tracked addressing-model refactor). ONE
  membership predicate `conn_is_audio_target`; ONE conv-derivation `webui_origin_fan_target`. Codec
  transforms split into from-PCM primitives so synth/resample/encode are shared with the single-target path.
- **Two callbacks:** the single-target `webui_sentence_audio_callback` stays for reinvoke (§6d picker, ONE
  viewer) + Tier-2 satellite self-target — pick-one intact structurally (reinvoke cannot reach the fanout
  path). The text + PTT-voice workers use `webui_sentence_audio_fanout_callback`.
- **v1 SCOPE: browser-to-browser.** A non-origin Tier-2 satellite can't be conv-matched
  (`webui_get_active_conversation_id` returns 0 for non-WEBUI sessions — no satellite active-conversation
  membership signal). Cross-device fan to satellites is deferred (needs that binding); a satellite that
  ORIGINATES a turn is unaffected.
- **Clients unchanged:** stock www plays fanned audio (its handler gates only on local TTS-enabled, not on
  origin/streaming state); Aurora likewise (WWFD source-confirmed). Live-verified two-browser WebUI↔Aurora,
  independent per-client stop.
- Reviewed big-three + correctness + reuse (0 crit/high; all findings applied: WEBUI-scope, resample-once,
  UAF fix, light-state-under-lock, guards, `webui_origin_fan_target` extraction, `pcm22k_to_opus` fold).
  121/121 CI.

## 13. Finalization notes (2nd review pass — remaining low-severity)

Both reviewers' 2nd pass verdict: **approve-to-build with the must-folds above applied** (R1 Phase-0
stream_id threading, R2 emitter-stamp table §6e, R3 promise-break in Phase 1 at the shared cancel site,
§6a reload stamps `data-message-id`, §6b retire-the-refetch pick). Remaining implementer-note items:

- **Reasoning-stash lifetime (master R5):** clear the final-reasoning stash at **turn start (or on read)**,
  else a reasoning-less turn inherits and persists the previous turn's reasoning onto the wrong row — same
  hazard class as the `stream_id=0` rule.
- **Transcript-fallback field (correctness R5 / master R6):** the non-streamed transcript fallback
  (`session_manager_llm.c:258`) is also a **pre-write** send, so stamping `server_saved` there re-creates
  the intent/confirmation overload the `will_persist` rename removed. Preferred: route it through
  `will_persist` and teach the transcript handler (`dawn.js:215`) to gate on `server_saved || will_persist`
  — one field, one meaning. (Acceptable fallback: keep `server_saved` with an explicit "promise here" code
  comment so no one "fixes" it the wrong way.)
- **`<thinking>`-strip fidelity (master R4):** confirm `llm_call_finalize` canonicalization strips inline
  `<thinking>…</thinking>` that some local models emit (the client strips it at `streaming.js:492`); if
  not, server-persisted rows from those models reload with raw thinking text. Covered by the §12
  local-model fidelity gate.
- **Efficiency (deferred):** full-body `message_appended` fan-out to all the user's sessions — fine at
  single-user scale (`any_session_matches` pre-flight); revisit only under multi-user bandwidth.
- **Addressing-model lift (architecture, deferred):** `stream_id`/`conversation_id` on `message_appended`
  are envelope/routing metadata — write the correlation liftable, migrate to the envelope when the tracked
  addressing-model refactor lands.
