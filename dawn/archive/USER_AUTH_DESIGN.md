# DAWN User Authentication System Design

**Status**: Phases 0-4 Complete (WebUI auth, multi-user, conversation history, session management)
**Date**: 2025-12-18
**Last Updated**: 2026-03-20

**Note**: DAP2 satellite authentication is now handled via pre-shared registration key in `secrets.toml`, validated in `handle_satellite_register()`. Satellite-to-user assignment is managed via the `satellite_mappings` DB table (schema v21) and the WebUI Satellite Management admin panel (`webui_admin_satellite.c`). Mapped satellites inherit their assigned user's music queue, memory extraction, and personalized prompt on connect. The `dawn-admin` CLI tool (v2.0.0) provides user, session, conversation, music DB, audit log, and IP management. DAP1 has been removed entirely.
**Note**: Phase 3 reviewed by architecture, efficiency, security, and UI agents (2026-01-04).

## Overview

This document describes a user account system for DAWN that provides:
- Multi-user support with individual preferences
- Role-based access control (Admin / User)
- Secure authentication for WebUI and DAP2 satellites
- Per-user settings (persona, locale, TTS preferences)

---

## Deployment Modes

DAWN supports multiple deployment configurations with different security requirements:

| Mode | Local Mic | DAP Clients | WebUI | Use Case |
|------|-----------|-------------|-------|----------|
| **1** | ✓ | ✗ | ✗ | Armor suit / embedded - no network interface |
| **2** | ✓ | ✓ | ✗ | ESP32 clients only, headless server |
| **3** | ✓ | ✗ | ✓ | Desktop/dev - web interface, no IoT |
| **4** | ✓ | ✓ | ✓ | Full deployment - all features |

### Mode Security Requirements

| Mode | Auth Required | Setup Method | Attack Surface |
|------|---------------|--------------|----------------|
| 1 | No | N/A | Physical only |
| 2 | DAP tokens | CLI tool (`dawn-admin`) | TCP (DAP port) |
| 3 | Full user auth | WebUI setup wizard | HTTP/WebSocket |
| 4 | Both | WebUI wizard + CLI tool | Both ports |

### Mode 1: Local Only

- **No authentication required** - local microphone is trusted (`LOCAL_SESSION_ID = 0`)
- Auth database not created (reduces attack surface)
- Smallest binary size (network code excluded at compile time)

### Mode 2: DAP Only (Headless)

- **Device token authentication required** for ESP32/DAP clients
- **CRITICAL**: Requires `dawn-admin` CLI tool for setup (no WebUI available)
- Admin account created via CLI before device tokens can be generated
- No web-based configuration available

### Mode 3: WebUI Only

- **Full user authentication** via WebUI
- Setup wizard with console-printed token (headless-friendly)
- No DAP/network audio support

### Mode 4: Full Deployment

- Combines WebUI user auth and DAP device tokens
- WebUI used for all management including device token creation
- CLI tool available as alternative/backup

---

## Build System

### Compile-Time Feature Toggles

```cmake
# CMakeLists.txt feature flags
option(ENABLE_DAP "Enable DAP server for ESP32 satellites" ON)
option(ENABLE_WEBUI "Enable WebUI server" ON)

# Auth is automatically enabled if any network feature is enabled
# Auth is skipped entirely for Mode 1 (local-only) builds
```

### Build Commands by Mode

| Mode | CMake Command |
|------|---------------|
| 1 (Local only) | `cmake -DENABLE_DAP=OFF -DENABLE_WEBUI=OFF ..` |
| 2 (DAP only) | `cmake -DENABLE_DAP=ON -DENABLE_WEBUI=OFF ..` |
| 3 (WebUI only) | `cmake -DENABLE_DAP=OFF -DENABLE_WEBUI=ON ..` |
| 4 (Full) | `cmake ..` (defaults: both ON) |

### CMakeLists.txt Changes Required

```cmake
# DAP subsystem (add before WebUI section)
option(ENABLE_DAP "Enable DAP server for ESP32 satellites" ON)
if(ENABLE_DAP)
    add_definitions(-DENABLE_DAP)
    message(STATUS "DAP server: ENABLED")
else()
    message(STATUS "DAP server: DISABLED")
endif()

# Conditional source inclusion
if(ENABLE_DAP)
    list(APPEND DAWN_SOURCES
        src/network/dawn_server.c
        src/network/dawn_network_audio.c
        src/network/dawn_wav_utils.c
        src/network/accept_thread.c
    )
endif()

# Auth is required if DAP or WebUI is enabled
if(ENABLE_DAP OR ENABLE_WEBUI)
    set(ENABLE_AUTH ON)
    add_definitions(-DENABLE_AUTH)
    # Requires libsodium and sqlite3
    pkg_check_modules(SODIUM REQUIRED libsodium)
    pkg_check_modules(SQLITE3 REQUIRED sqlite3)
    message(STATUS "Authentication: ENABLED (network features active)")
else()
    set(ENABLE_AUTH OFF)
    message(STATUS "Authentication: DISABLED (no network features)")
endif()
```

### Source Code Guards

```c
// In dawn.c
#ifdef ENABLE_DAP
#include "network/dawn_server.h"
#endif

// In main() or initialization
#ifdef ENABLE_DAP
   if (g_config.network.enabled) {
      dawn_server_init();
   }
#endif

#ifdef ENABLE_AUTH
   if (auth_network_features_enabled()) {
      auth_init();
   }
#endif
```

### CMake Presets (Optional)

For convenience, provide presets in `CMakePresets.json`:

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "mode1-local",
      "displayName": "Mode 1: Local Only (Armor)",
      "cacheVariables": {
        "ENABLE_DAP": "OFF",
        "ENABLE_WEBUI": "OFF"
      }
    },
    {
      "name": "mode2-dap",
      "displayName": "Mode 2: DAP Only (Headless IoT)",
      "cacheVariables": {
        "ENABLE_DAP": "ON",
        "ENABLE_WEBUI": "OFF"
      }
    },
    {
      "name": "mode3-webui",
      "displayName": "Mode 3: WebUI Only (Desktop)",
      "cacheVariables": {
        "ENABLE_DAP": "OFF",
        "ENABLE_WEBUI": "ON"
      }
    },
    {
      "name": "mode4-full",
      "displayName": "Mode 4: Full (All Features)",
      "cacheVariables": {
        "ENABLE_DAP": "ON",
        "ENABLE_WEBUI": "ON"
      }
    }
  ]
}
```

---

## CLI Administration Tool (`dawn-admin`)

### Purpose

The `dawn-admin` CLI tool provides authentication management for headless deployments (Mode 2) and serves as an alternative/backup for WebUI-based management.

**CRITICAL for Mode 2**: Without WebUI, this is the only way to create admin accounts and device tokens.

### Commands

#### User Management

```bash
# Create first admin account (requires setup token)
$ dawn-admin user create admin --admin
Enter setup token: DAWN-A7K2-M9P4-X3L8-Q5R1
Password: ********
Confirm password: ********
Admin account 'admin' created successfully.
Setup token invalidated.

# Create additional admin (requires existing admin password)
$ dawn-admin user create operator --admin
Admin password: ********
Password for new user: ********
Confirm password: ********
Admin account 'operator' created successfully.

# Create regular user
$ dawn-admin user create kris
Password: ********
Confirm password: ********
User 'kris' created successfully.

# List users
$ dawn-admin user list
ID  Username  Role   Created              Last Login
1   admin     Admin  2024-12-18 10:00:00  2024-12-18 14:30:00
2   kris      User   2024-12-18 11:00:00  Never

# Delete user
$ dawn-admin user delete olduser
Are you sure you want to delete user 'olduser'? [y/N] y
User 'olduser' deleted.

# Change password
$ dawn-admin user passwd admin
Current password: ********
New password: ********
Confirm new password: ********
Password changed for 'admin'.

# Unlock locked account
$ dawn-admin user unlock admin
Account 'admin' unlocked.
```

#### Session Management

```bash
# List active sessions
$ dawn-admin session list
Token       User   IP             Last Activity  Created
--------    -----  -------------  -------------  -------------------
a1b2c3d4..  admin  192.168.1.10   5m ago         2024-12-18 10:00:00
e5f6g7h8..  kris   192.168.1.20   1h ago         2024-12-18 09:30:00

# Revoke specific session by token prefix
$ dawn-admin session revoke a1b2c3d4
Admin password: ********
Session revoked.

# Revoke all sessions for a user
$ dawn-admin session revoke --user kris
Admin password: ********
2 sessions revoked for user 'kris'.
```

#### Device Token Management (Mode 2 Critical)

```bash
# Create device token
$ dawn-admin device create "ESP32-Kitchen"
Admin password: ********

Device Token (SAVE THIS - shown only once):
═══════════════════════════════════════════════════════════════════
  a1b2c3d4e5f6789012345678abcdef01234567890abcdef0123456789abcdef
═══════════════════════════════════════════════════════════════════

Flash this token to your ESP32 NVS storage.
Device 'ESP32-Kitchen' registered successfully.

# List devices
$ dawn-admin device list
ID  Name           Owner   Created              Last Used            Status
1   ESP32-Kitchen  admin   2024-12-18 10:30:00  2024-12-18 14:00:00  Active
2   ESP32-Bedroom  admin   2024-12-18 11:00:00  Never                Active
3   ESP32-Old      admin   2024-12-01 09:00:00  2024-12-05 10:00:00  Revoked

# Revoke device
$ dawn-admin device revoke "ESP32-Kitchen"
Device 'ESP32-Kitchen' revoked. It can no longer connect.

# Rotate token (generates new token, old token valid for 24h grace period)
$ dawn-admin device rotate "ESP32-Kitchen"
Admin password: ********

New Device Token:
═══════════════════════════════════════════════════════════════════
  f1e2d3c4b5a6789012345678fedcba98765432100abcdef0123456789abcdef
═══════════════════════════════════════════════════════════════════

Old token remains valid until: 2024-12-19 10:30:00 (24h grace period)
```

#### Audit Log

```bash
# View recent auth events
$ dawn-admin log show --last 50
2024-12-18 10:30:01 LOGIN_SUCCESS    user=admin ip=192.168.1.10
2024-12-18 10:31:15 DEVICE_CREATED   name="ESP32-Kitchen" by=admin
2024-12-18 10:45:00 LOGIN_FAILED     user=admin ip=192.168.1.99
2024-12-18 10:45:01 RATE_LIMITED     ip=192.168.1.99
2024-12-18 11:00:00 DAP_AUTH_SUCCESS device="ESP32-Kitchen" ip=192.168.1.20

# Filter by event type
$ dawn-admin log show --type LOGIN_FAILED --last 100

# Export for analysis
$ dawn-admin log export --format json --since "2024-12-01" > audit.json
```

#### Database Management

```bash
# Check database status
$ dawn-admin db status
Database: /home/user/.config/dawn/dawn.db
Schema version: 1
Users: 2
Active sessions: 3
Device tokens: 5
Auth log entries: 1,247

# Backup database
$ dawn-admin db backup /path/to/backup.db
Database backed up to /path/to/backup.db

# Compact database (run incremental vacuum)
$ dawn-admin db compact
Database compacted. Freed 2.3 MB.
```

#### IP Management

```bash
# List IPs with failed login attempts in the rate limit window
$ dawn-admin ip list
IP Address       Failed Attempts  Last Attempt
---------------  ---------------  -------------------
192.168.1.99     5                2024-12-18 14:30:00
10.0.0.15        3                2024-12-18 14:25:00

# Unblock a specific IP (clears failed attempt count)
$ dawn-admin ip unblock 192.168.1.99
Admin password: ********
IP 192.168.1.99 unblocked.

# Unblock all IPs
$ dawn-admin ip unblock --all
Admin password: ********
Cleared login attempts for 2 IPs.
```

### Security Requirements

| Requirement | Implementation | Rationale |
|-------------|----------------|-----------|
| No CLI password args | Interactive prompt only | Shell history exposure |
| Echo disabled | `tcsetattr()` with `~ECHO` | Shoulder surfing |
| Same database | `config/dawn.db` | Single source of truth |
| Same lock ordering | `s_db_mutex` rules apply | Prevent deadlocks |
| Admin required | Most ops need admin auth | Privilege escalation prevention |
| Audit logging | All CLI ops logged | Accountability |

### Implementation

```c
// dawn-admin/main.c
#include <termios.h>
#include <unistd.h>

// Secure password prompt (no echo)
static int prompt_password(const char *prompt, char *buf, size_t buflen) {
   struct termios old_term, new_term;

   // Disable echo
   if (tcgetattr(STDIN_FILENO, &old_term) != 0) {
      return AUTH_FAILURE;
   }
   new_term = old_term;
   new_term.c_lflag &= ~ECHO;
   if (tcsetattr(STDIN_FILENO, TCSANOW, &new_term) != 0) {
      return AUTH_FAILURE;
   }

   fprintf(stderr, "%s", prompt);
   if (fgets(buf, buflen, stdin) == NULL) {
      tcsetattr(STDIN_FILENO, TCSANOW, &old_term);
      return AUTH_FAILURE;
   }
   fprintf(stderr, "\n");

   // Restore terminal
   tcsetattr(STDIN_FILENO, TCSANOW, &old_term);

   // Strip newline
   buf[strcspn(buf, "\n")] = '\0';
   return AUTH_SUCCESS;
}

// Confirm password (prompt twice, compare)
static int prompt_password_confirm(char *buf, size_t buflen) {
   char confirm[256];

   if (prompt_password("Password: ", buf, buflen) != AUTH_SUCCESS) {
      return AUTH_FAILURE;
   }
   if (prompt_password("Confirm password: ", confirm, sizeof(confirm)) != AUTH_SUCCESS) {
      sodium_memzero(buf, buflen);
      return AUTH_FAILURE;
   }

   if (strcmp(buf, confirm) != 0) {
      fprintf(stderr, "Passwords do not match.\n");
      sodium_memzero(buf, buflen);
      sodium_memzero(confirm, sizeof(confirm));
      return AUTH_FAILURE;
   }

   sodium_memzero(confirm, sizeof(confirm));
   return AUTH_SUCCESS;
}
```

### Mode 2 First-Run Bootstrap

For Mode 2 (DAP only, no WebUI), the bootstrap sequence is:

```
1. Install DAWN, start service
2. Service detects no admin account
3. Generates setup token and prints to console/journald:

   ═══════════════════════════════════════════════════════════════════
     DAWN SETUP REQUIRED (Mode 2: Headless)

     Setup token: DAWN-A7K2-M9P4-X3L8-Q5R1
     (Valid for 5 minutes)

     Create admin account with:
       dawn-admin user create <username> --admin

     DAP connections will be rejected until setup is complete.
   ═══════════════════════════════════════════════════════════════════

4. User runs: dawn-admin user create admin --admin
5. dawn-admin prompts: "Enter setup token:"
6. User enters token from console output
7. Admin account created, token invalidated
8. User runs: dawn-admin device create "ESP32-Kitchen"
9. User flashes device token to ESP32
10. ESP32 connects successfully
```

### Setup Token Communication

The `dawn-admin` CLI tool communicates with the running DAWN daemon via Unix socket to validate setup tokens:

```c
// DAWN daemon creates socket on startup if no admin exists
#define ADMIN_SOCKET_PATH "/run/dawn/admin.sock"

// Socket message format
typedef struct {
   uint8_t msg_type;      // ADMIN_MSG_VALIDATE_SETUP_TOKEN
   char token[25];        // DAWN-XXXX-XXXX-XXXX-XXXX
} admin_msg_t;

// dawn-admin connects, sends token, receives AUTH_SUCCESS/AUTH_FAILURE
int validate_setup_token_via_socket(const char *token) {
   int sock = socket(AF_UNIX, SOCK_STREAM, 0);
   struct sockaddr_un addr = { .sun_family = AF_UNIX };
   strncpy(addr.sun_path, ADMIN_SOCKET_PATH, sizeof(addr.sun_path) - 1);

   if (connect(sock, (struct sockaddr *)&addr, sizeof(addr)) != 0) {
      // DAWN not running or socket not available
      return AUTH_DAEMON_UNAVAILABLE;
   }

   admin_msg_t msg = { .msg_type = ADMIN_MSG_VALIDATE_SETUP_TOKEN };
   strncpy(msg.token, token, sizeof(msg.token) - 1);

   write(sock, &msg, sizeof(msg));

   int result;
   read(sock, &result, sizeof(result));
   close(sock);

   return result;
}
```

### Security Model

The setup token proves **temporal proximity to daemon startup**:

| Access Level | Can Create First Admin? |
|--------------|------------------------|
| Started `./dawn` in terminal | ✓ Sees token in output |
| Ran `systemctl start dawn` + journald access | ✓ `journalctl -u dawn` |
| SSH access only (didn't start daemon) | ✗ Doesn't know the token |
| Attacker with stolen SSH key (arrived later) | ✗ Token may have expired |

This is not about physical console access - it's about proving you were the operator who started the daemon (or have access to its logs).

---

## Startup Security Validation

DAWN validates security configuration at startup and refuses to run in insecure states.

### Validation Rules

```c
static int validate_security_config(void) {
   int warnings = 0;
   int errors = 0;

   // Rule 1: DAP without auth is forbidden (after auth is implemented)
   #ifdef ENABLE_DAP
   if (g_config.network.enabled && !g_config.auth.require_login) {
      LOG_ERROR("SECURITY: DAP enabled without authentication");
      LOG_ERROR("Set [auth] require_login = true or disable [network]");
      errors++;
   }
   #endif

   // Rule 2: WebUI on non-localhost without HTTPS is forbidden
   #ifdef ENABLE_WEBUI
   if (g_config.webui.enabled &&
       !is_localhost_only(g_config.webui.bind_address) &&
       !g_config.webui.https) {
      LOG_ERROR("SECURITY: WebUI exposed on network without HTTPS");
      LOG_ERROR("Set webui.bind_address = \"127.0.0.1\" or enable webui.https");
      errors++;
   }
   #endif

   // Rule 3: Auth enabled but no users is a warning (first-run scenario)
   #ifdef ENABLE_AUTH
   if (auth_is_required() && auth_db_user_count() == 0) {
      LOG_WARNING("SECURITY: Auth required but no users configured");
      LOG_WARNING("Complete setup via WebUI or dawn-admin");
      warnings++;
      // Allow startup for first-run setup
   }
   #endif

   // Rule 4: DAP enabled but no device tokens is a warning
   #ifdef ENABLE_DAP
   if (g_config.network.enabled && auth_db_device_count() == 0) {
      LOG_WARNING("SECURITY: DAP enabled but no device tokens configured");
      LOG_WARNING("Create tokens with: dawn-admin device create <name>");
      warnings++;
   }
   #endif

   if (errors > 0) {
      LOG_ERROR("Startup aborted due to %d security error(s)", errors);
      return FAILURE;
   }

   if (warnings > 0) {
      LOG_WARNING("Startup proceeding with %d security warning(s)", warnings);
   }

   return SUCCESS;
}
```

### Runtime Auth Initialization

```c
// Skip auth entirely for Mode 1 (no network features)
int auth_init(void) {
   #ifndef ENABLE_AUTH
   LOG_INFO("Auth disabled at compile time (Mode 1)");
   return SUCCESS;
   #endif

   // Check if any network features are enabled at runtime
   bool dap_enabled = false;
   bool webui_enabled = false;

   #ifdef ENABLE_DAP
   dap_enabled = g_config.network.enabled;
   #endif

   #ifdef ENABLE_WEBUI
   webui_enabled = g_config.webui.enabled;
   #endif

   if (!dap_enabled && !webui_enabled) {
      LOG_INFO("No network features enabled - auth not initialized");
      g_auth_initialized = false;
      return SUCCESS;
   }

   // Initialize auth database
   int result = auth_db_init(g_config.auth_db_path);
   if (result != SUCCESS) {
      return result;
   }

   g_auth_initialized = true;

   // Run security validation
   return validate_security_config();
}
```

### Configuration Options

```toml
[auth]
# Require authentication for all network access
require_login = true

