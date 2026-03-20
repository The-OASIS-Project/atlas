# IMAP/SMTP Email Integration — Design Document

**Created**: March 9, 2026
**Implemented**: March 13, 2026
**Status**: Complete
**CMake option**: `DAWN_ENABLE_EMAIL_TOOL` (OFF by default)

---

## Overview

Multi-account email integration for DAWN via IMAP (read/search) and SMTP (send). Users configure accounts through the WebUI and interact via voice: "Read my recent emails", "Send an email to Bob", "Any emails from Alice this week?"

Uses standard IMAP/SMTP protocols via libcurl — works with any provider (Gmail, iCloud, Outlook, Fastmail, self-hosted, corporate Exchange via IMAP). Supports both app password authentication and Google OAuth 2.0 (XOAUTH2 via the shared `oauth_client.c` module). No vendor lock-in.

### Design Decisions

| Decision           | Choice                                                        | Rationale                                                                                           |
| ------------------ | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Contact resolution | Contacts table (entity extension)                             | Structured data, not fragile fact strings; supports multi-value; disambiguation for duplicate names |
| Accounts           | Multi-account (DB-stored)                                     | Mirrors CalDAV pattern; per-user isolation; encrypted passwords via `crypto_store`                  |
| Account selection  | By name (string match)                                        | LLM gets names from `accounts` action, passes them back; position-independent                       |
| Auth types         | App password + Google OAuth                                   | Covers most providers; OAuth uses shared `oauth_client.c` with PKCE S256                            |
| Send confirmation  | Two-step (draft → confirm)                                    | Email is irreversible; voice UX benefits from read-back                                             |
| Content handling   | Headers + preview for listings; plain text for full read      | Fits tool result buffer; matches voice interaction patterns                                         |
| Caching            | No cache (live IMAP queries)                                  | Email changes constantly; one-shot queries don't benefit from caching                               |
| Architecture       | Three layers (tool → service → client)                        | Service handles multi-account routing + auth dispatch; client handles raw IMAP/SMTP                 |
| Read-only accounts | Per-account `read_only` flag                                  | Blocks send/confirm_send; useful for monitoring accounts                                            |
| Trash/archive      | Two-step trash (pending → confirm), single-step archive       | Trash is destructive (confirmation needed); archive is non-destructive (immediate)                   |
| Gmail API          | Dual backend: Gmail REST API + IMAP                           | Gmail API for OAuth accounts (faster, no IMAP); IMAP for all other providers                        |
| Pagination         | pageToken/nextPageToken for result sets                       | Large inboxes need paging; LLM iterates naturally                                                   |
| Scope              | 10 actions: accounts, recent, read, search, folders, send, confirm_send, trash, confirm_trash, archive | Full inbox management with safety; reply/forward/attachments deferred                               |

---

## Prerequisites: Contacts System

Email depends on a structured contacts system for recipient resolution. This is an extension of the existing entity graph and is independently useful (phone numbers, addresses, "What's Bob's number?").

### Database Schema

```sql
-- v26 migration in auth_db_core.c
CREATE TABLE contacts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    entity_id INTEGER NOT NULL,
    field_type TEXT NOT NULL,       -- 'email', 'phone', 'address'
    value TEXT NOT NULL,
    label TEXT DEFAULT '',          -- 'work', 'personal', 'mobile'
    created_at INTEGER NOT NULL,
    FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY(entity_id) REFERENCES memory_entities(id) ON DELETE CASCADE
);

CREATE INDEX idx_contacts_entity ON contacts(entity_id);
CREATE INDEX idx_contacts_user_type ON contacts(user_id, field_type);
```

### Contact Lookup API (`include/memory/contacts_db.h`)

```c
typedef struct {
   int64_t contact_id;
   int64_t entity_id;
   char entity_name[64];
   char canonical_name[64];
   char field_type[16];
   char value[256];
   char label[32];
} contact_result_t;

int contacts_find(int user_id, const char *name, const char *field_type,
                  contact_result_t *out, int max_results);
int contacts_add(int user_id, int64_t entity_id, const char *field_type,
                 const char *value, const char *label);
int contacts_delete(int user_id, int64_t contact_id);
int contacts_list(int user_id, const char *field_type,
                  contact_result_t *out, int max_results);
```

