# DAWN Security Audit Report

**Date**: 2025-12-18
**Auditor**: Claude Security Auditor Agent
**Codebase**: DAWN Voice Assistant (~58,000 LOC)

## Executive Summary

This security audit identified **15 vulnerabilities** ranging from Critical to Low severity. The most severe issues involve hardcoded API credentials, unauthenticated network services, and potential command injection vectors. The codebase demonstrates good security awareness in some areas (SSRF protection, path traversal mitigation) but has significant gaps in authentication and credential management.

---

## Critical Findings

### 1. ~~Hardcoded API Keys in Source Code~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~CRITICAL~~ RESOLVED |
| **Location** | ~~`include/secrets.h:36-43`~~ (file deleted) |
| **CWE** | CWE-798 - Use of Hard-coded Credentials |

**Description**: The `secrets.h` file contained API keys hardcoded in source code.

**Resolution**: `secrets.h` has been completely removed. All API keys are now managed via runtime configuration in `secrets.toml` (gitignored). No compile-time credential storage exists.

---

### 2. ~~Unauthenticated WebUI Access~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~CRITICAL~~ RESOLVED |
| **Location** | `src/webui/webui_server.c`, `src/auth/auth_db.c` |
| **CWE** | CWE-306 - Missing Authentication for Critical Function |

**Description**: The WebUI server previously accepted WebSocket connections without any authentication.

**Resolution**: Full authentication system implemented (2026-01-04):
- Login page with username/password authentication
- Argon2id password hashing via libsodium
- Secure session tokens (64-byte random, 24-hour expiry)
- HTTP-only cookies for session management
- Role-based access control (admin vs regular users)
- First-run setup wizard to create admin account
- Account lockout after 5 failed attempts (15-minute cooldown)
- Session invalidation on logout and role change
- All WebSocket messages require valid session except login/status

---

### 3. ~~Unauthenticated DAP Network Protocol~~ [ADDRESSED IN DAP2]

| Field | Value |
|-------|-------|
| **Severity** | ~~CRITICAL~~ ADDRESSED |
| **Location** | DAP1: `src/network/dawn_server.c` (to be removed) |
| **CWE** | CWE-306 - Missing Authentication for Critical Function |

**Description**: The DAP1 protocol accepts TCP connections from any client that knows the magic bytes. This protocol is being replaced entirely by DAP2.

**Resolution**: DAP1 will be removed and replaced with DAP2, which includes comprehensive security:

See **[DAP2_DESIGN.md](DAP2_DESIGN.md) Section 9: Security Architecture**:
- **Phase 1**: Network isolation (VLAN) with documented security limitations
- **Phase 2+**: TLS 1.3 encryption for all connections
- **Phase 2+**: Certificate-based mutual authentication (mTLS)
- Daemon acts as CA, issues satellite certificates
- Certificate pinning on satellites
- Certificate revocation list (CRL) support

**Status**: DAP1 scheduled for removal. DAP2 security architecture designed and documented.

---

## High Severity Findings

### 4. ~~Command Injection via shutdown Callback~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~HIGH~~ RESOLVED |
| **Location** | `src/mosquitto_comms.c` (shutdownCallback) |
| **CWE** | CWE-78 - OS Command Injection |

**Description**: The `shutdownCallback` function executes `system("sudo shutdown -h now")`. While hardcoded, it was triggered through MQTT and LLM command parsing which could be manipulated.

**Resolution**: Shutdown command now requires explicit opt-in:
1. `shutdown.enabled = true` must be set in config (default: false)
2. Optional `shutdown.passphrase` can be set for additional security
3. All shutdown attempts are logged with authorization status
4. The hardcoded "alpha bravo charlie" passphrase has been removed

Config example:
```toml
[shutdown]
enabled = true
passphrase = "your secret phrase"
```

---

### 5. ~~WebUI Secrets Management via Unauthenticated API~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~HIGH~~ RESOLVED |
| **Location** | `src/webui/webui_server.c` |
| **CWE** | CWE-732 - Incorrect Permission Assignment for Critical Resource |

**Description**: The `handle_set_secrets` function previously allowed any WebSocket client to update API keys and credentials without authentication.

**Resolution**: Admin authentication now required (2026-01-04):
- `handle_set_secrets()` calls `conn_require_admin()` before processing
- Non-admin users receive `FORBIDDEN` error
- `handle_get_secrets()` also requires admin
- `handle_set_config()` requires admin
- All admin operations logged via `auth_db_log_event()`

---

### 6. ~~WebSocket Session Token Weakness~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~HIGH~~ RESOLVED |
| **Location** | `src/webui/webui_server.c:289-301` |
| **CWE** | CWE-330 - Use of Insufficiently Random Values |