# Block startup if security validation fails (vs just warn)
strict_security = true

# Allow grace period for first-run setup (minutes)
# During this window, /setup is accessible even without auth
setup_grace_period_minutes = 30
```

### DAP Connection Rejection During Bootstrap

**CRITICAL**: The DAP server must explicitly reject all connections before authentication is configured. This prevents a race condition where a malicious client could connect during the bootstrap window.

```c
// In dawn_server.c - called when accepting a new DAP client
int dawn_server_accept_client(int client_fd, struct sockaddr_in *client_addr) {
   // Check auth state BEFORE accepting any data
   #ifdef ENABLE_AUTH
   if (!auth_is_initialized()) {
      LOG_WARNING("DAP: Rejecting connection - auth not initialized");
      dap_send_nack(client_fd, NACK_SERVICE_UNAVAILABLE);
      close(client_fd);
      return FAILURE;
   }

   if (auth_db_device_count() == 0) {
      LOG_WARNING("DAP: Rejecting connection - no device tokens configured");
      LOG_WARNING("DAP: Create tokens with: dawn-admin device create <name>");
      dap_send_nack(client_fd, NACK_AUTH_NOT_CONFIGURED);
      close(client_fd);
      return FAILURE;
   }
   #endif

   // Proceed with authenticated handshake
   return dap_handle_auth_handshake(client_fd, client_addr);
}
```

**DAP Server State Machine:**

```
┌─────────────────────┐
│  DAP_STATE_INIT     │ ◄── Server starting, not listening yet
└─────────┬───────────┘
          │ Auth subsystem initializes
          ▼
┌─────────────────────┐
│  DAP_STATE_WAITING  │ ◄── Listening, but rejecting all connections
└─────────┬───────────┘     (no device tokens configured)
          │ First device token created
          ▼
┌─────────────────────┐
│  DAP_STATE_READY    │ ◄── Normal operation, authenticated handshakes
└─────────────────────┘
```

---

## Roles

| Role | Description |
|------|-------------|
| **Admin** | Full system access. Can manage users, settings, secrets, and shutdown. |
| **User** | Can interact with AI and control devices. Has personal settings. Cannot modify system config. |

---

## Permissions Matrix

| Capability | Admin | User |
|------------|-------|------|
| **Interaction** |
| Chat with AI | Yes | Yes |
| Voice input | Yes | Yes |
| Trigger commands (lights, music, etc.) | Yes | Yes |
| View conversation history | Yes | Own |
| **Personal Settings** |
| Persona (AI personality) | Yes | Own |
| Localization (location, timezone, units) | Yes | Own |
| TTS voice / speed | Yes | Own |
| **System Settings** |
| Audio devices | Yes | No |
| ASR / VAD configuration | Yes | No |
| LLM provider / models | Yes | No |
| Network / WebUI / MQTT | Yes | No |
| Paths | Yes | No |
| Debug options | Yes | No |
| **Security** |
| API keys / secrets | Yes | No |
| Manage users | Yes | No |
| Shutdown command | Yes | No |

---

## Device Access Control

### Current Implementation (Pre-Auth)

LLM-generated commands are validated against `commands_config_nuevo.json`:
- Only devices defined in the config can be executed
- Unknown/hallucinated devices are rejected and logged
- All command attempts are logged for audit (see Security Audit Issue #10)

### Future: Per-User Device Limits

When user authentication is implemented, extend device access control:

```toml
# users/kris.toml - Per-user device restrictions
[device_access]
# Allowlist mode: only these devices can be controlled
allowed_devices = ["lights", "music", "volume", "stream"]

# Or blocklist mode: these devices are forbidden
# blocked_devices = ["shutdown", "faceplate"]

# Topic restrictions (for multi-zone setups)
allowed_topics = ["dawn", "hud"]
```

### Implementation Notes

1. **Command Validation Flow**:
   ```
   LLM generates command
   → validate_device_in_config() [current]
   → check user device permissions [future]
   → execute or reject with audit log
   ```

2. **Admin Override**: Admins bypass device restrictions

3. **Audit Trail**: All command attempts logged with:
   - Username (when auth implemented)
   - Device requested
   - Allowed/rejected status
   - Timestamp

4. **Use Cases**:
   - Kiosk user can only control music, not system settings
   - Guest user can chat but not trigger home automation
   - Child account blocked from shutdown/recording

---

## Data Model

### Storage Backend: SQLite

After evaluating TOML files vs SQLite for authentication storage, **SQLite was chosen** for:

1. **ACID transactions**: Crash-safe writes, no corrupted password hashes
2. **Concurrent access**: Multiple ESP32 clients + WebUI can authenticate simultaneously
3. **Indexed queries**: O(1) lookups vs O(n) file parsing
4. **Session persistence**: Trivial to persist sessions across restarts
5. **Referential integrity**: `ON DELETE CASCADE` cleans up related data automatically

The database file is stored at `config/dawn.db` (gitignored).

### Database Schema

```sql
-- Schema version for future migrations
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY
);
INSERT INTO schema_version VALUES (1);

-- Users table
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,       -- Argon2id hash
    is_admin INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,       -- Unix timestamp
    last_login INTEGER,
    failed_attempts INTEGER DEFAULT 0,
    lockout_until INTEGER DEFAULT 0
);

