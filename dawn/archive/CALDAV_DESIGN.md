# CalDAV Calendar Integration — Design Document

**Created**: March 9, 2026
**Status**: Implemented (2026-03-12), read_only flag added (2026-03-12), Google OAuth 2.0 added (2026-03-13), calendar name filtering added (2026-03-15)
**Dependencies**: libical-dev (apt), libcurl (already linked), libxml2 (already linked), libsodium (already linked)

---

## Overview

Add CalDAV calendar integration to DAWN. Users configure one or more CalDAV accounts (self-hosted Radicale, Google Calendar, iCloud, etc.) and interact via voice commands: "What's on my calendar today?", "Add dentist appointment tomorrow at 2pm", "When is my next meeting?", "Cancel my 3pm."

CalDAV is an open standard (RFC 4791) that works with any compliant server. This aligns with DAWN's privacy-first philosophy — users can run a local CalDAV server for fully offline calendar management, or connect to cloud providers without vendor lock-in.

### Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Date/time input | ISO 8601 from LLM | Same pattern as scheduler tool; LLM already converts natural language |
| Protocol library | libical (apt) + libcurl + libxml2 | All available; libical handles RRULE, timezones, edge cases |
| Caching | SQLite local cache | Fast queries, offline capability, reduces server round-trips |
| Account model | Multiple accounts, multiple calendars per account | Matches real use (Work + Personal); user selects active calendars |
| Event scope | Full CRUD + query | Production-ready from day one |

---

## Use Cases

### Voice Commands (via LLM tool)

| Category | Example | Tool Action |
|----------|---------|-------------|
| **Query today** | "What's on my calendar today?" | `today` |
| **Query range** | "Do I have anything this weekend?" | `range` |
| **Next event** | "When is my next meeting?" | `next` |
| **Search** | "Find meetings with Bob" | `search` |
| **Add event** | "Add dentist appointment tomorrow at 2pm" | `add` |
| **Add with duration** | "Schedule a 90 minute team meeting Friday at 10am" | `add` |
| **Add all-day** | "Add vacation day next Monday" | `add` (all-day flag) |
| **Delete** | "Cancel my 3pm appointment" | `delete` |
| **Reschedule** | "Move my dentist appointment to Thursday" | `update` |
| **Upcoming** | "What do I have coming up this week?" | `range` |
| **Free/busy** | "Am I free tomorrow afternoon?" | `range` (LLM interprets gaps) |
| **Recurring** | "Add standup meeting every weekday at 9am" | `add` (with recurrence) |

### Non-Voice (WebUI)

- View configured accounts and calendars in settings panel
- Add/remove CalDAV accounts (password encrypted at rest)
- Test connection to CalDAV server (triggers discovery + initial sync)
- Select which calendars are active (synced/searchable) via per-calendar toggles
- Toggle per-account "Read only" flag (prevents LLM from creating/modifying/deleting events)
- Manual sync trigger
- Visual indicators: sync status dots, read-only amber border, calendar color dots

---

## Architecture

### Component Layout

```
┌──────────────────────────────────────────┐
│           calendar_tool.c                │  ← LLM tool (actions, formatting)
│  register, callback, result formatting   │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│           calendar_service.c             │  ← Business logic
│  query, add, update, delete, sync        │
│  multi-account, calendar selection       │
└──────────┬──────────────┬────────────────┘
           │              │
┌──────────▼──────┐ ┌─────▼────────────────┐
│ calendar_db.c   │ │ caldav_client.c       │  ← Network + storage
│ SQLite cache    │ │ CalDAV HTTP protocol  │
│ CRUD, queries   │ │ libcurl + libxml2     │
└─────────────────┘ │ + libical parsing     │
                    └──────────────────────┘
```

### Layer Responsibilities

| Layer | File | Responsibility |
|-------|------|---------------|
| **Tool** | `calendar_tool.c` | Tool registration, parameter parsing, result formatting for LLM |
| **Service** | `calendar_service.c` | Multi-account routing, cache-vs-server decisions, business rules |
| **Cache** | `calendar_db.c` | SQLite CRUD for events, accounts, calendars |
| **Protocol** | `caldav_client.c` | CalDAV HTTP operations (PROPFIND, REPORT, PUT, DELETE) via libcurl + libxml2 + libical |

---

## Database Schema

All tables in the existing `auth.db` database.

