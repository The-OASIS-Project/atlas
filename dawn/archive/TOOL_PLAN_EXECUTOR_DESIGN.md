# Tool Plan Executor Design

**Created:** 2026-02-22
**Reviewed:** 2026-02-22 (architecture-reviewer, embedded-efficiency-reviewer, security-auditor)
**Implemented:** 2026-03-18

## Overview

A client-side tool orchestration system that allows LLMs to emit structured multi-step "tool plans" instead of requiring multiple round trips for complex workflows. The plan executor interprets a constrained JSON DSL, executing tools locally with conditional logic, variable binding, and loop support — all without an external scripting runtime.

This is inspired by Anthropic's "Programmatic Tool Calling" concept but implemented as a **client-side JSON plan executor** rather than server-side Python execution. This approach is:

- **LLM-agnostic** — works with OpenAI, Claude, Ollama, llama.cpp, Gemini
- **Zero new dependencies** — pure C, no embedded interpreter
- **Safe** — interprets a constrained data structure, never executes arbitrary code
- **Fast** — tools execute locally with no API round trips between steps
- **Token-efficient** — intermediate results stay local; only the final summary goes to the LLM

## Motivation

### Current Flow (Multi-Tool)

```
User: "Do I have any alarms tomorrow? If not, set one for 7am"

Round 1: LLM → tool_use: scheduler(action=query) → execute → result → LLM
Round 2: LLM → tool_use: scheduler(action=create, ...) → execute → result → LLM
Round 3: LLM → final response to user
```

**Cost:** 3 LLM round trips, full context window consumed each time, ~3-6 seconds of latency.

### Proposed Flow (Plan Executor)

```
User: "Do I have any alarms tomorrow? If not, set one for 7am"

Round 1: LLM → tool_use: execute_plan(plan=[...]) → plan executor runs locally → result
Round 2: LLM → final response to user (or skip_followup with direct result)
```

**Cost:** 1 LLM round trip for plan generation, local execution, 1 optional follow-up. ~1-2 seconds.

## Plan DSL Specification

### Plan Structure

A plan is a JSON array of **steps**. Each step is an object with a `type` field:

```json
{
  "plan": [
    { "type": "call", ... },
    { "type": "if", ... },
    { "type": "loop", ... },
    { "type": "set", ... },
    { "type": "log", ... }
  ]
}
```

### Step Types

#### 1. `call` — Execute a Tool

```json
{
  "type": "call",
  "tool": "scheduler",
  "args": {
    "action": "query",
    "type": "alarm"
  },
  "store": "alarms"
}
```

- `tool` (required): Tool name — must be an exact registered name (not alias or device_string)
- `args` (required): Tool arguments object — values support `$var` interpolation
- `store` (optional): Variable name to store the result string

#### 2. `if` — Conditional Execution

```json
{
  "type": "if",
  "condition": "alarms.empty",
  "then": [
    { "type": "call", "tool": "scheduler", "args": {"action": "create", "type": "alarm", "time": "7:00 AM"} }
  ],
  "else": [
    { "type": "log", "message": "Alarms already exist: $alarms" }
  ]
}
```

- `condition` (required): Expression string (see Condition Expressions below)
- `then` (required): Array of steps to execute if true
- `else` (optional): Array of steps to execute if false

#### 3. `loop` — Iterate Over Items

```json
{
  "type": "loop",
  "over": ["kitchen", "living room", "bedroom"],
  "as": "room",
  "steps": [
    { "type": "call", "tool": "home_assistant", "args": {"action": "off", "entity": "$room light"} }
  ]
}
```

- `over` (required): JSON array of string values to iterate (max `PLAN_MAX_LOOP_ITERATIONS` items)
- `as` (required): Variable name for current item
- `steps` (required): Array of steps to execute per iteration
- The `over` array length is capped at `PLAN_MAX_LOOP_ITERATIONS` (10) server-side. Any LLM-provided `max_iterations` field is ignored.

#### 4. `set` — Assign a Variable

```json
{
  "type": "set",
  "var": "greeting",
  "value": "Hello from $room"
}
```

- `var` (required): Variable name (must match `^[a-z_][a-z0-9_]{0,31}$`)
- `value` (required): String value (supports `$var` interpolation)

#### 5. `log` — Append to Output

```json
{
  "type": "log",
  "message": "Found $alarm_count alarms for tomorrow"
}
```

