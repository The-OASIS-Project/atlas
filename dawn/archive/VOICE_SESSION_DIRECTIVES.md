# Voice-Session Prompt Directives

**Status:** Shipped July 11, 2026 (DAWN, pending commit — 17 files). Live-verified in the browser across every voice/text + TTS toggle combination.

Guidance-injection feature that tells the LLM, *only on voice surfaces*, how to shape its output for speech and how to interpret speech-transcribed input — without hard word/sentence caps. Three global, WebUI-editable directives, each with a compile-time built-in default.

## Motivation

DAWN's voice surfaces are full-duplex: input arrives via ASR (Whisper / satellite-local ASR) and output is read aloud by Piper TTS. Two problems had no surface-specific steering:

1. **Output shape** — spoken replies are *heard*, not read. Markdown, bullet/numbered lists, tables, code blocks, URLs, and long passages translate badly to speech. The only prior steering was one generic line in `NATIVE_TOOLS_RULES` ("Keep responses concise and conversational.") applied to *every* surface, including WebUI text chat where verbosity + formatting are fine.
2. **Input reliability** — ASR mis-transcribes homophones and similar-sounding words (names, technical terms). The model had no hint to interpret such input from context.

## The three directives

| Config key | Section / WebUI panel | Applies to | Purpose |
|---|---|---|---|
| `voice_directive` | `[tts]` / Text-to-Speech | satellites (DAP2 Tier-1/2 + legacy DAP) **and** local mic | Spoken output: concise, speech-friendly, no visual-only formatting |
| `voice_directive_webui` | `[tts]` / Text-to-Speech | WebUI voice turns | Softer variant — the whole prose reply is read aloud, but the screen stays free for `render_visual`/images |
| `disambiguation_hint` | `[asr]` / Speech Recognition | any voice-**input** turn (satellites, local mic, WebUI voice) | Warns the model its input was speech-transcribed; interpret homophone/similar-sounding mishearings from context. Generalized wording — **no hardcoded names** (per the codebase no-real-names rule) |

All three are **global** config (not per-user), always-on, editable in the WebUI from day one. Empty field ⇒ built-in default. Defaults are macros in `include/dawn.h` (`DEFAULT_VOICE_OUTPUT_DIRECTIVE[_WEBUI]`, `DEFAULT_ASR_DISAMBIGUATION_HINT`); effective-value accessors (`voice_directive_effective()` etc. in `llm_command_parser.c`) resolve config-or-default.

## Correctness facts (verified in code)

- On WebUI voice, the **entire** prose reply is synthesized sentence-by-sentence and read aloud (`webui_sentence_audio_callback`, gated only on `conn->tts_enabled`). There is **no** way to mark some prose spoken and other prose screen-only — so the WebUI directive says "your whole spoken reply is read aloud; use the `render_visual` tool / images for anything better seen than heard." The `render_visual`/`<dawn-visual>` tool output (routed to a `"visual"` transcript entry, never entering TTS) is the *only* silent, screen-only channel.

## Injection — three seams

The base system instructions (`get_remote_command_prompt`) are shared by DAP2/WebUI/messaging, so the directives can't live there (WebUI-text + messaging must not get them). They are injected per surface:

1. **Local mic** (`SESSION_TYPE_LOCAL`) — baked into the static prompt in `initialize_command_prompt()` (`llm_command_parser.c`). Local-only buffer; rebuilt on `invalidate_system_instructions()`.
2. **Satellites** (`SESSION_TYPE_DAP2` / `DAP`) — appended to the **cacheable stable prefix** in the per-turn builder `dawn_build_prompt()` (`webui_auth_helpers.c`), via a new OOM-safe `append_block_directive()` helper. Constant per surface, so stable placement is cache-safe (rides alongside the existing `Room=` / messaging-channel context).
3. **WebUI** — appended to the **volatile (non-cached) segment**, because its gates vary per turn:
   - `voice_directive_webui` gated on `conn->tts_enabled` (spoken **output**).
   - `disambiguation_hint` gated on `session->input_was_voice` (voice **input**).
   These are **independent** gates: a voice-in + TTS-off turn gets only the ASR hint. Volatile placement keeps the Anthropic prompt cache warm across mode toggles — same reasoning as the SMS `channel_hint`.

## The per-turn voice-input flag (`session->input_was_voice`)

WebUI is the only mixed surface (voice or typed per turn). A new `_Atomic bool input_was_voice` on `session_t` gates the ASR hint. **Key discipline (learned the hard way):** stamp it on the **worker thread, immediately before the synchronous build** — the same place `tts_enabled` is read (`webui_text_processing.c`). It is carried per-turn in the `text_work_t` item (a `bool` param on `webui_process_text_input[_with_vision]`); the push-to-talk audio path stamps it directly before its own synchronous dispatch.

An earlier version set the flag at the *entry points* (typed at `handle_text_message` on the **LWS thread**, voice on the always-on/audio worker). With always-on voice enabled, a typed turn's `false` reset could be clobbered by a concurrent always-on voice turn, so the ASR hint stuck after switching back to typing. Stamping on the worker thread adjacent to the build closes that window. **Rule: gate flags read by the prompt builder should be set on the same thread that runs the build, right before it — not at the entry point.**

## Config plumbing (global-string 7 touchpoints)

Standard DAWN global-string pattern (template = `persona.description`): struct fields (`dawn_config.h`) → `parse_tts`/`parse_asr` (+ `known_keys`) → defaults (empty) → `config_env.c` (`config_write_toml` + `config_to_json` + env overrides + `--dump-config` printers) → `webui_config.c` `apply_config_from_json` → `dawn.toml.example` → `schema.js` (three `textarea` fields + `SCHEMA_VERSION` bump). A `[tts]`/`[asr]` directive edit runs `invalidate_system_instructions()` + `session_manager_refresh_all_prompts()` (gated by `local_prompt_directives_changed`) so the cached local-mic prompt applies live without a restart; satellites/WebUI rebuild per turn and pick up edits automatically.

## Security note — TOML `'''` sanitization

The multi-line writer `write_toml_string_multiline()` (`config_env.c`) **sanitizes** any run of apostrophes down to 2, so a stored value can never contain `'''`. Reasons: (a) `'''` terminates / breaks out of a TOML multi-line literal, letting a crafted value forge new keys/sections on the next load (CWE-91); (b) DAWN's TOML reader (tomlc99) **mis-parses `'''` even inside a double-quoted basic string** — so there is *no* safe TOML representation of a value containing `'''`. `'''` is meaningless in a prose directive, so collapsing it is lossless and guarantees the config always round-trips. `persona.description` was routed through the same hardened writer (it shared the latent breakout). Admin-only surface, so LOW severity, fixed on merit.

## Review & verification

- Three-agent review (architecture / security / embedded-efficiency): 0 critical/high. Applied: cross-reference comments at the two injection sites, `local_prompt_directives_changed` rename, worker-thread-serialization notes, and the `'''` sanitizer.
- Config round-trip verified via `--dump-config` (custom values, empty→built-in default, `'''` sanitizer micro-tested).
- Live-verified in the browser: Friday correctly reports ASR/TTS directive presence for every combination — local mic = both; WebUI neither / both / typed-after-voice all correct.

## Non-goals

- `NATIVE_TOOLS_RULES` unchanged (generic "concise" stays; these layer on for voice only).
- Messaging + WebUI-text turns untouched (rich text; SMS has its own `channel_hint`).
- No enable/disable toggle (always-on; empty field = built-in default).