```sql
-- CalDAV accounts (one per server connection)
CREATE TABLE calendar_accounts (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    name TEXT NOT NULL,                    -- Display name ("Work", "Personal")
    caldav_url TEXT NOT NULL,              -- Base URL (e.g., https://caldav.icloud.com/)
    username TEXT NOT NULL,
    encrypted_password BLOB NOT NULL,     -- Encrypted via libsodium secretbox (nonce + ciphertext)
    encrypted_password_len INTEGER,       -- Length of encrypted_password data
    auth_type TEXT DEFAULT 'basic',        -- 'basic', 'app_password', or 'oauth'
    oauth_account_key TEXT DEFAULT '',     -- Links to oauth_tokens when auth_type='oauth'
    principal_url TEXT,                    -- Cached from RFC 5397 discovery step 1
    calendar_home_url TEXT,               -- Cached from RFC 4791 discovery step 2
    enabled INTEGER DEFAULT 1,
    read_only INTEGER DEFAULT 0,           -- 1 = prevent LLM from modifying events
    last_sync INTEGER DEFAULT 0,           -- Unix timestamp of last successful sync
    sync_interval_sec INTEGER DEFAULT 900, -- How often to re-sync (default 15 min)
    created_at INTEGER NOT NULL,
    FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Calendars discovered on each account
CREATE TABLE calendar_calendars (
    id INTEGER PRIMARY KEY,
    account_id INTEGER NOT NULL,
    caldav_path TEXT NOT NULL,             -- Full URL path to calendar collection
    display_name TEXT NOT NULL,            -- "Work Calendar", "Birthdays", etc.
    color TEXT,                            -- Hex color from server (#FF5733)
    is_active INTEGER DEFAULT 1,          -- User toggle: include in queries?
    ctag TEXT,                             -- Calendar collection tag (change detection)
    created_at INTEGER NOT NULL,
    FOREIGN KEY(account_id) REFERENCES calendar_accounts(id) ON DELETE CASCADE
);

-- Master events (one row per calendar event or recurring series)
CREATE TABLE calendar_events (
    id INTEGER PRIMARY KEY,
    calendar_id INTEGER NOT NULL,
    uid TEXT NOT NULL,                     -- iCalendar UID (globally unique)
    etag TEXT,                             -- Server ETag (change detection)
    summary TEXT NOT NULL,                 -- Event title
    description TEXT,                      -- Event description/notes
    location TEXT,                         -- Event location
    dtstart INTEGER NOT NULL,              -- Start time (Unix timestamp, for indexing)
    dtend INTEGER,                         -- End time (Unix timestamp)
    duration INTEGER,                      -- Duration in seconds (for recurring events)
    all_day INTEGER DEFAULT 0,             -- 1 = all-day event
    dtstart_date TEXT,                     -- All-day: "2026-03-10" (NULL for timed events)
    dtend_date TEXT,                       -- All-day: "2026-03-11" exclusive (NULL for timed)
    rrule TEXT,                            -- Recurrence rule (iCalendar RRULE string)
    raw_ical TEXT NOT NULL,                -- Original iCalendar data (for PUT back)
    last_synced INTEGER NOT NULL,          -- When this event was last fetched
    FOREIGN KEY(calendar_id) REFERENCES calendar_calendars(id) ON DELETE CASCADE,
    UNIQUE(calendar_id, uid)
);

-- Pre-expanded occurrences (populated during sync, queried at runtime)
-- Non-recurring events also get a single row here for uniform queries.
CREATE TABLE calendar_occurrences (
    id INTEGER PRIMARY KEY,
    event_id INTEGER NOT NULL,             -- FK to calendar_events master
    dtstart INTEGER NOT NULL,              -- Occurrence start (Unix timestamp)
    dtend INTEGER NOT NULL,                -- Occurrence end (Unix timestamp)
    all_day INTEGER DEFAULT 0,             -- 1 = all-day occurrence
    dtstart_date TEXT,                     -- All-day: "2026-03-10"
    dtend_date TEXT,                       -- All-day: "2026-03-11" exclusive
    summary TEXT,                          -- Copied from master, or overridden
    location TEXT,                         -- Copied from master, or overridden
    is_override INTEGER DEFAULT 0,         -- 1 if from RECURRENCE-ID component
    is_cancelled INTEGER DEFAULT 0,        -- 1 if this occurrence was cancelled
    recurrence_id TEXT,                    -- Original DTSTART if override
    FOREIGN KEY(event_id) REFERENCES calendar_events(id) ON DELETE CASCADE,
    UNIQUE(event_id, dtstart)
);

CREATE INDEX idx_cal_occurrences_range ON calendar_occurrences(dtstart, dtend)
    WHERE is_cancelled = 0;
CREATE INDEX idx_cal_events_uid ON calendar_events(uid);
CREATE INDEX idx_cal_events_cal ON calendar_events(calendar_id);
CREATE INDEX idx_cal_accounts_user ON calendar_accounts(user_id);
```

### Why Cache `raw_ical`

CalDAV requires sending back the full iCalendar object when updating events (PUT). Storing the original avoids reconstructing it from parsed fields — preserving any vendor-specific properties (Google, iCloud, etc.) that libical parsed but we don't model.

### Why Pre-Expand Occurrences

Recurring events are stored as a single master row in `calendar_events` with an RRULE. During sync, libical's `icalrecur_iterator` expands occurrences into `calendar_occurrences` rows within the cache window. This means:

