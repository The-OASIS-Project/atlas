# Conversation Export to JSON — Design Document

**Created**: March 3, 2026
**Status**: Reviewed — Architecture + UI approved with revisions

---

## Goal

Allow users to export any conversation from history as a JSON file for backup, sharing, debugging, or analysis. Single conversation export via a download button in the history panel.

## Export Format

```json
{
   "exported_at": "2026-03-03T10:30:00Z",
   "dawn_version": "1.2.3-abc1234",
   "conversation": {
      "id": 42,
      "title": "Conversation Title",
      "created_at": "2026-03-01T14:00:00Z",
      "updated_at": "2026-03-01T14:35:00Z",
      "message_count": 12,
      "origin": "webui",
      "is_private": false,
      "continued_from": null,
      "llm_settings": {
         "llm_type": "cloud",
         "cloud_provider": "claude",
         "model": "claude-sonnet-4-20250514",
         "tools_mode": "native",
         "thinking_mode": "auto"
      }
   },
   "messages": [
      {
         "role": "user",
         "content": "Hello, can you help me with something?",
         "timestamp": "2026-03-01T14:00:00Z"
      },
      {
         "role": "assistant",
         "content": "Of course! What do you need help with?",
         "timestamp": "2026-03-01T14:00:05Z"
      }
   ]
}
```

### Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Images | Metadata note only (no base64) | Keep exports small and portable |
| Thinking content | Not in DB — excluded | `conversation_message_t` has `role` + `content` only; thinking is ephemeral |
| Timestamps | ISO 8601 (UTC) | Standard, parseable format |
| File naming | `{sanitized_title}-{YYYY-MM-DD}-{conv_id}.json` | Human-readable, sortable, collision-free |
| Transport | WebSocket JSON message | Consistent with all other history operations |
| Auth | Standard `conn_require_auth()` | Users can only export their own conversations |
| LLM settings | Full `llm_settings` object | Already loaded by `conv_db_get()`, useful for debugging/analysis |
| Export cap | Configurable max messages (default 5000) | Prevents memory pressure on very large conversations |

## Configuration

### dawn.toml

```toml
[webui]
export_max_messages = 5000    # Max messages per export (0 = unlimited, default: 5000)
```

New field in `webui_config_t`:
```c
int export_max_messages;  /* Max messages per export (0 = unlimited, default: 5000) */
```

### WebUI Settings

Added to the WebUI section of the settings schema as an advanced field:
```
Export Message Limit: [5000] (number input, min: 0, hint: "Max messages per export (0 = unlimited)")
```

## Architecture

### Data Flow

```
Browser                          Server (webui_history.c)
  │                                  │
  │  { type: "export_conversation",  │
  │    payload: { conversation_id }} │
  │ ─────────────────────────────►   │
  │                                  │ 1. conn_require_auth()
  │                                  │ 2. conv_db_get(conv_id, user_id)
  │                                  │ 3. Check message_count vs export_max_messages
  │                                  │ 4. conv_db_get_messages(conv_id, user_id)
  │                                  │ 5. Build JSON export object
  │                                  │
  │  { type: "export_conversation_  │
  │    response",                    │
  │    payload: { success, data }}   │
  │ ◄─────────────────────────────   │
  │                                  │
  │  Blob → URL.createObjectURL()   │
  │  → <a download="..."> click     │
  └──────────────────────────────────┘
```

### Server Side — `handle_export_conversation()`

New function in `src/webui/webui_history.c`. Follows the exact same pattern as `handle_load_conversation()`:

1. `conn_require_auth()` gate
2. Extract `conversation_id` from payload
3. `conv_db_get()` to fetch metadata (title, timestamps, origin, model, etc.)
4. Check `conv.message_count` against `export_max_messages` config; return error if exceeded
5. `conv_db_get_messages()` to fetch all messages (chronological order)
6. Build export JSON:
   - `exported_at`: current time as ISO 8601
   - `dawn_version`: from `version.h` (`DAWN_VERSION_STRING`)
   - `conversation`: metadata from `conversation_t` including full `llm_settings` object
   - `messages`: array from callback, each with `role`, `content`, ISO 8601 `timestamp`
7. Serialize and send via `send_json_response()`

A new `export_msg_callback()` builds message objects with ISO 8601 timestamps (vs. the Unix epoch integers used by `load_msg_callback()`).