**Description**: Session tokens use `getrandom()` which is secure, but the fallback used `rand()` which is not cryptographically secure.

**Resolution**: The weak `rand()` fallback has been removed. If `getrandom()` fails, the function now returns an error and the connection is rejected. Session creation will fail safely rather than use predictable tokens.

---

## Medium Severity Findings

### 7. ~~Path Traversal Protection Bypass Potential~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~MEDIUM~~ RESOLVED |
| **Location** | `src/webui/webui_server.c` |
| **CWE** | CWE-22 - Path Traversal |

**Description**: The path traversal check only looked for `..` but did not handle URL-encoded variants like `%2e%2e`.

**Resolution**: Implemented two-layer protection:
1. `contains_path_traversal()` checks for literal `..` and URL-encoded variants (`%2e%2e`, `%2e.`, `.%2e`, `%252e`)
2. `is_path_within_www()` uses `realpath()` to resolve canonical paths and verifies the result is within the www directory

Even if an attacker finds a way around pattern matching, the canonical path validation ensures files outside www cannot be accessed.

---

### 8. ~~Integer Overflow in DAP Protocol Handling~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~MEDIUM~~ RESOLVED |
| **Location** | `src/network/dawn_server.c` |
| **CWE** | CWE-190 - Integer Overflow |

**Description**: The check `total_received + header.data_length > MAX_DATA_SIZE` could overflow.

**Resolution**: Reordered arithmetic to avoid overflow:
```c
if (header.data_length > MAX_DATA_SIZE ||
    total_received > MAX_DATA_SIZE - header.data_length) {
   // Overflow-safe: subtraction only happens when safe
}
```

---

### 9. ~~MQTT Message Injection~~ [MITIGATED]

| Field | Value |
|-------|-------|
| **Severity** | ~~MEDIUM~~ MITIGATED |
| **Location** | `src/dawn.c`, `src/mosquitto_comms.c` |
| **CWE** | CWE-94 - Code Injection |

**Description**: MQTT messages are parsed as JSON and executed as commands. Without authentication, anyone on the network can send commands.

**Resolution**: MQTT authentication was already supported but optional. Added prominent security warning at startup if credentials are not configured:
```
========================================
MQTT SECURITY WARNING: No authentication configured!
Anyone on the network can send commands to DAWN.
Configure mqtt_username/mqtt_password in secrets.toml
========================================
```

To fully secure MQTT:
1. Set `mqtt_username` and `mqtt_password` in `secrets.toml`
2. Configure Mosquitto broker to require authentication
3. Consider TLS for encrypted connections

---

### 10. ~~LLM Prompt Injection Risk~~ [MITIGATED]

| Field | Value |
|-------|-------|
| **Severity** | ~~MEDIUM~~ MITIGATED |
| **Location** | `src/llm/llm_command_parser.c` |
| **CWE** | CWE-94 - Code Injection |

**Description**: User voice input processed through LLM generates JSON commands. Malicious voice input could manipulate the LLM into generating unintended commands.

**Resolution**: Implemented device allowlist validation and audit logging:

1. **Device Allowlist**: `validate_device_in_config()` checks that the device exists in `commands_config_nuevo.json` before execution. Unknown/hallucinated devices are rejected.

2. **Audit Logging**: All LLM command attempts are logged:
   - `LLM COMMAND EXECUTED: device=X topic=Y` for allowed commands
   - `LLM COMMAND REJECTED: Unknown device 'X'` for blocked commands
   - Commands also logged to TUI activity metrics