- **Voice queries** ("what's on my calendar today?") are a trivial indexed range scan — no libical in the query path
- **EXDATE** (exclusions like "daily standup except Dec 25") are applied during sync by skipping or deleting rows
- **RECURRENCE-ID** (modified occurrences like "Tuesday's meeting moved to Wednesday") are upserted with `is_override = 1`
- **Non-recurring events** also get a row in `calendar_occurrences` so the query path is always uniform
- **Storage cost** is negligible: 10 recurring events × 200 occurrences × ~120 bytes = ~240 KB

### Why Separate All-Day Date Columns

All-day events in iCalendar use `VALUE=DATE` (e.g., `DTSTART;VALUE=DATE:20260310`). Converting date-only values to Unix timestamps introduces timezone-dependent off-by-one errors — a March 10 event stored as midnight UTC appears on March 9 for users in US timezones. Storing as date strings (`dtstart_date`/`dtend_date`) eliminates this ambiguity. The `dtstart` INTEGER column is still populated (midnight UTC) for index ordering.

---

## CalDAV Protocol Layer (`caldav_client.c`)

### Operations

All operations use libcurl for HTTP and libxml2 for XML request/response parsing.

#### 1. Discover Calendars (3-Step RFC-Compliant PROPFIND)

CalDAV discovery follows RFC 4791 + RFC 5397. This is a 3-step process, not a single PROPFIND. iCloud, Google, Fastmail, and any compliant server require all three steps. Simple servers (Radicale) happen to work with just step 3, so the implementation adaptively short-circuits when possible.

**Step 1: Find current-user-principal** (RFC 5397)

```http
PROPFIND /caldav/v2/user@example.com/ HTTP/1.1
Depth: 0
Content-Type: application/xml

<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:" xmlns:c="urn:ietf:params:xml:ns:caldav">
  <d:prop>
    <d:current-user-principal/>
    <d:resourcetype/>
  </d:prop>
</d:propfind>
```

Returns the principal URL (e.g., `/principals/users/foo/`). If this fails or the base URL already contains calendar collections, fall back to treating the base URL as calendar-home directly.

**Step 2: Find calendar-home-set** (RFC 4791 §6.2.1)

```http
PROPFIND /principals/users/foo/ HTTP/1.1
Depth: 0
Content-Type: application/xml

<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:" xmlns:c="urn:ietf:params:xml:ns:caldav">
  <d:prop>
    <c:calendar-home-set/>
  </d:prop>
</d:propfind>
```

Returns the calendar-home-set URL (e.g., `/calendars/user@example.com/`).

**Step 3: Enumerate calendar collections** (PROPFIND Depth:1)

```http
PROPFIND /calendars/user@example.com/ HTTP/1.1
Depth: 1
Content-Type: application/xml

<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:" xmlns:c="urn:ietf:params:xml:ns:caldav"
            xmlns:cs="http://calendarserver.org/ns/"
            xmlns:a="http://apple.com/ns/ical/">
  <d:prop>
    <d:resourcetype/>
    <d:displayname/>
    <cs:getctag/>
    <d:current-user-privilege-set/>
    <a:calendar-color/>
  </d:prop>
</d:propfind>
```

Parses response to find calendar collections (resources with `<cal:calendar/>` resourcetype). The `apple-color` property provides a fallback for iCloud's non-standard color field.

**URL Resolution**: Servers return relative, absolute-path, or full URLs in href responses. A `resolve_href()` helper handles all three cases — this is the #1 source of bugs in CalDAV client implementations.

**Caching**: Resolved `principal_url` and `calendar_home_url` are stored in `calendar_accounts` so discovery only runs once (on account creation and on explicit re-discovery via WebUI "Test Connection").

#### 2. Fetch Events in Range (REPORT)

```http
REPORT /caldav/v2/user@example.com/personal/ HTTP/1.1
Content-Type: application/xml
Depth: 1

<?xml version="1.0"?>
<c:calendar-query xmlns:d="DAV:" xmlns:c="urn:ietf:params:xml:ns:caldav">
  <d:prop>
    <d:getetag/>
    <c:calendar-data/>
  </d:prop>
  <c:filter>
    <c:comp-filter name="VCALENDAR">
      <c:comp-filter name="VEVENT">
        <c:time-range start="20260309T000000Z" end="20260310T000000Z"/>
      </c:comp-filter>
    </c:comp-filter>
  </c:filter>
</c:calendar-query>
```

Response contains iCalendar data for each event, parsed by libical.

#### 3. Create Event (PUT)

```http
PUT /caldav/v2/user@example.com/personal/event-uuid.ics HTTP/1.1
Content-Type: text/calendar
If-None-Match: *

BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//DAWN//CalDAV Client//EN
BEGIN:VEVENT
UID:event-uuid@dawn.local
DTSTAMP:20260309T120000Z
DTSTART:20260310T140000Z
DTEND:20260310T150000Z
SUMMARY:Dentist Appointment
END:VEVENT
END:VCALENDAR
```

