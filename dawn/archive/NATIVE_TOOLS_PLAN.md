> **STATUS: COMPLETE** - Archived 2025-12-25

# Native Tool/Function Calling Implementation Plan

## Overview

Replace `<command>` tag parsing with native LLM tool calling for OpenAI, Claude, and local Qwen models. This will:
- **Reduce system prompt from ~7,700 chars to ~2,000 chars** (~70% reduction)
- **Improve reliability** - structured responses instead of text parsing
- **Enable parallel tool calls** - multiple actions in one response

## Design Decisions

- **Enablement**: Config option (`native_tools_enabled`), default OFF initially
- **Local LLM**: Already using `--jinja` flag - tool calling ready
- **Streaming**: Full streaming support from the start

## Provider Compatibility

| Provider | Support | Notes |
|----------|---------|-------|
| OpenAI | Native | `tools` parameter, `tool_calls` response |
| Claude | Native | `tools` parameter, `tool_use` content blocks |
| Local Qwen | Supported | llama.cpp with `--jinja` flag (already configured) |

---

## Phase 1: Tool Definition Infrastructure

### New Files

**`include/llm/llm_tools.h`**
```c
#define MAX_TOOLS 32

typedef struct {
   char id[64];           /* Tool call ID for response correlation */
   char name[64];         /* Tool name (maps to device type) */
   char arguments[4096];  /* JSON arguments string */
} tool_call_t;

typedef struct {
   tool_call_t calls[8];
   int count;
} tool_call_list_t;

typedef struct {
   char tool_call_id[64];
   char result[8192];
   bool success;
} tool_result_t;

/* Core API */
void llm_tools_init(void);
void llm_tools_refresh(void);
struct json_object *llm_tools_get_openai_format(void);
struct json_object *llm_tools_get_claude_format(void);
int llm_tools_execute(const tool_call_t *call, tool_result_t *result);
bool llm_tools_supported(void);
```

**`src/llm/llm_tools.c`**
- Define tool schemas for each deviceType
- Generate OpenAI/Claude JSON formats
- Bridge tool execution to existing deviceCallback system

### Tool Definitions (10 tools)

| Tool | Parameters | Description |
|------|------------|-------------|
| `weather` | action: today/tomorrow/week, location: string | Weather forecast |
| `search` | action: web/news/etc, query: string | Web search |
| `calculator` | action: evaluate/convert, expression: string | Math/conversions |
| `url` | url: string | Fetch webpage content |
| `smartthings` | action: list/on/off/etc, device?: string, value?: string | Smart home |
| `date` | (none) | Current date |
| `time` | (none) | Current time |
| `llm_status` | (none) | Current LLM info |
| `volume` | level: integer | Set volume |
| `music` | action: play/stop/next, query?: string | Music control |

---

## Phase 2: Modify LLM Providers

### `src/llm/llm_openai.c`

1. **Add tools to request:**
```c
json_object *tools = llm_tools_get_openai_format();
if (tools) {
   json_object_object_add(root, "tools", tools);
   json_object_object_add(root, "tool_choice", json_object_new_string("auto"));
}
```

2. **Parse tool_calls response:**
```c
if (finish_reason == "tool_calls") {
   // Extract tool_calls array from message
   // Return tool_call_list_t instead of text
}
```

### `src/llm/llm_claude.c`

1. **Add tools to request:**
```c
json_object *tools = llm_tools_get_claude_format();
if (tools) {
   json_object_object_add(claude_request, "tools", tools);
}
```

2. **Parse tool_use content blocks:**
```c
if (content_type == "tool_use") {
   // Extract id, name, input from content block
}
```

---

## Phase 3: Tool Execution Loop

### `src/llm/llm_interface.c`

New function with tool loop:

