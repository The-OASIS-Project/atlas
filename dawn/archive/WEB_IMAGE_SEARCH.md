# Web Image Search Tool

**Date**: April 2026
**Status**: Complete
**Depends on**: Image Store Phase 1 (complete), `web_search.c` backend library

---

## Overview

New LLM tool that searches SearXNG for images, pre-fetches them server-side, caches them via the image store, and returns local URLs. Clients never see upstream image URLs — all images are served from DAWN's image store.

This is Phase 3 of the [Unified Image Store](UNIFIED_IMAGE_STORE_DESIGN.md) plan.

## Architecture

The image search tool follows the same layered architecture as the existing text search:

```
web_search.c              ← SearXNG HTTP client library (shared backend)
├── search_tool.c         ← text search tool (existing, category=web/news/science/...)
└── image_search_tool.c   ← image search tool (new, category=images)
```

`web_search.c` owns the SearXNG connection (base URL, curl setup, JSON parsing). Both tools are thin wrappers that call into the backend and handle the response differently:
- **Text search**: calls `web_search_query_typed()` → gets `search_response_t` → formats text
- **Image search**: calls `web_search_query_images_raw()` → gets `json_object*` → fetches binaries → caches → returns URLs

### Why a separate tool (not a category on `search`)

The tools share the SearXNG backend but differ in:
- **Return format**: JSON array of image URLs vs formatted text
- **Parameters**: `count` doesn't apply to text search; `time_range` doesn't apply to images
- **Capabilities**: text search is `SCHEDULABLE`, image search is not
- **Dependencies**: image search requires `image_store_is_ready()`, text search doesn't
- **Callback logic**: image search fetches binaries, validates magic bytes, caches files — completely different code path

Zero code duplication — the SearXNG query is handled by the shared backend.

## Tool Definition

```
Name:        image_search
Device:      image_search
Capabilities: TOOL_CAP_NETWORK
Not SCHEDULABLE, not DANGEROUS
```

### Parameters

| Name | Type | Required | Default | Maps To | Description |
|------|------|----------|---------|---------|-------------|
| `query` | string | yes | — | value | Image search query |
| `count` | int | no | 5 | custom | Number of images to fetch and return |

`vision_rerank` and `return_count` are deferred to v2 when a local vision model is available. Adding stub parameters that LLMs will set but that silently do nothing creates confusion. When vision reranking is implemented, `count` becomes the candidate pool size and a new `return_count` controls how many pass the reranker.

### Tool Description (LLM dispatch guidance)

Included in the tool's `description` field so the LLM knows when and how to call it:

- **WebUI session**: `count=5` (user scans a grid)
- **HUD / MIRAGE session**: `count=1` (single focal image)
- **Voice-only satellite**: Don't call this tool — describe verbally or offer WebUI
- **Ambiguous / shortlist**: `count=3`

The session type (LOCAL / WEBUI / SATELLITE) is already visible to the model in the system prompt.

## Implementation

### Changes to `web_search.c` (shared backend)

1. Add `SEARCH_TYPE_IMAGES` to `search_type_t` enum in `web_search.h`
2. Add new function `web_search_query_images_raw()`:
   ```c
   struct json_object *web_search_query_images_raw(const char *query, int max_results);
   ```
   This builds the SearXNG URL (with `categories=images&safesearch=1`), performs the curl request, and returns the parsed `json_object*` root. The caller owns the reference and must call `json_object_put()` when done.

   This avoids polluting `search_result_t` with image-only fields (`img_src`, `thumbnail_src`, `resolution`) and avoids duplicating the curl/URL-building/error-handling logic. The existing `web_search_query_typed()` continues to return `search_response_t` for text results.

### Callback flow

1. Call `web_search_query_images_raw(query, count * 2)` — over-requests to compensate for SSRF/format rejections
2. Parse `results` array from the returned `json_object*`: extract `img_src`, `title`, `source`, `resolution` per result
3. SSRF pre-flight on each `img_src`:
   - `url_is_blocked_with_resolve()` — rejects private IPs, captures resolved IP + host + port
   - Fail-closed: if DNS resolution returns no IP, skip (prevents TOCTOU)
   - Build `CURLOPT_RESOLVE` entry to pin DNS (prevents rebinding between check and connect)
4. Fetch images concurrently using `curl_multi_*`:
   - `CURLMOPT_MAX_TOTAL_CONNECTIONS = 4` — limits concurrent TLS sessions on constrained Jetson
   - `CURLOPT_FOLLOWLOCATION = 0` — no implicit redirects (SSRF defense)
   - 1MB max download per image (`IMAGE_SEARCH_MAX_FETCH_SIZE`), 10s per-transfer timeout
   - **Wall-clock cap: 10s total** (CLOCK_MONOTONIC) — when deadline hits, mark remaining as invalid
   - HTTPS/HTTP only via `CURLOPT_PROTOCOLS`
5. **Redirect-with-revalidation** (second pass): any 3xx response from the first pass triggers:
   - Validate redirect URL scheme (must be http/https)
   - SSRF-check the redirect target (same `url_is_blocked_with_resolve` + DNS pinning)
   - Fail-closed on empty DNS resolution
   - Fetch the validated redirect target with same `FOLLOWLOCATION=0` (max 1 redirect, no chains)
   - Second pass shares the remaining wall-clock time from the first pass
6. For each successfully fetched image:
   - Validate magic bytes (JPEG: `FF D8 FF`, PNG: `89 50 4E 47`, GIF: `GIF87a`/`GIF89a`, WebP: `RIFF...WEBP`)
   - Determine MIME type from magic bytes (ignore Content-Type — servers lie)
   - `image_store_save_ex(user_id, data, size, mime, IMAGE_SOURCE_SEARCH, IMAGE_RETAIN_CACHE, id_out)`
   - Skip on invalid magic, oversized, no valid user context — these are expected