`If-None-Match: *` prevents overwriting an existing event with the same UID.

#### 4. Update Event (PUT with ETag)

```http
PUT /caldav/v2/user@example.com/personal/event-uuid.ics HTTP/1.1
Content-Type: text/calendar
If-Match: "etag-from-server"
```

Uses the stored `raw_ical` with modifications applied via libical, and the cached ETag for optimistic concurrency.

#### 5. Delete Event (DELETE)

```http
DELETE /caldav/v2/user@example.com/personal/event-uuid.ics HTTP/1.1
If-Match: "etag-from-server"
```

### Authentication

| Provider | Auth Type | How |
|----------|----------|-----|
| Self-hosted (Radicale, Baikal) | HTTP Basic | Username + password |
| Google Calendar | OAuth 2.0 | Browser consent flow via shared `oauth_client.c` module (PKCE S256) |
| iCloud | App-specific password | Apple ID + app-specific password |
| Fastmail, Zoho, etc. | HTTP Basic or app password | Provider-dependent |

**App-password accounts**: Passwords are encrypted at rest using libsodium's `crypto_secretbox` (XSalsa20-Poly1305) via the shared `crypto_store` module. Each password is stored as `nonce + ciphertext` in the `encrypted_password` BLOB column. The encryption key is stored in `<data_dir>/dawn.key`.

**OAuth accounts**: Access and refresh tokens are stored in the `oauth_tokens` table, encrypted via the same `crypto_store` module. Token refresh is automatic — `oauth_get_access_token()` checks expiry (300s margin) and refreshes transparently. CalDAV requests use `Authorization: Bearer` via `CURLOPT_XOAUTH2_BEARER`. See `docs/GOOGLE_OAUTH_SETUP.md` for Google Cloud Console setup.

### Error Handling

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| 200-207 | Success | Parse response |
| 401 | Bad credentials | Log error, mark account as auth-failed, tell LLM |
| 403 | Forbidden | May be rate-limited or missing scope |
| 404 | Calendar/event not found | Remove from cache |
| 412 | Precondition failed (ETag mismatch) | Re-sync, retry |
| 5xx | Server error | Retry once, then report failure |

---

## Cache Strategy

### Sync Flow

```
1. On startup: sync all enabled accounts (background thread)
2. Every sync_interval_sec (default 15 min): check ctag for changes
3. If ctag unchanged: skip (no server changes)
4. If ctag changed: fetch updated events (REPORT with time range)
5. On tool query: serve from cache (instant)
6. On tool add/update/delete: write to server first, update cache on success
```

### CTag-Based Change Detection

CalDAV servers expose a `ctag` (collection tag) that changes whenever any event in a calendar is modified. By comparing the stored ctag with the server's current ctag, we avoid re-fetching unchanged calendars.

```
Sync check:
  PROPFIND calendar → get current ctag
  Compare with stored ctag
  If different → REPORT to fetch changed events
  Update stored ctag
```

### Cache Window

Sync fetches events from `now - 30 days` to `now + 365 days`. Configurable via:

```toml
[calendar]
cache_past_days = 30
cache_future_days = 365
```

Events outside this window are not cached but can be fetched on-demand by the `search` action.

### Write-Through

When the tool creates, updates, or deletes an event:
1. Send the request to the CalDAV server
2. If server returns success → update local cache
3. If server returns error → report failure to LLM, cache unchanged
4. Never modify cache without server confirmation

---

## Timezone Handling

### Approach

Use **libical's `icaltimezone`** for all timezone conversions — it is thread-safe and does not mutate the process `TZ` environment variable (which the scheduler tool depends on being stable).

### Input from LLM

- Bare ISO 8601 (e.g., `2026-03-10T14:00:00`) → interpret in user's configured timezone via `icaltimezone_get_builtin_timezone(user_tz)`
- Explicit offset (e.g., `2026-03-10T14:00:00-05:00` or `...Z`) → use as-is

### Display Output

Convert epoch to user-local time via `icaltime_from_timet_with_zone()`, then format with `strftime`. Example: `"9:00 AM"`, `"Thu Mar 12"`.

### All-Day Events

All-day events use date strings, not epoch conversion:
- Input: `"all_day": true, "start": "2026-03-10"`
- Storage: `dtstart_date = "2026-03-10"`, `dtend_date = "2026-03-11"` (exclusive)
- iCalendar: `DTSTART;VALUE=DATE:20260310`, `DTEND;VALUE=DATE:20260311`
- Display: `"All Day | Company Holiday (Work)"`

### Recurring Event DST

RRULE expansion uses the original DTSTART with TZID from `raw_ical`. libical correctly handles DST transitions — a "daily at 9:00 AM" recurring event stays at 9:00 AM local time even when clocks change.

### Mutation Safety

