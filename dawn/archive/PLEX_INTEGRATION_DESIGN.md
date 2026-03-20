# Plex Music Integration Design

**Goal:** Allow DAWN to stream music from a local Plex Media Server as an alternative to the local filesystem music library, selectable via a settings toggle.

**Last Updated:** March 2026

---

## Overview

DAWN plays music from a local directory (`paths.music_dir`) and optionally a Plex Media Server, both indexed into a unified SQLite database (`music.db`) by `music_scanner.c` / `music_db.c`. The audio pipeline is:

```
local file (or Plex temp file) → audio_decoder (FLAC/MP3/OGG) → resampler → Opus encoder → WebSocket broadcast
```

Plex integration adds an alternative **source** that replaces the first step for Plex tracks:

```
HTTP download from Plex → temp file → audio_decoder (FLAC/MP3/OGG) → resampler → Opus encoder → WebSocket broadcast
```

Everything downstream of `audio_decoder_open()` stays the same. The key components are:

1. A **unified music database** (`music_db.c`) with a `source` column (LOCAL=0, PLEX=1) and priority-based deduplication (local wins over Plex, case-insensitive matching)
2. A **music source abstraction** (`music_source.c`) defining source enum, path prefixes, and provider callbacks
3. A **Plex API client** (`plex_client.c`) for auth, track listing, and stream download
4. A **Plex DB sync layer** (`plex_db.c`) that periodically syncs Plex metadata into the unified DB via provider callbacks
5. An **HTTP download step** in `webui_music_start_playback()` that fetches Plex tracks to temp files before opening the decoder

Sources are **auto-detected**: if `paths.music_dir` exists, local scanning runs; if `music.plex.host` and `plex_token` are configured, Plex sync runs. No manual source selector is needed. The WebUI, satellite music panels, LLM music tool, and Opus streaming pipeline all operate on the unified `music_db_*()` API regardless of track source.

---

## Settings Integration

### Config Schema (`dawn.toml`)

```toml
[music]
scan_interval_minutes = 60    # Rescan interval for all sources (0 = disabled)

[music.plex]
host = "192.168.1.100"        # Plex server IP or hostname
port = 32400                  # Default Plex port
music_section_id = 0          # 0 = auto-discover from /library/sections
ssl = false                   # Use HTTPS for Plex API calls
ssl_verify = true             # Verify TLS certificates (set false for self-signed)
client_identifier = ""        # Auto-generated UUID on first run (persistent)
```

> **Note:** The original design included a `source = "local"` / `"plex"` selector. This was removed in favor of auto-detection: sources are enabled based on whether they are configured (local path exists, Plex host + token set). The `music_config_t.source` field has been removed from the codebase.

**Token storage:** The Plex token is stored in `secrets.toml` (NOT `dawn.toml`), following the same pattern as OpenAI/Claude/Gemini API keys:

```toml
# In secrets.toml
[secrets]
plex_token = "your-X-Plex-Token-here"
```

This ensures the token is never serialized to `dawn.toml`, never sent in `get_config_response` WebSocket messages, and is protected by `chmod 0600` via the existing `secrets_write_toml()` function.

### WebUI Settings Panel

The Music & Media settings section shows all fields without conditional visibility:

| Setting | Type | Notes |
|---------|------|-------|
| Library Rescan Interval | number | Applies to all configured sources |
| Plex Server | text | IP or hostname |
| Plex Port | number | Default 32400 |
| Plex Music Library | number | 0 = auto-detect |
| Use SSL | toggle | HTTPS for API calls |
| Verify SSL | toggle | TLS certificate verification |
| Plex Token | password | In Secrets panel |

> **Note:** The original design included a "Music Source" dropdown with `showWhen` conditional visibility. This was removed — sources are auto-detected based on configuration, so there is no selector. The `showWhen` schema feature was implemented but is no longer used by the music settings.

**Token in Secrets panel:** A "Plex Token" row in the Secrets panel alongside OpenAI/Claude/Gemini keys. Routes through `handle_set_secrets()` and `secrets_to_json_status()` (is_set flag only, never the actual value).

### LLM Tool Integration

The existing `music_tool.c` handles actions like `play`, `search`, `queue`, `stop`. These route through `webui_music.c` which calls `music_db_search()` for library queries and `audio_decoder_open()` for playback.

With the unified DB, all library queries go to `music_db_search()` (SQLite) regardless of track source. The dedup subquery ensures each unique track appears once, preferring local sources. At playback time, `plex:` prefixed paths trigger an HTTP download to a temp file before the decoder opens.

The LLM tool itself doesn't change — it still says "play Dark Side of the Moon" and the backend resolves it through the unified database. The tool's parameter schema (`action`, `query`, `artist`, `album`) maps naturally to the combined library.

---

## Plex API Reference

All endpoints live at `http://<host>:<port>/`. Plex returns XML by default; request JSON via `Accept: application/json` header.

### Authentication

Every request requires an `X-Plex-Token`, passed as either:
- Query parameter: `?X-Plex-Token=<token>`
- HTTP header: `X-Plex-Token: <token>`

**Required client identification headers** (all requests):

```
X-Plex-Client-Identifier: <persistent-uuid>   (REQUIRED — generate once, store in config)
X-Plex-Product: DAWN
X-Plex-Version: <dawn-version>
X-Plex-Platform: Linux
X-Plex-Device: <hostname>
```

The `X-Plex-Client-Identifier` must be generated with `/dev/urandom` or `uuid_generate()` and stored persistently in `[music.plex] client_identifier`. It is not a secret, just needs to be stable and unique. Auto-generate on first use if empty.

**Obtaining a token:**

