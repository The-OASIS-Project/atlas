# DAWN Visual Rendering Tool

**Inline SVG/HTML Diagrams via LLM Tool Calling**

| Field | Value |
|-------|-------|
| Date | March 2026 |
| Status | Shipped (Phase 1-2 complete, Phase 3 partial) |
| Depends On | DAWN WebUI, LLM function calling, two-step tool pattern |

---

## Overview

This document describes how to add a visual rendering tool to DAWN that enables the LLM to generate inline SVG diagrams and interactive HTML widgets as part of its conversation responses. When the LLM determines that a visual explanation would help the user understand something, it invokes the `render_visual` tool with generated SVG or HTML code, which the WebUI renders inline in the conversation thread.

This is not image generation (that is covered by the separate stable-diffusion.cpp integration). This is the LLM writing SVG/HTML code that the browser renders natively. The output includes flowcharts, architecture diagrams, data visualizations, interactive explainers, and illustrative cross-sections.

### How It Works (Three Components)

The visual rendering capability consists of three parts, each simple on its own:

1. **A two-step tool** — the LLM first calls `render_visual_load_guidelines` to load the design system for the type of visual it wants to create, then calls `render_visual` with the generated code. See [TWO_STEP_TOOL_PATTERN.md (archived)](https://github.com/The-OASIS-Project/atlas/blob/main/dawn/archive/TWO_STEP_TOOL_PATTERN.md) for the general pattern.
2. **A tool definition** in the function calling schema that the LLM invokes with generated code as a string parameter.
3. **A frontend renderer** in the WebUI that takes the code string and renders it inline in the conversation via a sandboxed iframe.

The LLM does the hard work of writing the SVG. The tooling is just plumbing.

---

## Two-Step Tool Pattern

The visual rendering tool uses the two-step pattern described in [TWO_STEP_TOOL_PATTERN.md (archived)](https://github.com/The-OASIS-Project/atlas/blob/main/dawn/archive/TWO_STEP_TOOL_PATTERN.md). This keeps the base system prompt lightweight (~400 tokens for the tool description) while loading the full design system (~2,000-4,000+ tokens) only when the LLM decides to actually draw something.

**Step 1: Load guidelines.** The LLM calls `render_visual_load_guidelines` with a `modules` parameter specifying what kind of visual it wants to create. The tool returns the detailed design rules for that visual type.

**Step 2: Render.** The LLM calls `render_visual` with the generated SVG or HTML code, following the rules it just loaded.

The base system prompt contains only enough information for the LLM to know *when* to use the tool and *which module to load*. The detailed rules for *how* to draw live in the guidelines files and are loaded on demand.

### Tool Schemas

```json
{
   "name": "render_visual_load_guidelines",
   "description": "Load design guidelines before creating a visual. Call this BEFORE render_visual. Returns detailed rules for creating high-quality SVG/HTML visuals.",
   "parameters": {
      "modules": {
         "type": "array",
         "items": {
            "type": "string",
            "enum": ["diagram", "chart", "interactive", "mockup", "art"]
         },
         "description": "Which guideline modules to load. Pick all that fit the visual you want to create."
      }
   }
}
```

```json
{
   "name": "render_visual",
   "description": "Render an inline SVG or HTML visual in the conversation. Always call render_visual_load_guidelines first.",
   "parameters": {
      "title": {
         "type": "string",
         "description": "Short snake_case identifier (e.g. 'tcp_handshake_flow', 'memory_layout')"
      },
      "code": {
         "type": "string",
         "description": "Raw SVG or HTML code to render. SVG starts with <svg>. HTML should not include DOCTYPE, html, head, or body tags."
      },
      "type": {
         "type": "string",
         "enum": ["svg", "html"],
         "description": "Content type: svg for static diagrams, html for interactive widgets with JavaScript."
      }
   }
}
```

### Base System Prompt (Always Loaded)

This goes in every conversation's system prompt. It is deliberately minimal — just enough for the LLM to decide when to draw and which module to load.

```
You have a visual rendering tool that renders inline SVG or HTML
diagrams in the conversation. Use it proactively when a visual
explanation would help the user.

TO USE:
1. Call render_visual_load_guidelines with the appropriate modules.
2. Read the returned design rules carefully.
3. Call render_visual with SVG or HTML code following those rules.

MODULES:
- diagram:     Flowcharts, architecture diagrams, structural layouts
- chart:       Data charts, bar/line/pie via Chart.js
- interactive: Widgets with sliders, toggles, state, user controls
- mockup:      UI mockups, forms, cards, dashboards
- art:         Illustrations, generative art, decorative visuals

WHEN TO USE:
- "How does X work" → diagram or interactive
- "Show me the architecture" → diagram
- "Chart this data" → chart
- "What would the UI look like" → mockup
- Explaining anything with spatial/sequential/systemic relationships

WHEN NOT TO USE:
- Simple factual answers
- Code generation (use code blocks instead)
- When the user asks for a generated image (use image_generate instead)
```

---

## Tool Handler (C Side)

The tool handlers on the DAWN daemon side are minimal.

### Guidelines Loader

The guidelines loader uses the shared `instruction_loader` module (`src/tools/instruction_loader.c`) to read markdown files from disk. The guideline files live in `tool_instructions/render_visual/` (relative to the working directory). The `_core.md` file is always prepended. Module names are validated for path traversal (rejects `/`, `\`, `..`, leading `.`). Session-based caching tracks which modules have been loaded to avoid re-sending 10KB+ guidelines within the same conversation.

The actual implementation delegates to `instruction_loader_load()`:

```c
/* In render_visual_tool.c — load_guidelines_callback() */
char *content = NULL;
int rc = instruction_loader_load("render_visual", modules_csv, &content);
/* content contains _core.md + requested modules, separated by "\n\n---\n\n" */
```

See `src/tools/instruction_loader.c` for the full loader implementation (path traversal validation, pre-scan sizing, 4KB-128KB buffer range) and `src/tools/render_visual_tool.c` for the tool callbacks.

### Visual Renderer

The render tool wraps the LLM-generated code in `<dawn-visual>` tags and returns it as the tool result string. The WebUI's streaming renderer detects these tags via regex and renders them as sandboxed iframes inline in the conversation.

```c
/* In render_visual_tool.c — render_visual_callback() */
/* Wraps code in: <dawn-visual title="..." type="svg|html">...code...</dawn-visual> */
/* The WebUI's visual-render.js parses these tags during streaming */
```

See `src/tools/render_visual_tool.c` for the actual callback implementation.

---

## WebUI Renderer

The WebUI renderer is the core frontend component. When it detects `<dawn-visual>` tags in the streamed conversation content, it creates a sandboxed iframe and injects the LLM-generated code. The actual implementation is in `www/js/ui/visual-render.js`. The code below illustrates the pattern:

### Iframe Sandbox Architecture

The generated SVG/HTML runs inside a sandboxed iframe for security isolation. The LLM's code cannot access the parent DOM, DAWN's WebSocket connection, or any other page state. The `sandbox` attribute restricts capabilities to only `allow-scripts` (needed for interactive widgets).

```javascript
// WebUI: rendering a visual message
function renderVisual(message) {
   const container = document.createElement('div');
   container.className = 'dawn-visual-container';

   const iframe = document.createElement('iframe');
   iframe.sandbox = 'allow-scripts';
   iframe.style.cssText = 'width:100%; border:none; overflow:hidden;';

   // Inject theme CSS variables + the LLM's code
   const themeCSS = getDawnThemeCSS();
   const bridge = getSendPromptBridge();

   let content;
   if (message.type === 'svg') {
      content = `
         <style>${themeCSS}</style>
         <style>
            body { margin: 0; padding: 0;
                   background: transparent;
                   overflow: hidden; }
            svg  { display: block; width: 100%; height: auto; }
         </style>
         ${bridge}
         ${message.code}
      `;
   } else {
      content = `
         <style>${themeCSS}</style>
         ${bridge}
         ${message.code}
      `;
   }

   iframe.srcdoc = content;
   container.appendChild(iframe);

   // Auto-size iframe to content height
   iframe.addEventListener('load', () => {
      const resizeObserver = new ResizeObserver(() => {
         const h = iframe.contentDocument.body.scrollHeight;
         iframe.style.height = h + 'px';
      });
      resizeObserver.observe(iframe.contentDocument.body);
   });

   return container;
}
```

### Theme CSS Injection

The WebUI injects DAWN's theme CSS variables into the iframe so the LLM's output automatically matches DAWN's color scheme. The actual implementation reads computed CSS variables from the parent document (via `getComputedStyle`) and injects them as `:root` rules. It also injects pre-built SVG classes (`t`, `ts`, `th`, `box`, `arr`, `node`) and the 9-color ramp system with automatic light/dark mode adaptation. See `buildThemeCSS()`, `buildVisualClasses()`, and `buildColorRampCSS()` in `visual-render.js`. The pattern is illustrated below:

```javascript
function getDawnThemeCSS() {
   const isDark = document.documentElement
                     .getAttribute('data-theme') === 'dark';
   return `
      :root {
         --color-bg-primary: ${isDark ? '#1a1a1a' : '#ffffff'};
         --color-bg-secondary: ${isDark ? '#2a2a2a' : '#f5f5f5'};
         --color-bg-tertiary: ${isDark ? '#333' : '#eee'};
         --color-text-primary: ${isDark ? '#e0e0e0' : '#1a1a1a'};
         --color-text-secondary: ${isDark ? '#999' : '#666'};
         --color-text-tertiary: ${isDark ? '#777' : '#888'};
         --color-border: ${isDark ? '#444' : '#ddd'};
         --color-border-light: ${isDark ? '#333' : '#eee'};
         --color-accent: #2E5090;
         --font-sans: 'Inter', system-ui, sans-serif;
         --font-mono: 'JetBrains Mono', monospace;
         --border-radius-md: 8px;
         --border-radius-lg: 12px;
      }
      body {
         font-family: var(--font-sans);
         color: var(--color-text-primary);
         background: transparent;
         margin: 0;
      }
   `;
}
```

### sendPrompt() Bridge

The `sendPrompt()` function allows interactive elements inside the visual to send messages back to the chat, as if the user typed them. This enables clickable nodes in diagrams that trigger follow-up explanations.

**Inside the iframe (injected before the LLM's code):**

```javascript
function getSendPromptBridge() {
   return `<script>
      function sendPrompt(text) {
         parent.postMessage(
            { type: 'dawn_prompt', text: text },
            '*'
         );
      }
   <\/script>`;
}
```

**In the parent WebUI:**

```javascript
window.addEventListener('message', (event) => {
   if (event.data && event.data.type === 'dawn_prompt') {
      dawn_send_message(event.data.text);
   }
});
```

---

## Guideline Modules

The following sections define the content of each guideline module file. These are the files that `render_visual_load_guidelines` reads from disk and returns to the LLM. Each module is stored as a separate markdown file in the guidelines directory.

### Core Design System (loaded with every module)

This content is prepended to every module load. It defines the universal rules that apply to all visual types.

**File:** `_core.md`

````markdown
# Visual rendering design system

## SVG Setup
- ViewBox: `<svg width="100%" viewBox="0 0 680 H">` — 680px wide, H computed to fit content.
- Safe area: x=40 to x=640, y=40 to y=(H-40).
- Background: transparent. Do not add background colors to the SVG or wrapper divs.
- One SVG per tool call.

## Typography
- Two sizes only: 14px for node labels, 12px for subtitles and annotations.
- Two weights only: 400 (regular) and 500 (bold/headings).
- No font-size below 11px.
- Sentence case always. Never Title Case or ALL CAPS.

## Font Width Calibration
At 14px, Inter renders approximately:
- Lowercase characters: ~7.5px average width
- Uppercase characters: ~9px average width
- Practical estimate: chars × 8px = rendered width at 14px
- At 12px: chars × 6.8px = rendered width

Before placing text in a box: box_width must be >= (char_count × 8) + 24px padding.
If the text doesn't fit, either shorten the label or widen the box.

SVG `<text>` never auto-wraps. Every line break needs an explicit
`<tspan x="..." dy="1.2em">`. If your subtitle needs wrapping, it's too long.

## Colors
Use CSS variables for text. Never hardcode text colors.
- `var(--color-text-primary)`: main text
- `var(--color-text-secondary)`: muted/subtitle text
- `var(--color-text-tertiary)`: hints, annotations

For fills, use the named color ramp classes. Apply to a `<g>` wrapping
shapes and text. Child text colors auto-adjust for the fill.

### Color Ramp Reference
9 ramps, each with 7 stops (lightest to darkest):

| Class      | 50 (fill)  | 200 (light) | 600 (stroke) | 800 (text on fill) |
|------------|-----------|-------------|--------------|-------------------|
| c-purple   | #EEEDFE   | #AFA9EC     | #534AB7      | #3C3489           |
| c-teal     | #E1F5EE   | #5DCAA5     | #0F6E56      | #085041           |
| c-coral    | #FAECE7   | #F0997B     | #993C1D      | #712B13           |
| c-pink     | #FBEAF0   | #ED93B1     | #993556      | #72243E           |
| c-gray     | #F1EFE8   | #B4B2A9     | #5F5E5A      | #444441           |
| c-blue     | #E6F1FB   | #85B7EB     | #185FA5      | #0C447C           |
| c-green    | #EAF3DE   | #97C459     | #3B6D11      | #27500A           |
| c-amber    | #FAEEDA   | #EF9F27     | #854F0B      | #633806           |
| c-red      | #FCEBEB   | #F09595     | #A32D2D      | #791F1F           |

Color assignment rules:
- Color encodes MEANING, not sequence. Don't cycle through colors.
- Group nodes by category — same type = same color.
- Use gray for neutral/structural nodes (start, end, generic steps).
- Limit to 2-3 colors per diagram. More = visual noise.
- Reserve red/green/amber for actual error/success/warning semantics.

Light mode: 50 fill + 600 stroke + 800 text.
Dark mode: 800 fill + 200 stroke + 100 text.
The `c-{ramp}` classes handle this automatically.

## Dark Mode
Mandatory. Every visual must work in both light and dark themes.
- In SVG: use `c-{ramp}` classes for colored nodes. They auto-adapt.
- In SVG: every `<text>` must have a class (`t`, `ts`, `th`). Never omit fill.
- In HTML: always use CSS variables for text and background colors.
- Never hardcode colors like `#333` or `#fff` for text.
- Mental test: if the background were near-black, would every element be visible?

## Arrow Marker
Include this `<defs>` block at the start of every SVG:

```xml
<defs>
  <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5"
          markerWidth="6" markerHeight="6" orient="auto-start-reverse">
    <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke"
          stroke-width="1.5" stroke-linecap="round"
          stroke-linejoin="round"/>
  </marker>
</defs>
```

Use `marker-end="url(#arrow)"` on lines. The head inherits line color
via `context-stroke`.

## Pre-built CSS Classes (injected by the WebUI)
- `class="t"` = sans 14px, primary color
- `class="ts"` = sans 12px, secondary color
- `class="th"` = sans 14px, font-weight 500 (bold)
- `class="box"` = neutral rect (bg-secondary fill, border stroke)
- `class="arr"` = arrow line (1.5px, open chevron head)
- `class="node"` = clickable group (cursor pointer, hover dim effect)

## Stroke Width
Use 0.5px strokes for borders and edges.

## Interaction
Make nodes clickable by default:
```xml
<g class="node" onclick="sendPrompt('Tell me about X')">
  <rect .../>
  <text .../>
</g>
```

## Hard Rules
- No gradients, drop shadows, blur, glow, or neon effects.
- No comments in SVG/HTML (waste tokens).
- No emoji — use CSS shapes or SVG paths.
- No rotated text.
- No font-size below 11px.
- No dark/colored backgrounds on outer containers.
- Connector paths need `fill="none"` (SVG defaults to fill:black).
- All text in boxes needs `dominant-baseline="central"` and proper y centering.
````

### Module: diagram

**File:** `diagram.md`

````markdown
# Diagram guidelines

## Diagram Types

Pick the right type based on INTENT, not subject matter.

### Flowchart
For sequential processes, cause-and-effect, decision trees.
- Trigger: "walk me through", "what are the steps", "what's the flow"
- Layout: single-direction flows (top-down or left-right)
- Max 4-5 nodes per diagram
- 60px minimum vertical spacing between boxes
- 24px padding inside boxes, 12px between text and edges

### Structural Diagram
For containment — things inside other things.
- Trigger: "what's the architecture", "how is this organized"
- Large rounded rects (rx=20) as containers, smaller rects inside
- 20px minimum padding inside every container
- Max 2-3 nesting levels at 680px width
- Use different color ramps for nested levels

### Illustrative Diagram
For building INTUITION. Draw the mechanism, not a diagram about it.
- Trigger: "how does X actually work", "explain X", "I don't get X"
- Physical subjects: simplified cross-sections, cutaways
- Abstract subjects: spatial metaphors (hash table = row of buckets,
  attention = fan of lines with varying thickness)
- Color encodes intensity (warm = active, cool = dormant)
- Shapes are freeform — use path, ellipse, circle, polygon
- One gradient allowed per diagram, only for continuous physical
  properties (temperature, pressure)
- Prefer interactive over static when the subject has a control

## Layout Rules

### Box Sizing
- Single-line box: 44px tall, title only
- Two-line box: 56px tall, title + subtitle
- Width: max(title_chars × 8, subtitle_chars × 7) + 24px minimum

### Spacing
- 60px minimum between connected boxes vertically
- 20px minimum gap between boxes in the same row
- 10px gap between arrowhead and box edge
- Two-line boxes: 22px between title and subtitle baselines

### Tier Packing (horizontal rows)
Compute total width BEFORE placing boxes:
- 4 boxes × 130px + 3 gaps × 20px = 580px (fits in 600px safe area)
- 4 boxes × 160px + 3 gaps × 20px = 700px (OVERFLOW — shrink boxes)

### Arrow Routing
- A line from A to B must NOT cross any other box or label.
- If direct path crosses something, route around with L-bend:
  `<path d="M x1 y1 L x1 ymid L x2 ymid L x2 y2" fill="none"/>`
- No standalone arrow labels — if meaning isn't obvious from source
  and target, put it in the box subtitle or prose.

### ViewBox Height
After layout, find max_y (bottom-most element including text baselines).
Set viewBox height = max_y + 40px buffer. Don't guess.

## Common Failure Modes
Check each before finalizing:

1. **Text overflow:** label wider than its box. Always compute width first.
2. **Arrow through box:** line crosses an unrelated node. Route around it.
3. **ViewBox too short:** bottom content clipped. Compute height from actual elements.
4. **Dark mode invisible:** hardcoded text color on transparent background.
5. **Overlapping boxes:** didn't compute tier packing before placing.
6. **text-anchor="end" near x=0:** text extends left past viewBox boundary.

## Flowchart Component Patterns

Single-line node (44px):
```xml
<g class="node c-blue" onclick="sendPrompt('Tell me about X')">
  <rect x="100" y="20" width="180" height="44" rx="8" stroke-width="0.5"/>
  <text class="th" x="190" y="42" text-anchor="middle"
        dominant-baseline="central">Label</text>
</g>
```

Two-line node (56px):
```xml
<g class="node c-blue" onclick="sendPrompt('Tell me about X')">
  <rect x="100" y="20" width="200" height="56" rx="8" stroke-width="0.5"/>
  <text class="th" x="200" y="38" text-anchor="middle"
        dominant-baseline="central">Title</text>
  <text class="ts" x="200" y="56" text-anchor="middle"
        dominant-baseline="central">Short subtitle</text>
</g>
```

Connector (no label):
```xml
<line x1="200" y1="76" x2="200" y2="120" class="arr"
      marker-end="url(#arrow)"/>
```

## Illustrative Diagram Specifics

### What Changes From Flowcharts
- Shapes are freeform: `<path>`, `<ellipse>`, `<circle>`, `<polygon>`
- Layout follows the subject's geometry, not a grid
- Color encodes intensity, not category
- Layering and overlap are encouraged for shapes (NOT for text)
- Small shape indicators allowed: triangles for flames, circles for
  bubbles, wavy lines for steam

### Label Placement
- Place labels OUTSIDE the drawn object with leader lines when possible
- Leader lines: 0.5px dashed, `var(--color-text-tertiary)` stroke
- Pick ONE side for labels — no room for both at 680px
- Reserve 140px+ of margin on the label side
- Default to right-side labels with `text-anchor="start"`

### Composition Order
1. Main object silhouette (largest shape, centered)
2. Internal structure (chambers, pipes, components)
3. External connections (arrows, inputs, outputs)
4. State indicators last (color fills, animation, small markers)

### Physical Color Scenes
For scenes with natural colors (sky, water, grass, materials):
use ALL hardcoded hex — never mix with `c-*` theme classes.
The scene should not invert in dark mode.

## Multi-Diagram Responses
For complex topics, use multiple render_visual calls with prose between.
Never stack two calls back-to-back without text in between.

## Cycles and Loops
DO NOT draw cycles as rings — stages around a circle always cause
collisions. Instead:
- Linear layout with a curved return arrow, OR
- HTML stepper (one panel per stage, Next wraps to first)
````

### Module: chart

**File:** `chart.md`

````markdown
# Chart guidelines

Use Chart.js for data visualization. Chart.js 4.4.1 is bundled locally.

## Setup
```html
<script src="/js/vendor/chart.umd.js"></script>

<canvas id="myChart" style="max-height:400px"></canvas>
<script>
const ctx = document.getElementById('myChart');
new Chart(ctx, {
   type: 'bar',
   data: { ... },
   options: {
      responsive: true,
      plugins: { legend: { position: 'top' } }
   }
});
</script>
```

## Theming
Read CSS variables and apply to Chart.js:
```javascript
const style = getComputedStyle(document.documentElement);
const textColor = style.getPropertyValue('--color-text-primary').trim();
const gridColor = style.getPropertyValue('--color-border').trim();

Chart.defaults.color = textColor;
Chart.defaults.borderColor = gridColor;
```

## Color Mapping for Datasets
Use color ramp 600 stops for dataset colors:
- Dataset 1: #534AB7 (purple-600)
- Dataset 2: #0F6E56 (teal-600)
- Dataset 3: #D85A30 (coral-600)
- Dataset 4: #185FA5 (blue-600)

## Chart Type Selection
- Comparison across categories → bar (horizontal if long labels)
- Trend over time → line
- Part of whole → doughnut (not pie — doughnut is more readable)
- Correlation → scatter
- Multi-variable comparison → radar
````

### Module: interactive

**File:** `interactive.md`

````markdown
# Interactive widget guidelines

Use HTML type for widgets with user controls. JavaScript executes after
the content is fully streamed/rendered.

## Controls
- Sliders: `<input type="range">` for continuous values
- Toggles: checkbox styled as switch for on/off states
- Buttons: for discrete actions or stepping through states
- Dropdowns: `<select>` for choosing between modes

## State Management
Use JavaScript variables. No localStorage or sessionStorage (not
available in sandboxed iframe).

```javascript
let state = { temperature: 50, heating: true };
function update() { /* re-render based on state */ }
```

## Inline SVG in HTML
For interactive diagrams, embed SVG directly in the HTML:

```html
<div>
  <svg width="100%" viewBox="0 0 680 400">
    <!-- diagram content, same rules as static SVG -->
  </svg>
  <div style="display:flex; gap:12px; margin-top:8px">
    <label>Temperature
      <input type="range" min="0" max="100" value="50"
             oninput="setTemp(this.value)">
    </label>
  </div>
</div>
```

## Animation Rules
- CSS `@keyframes` only. Animate `transform` and `opacity` only.
- Keep loops under 2 seconds.
- Always wrap in `@media (prefers-reduced-motion: no-preference)`.
- Animations show behavior (flow, rotation), not decoration.

## sendPrompt() for Drill-Down
Use clickable elements to trigger follow-up conversations:
```html
<button onclick="sendPrompt('Explain the cooling system in detail')">
  Learn more
</button>
```

## Stepper Pattern (for cycles/processes)
When the subject has stages that loop, build a stepper instead of
a ring diagram:

```html
<div id="stepper">
  <div id="stage-content"><!-- filled by JS --></div>
  <div style="display:flex; gap:8px; margin-top:12px">
    <button onclick="prev()">Back</button>
    <span id="dots"></span>
    <button onclick="next()">Next</button>
  </div>
</div>
```

Next on the last stage wraps to the first — that IS the loop.

## External Libraries
- Chart.js 4.4.1: bundled locally at `/js/vendor/chart.umd.js` (MIT license, offline-first)
- D3, Three.js: not bundled — would require CDN access (not currently available in sandboxed iframe)
````

---

## Voice-First Considerations

Visual output is inherently a WebUI feature. DAWN cannot speak an SVG diagram over TTS. The interaction pattern for voice-first users is:

- The LLM explains the concept verbally through TTS (the spoken explanation is complete on its own).
- Simultaneously, the LLM invokes `render_visual` to place a diagram in the WebUI conversation.
- The spoken response can reference the visual: "I've put together a diagram on the WebUI if you want to see the architecture laid out."
- On satellites without displays, the visual is simply available in the WebUI — the spoken explanation stands alone.

This is the same pattern as image generation: voice gets the explanation, WebUI gets the visual. The two complement each other but neither depends on the other.

---

## Security

The sandboxed iframe is the primary security boundary. The `sandbox="allow-scripts"` attribute permits JavaScript execution within the iframe but blocks all of the following:

- Access to the parent page's DOM
- Access to cookies, localStorage, or sessionStorage of the parent origin
- Navigation of the top-level browsing context
- Form submission
- Plugin content
- Pointer lock and orientation lock

The only communication channel is `postMessage`, which the parent WebUI validates (accepting only `dawn_prompt` messages with a text string).

Since the LLM is running locally and the SVG/HTML is generated by DAWN's own LLM (not received from an external source), the attack surface is limited to prompt injection scenarios where a malicious input could cause the LLM to generate harmful code. The iframe sandbox mitigates this risk: even if the LLM generates JavaScript that attempts to exfiltrate data, it cannot access anything outside the iframe.

---

## Conversation History

Visuals are stored in the conversation history as tool call results, the same way any other tool result is stored. The conversation record includes the title, type, and generated code. When the user scrolls back through the conversation or the WebUI reloads, visuals are re-rendered from the stored code.

This means visuals are persistent and reproducible. The code is the source of truth — there are no external dependencies or cached image files to manage.

---

## Implementation Plan

### Phase 1: Static SVG — SHIPPED

- ✅ `render_visual_load_guidelines` and `render_visual` registered in tool registry
- ✅ `_core.md` and `diagram.md` guideline files written
- ✅ Sandboxed iframe renderer with theme CSS injection
- ✅ Tested with flowcharts and architecture diagrams (Claude Sonnet 4.6)

### Phase 2: Interactive HTML — SHIPPED

- ✅ `html` type enabled in tool schema
- ✅ `interactive.md`, `chart.md`, `art.md`, `mockup.md` guideline files written
- ✅ `sendPrompt()` bridge with WeakSet source validation
- ✅ Chart.js bundled locally (`www/js/vendor/chart.umd.js`, MIT)
- ✅ History persistence (server-side `pending_visual` + client interleaved save)
- ✅ Inline visual positioning (text → diagram → text within one message)
- ✅ Download button (SVG/HTML export, matches copy-btn pattern)
- ✅ Tested with charts, interactive widgets, mockups, logos

### Phase 3: Refinement — PARTIAL

- ✅ 9-color ramp design system with automatic light/dark mode
- ✅ Download button for SVG/HTML export
- ✅ Guideline iteration based on observed failure modes
- ✅ Generation progress placeholder (pulsing dot + elapsed timer)
- ⏸ Model quality testing across quantization levels (testing task)

### Implementation Notes (deviations from original design)

- **CSP**: Added `'unsafe-inline'` to `script-src` for srcdoc iframe interactivity. Original design proposed blob URLs but srcdoc was simpler and works with the sandbox.
- **Persistence**: Client-side streaming save interleaves visual content at the correct text position. Server clears `pending_visual` on save (doesn't append — client owns ordering).
- **Inline positioning**: Streaming splits text at tool call boundary. `preVisualContent` tracks pre-visual text, new `.text` div created post-visual.
- **Vendor scripts**: Chart.js bundled at `www/js/vendor/chart.umd.js` (MIT, 4.4.1). Inlined into srcdoc (fetched once, cached) since srcdoc iframes can't resolve relative URLs.
- **Instruction loader**: Guidelines loaded via shared `instruction_loader.c` module (not direct fopen). Supports path traversal sanitization, pre-scan sizing, and reuse by future two-step tools.
- **Generation progress indicator**: When `render_visual` tool_use starts, the server sends a `visual_progress_start` WebSocket message. The frontend shows an inline placeholder card with a pulsing accent dot and a client-side elapsed timer that ticks every second. When the complete visual arrives, the placeholder is replaced in-place with the rendered iframe (CSS fade-in animation). If the tool fails, `finalizeStream` marks the placeholder as "Visual generation failed." The placeholder splits the streaming entry the same way the visual handler does, avoiding a double-split race condition on arrival.

### Visual Progress: API Limitation (Claude tool arg streaming)

We attempted to show real-time progress (byte count, progress bar, title extraction) during visual generation. The server detects `render_visual` tool_use in `content_block_start` and can observe `input_json_delta` events accumulating the SVG code. However, **Claude's API does not stream tool arguments smoothly** — the `input_json_delta` events arrive in bursts, with 30-40 seconds of silence followed by the entire SVG dumped in the last 1-2 seconds. This means:

- **Progress bar**: Sits at 0% for the entire generation, then jumps to 100% at the end. Useless.
- **Byte counter**: Same problem — shows 0 KB for 30s, then 8 KB in the last second.
- **Title extraction**: The title is inside the tool args JSON (`details` field, double-encoded). It arrives in the same final burst as the code, so the title updates ~1 second before the visual completes — not enough lead time to matter.

All three features were implemented, tested, and removed after confirming the API behavior. The pulsing dot + elapsed timer is the honest UX for this constraint — it tells the user "still generating" without pretending to know how far along it is. If Anthropic changes the tool arg streaming behavior in the future, the server-side hooks (`visual_progress_active` flag, `webui_send_session_json` helper) are in place to re-add real-time progress.

---

## Open Questions — Resolved

1. **Streaming rendering:** Investigated. Claude's API does not stream tool arguments smoothly — they arrive in a burst at completion. A generation progress placeholder (pulsing dot + timer) bridges the gap instead. Full streaming SVG rendering is not feasible with current API behavior.
2. **CDN access:** Resolved — Chart.js bundled locally. No CDN dependency. Offline-first.
3. **Token cost:** Not formally measured. Guideline modules are ~2-4K tokens each. Well within context budget.
4. **Quality floor:** Claude Sonnet 4.6 produces excellent SVG. Not tested with smaller models.
5. **Export:** Shipped — download button for SVG/HTML files.
6. **Guideline evolution:** In place — markdown files on disk, edit without recompiling.
