# Conversation History Design

## Overview

Per-user conversation history allowing users to view, resume, search, and manage their past conversations.

**Status:** ✅ Complete (2026-01-04)

---

## UI Design

### Panel Location

**Left slide-out panel** - mirrors Settings panel (right) creating spatial symmetry:
- Left = User content (history)
- Right = System settings

### Trigger

New history button in `#header-left` after user badge:
```
[D.A.W.N.] [user badge] [history icon]    [debug] [metrics] [settings] [logout]
```

### Panel Structure

```
+------------------------------------------+
|  Conversations              [+ New] [X]  |  <- Header
+------------------------------------------+
|  [Search conversations...]               |  <- Search (P2)
+------------------------------------------+
|  TODAY                                   |  <- Date grouping
|  +--------------------------------------+|
|  | What is photosynthesis?         [...] |  <- Conversation item
|  | 2:34 PM - 12 messages                ||     (actions on hover)
|  +--------------------------------------+|
|  | Smart home automation setup          ||
|  | 11:15 AM - 8 messages                ||
|  +--------------------------------------+|
+------------------------------------------+
|  YESTERDAY                               |
|  ...                                     |
+------------------------------------------+
```

### Conversation Item Display

Each item shows:
- **Title** - Auto-generated from first user message (truncated with ellipsis)
- **Timestamp** - Relative for recent ("2 hours ago"), absolute for older
- **Message count** - Quick depth indicator
- **Active indicator** - Cyan left border + glow for current conversation

### Hover Actions

- **Rename** - Edit auto-generated title
- **Delete** - With confirmation modal

### Visual Treatment

- Same glassmorphism as Settings panel (`rgba(18, 18, 26, 0.92)`, `backdrop-filter: blur(20px)`)
- Width: 380px (slightly narrower than Settings)
- Slide animation from left
- Active conversation: cyan accent border + subtle glow

### Mobile (< 600px)

- Full-screen overlay (100% width)
- Larger touch targets (44px minimum)
- Actions always visible (no hover state)
- Search input 16px font (prevent iOS zoom)

---

## User Interactions

### Opening History Panel

1. Click history button in header
2. Panel slides in from left
3. Overlay dims main content
4. Conversations load from server

### Switching Conversations

| Scenario | Behavior |
|----------|----------|
| Empty current chat | Switch immediately |
| Current chat has content | Auto-save, switch immediately |
| AI mid-response | Confirm dialog: "Response in progress. Switch anyway?" |

### New Conversation

- Button in panel header
- Clears transcript area
- Sends `new_conversation` message to server
- New item appears at top of list (titled "New conversation" until first message)

### Search (Priority 2)

- Simple substring search across conversation titles
- Real-time filter as user types
- "No matches" empty state

### Delete Conversation

- Confirmation modal (existing pattern)
- Remove from list
- If deleting active conversation, start new one

---

## Backend Design

### Database Schema

```sql
-- Conversations table - metadata per conversation
CREATE TABLE IF NOT EXISTS conversations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    title TEXT NOT NULL DEFAULT 'New Conversation',
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    message_count INTEGER DEFAULT 0,
    is_archived INTEGER DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Indexes for efficient queries
CREATE INDEX IF NOT EXISTS idx_conversations_user ON conversations(user_id, updated_at DESC);
CREATE INDEX IF NOT EXISTS idx_conversations_search ON conversations(user_id, title);

-- Messages table - individual messages within conversations
CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    conversation_id INTEGER NOT NULL,
    role TEXT NOT NULL CHECK(role IN ('system', 'user', 'assistant')),
    content TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE
);

-- Index for loading conversation messages in order
CREATE INDEX IF NOT EXISTS idx_messages_conversation ON messages(conversation_id, id ASC);
```

### C Data Structures

```c
#define CONV_TITLE_MAX 256
#define CONV_MESSAGE_MAX 65536  /* 64KB max message content */

typedef struct {
    int64_t id;
    int user_id;
    char title[CONV_TITLE_MAX];
    time_t created_at;
    time_t updated_at;
    int message_count;
    bool is_archived;
} conversation_t;

typedef struct {
    int64_t id;
    int64_t conversation_id;
    char role[16];              /* "system", "user", "assistant" */
    char *content;              /* Dynamically allocated */
    time_t created_at;
} conversation_message_t;
```

### Core Functions (conversation_db.h)

