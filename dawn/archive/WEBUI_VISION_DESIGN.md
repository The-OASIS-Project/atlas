# WebUI Vision Support Design

## Overview

This document describes the design for adding vision/image support to DAWN's WebUI, enabling users to share images for AI analysis similar to ChatGPT's image upload feature.

## Research Summary

### Industry Practices

**OpenWebUI** (open-source ChatGPT alternative):
- Supports multiple input methods: file upload, paste, drag-and-drop, screen capture
- Reads images as base64 data URLs via `FileReader.readAsDataURL()`
- Optionally compresses images before upload
- Checks model vision capability before allowing image input

**Home Assistant Assist**:
- Integrates AI image recognition for camera snapshots
- Uses cloud providers (ChatGPT, Gemini, Claude) or local Ollama
- Separates command processing from AI vision queries

**ADA V2**:
- Uses webcam-based vision via MediaPipe for face/gesture recognition
- Keeps biometric data local for privacy
- Real-time camera feed rather than static image uploads

### Key Insight

The common pattern is:
1. Client-side image capture/selection
2. Base64 encoding in browser
3. JSON message with embedded base64 data
4. Server passes to vision-capable LLM

---

## DAWN Architecture Context

### Current State

- **Existing Vision Support**: The `viewing` tool captures images via MQTT camera integration and passes base64 data to the LLM
- **LLM Integration**: Both `llm_openai.c` and `llm_claude.c` support vision via `vision_image` parameter (base64 encoded)
- **WebSocket Protocol**: Binary frames for audio, JSON text frames for control messages
- **Session Management**: Per-session conversation history with LLM config

### Existing Code Patterns

```c
// llm_interface.c - Vision support already exists
char *llm_chat_completion(json_object *conversation_history,
                          const char *input_text,
                          char *vision_image,      // Base64 image data
                          size_t vision_image_size);

// llm_openai.c - OpenAI vision format
char *data_uri_prefix = "data:image/jpeg;base64,";

// llm_claude.c - Claude vision format
json_object_object_add(source_obj, "type", json_object_new_string("base64"));
json_object_object_add(source_obj, "media_type", json_object_new_string("image/jpeg"));
```

---

## Design Decisions

### 1. Image Transport: JSON with Base64 (Recommended)

**Options Considered:**

| Method | Pros | Cons |
|--------|------|------|
| Base64 in JSON | Simple, matches existing patterns, atomic | ~33% size overhead |
| Binary WebSocket | Efficient, minimal overhead | Requires new protocol, fragmentation |
| HTTP multipart | Standard, resumable | Separate endpoint, CORS complexity |

**Decision: Base64 in JSON**

Rationale:
- Matches existing vision image handling in `llm_openai.c` and `llm_claude.c`
- WebUI already handles large JSON messages (conversation history)
- Images for vision are typically <5MB (compressible to ~500KB base64)
- Simpler implementation - reuses existing `handle_json_message()` pattern
- Atomic message delivery - no fragmentation concerns

### 2. Input Methods: All Three

Support multiple input methods for user convenience:

| Method | Priority | Implementation |
|--------|----------|----------------|
| File Upload | P0 | `<input type="file" accept="image/*">` |
| Paste | P0 | `paste` event handler on chat input |
| Camera Capture | P1 | `navigator.mediaDevices.getUserMedia()` (mobile-friendly) |
| Drag-and-Drop | P2 | `dragover`/`drop` events on chat area |

### 3. Image Preview: Inline Chat Thumbnail

Show a small preview in the transcript before sending:

```
You (with image):
[thumbnail: 100x100px] What is this plant?

DAWN:
This appears to be a Monstera deliciosa...
```

### 4. Message Format

New JSON message type for vision requests:

```json
// Client -> Server
{
  "type": "vision",
  "payload": {
    "text": "What do you see in this image?",
    "image": {
      "data": "<base64-encoded-image>",
      "mime_type": "image/jpeg",
      "filename": "photo.jpg"  // optional
    }
  }
}

// Server -> Client (transcript with image reference)
{
  "type": "transcript",
  "payload": {
    "role": "user",
    "text": "What do you see in this image?",
    "image_preview": "<base64-thumbnail-50x50>"  // Optional small preview for history
  }
}
```

### 5. Security Considerations

| Risk | Mitigation |
|------|------------|
| File size DoS | Client-side: 10MB limit. Server-side: reject messages >15MB |
| Image bomb (decompression) | Server validates image header before LLM call |
| Non-image files | Validate MIME type and magic bytes |
| Base64 injection | Sanitize before passing to LLM |

**Recommended Limits:**
- Max image size: 10MB raw (before compression)
- Max base64 payload: 15MB (10MB * 1.33 + text)
- Max image dimensions: 4096x4096 (resize if larger)
- Supported formats: JPEG, PNG, GIF, WebP

---

## Implementation Plan

### Phase 1: Core Vision Support (MVP)

**Server-side (`webui_server.c`):**

```c
// New handler for vision messages
static void handle_vision_message(ws_connection_t *conn,
                                  const char *text,
                                  const char *image_base64,
                                  const char *mime_type);

// Validation helper
static bool validate_image_data(const char *base64_data,
                                size_t max_size,
                                char *error_buf,
                                size_t error_len);
```

1. Add `handle_vision_message()` in message dispatcher
2. Validate image data (size, format, magic bytes)
3. Call `session_llm_call_with_vision()` (new function)
4. Return response as normal transcript