| Method | How |
|--------|-----|
| **Manual** (recommended for MVP) | User copies token from Plex Web → Settings → Account → XML link URL contains `X-Plex-Token` |
| **MyPlex sign-in** | `POST https://plex.tv/api/v2/users/signin` with `login` + `password` form fields. Returns `{ "authToken": "..." }` |
| **Local auth bypass** | Plex Server Settings → Network → "List of IP addresses allowed without auth". No token needed if DAWN's IP is listed |

**Token notes:**
- Plex is transitioning to JWT-based tokens (2025+). Transient tokens may expire after 48 hours.
- For a persistent appliance, storing a long-lived token in `secrets.toml` (mode 0600) is safest.
- If token expires, API calls return HTTP 401. The daemon should send a `music_state` update with an `error` field ("Plex auth failed — update your token in Settings > Secrets"). The music player panel displays this as a persistent error banner (not a transient toast) with a "Go to Settings" link.

Reference: [Finding an authentication token](https://support.plex.tv/articles/204059436-finding-an-authentication-token-x-plex-token/)

### Server Discovery (GDM Protocol)

Plex servers advertise via the "G'Day Mate" (GDM) UDP protocol. This enables auto-discovery without manual IP configuration.

**Discovery request:**
- Send `M-SEARCH * HTTP/1.1\r\n\r\n` via UDP to port **32414**
- Target: multicast `239.0.0.250` (preferred) or subnet broadcast derived from the host's network interface (do NOT hardcode `192.168.1.255`)

**Discovery response** (from each Plex server on the network):

```
HTTP/1.1 200 OK\r
Content-Type: plex/media-server\r
Name: My Plex Server\r
Port: 32400\r
Resource-Identifier: a1b2c3d4-...\r
Version: 1.40.0.1234-abcdef\r
```

**Security:** GDM responses are unauthenticated UDP — any device on the LAN can spoof a response. Discovery results MUST be presented to the user for manual confirmation, never auto-applied. After user selects a discovered server, validate by calling `GET /identity` with the token to confirm the server's `machineIdentifier`.

**Implementation requirements:**
- Run in a detached thread or with `select()`/`poll()` (not blocking the main/LWS thread)
- 3-second timeout, reset `fromlen` before each `recvfrom()`, cap at 10 responses
- Parse header values with `snprintf`/`strncpy` and explicit size limits
- Validate `Port:` is integer in range 1-65535

Reference: [GDM protocol — python-plexapi](https://python-plexapi.readthedocs.io/en/latest/modules/gdm.html)

### Library Browsing Endpoints

**Step 1: Find the music library section**

```
GET /library/sections
Accept: application/json
X-Plex-Token: <token>
```

Response:
```json
{
  "MediaContainer": {
    "Directory": [
      {
        "key": "3",
        "title": "Music",
        "type": "artist"
      }
    ]
  }
}
```

Filter for `type == "artist"` to find music sections. Cache the `key` (e.g., `"3"`).

**Step 2: List artists**

```
GET /library/sections/{section_id}/all?type=8
X-Plex-Container-Start=0
X-Plex-Container-Size=50
Accept: application/json
```

Type codes: `8` = artist, `9` = album, `10` = track.

Response:
```json
{
  "MediaContainer": {
    "size": 150,
    "totalSize": 150,
    "Directory": [
      {
        "ratingKey": "12345",
        "key": "/library/metadata/12345/children",
        "type": "artist",
        "title": "Pink Floyd",
        "thumb": "/library/metadata/12345/thumb/1234567890"
      }
    ]
  }
}
```

Supports pagination (`X-Plex-Container-Start`, `X-Plex-Container-Size`) and filtering (`?title=Pink`).

**Always paginate** with a reasonable page size (50-100 items). A Plex library with 10,000+ artists produces multi-megabyte JSON responses that spike memory if parsed all at once. Cap the API response buffer at 2MB (`CURL_BUFFER_MAX_PLEX_API`), analogous to the existing `CURL_BUFFER_MAX_WEB_SEARCH` pattern.

**Step 3: List albums for an artist**

```
GET /library/metadata/{artist_ratingKey}/children
Accept: application/json
```

Response:
```json
{
  "MediaContainer": {
    "Directory": [
      {
        "ratingKey": "12346",
        "key": "/library/metadata/12346/children",
        "type": "album",
        "title": "The Dark Side of the Moon",
        "year": 1973,
        "parentTitle": "Pink Floyd"
      }
    ]
  }
}
```

**Step 4: List tracks for an album**

```
GET /library/metadata/{album_ratingKey}/children
Accept: application/json
```

Response (critical — contains stream URL data):
```json
{
  "MediaContainer": {
    "Metadata": [
      {
        "ratingKey": "12347",
        "type": "track",
        "title": "Time",
        "parentTitle": "The Dark Side of the Moon",
        "grandparentTitle": "Pink Floyd",
        "index": 4,
        "duration": 413000,
        "Media": [
          {
            "container": "flac",
            "audioCodec": "flac",
            "audioChannels": 2,
            "bitrate": 1024,
            "Part": [
              {
                "key": "/library/parts/9877/1234567890/file.flac",
                "container": "flac",
                "duration": 413000,
                "size": 52428800
              }
            ]
          }
        ]
      }
    ]
  }
}
```

**Step 5: All tracks for an artist (flat list)**

```
GET /library/metadata/{artist_ratingKey}/allLeaves
X-Plex-Container-Size=200
Accept: application/json
```

Returns all tracks under an artist without drilling through albums. Useful for "play all by artist." Always paginate with `X-Plex-Container-Size` to avoid unbounded responses.

Reference: [Plex Media Server URL Commands](https://support.plex.tv/articles/201638786-plex-media-server-url-commands/) | [plexapi.dev Library API](https://plexapi.dev/api-reference/library/get-all-libraries)

### Playlist Browsing

```
GET /playlists
Accept: application/json
X-Plex-Token: <token>
```

Get playlist items:
```
GET /playlists/{playlist_id}/items
Accept: application/json
X-Plex-Token: <token>
```

Returns `Track` elements with full `Media`/`Part` data for music playlists. Maps cleanly into existing `music_library_response` format.

### Search

```
GET /hubs/search?query=dark+side&sectionId={section_id}&limit=10
Accept: application/json
```

Response groups results into "Hubs" by type:
```json
{
  "MediaContainer": {
    "Hub": [
      {
        "type": "artist",
        "title": "Artists",
        "Metadata": [
          { "ratingKey": "12345", "title": "Pink Floyd", "type": "artist" }
        ]
      },
      {
        "type": "album",
        "title": "Albums",
        "Metadata": [
          { "ratingKey": "12346", "title": "The Dark Side of the Moon", "type": "album" }
        ]
      },
      {
        "type": "track",
        "title": "Tracks",
        "Metadata": [
          {
            "ratingKey": "12347",
            "title": "Time",
            "type": "track",
            "Media": [{ "Part": [{ "key": "/library/parts/..." }] }]
          }
        ]
      }
    ]
  }
}
```

Track results include `Media`/`Part` data for direct stream URL construction.

Library-specific search alternative:
```
GET /library/sections/{section_id}/search?type=10&query=time
```

Genre filtering:
```
GET /library/sections/{section_id}/all?type=10&genre=rock
```

Reference: [plexapi.dev Search API](https://plexapi.dev/api-reference/search/perform-a-search)

### Audio Streaming (Direct Play)

The `Part.key` from track metadata gives a direct HTTP URL to the raw audio file:

```
http://{host}:{port}{Part.key}?X-Plex-Token={token}
```

Example:
```
http://192.168.1.100:32400/library/parts/9877/1234567890/file.flac?X-Plex-Token=abc123
```

This returns the **original file bytes** with no transcoding. The response is a standard HTTP download with support for:
- `Content-Length` header (total file size)
- `Accept-Ranges: bytes` (HTTP range requests for seeking)
- `Content-Type` reflecting the actual file format

On a local LAN, even FLAC files stream instantly. **Direct Play is the only mode we need** — the daemon already has FLAC, MP3, and OGG decoders.

There is no need for Plex's transcode/HLS endpoints. Those exist for remote streaming over the internet where bandwidth is limited. On LAN, direct play is simpler, faster, and higher quality.

**IMPORTANT:** The full URL (with token) is constructed server-side at download time only — see "Token Separation" in Security Considerations.

Reference: [Streaming: Direct Play and Direct Stream](https://support.plex.tv/articles/200250387-streaming-media-direct-play-and-direct-stream/)

### Playback Reporting (Optional)

Plex tracks "On Deck", play counts, and last-played timestamps via timeline updates from clients.

**Timeline update** (call every ~10s during playback and on state changes):

```
POST /:/timeline?ratingKey={id}&state={playing|paused|stopped}&time={ms}&duration={ms}
X-Plex-Client-Identifier: <uuid>
X-Plex-Token: <token>
```

**Scrobble** (mark as played, call when track finishes or > 90% played):

```
GET /:/scrobble?key={ratingKey}&identifier=com.plexapp.plugins.library
X-Plex-Token: <token>
```

For MVP, start with scrobble only. Add timeline updates later for proper "Now Playing" on the Plex dashboard.

**Threading:** Scrobble and timeline updates must NOT run on the streaming thread (would introduce jitter in the decode/encode loop). Signal "position changed" via an atomic variable; a separate timer or the main thread fires the HTTP POST.

Reference: [plexapi.dev Timeline API](https://plexapi.dev/api-reference/timeline/report-media-timeline)

---

## Architecture

### Token Separation (Critical Design Decision)

**The Plex token MUST NOT appear in the `path` field of queue entries or search results.**

The `path` field in `music_queue_entry_t` and `music_search_result_t` is broadcast to every WebSocket client via `webui_music_send_state()` (line 355 of `webui_music.c`), sent in search/library JSON responses, sent to satellite devices, and logged at `LOG_INFO` level (line 838). If the token were embedded in the URL, it would leak to:
- Every authenticated WebUI user (including non-admins)
- All DAP2 satellite devices
- Log files (typically 0644)
- Any future log aggregation

**Solution:** Store only the `Part.key` (e.g., `/library/parts/9877/1234567890/file.flac`) in the `path` field. The full URL with token is constructed server-side inside `plex_client_get_stream_url()` at the moment of HTTP download only. Clients never see the token.

For the `path` field to be distinguishable from a local file path, use a `plex:` prefix:
```
plex:/library/parts/9877/1234567890/file.flac
```

This prefix is checked in `webui_music_start_playback()` to route to the Plex download path, and in `webui_music_is_path_valid()` for URL-specific validation.

### Source Abstraction

The `music_source.c` module defines source types and provider callbacks:

```c
typedef enum { MUSIC_SOURCE_LOCAL = 0, MUSIC_SOURCE_PLEX = 1 } music_source_t;

typedef struct {
   music_source_t source;
   int (*init)(const char *db_path);
   int (*sync)(sqlite3 *db);
   void (*cleanup)(void);
} music_source_provider_t;
```

The scanner thread iterates all registered providers, calling `init()` once and `sync()` on each scan interval. Local scanning reads filesystem metadata; Plex sync calls `plex_client_list_all_tracks()` in pages and inserts into the same `music_metadata` table with `source = MUSIC_SOURCE_PLEX`.

Playback routing uses the `plex:` path prefix:

```
┌──────────────────────────────────────────────────────────┐
│                   webui_music.c                           │
│          (playback engine, Opus encode, broadcast)        │
│                                                           │
│   webui_music_start_playback(state, path)                │
│          │                                                │
│    ┌─────┴─────┐                                         │
│    │ local     │  plex:// prefix?                        │
│    │ file path │  → plex_client_download_track()         │
│    │           │    (constructs full URL with token,     │
│    │           │     downloads to temp file)              │
│    ▼           ▼                                         │
│   audio_decoder_open(local_path_or_temp_path)            │
│          │                                                │
│          ▼                                                │
│  flac_decoder / mp3_decoder / ogg_decoder                │
│          │                                                │
│          ▼                                                │
│  resampler → opus_encoder → WebSocket broadcast          │
└──────────────────────────────────────────────────────────┘
```

**Key design decision:** The decoder layer (`audio_decoder.c`) is never modified. It always receives a local file path. HTTP download is handled in `webui_music_start_playback()` before the decoder is invoked.

### Library / Search / Browse Routing

```
┌─────────────────────────────────────────────────────────┐
│             handle_music_library()                        │
│             handle_music_search()                         │
│             handle_music_control("play", query)           │
│                      │                                    │
│                      ▼                                    │
│             music_db_*() (unified SQLite)                 │
│             (dedup: local wins over plex)                 │
│                      │                                    │
│                      ▼                                    │
│         JSON response to WebUI/satellite                 │
└─────────────────────────────────────────────────────────┘
```

All library/search/browse operations go through the unified `music_db_*()` API. The dedup subquery (`NOT EXISTS ... WHERE source < m.source`) ensures each unique artist+album+title combination appears once, preferring the highest-priority source (local=0 wins over plex=1). Matching uses `COLLATE NOCASE` for cross-source dedup resilience.

The WebUI and satellite UI code require zero changes — they receive the same JSON payloads regardless of underlying source.

### Path Validation

`webui_music_is_path_valid()` handles both local and Plex paths:

- **Local paths**: `realpath()` and verify within `g_config.paths.music_dir`
- **Plex paths**: validate `plex:` prefix, Part.key starts with `/library/parts/` or `/library/metadata/`, no path traversal, Plex must be configured (`plex_client_is_configured()`)

> **Note:** The original design checked `source != "plex"` to reject Plex paths when local was selected. Since the source selector was removed, Plex paths are accepted whenever Plex is configured.

### Metadata Handling

The existing `audio_decoder_get_metadata()` opens a file independently to extract ID3/Vorbis tags. For Plex tracks, metadata comes from the API response, not embedded tags. Code paths that call `audio_decoder_get_metadata()` on a Plex path will fail.

**Solution:** For Plex tracks, metadata (title, artist, album, duration) is populated from the Plex API response at search/browse time and stored in the `music_queue_entry_t` struct. The `audio_decoder_get_metadata()` call is skipped when the path has a `plex:` prefix. The fallback path in `handle_music_control()` (line 1144: `music_db_get_by_path()`) is also skipped for Plex paths since the metadata is already in the queue entry.

### HTTP Download for Playback (Temp File Approach)

The download step lives in `webui_music_start_playback()`, NOT in `audio_decoder_open()`:

```c
int webui_music_start_playback(session_music_state_t *state, const char *path) {
   char local_path[PATH_MAX];

   if (strncmp(path, "plex:", 5) == 0) {
      /* Download Plex track to temp file */
      const char *part_key = path + 5;
      if (plex_client_download_track(part_key, local_path, sizeof(local_path)) != 0) {
         return 1; /* Download failed */
      }
      /* Store temp path for cleanup */
      safe_strncpy(state->temp_file, local_path, sizeof(state->temp_file));
   } else {
      safe_strncpy(local_path, path, sizeof(local_path));
   }

   /* Existing decoder open logic — always receives a local file path */
   audio_decoder_t *dec = audio_decoder_open(local_path);
   // ...
}
```

**Temp file management:**
- Use `mkstemps()` for atomic creation with the correct extension (`.flac`, `.mp3`, `.ogg`)
- Set permissions via `fchmod(fd, 0600)` immediately on the open file descriptor
- `unlink()` the temp file immediately after the decoder opens its fd (Unix trick — file stays accessible via fd until closed, but is automatically cleaned up if daemon crashes)
- Add startup cleanup sweep: `glob("/tmp/dawn_plex_*", ...)` and remove any orphaned files

**Download size limit:** Set `CURLOPT_MAXFILESIZE_LARGE` to 500MB (covers even 24-bit/96kHz FLAC). Abort early on `Content-Length` before downloading. Log a warning if a file exceeds 100MB.

**Disk considerations (Jetson):** Verify `/tmp` is tmpfs (RAM-backed) via `statfs()` at startup and warn if it is not. A 50MB temp file on tmpfs is 0.6% of 8GB RAM — acceptable.

---

## Implementation Plan

### Buffer Size Reconciliation

Before any implementation, reconcile the path buffer sizes:

| Struct | Field | Current Size | Issue |
|--------|-------|-------------|-------|
| `music_search_result_t` | `path` | 1024 (`MUSIC_DB_PATH_MAX`) | OK for Plex `plex:` + Part.key |
| `music_queue_entry_t` | `path` | 512 (`WEBUI_MUSIC_PATH_MAX`) | Truncation risk on copy from search result |

A `plex:` path is typically ~60-80 chars (e.g., `plex:/library/parts/9877/1234567890/file.flac`), well within 512. But to eliminate the existing size mismatch (which could truncate long local paths too), increase `WEBUI_MUSIC_PATH_MAX` to 1024 to match `MUSIC_DB_PATH_MAX`.

**Memory impact:** `music_queue_entry_t` increases from ~900 to ~1400 bytes. With 100-entry queues, per-session memory goes from ~88KB to ~137KB. Acceptable on 8GB Jetson.

### New Files

| File | Purpose | Est. Lines |
|------|---------|------------|
| `src/audio/plex_client.c` | Plex REST API client (auth, browse, search, stream URL, scrobble) | ~600-800 |
| `include/audio/plex_client.h` | Public API for Plex client | ~80 |
| `src/audio/http_download.c` | libcurl download-to-temp-file utility | ~150 |
| `include/audio/http_download.h` | Public API for HTTP download | ~30 |

### Modified Files

| File | Change |
|------|--------|
| `src/webui/webui_music.c` | Route library/search/browse to Plex or local based on `source` config; add Plex download step to `start_playback()`; update `is_path_valid()` for `plex:` prefix; skip `audio_decoder_get_metadata()` for Plex paths |
| `include/webui/webui_music.h` | Increase `WEBUI_MUSIC_PATH_MAX` to 1024; add `temp_file` field to `session_music_state_t` |
| `include/config/dawn_config.h` | Add `plex_config_t` to `music_config_t`; add `plex_token` to `secrets_config_t` |
| `src/config/config_parser.c` | Parse `[music]` source + `[music.plex]` section |
| `src/config/config_defaults.c` | Default values for Plex config |
| `src/config/config_env.c` | JSON serialization + TOML write for Plex config; secrets JSON status for plex_token |
| `src/webui/webui_config.c` | Add plex_token to `handle_set_secrets()`; add `[music.plex]` to `apply_config_from_json()` |
| `www/js/ui/settings/schema.js` | Add Music Source dropdown + Plex fields with `showWhen` conditional visibility |
| `www/js/ui/settings.js` | Add Plex Token row to Secrets panel HTML |
| `www/js/ui/settings/config.js` | Add plex_token to secrets save/load flow |
| `www/js/ui/music.js` | Add source indicator badge; add persistent error banner for Plex failures |

### Phase 1: Plex Client + Config (MVP)

1. Increase `WEBUI_MUSIC_PATH_MAX` to 1024
2. Add `plex_config_t` struct and `plex_token` to `secrets_config_t`
3. Add config parsing, defaults, TOML write, JSON serialization
4. Implement `plex_client.c`:
   - `plex_client_init()` — create two persistent CURL handles (one for API queries, one for downloads) with `CURLOPT_TCP_KEEPALIVE` and HTTP keep-alive. Reuse the existing `curl_buffer_t` from `curl_buffer.h` for API responses with `curl_buffer_init_with_max(buf, CURL_BUFFER_MAX_PLEX_API)`.
   - `plex_client_discover_section()` — `GET /library/sections`, find music section
   - `plex_client_list_artists(page, limit)` — paginated artist list
   - `plex_client_list_albums(artist_id, page, limit)` — albums for artist
   - `plex_client_list_tracks(album_id)` — tracks for album
   - `plex_client_search(query, limit)` — hub search
   - `plex_client_get_stream_url(part_key)` — construct full URL with token (used only for download, never stored)
   - `plex_client_download_track(part_key, out_path, out_path_size)` — download via HTTP to temp file
   - `plex_client_scrobble(rating_key)` — mark track as played
   - `plex_client_cleanup()` — free CURL handles
5. Implement `http_download.c`:
   - `http_download_to_temp(url, headers, out_path, out_path_size, max_size)` — download via libcurl to `/tmp/dawn_plex_XXXXXX.ext` using `mkstemps()`, `fchmod(0600)`, `unlink()` after decoder open
   - Set `CURLOPT_MAXFILESIZE_LARGE` to cap download size
   - Set `CURLOPT_CONNECTTIMEOUT = 10`, `CURLOPT_LOW_SPEED_LIMIT = 1024`, `CURLOPT_LOW_SPEED_TIME = 30`
   - Startup cleanup: `glob("/tmp/dawn_plex_*", ...)` to remove orphans
6. Add `plex_token` to Secrets panel (HTML + JS)

### Phase 2: Wire Into Music Pipeline

7. Update `webui_music_is_path_valid()` for `plex:` prefix validation
8. Update `webui_music_start_playback()` to detect `plex:` prefix and download to temp file
9. Route `handle_music_library()` / `handle_music_search()` in `webui_music.c` through source switch
10. Map Plex JSON responses to existing `music_library_response` / `music_search_response` formats:
    - Artists: `{ name, track_count }` — use `Directory[].leafCount` and `Directory[].childCount`
    - Albums: `{ name, artist, year, track_count }` — all available from Plex
    - Tracks: `{ title, artist, album, duration_sec, path }` — `path` is `plex:{Part.key}` (no token)
11. Add `showWhen` conditional visibility to schema system
12. Add Plex connection fields to WebUI settings panel
13. Add source indicator badge to music player panel
14. Add persistent error banner for Plex auth/connection failures
15. Add confirmation dialog on source switch during active playback

### Phase 2.5: Unified Music DB (Completed March 2026)

16. ✅ Add genre extraction to `build_track_json()` in `plex_client.c`
17. ✅ Add `plex_client_get_library_updated_at()` for change detection
18. ✅ Implement `plex_db.c` — Plex sync provider using `music_source_provider_t` callbacks
19. ✅ Implement `music_source.c` — source abstraction with enum, path prefixes, provider registry
20. ✅ Extend `music_scanner.c` with multi-source provider iteration
21. ✅ Add `source` column to `music_metadata` table with priority-based dedup
22. ✅ Add `COLLATE NOCASE` to dedup index and queries
23. ✅ Replace all Plex API browse/search calls with unified `music_db_*()` queries
24. ✅ Remove deprecated `plex_client` browse/search functions and enrich cache
25. ✅ Remove `config.music.source` field (auto-detection replaces selector)
26. ✅ Update WebUI settings (remove source dropdown)
27. ✅ Add `test_music_db.c` unit tests (22 tests, 60 assertions)

### Phase 3: Polish

22. GDM server discovery — "Discover" button next to Plex Server field; results presented for user confirmation; run in detached thread with 3-second timeout
17. Scrobble reporting on track completion (from main thread, not streaming thread)
18. Timeline updates during playback (for Plex "Now Playing" dashboard)
19. Album art thumbnails — add `thumbnail` field to track JSON response (Plex: `/photo/:/transcode` URL; local: `null` for now). Daemon proxies art to WebUI (browser should not need direct Plex access).
20. "Test Connection" button in settings — sends `test_plex_connection` WebSocket message, daemon calls `GET /library/sections`, returns server name or error
21. Token refresh / expiry handling with user notification
22. Pre-fetch next track during current playback (last 10% triggers download of `queue[index+1]` — eliminates inter-track silence on slower networks)
23. Plex playlist browsing (list playlists, get playlist tracks)

---

## Response Format Mapping

The WebUI and satellites expect specific JSON formats. Here's how Plex data maps to them.

### Artists (`browse_type: "artists"`)

Current format:
```json
{
  "type": "music_library_response",
  "payload": {
    "browse_type": "artists",
    "artists": [
      { "name": "Pink Floyd", "album_count": 15, "track_count": 165 }
    ],
    "total": 150,
    "offset": 0,
    "limit": 50
  }
}
```

Plex source:
- `name` ← `Directory[].title`
- `album_count` / `track_count` — not directly available in artist listing. Options:
  - Use `Directory[].leafCount` (total tracks) and `Directory[].childCount` (albums) if available
  - Or omit counts (set to 0) and populate on drill-down

### Albums (`browse_type: "albums"`)

Current format:
```json
{
  "payload": {
    "browse_type": "albums",
    "albums": [
      { "name": "The Dark Side of the Moon", "artist": "Pink Floyd", "year": 1973, "track_count": 10 }
    ]
  }
}
```

Plex source:
- `name` ← `Directory[].title`
- `artist` ← `Directory[].parentTitle`
- `year` ← `Directory[].year`
- `track_count` ← `Directory[].leafCount`

### Tracks

Current format:
```json
{
  "payload": {
    "results": [
      {
        "title": "Time",
        "artist": "Pink Floyd",
        "album": "The Dark Side of the Moon",
        "duration_sec": 413,
        "path": "/home/user/Music/Pink Floyd/DSOTM/04 - Time.flac"
      }
    ]
  }
}
```

Plex source:
- `title` ← `Metadata[].title`
- `artist` ← `Metadata[].grandparentTitle`
- `album` ← `Metadata[].parentTitle`
- `duration_sec` ← `Metadata[].duration / 1000` (Plex uses milliseconds)
- `path` ← `plex:{Part.key}` (e.g., `plex:/library/parts/9877/1234567890/file.flac`)

**The `path` field does NOT contain the token.** The full stream URL is constructed server-side only at download time in `plex_client_download_track()`.

### Search

Current `music_search_response` already returns tracks in the same format as above. Plex's `/hubs/search` returns tracks with full `Media`/`Part` data, so mapping is straightforward.

---

## Dependencies

| Library | Purpose | Status |
|---------|---------|--------|
| **libcurl** | HTTP client for all Plex API calls + file download | Already used by LLM providers (`llm_openai.c`, `llm_claude.c`) |
| **json-c** | JSON parsing for Plex API responses | Already used throughout daemon |
| **POSIX sockets** | GDM UDP discovery | Standard, no additional library |

**No new dependencies required.** The Plex API is pure REST/JSON over HTTP.

---

## Security Considerations

### 1. Token Separation (HIGH)

The Plex token MUST NOT appear in `path` fields, JSON messages to clients, or log output. Store only `plex:{Part.key}` in path fields. Construct the full URL with token server-side at download time only. See "Token Separation" in Architecture section.

### 2. Token Storage (HIGH)

Store in `secrets_config_t.plex_token` (loaded from `secrets.toml`, mode 0600). Add to `secrets_to_json_status()` as is_set flag only. Add to `handle_set_secrets()` for admin-only updates. Never include in `config_to_json()` or `get_config_response`.

### 3. Path Validation (HIGH)

`webui_music_is_path_valid()` must be updated to handle `plex:` prefix paths. Validate:
- Scheme is exactly `plex:` (not `http://`, not arbitrary URLs)
- Part.key starts with `/library/parts/` or `/library/metadata/`
- No `@` character (prevents authority injection)
- No path traversal sequences (`../`)
- Plex must be configured (`plex_client_is_configured()` — host and token present)

### 4. URL Construction Safety

When constructing the full download URL in `plex_client_get_stream_url()`:
- Validate the Part.key starts with `/library/` before appending to host
- Use `CURLOPT_FOLLOWLOCATION = 0` (do not follow redirects — prevents redirect to attacker-controlled host)
- Validate total URL length fits in buffer before construction (use `snprintf` return value check)

### 5. SSL/TLS

When `ssl = true`:
- Default to full certificate verification (`CURLOPT_SSL_VERIFYPEER = 1`)
- Add `ssl_verify = false` as a separate config option with a log warning when disabled
- For self-signed Plex certs: document how to export and add to DAWN's existing private CA trust store (same infrastructure used for satellite TLS)
- Never silently disable verification

### 6. GDM Discovery

- Never auto-apply discovered servers — always present to user for confirmation
- Validate discovered `Port:` is integer 1-65535
- Parse response headers with `snprintf`/`strncpy` and explicit size limits
- After user selects, validate via `GET /identity` with the token

### 7. Temp File Security

- Create with `mkstemps()` (atomic, suffix-aware)
- Set `fchmod(fd, 0600)` immediately
- `unlink()` after decoder opens its fd (file persists until fd closed)
- Startup orphan cleanup sweep
- `CURLOPT_MAXFILESIZE_LARGE` prevents disk exhaustion from malicious server

### 8. Log Sanitization

The existing `LOG_INFO("WebUI music: Playing %s", path)` at line 838 of `webui_music.c` would previously only log local file paths. With the `plex:` prefix approach, the logged path is just `plex:/library/parts/...` — no token. No code changes needed.

---

## Agent Review Findings (Incorporated)

This design was reviewed by all four DAWN review agents (architecture, security, efficiency, UI). All findings have been incorporated into the sections above. Key changes from the original design:

| Finding | Source Agents | Resolution |
|---------|---------------|------------|
| Token in path leaks to all WebSocket clients | Security, Architecture, Efficiency | `plex:` prefix with server-side URL construction |
| `webui_music_is_path_valid()` rejects URLs | Architecture, Security | Updated validation for `plex:` prefix with whitelist |
| `WEBUI_MUSIC_PATH_MAX` (512) < `MUSIC_DB_PATH_MAX` (1024) | Architecture, Security | Increase to 1024 |
| Token in `dawn.toml` config struct | Architecture, Security, UI | Move to `secrets_config_t` |
| `audio_decoder_open()` extension detection fails on URLs | Architecture | Download in `start_playback()`, not in decoder |
| Schema has no conditional field visibility | UI | Implement `showWhen` mechanism |
| Schema has no `password` field type for token | UI | Route through existing Secrets panel |
| No error state UX for Plex failures | UI, Architecture | Persistent error banner + `music_state.error` field |
| Temp file cleanup on crash | Efficiency, Security | `unlink()` after open + startup sweep |
| No download size limit | Efficiency, Security | `CURLOPT_MAXFILESIZE_LARGE` (500MB) |
| No CURL handle reuse | Efficiency | Two persistent handles (API + download) with keep-alive |
| GDM discovery spoofable | Security | User confirmation required, never auto-apply |
| `CURLOPT_SSL_VERIFYPEER = 0` defeats TLS | Security | Separate `ssl_verify` toggle, reuse private CA |
| No source switch confirmation | UI | Confirm dialog when playback active |
| No source indicator in music player | UI | Source badge in `music_state` |
| `webui_music.c` already 2,507 lines | Architecture | Keep routing thin, delegate to `plex_client_*()` |
| `audio_decoder_get_metadata()` fails on URLs | Architecture | Skip for Plex paths, use API metadata |
| Large library JSON spikes memory | Efficiency | Mandate pagination, cap API buffer at 2MB |
| Blocking download on track transition | Efficiency | Phase 3: pre-fetch next track |
| No connection/low-speed timeouts | Efficiency | CURL timeout settings specified |
| Scrobble/timeline on streaming thread adds jitter | Efficiency | Fire from main thread via atomic signal |
| Album art needs `thumbnail` field | UI | Add to track JSON (Phase 3) |
| GDM implementation guards | Efficiency | Thread, timeout, iteration cap, `fromlen` reset |

---

## Reference Links

- [Plex API Documentation (plexapi.dev)](https://plexapi.dev/)
- [Finding an Authentication Token](https://support.plex.tv/articles/204059436-finding-an-authentication-token-x-plex-token/)
- [Plex Media Server URL Commands](https://support.plex.tv/articles/201638786-plex-media-server-url-commands/)
- [Streaming: Direct Play and Direct Stream](https://support.plex.tv/articles/200250387-streaming-media-direct-play-and-direct-stream/)
- [Authentication for Local Network Access](https://support.plex.tv/articles/200890058-authentication-for-local-network-access/)
- [Network Ports / Firewall (Port 32400 default, 32414 GDM)](https://support.plex.tv/articles/201543147-what-network-ports-do-i-need-to-allow-through-my-firewall/)
- [GDM Discovery — python-plexapi](https://python-plexapi.readthedocs.io/en/latest/modules/gdm.html)
- [plexapi.dev Search API](https://plexapi.dev/api-reference/search/perform-a-search)
- [plexapi.dev Timeline API](https://plexapi.dev/api-reference/timeline/report-media-timeline)
- [python-plexapi audio.py (GitHub)](https://github.com/pkkid/python-plexapi/blob/master/plexapi/audio.py)
- [Plex Pro Week '25: API Unlocked](https://www.plex.tv/blog/plex-pro-week-25-api-unlocked/)
- [Plex Playlist API (Plexopedia)](https://www.plexopedia.com/plex-media-server/api/playlists/view-items/)

---

## Plex Library Cache (Implemented)

**Implemented:** March 2026

### Problem

`plex_client_search()` uses `/library/sections/{id}/search?type=10&query=X` which searches **track titles only**. Queries by artist name ("Alan Silvestri"), album name, or genre ("epic music", "rock") return zero results even when the library contains matching tracks. This is a fundamental limitation of Plex's section-level search API with type filtering.

Meanwhile, local music search works well because `music_db_search()` does SQL LIKE matching across title, artist, album, and path fields simultaneously in a local SQLite database.

### Solution

Cache the entire Plex library metadata into a local SQLite table (`plex_tracks`) in the same `music.db` database. Replace all Plex API search/browse calls with local SQLite queries using the same LIKE pattern matching as local music. The Plex API is then only used for:

1. **Sync**: Periodic metadata sync (background thread, same pattern as `music_scanner.c`)
2. **Download**: `plex_client_download_track()` at playback time (unchanged)

This also adds **genre support** since Plex provides genre metadata in `Genre[].tag` arrays on track responses.

### Schema

> **Design evolution:** The original design proposed a separate `plex_tracks` table. The actual implementation uses the **existing `music_metadata` table** with an added `source` column, enabling unified queries with priority-based deduplication via a single `NOT EXISTS` subquery. This proved simpler and more powerful than UNION-based approaches.

Plex tracks are stored in `music_metadata` with `source = 1` (MUSIC_SOURCE_PLEX) and `path` prefixed with `plex:`:

```sql
-- Existing table, extended with source column
CREATE TABLE IF NOT EXISTS music_metadata (
   id INTEGER PRIMARY KEY AUTOINCREMENT,
   path TEXT UNIQUE NOT NULL,     -- Local: /home/.../file.flac, Plex: plex:/library/parts/...
   title TEXT, artist TEXT, album TEXT, genre TEXT,
   duration_sec INTEGER, mtime INTEGER,
   source INTEGER DEFAULT 0       -- 0=LOCAL, 1=PLEX
);

-- Case-insensitive dedup index (critical for query performance)
CREATE INDEX IF NOT EXISTS idx_music_dedup
   ON music_metadata(artist COLLATE NOCASE, album COLLATE NOCASE,
                     title COLLATE NOCASE, source);
```

The dedup query pattern used throughout `music_db.c`:
```sql
WHERE NOT EXISTS (
   SELECT 1 FROM music_metadata m2
   WHERE m2.artist = m.artist COLLATE NOCASE
     AND m2.album = m.album COLLATE NOCASE
     AND m2.title = m.title COLLATE NOCASE
     AND m2.source < m.source
)
```

### Search Query

Extends local search pattern with genre matching:

```sql
SELECT rating_key, part_key, title, artist, album, genre, duration_sec
FROM plex_tracks
WHERE title LIKE ? OR artist LIKE ? OR album LIKE ? OR genre LIKE ?
ORDER BY artist COLLATE NOCASE, album COLLATE NOCASE, title COLLATE NOCASE
LIMIT ?
```

Results map to the existing `music_search_result_t` struct:
- `path` = `"plex:" + part_key` (token-free, same prefix convention)
- `title`, `artist`, `album`, `duration_sec` = direct copy
- `display_name` = `"Artist - Title"` (same format as local)

### Sync Mechanism

Reuses `music_scanner.c` background thread pattern (interval + `pthread_cond_timedwait`):

1. **Change detection**: Call `GET /library/sections/{id}` → check `scannedAt` timestamp. If unchanged since last sync, skip (~50ms).
2. **Full sync**: Paginate `plex_client_list_all_tracks(offset, 200)` through entire library. `INSERT OR REPLACE` each track. Mark-and-sweep to delete removed tracks.
3. **Initial sync**: ~3-5 seconds for 3,000 tracks on LAN (16 pages × ~200ms each).
4. **Mutex yielding**: Every 50 inserts, yield DB mutex for 1ms (same as local scan pattern).

### Handler Convergence (Completed)

The implementation went further than the original proposal — instead of two parallel SQLite query functions (`plex_db_search()` vs `music_db_search()`), there is a **single `music_db_*()` API** that queries the unified table with built-in dedup:

```c
// All handlers use the same API regardless of source
music_search_result_t *results = malloc(...);
int count = music_db_search(query, results, max);  // Searches both sources, deduped
```

The `source` column and dedup subquery are internal to `music_db.c`. Callers never need to know which sources are active. This eliminated ~400 lines of source-switching logic from `webui_music_handlers.c`.

### Files

| File | Purpose |
|------|---------|
| `src/audio/plex_db.c` (new) | Plex cache DB layer: init, sync, search, browse |
| `include/audio/plex_db.h` (new) | Public API |
| `src/audio/plex_client.c` | Add genre extraction to `build_track_json()`, add `plex_client_get_library_updated_at()` |
| `src/audio/music_scanner.c` | Add Plex sync mode (reuse thread pattern) |
| `src/webui/webui_music_handlers.c` | Replace `plex_client_search/list_*()` with `plex_db_*()` |
| `src/dawn.c` | Init plex_db when source=plex, route scanner mode |

### Deprecated API Functions (Removed)

The following Plex API browse/search functions were removed from `plex_client.c` (March 2026) along with their enrich cache infrastructure, as all operations now go through the unified `music_db_*()` API:

- `plex_client_search()` → replaced by `music_db_search()`
- `plex_client_list_artists()` → replaced by `music_db_list_artists()`
- `plex_client_list_albums()` → replaced by `music_db_list_albums()`
- `plex_client_list_tracks()` → replaced by `music_db_search_by_album()`
- `plex_client_list_artist_tracks()` → replaced by `music_db_search_by_artist()`
- `plex_client_get_stats()` → replaced by `music_db_get_stats()`
- `enrich_counts_from_tracks()` / `populate_enrich_cache()` / enrich cache structs — removed

**Retained**: `plex_client_list_all_tracks()` (used by `plex_db.c` for sync), `plex_client_download_track()`, `plex_client_scrobble()`, `plex_client_test_connection()`, `plex_client_discover_section()`, `plex_client_get_library_updated_at()`.

---

## Source Upgrade Path (Completed)

The original design proposed a three-step upgrade path:

1. ~~**Option A: Either/Or** — user selects "local" or "plex" exclusively~~ (Phase 1, February 2026)
2. ~~**Option C: Switchable with Mixed Queue** — queue can contain tracks from both sources~~ (skipped)
3. **Option B: Joined Library** — unified view with deduplication (implemented March 2026)

The implementation went directly from Option A to Option B by adding a `source` column to the existing `music_metadata` table and using a `NOT EXISTS` subquery for priority-based deduplication (local wins over Plex). Case-insensitive matching (`COLLATE NOCASE`) handles metadata differences between sources (e.g., "The Beatles" vs "the beatles").

The `source` config selector was removed entirely — sources are auto-detected based on configuration. The unified `music_db_*()` API transparently handles both sources with dedup, and the `plex:` path prefix routes playback to the appropriate download mechanism.

### Unit Test Coverage

`tests/test_music_db.c` provides 22 tests with 60 assertions covering:
- Source abstraction (names, prefixes, path identification)
- Init/cleanup lifecycle and schema migration idempotency
- Search with dedup (local wins, plex-only, case-insensitive, LIKE escaping)
- Browse with dedup (artists, albums, artist tracks, stats)
- Path lookup (local, plex, missing)
- Source-scoped stale deletion (plex rows survive local cleanup)
