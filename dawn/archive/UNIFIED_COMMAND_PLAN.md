> **STATUS: COMPLETE** - Archived 2025-12-25

# Unified Command Execution Architecture Plan

## Problem Statement

Dawn has three command execution paths with significant asymmetries:

| Path | Entry | Execution | Problem |
|------|-------|-----------|---------|
| Direct (pattern match) | `text_to_command_nuevo.c` | `deviceCallbackArray` | Only works for devices with callbacks |
| `<command>` tags | `llm_command_parser.c:862` | Direct MQTT publish | **Bypasses callbacks entirely** |
| Native tools | `llm_tools.c:886` | Callbacks + MQTT fallback | 160-line switch, hardcoded definitions |

**Key Issues:**
1. Commands defined in TWO places (JSON config + hardcoded `llm_tools_init()`)
2. `<command>` path publishes to MQTT, never reaches software callbacks (weather, search, calculator)
3. 160-line switch statement in `llm_tools_execute()` duplicates config semantics

## Solution Overview

Create a **unified command registry** and **single execution function** that all paths call.

```
┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  Direct Path    │   │  <command> Path  │   │  Native Tool    │
│  (pattern)      │   │  (LLM response)  │   │  (function call)│
└────────┬────────┘   └────────┬─────────┘   └────────┬────────┘
         │                     │                      │
         └─────────────────────┼──────────────────────┘
                               ▼
                    ┌──────────────────────┐
                    │  command_execute()   │
                    │  (unified entry)     │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       ┌──────────┐     ┌──────────┐     ┌──────────┐
       │ Callback │     │   MQTT   │     │ Sync Wait│
       │ (software)│    │ (hardware)│    │ (viewing)│
       └──────────┘     └──────────┘     └──────────┘
```

---

## Phase 1: Command Registry Foundation

**Goal:** Create new modules without changing existing behavior.

### New Files

**`include/core/command_registry.h`**
```c
typedef struct {
   char name[64];              // "weather", "hud_control"
   char description[512];      // Tool description for LLM
   char device_string[64];     // Device name for callback lookup
   char topic[32];             // MQTT topic
   bool has_callback;          // true if in deviceCallbackArray
   bool mqtt_only;             // No callback, MQTT-only
   bool sync_wait;             // Wait for MQTT response (viewing)
   bool skip_followup;         // Don't call LLM after (switch_llm)
   cmd_param_t parameters[8];  // Parameter definitions with maps_to
   int param_count;
} cmd_definition_t;

int command_registry_init(void);
const cmd_definition_t *command_registry_lookup(const char *name);
bool command_registry_validate(const char *device, char *topic_out, size_t size);
void command_registry_foreach(void (*cb)(const cmd_definition_t *, void *), void *);
```

**`include/core/command_executor.h`**
```c
typedef struct {
   char *result;           // Caller must free
   bool success;
   bool should_respond;
   bool skip_followup;
} cmd_exec_result_t;

int command_execute(const char *device, const char *action,
                    const char *value, struct mosquitto *mosq,
                    cmd_exec_result_t *result);
```

**`src/core/command_registry.c`**
- Parse `commands_config_nuevo.json` at startup
- Build lookup table from device names to definitions
- Detect which devices have callbacks via `get_device_callback()` probe

**`src/core/command_executor.c`**
```c
int command_execute(...) {
   const cmd_definition_t *cmd = command_registry_lookup(device);
   if (!cmd) return error;

   if (cmd->has_callback) {
      callback = get_device_callback(cmd->device_string);
      result = callback(action, value, &should_respond);
      return success;
   }

   if (cmd->sync_wait) {
      return command_execute_sync(...);  // viewing
   }

   // MQTT publish
   mosquitto_publish(mosq, topic, json_command);
   return success;
}
```

### Files to Modify

| File | Change |
|------|--------|
| `CMakeLists.txt` | Add `src/core/command_registry.c`, `src/core/command_executor.c` |
| `src/dawn.c` | Call `command_registry_init()` at startup |

---

## Phase 2: Migrate `<command>` Tag Path

**Goal:** Route `<command>` execution through unified executor.

### File: `src/llm/llm_command_parser.c`

**Before (line 862):**
```c
int rc = mosquitto_publish(mosq, NULL, topic, strlen(command), command, 0, false);
```

**After:**
```c
cmd_exec_result_t exec_result;
int rc = command_execute(device, action, value, mosq, &exec_result);
command_exec_result_free(&exec_result);
```

**Benefit:** Software callbacks (weather, search, calculator) now work from `<command>` tags.

---

## Phase 3: Migrate Native Tool Path

**Goal:** Auto-generate tools from registry, replace switch statement.

### File: `src/llm/llm_tools.c`