- `message` (required): Text to append to the plan's output buffer (supports interpolation)
- The accumulated log output becomes the plan's result returned to the LLM
- Output is bounded: if the buffer would overflow, the message is truncated with a `[truncated]` marker

#### 6. `sleep` — Pause Execution

```json
{
  "type": "sleep",
  "seconds": 30
}
```

- `seconds` (required): Duration to pause in seconds (range 1-300, capped at `PLAN_MAX_SLEEP_S`)
- Sleeps in 1-second intervals, checking the plan timeout between each
- If the plan timeout expires during sleep, the plan aborts with a timeout error
- Use cases: rate-limiting API calls, waiting between device state changes, timed sequences

### Condition Expressions

Conditions are simple string expressions evaluated against the variable store. Keep the expression language minimal — the LLM can pre-compute complex logic by choosing which plan structure to emit.

| Expression | Meaning |
|-----------|---------|
| `varname.empty` | Variable is unset, NULL, empty string, or `"[]"` / `"none"` / `"no results"` |
| `varname.notempty` | Negation of `.empty` |
| `varname.contains:text` | Variable string contains "text" (case-insensitive) |
| `varname.equals:text` | Variable string equals "text" (case-insensitive) |
| `varname.success` | Last tool call stored in `varname` returned success |
| `varname.failed` | Last tool call stored in `varname` returned failure |
| `true` | Always true (useful for unconditional `else`-like blocks) |
| `false` | Always false |

**Why not a full expression parser?** Simplicity and security. A minimal condition set is:
- Easy to implement (~50 lines of C)
- Impossible to exploit (no eval, no injection surface)
- Sufficient for 90%+ of multi-tool workflows
- If the LLM needs complex logic, it can structure the plan differently (nested ifs, pre-computed values)

### Variable Interpolation

Variables are stored as strings in a simple key-value map. Interpolation uses `$varname` or `${varname}` syntax:

```json
{"type": "call", "tool": "music", "args": {"action": "play", "query": "$search_result"}}
```

**Rules:**
- `$varname` — expands to the stored string value
- `${varname}` — same, but allows adjacent text: `"${room}_light"`
- Undefined variables expand to empty string
- No nested interpolation (no `$$var`)
- Maximum interpolated string length: `LLM_TOOLS_RESULT_LEN` (8192 bytes) — truncated if exceeded
- Interpolation happens on JSON string values, not on raw JSON text (see Safe Argument Construction below)

### Variable Name Validation

All variable names (`store` in `call`, `var` in `set`, `as` in `loop`) are validated:
- Pattern: `^[a-z_][a-z0-9_]{0,31}$` (lowercase alphanumeric + underscore, max 32 chars)
- Must not contain `$`, `.`, `{`, `}` (prevents recursive interpolation and condition confusion)
- Invalid names are rejected at parse time with an error

## Integration with Existing Tool System

### Registration as a Meta-Tool

The plan executor registers as a regular tool in the tool registry:

```c
static const treg_param_t plan_params[] = {
   {.name = "plan",
    .type = TOOL_PARAM_TYPE_STRING,
    .required = true,
    .description = "JSON array of plan steps to execute sequentially. "
                   "Each step is an object with 'type' field: "
                   "'call' (execute tool), 'if' (conditional), "
                   "'loop' (iterate), 'set' (variable), 'log' (output)."},
};

static const tool_metadata_t plan_executor_metadata = {
   .name = "execute_plan",
   .device_string = "plan executor",
   .description =
      "Execute a multi-step tool plan locally. Use this when a task "
      "requires multiple tool calls with conditional logic or data "
      "dependencies between steps. The plan runs entirely on the server "
      "without additional LLM round trips. Returns aggregated results.",
   .params = plan_params,
   .param_count = 1,
   .device_type = TOOL_DEVICE_TYPE_GETTER,
   .capabilities = 0,
   .is_getter = true,
   .callback = plan_executor_callback,
};
```

**Note on the `plan` parameter type:** The plan is declared as `TOOL_PARAM_TYPE_STRING` carrying a JSON array. This means double-encoding (JSON in a JSON string). The parser handles both forms: if the incoming value starts with `[`, parse directly as JSON array; otherwise treat as an escaped JSON string. Different LLM providers may emit either form.

### Execution Flow