Calendar mutations (add/update/delete) do **not** use `TOOL_CAP_DANGEROUS`. Unlike email send (irreversible, externally visible), calendar operations are reversible and affect only the user's own data. Confirmation before delete/update is handled by the LLM system prompt: "Before deleting or modifying calendar events, confirm the event details with the user first."

---

## Tool Registration

```c
static const treg_param_t calendar_params[] = {
   {
       .name = "action",
       .description = "Calendar action: 'today' (today's events), 'range' (events in date range), "
                      "'next' (next upcoming event), 'search' (find events by text), "
                      "'add' (create event), 'update' (reschedule/modify), 'delete' (cancel event)",
       .type = TOOL_PARAM_TYPE_ENUM,
       .required = true,
       .maps_to = TOOL_MAPS_TO_ACTION,
       .enum_values = { "today", "range", "next", "search", "add", "update", "delete" },
       .enum_count = 7,
   },
   {
       .name = "details",
       .description =
           "JSON object with action-specific fields. "
           "For 'today': {} (no fields needed). "
           "For 'range': {start (ISO 8601), end (ISO 8601)}. "
           "For 'next': {} or {calendar (name filter)}. "
           "For 'search': {query (text to match in summary/description/location)}. "
           "For 'add': {summary (required), start (ISO 8601, required), "
           "end (ISO 8601, optional — defaults to start + 1 hour), "
           "location (optional), description (optional), all_day (bool, optional), "
           "calendar (name, optional — uses default if omitted), "
           "recurrence (none|daily|weekly|monthly|yearly, optional), "
           "recurrence_until (ISO 8601, optional end date for recurrence)}. "
           "For 'update': {uid (required, from previous query), summary, start, end, "
           "location, description — only include fields to change}. "
           "For 'delete': {uid (required, from previous query)}.",
       .type = TOOL_PARAM_TYPE_STRING,
       .required = false,
       .maps_to = TOOL_MAPS_TO_VALUE,
   },
};

static const tool_metadata_t calendar_metadata = {
   .name = "calendar",
   .device_string = "calendar",
   .description = "Access the user's calendar. Query today's schedule, upcoming events, "
                  "search for events, add new events, reschedule, or cancel. "
                  "Use ISO 8601 format for all dates and times. "
                  "The user may have multiple calendars (Work, Personal, etc.) — "
                  "include the calendar name when adding events if the user specifies one. "
                  "When listing events, include the calendar name for context. "
                  "Event UIDs are returned in query results — use them for update/delete.",
   .params = calendar_params,
   .param_count = 2,
   .device_type = TOOL_DEVICE_TYPE_GETTER,
   .capabilities = TOOL_CAP_NETWORK | TOOL_CAP_SECRETS | TOOL_CAP_SCHEDULABLE,
   .is_getter = false,  /* Can modify (add/update/delete) */
   .config = &s_cal_config,
   .config_size = sizeof(s_cal_config),
   .config_parser = calendar_parse_config,
   .config_writer = calendar_write_config,
   .config_section = "calendar",
   .secret_requirements = calendar_secrets,
   .is_available = calendar_is_available,
   .init = calendar_init,
   .cleanup = calendar_cleanup,
   .callback = calendar_callback,
};
```

### Availability

Returns true when at least one account is configured, enabled, and has successfully synced.

---

## Result Formatting

### Today / Range / Next

```
CALENDAR: Today, Monday March 9, 2026

[1] 9:00 AM - 10:00 AM | Team Standup (Work)
    Location: Conference Room B

[2] 12:00 PM - 1:00 PM | Lunch with Alice (Personal)
    Location: The Grill on Main St

[3] 3:00 PM - 3:30 PM | Dentist Appointment (Personal)
    UID: event-abc123@dawn.local

3 events today.
```

### Search

```
CALENDAR SEARCH: "Bob" (4 results)

[1] Thu Mar 12, 2:00 PM - 3:00 PM | 1:1 with Bob (Work)
    UID: event-def456@dawn.local

[2] Mon Mar 16, 10:00 AM - 11:00 AM | Project Review with Bob & Carol (Work)
    UID: event-ghi789@dawn.local

[3] Fri Mar 20, 12:00 PM - 1:00 PM | Lunch with Bob (Personal)
    UID: event-jkl012@dawn.local
```

UIDs are included so the LLM can reference them for update/delete actions in follow-up tool calls.

### Add / Update / Delete Confirmation

```
EVENT CREATED: Dentist Appointment
  When: Tue Mar 10, 2:00 PM - 3:00 PM
  Calendar: Personal
  UID: event-mno345@dawn.local
```

---

## Configuration

### dawn.toml

```toml
[calendar]
enabled = true
sync_interval_sec = 900          # 15 minutes
cache_past_days = 30
cache_future_days = 365
default_event_duration_min = 60  # When end time is not specified

# Account configuration is done via WebUI admin panel.
# Passwords reference keys in secrets.toml.
```