```c
/* Lifecycle */
int conversation_db_init(void);
void conversation_db_shutdown(void);

/* Conversation CRUD */
int conversation_db_create(int user_id, const char *title, int64_t *conv_id_out);
int conversation_db_get(int64_t conv_id, int user_id, conversation_t *conv_out);
int conversation_db_list(int user_id, bool include_archived,
                         const pagination_t *pagination,
                         conversation_callback_t callback, void *ctx);
int conversation_db_rename(int64_t conv_id, int user_id, const char *new_title);
int conversation_db_delete(int64_t conv_id, int user_id);
int conversation_db_search(int user_id, const char *query,
                           const pagination_t *pagination,
                           conversation_callback_t callback, void *ctx);

/* Message operations */
int conversation_db_add_message(int64_t conv_id, int user_id,
                                const char *role, const char *content);
int conversation_db_get_messages(int64_t conv_id, int user_id,
                                 message_callback_t callback, void *ctx);
int conversation_db_get_messages_json(int64_t conv_id, int user_id,
                                      struct json_object **history_out);

/* Auto-titling */
void conversation_generate_title(const char *content, char *title_out, size_t max_len);
```

### Session Manager Integration

Add to `session_t`:
```c
int64_t active_conversation_id;  /* 0 = no persistent conversation */
int conversation_user_id;        /* User ID who owns the conversation */
```

New functions:
```c
int session_attach_conversation(session_t *session, int64_t conv_id, int user_id);
int64_t session_save_conversation(session_t *session, int user_id);
int session_add_message_persistent(session_t *session, const char *role, const char *content);
```

### WebSocket Messages

| Message | Direction | Purpose |
|---------|-----------|---------|
| `new_conversation` | → Server | Create new conversation |
| `new_conversation_response` | ← Server | Returns `{ conversation_id: 123 }` |
| `load_conversation` | → Server | Load conversation by ID |
| `load_conversation_response` | ← Server | Success + history replay |
| `list_conversations` | → Server | List with pagination |
| `list_conversations_response` | ← Server | `{ conversations: [...], total, has_more }` |
| `delete_conversation` | → Server | Delete by ID |
| `delete_conversation_response` | ← Server | Success/failure |
| `rename_conversation` | → Server | Update title |
| `rename_conversation_response` | ← Server | Success/failure |
| `search_conversations` | → Server | Search by title |
| `search_conversations_response` | ← Server | Matching conversations |

### Auto-Save Flow

Messages are persisted automatically during chat:

1. User sends message → create conversation if needed → save user message
2. LLM responds → save assistant message
3. Conversation `updated_at` and `message_count` updated

### Cleanup

- **CASCADE on user delete**: All user's conversations and messages deleted automatically
- **Authorization**: Every operation validates `user_id` matches conversation owner
- **Rate limits**: Consider `MAX_CONVERSATIONS_PER_USER = 100`

### Error Codes

```c
#define CONV_DB_SUCCESS 0
#define CONV_DB_FAILURE 1
#define CONV_DB_NOT_FOUND 2
#define CONV_DB_FORBIDDEN 3      /* User doesn't own conversation */
#define CONV_DB_INVALID 4
#define CONV_DB_LIMIT_EXCEEDED 5
```

---

## Files to Modify

| File | Changes |
|------|---------|
| `www/index.html` | Add history panel HTML, history button |
| `www/css/dawn.css` | History panel styles (~200 lines) |
| `www/js/dawn.js` | History panel logic, message handlers |
| `include/auth/auth_db.h` | Conversation structures and functions |
| `src/auth/auth_db.c` | Database schema, conversation CRUD |
| `src/webui/webui_server.c` | WebSocket message handlers |

---

## Implementation Phases

### Phase 1: Core Functionality ✅
- [x] Database schema for conversations
- [x] Backend CRUD operations
- [x] WebSocket handlers
- [x] Basic UI panel (list, switch, new)

### Phase 2: Polish ✅
- [x] Search functionality
- [x] Rename conversations
- [x] Delete with confirmation
- [x] Auto-generate titles from first message

### Phase 3: Enhancements (Future)
- [ ] Conversation export
- [x] Pagination for long history
- [ ] Auto-cleanup of old conversations

---

## Open Questions (Resolved)

1. **Message storage format** - Store raw messages or include LLM responses?
   - **Resolution:** Store both user and assistant messages. System prompts not stored.
2. **Title generation** - Client-side truncation or server-side with LLM summarization?
   - **Resolution:** Server-side truncation of first user message (50 chars with ellipsis)
3. **Pagination** - How many conversations to load initially?
   - **Resolution:** 50 conversations per page with "Load More" button
4. **Conversation limit** - Max conversations per user? Max messages per conversation?
   - **Resolution:** No hard limits imposed; soft limit of 100 conversations per user