**Helper**: `time_to_iso8601()` — formats `time_t` to `"2026-03-03T14:30:00Z"` string. Small static helper in `webui_history.c`.

### Client Side — `history.js`

1. **Export button**: Added to `renderConversationItem()` **after** the Delete button
   - Lucide `download` icon SVG (consistent with existing icon family)
   - `data-action="export"` attribute
   - Same 28×28px button style as rename/delete (44×44px on mobile)
   - Button order: **Rename | (Reassign, if admin+voice) | Delete | Export**
   - Preserves existing muscle memory for Rename and Delete positions
2. **Click handler**: Sends `export_conversation` WebSocket message with `conversation_id`
   - Disables button and adds `.exporting` class (prevents double-click)
3. **Response handler** (`handleExportConversationResponse`):
   - Re-enables button, removes `.exporting` class
   - Extracts `data` (the full export JSON object) from payload
   - Creates `Blob` with `application/json` MIME type
   - Generates filename: `{title}-{date}-{conv_id}.json`
     - Title sanitized: alphanumeric, hyphens, underscores only, max 50 chars
   - Creates temporary `<a>` element with `download` attribute, clicks it, revokes URL
   - Shows `DawnToast.show('Exported: filename.json', 'success')`
4. **Error handling**: Re-enables button, toast error with server message

### CSS — `history.css`

- `.export` button hover style (blue tint, matching rename)
- `.exporting` class: reduced opacity / pulse animation for loading state

## Files to Modify

| File | Changes |
|------|---------|
| `src/webui/webui_history.c` | Add `handle_export_conversation()`, `export_msg_callback()`, `time_to_iso8601()` |
| `include/webui/webui_internal.h` | Declare `handle_export_conversation()` |
| `src/webui/webui_server.c` | Add `"export_conversation"` to message dispatch |
| `include/config/dawn_config.h` | Add `export_max_messages` to `webui_config_t` |
| `src/config/config_defaults.c` | Default `export_max_messages = 5000` |
| `src/config/config_parser.c` | Parse `[webui] export_max_messages` |
| `www/js/ui/history.js` | Export button in render, click handler, response handler, download logic |
| `www/js/ui/settings/schema.js` | Add export_max_messages to WebUI settings |
| `www/css/components/history.css` | `.export` button hover style, `.exporting` loading state |

No new files needed. No schema changes. No new dependencies.

## Size Estimates

- `webui_history.c`: ~90 lines added (handler + callback + helper + cap check)
- `webui_internal.h`: 1 declaration
- `webui_server.c`: 3 lines (dispatch entry)
- `dawn_config.h`: 1 field
- `config_defaults.c`: 1 line
- `config_parser.c`: ~3 lines
- `history.js`: ~60 lines (button HTML, handler, download logic, loading state)
- `settings/schema.js`: ~8 lines
- `history.css`: ~15 lines (hover + loading styles)

**Total: ~180 lines across 9 files**

## Security Considerations

- Auth gate: `conn_require_auth()` — unauthenticated users cannot export
- Ownership: `conv_db_get()` enforces `user_id` check — users can only export their own conversations
- Filename sanitization: Client-side regex strips non-safe characters
- No file system writes: Export is purely in-memory JSON → WebSocket → browser Blob
- Export cap: Configurable message limit prevents memory exhaustion from pathologically large conversations
- Expected payload ceiling: 1000 messages at ~2KB average = ~2MB (well within WebSocket and memory bounds)

## Edge Cases

| Case | Handling |
|------|----------|
| Archived conversation | Export works — `conv_db_get_messages()` doesn't filter by archive status |
| Very long conversation (>5000 msgs) | Server returns error: "Conversation has N messages, exceeding export limit of 5000" |
| Conversation with image markers | Image markers appear as text in `content` field — metadata only, no base64 |
| Private conversation | Exported normally — privacy flag only affects memory extraction |
| Conversation with compaction summary | `continued_from` field in metadata; summary not included (it's in the parent) |
| Double-click | Button disabled during export request; re-enabled on response |

## Review History

- **Architecture review**: Approved. WebSocket transport confirmed correct (consistent with all history ops). Added: export cap, full LLM settings, conv_id in filename, version.h include verification.
- **UI review**: Approved. Changed: button order (Export after Delete to preserve muscle memory), Lucide download icon, loading/disabled state during export, 50-char title cap.
