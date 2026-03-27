# DAWN Two-Step Tool Pattern

**On-Demand Instruction Loading for Complex Tools**

| Field | Value |
|-------|-------|
| Date | March 2026 |
| Status | Complete — implemented, tested (42 assertions), documented in TOOL_DEVELOPMENT_GUIDE.md |
| Applies To | Any DAWN tool that needs detailed, context-dependent instructions |

---

## Overview

Some DAWN tools are simple enough that the LLM can use them with just a function calling schema — email, calendar, web search. The schema describes the parameters, and the LLM fills them in. No special instructions needed.

Other tools require detailed operational rules to produce high-quality output — rules that are too large or too specialized to keep in every conversation's system prompt. The visual rendering tool is the first example: the LLM needs 2,000-4,000+ tokens of design guidelines (color palettes, layout rules, font calibration, dark mode handling, SVG conventions) to generate good diagrams. Loading all of that into every conversation wastes context window space in the 95% of conversations that never draw anything.

The two-step pattern solves this: the LLM calls a lightweight "load instructions" tool first, gets back the detailed rules it needs, then calls the actual tool with full knowledge of how to do it well.

---

## The Pattern

### Step 1: Load Instructions

The LLM calls a `{tool}_load_instructions` function that returns detailed operational guidelines from a file on disk. The base system prompt contains only enough information for the LLM to know *when* to use the tool and *what instruction set to load*.

### Step 2: Execute

The LLM calls the actual tool, following the detailed rules it just received in Step 1's response.

### What Goes Where

| Location | Content | Size |
|----------|---------|------|
| Base system prompt | Tool exists, when to use it, what modules/categories are available | ~200-500 tokens |
| Instruction files on disk | Detailed operational rules, examples, failure modes, templates | ~1,000-4,000 tokens per module |
| Tool call result | The instruction content, injected into the conversation context | Loaded on demand |

---

## Why This Works

The instructions returned by Step 1 land in the conversation context as a tool result — the same place any other tool output goes. The LLM reads them the same way it reads web search results or file contents. From the LLM's perspective, it asked for instructions and received them. There is no special mechanism required.

The key insight is that this is just a file read wrapped in a tool call. The "two-step pattern" is not a complex framework — it's a convention that says: *before you do the complex thing, read the instructions for doing it well*.

---

## Architecture

### System Prompt (Always Present)

The base system prompt includes a minimal tool description. This is the "table of contents" — it tells the LLM what the tool can do and how to decide which instruction module to load.

```
You have a tool called [tool_name] that [brief description].

TO USE:
1. Call [tool_name]_load_instructions with the appropriate module.
2. Read the returned instructions carefully.
3. Call [tool_name] following those instructions.

MODULES:
- module_a: [one-line description]
- module_b: [one-line description]
- module_c: [one-line description]

WHEN TO USE:
- [trigger phrase] → module_a
- [trigger phrase] → module_b
```

### Instruction Files (On Disk)

Instruction files live in a known directory on the filesystem. They are plain markdown files that can be edited, version-controlled, and updated independently of the DAWN codebase.

```
/data/dawn/tool_instructions/
├── render_visual/
│   ├── _core.md          # Loaded with every module
│   ├── diagram.md        # Flowcharts, structural, illustrative
│   ├── chart.md          # Chart.js data visualization
│   ├── interactive.md    # Widgets with controls
│   ├── mockup.md         # UI mockups
│   └── art.md            # Illustrations, generative
├── document_writer/
│   ├── _core.md
│   ├── report.md
│   ├── email.md
│   └── technical.md
└── code_generator/
    ├── _core.md
    ├── c_cpp.md
    ├── python.md
    └── bash.md
```

### Tool Handler (C Side)

The instruction loader is a generic function that reads markdown from disk. Every tool that uses the two-step pattern shares the same loader — only the directory path differs.