Name search uses case-insensitive `LIKE '%name%'` with proper LIKE metacharacter escaping (`%`, `_`, `\` escaped with `ESCAPE '\'`).

### Disambiguation Flow

```
contacts_find(user_id, "Bob", "email", results, 5)
  → 0 results: "No email address found for 'Bob'"
  → 1 result:  proceed with that address
  → 2+ results: return all matches, LLM asks user to clarify
```

### Contacts in the Memory Tool

Four new actions added to the existing memory tool (not a separate tool):

```c
"save_contact"    // Save contact info for an entity
"find_contact"    // Look up contact info by name
"list_contacts"   // List all contacts (optionally filtered by type)
"delete_contact"  // Remove a contact entry
```

---

## Email Architecture

### Component Layout

```
┌──────────────────────────────────────┐
│          email_tool.c                │  ← LLM tool (10 actions, formatting)
│  register, callback, disambiguation │
└──────────────┬───────────────────────┘
               │
┌──────────────▼───────────────────────┐
│        email_service.c               │  ← Multi-account routing, auth, drafts
│  account resolution, conn builder,   │
│  draft CRUD, confirm throttling      │
└──────────────┬───────────────────────┘
               │
┌──────────────▼───────────────────────┐
│         email_client.c               │  ← IMAP/SMTP via libcurl
│  fetch, search, send, trash,         │
│  archive, folders, test_conn         │
└──────────────────────────────────────┘
               │
┌──────────────▼───────────────────────┐
│         gmail_client.c               │  ← Gmail REST API (OAuth accounts)
│  fetch, search, read, send, trash,   │
│  archive, folders, labels            │
└──────────────────────────────────────┘
               │
┌──────────────▼───────────────────────┐
│          email_db.c                  │  ← Account CRUD (SQLite)
│  create, get, list, update, delete,  │
│  set_read_only, set_enabled          │
└──────────────────────────────────────┘
```

Three layers: tool → service → client(s), with a separate DB layer. Service handles account selection (by name or email address), auth dispatch (app password vs OAuth), read-only enforcement, draft management, and pending trash management. Two client backends: `gmail_client.c` (REST API for OAuth Gmail accounts) and `email_client.c` (IMAP/SMTP for all other providers). Service routes automatically based on account type.

---

## Email Account Storage (`email_db.c`)

### Database Schema

```sql
-- v26 migration in auth_db_core.c (same transaction as contacts)
CREATE TABLE email_accounts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    name TEXT NOT NULL,
    imap_server TEXT NOT NULL,
    imap_port INTEGER DEFAULT 993,
    imap_ssl INTEGER DEFAULT 1,
    smtp_server TEXT NOT NULL,
    smtp_port INTEGER DEFAULT 465,
    smtp_ssl INTEGER DEFAULT 1,
    username TEXT NOT NULL,
    display_name TEXT DEFAULT '',
    encrypted_password BLOB,
    encrypted_password_len INTEGER DEFAULT 0,
    auth_type TEXT DEFAULT 'app_password',
    oauth_account_key TEXT DEFAULT '',
    enabled INTEGER DEFAULT 1,
    read_only INTEGER DEFAULT 0,
    max_recent INTEGER DEFAULT 10,
    max_body_chars INTEGER DEFAULT 4000,
    created_at INTEGER NOT NULL,
    FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
);
CREATE INDEX idx_email_acct_user ON email_accounts(user_id);
```

### Account Struct

```c
typedef struct {
   int64_t id;
   int user_id;
   char name[128];              // "Gmail", "Work", etc.
   char imap_server[256];
   int imap_port;               // 993 default
   bool imap_ssl;
   char smtp_server[256];
   int smtp_port;               // 465 default
   bool smtp_ssl;
   char username[128];
   char display_name[64];       // For From: header
   uint8_t encrypted_password[384];
   int encrypted_password_len;
   char auth_type[16];          // "app_password" or "oauth"
   char oauth_account_key[128];
   bool enabled;
   bool read_only;
   int max_recent;              // Default 10
   int max_body_chars;          // Default 4000
   time_t created_at;
} email_account_t;
```

Passwords are encrypted at rest via `crypto_store_encrypt/decrypt` (shared libsodium `crypto_secretbox` module using `dawn.key`).

---

## Email Client (`email_client.c`)

All operations use libcurl's built-in IMAP/SMTP protocol support.

### Connection Types

```c
typedef struct {
   char imap_url[512];       // "imaps://imap.gmail.com:993"
   char smtp_url[512];       // "smtps://smtp.gmail.com:465"
   char username[128];
   char password[256];       // Empty when using OAuth
   char bearer_token[512];   // OAuth access token
   char display_name[64];
   int max_body_chars;
} email_conn_t;

typedef struct {
   uint32_t uid;
   char from_name[64];
   char from_addr[256];
   char subject[256];
   char date_str[32];
   time_t date;
   char preview[512];
} email_summary_t;

typedef struct {
   uint32_t uid;
   char from_name[64];
   char from_addr[256];
   char to[256];
   char subject[256];
   char date_str[32];
   char *body;              // Heap-allocated, caller frees via email_message_free()
   int body_len;
   int attachment_count;
   bool truncated;
} email_message_t;
```

### Client Functions

```c
int email_fetch_recent(const email_conn_t *conn, int count,
                       email_summary_t *out, int max_out, int *out_count);
int email_read_message(const email_conn_t *conn, uint32_t uid,
                       email_message_t *out);
int email_search(const email_conn_t *conn, const email_search_params_t *params,
                 email_summary_t *out, int max_out, int *out_count);
int email_send(const email_conn_t *conn, const char *to_addr, const char *to_name,
               const char *subject, const char *body);
int email_test_connection(const email_conn_t *conn, bool *imap_ok, bool *smtp_ok);
void email_message_free(email_message_t *msg);
```

### Security Measures

- **IMAP command injection prevention**: User-provided values in SEARCH commands use IMAP literal syntax `{N}\r\n<data>` — length-prefixed, cannot be escaped out of
- **IMAP date validation**: Date params parsed via `strptime()` into `struct tm`, reconstructed via `strftime("%d-%b-%Y")` — raw date strings never passed to IMAP
- **SMTP header injection prevention**: `sanitize_header_value()` strips CR/LF from all values used in email headers (subject, to_name, display_name, to_addr)
- **TLS enforcement**: Refuse plaintext IMAP/SMTP to non-loopback servers; `CURLOPT_USE_SSL = CURLUSESSL_ALL` for STARTTLS ports (587, 143)
- **TLS certificate verification**: Explicit `CURLOPT_SSL_VERIFYPEER=1` and `CURLOPT_SSL_VERIFYHOST=2` on all curl handles
- **Credential wiping**: All functions use `sodium_memzero()` on `email_conn_t` on all code paths (success and error)

### Authentication

| Provider    | IMAP Server               | SMTP Server             | Auth                         |
| ----------- | ------------------------- | ----------------------- | ---------------------------- |
| Gmail       | imap.gmail.com:993        | smtp.gmail.com:587      | App password or Google OAuth |
| iCloud      | imap.mail.me.com:993      | smtp.mail.me.com:587    | App-specific password        |
| Outlook/365 | outlook.office365.com:993 | smtp.office365.com:587  | App password                 |
| Fastmail    | imap.fastmail.com:993     | smtp.fastmail.com:465   | App password                 |
| Proton Mail | 127.0.0.1:1143 (Bridge)   | 127.0.0.1:1025 (Bridge) | Bridge password              |

**Google OAuth**: Uses `CURLOPT_XOAUTH2_BEARER` + `CURLOPT_LOGIN_OPTIONS "AUTH=XOAUTH2"` for IMAP/SMTP. Calendar and email use **separate token sets** keyed `email:<email>` vs `calendar:<email>` for revocation isolation. See `docs/GOOGLE_OAUTH_SETUP.md` Section 7.

---

## Email Service (`email_service.c`)

### Account Resolution

- `find_account(user_id, account_name)` — case-insensitive match on account name; `NULL` returns first enabled account
- `find_writable_account(user_id)` — first enabled, non-read-only account
- Account arrays are wiped with `sodium_memzero()` after use (encrypted passwords on stack)

### Auth Dispatch

```c
static int build_conn_for_account(const email_account_t *acct, email_conn_t *conn) {
   // Enforce TLS for non-loopback
   // Build IMAP/SMTP URLs from server/port/ssl
   if (auth_type == "oauth") {
      // oauth_get_access_token() → conn->bearer_token
   } else {
      // crypto_store_decrypt() → conn->password
   }
}
```

### Draft Management (Send)

```c
#define EMAIL_MAX_DRAFTS 4
#define EMAIL_DRAFT_EXPIRY_SEC 300     // 5 minutes
#define EMAIL_MAX_BODY_LEN 4000
#define EMAIL_MAX_SUBJECT_LEN 250
#define EMAIL_CONFIRM_MAX_FAILURES 3
#define EMAIL_CONFIRM_LOCKOUT_SEC 60

typedef struct {
   char draft_id[16];        // hex-encoded randombytes_buf() (7 random bytes)
   int user_id;
   char to_address[256];
   char to_name[64];
   char subject[256];
   char body[4096];
   time_t created_at;
   bool used;
} email_draft_t;
```

- Drafts are in-memory only (static array, mutex-protected)
- `draft_id` generated via `randombytes_buf()` from libsodium (not `rand()`)
- `confirm_send` validates that caller's `user_id` matches the draft's `user_id` (user-bound, not session-bound — avoids orphaning on WebSocket reconnect)
- Failed confirmation attempts are tracked per-user with mutex protection: 3 failures within 60 seconds triggers lockout
- Field length validation: body > 4000 chars or subject > 250 chars returns error (no silent truncation)
- Expired/used drafts are wiped with `sodium_memzero()`

### Pending Trash Management (Delete)

```c
#define EMAIL_MAX_PENDING_TRASH 10
#define EMAIL_PENDING_TRASH_EXPIRY_SEC 300  // 5 minutes

typedef struct {
   char pending_id[16];      // hex-encoded randombytes_buf()
   int user_id;
   char message_id[192];     // Gmail hex ID or IMAP folder:uid composite
   char account_name[128];   // Resolved account name for confirm step
   char subject[256];        // For confirmation display
   char from[128];           // For confirmation display
   time_t created_at;
   bool used;
} email_pending_trash_t;
```

- Same pattern as drafts: in-memory static array, mutex-protected, hex IDs, user-bound
- `trash` fetches message metadata (subject, from) so the LLM can read it back for confirmation
- `confirm_trash` routes to Gmail API or IMAP based on account type
- IMAP trash: `UID COPY` to `[Gmail]/Trash` or `Trash` → `UID STORE +FLAGS (\Deleted)` → `EXPUNGE`
- Gmail API trash: `POST /messages/{id}/trash`
- Shares the throttle mechanism with `confirm_send`

---

## Tool Registration (`email_tool.c`)

```c
static const treg_param_t email_params[] = {
   {
       .name = "action",
       .type = TOOL_PARAM_TYPE_ENUM,
       .enum_values = { "recent", "read", "search", "folders", "send", "confirm_send",
                        "accounts", "trash", "confirm_trash", "archive" },
       .enum_count = 10,
   },
   {
       .name = "details",
       .type = TOOL_PARAM_TYPE_STRING,
       .description = "JSON: recent {count?, account?, folder?, unread_only?, page_token?}, "
                      "read {message_id, account?}, "
                      "search {from?, subject?, text?, since?, before?, account?, page_token?}, "
                      "folders {account?}, "
                      "send {to, subject, body}, confirm_send {draft_id}, "
                      "trash {message_id, account?}, confirm_trash {pending_id}, "
                      "archive {message_id, account?}",
   },
};
```

- **Capabilities**: `TOOL_CAP_NETWORK | TOOL_CAP_DANGEROUS`
- **CMake**: `DAWN_ENABLE_EMAIL_TOOL=OFF` by default (compile-time gate)
- **Config**: requires `enabled = true` in `[email]` section of `dawn.toml` (runtime gate)
- **Double safety**: disabled at compile time AND runtime by default

---

## Action Details

### `accounts` — List Configured Accounts

**Input**: (none)

**Output**:

```
Email accounts (2):
- Gmail (Google OAuth) (user@gmail.com)
- Work [READ-ONLY] (user@company.com)
```

### `recent` — List Recent Emails

**Input**: `{count: 10, account: "Gmail"}` (both optional)

**Output**:

```
Recent emails (3):

1. From: Alice Johnson alice@acme.com
   Subject: Q1 Report Draft
   Date: Mon, 09 Mar 2026 14:15:00 -0500
   [UID: 4523]

2. From: GitHub noreply@github.com
   Subject: [dawn] New issue #42
   Date: Mon, 09 Mar 2026 11:30:00 -0500
   [UID: 4521]
```

UIDs are included for follow-up `read` actions.

### `read` — Full Email Content

**Input**: `{uid: 4523, account: "Gmail"}` (account optional)

Fetches full message by UID. Plain text extracted from body. Truncated at `max_body_chars` with notice. Attachments counted but not downloaded.

### `search` — Find Emails

**Input**: `{from: "alice", since: "2026-03-01", account: "Gmail"}`

Maps to IMAP SEARCH with literal syntax for safety. Dates validated via `strptime()`/`strftime()`. Returns up to 25 results (most recent first).

### `send` — Draft Email (Does NOT Send)

**Input**: `{to: "Bob", subject: "Meeting rescheduled", body: "Meeting rescheduled to 3pm."}`

**Contact resolution**:

- If `to` contains `@` → use as-is
- Otherwise → `contacts_find()` for name match
   - 0 matches → error with suggestion to provide address
   - 1 match → use that address
   - 2+ matches → return disambiguation list

**Read-only enforcement**: If all accounts are read-only, returns error (rc=2).

**Output**: Draft preview with `draft_id`. LLM reads back to user for confirmation.

### `confirm_send` — Actually Send

**Input**: `{draft_id: "d7f3a1bc2e09ab"}`

- Validates user_id match, checks throttle, sends via first writable account
- Return codes: 0=sent, 2=not found/expired, 3=throttled

### `folders` — List Available Folders/Labels

**Input**: `{account: "Gmail"}` (optional)

Lists IMAP folders or Gmail labels for an account. Useful for browsing non-inbox folders.

### `trash` — Prepare Trash (Does NOT Delete)

**Input**: `{message_id: "abc123", account: "Gmail"}` (account optional)

Creates a pending trash action. Fetches message metadata (subject, from) for confirmation display. Returns a `pending_id` for the confirmation step.

- Read-only enforcement: returns error (rc=2) if target account is read-only
- Pending actions expire after 5 minutes
- Up to 10 pending trash actions simultaneously

### `confirm_trash` — Execute Trash

**Input**: `{pending_id: "a1b2c3d4e5f6g7"}`

Confirms and executes a pending trash action. Routes to Gmail API (`POST /messages/{id}/trash`) for OAuth Gmail accounts, or IMAP (COPY to Trash + STORE \Deleted + EXPUNGE) for other providers.

- Return codes: 0=trashed, 2=not found/expired, 3=throttled
- Shares throttle mechanism with `confirm_send`

### `archive` — Archive Message (Immediate)

**Input**: `{message_id: "abc123", account: "Gmail"}` (account optional)

Archives a message immediately (no confirmation needed — non-destructive). Routes to Gmail API (`POST /messages/{id}/modify` with `removeLabelIds: ["INBOX"]`) for OAuth Gmail accounts, or IMAP (COPY to Archive/All Mail + delete from source) for other providers.

- Return codes: 0=archived, 1=failure, 2=read-only account

---

## Configuration

### dawn.toml

```toml
[email]
enabled = true    # Required (TOOL_CAP_DANGEROUS)
```

Account configuration is stored in the database (not toml), managed through the WebUI. The toml section only controls the tool's enabled state.

### Provider Setup Examples

Accounts are configured through the WebUI with provider presets (Gmail, iCloud, Outlook, Fastmail) that auto-fill server/port/SSL fields.

**Gmail (App Password)**: Generate at Google Account → Security → 2-Step Verification → App Passwords

**Gmail (Google OAuth)**: Follow `docs/GOOGLE_OAUTH_SETUP.md` Section 7. Add Gmail API scope `https://mail.google.com/` to the Google Cloud consent screen.

**iCloud**: Generate app password at appleid.apple.com → Sign-In & Security → App-Specific Passwords

**Proton Mail**: Requires Proton Mail Bridge running locally (loopback, SSL disabled is allowed).

---

## WebUI Account Management

### WebSocket Messages

| Message                          | Direction | Payload                                                                                                                                                 |
| -------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `email_list_accounts`            | C→S       | —                                                                                                                                                       |
| `email_list_accounts_response`   | S→C       | `{accounts: [...]}`                                                                                                                                     |
| `email_add_account`              | C→S       | `{name, imap_server, imap_port, imap_ssl, smtp_server, smtp_port, smtp_ssl, username, display_name, password, auth_type, oauth_account_key, read_only}` |
| `email_add_account_response`     | S→C       | `{success, error}`                                                                                                                                      |
| `email_update_account`           | C→S       | `{id, name, ...}`                                                                                                                                       |
| `email_update_account_response`  | S→C       | `{success, error}`                                                                                                                                      |
| `email_remove_account`           | C→S       | `{id}`                                                                                                                                                  |
| `email_remove_account_response`  | S→C       | `{success}`                                                                                                                                             |
| `email_test_connection`          | C→S       | `{id}`                                                                                                                                                  |
| `email_test_connection_response` | S→C       | `{success, imap_ok, smtp_ok, error}`                                                                                                                    |
| `email_set_read_only`            | C→S       | `{id, read_only}`                                                                                                                                       |
| `email_set_read_only_response`   | S→C       | `{success}`                                                                                                                                             |
| `email_set_enabled`              | C→S       | `{id, enabled}`                                                                                                                                         |
| `email_set_enabled_response`     | S→C       | `{success}`                                                                                                                                             |

### Account Card UI

Each account card shows: name, username, server info (or "Google OAuth" badge), and toggle switches for enabled/read-only. Actions: Test, Edit, Remove.

### Google OAuth Multi-Account

The "Connect with Google" button always launches the OAuth popup, even if an existing account already has the same scopes. This lets users select a different Google account via Google's own account picker in the consent screen. On success, the button resets to "Re-authorize" (not stuck on "Connecting...").

### Add Account Form

- **Auth type selector**: App Password or Google OAuth tabs
- **Provider presets**: Gmail, iCloud, Outlook, Fastmail, Other — auto-fills server/port/SSL
- **Form layout**: Account name, IMAP settings (server + port row, SSL toggle), SMTP settings (same), Credentials (username, display name, password with show/hide), Read-only toggle
- **Modal**: `max-width: 480px`, Escape key dismissal, focus trap, ARIA `role="dialog"` + `aria-modal="true"`
- **Password security**: Rejected over non-TLS WebSocket connections

### Edit Account Form

Same as add form, minus auth type selector. Password field shows "leave blank to keep current". Advanced settings section (collapsible) with `max_recent` (1-50) and `max_body_chars` (500-16000).

---

## File Manifest

### New Files (17)

| File                                    | Purpose                                   | Lines |
| --------------------------------------- | ----------------------------------------- | ----- |
| `include/memory/contacts_db.h`          | Contacts CRUD API                         | ~60   |
| `src/memory/contacts_db.c`              | SQLite operations with LIKE escape        | ~300  |
| `include/tools/email_types.h`           | Shared email types (summary, message, search params) | ~80   |
| `include/tools/email_db.h`             | Email account struct and CRUD             | ~80   |
| `src/tools/email_db.c`                  | Email account DB operations               | ~280  |
| `include/tools/email_client.h`          | IMAP/SMTP protocol API                    | ~130  |
| `src/tools/email_client.c`              | libcurl IMAP/SMTP operations              | ~1000 |
| `include/tools/gmail_client.h`          | Gmail REST API declarations               | ~80   |
| `src/tools/gmail_client.c`              | Gmail API (fetch, search, read, send, trash, archive) | ~1400 |
| `include/tools/email_service.h`         | Service layer + draft/pending trash types | ~195  |
| `src/tools/email_service.c`             | Multi-account routing, drafts, trash, auth | ~800  |
| `include/tools/email_tool.h`            | Tool registration header                  | ~30   |
| `src/tools/email_tool.c`                | LLM tool interface, 10 actions            | ~700  |
| `include/webui/webui_email.h`           | WebSocket handler declarations            | ~40   |
| `src/webui/webui_email.c`               | WebSocket handlers for account CRUD       | ~490  |
| `www/js/ui/email-accounts.js`           | Account management UI                     | ~810  |
| `www/css/components/email-accounts.css` | Email account styles                      | ~185  |

### Modified Files (14)

| File                              | Change                                                         |
| --------------------------------- | -------------------------------------------------------------- |
| `src/auth/auth_db_core.c`         | v26 migration (contacts + email_accounts), prepared statements |
| `include/auth/auth_db_internal.h` | Schema version 26, new stmt pointers                           |
| `src/memory/memory_callback.c`    | 4 contact actions (save/find/list/delete)                      |
| `src/tools/memory_tool.c`         | Contact actions in enum + description                          |
| `src/tools/calendar_db.c`         | Fix LIKE metacharacter escaping (drive-by)                     |
| `include/tools/calendar_db.h`     | `calendar_db_account_set_enabled()` declaration                |
| `src/tools/calendar_db.c`         | `calendar_db_account_set_enabled()` implementation             |
| `cmake/DawnTools.cmake`           | `DAWN_ENABLE_EMAIL_TOOL` option (OFF default)                  |
| `src/tools/tools_init.c`          | Email tool registration                                        |
| `src/webui/webui_server.c`        | Email + calendar WS message dispatch                           |
| `www/index.html`                  | Email accounts section + includes                              |
| `www/js/dawn.js`                  | Email + calendar response handlers                             |
| `include/tools/oauth_client.h`    | `GOOGLE_EMAIL_SCOPE` define                                    |
| `docs/GOOGLE_OAUTH_SETUP.md`      | Gmail scope + Section 7 email setup                            |

---

## Security Summary

| Measure            | Detail                                                                 |
| ------------------ | ---------------------------------------------------------------------- |
| Compile-time gate  | `DAWN_ENABLE_EMAIL_TOOL=OFF` in CMake                                  |
| Runtime gate       | `TOOL_CAP_DANGEROUS` requires `enabled = true` in `[email]` config     |
| Two-step send      | draft → read back → confirm_send                                       |
| Two-step trash     | pending_trash → read back → confirm_trash                              |
| Draft IDs          | `randombytes_buf()` from libsodium (7 bytes, hex-encoded)              |
| Draft binding      | User-ID bound (not session-bound)                                      |
| Draft throttling   | 3 failed confirms → 60-second lockout (mutex-protected)                |
| Draft expiry       | 5-minute TTL                                                           |
| Field validation   | Body > 4000 / subject > 250 chars rejected (no silent truncation)      |
| IMAP injection     | Literal syntax `{N}\r\n<data>` for user values in SEARCH               |
| IMAP dates         | Parsed via strptime → reconstructed via strftime                       |
| SMTP injection     | CR/LF stripped from all header fields                                  |
| TLS enforcement    | Plaintext refused for non-loopback servers                             |
| TLS verification   | Explicit VERIFYPEER + VERIFYHOST on all curl handles                   |
| Credential storage | `crypto_store_encrypt/decrypt` (libsodium crypto_secretbox)            |
| Credential wiping  | `sodium_memzero()` on all code paths + stack account arrays            |
| OAuth              | PKCE S256, encrypted token storage, separate email/calendar token sets |
| Password over WS   | Rejected if WebSocket connection is not TLS                            |
| Read-only          | Per-account flag blocks send/confirm_send/trash/confirm_trash           |
| No attachments     | Counted but not downloaded (avoids arbitrary file execution)           |
| Account ownership  | `verify_account_owner()` on all WebUI operations                       |

---

## Future Considerations

- **Reply/forward** — requires threading (In-Reply-To, References headers), quoted body formatting
- **Attachments** — download, view (for text/PDF), send with attachments
- **Mark read/unread** — IMAP STORE FLAGS command
- **IMAP IDLE / polling** — persistent IMAP connection for real-time "you have new mail" notifications
- **MIME multipart parsing** — proper text/plain extraction from multipart messages (currently extracts body after blank line)
- **Auto-contact extraction** — scan email headers to populate contacts database
- **Microsoft OAuth** — outlook.office365.com OAuth 2.0 for Outlook/Office 365 accounts

### Implemented Since v1

- ~~**Folders/labels**~~ — `folders` action lists available IMAP folders/Gmail labels (2026-03-15)
- ~~**Delete**~~ — `trash`/`confirm_trash` two-step delete via IMAP or Gmail API (2026-03-15)
- ~~**Archive**~~ — `archive` single-step action via IMAP or Gmail API (2026-03-15)
- ~~**Pagination**~~ — `page_token`/`next_page_token` for large result sets (2026-03-15)
- ~~**Gmail REST API backend**~~ — `gmail_client.c` for OAuth Gmail accounts (faster than IMAP) (2026-03-14)
- ~~**Multi-account**~~ — DB-stored accounts with WebUI management, per-account read-only (2026-03-13)