**Client-side (`dawn.js`):**

1. Add file input button next to text input
2. Implement `handleImageFile()` for FileReader + base64
3. Show thumbnail preview before send
4. Send `vision` message type on submit

### Phase 2: Enhanced Input Methods

1. **Paste support**: Listen for `paste` event, extract `image/*` items
2. **Drag-and-drop**: Add drop zone overlay on chat area
3. **Camera capture** (mobile): `getUserMedia()` with canvas capture

### Phase 3: Polish

1. Image compression (client-side canvas resize)
2. Progress indicator for large images
3. History with image thumbnails
4. Clear pending image button

---

## File Changes

### Server-side

| File | Changes |
|------|---------|
| `include/webui/webui_server.h` | Add `WEBUI_MAX_IMAGE_SIZE` constant |
| `src/webui/webui_server.c` | Add `handle_vision_message()`, update `handle_json_message()` |
| `src/core/session_manager.c` | Add `session_llm_call_with_vision()` |
| `include/core/session_manager.h` | Declare new function |

### Client-side

| File | Changes |
|------|---------|
| `www/index.html` | Add image upload button, preview container |
| `www/js/dawn.js` | Add image handling functions, vision message type |
| `www/css/style.css` | Image preview styles, drop zone styles |

---

## API Reference

### New WebSocket Message Types

#### `vision` (Client -> Server)

Request AI analysis of an image with optional text prompt.

```json
{
  "type": "vision",
  "payload": {
    "text": "Describe this image",
    "image": {
      "data": "iVBORw0KGgoAAAANSUhEUgAA...",
      "mime_type": "image/png"
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | string | No | Text prompt (default: "What do you see?") |
| `image.data` | string | Yes | Base64-encoded image data |
| `image.mime_type` | string | Yes | MIME type (image/jpeg, image/png, etc.) |

#### Response

Vision requests return standard `transcript` messages:

```json
{
  "type": "transcript",
  "payload": {
    "role": "user",
    "text": "Describe this image",
    "has_image": true
  }
}
```

Followed by:

```json
{
  "type": "transcript",
  "payload": {
    "role": "assistant",
    "text": "This image shows a sunset over mountains..."
  }
}
```

### Error Responses

```json
{
  "type": "error",
  "payload": {
    "code": "VISION_NOT_SUPPORTED",
    "message": "Current LLM does not support vision",
    "recoverable": true
  }
}
```

| Error Code | Description |
|------------|-------------|
| `VISION_NOT_SUPPORTED` | LLM provider doesn't support vision |
| `IMAGE_TOO_LARGE` | Image exceeds size limit |
| `INVALID_IMAGE` | Corrupted or unsupported image format |

---

## Configuration

### TOML Settings (future)

```toml
[webui]
# Vision support
vision_enabled = true
vision_max_size_mb = 10
vision_allowed_formats = ["jpeg", "png", "gif", "webp"]
```

---

## Compatibility Matrix

| LLM Provider | Vision Support | Notes |
|--------------|----------------|-------|
| OpenAI (gpt-4o, gpt-4-turbo) | Yes | Full support |
| OpenAI (gpt-3.5-turbo) | No | Text only |
| Claude (claude-3-*) | Yes | Full support |
| Claude (claude-2) | No | Text only |
| Local (llama.cpp) | Depends | Requires vision model (LLaVA, etc.) |

The WebUI should check `vision_enabled` in session's resolved LLM config and show/hide the image upload button accordingly.

---

## Testing Plan

### Unit Tests

1. `test_validate_image_data()` - Size limits, format validation
2. `test_base64_decode()` - Valid/invalid base64 handling

### Integration Tests

1. Upload JPEG < 1MB -> successful vision response
2. Upload PNG with transparency -> handled correctly
3. Upload 15MB image -> rejected with `IMAGE_TOO_LARGE`
4. Upload non-image file -> rejected with `INVALID_IMAGE`
5. Vision request with non-vision LLM -> `VISION_NOT_SUPPORTED`
6. Paste image -> same as file upload

### Manual Tests

1. Mobile: camera capture works on iOS/Android
2. Desktop: drag-and-drop from file manager
3. Session reconnect: image in history shown correctly
4. Concurrent vision requests from multiple sessions

---

## Implementation Status

**Status: COMPLETE** (January 2026)

All planned features implemented:
- File upload, paste, drag-and-drop ✓
- Camera capture with live viewfinder ✓
- Multi-image support (up to 5) ✓
- Client-side compression ✓
- Vision capability detection ✓
- History with image thumbnails ✓

See `docs/WEBUI_VISION_NEXT_STEPS.md` for implementation details.

---

## Future Enhancements

1. **Image annotations**: Draw boxes/arrows before sending
2. **OCR mode**: Specialized prompt for text extraction
3. **Image generation**: Integrate DALL-E/Stable Diffusion responses
4. **Video frame capture**: Extract frames from video for analysis

---

## References

- OpenWebUI image handling: [MessageInput.svelte](https://github.com/open-webui/open-webui)
- [Home Assistant Voice Pipeline](https://developers.home-assistant.io/docs/voice/pipelines/)
- DAWN existing vision: `src/llm/llm_tools.c` (`execute_viewing_sync`)
- DAWN LLM vision: `src/llm/llm_openai.c`, `src/llm/llm_claude.c`