```c
char *llm_chat_completion_with_tools(
   struct json_object *history,
   const char *input,
   llm_text_chunk_callback callback,
   void *userdata
) {
   int max_iterations = 5;

   while (max_iterations-- > 0) {
      response = call_llm(...);

      if (response.has_tool_calls) {
         for (each tool_call) {
            llm_tools_execute(&call, &result);
            add_tool_result_to_history(history, &result);
         }
         // Loop continues - LLM will respond with more tools or text
      } else {
         return response.text;  // Final text response
      }
   }
}
```

### Tool Result Format

**OpenAI:**
```json
{"role": "tool", "tool_call_id": "call_xxx", "content": "result"}
```

**Claude:**
```json
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_xxx", "content": "result"}]}
```

---

## Phase 4: Streaming Support

### `src/llm/llm_streaming.c`

**OpenAI streaming:**
- Tool calls come as deltas in `tool_calls[].function.arguments`
- Accumulate arguments across chunks
- Detect via `finish_reason: "tool_calls"`

**Claude streaming:**
- `content_block_start` with `type: "tool_use"`
- `content_block_delta` with `type: "input_json_delta"`
- `content_block_stop` signals end

Add to `llm_stream_context_t`:
```c
tool_call_list_t pending_tool_calls;
int tool_calls_detected;
```

---

## Phase 5: Reduce System Prompt

### `src/llm/llm_command_parser.c`

When tools enabled, `get_system_instructions()` returns minimal prompt:
- Keep: Personality, response length rules, behavior guidelines
- Remove: All tool-specific instructions (weather, search, calculator, etc.)
- Remove: `<command>` tag format examples

**Before:** ~7,700 chars (~1,900 tokens)
**After:** ~2,000 chars (~500 tokens)

---

## Phase 6: Configuration & Backward Compatibility

### Config Addition

**`include/config/dawn_config.h`** - Add to llm_config_t:
```c
struct {
   bool native_tools_enabled;  /* Use native tool calling (default: false) */
} tools;
```

**`config.toml`:**
```toml
[llm]
native_tools_enabled = false  # Enable native tool calling (reduces prompt size)
```

### Fallback Strategy

1. Check `g_config.llm.tools.native_tools_enabled`
2. If enabled AND provider supports tools: Use native tool calling
3. Otherwise: Fall back to `<command>` tag parsing

### Keep existing code paths:
- `parse_llm_response_for_commands()` remains for fallback
- Can switch between modes via config without code changes

---

## Files Summary

| File | Action | Changes |
|------|--------|---------|
| `include/llm/llm_tools.h` | **Create** | Tool definitions, API |
| `src/llm/llm_tools.c` | **Create** | Schema generation, execution bridge |
| `include/config/dawn_config.h` | Modify | Add native_tools_enabled to llm_config_t |
| `src/config/config_parser.c` | Modify | Parse native_tools_enabled from TOML |
| `src/config/config_defaults.c` | Modify | Set default to false |
| `src/llm/llm_interface.c` | Modify | Add tool-enabled completion function |
| `src/llm/llm_openai.c` | Modify | Add tools to request, parse tool_calls |
| `src/llm/llm_claude.c` | Modify | Add tools to request, parse tool_use |
| `src/llm/llm_streaming.c` | Modify | Handle tool calls in streams |
| `src/llm/llm_command_parser.c` | Modify | Minimal prompt when tools enabled |
| `CMakeLists.txt` | Modify | Add llm_tools.c |

---

## Implementation Order

1. **Phase 1** - Tool infrastructure (llm_tools.h/c)
2. **Phase 6** - Backward compatibility check function
3. **Phase 2** - Provider modifications (non-streaming first)
4. **Phase 3** - Execution loop
5. **Phase 5** - Prompt reduction
6. **Phase 4** - Streaming support

---

## Testing Strategy

1. Test each provider independently with a simple tool (weather)
2. Test tool execution loop (multi-turn tool calls)
3. Test fallback to `<command>` tags
4. Test streaming with tool calls
5. Verify prompt size reduction

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Local LLM tool support varies | Capability detection + fallback |
| Infinite tool loops | Max 5 iterations |
| Breaking existing functionality | Feature flag, gradual rollout |
| Streaming complexity | Implement non-streaming first |