```c
#define DAWN_INSTRUCTIONS_BASE_DIR "/data/dawn/tool_instructions"

int dawn_load_tool_instructions(
      const char *tool_name,
      const char **modules, int module_count,
      char *output, size_t output_len) {

   char dir[PATH_MAX];
   snprintf(dir, sizeof(dir), "%s/%s",
            DAWN_INSTRUCTIONS_BASE_DIR, tool_name);

   size_t offset = 0;

   // Always load _core.md first if it exists
   char core_path[PATH_MAX];
   snprintf(core_path, sizeof(core_path), "%s/_core.md", dir);
   FILE *core = fopen(core_path, "r");
   if (core) {
      offset += fread(output, 1, output_len - 1, core);
      fclose(core);
      if (offset < output_len - 5) {
         offset += snprintf(output + offset,
                            output_len - offset, "\n---\n");
      }
   }

   // Load each requested module
   for (int i = 0; i < module_count; i++) {
      // Sanitize module name (prevent path traversal)
      if (strchr(modules[i], '/') || strchr(modules[i], '\\') ||
          strstr(modules[i], "..")) {
         LOG_WARN("Invalid module name: %s", modules[i]);
         continue;
      }

      char path[PATH_MAX];
      snprintf(path, sizeof(path), "%s/%s.md", dir, modules[i]);

      FILE *f = fopen(path, "r");
      if (!f) {
         LOG_WARN("Instruction module not found: %s/%s",
                  tool_name, modules[i]);
         continue;
      }

      size_t bytes = fread(output + offset, 1,
                           output_len - offset - 1, f);
      offset += bytes;
      fclose(f);

      if (i < module_count - 1 && offset < output_len - 5) {
         offset += snprintf(output + offset,
                            output_len - offset, "\n---\n");
      }
   }

   output[offset] = '\0';
   return (offset > 0) ? DAWN_OK : DAWN_ERR_NOT_FOUND;
}
```

Each tool registers its own `_load_instructions` function that wraps this generic loader:

```c
int dawn_tool_render_visual_load_guidelines(
      const char **modules, int module_count,
      char *output, size_t output_len) {
   return dawn_load_tool_instructions(
      "render_visual", modules, module_count, output, output_len);
}
```

### Function Calling Schema (Generic Pattern)

```json
{
   "name": "{tool_name}_load_instructions",
   "description": "Load detailed instructions before using {tool_name}. Call this first.",
   "parameters": {
      "modules": {
         "type": "array",
         "items": {
            "type": "string",
            "enum": ["module_a", "module_b", "module_c"]
         },
         "description": "Which instruction modules to load."
      }
   }
}
```

---

## When to Use This Pattern

### Good Candidates

A tool should use the two-step pattern when:

- **The instructions are large** (>500 tokens) and would waste context in conversations that don't use the tool.
- **The instructions vary by sub-task** — different modules for different use cases (diagram vs. chart vs. interactive widget).
- **Output quality depends heavily on following specific rules** — the difference between "works" and "works well" is significant.
- **The instructions evolve independently** of the main codebase — you want to update the rules without recompiling DAWN or changing the system prompt.

### Bad Candidates

A tool should NOT use the two-step pattern when:

- **The tool is simple enough** that the function calling schema is sufficient. Email sending, timer setting, light control — these just need parameter descriptions.
- **The instructions are short** (<200 tokens) — just include them in the base system prompt.
- **Speed matters more than quality** — the extra round-trip adds latency. For tools the user expects to be instant, the tradeoff isn't worth it.
- **The LLM already knows how to do it** — web search, basic math, general knowledge queries don't need custom instructions.

### Assessment Matrix

| Tool | Instructions Size | Quality Sensitive | Varies by Sub-task | Pattern? |
|------|------------------|-------------------|-------------------|----------|
| render_visual | 2,000-4,000 tokens | Very | Yes (diagram, chart, etc.) | Yes |
| image_generate | <100 tokens | No (sd.cpp handles quality) | No | No |
| email_send | <100 tokens | Moderate | No | No |
| web_search | <100 tokens | No | No | No |
| document_writer | 1,000-3,000 tokens | Very | Yes (report, email, technical) | Yes |
| code_generator | 1,000-2,000 tokens | Yes | Yes (language-specific rules) | Yes |
| home_assistant | 500-1,500 tokens | Moderate | Yes (device types) | Maybe |

---

## Design Considerations

### The _core.md Convention

Every tool's instruction directory can include a `_core.md` file that is loaded automatically with every module request. This contains rules that apply universally regardless of which specific module was requested. For the visual rendering tool, this is the color palette, typography rules, dark mode requirements, and SVG setup conventions. Module-specific files then add specialized rules on top.

### Path Traversal Prevention

The instruction loader must sanitize module names to prevent path traversal attacks. Even though the LLM is local, the module names come from the LLM's function call output, which is influenced by user input. A prompt injection could attempt to read arbitrary files by requesting a module named `../../etc/passwd`. The loader rejects any module name containing `/`, `\`, or `..`.

### Token Budget Awareness