**Replace `llm_tools_init()` (lines 438-686):**
```c
void llm_tools_init(void) {
   s_tool_count = 0;
   command_registry_foreach(generate_tool_from_cmd, NULL);
   llm_tools_refresh();
}

static void generate_tool_from_cmd(const cmd_definition_t *cmd, void *data) {
   tool_definition_t *t = &s_tools[s_tool_count++];
   safe_strncpy(t->name, cmd->name, ...);
   safe_strncpy(t->description, cmd->description, ...);
   t->device_name = cmd->device_string;
   // Copy parameters from cmd->parameters
}
```

**Replace switch statement (lines 989-1153) with table-driven extraction:**
```c
int llm_tools_execute(const tool_call_t *call, tool_result_t *result) {
   const cmd_definition_t *cmd = command_registry_lookup(call->name);

   char device[64], action[64], value[4096];
   extract_params_from_args(cmd, call->arguments, device, action, value);

   cmd_exec_result_t exec_result;
   command_execute(device, action, value, mosq, &exec_result);

   result->success = exec_result.success;
   result->skip_followup = exec_result.skip_followup;
   // ...
}
```

---

## Phase 4: Extend JSON Config

**Goal:** Move tool definitions from code to config.

### File: `commands_config_nuevo.json`

Add `tool` blocks to devices that should be LLM-callable:

```json
{
  "devices": {
    "weather": {
      "type": "getter",
      "topic": "dawn",
      "callback": "weather",
      "tool": {
        "description": "Get weather forecast for a location",
        "parameters": [
          {
            "name": "location",
            "type": "string",
            "description": "City and state/country",
            "required": true,
            "maps_to": "value"
          }
        ]
      }
    },
    "hud_control": {
      "type": "boolean",
      "topic": "hud",
      "mqtt_only": true,
      "tool": {
        "description": "Control HUD display elements",
        "parameters": [
          {"name": "element", "type": "enum", "enum": ["armor_display", "detect", "map", "info"], "maps_to": "device"},
          {"name": "action", "type": "enum", "enum": ["enable", "disable"], "maps_to": "action"}
        ]
      }
    },
    "viewing": {
      "type": "getter",
      "topic": "hud",
      "sync_wait": true,
      "tool": {
        "description": "Take a photo and analyze what you see",
        "parameters": []
      }
    }
  }
}
```

---

## Phase 5: Cleanup

**Remove deprecated code:**
- `validate_device_in_config()` from `llm_command_parser.c` (use registry)
- Hardcoded tool definitions from `llm_tools_init()` (now auto-generated)
- 160-line switch statement from `llm_tools_execute()` (now table-driven)

---

## Special Cases to Handle

| Tool | Behavior | Implementation |
|------|----------|----------------|
| `switch_llm` | Skip LLM follow-up after execution | `skip_followup: true` in config |
| `viewing` | Synchronous MQTT wait for image | `sync_wait: true` → `command_execute_sync()` |
| `hud_control` | `element` param becomes device | `maps_to: "device"` in param config |
| `recording` | `mode` param selects device | `maps_to: "device"` (record/stream/record and stream) |

---

## Files Summary

### New Files (4)
- `include/core/command_registry.h`
- `src/core/command_registry.c`
- `include/core/command_executor.h`
- `src/core/command_executor.c`

### Modified Files (5)
- `CMakeLists.txt` - Add new source files
- `commands_config_nuevo.json` - Add tool metadata
- `src/dawn.c` - Initialize registry at startup
- `src/llm/llm_command_parser.c` - Use unified executor
- `src/llm/llm_tools.c` - Auto-generate tools, table-driven params

---

## Implementation Order

### First Implementation Session (Phases 1-2)
1. **Phase 1** - Create registry + executor (non-breaking, parallel to existing)
2. **Phase 2** - Migrate `<command>` path (fixes software callback gap)
3. **Validate** - Test that `<command>` tags now work for weather/search/calculator

### Future Sessions (Phases 3-5)
4. **Phase 3** - Migrate tool path (removes 160-line switch)
5. **Phase 4** - Extend JSON config (single source of truth)
6. **Phase 5** - Remove deprecated code

Each phase can be tested independently and reverted if issues arise.

---

## Tool Block Policy

Tool definitions in the JSON config are **optional**. Only devices that should be callable by the LLM need a `tool` block:

**LLM-callable (need tool blocks):**
- Information: `weather`, `search`, `calculator`, `date`, `time`
- Vision: `viewing`
- Home automation: `smartthings`
- Device control: `hud_control`, `faceplate`, `recording`, `voice_amplifier`
- Assistant control: `switch_llm`, `reset conversation`, `llm_status`

**Voice-only (no tool block needed):**
- Audio routing: `microphone`, `headphones`, `speakers`, `audio capture device`, `audio playback device`
- HUD calibration: `visual offset`
- Music: `flac`
