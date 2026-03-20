# Dynamic Command Tags Generation Plan

## Goal

Eliminate hard-coded `LEGACY_RULES_*` strings and generate `<command>` tag instructions dynamically from the tool_registry, just like we do for native tool calling.

## Current State

### Hard-coded strings in `llm_command_parser.c`:
- `LEGACY_RULES_CORE` - General command tag behavior rules
- `LEGACY_RULES_VISION` - Vision-specific instructions
- `LEGACY_RULES_WEATHER` - Weather tool format
- `LEGACY_RULES_SEARCH` - Search tool format
- `LEGACY_RULES_CALCULATOR` - Calculator tool format
- `LEGACY_RULES_URL` - URL fetcher format
- `LEGACY_RULES_LLM_STATUS` - LLM control format
- `LEGACY_RULES_SMARTTHINGS` - SmartThings format

### Native tool calling (already dynamic):
```c
tool_registry_foreach_enabled(generate_tool_from_treg, NULL);
```

## Tool Registry Data Available

From `tool_metadata_t`:
| Field | Use for Command Tags |
|-------|---------------------|
| `name` | Device name in JSON |
| `description` | Tool description in prompt |
| `params[]` | Actions and value format |
| `param_count` | Number of parameters |
| `device_type` | Determines default action |
| `is_getter` | Affects response handling hint |
| `default_remote` | Filter for remote clients |
| `capabilities` | Check if tool is available |

From `treg_param_t`:
| Field | Use for Command Tags |
|-------|---------------------|
| `name` | Parameter name |
| `description` | Parameter description |
| `type` | STRING, ENUM, NUMBER, BOOLEAN |
| `maps_to` | ACTION, VALUE, or NONE |
| `enum_values[]` | Valid actions list |
| `enum_count` | Number of valid actions |

## Proposed Design

### 1. Keep Minimal Core Rules

Replace `LEGACY_RULES_CORE` with a simpler, generic version:

```c
static const char *COMMAND_TAG_FORMAT_RULES =
   "COMMAND FORMAT: Use <command>{\"device\":\"NAME\",\"action\":\"ACTION\",\"value\":\"VALUE\"}</command>\n"
   "RULES:\n"
   "1. For actions: one sentence, then the command tag. No prose after the tag.\n"
   "2. For getters: send ONLY the tag, wait for response, then confirm.\n"
   "3. Use only the devices and actions listed below.\n"
   "4. If ambiguous, ask for clarification.\n"
   "5. Multiple commands can use multiple <command> tags.\n";
```

### 2. New Function: Generate Tool Instructions

```c
/**
 * @brief Generate command tag instructions for a single tool
 *
 * @param tool The tool metadata
 * @param buffer Output buffer
 * @param buffer_size Buffer size
 * @return Number of bytes written
 */
static int generate_command_tag_instructions(const tool_metadata_t *tool,
                                              char *buffer,
                                              size_t buffer_size);
```

Output format per tool:
```
WEATHER: Get weather forecasts for a location.
  <command>{"device":"weather","action":"ACTION","value":"location"}</command>
  Actions: today (current), tomorrow (2-day), week (7-day)
```

### 3. New Function: Build Dynamic Capabilities List

```c
/**
 * @brief Build "CAPABILITIES: You CAN..." list from registry
 */
static int build_capabilities_list(char *buffer, size_t buffer_size);
```

Iterates enabled tools and builds:
```
CAPABILITIES: You CAN get weather, perform web searches, do calculations, fetch URLs, ...
```

### 4. New Callback for Iteration

```c
typedef struct {
   char *buffer;
   size_t buffer_size;
   int offset;
} command_tag_build_ctx_t;

static void generate_tool_command_tag(const tool_metadata_t *tool, void *user_data) {
   command_tag_build_ctx_t *ctx = (command_tag_build_ctx_t *)user_data;

   // Skip tools that aren't available (check conditions)
   if (!is_tool_available(tool)) return;

   ctx->offset += generate_command_tag_instructions(tool,
                                                     ctx->buffer + ctx->offset,
                                                     ctx->buffer_size - ctx->offset);
}
```

### 5. Tool Availability Checks

Some tools have runtime conditions. Add to tool_metadata_t or check inline:

| Tool | Condition |
|------|-----------|
| search | `is_search_enabled()` - SearXNG endpoint configured |
| viewing | `is_vision_enabled_for_current_llm()` |
| smartthings | `smartthings_is_authenticated()` |
| weather | Always available |
| calculator | Always available |
| url | Always available |
| llm | Always available |
| time/date | Always available |
| memory | `g_config.memory.enabled` |

**Option A**: Add `is_available` function pointer to `tool_metadata_t`
**Option B**: Check capabilities flags + known conditions in the generator

