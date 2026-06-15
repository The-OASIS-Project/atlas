# DAWN Coding Harness (Code Projects + MCP Bridge) — Design & Implementation

**Status: SHIPPED (archived 2026-06-15).** DAWN can import or link a source repository,
index it into an external **code graph** (the operator-launched `codebase-memory-mcp` / cbm
server, reached over an HTTP+SSE MCP bridge — DAWN never spawns it), and let Friday answer
questions about the code. Two phases shipped to `main`; this is the consolidated historical
design + as-built record for the whole program. It graduated here from three working docs in
`dawn/docs/` (the Phase 1 plan, the Phase 2 plan, and the cbm-sharing note) once the program
settled.

**Feature flag:** the CMake *option* `DAWN_ENABLE_CODE_PROJECTS` defaults OFF (it pulls in
`DAWN_ENABLE_MCP_BRIDGE_TOOL` and libgit2 ≥ 1.6). As of **Phase 9**, the `default`/`full`/`debug`
presets enable both flags, so those builds compile the subsystem in and require libgit2 (the
installer builds it by default); the `local`/`server`/`ci` presets omit it. The runtime is still
gated by `[code_projects] enabled` in `dawn.toml`, so a preset build with no cbm server configured
is inert.

**User/operator guide (lives in-repo, not here):**
[`dawn/docs/CODING_PROJECTS.md`](https://github.com/The-OASIS-Project/dawn/blob/main/docs/CODING_PROJECTS.md)
— setup, the cbm sandbox, link-local permissions, refresh-vs-rebuild, troubleshooting.

**Ship history (branch `coding-harness` → `coding-harness-test`, all on `main`):**

*Phase 1 — foundation (MCP bridge + clone-and-index + WebUI):*
- `93a0669` — MCP bridge foundation (transport/client/schema, Steps 1–6).
- `bd87f45` — code projects subsystem + config/admin + review hardening (Steps 7–20).
- `66c1da2` — WebUI for code projects (Steps 17–19).
- `25fd748` — WebUI test-phase fixes + probe-before-import.
- `bf6a8a6` — clearer status when no code server is connected (degraded mode).
- `59ee5da` — `cbm-mcp` systemd service (mcp-proxy + cbm).
- `e3d2e39` — fix admin MCP grant + hide cbm path/slug from the LLM.
- `eebac02` — hide `cbm_list_projects` from the LLM.

*Phase 2 — branch tracking, link-local, rebuild, multi-project namemap, cbm sharing (PR #20):*
- `df8f28e` — branch tracking, link-local repos, rebuild, cbm sharing (schema **v66**).
- `76ffb64` — surface `CODING_PROJECTS.md` in README/GETTING_STARTED.
- `f9a2da6` — address Qodo + Copilot PR #20 review (SSRF same-origin guard, CI-grep
  hardening, Doxygen, `valid_url` fail-closed, `0`→`SUCCESS`).

*Phase 9 — install/UX polish:*
- `7bd5d6a` — flags added to the `default`/`full`/`debug` CMake presets (libgit2 install flipped
  default-on); cbm-mcp installer interactive `--local-roots` → systemd drop-in + traverse ACLs;
  `allowed_local_roots` surfaced in WebUI Settings (round-trip); `.dawn-tabs`/`.dawn-tab` CSS
  primitive extracted + 3 consumers migrated. `styledPrompt` deferred to the Odysseus Tier-0 batch.

---

## As-built reconciliation (read before the Phase 1 plan below)

The Phase 1 section that follows is the **original implementation plan, preserved verbatim**
for the design rationale (every agent-review fold-in is in it). A few concrete identifiers
drifted between the plan and what actually shipped — Phase 2 renumbered schema versions and
moved the opcode band. When the plan and this table disagree, **this table is the shipped
truth**:

| Concern | Phase 1 plan said | Shipped as |
|---|---|---|
| `mcp_user_access` schema version | v55 | **v64** (`auth_db_migrations_v64.c`) |
| `code_projects` table schema version | v56 | **v65** (`auth_db_migrations_v65.c`) |
| Phase 2 columns (`branch`/`kind`/`graph_name`) | — | **v66** (`auth_db_migrations_v66.c`) |
| Code-project admin opcodes | 0xB5–0xB8 (same band as MCP) | **0xD0–0xD6** (own band; MCP keeps 0xB0–0xB4, OTA owns 0xCx) |
| WebUI backend file | `webui_projects.c` | **`webui_code_projects.c`** |
| WebUI JS / CSS | `projects.js` / `projects.css` | **`code-projects.js`** / **`code-projects.css`** |
| cbm SSE port (placeholder) | 9000 | **9750** (`127.0.0.1`, no auth, localhost-only) |
| cbm `capabilities` | `read_only` (placeholder) | **`dangerous`** (cbm needs index/delete; the 3 mutating tools still get the hardcoded `TOOL_CAP_DANGEROUS` denylist) |

The plan's "**Phase 2 & 3 sketches**" section (auth/write-side/push) is **still future** — it is
*not* what the shipped Phase 2 delivered. Shipped Phase 2 = branch/link-local/rebuild/sharing
(documented in its own section after Phase 1 below). The "No-cbm minimal path (local provider)"
sketch in that section also remains unbuilt.

---

## Program overview

A voice/dev assistant that reasons over code needs three things DAWN didn't have: a **client**
for an external code-graph service, a **project registry** that owns clone/link/index/branch
lifecycle, and a **name-translation boundary** so the LLM never sees the host filesystem layout.

- **What does the indexing:** **cbm** (`codebase-memory-mcp`), an external MIT-licensed,
  **operator-launched** code-graph server. cbm speaks MCP over **stdio only**, so the operator
  fronts it with off-the-shelf **`mcp-proxy`**, which re-exposes it over HTTP+SSE on
  `127.0.0.1:9750`. DAWN is a pure SSE **client**.
- **Hard architectural invariant (operator-locked, 2026-05-29):** no harness code may call
  `fork`/`exec`/`popen`/`system`/`posix_spawn`/`daemon`/`dlopen`/etc. The bridge is socket-only;
  git is in-process via **libgit2**. `scripts/check_no_process_mgmt.sh` (a CI grep over the
  harness file globs, comment-stripped to avoid prose false-positives) enforces this on every
  build. `nftw()` is the one explicitly-allowed file-walk primitive.
- **Topology:** `DAWN bridge ──SSE──► mcp-proxy ──stdio──► cbm`. The `cbm-mcp` systemd service
  (`services/cbm-mcp/`) is "`mcp-proxy` wrapping `cbm`," run as the `dawn` user, graph cache at
  `/var/lib/dawn/cbm-cache`.

The five tightly-coupled Phase 1 components (MCP bridge, code-projects subsystem, libgit2 client,
WebUI popover, LLM scoping) are detailed below.

---

## Phase 1 — foundation (original implementation plan, preserved verbatim)

## Context

DAWN is growing a coding harness alongside its voice/memory/scheduler/HA capabilities. The destination is a developer's environment where Friday can answer code questions, reason about architecture, run impact analyses on diffs, and (later) help author + commit changes. This Phase 1 lays the rails: a user can paste a GitHub URL into DAWN's WebUI, DAWN clones the repo locally, indexes it via the external code-graph service `codebase-memory-mcp` (cbm-mcp), and Friday can answer questions about the codebase via cbm-mcp's tools.

cbm-mcp is an external MIT-licensed binary launched by the operator (systemd, tmux, docker — whatever they prefer). DAWN is a pure client; it speaks MCP over network sockets and never spawns or manages external processes.

**Hard architectural invariant (operator-locked, 2026-05-29):** No code added in this phase may call `fork`, `vfork`, `clone`, `clone3`, `posix_spawn*`, `exec*`, `popen`, `pclose`, `system`, `pidfd_*`, `daemon`, `setsid`, `setpgid`, `unshare`, `io_uring_setup`, `dlopen`, or any other process-creation/management primitive. A CI grep enforces this on every new source file. Bridge is socket-only; git is in-process (libgit2). Note: POSIX `nftw` is explicitly allowed for in-process directory walks.

**Five tightly-coupled components**:

| # | Component | Layer | Why it's needed |
|---|---|---|---|
| 1 | **MCP bridge** | Tools | DAWN-as-MCP-client over HTTP+SSE to operator-launched cbm-mcp |
| 2 | **Code projects subsystem** | Tools (DB) + Services (orchestrator) | Tracks imported projects; orchestrates clone + index via a provider vtable |
| 3 | **libgit2 in-process git client** | Foundation | Clones GitHub HTTPS repos to `/var/lib/dawn/source/<project>/` |
| 4 | **WebUI Projects panel** | UI | New "Coding" header-icon popover (DAWN's actual UI pattern is popover, not tabs) |
| 5 | **LLM scoping + system prompt** | LLM | Per-session active project; per-call visibility re-auth; system-prompt project list |

User-locked decisions (this round):
- Git client: **libgit2 in-process**
- Project visibility: **hybrid per-user + global flag**
- Project scoping: **hybrid active default + LLM override**
- WebUI placement: **header-icon popover** (not tabs — DAWN's existing IA is popover-based; mobile-overflow is a separate parallel effort, see §Mobile Tools pop-down)
- Diff render: **full re-render Phase 1**; defer extraction of a shared diff helper
- Clone thread: **dedicated thread**, `nice 10`, 256 KB stack — not in worker_pool
- Provider abstraction: **`code_graph_provider_t` vtable** now, cbm as first impl

Decisions from prior rounds:
- Transports eventually = stdio + HTTP+SSE + WS; **stdio permanently out-of-scope for the in-DAWN bridge** (stdio-only servers run behind operator-managed `mcp-proxy`)
- Per-server `capabilities` (default `dangerous`, opt-in `read_only`); per-tool override deferred (but see §Security: dangerous-tool denylist hardcoded for cbm's 3 mutating tools)
- Tools only in v1
- Bearer-token auth Phase 1 via secrets.toml primary + env-var fallback; OAuth Phase 2; WS Phase 2
- Per-user MCP allowlist via auth_db
- Admin surface: `dawn-admin mcp {list, status, grant, revoke}`

---

## End-to-end user flow

```
1. User: opens WebUI → clicks "Coding" header icon → popover opens → "Import" tab → pastes
        https://github.com/OWNER/REPO  → optional name "myrepo".
2. DAWN: SSRF-checks URL (scheme allowlist + host allowlist + url_is_blocked()).
   Validates name (NFKC + ^[a-z0-9][a-z0-9_-]{0,62}$ + realpath check).
   Writes code_projects row {status: "cloning"}; emits WS progress event via weak-symbol broadcast.
3. DAWN: libgit2 clones to /var/lib/dawn/source/myrepo/ in a dedicated clone thread,
        symlinks disabled, redirects disabled, file-count + size guarded.
4. DAWN: post-clone lstat sweep refuses S_IFLNK entries (defense in depth).
        status → "indexing"; bridge call provider->index_start(repo_path=local_path, mode="full").
5. DAWN: progress comes back as MCP `notifications/progress` if cbm supports it;
        otherwise adaptive polling 2s → 30s ceiling (worker frees clone thread between polls).
6. DAWN: status → "ready", indexed_at = now().
7. Session: at next session start, system prompt lists available projects + active project name.
8. User: "Friday, where do we handle MQTT QoS in myrepo?"
   LLM: calls cbm_search_graph (bridge re-checks user's visibility for the project
        per call, auto-fills project="myrepo" from active project, dispatches).
   Friday: narrates the cbm result.
```

---

## Architecture — module layout

```
include/tools/
  mcp_bridge.h              # opaque server handle, status, init/shutdown, FSM diagram in header comment
  mcp_client.h              # JSON-RPC + lifecycle FSM
  mcp_transport.h           # transport vtable
  mcp_bridge_schema.h       # JSON Schema → treg_param_t (with hardening caps)
  code_project_service.h    # orchestrator
  code_project_git.h        # libgit2 wrapper
  code_project_db.h         # CRUD on code_projects table (moved here per arch-H5)
  code_graph_provider.h     # provider vtable (cbm impl in Phase 1)

src/tools/
  mcp_bridge_tool.c         # config-driven registration + X-macro trampolines (cap 64)
  mcp_client.c              # protocol state machine + pending table
  mcp_transport_http_sse.c  # libcurl multi (curl_multi_poll) + SSE inbound, easy POST outbound
  mcp_bridge_schema.c       # JSON Schema translator (with hardening)
  code_project_tool.c       # native LLM tools: list / status / set_active
  code_project_service.c    # orchestrator (clone → provider->index_start → progress → ready)
  code_project_git.c        # libgit2 wrapper (clone, future fetch/push)
  code_project_db.c         # code_projects CRUD (was auth_db_code_projects per arch-H5)
  code_graph_provider_cbm.c # cbm implementation of provider vtable

include/auth/
  auth_db_mcp.h             # per-user MCP allowlist CRUD

src/auth/
  auth_db_mcp.c             # schema v54 migration + queries
  admin_socket_mcp.c        # 0xB0–0xB3 handlers
  admin_socket_code_project.c # 0xB5–0xB8 handlers

dawn-admin/
  dawn_admin_mcp.c
  dawn_admin_code_project.c

src/webui/
  webui_projects.c          # REST + WebSocket: project CRUD + progress events

www/
  js/ui/projects.js         # popover panel logic
  css/components/projects.css

scripts/
  check_no_process_mgmt.sh  # CI invariant grep

CMake:
  cmake/DawnTools.cmake     # DAWN_ENABLE_MCP_BRIDGE_TOOL=ON, DAWN_ENABLE_CODE_PROJECTS=ON
  Link libgit2 (find_package + version pin)
```

No subprocess management code. CI grep blocks regressions.

---

## Component 1: MCP bridge

### Lifecycle FSM

```
                    +-----+
                    | OFF |  (initial, daemon not yet up)
                    +--+--+
                       v
              +----------------+
              |  DISCONNECTED  | <-----+
              +--------+-------+       |
                       | first call    | drop / reconnect with backoff
                       v               |
              +----------------+       |
              |   CONNECTING   |       |
              +--------+-------+       |
                       | handshake ok  |
                       v               |
              +----------------+       |
              |   CONNECTED    |-------+
              +--------+-------+
                       | shutdown OR 3 failures within 60s
                       v
              +----------------+         +-----------------+
              | SHUTTING_DOWN  |  --->   |    DISABLED     |
              +----------------+         +-----------------+
              (added per arch-H3)        (operator clears via `dawn-admin mcp restart-alias` —
                                          name changed since DAWN doesn't restart processes;
                                          the command resets state to DISCONNECTED)
```

**SHUTTING_DOWN handling** (arch-H3): `mcp_bridge_shutdown()` sets SHUTTING_DOWN under `state_mtx`, broadcasts `state_cv`, fails all pending requests with `MCP_ERR_SHUTDOWN`, joins SSE reader with timeout, closes curl handles. Orchestrator's polling/wait loops check bridge state before each iteration and abort cleanly.

### Threading + lock order

Per server: 1 SSE reader thread + N caller threads + 1 global janitor.

```
mcp_bridge.servers_mtx (table lookup, briefly)
  → srv->state_mtx
     → srv->pending_mtx
        → pending->mtx (paired with pending->cv)
```

**Hard invariant (arch-C2)**: the code_project_service orchestrator NEVER holds a `code_projects` row lock across a bridge call. The orchestrator snapshots the active state, releases the row lock, makes the bridge call, then reacquires the row lock to write status updates. Bridge auto-fill resolves `active_project_id` from a copied snapshot of `session_get_command_context()`, not by reaching back into orchestrator state. Documented in `mcp_bridge.h` alongside the FSM diagram.

### HTTP+SSE transport

- **`curl_multi_poll`** with 1–5s timeout (eff-4) — never busy-loop.
- **TCP keepalive**: `CURLOPT_TCP_KEEPALIVE=1`, idle=60s, intvl=30s (eff-4).
- **TLS verify on by default**; `tls_verify = false` requires AND with `[mcp] dev_mode = true` (sec-M5); refuses registration otherwise.
- **Redirects disabled** (sec-H3): `CURLOPT_FOLLOWLOCATION=0`. Server should never redirect SSE; libgit2 redirects also handled separately (§Component 3).
- **Bearer token**: from `secrets.toml` primary (`auth_bearer_token_secret = "cbm_token"`), env-var fallback (`auth_bearer_env = "CBM_TOKEN"`) (std-H4); `sodium_memzero` the token buffer after handing to curl (sec-H5).

### Callback routing — pre-generated trampolines (X-macro, cap 64)

```c
#define MCP_TRAMPOLINE(N) \
  static char *mcp_cb_##N(const char *a, char *v, int *r) { \
     return mcp_bridge_dispatch(&s_slots[N], a, v, r); }
MCP_TRAMPOLINE(0) MCP_TRAMPOLINE(1) ... MCP_TRAMPOLINE(63)
static tool_callback_fn s_trampolines[64] = { mcp_cb_0, ... mcp_cb_63 };
```

**Capacity audit (arch-H1) — DONE (Phase 0, 2026-05-29)**: native count = **36** (`tools_init.c` `_register()` call-sites). With cbm's 14 that's 50/64 = 78%, at the threshold. **Resolved: bump `TOOL_MAX_REGISTERED` 64 → 128** (`include/tools/tool_registry.h:48`). Trampoline cap (64) and `PENDING_MAX` (64) stay — they bound bridged tools, and 36 + ≤64 = 100 ≤ 128.

### JSON-RPC protocol layer

- **Pending table**: sorted array + binary search; hard cap `PENDING_MAX=64` (eff-10) matching trampoline cap; reject excess with "bridge backpressure" error.
- **Frame buffer**: initial **16 KiB** (eff-2; was 4 KiB), doubles, hard cap 16 MiB → clean tool error, transition to DISCONNECTED.
- **Request ID**: atomic monotonic `uint64_t`.
- **Timeout**: `pthread_cond_timedwait` on `CLOCK_MONOTONIC`.
- **Handshake**: `initialize` → `notifications/initialized` → `tools/list`. Protocol version ≥ "2024-11-05".
- **Progress notifications**: SSE reader dispatches `notifications/progress` to the orchestrator's per-request progress callback (eff-3); avoids polling.
- **idle_close_seconds default 600s** (eff-11): SSE reader joins after 10 min idle, re-spins on next dispatch. ~1-2s reconnect latency on next use.

### Schema translation (`mcp_bridge_schema.c`)

| MCP input | → treg_param_t.type | Caps |
|---|---|---|
| `string` (no enum) | `STRING` | |
| `string` + `enum` | `ENUM` | **reject** (not truncate) tools with >16 enum values (sec-H1) |
| `integer` / `number` / `boolean` | direct | |
| `array` of scalars | `STRING` (JSON passthrough), `unit="json"` | `LOG_WARNING` at registration noting opaque fallback (arch-M4) |
| `object`, `oneOf`, `anyOf`, mixed | fallback `params` STRING | depth-limit `oneOf`/`anyOf` at 2 levels (sec-H1); reject `$ref` (sec-H1) |

**Translator hardening** (sec-H1):
- Cap properties per tool at 32; reject if exceeded.
- Cap tools per server at 32 (cbm has 14; headroom for growth).
- Per-server arena allocation (eff-5): `mcp_bridge_arena` sized from `tools/list` response; bump-allocate all params + strings; single free at shutdown.

**Description handling** (sec-H2): upstream description verbatim → wrap in `[BEGIN UNTRUSTED MCP TOOL DESCRIPTION from server '<alias>']\n<text>\n[END UNTRUSTED MCP TOOL DESCRIPTION]` markers; strip control chars / zero-width / bidi via `memory_filter_check()`-style normalization; truncate to `TOOL_DESC_MAX=512`. The LLM sees the wrapped form in the system prompt.

### Per-slot config struct (std-H2)

Each `mcp_slot_t` embeds:
```c
typedef struct {
   bool enabled;                   /* MUST be first per registry contract */
   const char *upstream_tool_name;
   mcp_server_t *server;
   tool_metadata_t metadata;       /* heap-allocated; lifetime = bridge */
   treg_param_t *params;           /* arena-allocated */
} mcp_slot_t;
```

For dangerous tools, registry validation passes because `enabled` is first and defaults to true (operator already opted server to `dangerous`).

**Heap-metadata deviation documented** (std-H1): block comment in `mcp_bridge_tool.c` explains why static-const is impossible (tool catalog discovered at runtime from upstream `tools/list`); test coverage in `test_mcp_bridge_tool.c` confirms registry accepts heap-allocated metadata pointers.

### MCP bridge config (TOML)

```toml
[mcp]
enabled = true
dev_mode = false                  # required true for any tls_verify=false server (sec-M5)

[[mcp.server]]
alias = "cbm"
url = "http://localhost:9000/sse"
transport = "http+sse"
enabled = true
capabilities = "read_only"
request_timeout_seconds = 30
idle_close_seconds = 600
tls_verify = true
auth_bearer_token_secret = "cbm_token"   # preferred path (std-H4)
# auth_bearer_env = "CBM_TOKEN"          # fallback for CI / containers

# DAWN-side hardcoded cbm denylist for mutating tools (sec-M1 / cbm carve-out):
# Even when capabilities="read_only", the bridge marks these three with TOOL_CAP_DANGEROUS
# at registration time and gates them behind admin-only invocation.

[mcp.server.cbm.tool_timeouts]
search_graph = 60
index_repository = 600
```

---

## Component 2: Code projects subsystem

### Provider vtable (arch-M6)

```c
/* code_graph_provider.h */
typedef struct code_graph_provider {
   const char *name;                                /* "cbm", future "github_mcp", etc. */

   int (*index_start)(const char *project_name, const char *repo_path,
                      const char *mode, int64_t *job_id_out);
   int (*index_poll_status)(const char *project_name, code_graph_status_t *out);
   int (*delete_project)(const char *project_name);

   /* Optional progress hook — if non-NULL, called on notifications/progress */
   void (*on_progress)(int64_t job_id, int percent, const char *message);
} code_graph_provider_t;

/* Phase 1: only cbm implementation. */
extern const code_graph_provider_t code_graph_provider_cbm;
```

Future providers (GitHub MCP, Sourcegraph, etc.) add a `code_graph_provider_<name>.c` file and a registration line. No orchestrator changes.

### Schema v55 (mcp_user_access) + v56 (code_projects)

**Production schema verified (Phase 0, 2026-05-29):** live DB + code are both at **v54** (`AUTH_DB_SCHEMA_VERSION = 54`; v54 = `scheduled_events.deliver_to`, messaging Phase 5 commit `2ec533c`). Versioning is tracked via the custom `schema_version` table (`SELECT version FROM schema_version`), **not** `PRAGMA user_version` (which reads 0). Our migrations are therefore **v55 (MCP)** and **v56 (code_projects)**.

```sql
-- v55 (MCP)
CREATE TABLE IF NOT EXISTS mcp_user_access (
   user_id      INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
   server_alias TEXT    NOT NULL,
   enabled      INTEGER NOT NULL DEFAULT 1,
   updated_at   INTEGER NOT NULL,
   PRIMARY KEY (user_id, server_alias)
);
CREATE INDEX idx_mcp_user_access_user ON mcp_user_access(user_id);

-- v56 (Code projects)
CREATE TABLE IF NOT EXISTS code_projects (
   id            INTEGER PRIMARY KEY AUTOINCREMENT,
   name          TEXT    NOT NULL UNIQUE,
   source_url    TEXT    NOT NULL,
   local_path    TEXT    NOT NULL,
   user_id       INTEGER REFERENCES users(id) ON DELETE CASCADE,   -- LOCKED Phase 0: CASCADE
   is_global     INTEGER NOT NULL DEFAULT 0,
   status        TEXT    NOT NULL DEFAULT 'pending',
   status_msg    TEXT,
   created_at    INTEGER NOT NULL,
   updated_at    INTEGER NOT NULL,
   indexed_at    INTEGER,
   imported_by   INTEGER REFERENCES users(id) ON DELETE SET NULL   -- nullable: NOT NULL + SET NULL was contradictory; importer identity is best-effort audit only
);
CREATE INDEX idx_code_projects_user ON code_projects(user_id);
```

**LOCKED (Phase 0):** `user_id` uses **`ON DELETE CASCADE`** — matches the `messaging_channels` v52 precedent. When a user is deleted, their private projects drop and the orphan-dir reconciliation (sec-M4) removes the stale clone. The `code_projects` SQL above must be updated from `SET NULL` to `CASCADE` accordingly.

Migrations split into `auth_db_migrations_v55.c` (MCP) and `auth_db_migrations_v56.c` (code_projects) (eff-17) to avoid worsening `auth_db_schema.c`'s 2828-line overrun.

### `code_project_db.{c,h}` (moved from auth_db per arch-H5)

Sits at `src/tools/code_project_db.c` paralleling `document_db.c`. CRUD on `code_projects`, visibility queries (`is_global=1 OR user_id=?`).

```c
int code_project_db_create(const code_project_t *p, int64_t *id_out);
int code_project_db_update_status(int64_t id, const char *status, const char *msg);
int code_project_db_set_indexed_at(int64_t id, time_t when);
int code_project_db_get(int64_t id, code_project_t *out);
int code_project_db_get_by_name(const char *name, code_project_t *out);
int code_project_db_list_visible(int64_t user_id, code_project_t out[],
                                  int max, int *count_out);  /* static array (eff-6) */
int code_project_db_set_global(int64_t id, bool is_global);
int code_project_db_delete(int64_t id);
```

`CODE_PROJECTS_MAX = 64` (eff-6, parallels `MCP_SERVERS_MAX = 16`).

### `code_project_service.c` orchestrator

```c
int code_project_import(int64_t requester_user_id, const char *source_url,
                        const char *desired_name, bool global,
                        int64_t *project_id_out);
int code_project_refresh(int64_t project_id);
int code_project_delete(int64_t project_id);
```

**Dedicated clone+index thread** (arch-H4, eff-7): not `worker_pool`. Pattern matches `scheduler_thread` and `memory_recovery_thread`. `nice 10`, `pthread_setname_np("dawn-code-import")`, `pthread_attr_setstacksize(256*1024)` (eff-21).

Worker thread does:
1. Validate name (NFKC + allowlist regex), validate URL (SSRF gate), reject if invalid.
2. Status → "cloning". Call `code_project_git_clone(opts)`. libgit2 progress callback emits WS updates via `code_project_broadcast_status_changed` weak symbol.
3. Post-clone lstat sweep: walk `local_path` with `nftw(..., FTW_PHYS)`; refuse `S_IFLNK` entries (sec-C1).
4. Status → "indexing". Call `provider->index_start(name, local_path, mode, &job_id)`.
5. Wait for completion via either `notifications/progress`+`done` (preferred) OR adaptive polling 2s → 30s ceiling. Worker thread frees during sleep so daemon can shut down (eff-3).
6. Status → "ready", `indexed_at = now()`.

**Decomposition expectation** (std-M3): worker thread function must be <50 lines; helpers `worker_do_clone`, `worker_do_index`, `worker_poll_index_status`. Acceptance criterion in Step 10.

**WebSocket broadcast via weak symbol** (arch-H2): orchestrator declares `extern void code_project_broadcast_status_changed(int64_t project_id) __attribute__((weak));`. WebUI defines the strong implementation. Mirrors `scheduler_broadcast_events_changed` precedent.

### dawn.toml [code_projects]

```toml
[code_projects]
enabled = true
source_root = "/var/lib/dawn/source"
default_index_mode = "full"
default_global = false
import_user_required = "admin"
max_repo_size_mb = 2048
max_file_count = 200000              # sec-H6
max_path_depth = 20                  # sec-H6
clone_depth = 0                      # 0 = full; 1 = shallow (eff-20)
allowed_host_pattern = "^([a-z0-9-]+\\.)?(github\\.com|gitlab\\.com|codeberg\\.org|bitbucket\\.org)$"  # sec-C3
```

**Startup warning** (eff-1): if `source_root` shares a filesystem with `/` or the auth.db path → `LOG_WARNING` with explicit message recommending an external mount. `dawn.toml.example` documents the Jetson eMMC recommendation.

---

## Component 3: libgit2 in-process git client

### Dependency

- libgit2 ≥ 1.6 (jammy apt is 1.1; verify before pinning — eff-19).
- License: GPLv2-with-linking-exception → GPLv3 compatible.
- TLS backend pinned to OpenSSL (matches DAWN's existing stack) (eff-16): `ldd` check during configure.
- Update `DEPENDENCIES.md`.

### `code_project_git.{c,h}`

```c
typedef struct {
   const char *source_url;
   const char *local_path;
   const char *branch;            /* NULL = HEAD */
   size_t max_size_bytes;
   uint32_t max_file_count;       /* sec-H6 */
   uint8_t max_path_depth;        /* sec-H6 */
   int clone_depth;               /* 0 = full, 1 = shallow */
   void (*progress_cb)(void *, int percent, const char *phase);
   void *progress_user_data;
} code_git_clone_opts_t;

int code_project_git_clone(const code_git_clone_opts_t *opts);
int code_project_git_fetch(const char *local_path);
int code_project_git_remove(const char *local_path);    /* nftw FTW_PHYS, in-process (std-L7) */
```

### Hardening

- **Symlinks off** (sec-C1): `git_repository_config_set_int32(cfg, "core.symlinks", 0)` immediately after init. Post-clone `nftw(..., FTW_PHYS)` lstat sweep refuses/removes any `S_IFLNK` entries (defense in depth — handles repos that bypassed core.symlinks).
- **Redirects off** (sec-H3): libgit2 `git_remote_callbacks.transport_message` cb refuses HTTP redirects. Or `git_libgit2_opts(GIT_OPT_SET_OWNER_VALIDATION, ...)` plus `CURLOPT_FOLLOWLOCATION=0` via the transport. Verify the approach against libgit2's API.
- **File count + path depth** (sec-H6): `transfer_progress_cb` aborts clone when entry count or max-depth exceeded.
- **Size guard** (eff-1): byte counter in `transfer_progress_cb` aborts at `max_size_bytes`.
- **`core.symlinks=false`**, **`core.fileMode=false`** (Windows safety carryover, harmless on Linux).
- **Tuning** (eff-8): `git_libgit2_opts(GIT_OPT_SET_MWINDOW_SIZE, 16<<20)` + `GIT_OPT_SET_MWINDOW_MAPPED_LIMIT, 64<<20`.
- **No credentials in Phase 1**: HTTPS public only. PAT / OAuth in Phase 2.

### Return code translation (std-H3)

All three wrappers translate libgit2's negative codes to `FAILURE` (1). Detail captured via `git_error_last()->message` and surfaced in `status_msg`. Error-message redaction: strip credentials matching `https?://[^:]+:[^@]+@` (sec-H5) before logging. One unit test asserts `== FAILURE` on a known bad-URL clone.

---

## Component 4: WebUI Projects panel

### Placement — new "Coding" header icon → popover (ui-C1)

Mirrors `#doc-library-btn` / `#doc-library-popover` pattern (`www/index.html:958-1022`, `www/js/ui/doc-library.js`). Sibling to Memory, Music, Scheduler, Documents buttons. New header icon labeled "Coding"; click toggles a popover dialog (`role="dialog" aria-modal="true"`).

Phase 1 ships the desktop popover. Mobile header overflow is a broader UI concern — see **§Mobile Tools pop-down (parallel follow-up)** below.

### Popover content

- **Top section — Import form** (admin or import-permitted users only — ui-M6):
  - URL field with regex validation on blur + HEAD-check before write (ui-H3)
  - Name field (auto-populated from URL slug; user can override)
  - "Make global" checkbox (admin only)
  - Import button
- **Project list** — full re-render on state change (full re-render confirmed for Phase 1):
  - Card per project: name + globe icon for global (matches `.global-badge` precedent — ui-M5), source URL, status chip, last_indexed, actions
  - Status chip colors via promoted `--status-*` tokens in `variables.css` (ui-L4) — shared with scheduler chips
  - Actions: Set Active, Refresh, Delete (delete + refresh both inline-confirm — scheduler pattern)
- **Empty states** (ui-M6):
  - First-time-no-projects: "Import a repo to get started" with import form prominent
  - Permission-denied: "No projects available to you" — import form hidden entirely

### Multi-session active project (ui-H4)

`set-active` POST includes `session_id`. Cards render THIS session's active state (e.g. badge "Active in this session"). Header line shows "Active in this session: <name>" when set.

### Long-running progress UI (ui-M2)

- Cloning: transfer-progress percent from libgit2 callback ("Cloning: 47%")
- Indexing: elapsed time + last status_msg line from provider ("Indexing: 3m 12s elapsed; parsing src/...")

Distinct treatment of the two phases so the user can tell they're still progressing.

### Reuse

- `.dawn-form` for inputs (NOT `.dawn-input` — that primitive is TODO'd; ui-H1)
- doc-library's `.global-badge` SVG + auth helper (ui-M5)
- scheduler's `.status-*` chip classes (promoted to `variables.css` tokens here)
- scheduler's inline-confirm pill pattern for destructive actions

### XSS hardening (sec-M3)

All render sites touching `status_msg`, `source_url`, `name` use `escapeHtml()` (per existing primitive). Strip control characters server-side in `code_project_db_*` writers as defense in depth.

### Lazy load (eff-12)

`projects.js` is lazy-loaded on first popover open (matches existing popover-loaded-on-click pattern), not bundled into initial paint.

---

## Component 5: LLM scoping + system prompt

### Active-project per session

New `session_t` field `int64_t active_project_id` + cached `char active_project_name[64]` (eff-14, avoids per-call SQLite lookup); both guarded by `session->llm_config_mutex` (std-L6, consistent with other per-session config fields).

Settable via:
- WebUI Projects panel "Set Active" button (per-session)
- Voice: `code_project_set_active(name)` native tool
- Config: `[code_projects] default_active = "myrepo"` fallback

### Per-call visibility re-auth (sec-H4)

Even with an `active_project_id` set on the session, every cbm tool call:
1. Resolves `user_id` from session
2. Calls `code_project_db_check_visible(user_id, project_name)` (cheap, indexed)
3. Denies cleanly if visibility was revoked since session start

Defense in depth: registration-time auth alone is insufficient (admin can flip global → private mid-session).

### System prompt injection

System prompt section (gated on `user has ≥1 visible project`):

```
--- AVAILABLE CODE PROJECTS ---
- myrepo (active, last indexed 2026-05-28 14:22)
- otherrepo (last indexed 2026-05-27 09:11)

When using cbm_* tools, the active project is used by default.
The LLM may pass `project="<name>"` to query a different one.
```

**Token cap** (arch-M3): list up to 3 projects in the system prompt; the rest are reachable via `code_project_list` native tool. ~150 tokens worst case.

### Bridge auto-fill of `project`

In `mcp_bridge_dispatch`, before forwarding args:
- If tool is cbm project-scoped AND LLM didn't supply `project`:
  - Snapshot `session->active_project_name` under lock; release lock
  - Re-check visibility (per-call re-auth, sec-H4)
  - Inject as `project = <name>` into params
- If LLM supplied `project`: pass through (and re-check visibility against it)
- If neither: error "No active project. Specify project or set active in WebUI."

### Native DAWN tools

- `code_project_list` (Layer 3 native)
- `code_project_set_active(name)` (Layer 3 native, updates `session_t`)
- `code_project_status(name?)` (Layer 3 native, defaults to active)

`code_project_import` and `code_project_delete` are NOT LLM-callable in Phase 1 — operator/user actions only.

### cbm dangerous-tool denylist (sec-M1)

Even though cbm is opted to `capabilities = "read_only"`, the bridge **hardcodes** these three cbm tools to register with `TOOL_CAP_DANGEROUS` AND admin-only invocation:
- `cbm_index_repository`
- `cbm_delete_project`
- `cbm_manage_adr` (when invoked with `mode="update"` — checked at dispatch)

Phase 1 denylist; Phase 3 generalizes to operator-configurable per-tool overrides.

---

## Security hardening (consolidated)

Section consolidates the security agent's findings. Every item is Phase 1, not deferred.

### Input validation

- **Project name** (sec-C2): NFKC normalize, then `^[a-z0-9][a-z0-9_-]{0,62}$`. Reject if normalized form differs from input. Final `realpath(local_path)` must prefix-match `realpath(source_root)`.
- **Source URL** (sec-C3): scheme allowlist (`https://` only), host allowlist from config (`allowed_host_pattern`), reuse `url_is_blocked()` from `src/tools/url_fetcher.c`, re-validate after every redirect (sec-H3 — but redirects are also disabled).
- **Bearer token** (sec-H5): primary path is `secrets.toml` keyed under server alias; env-var fallback only. `sodium_memzero` after handing to curl.

### LLM-prompt-area surface (sec-H2)

Tool descriptions wrapped in `[BEGIN UNTRUSTED MCP TOOL DESCRIPTION ...]` / `[END ...]` markers. Stripped of control / zero-width / bidi chars. Run through `memory_filter_check()` style normalization.

### Defense in depth

- **Per-call visibility re-auth** (sec-H4) on every bridged tool dispatch.
- **Symlink containment** (sec-C1) via `core.symlinks=false` + post-clone lstat sweep.
- **File count / depth caps** (sec-H6) enforced during clone, before checkout.
- **Concurrent-import race** (sec-M4): `INSERT … ON CONFLICT(name) DO NOTHING RETURNING id`; on startup, reconcile orphan dirs in `/var/lib/dawn/source/` against DB.

### Audit logging (sec-M7)

Audit-log entries for:
- `is_global` flip (admin-only)
- `mcp grant` / `mcp revoke`
- Project imports (which user, which URL)
- Project deletes (which user, what was deleted)

### CI invariant grep — expanded forbidden list

```bash
FORBIDDEN='\b(fork|vfork|clone|clone3|posix_spawn[p]?|exec[vl][peP]?|execve|popen|pclose|system|pidfd_open|pidfd_send_signal|waitpid|wait3|wait4|daemon|setsid|setpgid|unshare|io_uring_setup|dlopen)\b'
```

`nftw` explicitly allowed (commented in the script — std-L7).

### TLS backend audit (sec-I3)

`curl-config --ssl-backends` captured in CI; refuses build if p11-kit is in the backend list (potential helper-process spawn).

### Rate limiting (sec-M2)

WebUI `POST /api/code-projects/import` rate-limited via existing `webui_rate_limit` (commit `c807fc7`): 5 imports/min per user, 30/hour. CSRF token required.

---

## Per-user authorization

Two layers: MCP allowlist (per-user, per-server) + code-project visibility (per-user OR global).

### MCP authorization in callback flow

`mcp_bridge_dispatch`:
1. Resolve session → `user_id`.
2. `auth_db_mcp_check_access(uid, slot->server->alias, &allowed)`.
3. If denied → clear error message.
4. (Per sec-H4) re-check code-project visibility.

### Bootstrap

In `auth_db_init()` after schema creation, find admin user and `INSERT OR IGNORE` enabled=1 rows for every configured `[[mcp.server]]` alias.

---

## Admin socket + CLI

### Opcode allocations (arch-C1 — moved out of memory/messaging bands)

```c
/* include/auth/admin_socket.h */

/* Phase 8: MCP bridge. */
ADMIN_MSG_MCP_LIST          = 0xB0,
ADMIN_MSG_MCP_STATUS        = 0xB1,
ADMIN_MSG_MCP_GRANT         = 0xB2,
ADMIN_MSG_MCP_REVOKE        = 0xB3,
ADMIN_MSG_MCP_RESET         = 0xB4,   /* sec-M9: clears DISABLED state */

/* Phase 9: Code projects. */
ADMIN_MSG_CODE_PROJ_LIST    = 0xB5,
ADMIN_MSG_CODE_PROJ_IMPORT  = 0xB6,
ADMIN_MSG_CODE_PROJ_REFRESH = 0xB7,
ADMIN_MSG_CODE_PROJ_DELETE  = 0xB8,
/* Next free: 0xB9. */
```

Update the "Next free memory opcode: 0x91" comment to note 0x91-0x9F stay for memory follow-ups; 0xA0-0xAF reserved for messaging Phase 4-6; 0xB0+ for MCP/code-projects.

### CLI

- `dawn-admin mcp {list, status, grant, revoke, reset <alias>}`
- `dawn-admin code-project {list, import <url> [--name n] [--global], refresh <name>, delete <name>, set-default <name>}`

### WebUI Settings exposure (std-M5)

`[mcp]` (`enabled`, `dev_mode`) and `[code_projects]` (`enabled`, `source_root`, `max_repo_size_mb`, `import_user_required`, `max_file_count`, `max_path_depth`, `clone_depth`, `allowed_host_pattern`) added to `www/js/ui/settings.js` `SETTINGS_SCHEMA`. Bearer-token settings explicitly excluded (credentials).

---

## CMake + tools_init wiring

```cmake
option(DAWN_ENABLE_MCP_BRIDGE_TOOL "Enable MCP bridge tool (Phase 1: HTTP+SSE only)" OFF)
option(DAWN_ENABLE_CODE_PROJECTS   "Enable code projects subsystem (requires libgit2 + MCP bridge)" OFF)

if(DAWN_ENABLE_MCP_BRIDGE_TOOL)
    add_definitions(-DDAWN_ENABLE_MCP_BRIDGE_TOOL)
    list(APPEND TOOL_SOURCES
        src/tools/mcp_bridge_tool.c
        src/tools/mcp_client.c
        src/tools/mcp_transport_http_sse.c
        src/tools/mcp_bridge_schema.c)
    list(APPEND AUTH_SOURCES
        src/auth/auth_db_mcp.c
        src/auth/admin_socket_mcp.c
        src/auth/auth_db_migrations_v55.c)
endif()

if(DAWN_ENABLE_CODE_PROJECTS)
    if(NOT DAWN_ENABLE_MCP_BRIDGE_TOOL)
        message(FATAL_ERROR "DAWN_ENABLE_CODE_PROJECTS requires DAWN_ENABLE_MCP_BRIDGE_TOOL")
    endif()
    find_package(Libgit2 REQUIRED)
    add_definitions(-DDAWN_ENABLE_CODE_PROJECTS)
    list(APPEND TOOL_SOURCES
        src/tools/code_project_service.c
        src/tools/code_project_git.c
        src/tools/code_project_tool.c
        src/tools/code_project_db.c
        src/tools/code_graph_provider_cbm.c)
    list(APPEND AUTH_SOURCES
        src/auth/admin_socket_code_project.c
        src/auth/auth_db_migrations_v56.c)
    list(APPEND WEBUI_SOURCES
        src/webui/webui_projects.c)
    target_link_libraries(dawn PRIVATE git2)
endif()

# CI invariant grep
add_custom_target(no_process_mgmt_check ALL
    COMMAND ${CMAKE_SOURCE_DIR}/scripts/check_no_process_mgmt.sh
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
add_dependencies(dawn no_process_mgmt_check)
```

Both options default OFF; flip ON after smoke validates.

`tools_init.c`:

```c
#ifdef DAWN_ENABLE_MCP_BRIDGE_TOOL
   if (mcp_bridge_init() != 0) {
      OLOG_WARNING("MCP bridge init failed; continuing without bridged tools");
   }
#endif
```

---

## Implementation sequence

| # | Step | Files | Tests |
|---|---|---|---|
| 0 | **Pre-checks** ✅ DONE 2026-05-29 | — | **36 native tools / cap 64** (78% → bump cap to 128, arch-H1); **live schema v54** via `schema_version` table → migrations are **v55/v56** (arch-M7); **libgit2 apt 1.1.0** < 1.6 → build-from-source in setup.sh (eff-19); **curl = OpenSSL v3+**, no p11-kit (sec-I3 ✅); **tomlc99** w/ `parse_inline_table` + `[[ ]]` ✅ (std-M1); admin opcodes 0xB0–0xB8 confirmed free |
| 1 | **CI invariant grep + script** | scripts/check_no_process_mgmt.sh, CMakeLists.txt | Run script against current tree — must pass with empty matches |
| 2 | **MCP bridge transport** | mcp_transport_http_sse.{c,h}, mcp_transport.h | Loopback HTTP+SSE fixture in in-test pthread; `curl_multi_poll` timeout; SSE frame parse + POST roundtrip + reconnect on drop |
| 3 | **MCP bridge client** | mcp_client.{c,h} | Mock SSE server; handshake, call, timeout, drop+reconnect, SHUTTING_DOWN cleanup |
| 4 | **MCP schema translator (hardened)** | mcp_bridge_schema.{c,h} | All type mappings, fallback, required flags, enum reject (>16), $ref reject, depth-limit oneOf/anyOf, property cap, prompt-injection wrap |
| 5 | **Schema v55 + auth_db_mcp** | auth_db_mcp.{c,h}, auth_db_internal.h, auth_db_migrations_v55.c | Migration; grant/revoke/check |
| 6 | **MCP bridge tool registration** | mcp_bridge_tool.c, mcp_bridge.h | Heap-allocated metadata; trampoline routing; per-slot enabled-first config; auth deny path; auto-project-injection; cbm dangerous-tool denylist |
| 7 | **TOML wiring (MCP)** | config_parser.c, config_validate.c, dawn_config.h | extend test_config_validate.c; secrets.toml bearer-token path |
| 8 | **admin_socket_mcp + dawn-admin mcp** | admin_socket_mcp.c, dawn_admin_mcp.c, admin_socket.h, dawn-admin/main.c | extend test_dawn_admin.sh; opcode 0xB0–0xB4 |
| 9 | **Code graph provider vtable + cbm impl** | code_graph_provider.h, code_graph_provider_cbm.c | Vtable contract; cbm impl calls bridge correctly |
| 10 | **Schema v56 + code_project_db** | code_project_db.{c,h}, auth_db_internal.h, auth_db_migrations_v56.c | Migration; CRUD; visibility filter; CASCADE on user delete |
| 11 | **libgit2 wrapper (hardened)** | code_project_git.{c,h} | File-fixture bare repo; size guard; file-count guard; path-depth guard; symlinks-off; post-clone lstat sweep; redirect refusal; libgit2-error → FAILURE translation |
| 12 | **code_project_service orchestrator** | code_project_service.{c,h} | Mocked bridge + mocked git; full state machine; failure paths; weak-symbol broadcast; dedicated thread (not worker_pool); name + URL validation; concurrent-import race |
| 13 | **Native project tools** | code_project_tool.c | list/set_active/status; session active-project resolution; cached name avoids per-call SQLite lookup |
| 14 | **System prompt injection + per-call re-auth** | prompt_compose.c, session_manager.c | active_project field added; system prompt section (max 3 projects); per-call visibility re-check |
| 15 | **TOML wiring (code_projects)** | config_parser.c, config_validate.c, dawn_config.h | extend tests; source_root warning; allowed_host_pattern validation |
| 16 | **admin_socket_code_project + dawn-admin code-project** | admin_socket_code_project.c, dawn_admin_code_project.c, admin_socket.h, dawn-admin/main.c | extend test_dawn_admin.sh; 0xB5–0xB8 |
| 17 | **WebUI Settings schema entries** | www/js/ui/settings.js | [mcp] + [code_projects] surfaced in WebUI Settings |
| 18 | **WebUI backend** | webui_projects.c | curl + WS integration test; CSRF; rate limit |
| 19 | **WebUI Coding popover** | www/index.html (header button), www/js/ui/projects.js, www/css/components/projects.css | Manual UI test; ui-design-architect review; matches doc-library popover pattern |
| 20 | **End-to-end smoke** | tests/smoke_test_harness.sh, tests/fixtures/ (bundled bare repo) | file:// URL to bundled bare repo (eff-15); operator launches cbm+proxy; harness checks `code_project_import` → `cbm_search_graph` |
| 21 | **atlas design docs** | atlas/dawn/archive/CODE_PROJECTS_DESIGN.md, atlas/dawn/archive/MCP_BRIDGE_DESIGN.md | untracked until ship (design-doc policy) |

Plus throughout: GPL header on every new file (std-L1); Doxygen on every public header API (std-L2).

---

## Reuse

| Need | Reuse |
|---|---|
| TOML array-of-tables | `parse_audio_named_devices` at `src/config/config_parser.c:235-308` |
| Config validation macros | `src/config/config_validate.c:34-66` |
| Admin socket extraction | `src/auth/admin_socket_memory.c` precedent |
| dawn-admin subcommand | `dawn-admin/main.c:1710-1850` |
| Session → user_id | `tool_get_current_user_id()` at `include/tools/tool_registry.h:733-738` |
| Per-user + global visibility | `document_t.is_global` + `document_db_list` filter |
| Doc library popover pattern | `www/index.html:958-1022`, `www/js/ui/doc-library.js` (closest functional analog) |
| Status chips | `scheduler.css` `.status-*` classes (promote to `variables.css`) |
| Inline-confirm pill | scheduler-queue.js |
| Global badge SVG | doc-library globe icon |
| SSRF gate | `url_is_blocked()` from `src/tools/url_fetcher.c` |
| Untrusted-content marker pattern | Tavily url_fetch wrap (around April-May 2026) |
| Rate limiter | `webui_rate_limit` (commit `c807fc7`) |
| Audit log | existing auth_db audit-log helpers |
| SSE parsing | `src/llm/llm_streaming.c` |
| Weak-symbol broadcast | `scheduler_broadcast_events_changed` precedent |
| ENV_SECRET log redaction | existing daemon-side `ENV_SECRET` macro |

---

## Verification

### CI invariant — no process management

Mandatory CMake custom target + GitHub Actions step (forbidden list above). Runs before bridge/project sources compile.

### Network primitives audit (allowed)

- **libcurl** — sockets only, OpenSSL backend (pinned).
- **libgit2** — uses libcurl/libssh2 internally; sockets only. No `git` subprocess.
- **pthreads, sockets, json-c, sqlite3, libsodium, libwebsockets** — all in-process.
- **nftw** — POSIX file walk, in-process (not a forbidden call).

### Test stages

1. **CI invariant check** — runs first, fails fast.
2. **Unit tests** (`ctest -L ci`): Unity per module. Transport tests use in-test pthread loopback fixture. libgit2 wrapper tests use bundled bare-repo fixture in `tests/fixtures/`.
3. **Smoke test** (`tests/smoke_test_harness.sh`):
   - Pre-check: probe `http://localhost:9000/sse`. Skip with clear message if unreachable.
   - Fixture: `file://` URL to bundled bare repo (eliminates network dependency).
   - Test path: `code_project_import` via admin socket → wait for ready → `cbm_search_graph` → assert non-empty.
   - Fail mode: `DAWN_SMOKE_REQUIRE_HARNESS=1` in CI.
4. **Manual end-to-end**: operator launches cbm + mcp-proxy; user imports a public GitHub repo via WebUI; watches status transitions; asks Friday a code question.
5. **Build sweep**: `cmake --preset debug -DDAWN_ENABLE_MCP_BRIDGE_TOOL=ON -DDAWN_ENABLE_CODE_PROJECTS=ON && make -j8 && ./format_code.sh --check && ctest -L ci`
6. **TSAN sweep**: lock-inversion + bridge FSM + orchestrator thread + WebSocket broadcast.
7. **ASAN sweep**: heap-metadata + project-row + arena lifetime.

---

## Deployment (operator setup)

**Authoritative install guide + artifacts: [`services/cbm-mcp/`](../services/cbm-mcp/)**
(systemd unit, `cbm-mcp.conf`, `install.sh`, README). This section is the design
rationale; the service dir is what you run.

### Transport reality: cbm is stdio-only → an adapter is required

cbm (codebase-memory-mcp) speaks MCP **only over stdio** and expects an agent to
launch it as a child process. DAWN's bridge is the opposite — an **HTTP+SSE
client** that connects to a running URL — and DAWN **never spawns subprocesses**
(the CI-enforced invariant). The two don't connect directly. The operator runs a
thin off-the-shelf adapter, **`mcp-proxy`**, which owns the cbm stdio child and
re-exposes it over SSE:

```
  DAWN bridge ──SSE(URL)──► mcp-proxy ──stdio──► cbm
  (SSE client,              (adapter, owns        (stdio MCP,
   spawns nothing)           the cbm child)        launched BY mcp-proxy)
```

This keeps the no-subprocess rule intact: the *operator's* service launches cbm,
never DAWN. The `cbm-mcp` systemd service is exactly "`mcp-proxy` wrapping `cbm`,"
running as the `dawn` user so it can read `/var/lib/dawn/source` and write its
graph cache. The adapter emits the legacy-SSE `event: endpoint` handshake DAWN's
bridge waits for (protocol `2024-11-05`).

### Canonical `dawn.toml` (real deployment)

The §"MCP bridge config (TOML)" example earlier in this doc used **placeholders**
(port 9000, `read_only`, a bearer token). The shipped cbm deployment is
localhost-only with no auth, so the real block is:

```toml
[mcp]
enabled = true
dev_mode = true            # required: tls_verify=false on plain-http localhost (sec-M5)

[[mcp.server]]
alias = "cbm"              # MUST be exactly "cbm" — code_graph_provider_cbm keys on it
url = "http://127.0.0.1:9750/sse"
transport = "http+sse"
enabled = true
capabilities = "dangerous" # cbm needs index_repository / delete_project
tls_verify = false

[code_projects]
enabled = true
```

cbm has no authentication of its own, so the SSE port is bound to `127.0.0.1`
only and must never be exposed off-box (no bearer token is involved on this
path). The denylist carve-out (sec-M1) still marks cbm's three mutating tools
`TOOL_CAP_DANGEROUS` regardless of `capabilities`.

### Build caveat: libgit2 disabled

cbm's *optional* libgit2 fast-path (git-history parsing) fails to compile against
a modern system libgit2: `cbm.c` includes `<git2.h>` for the `git_allocator`
type, which now lives in `<git2/sys/alloc.h>`. Since DAWN ships a libgit2 in
`/usr/local`, pkg-config auto-enables the broken path on this box. The service
builds cbm with `LIBGIT2_FLAGS= LIBGIT2_LIBS=`; cbm then falls back to
`git log` (functionally complete). Upstream one-line fix worth filing: add
`#include <git2/sys/alloc.h>` to `cbm.c`.

### Degraded mode (no code server)

If `cbm-mcp` isn't running/configured, `code_project_import` still clones the repo
but the index step reports a clear status — *"clone ready, but no code server
connected — start cbm-mcp, then re-index"* — rather than a bare "indexing
failed." `worker_do_index` gates on `code_graph_provider_cbm.is_available()`
(backed by `mcp_bridge_server_connected("cbm")`). See also the §"No-cbm minimal
path" sketch below for the planned local fallback provider.

---

## Mobile Tools pop-down (parallel follow-up)

**Scope is bigger than Phase 1 and benefits every header icon, not just Coding.**

DAWN's header icons (Memory, Music, Scheduler, Documents, + new Coding) already overflow on mobile viewports. This is a known UI debt independent of the coding harness work. Right shape:

- A "Tools" overflow button on narrow viewports (≤640px or similar breakpoint)
- Tapping it opens a pop-down menu listing all header tools (icon + name)
- Selecting one opens its existing popover
- Desktop unchanged

This is its own ship — affects every existing tool icon, needs UI review, doesn't gate Phase 1. File as a TODO entry independent of the coding harness; tackle when at least 2 of {Memory, Music, Scheduler, Documents, Coding, Settings} need attention on mobile.

Phase 1 of the coding harness simply adds the "Coding" header icon without doing this overflow work.

---

## Forward-looking sketches from the Phase 1 plan (auth / write-side — STILL FUTURE)

> ⚠️ These are the Phase 1 plan's *original* forward-looking sketches and are **not** what the
> shipped Phase 2 delivered. The actual shipped Phase 2 (branch / link-local / rebuild / sharing)
> has its own section further below. The "Authentication + write-side" work here remains unbuilt.

**Phase 2 (sketch) — Authentication + write-side**:
- libgit2: commit, push, branch create/checkout
- Auth: PAT (secrets.toml) for private repos, OAuth flow extension
- Transport: WebSocket via libwebsockets
- MCP OAuth: extend `oauth_client.c` for per-MCP-server providers

**Phase 3 — Deeper IDE feel**:
- Per-tool MCP capability overrides (formalizes the cbm-denylist into config)
- Optional MCP resources translation → DAWN documents
- Optional MCP prompts translation → system-prompt presets
- Project detail view (file tree via cbm.search_graph; ADR editor via cbm.manage_adr)
- Diff query mode via cbm.detect_changes

### No-cbm minimal path (local code_graph_provider fallback)

**Problem:** today 100% of code *intelligence* (indexing, graph search, reading)
is delegated to cbm via the bridge. With cbm absent, `import` clones the repo to
`source_root/<name>/` but the index step fails and the project lands in `error`;
there are zero query tools (the `cbm_*` tools only register when cbm is
connected) and no local reader of the clone. So a DAWN install without cbm
configured has a Coding popover that can clone repos it can do nothing with —
weird/dead-end UX. Filed 2026-06-13 during pre-cbm UI testing.

**Fix — add a second `code_graph_provider` impl (`code_graph_provider_local`)**
that operates directly on the cloned working tree, no external server:
- `index_start`: no-op (or build a lightweight in-process file index / reuse the
  RAG `document_index_pipeline` over the clone). Status becomes `ready` instead
  of `error`.
- query surface: literal/grep search + paginated file read over `source_root/
  <name>/` — reuse the existing `document_grep` / `document_read` machinery
  (main shipped structure-aware chunking + `document_grep` already), or a small
  `nftw` + ripgrep-style in-process scan (in-process only — honor the
  no-subprocess invariant). Expose as native `code_project_search` /
  `code_project_read` tools (NOT bridged `cbm_*`), so they exist with zero MCP
  config.
- **provider selection** (the vtable seam already exists): prefer cbm when its
  server is connected; else fall back to local. A `[code_projects] provider =
  "auto" | "cbm" | "local"` knob makes it explicit. `auto` = cbm-if-connected,
  else local.
- net effect: the feature is useful out of the box (grep/read the repo Friday
  imported) and *upgrades* to graph-aware answers when cbm is wired. Removes the
  "clone-then-error" dead end.

Effort: medium — the vtable + worker already abstract this; the work is the
local provider impl + 2 native query tools + WebUI surfacing of `ready (local)`
vs `ready (cbm)`. Sequence it as the first Phase-2 item since it materially
changes the default-install experience.

---

## Open decisions remaining (post agent-review)

**ALL RESOLVED in Phase 0 (2026-05-29).** Locked answers below; no open blockers before Step 1.

1. **`code_projects.user_id` ON DELETE behavior** — **LOCKED: CASCADE** (matches messaging_channels v52 precedent). `imported_by` made nullable + `ON DELETE SET NULL` (was a contradictory `NOT NULL` + `SET NULL`). (arch-M1)
2. **libgit2 ≥ version pin** — **LOCKED: build-from-source in `setup.sh`.** Confirmed apt candidate is 1.1.0 (< 1.6 required), not installed, absent from pkg-config. Guarded build step pins a version with the OpenSSL backend; `DEPENDENCIES.md` updated. (eff-19)
3. **MCP `notifications/progress` actually supported by cbm?** — **Deferred to a pre-Step-9 spike** (can't probe cbm source from the build host now). Adaptive polling 2s → 30s is the safe fallback regardless, so this does not gate Phase 1. (eff-3)
4. **`clone_depth` default** — **LOCKED: 0 (full)** with shallow (1) knob. (eff-20)
5. **`TOOL_MAX_REGISTERED` check** — **LOCKED: bump 64 → 128.** Audit found 36 native tools (78% of 64 with cbm's 14); only 14 free slots and per-server cap is 32, so 64 has no real bridge headroom. The trampoline cap (64) and `PENDING_MAX` (64) can stay — they bound *bridged* tools/in-flight requests, and 36 native + ≤64 bridged = 100 ≤ 128 fits. Revisit the trampoline cap only if we ever want >64 bridged tools live at once. (arch-H1)

---

## Agent review fold-ins (changelog from prior plan version)

Material changes from the prior plan based on the parallel review of architecture / embedded-efficiency / security / UI / coding-standards agents:

- **Opcodes** 0x91–0x99 → **0xB0–0xB8** (arch-C1; sec-M9 added 0xB4 `mcp reset`)
- **WebUI placement**: tab strip → **header-icon popover** (ui-C1)
- **Diff render claim dropped**: **full re-render** for Phase 1 (ui-C2)
- **Clone thread**: worker_pool → **dedicated thread** with `nice 10`, 256 KB stack (arch-H4, eff-21)
- **Provider abstraction**: hardcoded cbm verbs → `code_graph_provider_t` **vtable**, cbm as first impl (arch-M6)
- **DB module placement**: `src/auth/auth_db_code_projects.c` → `src/tools/code_project_db.c` paralleling `document_db.c` (arch-H5)
- **SHUTTING_DOWN FSM state** added (arch-H3)
- **WebUI broadcast** via weak-symbol pattern (arch-H2)
- **Symlink containment** via `core.symlinks=false` + post-clone lstat sweep (sec-C1)
- **Project name validation**: NFKC + allowlist regex + realpath check (sec-C2)
- **SSRF gate** on source_url via `url_is_blocked` + scheme/host allowlist (sec-C3)
- **Schema translator hardening**: property caps, `$ref` reject, `oneOf`/`anyOf` depth limit, enum reject >16 (sec-H1)
- **Prompt-injection wrap** on tool descriptions (sec-H2)
- **libgit2 redirects disabled** (sec-H3)
- **Per-call visibility re-auth** (sec-H4)
- **Bearer token**: `secrets.toml` primary + env-var fallback; `sodium_memzero` (sec-H5, std-H4)
- **File-count + path-depth caps** (sec-H6)
- **Concurrent-import race** + orphan-dir reconciliation (sec-M4)
- **dangerous-tool denylist** for cbm's 3 mutating tools (sec-M1)
- **Audit log** for is_global flips, grants, imports, deletes (sec-M7)
- **TLS backend pin** to OpenSSL, refuse p11-kit (sec-I3, eff-16)
- **`source_root` Jetson warning** + dawn.toml.example doc (eff-1)
- **JSON-RPC frame buffer** initial 4 KiB → **16 KiB** (eff-2)
- **Polling → `notifications/progress`** preferred; fallback adaptive 2s → 30s (eff-3)
- **`curl_multi_poll`** with timeout, TCP keepalive (eff-4)
- **Per-server arena allocation** for translator (eff-5)
- **Static-array list API** for code_project_db (eff-6)
- **libgit2 mwindow tuning** (eff-8)
- **idle_close_seconds** default 0 → **600** (eff-11)
- **PENDING_MAX = 64** explicit cap (eff-10)
- **Session-cached active project name** to avoid per-call SQLite (eff-14)
- **Bundled bare-repo fixture** for smoke test (eff-15)
- **Lazy-load `projects.js`** (eff-12)
- **Project metadata heap deviation** documented; per-slot config with `enabled` first (std-H1, std-H2)
- **libgit2 negative returns** → translated to FAILURE (std-H3)
- **TOML inline-table feasibility check** pre-implementation (std-M1)
- **WebUI Settings schema** entries added (std-M5)
- **Migration files split** to avoid worsening auth_db_schema.c overrun (eff-17)
- **Multi-session active-project** explicit handling (ui-H4)
- **Empty state copy** differentiates first-time vs permission-denied (ui-M6)
- **Status token promotion** to variables.css (ui-L4)
- **Mobile Tools pop-down** filed as parallel follow-up (user direction)
- **CI invariant grep** expanded: clone3, daemon, setsid, setpgid, unshare, io_uring_setup, dlopen; nftw allow-comment (std-L7, sec-I2)

---

## Phase 2 — branch tracking, link-local repos, rebuild, multi-project namemap, cbm sharing

Phase 1 shipped clone-from-URL + single-project indexing. Live use surfaced four gaps, all
closed in PR #20 (`df8f28e`). This section is the **as-built** record; the original plan is
`docs/CODING_HARNESS_PHASE2_PLAN.md` (working doc, retired). Plan reviewed by
architecture-reviewer + master-plan-reviewer (2026-06-14); five-agent code review applied before
merge.

### The four gaps

1. **Stale graph after a cbm upgrade or branch change.** `refresh` re-ran cbm's *incremental*
   index (preserves the old graph), and the "full reindex" escape hatch was **silently broken**:
   `code_project_delete` was **doubly wrong** — it passed DAWN's clean project name instead of
   cbm's path-derived **slug**, *and* sent it under the arg key `"project_name"` when cbm expects
   `"project"`. So the cbm graph was **never deleted**; re-importing the same name re-attached a
   stale graph. A live bug, not future risk.
2. **No branch control.** The git wrapper supported branch-on-clone but the orchestrator
   hardcoded `.branch = NULL`; there was no DB column and no fetch/checkout to *change* a branch.
3. **No way to point DAWN at a local checkout** the developer (and Claude Code) was already
   editing — only URL-clone existed.
4. **Sharing cbm with Claude Code** so both assistants reason over one graph.

**Locked decisions:** support **both** clone-from-URL and link-existing-local-path; share cbm via
the **same SSE endpoint** (one process, shared cache — accept that a long index head-of-line
blocks the other side); **two reindex actions** — cheap `refresh` (fetch + incremental) and clean
`rebuild` (delete graph + full re-index).

### Schema v66 (`auth_db_migrations_v66.c`)

Adds three columns to `code_projects` (table itself created v65):

- `branch TEXT` — the tracked branch (clone repos) or the detected checked-out branch (local).
- `kind TEXT NOT NULL DEFAULT 'clone'` — `clone` | `local`. **C-validated in `db_create`**, not a
  CHECK constraint, so every write must go through `code_project_db_create`.
- `graph_name TEXT` — cbm's path-derived **slug**, persisted so `delete`/`rebuild` can target the
  right graph even when the namemap cache is cold.

**Idempotency (the load-bearing migration detail):** the v64/v65 ladder *hard-gates* the version
bump on migration success, and `ALTER TABLE ADD COLUMN` errors on a duplicate column — so a naive
re-run would wedge the schema. v66 probes `PRAGMA table_info(code_projects)` per column
(`cp_column_exists`) and skips existing ones, returning SUCCESS when all three are present
regardless of who added them. The migration test opens a v65 DB and migrates **twice** to prove
this. Link-local rows store `source_url = ''` (SQLite can't drop NOT NULL without a table
rebuild; empty string is the honest "no remote") and branch all clone-only logic on `kind`.

### Delete-identifier fix + persisted `graph_name`

`code_project_delete` now resolves the cbm graph name from `p.graph_name` (fallback
`code_project_namemap_to_graph(p.name, …)`) and sends **that** under the `"project"` key. After
any index, cbm's slug is captured (matched by `local_path`) and written back via
`code_project_db_set_graph_name`. **Critical `kind` guard:** the working-tree removal
(`code_project_git_remove`) runs **only for `kind=='clone'`** — a `local` repo's tree is the
user's live checkout and is never touched; delete only removes the registration + the cbm graph.

**Thread discipline:** `code_project_delete` runs on the lws service thread (which carries audio);
the post-delete namemap re-capture makes a blocking cbm call, so it is routed through the worker,
never run on the caller thread. Same rule for every graph-mutating op.

### Multi-project namemap (`code_project_namemap.c`)

Phase 1 assumed one project under one `source_root` and translated a single prefix. Link-local
repos live anywhere, so the namemap was rebuilt as a **per-project map** —
`{ clean[64], graph[256], path[512] }[CODE_PROJECTS_MAX]` under `s_mtx`:

- **`capture()`** iterates DAWN rows (`code_project_db_list_all`) and matches each to cbm
  `list_projects` by **exact `root_path == local_path`** (the old `source_root` prefix filter is
  gone). `graph_name` reconcile rule, **DB authoritative for routing**: empty → fill from the live
  list; present-but-different (cbm re-slugged) → live corrects + write-back; absent from the live
  list → **keep** the DB value (so delete can still target it), never null it.
- **`to_graph(clean)`** maps the leading token (before the first `.`) of a `qualified_name`
  clean→slug and leaves the remainder verbatim; the `project` arg maps whole. cbm's
  `qualified_name` is `<slug>.<dotted path>` and the **first `.`** always separates them (DAWN
  names can't contain `.`), so the leading token is unambiguous. `file_path` values are
  repo-relative and need no translation; only absolute `local_path` is scrubbed.
- **`scrub(result)`** replaces every known slug→clean and strips every known `local_path` (plain +
  JSON-escaped), **longest-needle-first**. This is up to ~4×N passes — fine at the dev's N; a
  comment names the trigger ("if linked-project count crosses ~20, switch to single-pass
  multi-needle").
- **Lock invariants:** only `capture()` touches `code_project_db`; `to_graph`/`scrub` are DB-free
  (hot path). `capture()` reads rows under `s_db.mutex`, **releases it**, *then* calls cbm
  `list_projects`, *then* takes `s_mtx` to publish — never a cbm network call under either lock.

### Git helpers (`code_project_git.c`, clone kind only)

- `run_post_checkout_sweep()` (extracted) — symlink strip + depth/size/count caps, shared by clone
  and fetch.
- `code_project_git_fetch_checkout(path, branch, depth)` — open → `origin` fetch with the clone
  hardening reused (`GIT_REMOTE_REDIRECT_NONE`, timeouts, **`fo.depth = clone_depth`** so a
  shallow clone doesn't unshallow) → revparse `refs/remotes/origin/<branch>` → create local branch
  + set upstream if absent → `checkout_tree(FORCE)` → `set_head` → `reset(HARD)` → sweep.
  (Order matches libgit2 best practice: checkout_tree *then* set_head.)
- `code_project_git_current_branch(path, …)` — head shorthand / short SHA / `(detached)`.
- `code_project_git_open_validate(path)` — `git_repository_open_ext(NO_SEARCH)`; **NO_SEARCH is
  essential** so linking a subdir doesn't silently attach to a parent `.git`.
- `code_project_git_branch_valid()` — libgit2 ref-name rules.

### Service ops + crash recovery (`code_project_service.c`)

New job ops `CP_JOB_REBUILD`, `CP_JOB_LINK`; `cp_job_t` gains `branch`/`kind`/`local_path`.

- **refresh** (incremental): clone → `fetch_checkout(branch)` then index; local → update stored
  branch from `current_branch()` then index. **Never `fetch_checkout` a local project.**
- **rebuild** (clean): [clone → `fetch_checkout`] → resolve graph → `delete_project(graph)` →
  index → write-back `graph_name` + capture. On failure, clear status + `graph_name` so the next
  rebuild recovers.
- **set-branch** (clone kind only; rejected for local) → enqueue a **rebuild** (a branch switch
  changes the code). The surface response says "branch set; rebuilding (several minutes)" so it
  isn't perceived as hung.
- **link** = validation on the caller thread (below) → create `kind='local'`, `source_url=''`,
  `branch=current_branch()` → index.
- **Startup reconciliation:** `index_repository` is synchronous and the status poll is effectively
  fake (provider hardcodes READY). A crash mid-rebuild leaves a deleted graph + a row stuck in a
  transient state with no running job. `code_project_service_init` sweeps rows stuck in
  `indexing`/`cloning` and marks them `error: interrupted — rebuild to retry` — the only
  non-self-healing window, closed in ~15 lines.

### Link-local: the content-exposure boundary (highest risk)

Linking a local repo means **cbm reads its file contents and they reach the LLM**, regardless of
path-scrubbing. Controls:

- **`[code_projects] allowed_local_roots`** (config, list of absolute dir prefixes,
  `CODE_PROJECTS_MAX_LOCAL_ROOTS = 8`). Validation runs **on the caller thread before enqueue**
  (mirrors import's synchronous checks): `realpath()` → is-dir → containment within an allowed
  root (both realpath'd, `/`-terminated) → `open_validate` (git repo, NO_SEARCH). The worker
  trusts the resolved absolute path and never re-derives it.
- **Admin-only** in v1; **`is_global` forbidden on `kind='local'`** (a global linked repo would
  expose an admin-chosen tree to every user) — owner-visible only.
- **Sandbox unit change (the single most error-prone step):** `ReadOnlyPaths=/home/...` does
  **not** work because `ProtectHome=true` re-hides `/home`. The correct idiom is
  **`ProtectHome=tmpfs` + `BindReadOnlyPaths=<root>:<root>`** per allowed root in
  `services/cbm-mcp/cbm-mcp.service`, plus a traverse-only ACL (`setfacl -m u:dawn:--x /home/you`)
  so the `dawn` service user can reach the tree. DAWN can't apply this (no process management) —
  it's an operator step, and `allowed_local_roots` must stay in sync with the unit's bind paths.
  Link-local is non-functional without it, so the config + validation + sandbox change shipped as
  one atomic deliverable.
- **Symlink backstop:** DAWN validates only the linked repo's *root*, but cbm's discovery walk
  **skips symlinks** (`lstat` + `S_ISLNK`), so a symlink inside the tree is never followed — its
  target's contents don't reach the LLM even if it points outside the allowed root.
  `BindReadOnlyPaths` is the second backstop. Keep secret-bearing trees out of the allowlist
  regardless.

### Surfaces

- **Admin opcodes moved to their own band 0xD0–0xD6** (`CODE_PROJ_*`): MCP bridge keeps 0xB0–0xB4,
  OTA owns 0xCx. Handlers in `admin_socket_code_project.c`; by-name ops resolve the row id and
  tail-call the `*_by_id` core (one mutation path per op).
- **dawn-admin:** `code-project rebuild|link|set-branch` + `import --branch`; `list` gains
  KIND/BRANCH columns.
- **WebUI** (`webui_code_projects.c` + `code-projects.js` + `code-projects.css`): segmented
  **Import URL / Link local** toggle (Link admin-only), per-project **↻ refresh / ♻ rebuild /
  ⛖ set-branch / × delete** actions, `local`/`global` tag, tracked-branch display. Admins see all
  projects (incl. operator/CLI imports); regular users see their own + global. `graph_name` (the
  internal slug) is never sent to the browser.

---

## Sharing the cbm code graph with Claude Code

The same cbm server can back both Friday and a developer-side coding assistant (Claude Code,
Codex, etc.) so they share one graph store. Point the assistant's MCP client at the same SSE
endpoint:

```
http://localhost:9750/sse
```

Both clients then see every project in `CBM_CACHE_DIR` (`/var/lib/dawn/cbm-cache`).

**Is sharing one server safe? Yes — with one caveat (latency, not correctness).** Verified against
cbm source:

- cbm's query tools **require** a `project` argument and re-select the on-disk store from it on
  every call (`resolve_store`, no implicit fallback). The per-process "current project" is just a
  one-slot cache, so two clients interleaving calls for different projects do **not**
  cross-contaminate.
- cbm is a single stdio process behind `mcp-proxy`; requests serialize over the one pipe.
  `index_repository` is **synchronous and slow** (minutes on a large repo), so an index started by
  either side **head-of-line-blocks the other's queries** until it finishes. Avoid indexing while
  the other side is mid-conversation.
- The graph store is SQLite (WAL); concurrent reads + a single writer are safe.
- DAWN stays isolated from the assistant's own projects: DAWN's project list comes from its own
  database, and the namemap only maps cbm projects whose `root_path` matches a DAWN row. Projects
  the assistant indexes elsewhere are simply ignored.

If concurrent-indexing latency becomes a problem, run a **second** `mcp-proxy` + cbm instance on a
different port pointed at the **same** `CBM_CACHE_DIR` (separate process, WAL-safe) and give the
assistant that port — both still see the same graphs without sharing the request pipe.

The read-access / sandbox requirements for repos outside `/var/lib/dawn/source` are identical to
the link-local section above (`ProtectHome=tmpfs` + `BindReadOnlyPaths`, kept in sync with
`allowed_local_roots`).

---

## Deferred / future (not shipped)

- **`styledPrompt`** (themed replacement for native `prompt`/`confirm`): set-branch still uses
  `window.prompt`. Deferred to the Odysseus Tier-0 batch where the shared primitive is built once
  for all native-dialog sites.
- **Top-level installer integration:** the cbm-mcp sandbox grant is automated in
  `services/cbm-mcp/install.sh --local-roots`, but wiring it fully into the top-level DAWN
  installer (one end-to-end step) is still possible polish.
- **No-cbm local provider** (the Phase 1 "No-cbm minimal path" sketch): a second
  `code_graph_provider` impl that greps/reads the cloned tree in-process so the feature is useful
  without cbm. Removes the "clone-then-error" dead end.
- **Auth / write-side** (the Phase 1 "Phase 2 & 3 sketches"): PAT/OAuth for private repos, libgit2
  commit/push/branch-create, WebSocket transport, per-tool MCP capability overrides, project
  detail view.
- **Store scheduler `deliver_to` / project refs as ids** rather than names to retire the rename
  cascade (would need a schema migration).