5. Return structured JSON array to LLM:
   ```json
   [
     {
       "url": "/api/images/img_xxx",
       "width": 1920,
       "height": 1080,
       "source": "wikipedia.org",
       "title": "Eiffel Tower at sunset"
     }
   ]
   ```
   Always return an array even when `count=1` (future-proof for multi-image display).
6. Publish to HUD for MIRAGE display (OCP v1.4 compliant):
   ```json
   {
     "device": "image",
     "action": "display",
     "msg_type": "request",
     "image_url": "/api/images/img_xxx",
     "title": "Eiffel Tower at sunset",
     "source": "wikipedia.org",
     "ttl": 30,
     "timestamp": 1713100000000
   }
   ```
   Note: `action` is correct — DAWN is commanding MIRAGE to display (imperative). `msg_type: "request"` required on shared `hud` topic per OCP v1.4.

   The existing `publish_hud()` in `phone_service.c` uses `event` semantics and is not reusable here. The image search tool constructs and publishes its own MQTT message inline using json-c + `mosquitto_publish()` (same pattern used by `command_executor.c` for fire-and-forget publishes).

   **Gated on session type**: only published when `session_get_command_context()->type == SESSION_TYPE_LOCAL` (voice/mic). WebUI sessions render the markdown image grid inline; satellite sessions don't receive the tool. This keeps the HUD display focused on physical presence (the Iron Man suit wearer).

### Image caching

Images are stored via the unified image store (`image_store_save_ex`) with:
- `user_id = 0` (system-wide, not per-user)
- `source = IMAGE_SOURCE_SEARCH`
- `retention = IMAGE_RETAIN_CACHE` (LRU eviction at `cache_size_mb` cap, default 200MB)

The image store handles:
- Atomic file writes (tmp + fsync + rename)
- Filesystem storage in `<data_dir>/images/`
- Zero-copy HTTP serving via `lws_serve_http_file()`
- LRU cleanup based on `last_accessed` timestamp
- Source-aware access control (SEARCH images accessible to any authenticated user)

No separate cache directory needed — the image store's RETAIN_CACHE policy provides the same LRU behavior.

### Error handling

Match `search_tool.c` patterns:
- SearXNG unreachable → return error string to LLM: "Image search unavailable"
- Zero results → return empty array `[]`
- All fetches failed (403s, timeouts) → return empty array with message: "Found results but could not fetch any images"
- `image_store_is_ready()` false → return error: "Image storage not available"

### Availability check

```c
static bool image_search_is_available(void) {
   return web_search_is_initialized() && image_store_is_ready();
}
```

Tool is only visible to the LLM when both SearXNG and the image store are initialized.

## Files

### Created

- `src/tools/image_search_tool.c` — tool registration + callback + concurrent image fetching
- `include/tools/image_search_tool.h` — registration function declaration

### Modified

- `include/tools/web_search.h` — add `SEARCH_TYPE_IMAGES` to enum, declare `web_search_query_images_raw()`
- `src/tools/web_search.c` — implement `web_search_query_images_raw()` (builds URL, curl, returns `json_object*`)
- `src/tools/tools_init.c` — call `image_search_tool_register()`
- `cmake/DawnTools.cmake` — `DAWN_ENABLE_IMAGE_SEARCH_TOOL` option (default ON when SearXNG is configured)

## Consumers

| Consumer | Behavior |
|----------|----------|
| **WebUI** | Render URLs as markdown image grid from the structured return |
| **MIRAGE** | Receives HUD MQTT message, fetches image via HTTP with Bearer service token, displays as notification widget (shipped — see `UNIFIED_IMAGE_STORE_DESIGN.md` Step 6) |
| **Voice satellite** | Tool should not be called; if it is, return structured data and let the caller decide |

## Testing

### Unit tests (`tests/test_image_search.c`)

Testable without network by extracting validation helpers as static functions:
- Magic byte validation: valid JPEG/PNG/GIF/WebP headers, invalid/truncated data, SVG rejection
- SSRF check integration: verify private IPs rejected before fetch
- Result parsing: valid SearXNG JSON → correct `img_src`/`title`/`source` extraction
- Count truncation: 10 results from SearXNG with `count=3` → only 3 returned
- Empty/error responses: SearXNG returns `{"results": []}` → empty array returned

### Manual tests

- "search for pictures of sunsets" → verify images cached locally, WebUI shows grid
- "find me a photo of the Eiffel Tower" with HUD → verify MQTT message published
- Edge cases: SearXNG down, all images 403'd, oversized images (>4MB), SVG results (should be skipped)
- Verify: `ls <data_dir>/images/` shows new `img_*.jpg` files after search
- Verify: cached images accessible via `/api/images/<id>` without re-fetching
- Verify: wall-clock cap works — slow servers don't block beyond 10s

## Deferred to v2

- **Vision reranking** — when a local vision model is available, add `vision_rerank` (bool) and `return_count` (int) params. `count` becomes the candidate pool, `return_count` the output after reranking.
- **Configurable safesearch** — currently hardcoded `safesearch=1` in the SearXNG URL. Add a `safesearch` field to `[search]` in `dawn.toml` (values: 0=off, 1=moderate, 2=strict, default 1). Pass through to `web_search_query_images_raw()`. Also useful for the text search tool.
- **Duplicate detection** — hash images before saving to skip duplicates. Not worth the complexity in v1 since LRU eviction handles the space.