```
LLM emits tool_use: execute_plan(plan=[...])
  ↓
plan_executor_callback() called by llm_tools_execute()
  ↓
Validate plan JSON size (≤ PLAN_MAX_PLAN_SIZE)
  ↓
Parse JSON plan array
  ↓
For each step (incrementing total_steps_executed on every execution, including nested):
  ├─ call:  validate tool → build tool_call_t → llm_tools_execute() → store result
  ├─ if:    evaluate condition → execute then/else branch (recurse)
  ├─ loop:  iterate array (capped) → execute steps per item (recurse)
  ├─ set:   validate name → store variable
  └─ log:   interpolate → bounded append to output buffer
  ↓
Return accumulated output string to LLM (via tool result)
```

### Tool Execution Within Plans — Delegating to `llm_tools_execute()`

Plan `call` steps **must** route through `llm_tools_execute()` rather than calling tool callbacks directly. This ensures all existing dispatch logic is preserved:

- Parameter mapping (`TOOL_MAPS_TO_ACTION`, `TOOL_MAPS_TO_VALUE`, `TOOL_MAPS_TO_DEVICE`, `TOOL_MAPS_TO_CUSTOM`)
- Meta-tool device resolution via `tool_registry_resolve_device()`
- MQTT-only tool dispatch via `command_execute()`
- Sync-wait handling (e.g., viewing tool)
- Execution notifications via `notify_tool_execution()` (feeds WebUI debug display)
- Session context inheritance (plan runs on calling thread, inherits session context)

```c
// Inside plan_step_call():

// 1. Validate tool name (exact match only — no aliases or device_strings)
const tool_metadata_t *meta = tool_registry_lookup(step->tool_name);
if (!meta) {
   snprintf(ctx->error, sizeof(ctx->error), "Unknown tool: %s", step->tool_name);
   return PLAN_ERR_UNKNOWN_TOOL;
}

// 2. Check capability whitelist (mirrors scheduler pattern)
if (!plan_tool_is_allowed(meta)) {
   snprintf(ctx->error, sizeof(ctx->error), "Tool '%s' not allowed in plans", step->tool_name);
   return PLAN_ERR_TOOL_NOT_ALLOWED;
}

// 3. Check runtime enabled state
if (!tool_registry_is_enabled(step->tool_name)) {
   snprintf(ctx->error, sizeof(ctx->error), "Tool '%s' is disabled", step->tool_name);
   return PLAN_ERR_TOOL_DISABLED;
}

// 4. Build tool_call_t with interpolated args as JSON string
tool_call_t call = {0};
snprintf(call.id, sizeof(call.id), "plan_%d", ctx->total_steps_executed);
safe_strncpy(call.name, step->tool_name, sizeof(call.name));
plan_build_args_json(ctx, step_args_obj, call.arguments, sizeof(call.arguments));

// 5. Execute through existing dispatch (handles MQTT, device mapping, etc.)
tool_result_t result = {0};
int rc = llm_tools_execute(&call, &result);

// 6. Store result in variable if requested
if (step->store_var) {
   plan_vars_set(ctx, step->store_var, result.result);
   plan_vars_set_success(ctx, step->store_var, result.success);
}
```

### Safe Argument Construction

Variable interpolation into tool arguments is done at the **JSON object level**, not via string substitution into raw JSON text. This prevents injection of JSON control characters or the `::field_name::value` delimiter pattern used by `llm_tools_execute_from_treg()`:

```c
// plan_build_args_json():
// 1. Walk the step's args json_object object
// 2. For each string value, run plan_interpolate() to expand $vars
// 3. Build a NEW json_object object with the interpolated string values
// 4. Serialize the clean json_object object to a JSON string via json_object_to_json_string_ext()
// 5. Copy into call.arguments buffer

// This ensures interpolated values are always properly JSON-escaped
// and cannot inject :: delimiters or break JSON structure
```

### Tool Capability Whitelist

Following the scheduler's established pattern (`scheduler.c:294-311`), the plan executor uses a **positive capability whitelist** — only tools explicitly marked safe can be called from plans:

```c
static bool plan_tool_is_allowed(const tool_metadata_t *meta) {
   // Block self-reference
   if (strcmp(meta->name, "execute_plan") == 0) return false;

   // Must have SCHEDULABLE capability (same whitelist as scheduler)
   if (!(meta->capabilities & TOOL_CAP_SCHEDULABLE)) return false;

   // Double-check: never allow DANGEROUS tools
   if (meta->capabilities & TOOL_CAP_DANGEROUS) return false;

   return true;
}
```