Account and calendar configuration is stored in the database (not TOML) because:
- Multiple accounts with varying credentials
- Calendar discovery is dynamic (server returns available calendars)
- Active/inactive calendar selection is per-user state

### WebUI Settings Panel

| Section | Fields |
|---------|--------|
| **Accounts** | Add/remove accounts: name, CalDAV URL, username, password (encrypted in DB), read-only toggle |
| **Calendars** | Per-account: discovered calendars with active/inactive toggle, color dots |
| **Connection test** | "Test Connection" button: 3-step discovery → sync → show calendars or error |
| **Sync status** | Last sync time per account with status dot (green=synced, amber=never) |
| **Read-only** | Per-account toggle: "Read only" checkbox (checked=restricted). Amber left border on card. |

All account management uses WebSocket messages (`calendar_add_account`, `calendar_delete_account`, `calendar_test_connection`, `calendar_toggle_read_only`, etc.) — not REST endpoints. Passwords are encrypted via libsodium before DB storage; the WebUI never receives passwords back.

**Google OAuth multi-account**: The "Connect with Google" button always launches the OAuth popup, even if an existing account already has the same scopes. This lets users select a different Google account via Google's own account picker in the consent screen. On success, the button resets to show "Connected as [email]".

---

## Multi-Account / Multi-Calendar Routing

### Query Routing

When the tool queries events (today, range, next, search):
1. Get all active calendars for the current user (optionally filtered by calendar name)
2. Query the local cache across matching calendars
3. Results include the calendar display name for context
4. Optional `calendar` parameter on all query actions (today, range, next, search) — case-insensitive match on display name. If no match, returns an error listing available calendars.

### Add Routing

When adding an event:
1. If `calendar` field specified → find matching calendar by display name
2. If matched calendar belongs to a read-only account → return error (rc=2)
3. If not specified → use the first active **writable** calendar (skip read-only accounts)
4. If all calendars are read-only → return error
5. If ambiguous match → return error listing available calendars

### Update / Delete Routing

UID is globally unique across all calendars. Look up the event by UID in the cache to determine which calendar (and therefore which account/server) to send the request to. If the resolved account is read-only, the operation is blocked (rc=2) before any server request.

### Read-Only Enforcement

The `read_only` flag on `calendar_account_t` provides a code-level guardrail at the account level:

- **Service layer**: `calendar_service_add()`, `calendar_service_update()`, and `calendar_service_delete()` all check `read_only` and return 2 (distinct from 0=success, 1=general error).
- **Tool layer**: rc=2 produces a clear LLM-facing error: "The target calendar belongs to a read-only account."
- **Proactive LLM awareness**: Query results (today/range/next/search) append an access summary footer when any read-only accounts exist, listing writable vs read-only calendars. This lets the LLM make intelligent decisions without trial-and-error.
- **JOIN piggyback**: `account_read_only` is populated on `calendar_calendar_t` via the `active_for_user` JOIN query — no N+1 lookups needed.
- **Dedicated setter**: `calendar_db_account_set_read_only()` is separate from the general account update to prevent accidental flag clears.
- **Account-level granularity**: v1 design decision. Per-calendar granularity can be added later without conflict.

---

## Background Sync Thread

```c
static void *calendar_sync_thread(void *arg) {
    while (!s_shutdown) {
        for each enabled account:
            if (now - account.last_sync >= account.sync_interval_sec):
                for each active calendar in account:
                    // Check ctag
                    char *server_ctag = caldav_get_ctag(account, calendar);
                    if (strcmp(server_ctag, calendar.ctag) == 0):
                        continue;  // No changes

                    // Fetch changed events
                    caldav_fetch_events(account, calendar, range_start, range_end);
                    for each changed event (etag mismatch):
                        calendar_db_upsert_event(event);
                        // Delete old occurrences, re-expand
                        calendar_db_delete_occurrences(event.id);
                        if (event.rrule):
                            // libical RRULE expansion within sync window
                            expand_recurrence(event, sync_window);
                            // Apply EXDATEs (skip), RECURRENCE-IDs (override)
                        else:
                            // Single occurrence row for non-recurring
                            calendar_db_insert_occurrence(event);

                    calendar_db_update_ctag(calendar.id, server_ctag);
                account.last_sync = now;

        // Sleep in 1-second increments (responsive to shutdown)
        for (int i = 0; i < sync_interval && !s_shutdown; i++)
            sleep(1);
    }
}
```

### Sync on Startup

First sync runs immediately on `calendar_init()`. The background thread handles subsequent periodic syncs.

### Sync on Demand

After add/update/delete, the affected calendar is re-synced immediately (ctag will have changed).

---

## Implementation Status

All phases implemented as of 2026-03-12. Google OAuth 2.0 added 2026-03-13. Calendar name filtering added 2026-03-15 (optional `calendar` parameter on today/range/next/search actions for filtering by calendar display name). Key implementation notes:

- **Schema**: v23 (initial tables), v24 (read_only column migration), v25 (oauth_tokens table + oauth_account_key column)
- **Password storage**: Changed from secrets.toml references to encrypted-in-DB via shared `crypto_store` module (libsodium secretbox, `dawn.key`)
- **OAuth 2.0**: Google CalDAV via OAuth with PKCE S256. Shared `oauth_client.c` + `crypto_store.c` modules. WebUI popup consent flow. Google-specific discovery (no RFC 4791 PROPFIND — uses known URL patterns).
- **WebUI**: WebSocket-based (not REST) — consistent with all other DAWN admin panels. Auth type selector (App Password / Google OAuth) in add-account modal.
- **Read-only flag**: Added post-implementation after 4-agent review cycle
- **Access summary**: LLM query results include writable/read-only calendar footer when mixed access exists

---

## File Manifest

### New Files

| File | Purpose | Lines |
|------|---------|-------|
| `include/tools/caldav_client.h` | CalDAV protocol API | 186 |
| `src/tools/caldav_client.c` | CalDAV HTTP operations (3-step discovery, REPORT, PUT, DELETE, URL resolution) | 1035 |
| `include/tools/calendar_db.h` | Calendar cache DB API (types + CRUD declarations) | 194 |
| `src/tools/calendar_db.c` | SQLite CRUD for accounts, calendars, events, occurrences (Pattern A, shared s_db) | 717 |
| `include/tools/calendar_service.h` | Calendar service API (lifecycle, queries, mutations, access summary) | 172 |
| `src/tools/calendar_service.c` | Multi-account routing, sync thread, read-only enforcement, RRULE expansion | 1185 |
| `include/tools/calendar_tool.h` | Tool registration header | 30 |
| `src/tools/calendar_tool.c` | LLM tool: params, actions, result formatting, access summary footer | 557 |
| `include/webui/webui_calendar.h` | WebUI calendar handler declarations | 53 |
| `src/webui/webui_calendar.c` | WebUI WebSocket handlers (accounts, calendars, sync, test, read-only toggle) | 446 |
| `www/js/ui/calendar-accounts.js` | Calendar accounts settings panel (CRUD, toggles, modal) | 663 |
| `www/css/components/calendar-accounts.css` | Calendar accounts styles (cards, toggles, modal, responsive) | 392 |
| `include/tools/oauth_client.h` | OAuth 2.0 client API (auth URL, code exchange, token refresh, storage) | ~100 |
| `src/tools/oauth_client.c` | OAuth implementation (PKCE S256, Google provider, encrypted token DB) | ~600 |
| `include/core/crypto_store.h` | Shared libsodium encryption API (init, encrypt, decrypt) | ~30 |
| `src/core/crypto_store.c` | Shared encryption module (crypto_secretbox, dawn.key) | ~200 |
| `www/js/ui/oauth.js` | OAuth popup handler (blocker mitigation, origin validation) | ~180 |
| `www/js/oauth-callback.js` | OAuth callback page script (postMessage to opener) | ~20 |
| `www/css/components/oauth.css` | OAuth styles (auth type selector, connect button, status dots) | ~50 |
| `docs/GOOGLE_OAUTH_SETUP.md` | Google OAuth setup guide (Cloud Console, secrets.toml, WebUI) | ~125 |

**Total**: ~7,565 lines across 21 files.

### Modified Files

| File | Change |
|------|--------|
| `src/tools/tools_init.c` | Register calendar tool |
| `cmake/DawnTools.cmake` | Add `DAWN_ENABLE_CALENDAR_TOOL`, link libical/libxml2 |
| `CMakeLists.txt` | `pkg_check_modules(LIBICAL libical)` |
| `www/index.html` | Add calendar settings panel container, CSS/JS imports |
| `www/js/dawn.js` | Route calendar WebSocket response messages to handlers |
| `www/js/ui/settings/schema.js` | Calendar settings schema entries |
| `src/webui/webui_server.c` | Route calendar + OAuth WebSocket message types to handlers |
| `src/webui/webui_http.c` | `/oauth/callback` endpoint, `is_public_path` allowlist |
| `include/webui/webui_internal.h` | Calendar + OAuth handler declarations |
| `include/auth/auth_db_internal.h` | Calendar + OAuth prepared statements, schema v23-v25 |
| `src/auth/auth_db_core.c` | Schema migrations (v23-v25), prepared statements, finalize |
| `src/config/config_parser.c` | Parse `[secrets.google]` client_id, client_secret |
| `src/config/config_env.c` | `DAWN_GOOGLE_*` env overrides, Google fields in secrets write/status |
| `src/webui/webui_config.c` | Google OAuth fields in `handle_set_secrets` |
| `dawn.toml.example` | Add `[calendar]` section |
| `secrets.toml.example` | Add `[secrets.google]` section |

