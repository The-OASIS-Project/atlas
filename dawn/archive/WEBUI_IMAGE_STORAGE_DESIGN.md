# WebUI Image Storage Design

## Problem Statement

Images in conversation history are stored inline as base64 data URIs (`[IMAGE:data:image/jpeg;base64,...]`). This causes:

1. **WebSocket frame overflow** - Thumbnails up to 150KB exceed 16KB WebSocket buffer
2. **Bandwidth waste** - Images re-sent on every reconnect
3. **Scaling issues** - Message size grows unbounded with images

## Solution Overview

Separate image storage from message content:
- **HTTP for bulk data** - Image upload/download via REST endpoints
- **WebSocket for signaling** - Small JSON messages with image references
- **Server-side storage** - Images persist across sessions/devices

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Image Upload Flow                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Browser                        Server                     Storage      │
│  ───────                        ──────                     ───────      │
│     │                              │                          │         │
│     │  POST /api/images            │                          │         │
│     │  [multipart/form-data]       │                          │         │
│     │  ─────────────────────────>  │                          │         │
│     │                              │   Write file             │         │
│     │                              │  ───────────────────────> │         │
│     │                              │                          │         │
│     │  {"id": "img_a1b2c3"}        │                          │         │
│     │  <─────────────────────────  │                          │         │
│     │                              │                          │         │
│     │  WebSocket: send message     │                          │         │
│     │  {..., images: ["img_a1b2c3"]}                          │         │
│     │  ─────────────────────────>  │                          │         │
│     │                              │                          │         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                       History Restore Flow                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Browser                        Server                     Storage      │
│  ───────                        ──────                     ───────      │
│     │                              │                          │         │
│     │  WebSocket: restore history  │                          │         │
│     │  <─────────────────────────  │                          │         │
│     │  {..., images: ["img_a1b2c3"]}  (small JSON)            │         │
│     │                              │                          │         │
│     │  GET /api/images/img_a1b2c3  │                          │         │
│     │  ─────────────────────────>  │   Read file              │         │
│     │                              │  ───────────────────────> │         │
│     │                              │  <─────────────────────── │         │
│     │  [image/jpeg binary]         │                          │         │
│     │  <─────────────────────────  │                          │         │
│     │                              │                          │         │
│     │  (cache in localStorage)     │                          │         │
│     │                              │                          │         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## API Design

### POST /api/images

Upload an image and receive an ID.

**Request:**
```http
POST /api/images HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
Cookie: dawn_session=...

------WebKitFormBoundary
Content-Disposition: form-data; name="image"; filename="photo.jpg"
Content-Type: image/jpeg

[binary image data]
------WebKitFormBoundary--
```

**Response (success):**
```json
{
  "id": "img_a1b2c3d4e5f6",
  "mime_type": "image/jpeg",
  "size": 45032,
  "created_at": "2026-01-22T07:45:00Z"
}
```

**Response (error):**
```json
{
  "error": "Image too large",
  "max_size": 4194304
}
```

**Validation:**
- Authenticated users only
- Max size: 4MB (configurable)
- Allowed types: image/jpeg, image/png, image/gif, image/webp
- **No SVG** (XSS risk)

---

### GET /api/images/:id

Retrieve an image by ID.

**Request:**
```http
GET /api/images/img_a1b2c3d4e5f6 HTTP/1.1
Cookie: dawn_session=...
```

**Response (success):**
```http
HTTP/1.1 200 OK
Content-Type: image/jpeg
Content-Length: 45032
Cache-Control: private, max-age=31536000

[binary image data]
```

**Response (not found):**
```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{"error": "Image not found"}
```

**Security:**
- User can only access their own images
- Or images from conversations they're part of (future multi-user)

---

## Storage Design

### SQLite BLOB Storage

Images are stored directly in the SQLite database as BLOBs:

```sql
CREATE TABLE images (
    id TEXT PRIMARY KEY,          -- img_a1b2c3d4e5f6
    user_id INTEGER NOT NULL,
    mime_type TEXT NOT NULL,
    size INTEGER NOT NULL,
    data BLOB NOT NULL,           -- actual image binary
    created_at INTEGER NOT NULL,
    last_accessed INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

**Rationale:**
- Single source of truth (no filesystem/DB sync issues)
- Atomic transactions (image + metadata together)
- Simpler backup (just the .db file)
- Thumbnails are ~150KB, well within SQLite's comfort zone
- No directory management needed

### Image ID Format

```
img_{12 chars alphanumeric}
```

Example: `img_a1b2c3d4e5f6`

**Generation:**
```c
// Use random bytes, base62 encoded
void generate_image_id(char *out, size_t len) {
    static const char charset[] = "0123456789"
                                  "abcdefghijklmnopqrstuvwxyz"
                                  "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    unsigned char random[12];
    getrandom(random, sizeof(random), 0);
    out[0] = 'i'; out[1] = 'm'; out[2] = 'g'; out[3] = '_';
    for (int i = 0; i < 12; i++) {
        out[4 + i] = charset[random[i] % 62];
    }
    out[16] = '\0';
}
```

### Storage Benefits

- **Atomic transactions** - Image data and metadata stored together
- **Easy access control** - User ownership check via simple query
- **Cleanup queries** - Delete old images with single SQL statement
- **Simpler deployment** - No filesystem paths or permissions to manage
- **Backup simplicity** - Single database file contains everything

---

## Message Format Change

### Before (inline base64)
```
What's in this photo?
[IMAGE:data:image/jpeg;base64,/9j/4AAQSkZJRg...]
```

### After (ID reference)
```
What's in this photo?
[IMAGE:img_a1b2c3d4e5f6]
```

### Backward Compatibility

During transition, support both formats:

```javascript
function parseImageMarkers(content) {
    const images = [];

    // New format: [IMAGE:img_xxx]
    const idRegex = /\[IMAGE:(img_[a-zA-Z0-9]+)\]/g;
    // Legacy format: [IMAGE:data:image/...]
    const dataRegex = /\[IMAGE:(data:image\/[^\]]+)\]/g;

    // Parse both, return unified structure
    // ...
}
```

---

## Client-Side Changes

### Vision Module (vision.js)

**Current flow:**
1. Capture/select image
2. Compress to thumbnail
3. Store base64 in `pendingImages`
4. Embed in message on send

**New flow:**
1. Capture/select image
2. Compress to thumbnail
3. **Upload via HTTP, receive ID**
4. Store ID in `pendingImages`
5. Embed ID in message on send
6. Cache image in localStorage by ID

```javascript
async function uploadImage(blob) {
    const formData = new FormData();
    formData.append('image', blob);

    const response = await fetch('/api/images', {
        method: 'POST',
        credentials: 'include',
        body: formData
    });

    if (!response.ok) {
        throw new Error('Upload failed');
    }

    return await response.json(); // { id: "img_...", ... }
}
```

### History Display (history.js)

**Current flow:**
1. Parse `[IMAGE:data:...]` from message
2. Render inline

**New flow:**
1. Parse `[IMAGE:img_...]` from message
2. Check localStorage cache
3. If cached, render immediately
4. If not cached, fetch via HTTP, cache, render

```javascript
async function loadImage(imageId) {
    // Check cache first
    const cached = localStorage.getItem(`dawn_img_${imageId}`);
    if (cached) {
        return cached; // data URL
    }

    // Fetch from server
    const response = await fetch(`/api/images/${imageId}`, {
        credentials: 'include'
    });

    if (!response.ok) {
        return null; // Show placeholder
    }

    const blob = await response.blob();
    const dataUrl = await blobToDataUrl(blob);

    // Cache for future
    try {
        localStorage.setItem(`dawn_img_${imageId}`, dataUrl);
    } catch (e) {
        // localStorage full, continue without caching
    }

    return dataUrl;
}
```

---

## Server-Side Changes

### New Files

| File | Purpose |
|------|---------|
| `src/webui/webui_images.c` | Image upload/download handlers |
| `include/webui/webui_images.h` | Public API |
| `include/image_store.h` | Image storage abstraction |
| `src/image_store.c` | SQLite BLOB operations |

### HTTP Handler Integration

In `webui_http.c`:

```c
static int handle_api_images(struct lws *wsi, const char *path,
                             enum http_method method,
                             webui_session_t *session) {
    if (method == HTTP_POST && strcmp(path, "/api/images") == 0) {
        return handle_image_upload(wsi, session);
    }

    if (method == HTTP_GET && strncmp(path, "/api/images/", 12) == 0) {
        const char *image_id = path + 12;
        return handle_image_download(wsi, session, image_id);
    }

    return 404;
}
```

### Database Schema Addition

In `src/db/schema.sql` or initialization:

```sql
CREATE TABLE IF NOT EXISTS images (
    id TEXT PRIMARY KEY,
    user_id INTEGER NOT NULL,
    mime_type TEXT NOT NULL,
    size INTEGER NOT NULL,
    created_at INTEGER DEFAULT (strftime('%s', 'now')),
    last_accessed INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_images_user ON images(user_id);
CREATE INDEX idx_images_created ON images(created_at);
```

---

## Configuration

In `dawn.toml`:

```toml
[webui.images]
enabled = true
max_size_bytes = 4194304          # 4MB max upload
max_per_user = 1000               # Max images per user
retention_days = 90               # Auto-delete after 90 days (0 = forever)
```

---

## Cleanup/Retention

### Automatic Cleanup

Background task or periodic cleanup:

```c
int image_store_cleanup(void) {
    time_t cutoff = time(NULL) - (retention_days * 86400);

    // Delete old images in single SQL statement
    sqlite3_reset(s_db.stmt_image_delete_old);
    sqlite3_bind_int64(s_db.stmt_image_delete_old, 1, cutoff);
    sqlite3_step(s_db.stmt_image_delete_old);

    return sqlite3_changes(s_db.db);
}
```

### Manual Cleanup API

```http
DELETE /api/images/img_a1b2c3d4e5f6
```

For user-initiated deletion.

---

## Migration Plan

### Phase 1: Add New System (Parallel)
1. Implement image upload/download endpoints
2. Add database table
3. Update vision.js to upload new images
4. Keep inline format support for display

### Phase 2: Client Migration
1. New images use ID format
2. Old images still display from inline data
3. No breaking changes

### Phase 3: Server Migration (Optional)
1. Background job to extract inline images
2. Upload to storage, replace with IDs
3. Only if storage savings needed

### Phase 4: Deprecate Inline (Future)
1. Remove inline image support from new code
2. Legacy messages still work (read-only)

---

## Security Considerations

1. **Authentication required** for all image endpoints
2. **User isolation** - users can only access their own images
3. **No SVG uploads** - XSS prevention
4. **Content-Type validation** - verify actual file type matches header
5. **Size limits** - prevent DoS via large uploads
6. **Rate limiting** - prevent abuse (future)
7. **Path traversal prevention** - validate image IDs strictly

```c
bool is_valid_image_id(const char *id) {
    if (strncmp(id, "img_", 4) != 0) return false;
    if (strlen(id) != 16) return false;

    for (int i = 4; i < 16; i++) {
        char c = id[i];
        if (!isalnum(c)) return false;
    }
    return true;
}
```

---

## Implementation Order

1. **Database schema** - Add images table
2. **Storage module** - `image_store.c` with save/load/delete
3. **HTTP endpoints** - POST and GET handlers
4. **Client upload** - vision.js uploads images
5. **Client display** - history.js fetches by ID
6. **Cleanup job** - Retention policy
7. **Migration tool** - Convert existing inline images (optional)

---

## Estimated Effort

| Component | Effort |
|-----------|--------|
| Database schema | 0.5 day |
| Storage module | 1 day |
| HTTP endpoints | 1 day |
| Client changes | 1 day |
| Testing | 1 day |
| **Total** | **4-5 days** |

---

## Alternatives Considered

### 1. Increase WebSocket buffer
- **Pros:** Simple, no architecture change
- **Cons:** Doesn't scale, wastes memory, still inefficient

### 2. Fragment large messages
- **Pros:** No new endpoints
- **Cons:** Complex reassembly, protocol changes

### 3. External blob storage (S3, etc.)
- **Pros:** Infinite scale, CDN support
- **Cons:** Overkill for local assistant, adds dependency

### 4. Keep images in localStorage only
- **Pros:** Simplest
- **Cons:** Lost on clear, no cross-device, poor UX

**Chosen approach:** Server-side SQLite BLOB storage with HTTP endpoints. Balances simplicity with proper architecture. BLOB storage chosen over filesystem for atomic transactions and simpler deployment.
