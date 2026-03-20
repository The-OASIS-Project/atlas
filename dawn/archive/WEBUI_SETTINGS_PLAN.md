> **STATUS: COMPLETE** - Archived 2025-12-25

# WebUI Settings Panel Implementation Plan

## Overview
Add a slide-out settings panel to the DAWN WebUI that allows viewing and modifying all configuration options from `dawn.toml` and `secrets.toml`, with changes saved directly to the config files.

## Requirements
- Display all configurable settings organized by section
- Save changes to the currently loaded config file
- API keys: password fields with show/hide toggle, indicator if set (never display saved values)
- Settings requiring restart: show warning badge
- Modern UI: slide-out panel with glass-morphism effect

## Files to Modify

### Server-Side (C)
1. **`src/config/config_env.c`** - Add new functions:
   - `config_to_json()` - serialize g_config to JSON for browser
   - `config_write_toml()` - write config to TOML file (adapt from `config_dump_toml()`)
   - `secrets_write_toml()` - write secrets to TOML file

2. **`src/config/config_parser.c`** - Add:
   - `config_backup_file()` - create .bak before modifying
   - `config_from_json()` - update g_config from JSON payload

3. **`include/config/config_env.h`** - Declare new functions

4. **`src/webui/webui_server.c`** - Add message handlers in `handle_json_message()`:
   - `get_config` → `handle_get_config()` - send current config as JSON
   - `set_config` → `handle_set_config()` - validate, update g_config, write file
   - `set_secrets` → `handle_set_secrets()` - update and write secrets

### Client-Side (HTML/CSS/JS)
5. **`www/index.html`** - Add:
   - Settings gear button in header
   - Settings panel overlay and slide-out panel structure
   - Secrets section with password inputs

6. **`www/css/dawn.css`** - Add:
   - Glass-morphism panel styles (backdrop-filter, blur)
   - Restart badge styles
   - Secret input with show/hide button
   - Responsive adjustments

7. **`www/js/dawn.js`** - Add:
   - Settings panel open/close functions
   - `requestConfig()` / `handleGetConfigResponse()`
   - Dynamic settings section rendering from schema
   - `collectSettingsChanges()` / `saveSettings()`
   - Secrets handling with status indicators
   - Message handlers for `get_config_response`, `set_config_response`

## Settings Requiring Restart Badge
- `asr.model`, `asr.models_path`
- `network.enabled`, `network.host`, `network.port`, `network.workers`
- `webui.port`, `webui.max_clients`, `webui.workers`, `webui.https`, `webui.ssl_*`

## WebSocket Message Format

### Request Config
```json
{ "type": "get_config" }
```

### Config Response
```json
{
  "type": "get_config_response",
  "payload": {
    "config_path": "/path/to/dawn.toml",
    "secrets_path": "/path/to/secrets.toml",
    "config": { /* full config object */ },
    "secrets": {
      "openai_api_key": { "is_set": true },
      "claude_api_key": { "is_set": false },
      "mqtt_username": { "is_set": false },
      "mqtt_password": { "is_set": false }
    },
    "requires_restart": ["asr.model", "network.port", ...]
  }
}
```

### Save Config
```json
{
  "type": "set_config",
  "payload": { /* partial config with changed values */ }
}
```

### Save Response
```json
{
  "type": "set_config_response",
  "payload": {
    "success": true,
    "requires_restart_fields": ["network.port"]
  }
}
```

## Implementation Order

1. **Server: Config serialization** (~2h)
   - `config_to_json()` in config_env.c
   - `config_write_toml()` adapting existing `config_dump_toml()`
   - `config_backup_file()` in config_parser.c

2. **Server: WebSocket handlers** (~1h)
   - `handle_get_config()`
   - `handle_set_config()` with validation
   - `handle_set_secrets()`

3. **Frontend: HTML structure** (~30m)
   - Settings button in header
   - Panel overlay and slide-out structure
   - Secrets section

4. **Frontend: CSS styling** (~1h)
   - Glass-morphism panel
   - Form controls and badges
   - Responsive layout

5. **Frontend: JavaScript** (~2h)
   - Panel open/close
   - Config request/render
   - Settings schema for dynamic form generation
   - Save handlers
   - Secrets handling

## Key Design Decisions

1. **Backup Strategy**: Create `.bak` file before any write operation
2. **Partial Updates**: Only send changed fields, merge with existing config
3. **Live vs Restart**: Apply what we can immediately, badge others
4. **Secrets Security**: Never send actual values to browser, only `is_set` status
5. **Glass-morphism**: `backdrop-filter: blur(20px)` with semi-transparent background