Each loaded module consumes context window tokens. The system prompt should hint at module sizes so the LLM can make informed decisions about how many modules to load simultaneously. For Qwen3.5-35B-A3B with 262K context, this is rarely a practical concern, but on smaller models with 8K-32K context windows, loading multiple large modules could crowd out conversation history.

### Instruction Caching — IMPLEMENTED

When the same module is loaded multiple times in a single conversation, the tool returns a short message ("Guidelines already loaded — refer to the instructions provided above") instead of re-reading 10KB+ from disk. Tracking is anchored to the session via `visual_modules_loaded` on `session_t`, using delimiter-bounded matching (`,diagram,chart,`) to prevent substring collisions (e.g., "art" vs "chart"). Cache resets on conversation clear.

### Versioning and Iteration

Instruction files are plain markdown on disk. They can be:

- Version-controlled in git alongside the DAWN codebase
- Edited by the user to customize output style
- A/B tested by swapping files and comparing output quality
- Incrementally grown as new failure modes are discovered

This is one of the pattern's biggest advantages over baking instructions into the system prompt: the iteration cycle is "edit a file and test" rather than "modify the prompt string in code, recompile, and test."

---

## Implementation Checklist

For adding a new two-step tool to DAWN:

1. **Create the instruction directory** under `/data/dawn/tool_instructions/{tool_name}/`.
2. **Write `_core.md`** with universal rules for the tool.
3. **Write module files** (`module_a.md`, `module_b.md`, etc.) with sub-task-specific rules.
4. **Register the load function** in the tool registry, wrapping `dawn_load_tool_instructions()`.
5. **Register the execution function** in the tool registry.
6. **Add the base prompt section** — tool description, module list, trigger phrases. Keep it under 500 tokens.
7. **Add the function calling schemas** for both `{tool}_load_instructions` and `{tool}`.
8. **Test the full loop**: user request → LLM loads instructions → LLM executes tool → output is correct.
9. **Iterate on instructions** based on observed output quality. Add rules for each new failure mode.

---

## Example: Complete Flow

Here is a concrete example showing how the two-step pattern works end-to-end for the visual rendering tool:

**User says:** "Show me how DAWN's satellite architecture works."

**LLM decides:** This needs a diagram. I should load the diagram guidelines first.

**Step 1 — LLM calls:**
```json
{
   "name": "render_visual_load_guidelines",
   "parameters": { "modules": ["diagram"] }
}
```

**DAWN reads** `/data/dawn/tool_instructions/render_visual/_core.md` and `/data/dawn/tool_instructions/render_visual/diagram.md`, concatenates them, and returns the combined content (~3,000 tokens) as the tool result.

**LLM reads** the returned guidelines, including color ramps, font sizing rules, SVG conventions, box sizing formulas, and arrow routing rules.

**Step 2 — LLM calls:**
```json
{
   "name": "render_visual",
   "parameters": {
      "title": "dawn_satellite_architecture",
      "type": "svg",
      "code": "<svg width=\"100%\" viewBox=\"0 0 680 400\">..."
   }
}
```

The generated SVG follows the rules from Step 1: correct viewBox width, color classes from the palette, properly sized boxes, arrows that don't cross other nodes, dark mode compatible text.

**DAWN sends** the SVG to the WebUI, which renders it inline in a sandboxed iframe.

**LLM also responds** with a spoken explanation via TTS: "I've put together a diagram of the satellite architecture on the WebUI. DAWN uses a hub-and-spoke model where the central daemon communicates with both Tier 1 RPi satellites and Tier 2 ESP32 pucks over the DAP2 protocol."

---

## Comparison: With vs. Without the Pattern

### Without (instructions baked into system prompt)

- Every conversation loads 3,000+ tokens of visual guidelines, even for "what's the weather" or "set a timer for 5 minutes."
- Updating guidelines requires modifying the system prompt string in code.
- Can't have sub-task-specific modules — everything is one blob.
- On smaller models with limited context, the guidelines crowd out conversation history.

### With (two-step pattern)

- Base prompt costs ~300-500 tokens. Guidelines load only when needed.
- Guidelines are markdown files on disk — edit and test without recompiling.
- Modules allow loading only the relevant subset of rules.
- One extra tool call of latency when a visual is first requested in a conversation.

The tradeoff is one additional LLM inference round-trip per conversation that uses the tool. For tools where quality matters (visuals, document generation), this is worth it. For tools where speed matters (timers, quick lookups), it is not.