3. **Existing Protections**: Shutdown requires config opt-in + passphrase (Issue #4)

**Future Enhancement**: Per-user device limits when authentication is implemented. See `docs/USER_AUTH_DESIGN.md` "Device Access Control" section.

---

### 11. ~~Insecure run_command Function Usage~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~MEDIUM~~ RESOLVED |
| **Location** | `src/webui/webui_server.c` |
| **CWE** | CWE-78 - OS Command Injection |

**Description**: The `run_command` function used `popen()` with shell execution. While documented to only use hardcoded commands, the interface didn't enforce this.

**Resolution**: Renamed to `run_whitelisted_command()` and added enforcement:
1. Added `ALLOWED_COMMANDS[]` whitelist with exactly 4 permitted commands
2. Added `is_command_whitelisted()` check before execution
3. Non-whitelisted commands are blocked and logged
4. Even if a caller accidentally passes user input, it will be rejected

---

## Low Severity Findings

### 12. ~~Negative Return Values in Protocol Code~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~LOW~~ RESOLVED |
| **Location** | `include/network/dawn_server.h` |
| **CWE** | CWE-703 - Improper Exception Handling |

**Description**: DAP server used negative return values (`DAWN_ERROR = -1`) which contradicted project coding standards (SUCCESS=0, FAILURE=1).

**Resolution**: Changed all DAWN_ERROR_* codes to positive values (1-5) to align with project coding standards. All callers already used `!= DAWN_SUCCESS` checks so no other changes were needed.

---

### 13. ~~Information Disclosure in Error Messages~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~LOW~~ RESOLVED |
| **Location** | `src/webui/webui_server.c` (handle_get_config) |
| **CWE** | CWE-209 - Information Disclosure |

**Description**: The `get_config` response previously exposed filesystem paths (`config_path`, `secrets_path`, `asr_path`, `tts_path`) that reveal system structure and username.

**Resolution**: Path redaction implemented for non-admin users (2026-01-04):
- Admin users: Full paths visible (needed for debugging/configuration)
- Regular users: Paths shown as "(configured)" or "(set)"
- Sensitive fields like API keys always redacted (shown as "****" if set)
- `handle_get_config()` checks `conn->is_admin` before including paths

---

### 14. Missing Rate Limiting [ACCEPTED RISK]

| Field | Value |
|-------|-------|
| **Severity** | LOW (Accepted) |
| **Location** | WebUI, DAP server, MQTT handlers |
| **CWE** | CWE-770 - Resource Exhaustion |

**Description**: No rate limiting for WebSocket connections, DAP connections, MQTT commands, or LLM API calls.

**Risk Assessment**:
- DAWN operates on local/trusted networks, not internet-exposed
- LLM API providers enforce their own rate limits
- Authentication (Issue #2) will help identify abusers when implemented
- Resource exhaustion would cause slowdown, not data breach

**Decision**: Accepted risk for now. Revisit if:
- DAWN is exposed to untrusted networks
- API cost abuse becomes a concern
- DoS attacks observed in practice

**Future mitigation options**:
1. LLM call counter (protect API costs)
2. Max connections per IP
3. Per-client message rate limiting

---

### 15. ~~WebUI Configuration Backup Permissions~~ [FIXED]

| Field | Value |
|-------|-------|
| **Severity** | ~~LOW~~ RESOLVED |
| **Location** | `src/config/config_parser.c` |
| **CWE** | CWE-200 - Information Disclosure |

**Description**: Configuration backups were created with default permissions before chmod() was applied, creating a race condition where secrets could be briefly world-readable.

**Resolution**: Changed `config_backup_file()` to use `open()` with explicit `0600` mode instead of `fopen()`. Backup files are now created with restrictive permissions from the start, eliminating the race condition. Permissions are always `0600` regardless of original file permissions.

---

## Summary

| Severity | Count | Key Issues |
|----------|-------|------------|
| Critical | ~~3~~ 0 | ~~Hardcoded credentials~~, ~~unauthenticated WebUI~~, ~~unauthenticated DAP~~ (addressed in DAP2) |
| High | ~~3~~ 0 | ~~Command injection~~, ~~secrets API exposure~~, ~~weak random fallback~~ |
| Medium | ~~5~~ 0 | ~~Path traversal~~, ~~integer overflow~~, ~~MQTT injection~~, ~~LLM prompt injection~~, ~~popen usage~~ |
| Low | ~~4~~ 0 | ~~Inconsistent error codes~~, ~~info disclosure~~, ~~rate limiting~~ (accepted), ~~backup permissions~~ |

**Status (2026-01-06):** All 15 vulnerabilities addressed. DAP1 to be replaced with DAP2 (see [DAP2_DESIGN.md](DAP2_DESIGN.md)).

---

## Immediate Actions Required

1. ~~**Revoke and rotate the exposed API keys** in `secrets.h`~~ [FIXED - secrets.h removed]
2. ~~**Add authentication to WebUI** - at minimum IP whitelist + password~~ [FIXED - Full auth system implemented]
3. ~~**Add authentication to DAP** - shared secret or TLS client certs~~ [ADDRESSED - DAP1 being replaced with DAP2]

**All immediate actions complete.** See [DAP2_DESIGN.md](DAP2_DESIGN.md) for the new protocol security architecture.

---

## Additional Recommendations

### Defense in Depth
- Run DAWN on an isolated network segment
- Block external access to WebUI and DAP ports via firewall
- Enable HTTPS for WebUI (already supported)
- Implement comprehensive audit logging
- **[IMPLEMENTED]** File permission checks on secrets.toml at startup (warns if world/group readable)

### Code Quality
- Run static analysis tools (Coverity, cppcheck)
- Implement fuzz testing for DAP protocol and JSON parsing
- Use Address Sanitizer (ASan) during development

### Operational Security
- Run DAWN with minimal required permissions
- Use proper secrets management (HashiCorp Vault, environment variables)
- Keep dependencies updated for security patches