**Rationale for reusing `TOOL_CAP_SCHEDULABLE`:** The safety requirements are nearly identical — both the scheduler and plan executor run tool callbacks without interactive user confirmation. Tools already marked schedulable have been vetted for unattended execution. Adding a separate `TOOL_CAP_PLAN_SAFE` flag would require touching every tool to set it, for no additional safety benefit.

**Tools currently marked `TOOL_CAP_SCHEDULABLE`:** These are the only tools callable from plans. New tools must explicitly opt in by adding this capability flag.

### Error Handling — Fail-Forward Semantics

Plans use **fail-forward** execution: a failed `call` step does not abort the plan.

- On `call` failure: the error message is stored in the variable, `success` is set to `false`, execution continues
- The plan can check `.failed` / `.success` conditions to handle errors gracefully
- **Only safety limit violations abort the plan:** max steps exceeded, max depth exceeded, timeout, unknown tool, disabled tool, or tool not allowed

This allows robust plans like:
```json
{"type": "call", "tool": "weather", "args": {"action": "forecast"}, "store": "w"},
{"type": "if", "condition": "w.failed", "then": [
  {"type": "log", "message": "Weather unavailable, skipping rain check"}
], "else": [
  {"type": "log", "message": "Forecast: $w"}
]}
```

### Safety Controls

| Control | Value | Notes |
|---------|-------|-------|
| Max total step executions | 50 (`PLAN_MAX_STEPS`) | Counts every execution including loop iterations and nested branches |
| Max tool calls per plan | 20 (`PLAN_MAX_TOOL_CALLS`) | Separate counter, only `call` steps |
| Max loop iterations | 10 (`PLAN_MAX_LOOP_ITERATIONS`) | Server-enforced, LLM-provided values ignored |
| Max nested depth | 5 (`PLAN_MAX_DEPTH`, if/loop nesting) | Enforced at **parse time** via `validate_step_depth()` |
| Max variables | 32 (`PLAN_MAX_VARS`) | |
| Max output size | 8192 bytes (`LLM_TOOLS_RESULT_LEN`) | Truncated with marker on overflow |
| Max plan JSON size | 16384 bytes | Validated before `json_tokener_parse()` |
| Max interpolated string | 8192 bytes (`LLM_TOOLS_RESULT_LEN`) | Pre-calculated, truncated if exceeded |
| Tool whitelist | `TOOL_CAP_SCHEDULABLE` required | Same as scheduler — positive opt-in |
| Dangerous tool block | `TOOL_CAP_DANGEROUS` rejected | Double-check on top of whitelist |
| Self-reference block | `execute_plan` cannot call itself | |
| Enabled check | `tool_registry_is_enabled()` | Respects runtime enable/disable |
| Name-only lookup | `tool_registry_lookup()` | No alias/device_string resolution — prevents bypass |
| Execution timeout | 30 seconds total | Configurable via `[llm.tools] plan_timeout` |
| Concurrency | Sequential only + mutex | Never runs in parallel with other tools |

### Audit Logging

Every `call` step is logged via the existing `notify_tool_execution()` callback, which feeds the WebUI debug display. This happens automatically because `call` steps delegate to `llm_tools_execute()`.

Additionally:
- Plan start: `LOG_INFO("plan_executor: executing plan (%d steps)", step_count)`
- Plan complete: `LOG_INFO("plan_executor: completed in %ldms (%d tool calls, %d steps)", ...)`
- Plan error: `LOG_WARNING("plan_executor: aborted at step %d: %s", step_num, error)`
- Full plan JSON logged at `LOG_DEBUG` level when execution begins

## System Prompt Addition