### New Dependency

```
apt install libical-dev   # libical 3.0.14 on Ubuntu 22.04 (Jammy)
```

---

## Testing

### Unit Tests

| Test | What |
|------|------|
| `test_calendar_db` | CRUD for accounts, calendars, events. User isolation. Cascade delete. |
| `test_ical_parsing` | Parse sample iCalendar data via libical. Recurring events. Timezones. All-day. |

### Integration Tests (per backend)

| Backend | Test |
|---------|------|
| **Radicale** (Docker) | Full CRUD cycle. Offline cache. Reconnect after server restart. |
| **Google Calendar** | Discovery (Google-specific), fetch, create, delete. OAuth 2.0 auth. |
| **iCloud** | Discovery, fetch, create, delete. App-specific password auth. |

### Voice Command Tests

| Command | Expected |
|---------|----------|
| "What's on my calendar today?" | Lists today's events with times and calendar names |
| "Add dentist appointment tomorrow at 2pm" | Creates event, confirms with details |
| "Schedule a 2 hour meeting with Bob on Friday at 10am" | Creates 10:00-12:00 event |
| "Cancel my dentist appointment" | Searches, confirms, deletes |
| "Move my 3pm to 4pm" | Searches for 3pm event, updates start/end |
| "Am I free tomorrow afternoon?" | Lists events in 12pm-6pm range, LLM interprets gaps |
| "What's on my work calendar this week?" | Filters by calendar name, shows range |
| "Add a recurring standup every weekday at 9am" | Creates with RRULE FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR |

---

## Provider-Specific Notes

### Google Calendar

- **CalDAV URL**: `https://apidata.googleusercontent.com/caldav/v2/{email}/events/` (per-user, auto-populated from OAuth email)
- **Auth**: OAuth 2.0 via shared `oauth_client.c` module. Browser consent flow with PKCE S256. See `docs/GOOGLE_OAUTH_SETUP.md`.
- **API**: Enable the **CalDAV API** (not "Google Calendar API") in Google Cloud Console.
- **Discovery**: Google does not support standard RFC 4791 PROPFIND discovery. DAWN uses known URL patterns: `/caldav/v2/{email}/events/` for the calendar collection and `/caldav/v2/{email}/user` for the principal. The `google_caldav_test()` function in `calendar_service.c` handles this.
- **Redirect URI**: Google requires a fully qualified domain name (FQDN) — bare IPs are rejected. Set up a DNS A record pointing to your server's LAN IP (e.g., `jetson.yourdomain.com → 192.168.1.159`).
- **Quirk**: Google may return `VTIMEZONE` components with non-standard timezone IDs. libical handles this.

### iCloud

- **CalDAV URL**: `https://caldav.icloud.com/`
- **Auth**: Apple ID + app-specific password (generate at appleid.apple.com → Sign-In & Security → App-Specific Passwords)
- **Note**: iCloud follows the full RFC 4791/5397 discovery flow (current-user-principal → calendar-home-set → collections). This is standard behavior, handled by the generic 3-step discovery — no special-casing needed.
- **Note**: iCloud returns calendar color as `apple-color` property instead of standard `calendar-color`. The discovery XML requests both and uses whichever is available.

### Radicale (Self-Hosted)

- **CalDAV URL**: `http://localhost:5232/username/`
- **Auth**: HTTP Basic (configured in Radicale)
- **Advantage**: Fully offline, no rate limits, fastest sync
- **Setup**: `pip install radicale && python -m radicale --storage-filesystem-folder=~/.radicale`

### Fastmail

- **CalDAV URL**: `https://caldav.fastmail.com/dav/calendars/user/you@fastmail.com/`
- **Auth**: App-specific password

---

## Future Considerations

- ~~**OAuth 2.0 authentication**~~ — **Done** (2026-03-13). Shared `oauth_client.c` module with PKCE S256, `crypto_store.c` for encrypted token storage, Google CalDAV working via Bearer auth. WebUI popup consent flow with blocker mitigation. See `docs/GOOGLE_OAUTH_SETUP.md`. Infrastructure ready for Microsoft 365, Outlook, and email (IMAP XOAUTH2).
- **Per-calendar read-only** — current granularity is account-level. A single CalDAV account may have both writable and read-only calendars on the server. Per-calendar `read_only` column on `calendar_calendars` table would allow finer control. No schema conflict with current design.
- **VTODO support** — task lists via CalDAV (same protocol, different component)
- **VFREEBUSY** — structured free/busy queries (most servers support this)
- **Push notifications** — CalDAV supports push via WebDAV sync, but polling with ctag is simpler and sufficient
- **CalDAV proxy** — act as a CalDAV server so other apps can sync with DAWN's cache
- **Calendar sharing** — multi-user households seeing each other's calendars (permission model needed)
- **Natural language recurrence** — "every other Tuesday" (currently: LLM must generate RRULE)