-- User preferences (replaces per-user TOML files)
CREATE TABLE user_settings (
    user_id INTEGER PRIMARY KEY,
    persona_description TEXT,
    location TEXT,
    timezone TEXT DEFAULT 'UTC',
    units TEXT DEFAULT 'metric',
    tts_voice_model TEXT,
    tts_length_scale REAL DEFAULT 1.0,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Sessions (enables persistence across restarts)
-- Note: auth_session_t in C to avoid collision with session_t (conversation sessions)
CREATE TABLE sessions (
    token TEXT PRIMARY KEY,            -- 256-bit random hex
    user_id INTEGER NOT NULL,
    created_at INTEGER NOT NULL,
    last_activity INTEGER NOT NULL,
    ip_address TEXT,
    user_agent TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_activity ON sessions(last_activity);

-- Device tokens for DAP protocol (Phase 3)
CREATE TABLE device_tokens (
    token TEXT PRIMARY KEY,            -- 256-bit random hex
    user_id INTEGER NOT NULL,
    device_name TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    last_used INTEGER,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
CREATE INDEX idx_device_tokens_user ON device_tokens(user_id);

-- Per-user device access control
CREATE TABLE user_device_access (
    user_id INTEGER NOT NULL,
    device_name TEXT NOT NULL,
    allowed INTEGER DEFAULT 1,         -- 1 = allowed, 0 = blocked
    PRIMARY KEY (user_id, device_name),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- IP-based rate limiting (prevents distributed brute force attacks)
-- Phase 1 requirement - must be implemented from day one
CREATE TABLE login_attempts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ip_address TEXT NOT NULL,
    username TEXT,                     -- NULL for invalid usernames
    timestamp INTEGER NOT NULL,
    success INTEGER DEFAULT 0
);
CREATE INDEX idx_attempts_ip ON login_attempts(ip_address, timestamp);
CREATE INDEX idx_attempts_user ON login_attempts(username, timestamp);

-- Audit log - Phase 1 requirement (moved from Phase 4)
-- Security events must be logged from day one
CREATE TABLE auth_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp INTEGER NOT NULL,
    event TEXT NOT NULL,               -- LOGIN_SUCCESS, LOGIN_FAILED, LOCKOUT, etc.
    username TEXT,
    ip_address TEXT,
    details TEXT,                      -- JSON for additional context
    session_token_prefix TEXT          -- First 8 chars for correlation
);
CREATE INDEX idx_auth_log_time ON auth_log(timestamp);
CREATE INDEX idx_auth_log_user ON auth_log(username);
```

### Log Rotation

Audit logs grow unbounded. Clean up old entries periodically:

```c
// Piggyback on session cleanup thread - keep 30 days of logs
#define AUTH_LOG_RETENTION_DAYS 30

static void cleanup_old_audit_logs(sqlite3 *db) {
   time_t cutoff = time(NULL) - (AUTH_LOG_RETENTION_DAYS * 24 * 60 * 60);
   sqlite3_stmt *stmt = ctx->stmt_delete_old_logs;
   sqlite3_bind_int64(stmt, 1, cutoff);
   sqlite3_step(stmt);
   sqlite3_reset(stmt);
}
```

Also enable incremental auto-vacuum to reclaim space:
```sql
PRAGMA auto_vacuum=INCREMENTAL;
```

### SQLite Configuration

```c
// Database context with prepared statement pool for 10x faster queries
typedef struct {
   sqlite3 *db;
   // Hot path statements (pre-compiled at init)
   sqlite3_stmt *stmt_lookup_user;
   sqlite3_stmt *stmt_lookup_session;
   sqlite3_stmt *stmt_update_session_activity;
   sqlite3_stmt *stmt_create_session;
   sqlite3_stmt *stmt_delete_expired_sessions;
   sqlite3_stmt *stmt_delete_old_attempts;
   sqlite3_stmt *stmt_delete_old_logs;
   sqlite3_stmt *stmt_log_attempt;
   sqlite3_stmt *stmt_count_recent_attempts;
} auth_db_t;

static auth_db_t s_auth_db;
static pthread_mutex_t s_db_mutex = PTHREAD_MUTEX_INITIALIZER;

int auth_db_init(const char *db_path) {
   if (sqlite3_open(db_path, &s_auth_db.db) != SQLITE_OK) {
      LOG_ERROR("Failed to open auth database: %s", sqlite3_errmsg(s_auth_db.db));
      return FAILURE;
   }

   // Optimize for embedded use
   sqlite3_exec(s_auth_db.db, "PRAGMA journal_mode=WAL", NULL, NULL, NULL);
   sqlite3_exec(s_auth_db.db, "PRAGMA synchronous=NORMAL", NULL, NULL, NULL);
   sqlite3_exec(s_auth_db.db, "PRAGMA foreign_keys=ON", NULL, NULL, NULL);
   sqlite3_exec(s_auth_db.db, "PRAGMA cache_size=16", NULL, NULL, NULL);   // 16 pages * 4KB = 64KB
   sqlite3_exec(s_auth_db.db, "PRAGMA temp_store=MEMORY", NULL, NULL, NULL);
   sqlite3_exec(s_auth_db.db, "PRAGMA auto_vacuum=INCREMENTAL", NULL, NULL, NULL);

   // Pre-compile hot path statements (10x faster than prepare-per-query)
   sqlite3_prepare_v2(s_auth_db.db,
      "SELECT id, password_hash, is_admin, lockout_until FROM users WHERE username = ?",
      -1, &s_auth_db.stmt_lookup_user, NULL);

   sqlite3_prepare_v2(s_auth_db.db,
      "SELECT user_id, created_at, last_activity, ip_address, user_agent FROM sessions WHERE token = ?",
      -1, &s_auth_db.stmt_lookup_session, NULL);

   sqlite3_prepare_v2(s_auth_db.db,
      "UPDATE sessions SET last_activity = ? WHERE token = ?",
      -1, &s_auth_db.stmt_update_session_activity, NULL);

   sqlite3_prepare_v2(s_auth_db.db,
      "DELETE FROM sessions WHERE last_activity < ?",
      -1, &s_auth_db.stmt_delete_expired_sessions, NULL);

   sqlite3_prepare_v2(s_auth_db.db,
      "DELETE FROM login_attempts WHERE timestamp < ?",
      -1, &s_auth_db.stmt_delete_old_attempts, NULL);

   sqlite3_prepare_v2(s_auth_db.db,
      "DELETE FROM auth_log WHERE timestamp < ?",
      -1, &s_auth_db.stmt_delete_old_logs, NULL);

   sqlite3_prepare_v2(s_auth_db.db,
      "SELECT COUNT(*) FROM login_attempts WHERE ip_address = ? AND timestamp > ? AND success = 0",
      -1, &s_auth_db.stmt_count_recent_attempts, NULL);

   // Session creation (hot path - used on every login)
   sqlite3_prepare_v2(s_auth_db.db,
      "INSERT INTO sessions (token, user_id, created_at, last_activity, ip_address, user_agent) "
      "VALUES (?, ?, ?, ?, ?, ?)",
      -1, &s_auth_db.stmt_create_session, NULL);

   // Login attempt logging (every login attempt)
   sqlite3_prepare_v2(s_auth_db.db,
      "INSERT INTO login_attempts (ip_address, username, timestamp, success) VALUES (?, ?, ?, ?)",
      -1, &s_auth_db.stmt_log_attempt, NULL);

   return SUCCESS;
}
```

### Error Codes

**MANDATORY**: Define numeric error codes per CLAUDE.md conventions (0 = success, 1 = generic failure, >1 = specific errors):

```c
// auth_error.h
#define AUTH_SUCCESS              0   // Operation succeeded
#define AUTH_FAILURE              1   // Generic failure
#define AUTH_INVALID_CREDENTIALS  2   // Wrong username or password
#define AUTH_TOKEN_EXPIRED        3   // Session or setup token expired
#define AUTH_TOKEN_LOCKED         4   // Too many failed attempts
#define AUTH_RATE_LIMITED         5   // IP-based rate limit exceeded
#define AUTH_TOKEN_ALREADY_USED   6   // Setup token already consumed
#define AUTH_INVALID_TOKEN        7   // Token doesn't match
#define AUTH_TOKEN_IN_USE         8   // Setup in progress by another session
#define AUTH_USER_LOCKED          9   // User account locked out
#define AUTH_SESSION_BOUND       10   // Session bound to different IP/UA
#define AUTH_INVALID_USERNAME    11   // Username doesn't meet requirements
#define AUTH_RESERVED_USERNAME   12   // Username is reserved
#define AUTH_WEAK_PASSWORD       13   // Password too weak
#define AUTH_DB_ERROR            14   // Database operation failed
#define AUTH_ENTROPY_FAILURE     15   // getrandom() failed
```

### C Structures

**IMPORTANT**: Use `auth_session_t` to avoid collision with existing `session_t` in `session_manager.h` (which manages conversation/client sessions, not authentication).

```c
#define MAX_USERS 16
#define USERNAME_MAX 32
#define AUTH_TOKEN_LEN 65  // 256-bit = 32 bytes = 64 hex chars + null

typedef struct {
   int user_id;
   char username[USERNAME_MAX];
   char password_hash[256];
   bool is_admin;
   time_t created;
   time_t last_login;
   int failed_attempts;
   time_t lockout_until;
} auth_user_t;

// Authentication session - NOT the same as session_t (conversation session)
typedef struct {
   char token[AUTH_TOKEN_LEN];
   int user_id;
   char username[USERNAME_MAX];
   bool is_admin;
   time_t created;
   time_t last_activity;
   char ip_address[64];
   char user_agent[256];
} auth_session_t;

typedef struct {
   // Personal settings (loaded per-user, falls back to global)
   persona_config_t persona;
   localization_config_t localization;
   tts_config_t tts;
} user_settings_t;
```

### Relationship: auth_session_t and session_t

The existing `session_t` in `session_manager.h` manages conversation state (LLM context, history, streaming). The new `auth_session_t` manages authentication state (user identity, permissions).

**Integration approach**: Embed `auth_session_t *` pointer in `session_t`:

```c
// In session_manager.h - add to existing session_t structure:
typedef struct session {
   uint32_t session_id;
   session_type_t type;
   // ... existing fields ...

   // Authentication context (NULL if unauthenticated)
   // IMPORTANT: This is a COPY of auth state at login time, not a live pointer.
   // - Set when: WebSocket login succeeds
   // - Cleared when: Logout or session expires
   // - No lock required for reads after initialization
   // - Modifications require: session->ref_mutex for pointer swap
   auth_session_t *auth;
} session_t;
```

**Lifecycle**:
```
session_t states:
  UNAUTHENTICATED: session->auth == NULL, only /login, /setup accessible
  AUTHENTICATED:   session->auth != NULL, full access per permissions

Transitions:
  - WebSocket connect -> UNAUTHENTICATED (until login message received)
  - Login success     -> AUTHENTICATED (auth_session_t populated as COPY)
  - Logout/expire     -> UNAUTHENTICATED (auth_session_t freed, pointer = NULL)

Special case:
  - LOCAL_SESSION_ID = 0 is exempt from authentication (trusted local microphone)
```

This allows permission checks like:
```c
if (session->auth && session->auth->is_admin) {
   // Admin operation allowed
}
```

**IMPORTANT**: The `auth` pointer is a **cached copy** of authentication state. This means:
- No database lock needed for permission checks (fast path)
- Session refresh updates the copy if needed
- If auth session is invalidated server-side, next refresh will clear the pointer

---

## Authentication Flow

### First-Run Setup

**Token-Based Setup** (headless-friendly):

```
1. DAWN starts with no users in database
2. Generate one-time setup token, print to console/logs
3. User accesses /setup from any IP
4. User enters setup token + creates admin account
5. Token invalidated, database populated
6. Redirect to login page
```

**Console Output on First Run:**
```
═══════════════════════════════════════════════════════════════════
  DAWN SETUP REQUIRED

  No admin account configured. To complete setup:
  1. Open http://<your-ip>:3000/setup in a browser
  2. Enter setup token: DAWN-7K3M-9X2P-4QRT-5NHV

  This token expires in 5 minutes (configurable).
  Token is single-use and will be invalidated after setup.
═══════════════════════════════════════════════════════════════════
```

**Note**: Token is printed to stderr only (not logged to files) to minimize exposure. Clear terminal scrollback after setup.

**Why Token-Based?**
- Works on headless systems (Jetson, Raspberry Pi) without local display
- No SSH tunnel required - just need console/log access to see token
- Proves the person setting up has physical or SSH access to the machine
- Single code path for all setup scenarios

**Security Considerations**:

1. **Token Entropy**: ~83-bit random token formatted as `DAWN-XXXX-XXXX-XXXX-XXXX` (16 chars from 34-char alphabet = log2(34^16) ≈ 81 bits, using rejection sampling to eliminate modulo bias)
2. **Time Window**: Token expires after configurable timeout (default 5 minutes)
3. **Single Use**: Token invalidated only after successful account creation (atomic)
4. **Rate Limiting**: Max 5 token attempts before lockout
5. **Audit Log**: Failed token attempts are logged
6. **Memory Safety**: Token wiped from memory immediately after use

**Implementation:**

```c
// Setup token state - protected by dedicated mutex
typedef struct {
   char token[25];           // "DAWN-XXXX-XXXX-XXXX-XXXX" + null (16 random chars)
   time_t created_at;
   int failed_attempts;
   bool used;
   bool in_progress;         // TOCTOU protection: setup wizard Step 2 active
   time_t in_progress_since; // Timeout for abandoned Step 2
} setup_token_t;

static setup_token_t s_setup_token = {0};
static pthread_mutex_t s_setup_mutex = PTHREAD_MUTEX_INITIALIZER;

// Generate setup token on startup if no users exist
int auth_init(void) {
   if (auth_db_user_count() == 0) {
      auth_generate_setup_token();
      LOG_INFO("═══════════════════════════════════════════════════════════════════");
      LOG_INFO("  DAWN SETUP REQUIRED");
      LOG_INFO("");
      LOG_INFO("  No admin account configured. To complete setup:");
      LOG_INFO("  1. Open http://<your-ip>:%d/setup in a browser",
               g_config.webui.port);
      LOG_INFO("  2. Enter setup token: %s", s_setup_token.token);
      LOG_INFO("");
      LOG_INFO("  This token expires in %d minutes.",
               g_config.auth.setup_timeout_minutes);
      LOG_INFO("═══════════════════════════════════════════════════════════════════");
   }
   return SUCCESS;
}

// Generate token in format DAWN-XXXX-XXXX-XXXX-XXXX (16 random chars, ~81 bits entropy)
// Uses rejection sampling to eliminate modulo bias
static int auth_generate_setup_token(void) {
   static const char charset[] = "0123456789ABCDEFGHJKLMNPQRSTUVWXYZ"; // No I,O (ambiguous)
   #define CHARSET_LEN 34
   #define TOKEN_CHARS 16

   // Rejection threshold: largest multiple of 34 that fits in a byte
   // 256 / 34 = 7 remainder 14, so reject >= 238
   const uint8_t reject_threshold = 256 - (256 % CHARSET_LEN);  // 238

   uint8_t random_bytes[TOKEN_CHARS * 2];  // Extra bytes for rejection sampling
   ssize_t ret = getrandom(random_bytes, sizeof(random_bytes), 0);
   if (ret != sizeof(random_bytes)) {
      LOG_ERROR("FATAL: getrandom() failed for setup token: %s",
                ret == -1 ? strerror(errno) : "partial read");
      return AUTH_ENTROPY_FAILURE;
   }

   int byte_idx = 0;
   int char_count = 0;
   char chars[TOKEN_CHARS];

   while (char_count < TOKEN_CHARS) {
      if (byte_idx >= (int)sizeof(random_bytes)) {
         // Need more entropy - extremely rare (probability < 2^-40)
         ret = getrandom(random_bytes, sizeof(random_bytes), 0);
         if (ret != sizeof(random_bytes)) {
            LOG_ERROR("FATAL: getrandom() failed on retry");
            return AUTH_ENTROPY_FAILURE;
         }
         byte_idx = 0;
      }

      uint8_t b = random_bytes[byte_idx++];
      if (b < reject_threshold) {
         chars[char_count++] = charset[b % CHARSET_LEN];
      }
      // else: reject and try next byte (unbiased selection)
   }

   pthread_mutex_lock(&s_setup_mutex);
   snprintf(s_setup_token.token, sizeof(s_setup_token.token),
            "DAWN-%c%c%c%c-%c%c%c%c-%c%c%c%c-%c%c%c%c",
            chars[0], chars[1], chars[2], chars[3],
            chars[4], chars[5], chars[6], chars[7],
            chars[8], chars[9], chars[10], chars[11],
            chars[12], chars[13], chars[14], chars[15]);

   s_setup_token.created_at = time(NULL);
   s_setup_token.failed_attempts = 0;
   s_setup_token.used = false;
   s_setup_token.in_progress = false;
   pthread_mutex_unlock(&s_setup_mutex);

   // Wipe temporary buffers
   sodium_memzero(random_bytes, sizeof(random_bytes));
   sodium_memzero(chars, sizeof(chars));

   return AUTH_SUCCESS;

   #undef CHARSET_LEN
   #undef TOKEN_CHARS
}

// Validate setup token - Step 1 of setup wizard
// Returns AUTH_SUCCESS if token valid and not already in use
// Sets in_progress=true to prevent TOCTOU race with concurrent requests
int auth_validate_setup_token(const char *provided_token, const char *ip_addr) {
   pthread_mutex_lock(&s_setup_mutex);

   // Already used and completed?
   if (s_setup_token.used) {
      pthread_mutex_unlock(&s_setup_mutex);
      return AUTH_TOKEN_ALREADY_USED;
   }

   // Expired? Wipe token on expiry
   time_t now = time(NULL);
   time_t elapsed = now - s_setup_token.created_at;
   if (elapsed > g_config.auth.setup_timeout_minutes * 60) {
      sodium_memzero(s_setup_token.token, sizeof(s_setup_token.token));
      pthread_mutex_unlock(&s_setup_mutex);
      return AUTH_TOKEN_EXPIRED;
   }

   // Too many failures? Wipe token on lockout
   if (s_setup_token.failed_attempts >= 5) {
      sodium_memzero(s_setup_token.token, sizeof(s_setup_token.token));
      pthread_mutex_unlock(&s_setup_mutex);
      return AUTH_TOKEN_LOCKED;
   }

   // Already in progress by another session? (TOCTOU protection)
   if (s_setup_token.in_progress) {
      // Allow if abandoned (>5 min since Step 1)
      if (now - s_setup_token.in_progress_since > 300) {
         s_setup_token.in_progress = false;  // Reset abandoned attempt
      } else {
         pthread_mutex_unlock(&s_setup_mutex);
         return AUTH_TOKEN_IN_USE;
      }
   }

   // Validate (constant-time comparison)
   size_t token_len = strnlen(s_setup_token.token, sizeof(s_setup_token.token));
   size_t provided_len = strnlen(provided_token, sizeof(s_setup_token.token) + 1);

   // Length check (constant-time safe - just bounds check)
   volatile size_t len_diff = provided_len ^ token_len;

   // Always do full comparison regardless of length
   int cmp_result = sodium_memcmp(provided_token, s_setup_token.token, token_len);

   if (len_diff != 0 || cmp_result != 0) {
      s_setup_token.failed_attempts++;
      pthread_mutex_unlock(&s_setup_mutex);
      auth_log_event("SETUP_TOKEN_FAILED", NULL, ip_addr, NULL);
      return AUTH_INVALID_TOKEN;
   }

   // Success - mark as in progress (NOT used yet - that happens on account creation)
   s_setup_token.in_progress = true;
   s_setup_token.in_progress_since = now;
   pthread_mutex_unlock(&s_setup_mutex);

   auth_log_event("SETUP_TOKEN_VALIDATED", NULL, ip_addr, NULL);
   return AUTH_SUCCESS;
}

// Complete setup - Step 2 of setup wizard
// ATOMIC: Token is invalidated only AFTER successful account creation
int auth_complete_setup(const char *username, const char *password, const char *ip_addr) {
   pthread_mutex_lock(&s_setup_mutex);

   // Verify still in valid state
   if (s_setup_token.used) {
      pthread_mutex_unlock(&s_setup_mutex);
      return AUTH_TOKEN_ALREADY_USED;
   }
   if (!s_setup_token.in_progress) {
      pthread_mutex_unlock(&s_setup_mutex);
      return AUTH_INVALID_TOKEN;  // Must validate token first
   }

   // Release mutex during potentially slow password hashing
   pthread_mutex_unlock(&s_setup_mutex);

   // Validate username format
   int result = auth_validate_username(username);
   if (result != AUTH_SUCCESS) {
      auth_cancel_setup();
      return result;
   }

   // Create admin account (includes password hashing)
   result = auth_create_user(username, password, true /* is_admin */);
   if (result != AUTH_SUCCESS) {
      auth_cancel_setup();  // Token still valid for retry
      return result;
   }

   // Success - NOW invalidate token (atomic with account creation)
   pthread_mutex_lock(&s_setup_mutex);
   sodium_memzero(s_setup_token.token, sizeof(s_setup_token.token));
   s_setup_token.used = true;
   s_setup_token.in_progress = false;
   pthread_mutex_unlock(&s_setup_mutex);

   auth_log_event("SETUP_COMPLETED", username, ip_addr, NULL);
   return AUTH_SUCCESS;
}

// Cancel setup - called if Step 2 fails or is abandoned
void auth_cancel_setup(void) {
   pthread_mutex_lock(&s_setup_mutex);
   s_setup_token.in_progress = false;
   // Token remains valid for retry
   pthread_mutex_unlock(&s_setup_mutex);
}
```

**Username Validation:**

```c
#define USERNAME_MIN 3
#define USERNAME_MAX 32
#define USERNAME_PATTERN "^[a-zA-Z][a-zA-Z0-9_-]{2,31}$"

int auth_validate_username(const char *username) {
   if (!username) return AUTH_INVALID_USERNAME;

   size_t len = strnlen(username, USERNAME_MAX + 1);

   // Length check
   if (len < USERNAME_MIN || len > USERNAME_MAX) {
      return AUTH_INVALID_USERNAME;
   }

   // Must start with letter
   if (!isalpha(username[0])) {
      return AUTH_INVALID_USERNAME;
   }

   // Allowed: alphanumeric, underscore, hyphen
   for (size_t i = 1; i < len; i++) {
      char c = username[i];
      if (!isalnum(c) && c != '_' && c != '-') {
         return AUTH_INVALID_USERNAME;
      }
   }

   // Reserved username check (case-insensitive)
   static const char *reserved[] = {"root", "system", "dawn", "admin", NULL};
   for (int i = 0; reserved[i]; i++) {
      if (strcasecmp(username, reserved[i]) == 0) {
         // "admin" allowed only for first user
         if (strcmp(reserved[i], "admin") == 0 && auth_db_user_count() == 0) {
            continue;
         }
         return AUTH_RESERVED_USERNAME;
      }
   }

   return AUTH_SUCCESS;
}
```

**Token Regeneration:**

If the token expires or is locked out, restart DAWN to generate a new one. This is intentional - it requires console access to restart, proving physical/SSH access.

### Login Flow

```
Browser                              Server
   |                                    |
   |-- GET / -------------------------->|
   |<-- 302 /login (no session) --------|
   |                                    |
   |-- GET /login --------------------->|
   |<-- Login page ---------------------|
   |                                    |
   |-- POST /login {user, pass} ------->|
   |   [verify password hash]           |
   |   [create session]                 |
   |   [load user settings]             |
   |<-- Set-Cookie: dawn_session=xxx ---|
   |<-- 302 / --------------------------|
   |                                    |
   |-- WS /ws (Cookie: dawn_session) -->|
   |   [validate session]               |
   |<-- WS connected -------------------|
```

### Session Management

- Sessions stored server-side (in-memory hash table)
- Session token: 256-bit random (via `getrandom()`)
- Default timeout: 24 hours of inactivity
- Max sessions per user: 5 (configurable)
- Session includes: username, is_admin, user_settings pointer

---

## WebSocket API Changes

### Authentication Messages

```json
// Already authenticated via cookie, but can also login via WS:
{"type": "login", "username": "kris", "password": "..."}
{"type": "login_response", "success": true, "username": "kris", "is_admin": false}

{"type": "logout"}
{"type": "logout_response", "success": true}
```

### User Management (Admin Only)

```json
// List users
{"type": "get_users"}
{"type": "get_users_response", "users": [
   {"username": "admin", "is_admin": true, "created": "...", "last_login": "..."},
   {"username": "kris", "is_admin": false, "created": "...", "last_login": "..."}
]}

// Create user
{"type": "create_user", "username": "newuser", "password": "...", "is_admin": false}
{"type": "create_user_response", "success": true}

// Delete user
{"type": "delete_user", "username": "olduser"}
{"type": "delete_user_response", "success": true}

// Change password (admin can change any, user can change own)
{"type": "change_password", "username": "kris", "new_password": "..."}
{"type": "change_password_response", "success": true}
```

### Personal Settings

```json
// Get own settings (merged: user overrides + global defaults)
{"type": "get_my_settings"}
{"type": "get_my_settings_response", "settings": {
   "persona": {"description": "..."},
   "localization": {"location": "...", "timezone": "...", "units": "..."},
   "tts": {"voice_model": "...", "length_scale": 1.0}
}}

// Update own settings
{"type": "set_my_settings", "settings": {
   "localization": {"location": "New York, NY"}
}}
{"type": "set_my_settings_response", "success": true}
```

---

## Permission Enforcement

### Server-Side Checks

```c
// In message handler
static void handle_set_config(ws_connection_t *conn, json_object *payload) {
   if (!conn->session || !conn->session->is_admin) {
      send_error(conn, "FORBIDDEN", "Admin access required");
      return;
   }
   // ... proceed with config change
}

static void handle_set_my_settings(ws_connection_t *conn, json_object *payload) {
   if (!conn->session) {
      send_error(conn, "UNAUTHORIZED", "Login required");
      return;
   }
   // ... update user's personal settings (any logged-in user can do this)
}
```

### Protected Operations

Operations requiring admin:
- `set_config` - System settings
- `set_secrets` - API keys
- `create_user`, `delete_user` - User management
- `shutdown` command
- Changing another user's password

Operations requiring login (any user):
- `set_my_settings` - Personal settings
- `change_password` (own password only)
- Chat / voice interaction
- Command execution

### Information Disclosure Protection

The `get_config` response exposes filesystem paths that reveal system structure:
- `config_path` - Path to dawn.toml
- `secrets_path` - Path to secrets.toml
- `asr_path` - Path to ASR models
- `tts_path` - Path to TTS models

**Mitigation**: When returning config to non-admin users, redact or omit these paths:

```c
if (conn->session->is_admin) {
   json_object_object_add(payload, "config_path", json_object_new_string(config_path));
   // ... other paths
} else {
   // Omit paths for non-admin users, or show generic placeholders
   json_object_object_add(payload, "config_path", json_object_new_string("(configured)"));
}
```

This prevents information disclosure while still allowing admins to see full system paths for debugging. See Security Audit Issue #13.

---

## WebUI Changes

### New Pages

### Design Philosophy

The auth UI follows the existing "Stark-grade" aesthetic:
- **Dark theme**: `#121417` primary, `#1b1f24` secondary
- **Glass-morphism**: `backdrop-filter: blur(20px)` on overlays
- **Cyan accent**: `#2dd4bf` for active elements only
- **Amber warning**: `#f0b429` for attention states
- **Typography**: IBM Plex Mono for system text, Source Sans 3 for human content
- **Minimal animations**: Purposeful, not decorative

### New CSS Variables

```css
:root {
  /* Auth-specific additions */
  --auth-card-bg: rgba(18, 18, 26, 0.92);
  --auth-card-border: rgba(34, 211, 238, 0.2);
  --auth-input-bg: var(--bg-tertiary);
  --auth-input-border: rgba(255, 255, 255, 0.1);
  --auth-input-border-focus: var(--accent);

  /* Admin badge */
  --admin-badge-bg: rgba(45, 212, 191, 0.15);
  --admin-badge-border: rgba(45, 212, 191, 0.3);
  --admin-badge-color: var(--accent);

  /* Lockout state */
  --lockout-bg: rgba(240, 180, 41, 0.15);
  --lockout-border: rgba(240, 180, 41, 0.4);
}
```

### Auth State Colors

| State | Color | Variable | Usage |
|-------|-------|----------|-------|
| Normal | Gray | `--text-secondary` | Default input borders |
| Focus | Cyan | `--accent` | Focused inputs, hover |
| Success | Green | `--success` | Login success, saved |
| Error | Red | `--error` | Failed login, validation |
| Warning | Amber | `--warning` | Lockout, rate limit |
| Admin | Cyan | `--accent` | Admin badge |

---

### 1. Login Page (`/login`)

**Layout:**
```
+----------------------------------------------------------+
|                                                          |
|                    [DAWN Ring Logo]                      |
|                       D.A.W.N.                           |
|                                                          |
|              +----------------------------+              |
|              |   Username                 |              |
|              | [________________________] |              |
|              |                            |              |
|              |   Password                 |              |
|              | [________________________] |              |
|              |                            |              |
|              |   [x] Remember this device |              |
|              |                            |              |
|              |   [     Sign In     ]      |              |
|              +----------------------------+              |
|                                                          |
|              [Error message area - hidden]               |
+----------------------------------------------------------+
```

**Login Card Styling:**
```css
.login-page {
  background: var(--bg-primary);
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}

.login-card {
  width: 100%;
  max-width: 360px;
  background: var(--auth-card-bg);
  backdrop-filter: blur(20px);
  border: 1px solid var(--auth-card-border);
  border-radius: var(--border-radius);
  padding: 2rem;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.5);
  animation: login-entrance 0.4s ease-out;
}

@keyframes login-entrance {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}
```

**Logo/Branding:**
```css
.login-branding {
  text-align: center;
  margin-bottom: 2rem;
}

.login-logo {
  width: 80px;
  height: 80px;
  margin: 0 auto 1rem;
  /* Static ring graphic - no animation to avoid distraction */
}

.login-title {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 1.5rem;
  font-weight: 600;
  color: var(--accent);
  letter-spacing: 0.15em;
}
```

**Form Fields:**
```css
.login-field {
  margin-bottom: 1.25rem;
}

.login-field label {
  display: block;
  font-size: 0.8125rem;
  color: var(--text-secondary);
  margin-bottom: 0.375rem;
  font-weight: 500;
}

.login-field input {
  width: 100%;
  padding: 0.75rem 1rem;
  background: var(--auth-input-bg);
  border: 1px solid var(--auth-input-border);
  border-radius: 6px;
  color: var(--text-primary);
  font-size: 0.9375rem;
  transition: border-color var(--transition-fast);
}

.login-field input:focus {
  outline: none;
  border-color: var(--auth-input-border-focus);
  box-shadow: 0 0 0 2px rgba(45, 212, 191, 0.15);
}
```

**Submit Button:**
```css
.login-submit {
  width: 100%;
  padding: 0.875rem;
  background: var(--accent);
  border: none;
  border-radius: var(--border-radius);
  color: var(--bg-primary);
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all var(--transition-fast);
}

.login-submit:hover:not(:disabled) {
  filter: brightness(1.1);
  transform: translateY(-1px);
}

.login-submit:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Loading spinner */
.login-submit.loading {
  position: relative;
  color: transparent;
}

.login-submit.loading::after {
  content: '';
  position: absolute;
  width: 20px;
  height: 20px;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border: 2px solid var(--bg-primary);
  border-top-color: transparent;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
```

**Error Display:**
```css
.login-error {
  margin-top: 1rem;
  padding: 0.75rem 1rem;
  background: rgba(239, 68, 68, 0.15);
  border: 1px solid rgba(239, 68, 68, 0.3);
  border-radius: var(--border-radius);
  color: var(--error);
  font-size: 0.875rem;
  text-align: center;
}

/* Error shake animation */
.login-card.error-shake {
  animation: shake 0.4s ease-out;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20% { transform: translateX(-8px); }
  40% { transform: translateX(8px); }
  60% { transform: translateX(-4px); }
  80% { transform: translateX(4px); }
}
```

**Lockout/Rate Limit Warning:**
```css
.login-lockout {
  background: var(--lockout-bg);
  border: 1px solid var(--lockout-border);
  color: var(--warning);
  padding: 0.75rem 1rem;
  border-radius: var(--border-radius);
  font-size: 0.875rem;
  text-align: center;
  margin-bottom: 1rem;
}
```

---

### 2. Setup Wizard (`/setup`)

First-run only, token-protected. Feels like "initializing a new system."

**Layout:**
```
+----------------------------------------------------------+
|                                                          |
|                    [DAWN Ring Logo]                      |
|                  SYSTEM INITIALIZATION                   |
|                                                          |
|     +----------------------------------------------+     |
|     |   Step 1 of 2: Verify Access                 |     |
|     |   ─────────────────────────────────────────  |     |
|     |                                              |     |
|     |   Enter the setup token from the DAWN        |     |
|     |   console output to continue.                |     |
|     |                                              |     |
|     |   Setup Token                                |     |
|     |   DAWN - [____] - [____] - [____] - [____]   |     |
|     |                                              |     |
|     |   [     Verify Token     ]                   |     |
|     |                                              |     |
|     +----------------------------------------------+     |
|                                                          |
+----------------------------------------------------------+

        ↓ After valid token ↓

+----------------------------------------------------------+
|                                                          |
|                    [DAWN Ring Logo]                      |
|                  SYSTEM INITIALIZATION                   |
|                                                          |
|     +----------------------------------------------+     |
|     |   Step 2 of 2: Create Administrator          |     |
|     |   ─────────────────────────────────────────  |     |
|     |                                              |     |
|     |   Administrator Username                     |     |
|     |   [________________________________]         |     |
|     |                                              |     |
|     |   Password                                   |     |
|     |   [________________________________]         |     |
|     |                                              |     |
|     |   Confirm Password                           |     |
|     |   [________________________________]         |     |
|     |                                              |     |
|     |   Password Requirements:                     |     |
|     |   [✓] At least 8 characters                  |     |
|     |   [ ] Contains number or symbol              |     |
|     |                                              |     |
|     +----------------------------------------------+     |
|                                                          |
|                  [     Create Admin     ]                |
|                                                          |
+----------------------------------------------------------+
```

**Setup Card Styling:**
```css
.setup-card {
  width: 100%;
  max-width: 480px;
  background: rgba(18, 18, 26, 0.95);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);  /* Safari */
  border: 1px solid rgba(34, 211, 238, 0.25);
  border-radius: var(--border-radius);
  overflow: hidden;
}

.setup-header {
  padding: 1.5rem;
  background: rgba(34, 211, 238, 0.08);
  border-bottom: 1px solid rgba(34, 211, 238, 0.15);
}

.setup-step {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 0.75rem;
  color: var(--accent);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin-bottom: 0.25rem;
}

.setup-title {
  font-size: 1.125rem;
  font-weight: 600;
  color: var(--text-primary);
}

/* Step transition animation */
.setup-step-content {
  animation: step-enter 0.3s ease-out;
}

@keyframes step-enter {
  from {
    opacity: 0;
    transform: translateX(20px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

/* Token verified success state */
.token-input-group.valid .token-segment {
  border-color: var(--success);
  background: rgba(34, 197, 94, 0.08);
}

.token-success-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  margin-top: 1rem;
  color: var(--success);
  font-size: 0.875rem;
  animation: fade-in 0.3s ease-out;
}

.token-success-indicator svg {
  width: 20px;
  height: 20px;
}

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* Setup verify/submit button loading state */
.setup-submit {
  position: relative;
  transition: all var(--transition-fast);
}

.setup-submit.loading {
  color: transparent;
  pointer-events: none;
}

.setup-submit.loading::after {
  content: '';
  position: absolute;
  width: 20px;
  height: 20px;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border: 2px solid rgba(18, 18, 26, 0.3);
  border-top-color: var(--bg-primary);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: translate(-50%, -50%) rotate(360deg); }
}
```

**Password Requirements Checklist:**
```css
.password-requirements {
  margin-top: 1rem;
  font-size: 0.8125rem;
}

.requirement {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.25rem 0;
  color: var(--text-secondary);
  transition: color var(--transition-fast);
}

.requirement.met {
  color: var(--success);
}
```

**Token Input Styling (Segmented):**

The token input uses 4 separate segments that auto-advance for better UX:

```html
<div class="token-input-group">
  <span class="token-prefix">DAWN</span>
  <span class="token-separator">-</span>
  <input type="text" class="token-segment" id="seg1" maxlength="4"
         pattern="[0-9A-Z]{4}" autocomplete="off" aria-label="Token segment 1">
  <span class="token-separator">-</span>
  <input type="text" class="token-segment" id="seg2" maxlength="4"
         pattern="[0-9A-Z]{4}" autocomplete="off" aria-label="Token segment 2">
  <span class="token-separator">-</span>
  <input type="text" class="token-segment" id="seg3" maxlength="4"
         pattern="[0-9A-Z]{4}" autocomplete="off" aria-label="Token segment 3">
  <span class="token-separator">-</span>
  <input type="text" class="token-segment" id="seg4" maxlength="4"
         pattern="[0-9A-Z]{4}" autocomplete="off" aria-label="Token segment 4">
</div>
```

```css
.token-input-group {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.25rem;
  flex-wrap: wrap;
}

.token-prefix {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 1.25rem;
  font-weight: 600;
  color: var(--accent);
  letter-spacing: 0.1em;
}

.token-separator {
  color: var(--text-secondary);
  font-weight: 600;
  font-size: 1.25rem;
}

.token-segment {
  width: 5rem;
  font-family: 'IBM Plex Mono', monospace;
  font-size: 1.25rem;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  text-align: center;
  padding: 0.75rem 0.5rem;
  background: var(--bg-tertiary);
  border: 2px solid var(--auth-input-border);
  border-radius: var(--border-radius);
  color: var(--text-primary);
  transition: border-color var(--transition-fast), background var(--transition-fast);
}

.token-segment:focus {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(45, 212, 191, 0.2);
}

.token-segment.filled {
  background: rgba(45, 212, 191, 0.08);
  border-color: rgba(45, 212, 191, 0.4);
}

.token-segment.invalid {
  border-color: var(--error);
  animation: shake 0.4s ease-out;
}

/* Mobile: stack segments */
@media (max-width: 400px) {
  .token-input-group {
    flex-direction: column;
    gap: 0.5rem;
  }
  .token-separator { display: none; }
  .token-segment { width: 100%; }
}
```

```javascript
// Auto-advance between token segments
document.querySelectorAll('.token-segment').forEach((input, index, inputs) => {
  input.addEventListener('input', (e) => {
    // Force uppercase
    e.target.value = e.target.value.toUpperCase().replace(/[^0-9A-Z]/g, '');

    // Mark as filled
    e.target.classList.toggle('filled', e.target.value.length === 4);

    // Auto-advance to next segment
    if (e.target.value.length === 4 && index < inputs.length - 1) {
      inputs[index + 1].focus();
    }
  });

  // Handle backspace to go to previous segment
  input.addEventListener('keydown', (e) => {
    if (e.key === 'Backspace' && e.target.value === '' && index > 0) {
      inputs[index - 1].focus();
    }
  });

  // Handle paste of full token
  input.addEventListener('paste', (e) => {
    const pasted = e.clipboardData.getData('text').toUpperCase()
      .replace(/[^0-9A-Z]/g, '');  // Strip DAWN- prefix and hyphens
    if (pasted.length >= 16) {
      e.preventDefault();
      for (let i = 0; i < 4 && i < inputs.length; i++) {
        inputs[i].value = pasted.substr(i * 4, 4);
        inputs[i].classList.add('filled');
      }
      inputs[inputs.length - 1].focus();
    }
  });
});
```
```

**Token Expiry Countdown:**
```html
<div class="token-expiry" aria-live="polite">
  Token expires in <span class="token-expiry-value" id="expiry-countdown">04:32</span>
</div>
```

```css
.token-expiry {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 0.75rem;
  color: var(--text-secondary);
  text-align: center;
  margin-top: 1rem;
}

.token-expiry.warning {
  color: var(--warning);
}

.token-expiry.critical {
  color: var(--error);
  animation: pulse 1s ease-in-out infinite;
}

.token-expiry-value {
  color: var(--accent);
  font-weight: 500;
}

.token-expiry.warning .token-expiry-value,
.token-expiry.critical .token-expiry-value {
  color: inherit;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.6; }
}
```

```javascript
// Token expiry countdown
function initExpiryCountdown(expiresAt) {
  const countdown = document.getElementById('expiry-countdown');
  const container = countdown.parentElement;

  function update() {
    const remaining = Math.max(0, expiresAt - Date.now());
    const minutes = Math.floor(remaining / 60000);
    const seconds = Math.floor((remaining % 60000) / 1000);

    countdown.textContent = `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;

    // Update warning states
    container.classList.toggle('warning', remaining < 120000 && remaining > 30000);
    container.classList.toggle('critical', remaining <= 30000);

    if (remaining <= 0) {
      countdown.textContent = '00:00';
      // Redirect to show expired message
      showTokenExpired();
    } else {
      requestAnimationFrame(update);
    }
  }

  update();
}
```

**Token Error States:**
```css
.token-error {
  margin-top: 0.75rem;
  padding: 0.5rem;
  font-size: 0.8125rem;
  color: var(--error);
  text-align: center;
  role: alert;
}

.token-expired {
  background: var(--lockout-bg);
  border: 1px solid var(--lockout-border);
  color: var(--warning);
  padding: 1rem;
  border-radius: var(--border-radius);
  text-align: center;
}

.token-expired-hint {
  font-size: 0.75rem;
  margin-top: 0.5rem;
  color: var(--text-secondary);
}

/* Connection/offline error */
.setup-offline-notice {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  background: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.3);
  border-radius: var(--border-radius);
  color: var(--error);
  font-size: 0.8125rem;
  margin-bottom: 1rem;
}
```

**Help Text:**
```css
.setup-help {
  margin-top: 1.5rem;
  padding: 1rem;
  background: rgba(45, 212, 191, 0.08);
  border: 1px solid rgba(45, 212, 191, 0.2);
  border-radius: var(--border-radius);
  font-size: 0.8125rem;
  color: var(--text-secondary);
}

.setup-help code {
  font-family: 'IBM Plex Mono', monospace;
  background: rgba(0, 0, 0, 0.3);
  padding: 0.125rem 0.375rem;
  border-radius: 3px;
  color: var(--accent);
}
```

---

### 3. User Management (Settings > Users)

Admin-only section in existing settings panel.

**Section Structure:**
```html
<div class="settings-section" id="users-section">
  <h3 class="section-header" data-section="users">
    <span class="section-icon">👤</span>
    User Management
    <span class="admin-only-badge">Admin</span>
    <span class="section-toggle">▼</span>
  </h3>
  <div class="section-content">
    <!-- User list -->
  </div>
</div>
```

**User List Layout:**
```
+------------------------------------------------+
|  USER MANAGEMENT                    [+ Add]    |
|------------------------------------------------|
|  admin                              [Admin]    |
|  Last login: 2 hours ago                       |
|  ─────────────────────────────────────────     |
|  kris                                          |
|  Last login: Yesterday                         |
|                               [Reset] [Delete] |
+------------------------------------------------+
```

**User Item Styling:**
```css
.user-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.user-item {
  padding: 1rem;
  background: rgba(0, 0, 0, 0.2);
  border-radius: 6px;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.user-item-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.25rem;
}

.user-name {
  font-weight: 600;
  color: var(--text-primary);
}

.user-admin-badge {
  display: inline-flex;
  padding: 0.125rem 0.5rem;
  font-size: 0.625rem;
  font-weight: 600;
  color: var(--admin-badge-color);
  background: var(--admin-badge-bg);
  border: 1px solid var(--admin-badge-border);
  border-radius: 1rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.user-last-login {
  font-size: 0.75rem;
  color: var(--text-secondary);
}

.user-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

.user-action-btn {
  padding: 0.375rem 0.75rem;
  font-size: 0.75rem;
  background: transparent;
  border: 1px solid var(--text-secondary);
  border-radius: 4px;
  color: var(--text-secondary);
  cursor: pointer;
  transition: all var(--transition-fast);
}

.user-action-btn:hover {
  border-color: var(--accent);
  color: var(--accent);
}

.user-action-btn.danger:hover {
  border-color: var(--error);
  color: var(--error);
}
```

**Add User Modal:**
```css
.add-user-modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100%;
  max-width: 400px;
  background: rgba(18, 18, 26, 0.98);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(34, 211, 238, 0.2);
  border-radius: var(--border-radius);
  padding: 1.5rem;
  z-index: 1000;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.6);
}

/* Mobile: full-screen modal */
@media (max-width: 500px) {
  .add-user-modal {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    max-width: none;
    border-radius: 0;
    transform: none;
  }
}
```

---

### 4. Profile Section (Settings > Profile)

Available to all logged-in users for changing own password and personal settings.

**Layout:**
```
+------------------------------------------------+
|  YOUR PROFILE                                  |
|------------------------------------------------|
|  Username: kris                                |
|                                                |
|  Change Password                               |
|  ─────────────────────────────────────────     |
|  Current Password                              |
|  [________________________________]            |
|  New Password                                  |
|  [________________________________]            |
|  Confirm New Password                          |
|  [________________________________]            |
|  [    Update Password    ]                     |
|                                                |
|  Personal Settings                             |
|  ─────────────────────────────────────────     |
|  Persona Description                           |
|  [________________________________]            |
|  Location / Timezone / TTS Voice               |
|  [    Save Preferences    ]                    |
+------------------------------------------------+
```

Uses existing settings-section styling.

---

### 5. Header User Indicator

**Location in header:**
```html
<div id="header-controls">
  <button id="debug-btn">...</button>
  <button id="metrics-btn">...</button>

  <!-- NEW: User indicator -->
  <div id="user-indicator">
    <span id="current-user">kris</span>
    <button id="logout-btn" title="Logout">
      <svg><!-- Logout icon --></svg>
    </button>
  </div>

  <button id="settings-btn">...</button>
  <span id="connection-status">Connected</span>
</div>
```

**Styling:**
```css
#user-indicator {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.25rem 0.5rem;
  background: rgba(45, 212, 191, 0.1);
  border: 1px solid rgba(45, 212, 191, 0.2);
  border-radius: 1rem;
}

#current-user {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 0.75rem;
  color: var(--text-primary);
  letter-spacing: 0.02em;
}

#current-user.is-admin::after {
  content: ' (Admin)';
  color: var(--accent);
  font-weight: 500;
}

#logout-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  padding: 0;
  background: transparent;
  border: none;
  color: var(--text-secondary);
  cursor: pointer;
  transition: color var(--transition-fast);
}

#logout-btn:hover {
  color: var(--error);
}

#logout-btn:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}

/* Logout confirmation popover */
.logout-confirm {
  position: absolute;
  top: 100%;
  right: 0;
  margin-top: 0.5rem;
  padding: 0.75rem;
  background: var(--bg-secondary);
  border: 1px solid rgba(239, 68, 68, 0.3);
  border-radius: var(--border-radius);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
  z-index: 100;
  animation: fade-in 0.15s ease-out;
}

.logout-confirm-text {
  font-size: 0.8125rem;
  color: var(--text-primary);
  margin-bottom: 0.75rem;
}

.logout-confirm-actions {
  display: flex;
  gap: 0.5rem;
}

.logout-confirm-btn {
  padding: 0.375rem 0.75rem;
  font-size: 0.75rem;
  border-radius: 4px;
  cursor: pointer;
  transition: all var(--transition-fast);
}

.logout-confirm-btn.cancel {
  background: transparent;
  border: 1px solid var(--text-secondary);
  color: var(--text-secondary);
}

.logout-confirm-btn.confirm {
  background: var(--error);
  border: 1px solid var(--error);
  color: white;
}

/* Mobile: hide username, show icon only */
@media (max-width: 600px) {
  #current-user { display: none; }
  #user-indicator {
    padding: 0.25rem;
    border-radius: 50%;
  }
}
```

```javascript
// Logout with confirmation
document.getElementById('logout-btn').addEventListener('click', (e) => {
  const existing = document.querySelector('.logout-confirm');
  if (existing) {
    existing.remove();
    return;
  }

  const confirm = document.createElement('div');
  confirm.className = 'logout-confirm';
  confirm.innerHTML = `
    <div class="logout-confirm-text">Sign out?</div>
    <div class="logout-confirm-actions">
      <button class="logout-confirm-btn cancel">Cancel</button>
      <button class="logout-confirm-btn confirm">Sign Out</button>
    </div>
  `;

  confirm.querySelector('.cancel').onclick = () => confirm.remove();
  confirm.querySelector('.confirm').onclick = () => {
    ws.send(JSON.stringify({ type: 'logout' }));
    window.location.href = '/login';
  };

  document.getElementById('user-indicator').appendChild(confirm);

  // Close on outside click
  setTimeout(() => {
    document.addEventListener('click', function handler(e) {
      if (!confirm.contains(e.target) && e.target !== document.getElementById('logout-btn')) {
        confirm.remove();
        document.removeEventListener('click', handler);
      }
    });
  }, 0);
});
```

---

### 6. Permission-Based Visibility

**CSS Classes:**
```css
/* Hide from non-admin users */
.admin-only {
  display: none;
}

body.user-is-admin .admin-only {
  display: initial;
}

/* Blur sensitive info for non-admin */
body:not(.user-is-admin) .admin-sensitive {
  filter: blur(4px);
  pointer-events: none;
  user-select: none;
}

/* Hide paths completely for non-admin */
body:not(.user-is-admin) .config-path,
body:not(.user-is-admin) .secrets-path {
  display: none;
}
```

**JavaScript Integration:**
```javascript
function updateAuthUI(user) {
  if (user.is_admin) {
    document.body.classList.add('user-is-admin');
  } else {
    document.body.classList.remove('user-is-admin');
  }

  document.getElementById('current-user').textContent = user.username;
  if (user.is_admin) {
    document.getElementById('current-user').classList.add('is-admin');
  }
}
```

---

### 7. Mobile/Responsive Considerations

**Login Page:**
```css
@media (max-width: 400px) {
  .login-page { padding: 1rem; }
  .login-card { padding: 1.5rem; }
  .login-logo { width: 60px; height: 60px; }
  .login-title { font-size: 1.25rem; }
}
```

**Setup Wizard:**
- Single column maintained
- Password requirements always visible
- Touch-friendly button sizes (min 44px)

**User Management:**
- User items stack vertically
- Action buttons wrap on small screens
- Add User modal becomes full-screen

---

### 8. Accessibility (a11y)

**Focus Management:**
```css
/* Keyboard focus indicators for all interactive elements */
.login-field input:focus-visible,
.token-segment:focus-visible,
.setup-submit:focus-visible,
.login-submit:focus-visible,
.user-action-btn:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}

/* Skip link for keyboard users */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: var(--accent);
  color: var(--bg-primary);
  padding: 0.5rem 1rem;
  z-index: 1000;
  transition: top 0.2s;
}

.skip-link:focus {
  top: 0;
}
```

**ARIA Attributes:**
```html
<!-- Error messages announced to screen readers -->
<div class="login-error" role="alert" aria-live="assertive">
  Invalid username or password
</div>

<!-- Token expiry countdown (polite announcements) -->
<div class="token-expiry" aria-live="polite" aria-atomic="true">
  Token expires in <span id="expiry-countdown">04:32</span>
</div>

<!-- Form labels properly associated -->
<label for="username">Username</label>
<input type="text" id="username" name="username" autocomplete="username"
       aria-describedby="username-hint" required>
<span id="username-hint" class="field-hint">3-32 characters, letters and numbers</span>

<!-- Password requirements linked to input -->
<input type="password" id="password" name="password" autocomplete="new-password"
       aria-describedby="password-requirements" required>
<ul id="password-requirements" class="password-requirements">
  <li class="requirement" aria-live="polite">At least 8 characters</li>
  <li class="requirement" aria-live="polite">Contains number or symbol</li>
</ul>

<!-- Loading states -->
<button class="setup-submit" aria-busy="true" aria-disabled="true">
  <span class="sr-only">Verifying token...</span>
</button>
```

**Screen Reader Only Text:**
```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

**Color Contrast:**
All auth UI colors meet WCAG AA standards (4.5:1 for text):
- Text primary (#e0e0e0) on bg-primary (#121417): 11.5:1 ✓
- Accent (#2dd4bf) on bg-primary (#121417): 8.3:1 ✓
- Error (#ef4444) on bg-primary (#121417): 5.2:1 ✓
- Warning (#f0b429) on bg-primary (#121417): 7.8:1 ✓

**Keyboard Navigation Order:**
1. Skip link (first focusable element)
2. Form fields in logical order
3. Submit button
4. Error messages (announced automatically via aria-live)

---

## DAP Protocol Authentication

> **Note**: This section describes the original DAP1 binary protocol authentication design, which was **never implemented**. DAP1 has been removed entirely. DAP2 satellites authenticate via a pre-shared registration key sent in the `satellite_register` WebSocket message and validated in `webui_satellite.c`. The binary packet types below are obsolete.

ESP32 and other DAP clients authenticate using device tokens.

### Protocol Extension (OBSOLETE — DAP1 removed)

Add a new packet type for authenticated handshake (maintains backwards compatibility):

```c
// In dawn_server.h - add new packet type
#define PACKET_TYPE_AUTH_HANDSHAKE 0x07  // Authenticated handshake

// Existing PACKET_TYPE_HANDSHAKE (0x01) remains for backwards compatibility
// during migration period
```

### Authenticated Handshake Flow

```
Client -> Server: [Header][PACKET_TYPE_AUTH_HANDSHAKE][Device Token: 32 bytes]
Server -> Client: [ACK] or [NACK + Error Code]
```

**Error codes**:
- `0x01`: Invalid token
- `0x02`: Token revoked
- `0x03`: Device locked out

### Recommendation

Use **Device Token authentication** for DAP clients:
- Simpler for embedded devices (no password handling)
- Token can be revoked without affecting user passwords
- Device accounts are clearly marked in user list
- Token rotation via WebUI without device reflash

### DAP Token Security

**Important**: Device tokens are transmitted in cleartext over the network. Mitigations:

1. **TLS Wrapper**: Recommend TLS for DAP connections on untrusted networks
2. **Token Rotation**: Tokens should be rotatable via WebUI without device reflash
3. **Network Segmentation**: IoT devices on isolated VLAN
4. **Token Entropy**: 256-bit random tokens (same as session tokens)

```c
// Efficient token generation - no malloc, lookup table for hex
static const char s_hex_chars[] = "0123456789abcdef";

int auth_generate_token(char token_out[AUTH_TOKEN_LEN]) {
   uint8_t random_bytes[32];
   if (getrandom(random_bytes, sizeof(random_bytes), 0) != sizeof(random_bytes)) {
      LOG_ERROR("FATAL: getrandom() failed - cannot generate secure token");
      return FAILURE;  // DO NOT fall back to rand()
   }
   for (int i = 0; i < 32; i++) {
      token_out[i * 2]     = s_hex_chars[random_bytes[i] >> 4];
      token_out[i * 2 + 1] = s_hex_chars[random_bytes[i] & 0x0F];
   }
   token_out[64] = '\0';
   return SUCCESS;
}
```

**Token Storage on ESP32**: Store in NVS (non-volatile storage) partition with flash encryption enabled. Document that tokens should be treated as revocable on physical access concern.

---

## Configuration

### Auth Settings (`dawn.toml`)

```toml
[auth]
# Require login for WebUI (false = localhost bypass)
require_login = true

# Session timeout in minutes (0 = no timeout)
session_timeout_minutes = 1440  # 24 hours

# Max concurrent sessions per user
max_sessions_per_user = 5

# Allow self-registration (not recommended)
allow_registration = false

# Lockout after failed attempts
max_login_attempts = 5
lockout_duration_minutes = 15

# IP-based rate limiting (prevents distributed attacks)
max_attempts_per_ip = 20
rate_limit_window_minutes = 15

# Password requirements
min_password_length = 8
check_common_passwords = true

# First-run setup timeout (0 = no timeout, minutes)
setup_timeout_minutes = 5

# Session binding (optional hardening)
bind_session_to_ip = false    # May break roaming users
bind_session_to_user_agent = true
```

### C Configuration Structure

Add to `dawn_config.h`:

```c
typedef struct {
   bool require_login;
   int session_timeout_minutes;
   int max_sessions_per_user;
   bool allow_registration;
   int max_login_attempts;
   int lockout_duration_minutes;
   int max_attempts_per_ip;
   int rate_limit_window_minutes;
   int min_password_length;
   bool check_common_passwords;
   int setup_timeout_minutes;
   bool bind_session_to_ip;
   bool bind_session_to_user_agent;
} auth_config_t;

// Add to dawn_config_t:
typedef struct {
   // ... existing fields ...
   auth_config_t auth;
} dawn_config_t;
```

### Security Settings (`dawn.toml`)

```toml
[webui]
# Localhost-only mode bypasses login requirement
bind_address = "127.0.0.1"

# LAN access requires HTTPS + login
# bind_address = "0.0.0.0"
# https = true
```

### HTTPS Enforcement

Authentication is pointless without transport security. Enforce at startup:

```c
int auth_init(void) {
   // CRITICAL: If not localhost-only, HTTPS must be enabled
   if (!is_localhost_only() && !g_config.webui.https) {
      LOG_ERROR("SECURITY: Cannot enable authentication without HTTPS");
      LOG_ERROR("Either set webui.bind_address = \"127.0.0.1\" or enable webui.https");
      return FAILURE;
   }
   return auth_db_init(g_config.auth_db_path);
}
```

---

## Migration Path

### Phase 0: Build System Prep (1-2 days)

**Purpose**: Prepare the build system for auth implementation. No auth logic yet.

#### 0.1: Add ENABLE_DAP Compile Toggle
- Add `option(ENABLE_DAP ...)` to CMakeLists.txt
- Conditional compilation of `src/network/*.c`
- Add `#ifdef ENABLE_DAP` guards to dawn.c
- Derive `ENABLE_AUTH` from `ENABLE_DAP || ENABLE_WEBUI`
- Test all 4 build modes compile correctly

#### 0.2: Add CMake Presets
- Create `CMakePresets.json` with mode1-mode4 presets
- Document build modes in README.md

#### 0.3: Admin Socket Infrastructure

**New Files:**
```
include/auth/admin_socket.h    # Protocol definitions and API
src/auth/admin_socket.c        # Unix socket listener implementation
```

**Socket Configuration:**
- Abstract socket namespace: `\0dawn-admin` (Linux-specific, no filesystem cleanup)
- Fallback for non-Linux: `/run/dawn/admin.sock` with proper permissions
- Single connection limit (`ADMIN_MAX_CONNECTIONS = 1`)
- 30-second connection timeout

**Protocol Definition:**
```c
// Socket identification
#define ADMIN_SOCKET_ABSTRACT "\0dawn-admin"
#define ADMIN_SOCKET_PATH "/run/dawn/admin.sock"  // Fallback

// Protocol version
#define ADMIN_PROTOCOL_VERSION 0x01

// Message types (Phase 0 only)
#define ADMIN_MSG_PING                  0x01
#define ADMIN_MSG_VALIDATE_SETUP_TOKEN  0x02

// Response codes (generic to prevent info leakage)
#define ADMIN_RESP_SUCCESS              0x00
#define ADMIN_RESP_FAILURE              0x01  // Covers invalid/expired/used
#define ADMIN_RESP_RATE_LIMITED         0x02
#define ADMIN_RESP_SERVICE_ERROR        0x03
#define ADMIN_RESP_VERSION_MISMATCH     0x04

// Wire format
#define ADMIN_MSG_HEADER_SIZE 4
#define ADMIN_MSG_MAX_PAYLOAD 256  // Setup token is only 24 bytes

typedef struct __attribute__((packed)) {
    uint8_t version;       // Protocol version (0x01)
    uint8_t msg_type;      // Message type
    uint16_t payload_len;  // Little-endian, max 256
} admin_msg_header_t;

typedef struct __attribute__((packed)) {
    uint8_t version;       // Echo back protocol version
    uint8_t response_code; // Response status
    uint16_t reserved;     // Future use
} admin_msg_response_t;
```

**Setup Token:**
- Format: `DAWN-XXXX-XXXX-XXXX-XXXX` (16 random chars)
- Character set: `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (32 chars, no ambiguous I/O/1/0)
- Entropy: 80 bits (16 chars × 5 bits each)
- Generated via `getrandom(GRND_RANDOM)` - **no fallback, fail closed**
- Valid for 5 minutes from daemon startup
- Max 5 failed attempts before lockout
- Printed to stderr only (never logged to files)
- Rate limit state persisted to `/run/dawn/token_lockout.state`

**Security Requirements (from security audit):**
- **Peer credential validation**: Use `SO_PEERCRED` to verify UID (root or dawn user only)
- **Constant-time comparison**: Prevent timing attacks on token validation
- **Socket permissions**: Set `umask(0177)` before bind, verify 0600 after
- **Length validation**: Reject payloads > `ADMIN_MSG_MAX_PAYLOAD`
- **Entropy failure**: If `getrandom()` fails, abort startup (don't fall back to weak PRNG)

**Integration Points:**
```c
// dawn.c initialization (BEFORE main loop, after WebUI init)
#ifdef ENABLE_AUTH
   if (admin_socket_init() != 0) {
      LOG_WARNING("Failed to initialize admin socket");
      // Graceful degradation - don't prevent startup
   }
#endif

// dawn.c shutdown (BEFORE accept_thread_stop - critical ordering!)
#ifdef ENABLE_AUTH
   admin_socket_shutdown();  // Disconnect admin clients first
#endif

#ifdef ENABLE_DAP
   accept_thread_stop();     // Then DAP clients
   dawn_network_audio_cleanup();
#endif
```

**Thread Lifecycle:**
- Self-pipe trick for shutdown (pattern from `accept_thread.c`)
- Listener thread with single-client handling
- Clean shutdown: write to self-pipe, join thread, cleanup socket

#### 0.4: Minimal dawn-admin Bootstrap CLI

**New Files:**
```
dawn-admin/
    CMakeLists.txt             # Separate executable target
    main.c                     # Entry point, command dispatcher
    socket_client.c            # Unix socket client
    socket_client.h
    password_prompt.c          # Secure password input
    password_prompt.h
```

**CMake Integration:**
```cmake
# Main CMakeLists.txt additions
if(ENABLE_AUTH)
    list(APPEND DAWN_SOURCES src/auth/admin_socket.c)
    add_subdirectory(dawn-admin)
endif()

# dawn-admin/CMakeLists.txt
add_executable(dawn-admin main.c socket_client.c password_prompt.c)
target_include_directories(dawn-admin PRIVATE ${CMAKE_SOURCE_DIR}/include)
target_link_libraries(dawn-admin pthread)
install(TARGETS dawn-admin DESTINATION sbin)  # /usr/local/sbin
```

**CLI Usage (Phase 0.4 only):**
```bash
# Only command implemented in Phase 0.4:
dawn-admin user create <username> --admin

# Flow:
# 1. Connect to abstract socket \0dawn-admin
# 2. Prompt for setup token (from daemon stderr output)
# 3. Validate token via ADMIN_MSG_VALIDATE_SETUP_TOKEN
# 4. Prompt for password (no echo, with confirmation)
# 5. Print success (actual user creation is Phase 1)

# Environment variable for automation:
DAWN_SETUP_TOKEN=DAWN-XXXX-XXXX-XXXX-XXXX dawn-admin user create admin --admin
```

**Password Prompt Security (from security audit):**
```c
// Signal-safe terminal handling
static struct termios s_saved_term;
static volatile sig_atomic_t s_term_modified = 0;

static void signal_handler(int sig) {
    if (s_term_modified) {
        tcsetattr(STDIN_FILENO, TCSANOW, &s_saved_term);
        s_term_modified = 0;
    }
    signal(sig, SIG_DFL);
    raise(sig);
}

// Install SIGINT/SIGTERM handlers before modifying terminal
// Always restore terminal state, even on error paths
// Use explicit_bzero() to clear password buffers after use
```

**Deliverables:**
- All 4 deployment modes build correctly
- `dawn-admin user create --admin` connects to daemon, validates setup token
- Admin socket accepts connections, validates tokens, returns success/failure
- Smoke test passes: `./scripts/smoke_test.sh`

**What Phase 0 does NOT include:**
- Auth database schema (Phase 1)
- Password hashing with Argon2id (Phase 1)
- Session management (Phase 1)
- Full dawn-admin CLI (list, delete, device commands - Phase 2)

---

### Phase 0.3/0.4 Security Review Summary

The following issues were identified during security audit and must be addressed:

| Priority | Issue | Mitigation |
|----------|-------|------------|
| HIGH | Wire protocol buffer overflow | Max payload 256 bytes, validate before read |
| HIGH | Token entropy fallback | Never fall back from `getrandom()`, fail closed |
| HIGH | Socket permission TOCTOU | Set `umask(0177)` before `bind()`, verify after |
| HIGH | Missing peer credential check | `SO_PEERCRED` to validate connecting UID |
| MEDIUM | Token timing attack | Constant-time comparison with volatile pointers |
| MEDIUM | Terminal state leak on Ctrl+C | Signal handlers restore terminal before exit |
| MEDIUM | Response code info leakage | Generic `ADMIN_RESP_FAILURE` for all failures |
| MEDIUM | Lockout bypass via restart | Persist rate limit to `/run/dawn/token_lockout.state` |
| LOW | Protocol version handling | Check version, return `ADMIN_RESP_VERSION_MISMATCH` |

### Phase 0.3/0.4 Architecture Review Summary

| Priority | Issue | Resolution |
|----------|-------|------------|
| CRITICAL | Socket directory creation | Use abstract socket `\0dawn-admin` (no filesystem) |
| CRITICAL | Shutdown ordering | `admin_socket_shutdown()` BEFORE `accept_thread_stop()` |
| IMPORTANT | CMake missing sources | Add `src/auth/admin_socket.c` to `DAWN_SOURCES` |
| IMPORTANT | Connection limits | `ADMIN_MAX_CONNECTIONS=1`, 30s timeout |
| SUGGESTED | Status metrics | Add `admin_socket_is_running()` for debugging |
| SUGGESTED | Non-interactive token | Support `DAWN_SETUP_TOKEN` env var |

---

### Phase 1 Implementation Plan

This section details the implementation approach for Phase 1 authentication.

#### Dependencies to Add

```cmake
# CMakeLists.txt additions (after existing pkg_check_modules)
if(ENABLE_AUTH)
    pkg_check_modules(SODIUM REQUIRED libsodium)
    include_directories(${SODIUM_INCLUDE_DIRS})
    pkg_check_modules(SQLITE3 REQUIRED sqlite3)
    include_directories(${SQLITE3_INCLUDE_DIRS})
endif()

# Link to dawn target
if(ENABLE_AUTH)
    target_link_libraries(dawn ${SODIUM_LIBRARIES} ${SQLITE3_LIBRARIES})
endif()

# Link to dawn-admin (for password hashing)
target_link_libraries(dawn-admin ${SODIUM_LIBRARIES})
```

#### New Files

```
include/auth/auth_db.h         # Database API and data structures
src/auth/auth_db.c             # SQLite operations, prepared statements
include/auth/auth_crypto.h     # Password hashing, token generation
src/auth/auth_crypto.c         # Argon2id, secure random, constant-time compare
```

#### Database Schema

```sql
-- /var/lib/dawn/auth.db (or configurable path)

CREATE TABLE schema_version (version INTEGER PRIMARY KEY);

CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    is_admin INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,
    last_login INTEGER,
    failed_attempts INTEGER DEFAULT 0,
    lockout_until INTEGER DEFAULT 0
);

CREATE TABLE sessions (
    token TEXT PRIMARY KEY,
    user_id INTEGER NOT NULL,
    created_at INTEGER NOT NULL,
    last_activity INTEGER NOT NULL,
    ip_address TEXT,
    user_agent TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE login_attempts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ip_address TEXT NOT NULL,
    username TEXT,
    timestamp INTEGER NOT NULL,
    success INTEGER DEFAULT 0
);

CREATE TABLE auth_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp INTEGER NOT NULL,
    event TEXT NOT NULL,
    username TEXT,
    ip_address TEXT,
    details TEXT
);
```

#### Core API

**auth_crypto.h:**
```c
#define AUTH_HASH_LEN 128
#define AUTH_TOKEN_LEN 65  // 64 hex + null

int auth_crypto_init(void);  // Initialize libsodium
void auth_crypto_shutdown(void);

// Password hashing (Argon2id, semaphore-limited)
int auth_hash_password(const char *password, char hash_out[AUTH_HASH_LEN]);
int auth_verify_password(const char *hash, const char *password);

// Token generation (getrandom, no fallback)
int auth_generate_token(char token_out[AUTH_TOKEN_LEN]);

// Constant-time comparison
bool auth_token_compare(const char *a, const char *b);
```

**auth_db.h:**
```c
typedef struct {
    int id;
    char username[64];
    char password_hash[AUTH_HASH_LEN];
    bool is_admin;
    time_t created_at;
    time_t last_login;
    int failed_attempts;
    time_t lockout_until;
} auth_user_t;

typedef struct {
    char token[AUTH_TOKEN_LEN];
    int user_id;
    char username[64];
    bool is_admin;
    time_t created_at;
    time_t last_activity;
} auth_session_t;

// Lifecycle
int auth_db_init(const char *db_path);
void auth_db_shutdown(void);

// User operations
int auth_db_create_user(const char *username, const char *password_hash, bool is_admin);
int auth_db_get_user(const char *username, auth_user_t *user_out);
int auth_db_user_count(void);
int auth_db_increment_failed_attempts(const char *username);
int auth_db_reset_failed_attempts(const char *username);
int auth_db_update_last_login(const char *username);

// Session operations
int auth_db_create_session(int user_id, const char *token, const char *ip, const char *user_agent);
int auth_db_get_session(const char *token, auth_session_t *session_out);
int auth_db_update_session_activity(const char *token);
int auth_db_delete_session(const char *token);
int auth_db_delete_expired_sessions(time_t cutoff);

// Rate limiting
int auth_db_count_recent_failures(const char *ip, time_t since);
int auth_db_log_attempt(const char *ip, const char *username, bool success);

// Audit
void auth_db_log_event(const char *event, const char *username, const char *ip, const char *details);
```

#### Integration Points

**1. dawn.c Initialization (after admin_socket_init):**
```c
#ifdef ENABLE_AUTH
   if (auth_crypto_init() != 0) {
      LOG_ERROR("Failed to initialize crypto - auth disabled");
   } else if (auth_db_init(g_config.auth.db_path) != 0) {
      LOG_ERROR("Failed to initialize auth database");
   }
#endif
```

**2. dawn.c Shutdown (before admin_socket_shutdown):**
```c
#ifdef ENABLE_AUTH
   auth_db_shutdown();
   auth_crypto_shutdown();
#endif
```

**3. admin_socket.c - Handle ADMIN_MSG_CREATE_USER:**
```c
// New message type
ADMIN_MSG_CREATE_USER = 0x10

// Handler validates setup token first (existing), then:
// 1. Hash password with auth_hash_password()
// 2. Create user with auth_db_create_user()
// 3. Log event with auth_db_log_event()
// 4. Clear setup token (no longer needed)
```

**4. dawn-admin/main.c - Complete user create flow:**
```c
// After token validation succeeds:
// 1. Send ADMIN_MSG_CREATE_USER with username + password
// 2. Daemon hashes and stores
// 3. Return success/failure
```

#### Implementation Order

1. **Add dependencies to CMakeLists.txt** - sqlite3, libsodium
2. **Create auth_crypto.c/h** - Argon2id hashing, token generation
3. **Create auth_db.c/h** - SQLite schema, prepared statements, CRUD
4. **Update admin_socket.c** - Add ADMIN_MSG_CREATE_USER handler
5. **Update dawn-admin** - Send create user message after token validation
6. **Update dawn.c** - Initialize auth subsystem
7. **Test** - Verify user creation persists to database

#### Phase 1 Scope Boundaries

**IN SCOPE:**
- SQLite database with users table
- Argon2id password hashing
- User creation via dawn-admin CLI
- Setup token → user creation flow complete
- Audit logging of user creation

**DEFERRED TO LATER:**
- WebUI login page (Phase 1b or separate task)
- Session cookie handling
- CSRF protection
- Rate limiting enforcement
- Device tokens (Phase 2)
- Full dawn-admin CLI commands (Phase 2)

#### Files to Modify

- `CMakeLists.txt` - Add sqlite3, libsodium
- `include/auth/admin_socket.h` - Add ADMIN_MSG_CREATE_USER
- `src/auth/admin_socket.c` - Handle user creation
- `dawn-admin/main.c` - Send user creation request
- `dawn-admin/socket_client.c` - Add create_user function
- `src/dawn.c` - Initialize auth_db and auth_crypto

#### Testing

```bash
# 1. Build with new dependencies
cmake --preset mode2-dap && make -C build-mode2 -j$(nproc)

# 2. Start daemon (shows setup token)
LD_LIBRARY_PATH=/usr/local/lib ./build-mode2/dawn

# 3. Create admin user
./build-mode2/dawn-admin/dawn-admin user create admin --admin

# 4. Verify database
sqlite3 /var/lib/dawn/auth.db "SELECT * FROM users;"

# 5. Restart daemon - should NOT show setup token (admin exists)
```

#### Phase 1 Security Review Summary

| Priority | Issue | Mitigation |
|----------|-------|------------|
| HIGH | Missing CREATE_USER payload format definition | Define explicit wire format with length prefixes (uint16_t username_len, uint16_t password_len) and bounds checking |
| HIGH | Password transmitted in cleartext over Unix socket | Acceptable for threat model (local-only, SO_PEERCRED validated); document assumption explicitly |
| MEDIUM | TOCTOU in token validation + user creation flow | Combine into single atomic `ADMIN_MSG_VALIDATE_AND_CREATE_USER` message |
| MEDIUM | Database file permissions not specified | Add explicit 0600/0700 permission checks in `auth_db_init()`, verify WAL files |
| MEDIUM | Argon2id parameters need OWASP compliance doc | Add compile-time `_Static_assert` for minimum 15MB memory, 2 iterations |
| MEDIUM | SQL injection verification needed | Add code comments prohibiting raw SQL; verify all paths use prepared statements |
| LOW | Audit log missing fields (source, result, peer_uid) | Extend schema with source, result, request_id columns |
| LOW | Username enumeration via timing | Normalize response timing for all error paths |
| LOW | Semaphore DoS potential | Use `sem_timedwait()` with 5-second timeout instead of blocking `sem_wait()` |
| INFO | Password memory handling | Add `sodium_memzero()` calls after password/hash use |

#### Phase 1 Architecture Review Summary

| Priority | Issue | Resolution |
|----------|-------|------------|
| CRITICAL | Thread safety: Prepared statements cannot be shared across threads | Use `SQLITE_OPEN_FULLMUTEX` and ensure `sqlite3_reset()` after each use; document that `s_db_mutex` serializes all auth DB access |
| CRITICAL | Shutdown order inverted in design doc | Correct order: `admin_socket_shutdown()` → `auth_db_shutdown()` → `auth_crypto_shutdown()` |
| CRITICAL | Initialization race: admin_socket starts before auth_db ready | Initialize `auth_crypto` and `auth_db` BEFORE `admin_socket_init()` |
| IMPORTANT | CMake linking incomplete: libsodium/sqlite3 not added | Add `pkg_check_modules` and `target_link_libraries` inside `ENABLE_AUTH` block |
| IMPORTANT | `sqlite3_prepare_v2` return values not checked | Add error checking for all 9 prepare calls; return FAILURE on any error |
| IMPORTANT | Semaphore init missing from `auth_crypto_init` | Add `sem_init()` call in `auth_crypto_init()` |
| SUGGESTED | WAL checkpoint not shown | Add periodic `sqlite3_wal_checkpoint_v2()` call during cleanup |
| SUGGESTED | Lock ordering with session_manager not documented | Add `s_db_mutex` as Level 5 leaf lock in documentation |

#### Phase 1 Efficiency Review Summary

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| HIGH | Argon2id 16MB×3 concurrent = 48MB on Pi problematic | Platform-aware params: 8MB/4 iterations for Pi, 16MB/3 iterations for Jetson |
| HIGH | Dedicated cleanup thread wastes stack memory (8KB-1MB) | Use lazy cleanup during `auth_db_get_session()` every N calls; no separate thread |
| MEDIUM | 8 prepared statements (~32KB) kept open | Keep as-is: 10x query speedup worth the memory cost |
| MEDIUM | WAL mode -wal file can grow unbounded | Add `PRAGMA wal_checkpoint(TRUNCATE)` during cleanup to reclaim space |
| LOW | Single connection with mutex vs pool | Correct choice for workload; pool adds ~100KB per connection |
| LOW | `user_agent` TEXT implies dynamic strings | Truncate to fixed 128-byte buffer or omit entirely |

#### Phase 1 Key Action Items

Based on the reviews, the following must be addressed during implementation:

1. **Fix initialization order**: `auth_crypto_init()` and `auth_db_init()` MUST complete BEFORE `admin_socket_init()` starts its listener thread
2. **Fix shutdown order**: `admin_socket_shutdown()` → `auth_db_shutdown()` → `auth_crypto_shutdown()`
3. **Define explicit wire format**: CREATE_USER payload must have length-prefixed strings with bounds checking
4. **Thread safety**: Use `SQLITE_OPEN_FULLMUTEX`, call `sqlite3_reset()` after each statement use
5. **Platform-aware Argon2id**: Detect platform at init, use 8MB for Pi, 16MB for Jetson
6. **Lazy cleanup**: Run expired session cleanup during `auth_db_get_session()` instead of dedicated thread
7. **Check all sqlite3_prepare_v2 returns**: Fail initialization if any prepare fails
8. **Initialize semaphore**: Call `sem_init()` in `auth_crypto_init()`

---

### Phase 1: Foundation (5-6 days)

**Purpose**: Core authentication system for WebUI (Mode 3).

- SQLite database schema with prepared statement pool
- User CRUD with Argon2id password hashing
- Session management with constant-time token validation
- **Audit logging from day one** (login_attempts + auth_log tables)
- IP-based rate limiting (with IPv6 /64 prefix normalization)
- Setup token generation and validation (completes Phase 0.3 stub)
- WebUI login page with CSRF protection
- First-run setup wizard (token-based, headless-friendly)
- HTTPS enforcement check at startup
- Protected settings endpoints
- Startup security validation

**Deliverables:**
- Mode 3 (WebUI) fully functional with authentication
- `dawn-admin user create --admin` now creates real database entries
- Setup wizard works for both WebUI and CLI bootstrap

---

### Phase 2: Complete CLI Administration (2-3 days)

**Status**: Implemented (2026-01-07)

**Purpose**: Full CLI tool and background maintenance tasks.

#### Complete dawn-admin CLI
- [x] `dawn-admin user list|delete|passwd|unlock`
- [x] `dawn-admin log show|export`
- [x] `dawn-admin db status|backup|compact`
- [x] `dawn-admin ip list|unblock` (IP rate-limit management)
- [x] `dawn-admin metrics list|summary` (session metrics)
- [x] `dawn-admin conv list|show|delete` (conversation management)
- [x] `dawn-admin session list|revoke` (admin session management)

#### Background Maintenance
- [x] Session cleanup thread (expire old sessions)
- [x] Database maintenance (log rotation, vacuum scheduling)

**Deliverables:**
- Complete CLI tool for headless user administration
- Automated session cleanup
- Mode 3 (WebUI) production-ready

---

### Phase 3: Multi-User (3-4 days)

**Status**: Implemented (2026-01-05)

**Scope Decisions:**
- Per-user device access control: **Deferred to Phase 4**
- Personal settings: **Full UI + Backend**

#### Phase 3 Deliverables

1. **User Management UI** (admin-only) - List users, create/delete users, reset passwords, unlock accounts
2. **Permission Enforcement** - Add admin checks to set_config, set_secrets, restart, etc.
3. **Per-User Settings** - Database storage + WebUI for persona, locale, TTS preferences
4. **Admin Visibility Control** - Hide admin-only UI from regular users

#### Phase 3 Database Changes

**Schema v1 → v2 Migration:**
```sql
CREATE TABLE IF NOT EXISTS user_settings (
    user_id INTEGER PRIMARY KEY,
    persona_description TEXT,
    location TEXT,
    timezone TEXT DEFAULT 'UTC',
    units TEXT DEFAULT 'metric',
    tts_voice_model TEXT,
    tts_length_scale REAL DEFAULT 1.0,
    updated_at INTEGER NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

**New struct (reduced per efficiency review):**
```c
typedef struct {
   char persona_description[512];
   char location[128];
   char timezone[64];
   char units[16];
   char tts_voice_model[128];
   float tts_length_scale;
} auth_user_settings_t;  // ~850 bytes
```

#### Phase 3 WebSocket Messages

**User Management (Admin Only):**
- `list_users` / `list_users_response`
- `create_user` / `create_user_response`
- `delete_user` / `delete_user_response`
- `change_password` / `change_password_response`
- `unlock_user` / `unlock_user_response`

**Personal Settings (Authenticated Users):**
- `get_my_settings` / `get_my_settings_response`
- `set_my_settings` / `set_my_settings_response`

#### Phase 3 Security Requirements

| Requirement | Implementation |
|-------------|----------------|
| Admin check on set_config/set_secrets/restart | `conn_require_admin()` with DB re-validation |
| Cannot delete self | Check in `handle_delete_user()` |
| Password change IDOR prevention | Verify requester is target OR admin |
| Self-password change requires current | Non-admin must provide current password |
| Session invalidation on role change | Delete all sessions when promoted/demoted |
| XSS prevention in user list | Use `textContent`, not `innerHTML` |
| Confirmation for destructive actions | `confirm()` or modal before delete |

**Critical**: The `is_admin` flag is NOT cached in `ws_connection_t`. Each admin operation re-validates against the database using `auth_db_get_session()` to prevent privilege escalation if a user is demoted mid-session.

#### Phase 3 Files to Modify

| File | Changes |
|------|---------|
| `include/auth/auth_db.h` | Add `auth_user_settings_t`, new function declarations |
| `src/auth/auth_db.c` | Schema v2 migration, user_settings table, prepared statements |
| `src/webui/webui_server.c` | Auth helpers, permission checks, 7 new message handlers |
| `www/js/dawn.js` | Auth state, 7 message handlers, request functions, toast notifications |
| `www/index.html` | User Management section, My Settings section, Add User modal |
| `www/css/dawn.css` | Admin visibility, user list styling, modal styling (glassmorphism) |

#### Phase 3 Architecture Review Summary

| Priority | Issue | Resolution |
|----------|-------|------------|
| CRITICAL | Auth state cache can become stale if user demoted mid-session | Re-validate `is_admin` against DB for admin checks using `auth_db_get_session()` |
| IMPORTANT | Schema migration needs atomicity | Wrap in transaction with `table_exists()` check for idempotency |
| SUGGESTED | Self-deletion prevention | Add check: users cannot delete themselves |
| SUGGESTED | Message dispatch complexity | Consider table-driven dispatch pattern (optional refactor) |

#### Phase 3 Efficiency Review Summary

| Priority | Issue | Resolution |
|----------|-------|------------|
| MEDIUM | `auth_user_settings_t` is 1.6KB | Reduce to ~850 bytes: `persona_description[512]`, `location[128]`, `tts_voice_model[128]` |
| LOW | `auth_db_list_users` not prepared | Add `stmt_list_users` to prepared statements |
| SUGGESTED | Settings caching | Lazy-load with 60s TTL, shared cache by `user_id` |
| SUGGESTED | UPSERT pattern | Use `INSERT ... ON CONFLICT DO UPDATE` for `set_user_settings` |

#### Phase 3 Security Review Summary

| Severity | Issue | Resolution |
|----------|-------|------------|
| CRITICAL | No backend auth on WebSocket handlers | Add `conn_require_admin()` to ALL admin handlers - CSS hiding is cosmetic only |
| HIGH | IDOR in password change | Verify requester is target user OR is admin before allowing |
| MEDIUM | No current password for self-change | Require current password for non-admin self-service password changes |
| MEDIUM | XSS in user list | Use `textContent` not `innerHTML` when rendering usernames |
| MEDIUM | Stale sessions on role change | Invalidate all sessions when user promoted/demoted |

#### Phase 3 UI Design Review Summary

| Priority | Issue | Resolution |
|----------|-------|------------|
| HIGH | Modal styling mismatch | Use `rgba(18, 18, 26, 0.95)` + cyan border tint to match existing glassmorphism |
| HIGH | Confirmation dialogs | Required for delete; use `confirm()` or custom modal |
| MEDIUM | Action buttons | Icons with tooltips (32px), expand to 44px touch targets on mobile |
| MEDIUM | Feedback mechanism | Implement toast notifications for action results |
| SUGGESTED | Section position | Move User Management to top of settings panel |

---

### Phase 4: Enhancements (2-3 days)

**Status**: Mostly implemented (2026-01-07)

#### Completed
- [x] Per-user conversation history (WebUI sidebar, load/save, continuations)
- [x] Conversation database with user ownership enforcement
- [x] Context compaction creates continuation conversations
- [x] `dawn-admin conv` commands for CLI management
- [x] Session management UI (view/revoke active sessions)
- [x] `dawn-admin session list|revoke` commands

#### Remaining
- [ ] Per-user device access control (deferred from Phase 3)
- [ ] Optional 2FA (TOTP)

---

### Phase 5: DAP2 Device Authentication — PARTIALLY SUPERSEDED

**Purpose**: Device authentication for satellite clients.

**Note**: DAP1 has been removed. DAP2 satellite authentication is implemented via a pre-shared registration key (`satellite_registration_key` in `secrets.toml`), validated during the `satellite_register` WebSocket handshake in `webui_satellite.c`. The `dawn-admin` CLI tool (v2.0.0) provides full user, session, conversation, music DB, audit log, and IP management. The token rotation and device CRUD features below remain unimplemented but are lower priority given the registration key approach.

#### DAP2 Authentication
- Device token table and CRUD operations
- DAP2 authenticated handshake (protocol TBD in DAP2 design)
- Device token validation (constant-time)
- DAP2 connection rejection before tokens configured
- Token rotation with configurable grace period

#### dawn-admin Device Commands
- `dawn-admin device create|list|revoke|rotate`

#### WebUI Device Management (if ENABLE_WEBUI)
- Device token creation/revocation UI
- Device list with last-seen timestamps

**Deliverables:**
- Mode 2 (DAP-only, headless) fully functional
- Mode 4 (full) fully functional
- DAP2 protocol with built-in authentication

---

## Implementation Estimate

| Phase | Scope | Effort | Modes Enabled |
|-------|-------|--------|---------------|
| Phase 0 | Build system prep | 1-2 days | All modes build |
| Phase 1 | Foundation auth | 5-6 days | Mode 3 functional |
| Phase 2 | Complete CLI + maintenance | 2-3 days | Mode 3 production-ready |
| Phase 3 | Multi-user | 3-4 days | Enhanced UX |
| Phase 4 | Enhancements | 2-3 days | Polish |
| Phase 5 | DAP2 device auth | TBD | Mode 2, 4 functional |

**Total (Phases 0-4): ~13-18 days**
**Phase 5**: Partially superseded by registration key approach (see note above)

### Phase Dependencies

```
Phase 0 (Build Prep)
    │
    ├─── Mode 1 ready (no auth needed)
    │
    └─── Phase 1 (Foundation)
             │
             ├─── Mode 3 ready (WebUI auth)
             │
             └─── Phase 2 (CLI + Maintenance)
                      │
                      ├─── Mode 3 production-ready
                      │
                      └─── Phase 3 (Multi-User)
                               │
                               └─── Phase 4 (Enhancements)

Phase 5 (DAP2 Device Auth) ─── Partially superseded (registration key implemented)
    │
    └─── Token rotation / device CRUD remain unimplemented
```

### Critical Path for Mode 3 (WebUI)

Mode 3 (WebUI-only) requires:
1. ✅ Phase 0: Build system + minimal bootstrap CLI
2. ✅ Phase 1: Core auth database + WebUI login
3. Phase 2: Complete CLI + session cleanup

**Minimum viable Mode 3: Phase 0 + Phase 1** (completed)

### Critical Path for Mode 2/4 (DAP2 Satellites)

DAP2 satellite authentication is implemented via pre-shared registration key:
1. ✅ Phase 0: Build system
2. ✅ Phase 1: Core auth database + setup token
3. ✅ Registration key: `satellite_registration_key` in `secrets.toml`, validated in `handle_satellite_register()`
4. Future: Per-device token rotation and CRUD (Phase 5 remainder)

---

## Review Action Items

The following items were identified during security, architecture, efficiency, and UI design reviews. Items are categorized by priority.

### Must Fix Before Implementation

| # | Issue | Source | Status | Description |
|---|-------|--------|--------|-------------|
| 1 | Setup token for dawn-admin | Security | ✅ Done | First admin creation via `dawn-admin` requires setup token (proves daemon access) |
| 2 | DAP bootstrap rejection | Security | Deferred (Phase 5) | DAP server must explicitly reject connections before device tokens exist |
| 3 | Lock ordering clarification | Architecture | ✅ Done | Document that `s_db_mutex` is leaf-level; never hold `session->ref_mutex` while acquiring it |
| 4 | Phase 0 restructure | Architecture | ✅ Done | Split Phase 0.2: minimal bootstrap `dawn-admin user create --admin` first, full CLI later |

### Should Fix Before Production

| # | Issue | Source | Description |
|---|-------|--------|-------------|
| 5 | Session nonce for TOCTOU | Security | Add session-bound nonce to setup flow to prevent race during password hashing |
| 6 | IPv6 prefix rate limiting | Security | Rate limit by /64 prefix for IPv6, not exact address (prevents address rotation bypass) |
| 7 | Non-blocking hash semaphore | Efficiency | ✅ Done — `sem_timedwait()` with `AUTH_CRYPTO_BUSY` in `auth_crypto.c` |
| 8 | SQLite cache size | Efficiency | ✅ Done — `PRAGMA cache_size=64` in `auth_db_core.c` |
| 9 | Password visibility toggle | UI | ⚠️ Partial — Settings page API key toggles done; login form toggle not implemented |
| 10 | Reduced motion support | UI | ✅ Done — `@media (prefers-reduced-motion: reduce)` in plan-orchestrator, visualizer, transcript CSS |

### Implementation Notes

**Item 1, 4** - ✅ **ADDRESSED** in this document:
- Item 1: See "Mode 2 First-Run Bootstrap" and "Setup Token Communication" sections
- Item 4: See restructured "Migration Path" - Phase 0 now contains only prep work

**Item 2** - Deferred to Phase 5 (DAP2 device authentication)

**Item 3** - The lock ordering hierarchy should be:
```
Level 1: session_manager_rwlock
Level 2: session->ref_mutex
Level 3: session->fd_mutex
Level 4: session->llm_config_mutex OR session->history_mutex
Level 5: s_db_mutex, s_setup_mutex (LEAF LEVEL - never hold while acquiring others)
```

**Item 6** - IPv6 rate limiting implementation:
```c
// Normalize IPv6 to /64 prefix for rate limiting
void normalize_ip_for_rate_limit(const char *ip, char *out, size_t out_len) {
   struct in6_addr addr6;
   if (inet_pton(AF_INET6, ip, &addr6) == 1) {
      // Zero the interface identifier (lower 64 bits)
      memset(&addr6.s6_addr[8], 0, 8);
      inet_ntop(AF_INET6, &addr6, out, out_len);
   } else {
      // IPv4 - use as-is
      strncpy(out, ip, out_len - 1);
   }
}
```

**Item 7** - Non-blocking semaphore:
```c
int auth_hash_password(const char *password, char hash_out[crypto_pwhash_STRBYTES]) {
   if (sem_trywait(&s_hash_semaphore) != 0) {
      if (errno == EAGAIN) {
         return AUTH_BUSY;  // Caller should retry after delay
      }
      return AUTH_FAILURE;
   }
   int result = crypto_pwhash_str(hash_out, password, strlen(password),
                                   AUTH_OPSLIMIT, AUTH_MEMLIMIT);
   sem_post(&s_hash_semaphore);
   return result == 0 ? AUTH_SUCCESS : AUTH_FAILURE;
}
```

**Item 8** - SQLite cache size:
```c
// In auth_db_init()
sqlite3_exec(s_auth_db.db, "PRAGMA cache_size=128", NULL, NULL, NULL);  // 512KB
```

**Item 10** - Reduced motion CSS:
```css
@media (prefers-reduced-motion: reduce) {
   .login-card,
   .setup-step-content,
   .token-segment,
   .login-submit::after {
      animation: none;
      transition: none;
   }

   .login-card.error-shake {
      animation: none;
      outline: 2px solid var(--error);
   }
}
```

---

## Security Considerations

### CRITICAL: Constant-Time Token Comparison

**MANDATORY**: All token comparisons MUST use constant-time functions to prevent timing attacks.

```c
#include <sodium.h>

// CORRECT - constant-time comparison
bool auth_validate_token(const char *provided, const char *stored) {
   if (strlen(provided) != AUTH_TOKEN_LEN - 1) {
      return false;  // Length mismatch OK to fail fast
   }
   // sodium_memcmp returns 0 if equal, prevents timing attacks
   return sodium_memcmp(provided, stored, AUTH_TOKEN_LEN - 1) == 0;
}

// WRONG - timing attack vulnerable
bool bad_validate_token(const char *provided, const char *stored) {
   return strcmp(provided, stored) == 0;  // NEVER DO THIS
}
```

This applies to:
- Session token validation
- Device token validation (DAP)
- CSRF token validation
- TOTP code verification (Phase 4)

### CRITICAL: Password Verification Timing Normalization

**MANDATORY**: Normalize timing to prevent username enumeration attacks.

```c
// Pre-computed dummy hash for timing normalization
static const char *DUMMY_HASH = "$argon2id$v=19$m=16384,t=3,p=1$...";

int auth_verify_password(const char *username, const char *password,
                         auth_user_t *user_out) {
   auth_user_t user;
   bool user_exists = auth_db_get_user(username, &user) == SUCCESS;

   if (!user_exists) {
      // User doesn't exist - still verify against dummy to normalize timing
      crypto_pwhash_str_verify(DUMMY_HASH, password, strlen(password));
      return AUTH_INVALID_CREDENTIALS;  // Same error as wrong password
   }

   if (crypto_pwhash_str_verify(user.password_hash, password,
                                 strlen(password)) != 0) {
      return AUTH_INVALID_CREDENTIALS;  // Same error as invalid user
   }

   // Success - copy user data
   if (user_out) *user_out = user;
   return AUTH_SUCCESS;
}
```

### CRITICAL: Error Message Uniformity

**MANDATORY**: Never reveal whether username or password was wrong.

```c
// CORRECT - same message for both cases
return "Invalid username or password";

// WRONG - information disclosure
if (!user_exists) return "Username not found";      // Reveals valid usernames
if (!password_ok) return "Incorrect password";      // Confirms user exists
```

### Password Hashing

**Algorithm**: Argon2id via libsodium

**Parameters** (tuned for embedded Jetson):
```c
// 16MB memory - safe for concurrent logins on 8GB system
// 64MB would consume 576MB with 9 concurrent logins
#define AUTH_MEMLIMIT (16 * 1024 * 1024)  // 16MB
#define AUTH_OPSLIMIT 3                    // 3 iterations

// Limit concurrent password hashing to prevent memory exhaustion
// 3 concurrent * 16MB = 48MB max, vs 144MB without limit
#define AUTH_CONCURRENT_HASH_LIMIT 3
static sem_t s_hash_semaphore;

// Call during auth_init()
void auth_hash_init(void) {
   sem_init(&s_hash_semaphore, 0, AUTH_CONCURRENT_HASH_LIMIT);
}

// Wrapper for password hashing with concurrency control
int auth_hash_password(const char *password, char hash_out[crypto_pwhash_STRBYTES]) {
   sem_wait(&s_hash_semaphore);  // Block if too many concurrent hashes
   int result = crypto_pwhash_str(hash_out, password, strlen(password),
                                   AUTH_OPSLIMIT, AUTH_MEMLIMIT);
   sem_post(&s_hash_semaphore);
   return result == 0 ? AUTH_SUCCESS : AUTH_FAILURE;
}

// Wrapper for password verification with concurrency control
int auth_verify_password_hash(const char *hash, const char *password) {
   sem_wait(&s_hash_semaphore);
   int result = crypto_pwhash_str_verify(hash, password, strlen(password));
   sem_post(&s_hash_semaphore);
   return result == 0 ? AUTH_SUCCESS : AUTH_INVALID_CREDENTIALS;
}
```

**Simple usage** (uses wrappers above):
```c
char hash[crypto_pwhash_STRBYTES];
if (auth_hash_password(password, hash) != AUTH_SUCCESS) {
   return AUTH_FAILURE;  // Out of memory or semaphore error
}
```

**Hash Upgrade Path**: Check on each login if parameters have changed:
```c
// After successful login, upgrade hash if needed
if (crypto_pwhash_str_needs_rehash(user.password_hash,
      AUTH_OPSLIMIT, AUTH_MEMLIMIT) == 1) {
   char new_hash[crypto_pwhash_STRBYTES];
   crypto_pwhash_str(new_hash, password, strlen(password),
      AUTH_OPSLIMIT, AUTH_MEMLIMIT);
   auth_db_update_password_hash(username, new_hash);
   LOG_INFO("AUTH: Upgraded password hash for user '%s'", username);
}
```

### Session Tokens

- 256-bit random via `getrandom()` only - **NO fallback to rand()**
- HTTP-only, Secure cookies (when HTTPS)
- Server-side storage with expiration
- Optional IP/User-Agent binding for extra security

### Session Security

**Session Fixation Prevention**: Regenerate token after successful login:
```c
// After password verification succeeds:
auth_session_destroy(old_session_token);
char new_token[AUTH_TOKEN_LEN];
auth_session_create(user.user_id, ip_addr, user_agent, new_token);
// Return new_token in Set-Cookie
```

**Max Sessions Enforcement**:
```c
int auth_session_create_for_user(int user_id, char *token_out) {
   int count = auth_db_count_sessions(user_id);
   if (count >= g_config.auth.max_sessions_per_user) {
      // Evict oldest session (better UX than rejection)
      auth_db_delete_oldest_session(user_id);
   }
   return auth_session_create(user_id, token_out);
}
```

**Cookie Attributes**:
```c
void auth_set_session_cookie(struct lws *wsi, const char *token, bool is_https) {
   char cookie[512];
   if (is_https) {
      snprintf(cookie, sizeof(cookie),
         "dawn_session=%s; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=%d",
         token, g_config.auth.session_timeout_minutes * 60);
   } else {
      // HTTP only allowed for localhost binding
      snprintf(cookie, sizeof(cookie),
         "dawn_session=%s; HttpOnly; SameSite=Strict; Path=/; Max-Age=%d",
         token, g_config.auth.session_timeout_minutes * 60);
   }
   // Note: Do NOT set Domain= attribute - cookies default to exact hostname
   lws_add_http_header_by_name(wsi, "Set-Cookie", cookie, strlen(cookie), ...);
}
```

### Brute Force Protection

**Per-User Lockout**:
```c
if (user.failed_attempts >= g_config.auth.max_login_attempts) {
   if (time(NULL) < user.lockout_until) {
      auth_log_event("LOCKOUT_ACTIVE", username, ip, NULL);
      return AUTH_LOCKED_OUT;
   }
   // Lockout expired - reset counter
   auth_db_reset_failed_attempts(username);
}
```

**Per-IP Rate Limiting** (prevents distributed attacks):
```c
int recent_failures = auth_db_count_recent_failures(ip_addr,
   time(NULL) - g_config.auth.rate_limit_window_minutes * 60);

if (recent_failures >= g_config.auth.max_attempts_per_ip) {
   auth_log_event("IP_RATE_LIMITED", NULL, ip, NULL);
   return AUTH_RATE_LIMITED;
}
```

### Reverse Proxy Configuration

**IMPORTANT**: If DAWN is deployed behind a reverse proxy (nginx, HAProxy, Caddy, etc.), rate limiting will see all requests as coming from the proxy's IP address, not the real client.

**Current Limitation**: DAWN uses the direct TCP peer address for rate limiting. X-Forwarded-For header is NOT trusted by default.

**Recommended Deployment Options**:

1. **Direct binding** (preferred for home networks):
   - Bind DAWN directly to the network interface
   - Let DAWN handle TLS termination via its built-in SSL support
   - Configure firewall rules to restrict access

2. **Reverse proxy with trusted network**:
   - If using a reverse proxy on localhost (127.0.0.1), rate limiting still functions correctly for session management but per-IP limiting becomes ineffective
   - Accept this limitation for home deployments where the proxy handles TLS
   - Consider per-user lockout as the primary brute force protection

3. **Future Enhancement** (not yet implemented):
   ```toml
   # dawn.toml (proposed config)
   [webui.proxy]
   # Trust X-Forwarded-For from these networks only
   trusted_proxies = ["127.0.0.1/8", "::1/128"]
   # Use real_ip module behavior
   real_ip_header = "X-Forwarded-For"
   real_ip_recursive = true
   ```

**Nginx Example** (for future X-Forwarded-For support):
```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**Security Note**: Never trust X-Forwarded-For without validating the immediate peer is a trusted proxy address, as this header can be spoofed by malicious clients.

### CSRF Protection

For HTTP form-based login:
```html
<form method="POST" action="/login">
   <input type="hidden" name="csrf_token" value="{{token}}">
   <!-- username/password fields -->
</form>
```

**CSRF Token Requirements**:
1. Generated server-side for each login page render
2. Stored in session or signed with server secret
3. Validated before password check (fail fast)
4. Single-use (regenerated after each attempt)

### Secure Defaults

- Localhost binding = no login required (trusted)
- LAN binding = login + HTTPS required (enforced at startup)
- Admin account required before any access
- First-run setup: localhost only, with optional timeout

---

## Thread Safety

### Lock Ordering

The auth subsystem introduces new locks that must be ordered relative to existing session locks in `session_manager.h` to prevent deadlocks.

**Lock Hierarchy** (acquire in this order, release in reverse):
```
Level 0: (none - auth locks are leaf-level)
Level 1: session_manager_rwlock
Level 2: session->ref_mutex
Level 3: session->fd_mutex
Level 4: session->llm_config_mutex OR session->history_mutex
Level 5: s_db_mutex (auth database - LEAF LEVEL)
Level 5: s_setup_mutex (setup token - LEAF LEVEL, independent of s_db_mutex)
```

**Critical Rules**:
1. `s_db_mutex` and `s_setup_mutex` are leaf-level locks - never hold them while acquiring other locks
2. `s_db_mutex` and `s_setup_mutex` are independent - never hold both simultaneously
3. The `auth` pointer in `session_t` is a cached copy - no lock needed for reads

```c
// CORRECT - acquire db_mutex, do work, release
pthread_mutex_lock(&s_db_mutex);
// ... database operations only ...
pthread_mutex_unlock(&s_db_mutex);

// CORRECT - setup mutex is independent
pthread_mutex_lock(&s_setup_mutex);
// ... setup token operations only ...
pthread_mutex_unlock(&s_setup_mutex);

// WRONG - could deadlock
pthread_mutex_lock(&s_db_mutex);
session_manager_lock();  // NEVER DO THIS

// WRONG - violates independence
pthread_mutex_lock(&s_db_mutex);
pthread_mutex_lock(&s_setup_mutex);  // NEVER DO THIS
```

**auth_session_t Access Pattern**:
```c
// Permission checks use cached copy (no lock needed)
if (session->auth && session->auth->is_admin) {
   // Fast path - no database access required
}

// Session refresh (rare) updates the cached copy
void refresh_auth_session(session_t *session) {
   pthread_mutex_lock(&session->ref_mutex);  // session lock first
   pthread_mutex_lock(&s_db_mutex);           // then db lock
   // Refresh from database...
   pthread_mutex_unlock(&s_db_mutex);
   pthread_mutex_unlock(&session->ref_mutex);
}
```

### SQLite Concurrency Model

SQLite with WAL mode handles most concurrency automatically:

- **Multiple readers**: Concurrent reads are lock-free
- **Single writer**: Writes are serialized by SQLite
- **Read during write**: WAL mode allows reads to continue during writes

```c
// Initialize SQLite for multi-threaded use
sqlite3_config(SQLITE_CONFIG_SERIALIZED);  // Fully thread-safe mode
```

### Connection Management

For DAWN's expected load (8 ESP32 + WebUI), use shared connection with mutex:

```c
static auth_db_t s_auth_db;
static pthread_mutex_t s_db_mutex = PTHREAD_MUTEX_INITIALIZER;

// All database operations go through wrapper functions
int auth_db_lookup_session(const char *token, auth_session_t *session_out) {
   pthread_mutex_lock(&s_db_mutex);

   sqlite3_stmt *stmt = s_auth_db.stmt_lookup_session;
   sqlite3_bind_text(stmt, 1, token, -1, SQLITE_STATIC);

   int result = FAILURE;
   if (sqlite3_step(stmt) == SQLITE_ROW) {
      // Populate session_out from row...
      result = SUCCESS;
   }

   sqlite3_reset(stmt);
   pthread_mutex_unlock(&s_db_mutex);
   return result;
}
```

### Session Cleanup

Background thread expires old sessions, login attempts, and audit logs:

```c
#define SESSION_CLEANUP_INTERVAL_SEC 300  // Every 5 minutes (not 60s)

static void *session_cleanup_thread(void *arg) {
   while (!shutdown_requested) {
      sleep(SESSION_CLEANUP_INTERVAL_SEC);

      time_t now = time(NULL);

      pthread_mutex_lock(&s_db_mutex);

      // 1. Delete expired sessions
      sqlite3_bind_int64(s_auth_db.stmt_delete_expired_sessions,
         1, now - g_config.auth.session_timeout_minutes * 60);
      sqlite3_step(s_auth_db.stmt_delete_expired_sessions);
      sqlite3_reset(s_auth_db.stmt_delete_expired_sessions);

      // 2. Delete old login attempts (for rate limiting)
      sqlite3_bind_int64(s_auth_db.stmt_delete_old_attempts,
         1, now - g_config.auth.rate_limit_window_minutes * 60 * 2);
      sqlite3_step(s_auth_db.stmt_delete_old_attempts);
      sqlite3_reset(s_auth_db.stmt_delete_old_attempts);

      // 3. Delete old audit logs (30 day retention)
      sqlite3_bind_int64(s_auth_db.stmt_delete_old_logs,
         1, now - AUTH_LOG_RETENTION_DAYS * 24 * 60 * 60);
      sqlite3_step(s_auth_db.stmt_delete_old_logs);
      sqlite3_reset(s_auth_db.stmt_delete_old_logs);

      pthread_mutex_unlock(&s_db_mutex);

      LOG_DEBUG("AUTH: Cleanup completed");
   }
   return NULL;
}
```

### In-Memory Session Cache (Optional)

For hot-path performance, cache active sessions in memory:

```c
// LRU cache of recently validated sessions
// Falls back to SQLite on cache miss
static session_cache_t session_cache;
static pthread_rwlock_t cache_lock = PTHREAD_RWLOCK_INITIALIZER;

session_t *session_lookup(const char *token) {
   // Try cache first (read lock)
   pthread_rwlock_rdlock(&cache_lock);
   session_t *s = cache_find(&session_cache, token);
   pthread_rwlock_unlock(&cache_lock);

   if (s) return s;

   // Cache miss - query SQLite
   s = session_lookup_db(token);
   if (s) {
      pthread_rwlock_wrlock(&cache_lock);
      cache_insert(&session_cache, s);
      pthread_rwlock_unlock(&cache_lock);
   }
   return s;
}
```

---

## Implementation Estimate (Summary)

| Phase | Scope | Effort |
|-------|-------|--------|
| Phase 1 | Foundation: SQLite + auth + sessions + audit logging + IP rate limiting | 6-7 days |
| Phase 2 | Complete CLI + session cleanup | 2-3 days |
| Phase 3 | Multi-user: user management UI, personal settings, permission enforcement | 3-4 days |
| Phase 4 | Enhancements: per-user history, optional 2FA (TOTP) | 2-3 days |
| Phase 5 | DAP2 integration: device tokens, authenticated handshake (deferred) | TBD |

**Total (Phases 1-4): ~13-17 days**
**Phase 5**: Deferred until DAP2 protocol redesign

### Phase 1 Deliverables (Critical - all must be implemented)
- `auth_db.c/h`: SQLite wrapper with prepared statement pool
- `auth_users.c/h`: User CRUD, Argon2id (16MB), timing normalization
- `auth_session.c/h`: Session CRUD, constant-time validation, cleanup thread
- `login_attempts` table with IP-based rate limiting
- `auth_log` table with event logging
- WebUI login page with CSRF protection
- First-run setup wizard (localhost only)
- HTTPS enforcement check
- Lock ordering documentation in code

---

## Open Questions

1. Should conversation history be per-user or shared?
2. Should there be a "family" mode where everyone shares settings?
3. How to handle ESP32 devices - separate "device" accounts or regular users?
4. Should admin be able to impersonate other users for debugging?
5. ~~Should sessions persist across DAWN restarts?~~ **Resolved**: Yes, SQLite `sessions` table persists automatically.

---

## Implementation Dependencies

### Required Libraries

| Library | Purpose | Installation |
|---------|---------|--------------|
| **libsodium** | Argon2id password hashing, secure random | `apt install libsodium-dev` |
| **libsqlite3** | Authentication database | `apt install libsqlite3-dev` |

**Why libsodium?**
- Production-quality Argon2id implementation
- `crypto_pwhash()` handles salt generation automatically
- Constant-time comparison functions for hash verification
- Already audited and widely deployed

```c
#include <sodium.h>

// Hash password
char hash[crypto_pwhash_STRBYTES];
if (crypto_pwhash_str(hash, password, strlen(password),
      crypto_pwhash_OPSLIMIT_MODERATE,
      crypto_pwhash_MEMLIMIT_MODERATE) != 0) {
   // Out of memory
   return FAILURE;
}

// Verify password
if (crypto_pwhash_str_verify(hash, password, strlen(password)) == 0) {
   // Password correct
}
```

**Why SQLite?**
- ACID transactions prevent corrupted data on crash
- WAL mode enables concurrent reads during writes
- Indexed queries for fast authentication (~20μs vs ~400μs for file parsing)
- Single dependency, ~500KB library, ~600KB runtime memory
- Standard in embedded systems (Android, iOS, Firefox all use SQLite)

```c
#include <sqlite3.h>

// ALWAYS use prepared statements to prevent SQL injection
sqlite3_stmt *stmt;
sqlite3_prepare_v2(db,
   "SELECT password_hash, is_admin FROM users WHERE username = ?",
   -1, &stmt, NULL);
sqlite3_bind_text(stmt, 1, username, -1, SQLITE_STATIC);

if (sqlite3_step(stmt) == SQLITE_ROW) {
   const char *hash = (const char *)sqlite3_column_text(stmt, 0);
   int is_admin = sqlite3_column_int(stmt, 1);
   // Verify password with libsodium...
}
sqlite3_finalize(stmt);
```

### Optional Dependencies

| Library | Purpose | Notes |
|---------|---------|-------|
| **OpenSSL/mbedTLS** | TLS for DAP protocol | Already used for HTTPS |
| **liboath** | TOTP 2FA (Phase 4) | Optional enhancement |

---

## Audit Log Format

All authentication events should be logged for security monitoring:

```
[2025-01-15 10:23:45] AUTH LOGIN_SUCCESS user=kris ip=192.168.1.50 session=a7b3...
[2025-01-15 10:24:12] AUTH LOGIN_FAILED user=admin ip=192.168.1.99 reason=bad_password
[2025-01-15 10:24:13] AUTH LOGIN_FAILED user=admin ip=192.168.1.99 reason=bad_password
[2025-01-15 10:24:14] AUTH LOCKOUT user=admin ip=192.168.1.99 duration=15m
[2025-01-15 10:30:00] AUTH LOGOUT user=kris session=a7b3...
[2025-01-15 11:00:00] AUTH SESSION_EXPIRED user=kris session=a7b3...
[2025-01-15 14:00:00] AUTH USER_CREATED user=newuser by=admin
[2025-01-15 14:05:00] AUTH PASSWORD_CHANGED user=kris by=kris
[2025-01-15 14:10:00] AUTH USER_DELETED user=olduser by=admin
```

### Log Fields

| Field | Description |
|-------|-------------|
| `user` | Username attempted/affected |
| `ip` | Client IP address |
| `session` | Session token (truncated for logs) |
| `by` | Admin who performed action (for user management) |
| `reason` | Failure reason (bad_password, locked_out, expired, etc.) |
| `duration` | Lockout duration |

### Log Destinations

1. **Syslog**: `LOG_AUTH` facility for system integration
2. **File**: `/var/log/dawn/auth.log` (configurable)
3. **In-memory**: Recent events viewable in WebUI (admin only)

```toml
[auth.logging]
syslog = true
file = "/var/log/dawn/auth.log"
recent_events_count = 100  # Keep in memory for WebUI
```

---

## Implementation Notes

### Device Config Caching

The `validate_device_in_config()` function currently reads `commands_config_nuevo.json` on every LLM command. For better performance:

```c
// Cache device list at startup and on config reload
static char **cached_device_names = NULL;
static size_t cached_device_count = 0;
static pthread_rwlock_t device_cache_lock = PTHREAD_RWLOCK_INITIALIZER;

void reload_device_cache(void) {
   pthread_rwlock_wrlock(&device_cache_lock);
   // Free old cache
   // Load and parse JSON
   // Populate cached_device_names
   pthread_rwlock_unlock(&device_cache_lock);
}

bool is_device_valid(const char *device) {
   pthread_rwlock_rdlock(&device_cache_lock);
   bool found = false;
   for (size_t i = 0; i < cached_device_count; i++) {
      if (strcmp(cached_device_names[i], device) == 0) {
         found = true;
         break;
      }
   }
   pthread_rwlock_unlock(&device_cache_lock);
   return found;
}
```

### Session Persistence

With SQLite, sessions persist automatically across restarts. The `sessions` table stores all active sessions with their tokens, user associations, and activity timestamps.

On startup:
1. Validate existing sessions (remove expired)
2. Sessions survive DAWN restarts - users stay logged in

```c
void auth_db_cleanup_expired_sessions(void) {
   time_t cutoff = time(NULL) - SESSION_TIMEOUT_SECONDS;
   sqlite3_exec(auth_db,
      "DELETE FROM sessions WHERE last_activity < ?",
      // bind cutoff...
      NULL, NULL, NULL);
}
```

---

## Related Security Issues

This design addresses:
- **#2 Unauthenticated WebUI Access** (Critical) - ✅ Implemented Phase 1
- **#3 Unauthenticated DAP Protocol** (Critical) - ✅ Resolved: DAP1 removed, DAP2 uses registration key authentication
- **#5 WebUI Secrets API Unauthenticated** (High) - ✅ Implemented Phase 1

---

## Phase 1 Security Hardening

The following security improvements were implemented after review by security, architecture, and efficiency auditors.

### CSRF Nonce Buffer Increase

**Issue**: CSRF nonce buffer (256 entries, 10-minute validity) could be exhausted by ~25 requests/minute, enabling token replay attacks.

**Fix**: Increased `CSRF_USED_NONCE_COUNT` from 256 to 512 entries.

```c
// webui_server.c
#define CSRF_USED_NONCE_COUNT 512  // Was 256, supports ~51 req/min with 10-min validity
```

**Rationale**: At 10-minute validity, 512 nonces allows ~51 requests/minute before rotation issues occur. This provides adequate headroom for legitimate form submissions while preventing replay attacks.

### Multi-IP CSRF Rate Limiting

**Issue**: Single-IP rate limiting was trivially bypassed in multi-user or shared-NAT scenarios. An attacker could exhaust another user's rate limit bucket.

**Fix**: Implemented per-IP tracking with LRU eviction for up to 32 concurrent IPs.

```c
// webui_server.c
#define CSRF_RATE_LIMIT_SLOTS 32
#define CSRF_RATE_LIMIT_MAX 30      // Max CSRF validations per window
#define CSRF_RATE_LIMIT_WINDOW 60   // Window in seconds

typedef struct {
   char ip[64];
   int count;
   time_t window_start;
   time_t last_access;
} csrf_rate_entry_t;

static struct {
   csrf_rate_entry_t entries[CSRF_RATE_LIMIT_SLOTS];
   pthread_mutex_t mutex;
} s_csrf_rate = { .entries = { { 0 } }, .mutex = PTHREAD_MUTEX_INITIALIZER };
```

**Algorithm**:
1. Find existing IP entry or allocate new slot
2. If no empty slots, evict least-recently-used entry
3. Reset counter if window expired
4. Increment counter, reject if over limit

**Memory**: 32 slots × ~80 bytes = ~2.5KB (acceptable for embedded systems)

### Multi-IP Login Rate Limiting

**Issue**: Login rate limiting used single IP storage, allowing cross-user interference.

**Fix**: Implemented in-memory fast-path rate limiting with per-IP tracking before database check.

```c
// webui_server.c
#define LOGIN_RATE_LIMIT_SLOTS 32
#define LOGIN_MAX_ATTEMPTS 5        // Max login attempts before blocking
#define LOGIN_RATE_LIMIT_WINDOW 300 // 5 minutes

typedef struct {
   char ip[64];
   int attempts;
   time_t window_start;
   time_t last_access;
} login_rate_entry_t;

static struct {
   login_rate_entry_t entries[LOGIN_RATE_LIMIT_SLOTS];
   pthread_mutex_t mutex;
} s_login_rate = { .entries = { { 0 } }, .mutex = PTHREAD_MUTEX_INITIALIZER };
```

**Integration with login flow**:
```c
static int handle_auth_login(session_t *session, cJSON *payload,
                             const char *client_ip, char **response) {
   // Fast-path: check in-memory rate limit first
   if (!login_check_rate_limit(client_ip)) {
      // Return 429 Too Many Requests
   }

   // ... password verification ...

   if (login_failed) {
      // Increment failure counter (don't reset on success for cooldown)
   }

   if (login_success) {
      login_reset_rate_limit(client_ip);  // Clear rate limit
   }
}
```

**Benefits**:
- O(1) rejection of rate-limited IPs (no database query)
- No cross-user interference
- LRU eviction prevents memory exhaustion
- Successful login resets the counter for that IP

### Shared Text Filter Utility

**Issue**: Command tag filtering (`<command>...</command>` stripping) was duplicated between `session_manager.c` and `webui_server.c`, creating cross-module dependencies.

**Fix**: Extracted to shared utility module `src/core/text_filter.c`.

**New files**:
- `include/core/text_filter.h` - Public API and state structure
- `src/core/text_filter.c` - Character-by-character state machine

**API**:
```c
// State structure - replaces per-session fields
typedef struct {
   char buffer[16];     // Tag accumulation buffer
   unsigned char len;   // Current buffer length
   bool in_tag;         // Currently inside <command>...</command>
} cmd_tag_filter_state_t;

// Reset filter state (call at session start)
void text_filter_reset(cmd_tag_filter_state_t *state);

// Callback-based filtering for streaming
typedef void (*text_filter_output_fn)(const char *text, size_t len, void *ctx);
void text_filter_command_tags(cmd_tag_filter_state_t *state,
                              const char *text,
                              text_filter_output_fn output_fn,
                              void *ctx);

// Buffer-based convenience function
int text_filter_command_tags_to_buffer(cmd_tag_filter_state_t *state,
                                       const char *text,
                                       char *out_buf,
                                       size_t out_size);
```

**Session structure update**:
```c
// include/core/session_manager.h - changed from individual fields to:
typedef struct session {
   // ...
   cmd_tag_filter_state_t cmd_tag_filter;  // Replaces cmd_tag_buffer/len/in_tag
   bool cmd_tag_filter_bypass;              // Native tools mode
   // ...
} session_t;
```

**Benefits**:
- Single implementation, tested once
- Clean module boundaries (no session_manager → webui_server dependency)
- 17 bytes per session (unchanged from before)
- Handles partial tags spanning chunk boundaries
- Supports both callback and buffer output modes

### Implementation Files Changed

| File | Changes |
|------|---------|
| `src/webui/webui_server.c` | Uses shared text filter and rate limiter; removed ~300 lines of duplicate code |
| `include/core/session_manager.h` | Uses `cmd_tag_filter_state_t` from text_filter.h |
| `src/core/session_manager.c` | Uses `text_filter_command_tags_to_buffer()` |
| `include/core/text_filter.h` | New: shared text filter API with constants and nested tag support |
| `src/core/text_filter.c` | New: state machine with nesting depth tracking |
| `include/core/rate_limiter.h` | New: generic rate limiter API |
| `src/core/rate_limiter.c` | New: multi-IP rate limiting with LRU eviction |
| `CMakeLists.txt` | Added text_filter.c and rate_limiter.c to build |

### Review Action Items Addressed

| Priority | Original Issue | Resolution |
|----------|---------------|------------|
| MEDIUM | CSRF nonce buffer exhaustion risk | Increased to 1024 entries (was 256 → 512 → 1024) |
| LOW | Single-IP CSRF rate limiter bypass | Multi-IP tracking via generic rate_limiter module |
| CRITICAL | Cross-module dependency (session_manager → webui_server) | Extracted to shared text_filter utility |
| LOW | Nested command tag handling | Use nesting depth counter instead of boolean |
| MEDIUM | Duplicate rate limiting code | Consolidated into generic rate_limiter.c/h module |
| LOW | CSRF handler IP normalization inconsistent | Added IPv6 /64 prefix normalization |
| LOW | Modulo in circular buffer | Use bitwise AND with static_assert for power-of-2 |
| LOW | IP buffer oversized (64→48 bytes) | Reduced to 48 bytes (max IPv6 = 45 chars + null) |
| INFO | Duplicate filter_command_tags_impl | Removed; now uses shared text_filter_command_tags() |
| MEDIUM | Integer overflow in nesting_depth | Added CMD_TAG_MAX_NESTING (100) with saturation |
| LOW | Rate limiter slot 0 targeting | Changed to random eviction when all slots active |
| IMPORTANT | Static init macro scope | Added documentation noting PTHREAD_MUTEX_INITIALIZER requires static storage |

### Final Implementation Details

**CSRF Nonce Buffer**: 1024 entries with compile-time verification:
```c
#define CSRF_USED_NONCE_COUNT 1024
_Static_assert((CSRF_USED_NONCE_COUNT & (CSRF_USED_NONCE_COUNT - 1)) == 0,
               "CSRF_USED_NONCE_COUNT must be power of 2");
```

**Nested Tag Overflow Protection**: Caps nesting depth at 100 to prevent integer overflow:
```c
#define CMD_TAG_MAX_NESTING 100

if (state->nesting_depth < CMD_TAG_MAX_NESTING) {
   state->nesting_depth++;
}
// Beyond max depth, stay at max (still filters content)
```

**Rate Limiter Random Eviction**: When all slots are full and active, uses random eviction to prevent targeted attacks:
```c
if (!entry) {
   entry = lru_entry;
   if (!entry) {
      /* All slots full and active - random eviction to prevent targeting */
      entry = &limiter->entries[rand() % limiter->config.slot_count];
   }
}
```