The plan DSL description is added to the system prompt **only when 3+ tools are enabled** (with fewer tools, multi-step plans are unlikely to be useful and the ~400 tokens of prompt overhead aren't justified, especially for local LLMs with small context windows):

```
## Multi-Step Tool Plans

When a task requires multiple tool calls, especially with conditions or dependencies
between results, use the `execute_plan` tool instead of individual tool calls.

Plan format: JSON array of steps.
Step types: call (execute tool), if (conditional), loop (iterate), set (variable), log (output), sleep (pause N seconds).

Example - check and conditionally create:
{
  "plan": [
    {"type": "call", "tool": "scheduler", "args": {"action": "query", "type": "alarm"}, "store": "alarms"},
    {"type": "if", "condition": "alarms.empty", "then": [
      {"type": "call", "tool": "scheduler", "args": {"action": "create", "type": "alarm", "time": "7:00 AM"}, "store": "result"},
      {"type": "log", "message": "Created alarm: $result"}
    ], "else": [
      {"type": "log", "message": "Existing alarms: $alarms"}
    ]}
  ]
}

Example - batch operations:
{
  "plan": [
    {"type": "loop", "over": ["kitchen", "living room", "bedroom"], "as": "room", "steps": [
      {"type": "call", "tool": "home_assistant", "args": {"action": "off", "entity": "$room light"}}
    ]},
    {"type": "log", "message": "All lights turned off"}
  ]
}

Use execute_plan when:
- A task needs 2+ tool calls with data dependencies
- You need to check a result before deciding the next action
- You need to perform the same action on multiple items
- Intermediate results don't need LLM reasoning

Use individual tool calls when:
- Only one tool call is needed
- You need to reason about intermediate results
- The task involves dangerous/destructive operations
```

## Example Scenarios

### Scenario 1: Conditional Alarm

**User:** "Set an alarm for 7am tomorrow if I don't already have one"

```json
{
  "plan": [
    {"type": "call", "tool": "scheduler", "args": {"action": "query", "type": "alarm"}, "store": "alarms"},
    {"type": "if", "condition": "alarms.empty", "then": [
      {"type": "call", "tool": "scheduler", "args": {"action": "create", "type": "alarm", "time": "7:00 AM tomorrow"}, "store": "result"},
      {"type": "log", "message": "Created: $result"}
    ], "else": [
      {"type": "log", "message": "You already have alarms set: $alarms"}
    ]}
  ]
}
```

### Scenario 2: Goodnight Routine

**User:** "Goodnight" (with configured routine)

```json
{
  "plan": [
    {"type": "loop", "over": ["kitchen", "living room", "office", "porch"], "as": "room", "steps": [
      {"type": "call", "tool": "home_assistant", "args": {"action": "off", "entity": "$room light"}}
    ]},
    {"type": "call", "tool": "home_assistant", "args": {"action": "lock", "entity": "front door"}},
    {"type": "call", "tool": "home_assistant", "args": {"action": "temperature", "entity": "thermostat", "setting": "68"}},
    {"type": "call", "tool": "music", "args": {"action": "stop"}},
    {"type": "log", "message": "Goodnight! Lights off, door locked, thermostat set to 68."}
  ]
}
```

### Scenario 3: Music Search and Queue

**User:** "Find some jazz songs and queue the first 3"

```json
{
  "plan": [
    {"type": "call", "tool": "music", "args": {"action": "search", "query": "jazz"}, "store": "results"},
    {"type": "if", "condition": "results.notempty", "then": [
      {"type": "call", "tool": "music", "args": {"action": "add_to_queue", "query": "jazz"}, "store": "queued"},
      {"type": "log", "message": "Queued jazz tracks: $queued"}
    ], "else": [
      {"type": "log", "message": "No jazz tracks found in the library."}
    ]}
  ]
}
```

### Scenario 4: Weather-Based Reminder

**User:** "If it's going to rain tomorrow, remind me to bring an umbrella at 8am"

```json
{
  "plan": [
    {"type": "call", "tool": "weather", "args": {"action": "forecast", "query": "tomorrow"}, "store": "forecast"},
    {"type": "if", "condition": "forecast.contains:rain", "then": [
      {"type": "call", "tool": "scheduler", "args": {"action": "create", "type": "reminder", "time": "8:00 AM tomorrow", "message": "Bring an umbrella - rain expected"}, "store": "result"},
      {"type": "log", "message": "Rain expected tomorrow. $result"}
    ], "else": [
      {"type": "log", "message": "No rain expected tomorrow. No reminder set."}
    ]}
  ]
}
```

## Implementation

### Data Structures

```c
#define PLAN_MAX_STEPS           50
#define PLAN_MAX_TOOL_CALLS      20
#define PLAN_MAX_LOOP_ITERATIONS 10
#define PLAN_MAX_VARS            32
#define PLAN_MAX_DEPTH           5
#define PLAN_MAX_PLAN_SIZE       16384
#define PLAN_VAR_NAME_MAX        32
#define PLAN_TIMEOUT_DEFAULT_S   30

/* Variable store — heap-allocated values for memory efficiency */
typedef struct {
   char name[PLAN_VAR_NAME_MAX + 1];
   char *value;    /* heap-allocated via strdup(), freed on overwrite/cleanup */
   bool success;   /* whether the tool call that set this var succeeded */
} plan_var_t;

typedef struct {
   plan_var_t vars[PLAN_MAX_VARS];
   int var_count;
   char output[LLM_TOOLS_RESULT_LEN];  /* accumulated log output */
   int output_len;
   int depth;                           /* current nesting depth */
   int total_steps_executed;            /* counts ALL executions including nested */
   int total_tool_calls;                /* counts only 'call' steps */
   char error[256];                     /* last error message */
   struct timespec start_time;          /* for timeout */
   int timeout_s;                       /* configurable timeout */
} plan_context_t;
```

**Memory note:** With heap-allocated variable values, `plan_context_t` is ~3KB base (vs. ~272KB with fixed 8KB buffers). This is safe for stack allocation on worker threads (512KB stacks) and improves cache locality during variable lookups — iterating the vars array only touches 33-byte name fields instead of striding across 8KB value buffers.

### Key Functions

**Public API** (declared in `plan_executor.h`):

```c
/* Parse and validate a plan (checks JSON size, structure, variable names, depth) */
int plan_parse(const char *json, struct json_object **out);

/* Execute a plan step array (recursive for if/loop bodies) */
int plan_execute_steps(plan_context_t *ctx, struct json_object *steps);

/* Condition evaluation */
bool plan_eval_condition(const plan_context_t *ctx, const char *expr);

/* Variable operations */
void plan_vars_set(plan_context_t *ctx, const char *name, const char *value);
const char *plan_vars_get(const plan_context_t *ctx, const char *name);
void plan_vars_set_success(plan_context_t *ctx, const char *name, bool success);
bool plan_validate_var_name(const char *name);

/* Argument construction — builds json-c args with interpolation, serializes safely */
int plan_build_args_json(plan_context_t *ctx, struct json_object *args,
                         char *out_json, size_t out_size);

/* Interpolation with size limit */
int plan_interpolate(const plan_context_t *ctx, const char *template_str,
                     char *out, size_t out_size);

/* Cleanup — frees all heap-allocated variable values */
void plan_context_cleanup(plan_context_t *ctx);

/* Tool capability check */
bool plan_tool_is_allowed(const tool_metadata_t *meta);

/* Tool registration (called from tools_init.c) */
int plan_executor_tool_register(void);
```

**Internal functions** (static in `plan_executor.c`):

```c
/* Individual step executors — not public API */
static int plan_step_call(plan_context_t *ctx, struct json_object *step);
static int plan_step_if(plan_context_t *ctx, struct json_object *step);
static int plan_step_loop(plan_context_t *ctx, struct json_object *step);
static int plan_step_set(plan_context_t *ctx, struct json_object *step);
static int plan_step_log(plan_context_t *ctx, struct json_object *step);

/* Depth-limited recursive validation (called at parse time) */
static int validate_step_depth(struct json_object *step, int depth);
static int validate_step_array_depth(struct json_object *steps, int depth);

/* json-c accessor helpers */
static const char *jobj_get_string(struct json_object *obj, const char *key);
static struct json_object *jobj_get(struct json_object *obj, const char *key);
```

### Files Created

| File | Purpose | Lines |
|------|---------|-------|
| `include/tools/plan_executor.h` | Public API, constants, data structures | 259 |
| `src/tools/plan_executor.c` | Plan parser, executor, condition evaluator, arg builder | ~580 |
| `src/tools/plan_executor_tool.c` | Tool registry metadata and callback wrapper | ~70 |
| `tests/test_plan_executor.c` | Unit tests (130 assertions, 30 test functions) | ~850 |
| `tests/test_plan_executor_stub.c` | Stubs for tool_registry and llm_tools | ~30 |
| `tests/PLAN_EXECUTOR_TEST_PROMPTS.md` | 15 LLM prompt test scenarios (P1-P15) | ~200 |

### Files Modified

| File | Changes |
|------|---------|
| `src/tools/tools_init.c` | Register `plan_executor_tool_register()` (unconditional, no ifdef) |
| `src/llm/llm_tools.c` | Add `"execute_plan"` to `SEQUENTIAL_TOOLS` array |
| `src/llm/llm_command_parser.c` | Add `PLAN_EXECUTOR_PROMPT` (~400 tokens), conditional on 3+ tools |
| `cmake/DawnTools.cmake` | Add `plan_executor.c` and `plan_executor_tool.c` to `TOOL_REGISTRY_SOURCES` |
| `tests/CMakeLists.txt` | Add `test_plan_executor` target and include in aggregate `tests` target |

## Implementation Phases

### Phase 1: Core Executor — COMPLETE

- [x] `plan_executor.h` — data structures, constants, API
- [x] `plan_executor.c` — `call`, `set`, `log` step types
- [x] Variable store with heap-allocated values and `plan_context_cleanup()`
- [x] Variable name validation (`^[a-z_][a-z0-9_]{0,31}$`)
- [x] Safe argument construction via json-c object building (not string interpolation)
- [x] Interpolation with pre-calculated size and `LLM_TOOLS_RESULT_LEN` cap
- [x] Condition evaluator (`.empty`, `.notempty`, `.contains`, `.equals`, `.success`, `.failed`)
- [x] Tool dispatch via `llm_tools_execute()` (not raw callbacks)
- [x] `TOOL_CAP_SCHEDULABLE` whitelist + `TOOL_CAP_DANGEROUS` block + `is_enabled` check
- [x] `tool_registry_lookup()` for name-only resolution (no alias/device_string)
- [x] Plan JSON size validation before `json_tokener_parse()`
- [x] Safety limits (max steps, max tool calls, max output size, timeout)
- [x] Fail-forward error semantics
- [x] Audit logging (start, complete, error, per-call via `notify_tool_execution()`)
- [x] `plan_executor_tool.c` — tool registry registration
- [x] Register in `tools_init.c`
- [x] Add `"execute_plan"` to `SEQUENTIAL_TOOLS` in `llm_tools.c`

### Phase 2: Control Flow — COMPLETE

- [x] `if` step type with `then`/`else` branches
- [x] `loop` step type with server-enforced iteration cap
- [x] Nesting depth tracking and enforcement
- [x] `total_steps_executed` incrementing on every nested execution

### Phase 3: Integration and Prompt — COMPLETE

- [x] Conditional system prompt inclusion (3+ tools enabled)
- [x] Handle both JSON string and JSON object forms for plan parameter
- [ ] Test with all LLM providers (Claude, OpenAI, local) — manual testing pending
- [x] Configurable timeout via `[plan_executor] timeout_seconds` in dawn.toml (range 5-300, default 60)
- [x] Sleep step: `{"type": "sleep", "seconds": N}` — pauses 1-300s, checks timeout each second

### Phase 4: Testing — COMPLETE

- [x] Unit test: plan parsing (valid/invalid JSON, oversized plans)
- [x] Unit test: condition evaluation
- [x] Unit test: variable interpolation (including size limits, edge cases)
- [x] Unit test: variable name validation (valid/invalid patterns)
- [x] Unit test: safety limits (max steps, depth, tool calls, timeout)
- [x] Unit test: tool capability checks (disabled tools, dangerous tools, alias bypass attempts)
- [x] Unit test: fail-forward behavior (failed call → `.failed` condition)
- [x] Unit test: output buffer overflow handling
- [ ] Integration test: multi-tool plans with real tool callbacks — manual testing pending
- [ ] Test with each LLM provider to verify plan generation quality — manual testing pending

### Test Results

```
140 assertions across 30+ test functions — ALL PASSED

Test coverage:
- Variable name validation (valid patterns, invalid patterns, edge cases)
- Variable store (set, get, overwrite, missing, cleanup)
- Condition evaluation (empty, notempty, contains, equals, success, failed, literals)
- Interpolation ($var, ${var}, undefined vars, adjacent text)
- Plan parsing (valid plans, invalid JSON, oversized plans, bad variable names)
- Execution (set+log, if-then, if-else, loop, nested if-in-loop)
- Safety limits (max steps, max depth, max tool calls)
- Tool capability whitelist (SCHEDULABLE required, DANGEROUS blocked, self-reference blocked)
- Argument construction safety (interpolation in args, JSON structure preservation)
```

Build: `make -C build-debug test_plan_executor && ./build-debug/tests/test_plan_executor`

## Future Enhancements (Not in v1)

| Enhancement | Description |
|------------|-------------|
| **Named plans** | Store reusable plans (e.g., "goodnight routine") that can be triggered by name |
| **Plan templates** | User-defined plan templates with parameter substitution |
| **Numeric expressions** | `varname.gt:5`, `varname.lt:100` for numeric comparisons |
| **JSON field access** | `varname.field.subfield` for structured result extraction |
| **Plan review UI** | WebUI displays the plan steps before execution for user approval |
| **Parallel call steps** | `{"type": "parallel", "calls": [...]}` — if implemented, must delegate to `llm_tools_execute_all()` to preserve parallel/sequential safety classification |
| **Lua fallback** | For plans that exceed the JSON DSL's expressiveness, optional Lua sandbox (~200KB) |

## Effort

| Phase | Estimate | Status |
|-------|----------|--------|
| Phase 1: Core executor (includes security controls) | ~2-3 days | COMPLETE |
| Phase 2: Control flow | ~1 day | COMPLETE |
| Phase 3: Integration | ~1 day | COMPLETE (config timeout shipped, sleep step added) |
| Phase 4: Testing | ~1 day | COMPLETE (140/140 unit tests pass) |
| **Total** | **~5-6 days** | **Implemented 2026-03-18** |

## Remaining Work

- [ ] Manual testing with Claude, OpenAI, and local LLM providers
- [x] Configurable timeout via `[plan_executor] timeout_seconds` — shipped
- [ ] Integration testing with real tool callbacks (scheduler, music, home_assistant)

## Review Findings Log

Design reviewed 2026-02-22 by architecture-reviewer, embedded-efficiency-reviewer, and security-auditor. Key changes incorporated:

1. **Route through `llm_tools_execute()`** not raw callbacks — preserves MQTT dispatch, parameter mapping, device resolution, execution notifications (Arch C1, Security C3)
2. **Positive capability whitelist** via `TOOL_CAP_SCHEDULABLE` — mirrors scheduler pattern, only vetted tools callable from plans (Security C1)
3. **Heap-allocated variable values** — reduces `plan_context_t` from ~272KB to ~3KB, eliminates stack overflow risk (Arch H1, Embedded C1)
4. **Safe argument construction** via json-c objects — prevents `::` delimiter injection and JSON structure corruption from interpolated values (Security C2)
5. **Name-only tool lookup** via `tool_registry_lookup()` — prevents alias-based capability bypass (Security H3)
6. **`tool_registry_is_enabled()` check** — prevents calling admin-disabled tools (Security C3)
7. **`SEQUENTIAL_TOOLS` from Phase 1** — prevents race conditions with concurrent tools (Arch H2, Security H2)
8. **Server-enforced loop caps** — `max_iterations` from LLM ignored, `over` array capped (Security H4)
9. **Separate tool call counter** (`PLAN_MAX_TOOL_CALLS = 20`) on top of step counter (Security H1)
10. **Step counter includes nested executions** — prevents nested loop explosion (Embedded M2, Security H1)
11. **Variable name validation** — `^[a-z_][a-z0-9_]{0,31}$` prevents recursive interpolation (Security M3)
12. **Bounded output buffer** with truncation marker on overflow (Embedded M3, Security M2)
13. **Interpolation size limits** — pre-calculated, capped at `LLM_TOOLS_RESULT_LEN` (Embedded H1)
14. **Fail-forward semantics** — failed calls don't abort plans, errors queryable via `.failed` (Arch M2)
15. **Plan JSON size validation** before `json_tokener_parse()` (Embedded H2)
16. **Conditional system prompt** — only added when 3+ tools enabled (Arch M3)
17. **Variable cleanup semantics** — `plan_vars_set()` frees old value, `plan_context_cleanup()` frees all (Embedded H3)
18. **Audit logging** via existing `notify_tool_execution()` path (Security L2)
19. **Configurable timeout** via `[llm.tools] plan_timeout` (Embedded L2)
20. **Handle both JSON forms** for plan parameter (Arch H3)

Post-implementation security hardening (2026-03-19):

21. **Parse-time depth validation** — `validate_step`/`validate_step_array` renamed to `validate_step_depth`/`validate_step_array_depth` with explicit `depth` parameter. Rejects plans exceeding `PLAN_MAX_DEPTH` at parse time before any execution begins, preventing stack overflow from malicious deeply-nested JSON

## References

- [Anthropic Programmatic Tool Calling](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/programmatic-tool-calling) — inspiration
- Current tool execution: `src/llm/llm_tools.c` (`llm_tools_execute()`, `llm_tools_execute_from_treg()`)
- Tool registry: `src/tools/tool_registry.c`, `include/tools/tool_registry.h`
- Scheduler safety pattern: `src/core/scheduler.c` (lines 291-311 — `TOOL_CAP_SCHEDULABLE` + `is_enabled` checks)
- Tool capability flags: `include/tools/tool_registry.h` (`tool_capability_t` enum)
- Sequential tools: `src/llm/llm_tools.c` (`SEQUENTIAL_TOOLS` array, line 141)