Recommend **Option A** for consistency with modular design.

### 6. Updated `build_system_instructions_to_buffer()`

```c
static int build_system_instructions_to_buffer(const char *mode, char *buffer, size_t buffer_size) {
   int len = 0;

   if (strcmp(mode, "native") == 0) {
      len += snprintf(buffer + len, buffer_size - len, "%s\n", NATIVE_TOOLS_RULES);
      return len;
   }

   if (strcmp(mode, "disabled") == 0) {
      return 0;
   }

   /* command_tags mode */

   // Core format rules (static, minimal)
   len += snprintf(buffer + len, buffer_size - len, "%s\n", COMMAND_TAG_FORMAT_RULES);

   // Dynamic capabilities list
   len += build_capabilities_list(buffer + len, buffer_size - len);

   // Dynamic tool instructions from registry
   command_tag_build_ctx_t ctx = {
      .buffer = buffer + len,
      .buffer_size = buffer_size - len,
      .offset = 0
   };
   tool_registry_foreach_enabled(generate_tool_command_tag, &ctx);
   len += ctx.offset;

   return len;
}
```

## Implementation Phases

### Phase 1: Add `is_available` to Tool Metadata

**File: `include/tools/tool_registry.h`**
```c
typedef struct tool_metadata {
   // ... existing fields ...

   /** Optional availability check (NULL = always available) */
   bool (*is_available)(void);
} tool_metadata_t;
```

**Files: `src/tools/*.c`**
Add `is_available` function to tools that need runtime checks:
- `search_tool.c` → `is_search_enabled`
- `viewing_tool.c` → `is_vision_enabled_for_current_llm`
- `smartthings_tool.c` → `smartthings_is_authenticated`
- `memory_tool.c` → check `g_config.memory.enabled`

### Phase 2: Implement Generation Functions

**File: `src/llm/llm_command_parser.c`**

1. Add `generate_command_tag_instructions()`:
   - Extract action enum values from params
   - Extract value description from params
   - Format as command tag example

2. Add `build_capabilities_list()`:
   - Iterate enabled + available tools
   - Build human-readable capability string

3. Add iteration callback `generate_tool_command_tag()`

### Phase 3: Update `build_system_instructions_to_buffer()`

Replace hard-coded LEGACY_RULES_* with dynamic generation.

### Phase 4: Remove Dead Code

Delete from `llm_command_parser.c`:
- `LEGACY_RULES_VISION`
- `LEGACY_RULES_WEATHER`
- `LEGACY_RULES_SEARCH`
- `LEGACY_RULES_CALCULATOR`
- `LEGACY_RULES_URL`
- `LEGACY_RULES_LLM_STATUS`
- `LEGACY_RULES_SMARTTHINGS`

Simplify `LEGACY_RULES_CORE` → `COMMAND_TAG_FORMAT_RULES`

## Example Output

For a tool like `weather_tool.c`:

```c
static const tool_metadata_t weather_metadata = {
   .name = "weather",
   .description = "Get weather forecasts for a location",
   .params = (treg_param_t[]){
      {
         .name = "action",
         .type = TOOL_PARAM_TYPE_ENUM,
         .maps_to = TOOL_MAPS_TO_ACTION,
         .enum_values = {"today", "tomorrow", "week"},
         .enum_count = 3,
         .description = "Forecast type: today (current), tomorrow (2-day), week (7-day)"
      },
      {
         .name = "location",
         .type = TOOL_PARAM_TYPE_STRING,
         .maps_to = TOOL_MAPS_TO_VALUE,
         .description = "City and state/country"
      }
   },
   .param_count = 2,
   // ...
};
```

Would generate:
```
WEATHER: Get weather forecasts for a location
  <command>{"device":"weather","action":"ACTION","value":"location"}</command>
  Actions: today (current), tomorrow (2-day), week (7-day)
```

## Files to Modify

| File | Changes |
|------|---------|
| `include/tools/tool_registry.h` | Add `is_available` function pointer |
| `src/tools/search_tool.c` | Add `is_available = is_search_enabled` |
| `src/tools/viewing_tool.c` | Add `is_available` check |
| `src/tools/smartthings_tool.c` | Add `is_available = smartthings_is_authenticated` |
| `src/tools/memory_tool.c` | Add `is_available` check |
| `src/llm/llm_command_parser.c` | Implement dynamic generation, remove LEGACY_RULES_* |

## Testing

1. Switch to `command_tags` mode
2. Verify prompt contains all enabled tools
3. Verify tools with unmet conditions are excluded
4. Test that LLM correctly uses the generated format
5. Compare prompt size (should be similar or smaller)

## Rollback

Keep LEGACY_RULES_* strings commented out until confident the dynamic generation works correctly.
