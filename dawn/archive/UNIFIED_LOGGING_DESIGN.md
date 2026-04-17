# Unified Logging Design

**Date**: April 2026
**Scope**: DAWN, ECHO, MIRAGE, STAT
**Status**: Approved — implementation in progress

---

## Problem

Four OASIS repos have four slightly different logging implementations with two lineages:

| Lineage | Repos | Macros | Syslog | Timestamps | LOC |
|---------|-------|--------|--------|------------|-----|
| DAWN-style | DAWN, MIRAGE | `LOG_*` | No | DAWN: `HH:MM:SS.mmm`, MIRAGE: none | 222, 164 |
| STAT-style | STAT, ECHO | `OLOG_*` | Yes | ECHO: `HH:MM:SS.mmm`, STAT: none | 197, 189 |

ECHO's logging.c says "copied from STAT" and MIRAGE's is derived from the same root. They've diverged in formatting style, feature set, and macro names. No repo has the complete feature set.

## Goal

One canonical `logging.h` + `logging.c` (~220 lines) that all four repos copy. Not a shared library or git submodule — just a file kept byte-identical across repos, like `.clang-format`.

## Canonical Feature Set

Pick the best from each:

| Feature | Source | Notes |
|---------|--------|-------|
| `OLOG_*` macros | STAT/ECHO | Avoids collision with syslog's `LOG_INFO` constant |
| Millisecond timestamps | DAWN/ECHO | `HH:MM:SS.mmm` via `gettimeofday()` on console output |
| Syslog support | STAT/ECHO | `init_syslog(ident)` for systemd/journalctl integration |
| File output | All | `init_logging(filename, LOG_TO_FILE)` |
| Console colors | All | ANSI green/yellow/red, disabled in file mode |
| Console suppression | DAWN | `logging_suppress_console()` for TUI or headless modes |
| Fixed-width preamble | All | 45 chars (accommodates timestamp + level + filename:line) |
| Newline stripping | All | Remove `\n`/`\r` from messages before output |
| `LOG_DEBUG` macro | DAWN | No-op in release builds; minor cost; useful across repos |
| `LOG_CREDENTIAL_STATUS` | DAWN | Credential-safe logging helper; useful across repos |

### Macro Signatures

```c
OLOG_INFO(fmt, ...)
OLOG_WARNING(fmt, ...)
OLOG_ERROR(fmt, ...)
LOG_DEBUG(fmt, ...)              /* debug builds only */
LOG_CREDENTIAL_STATUS(key)       /* evaluates to "(configured)" or "(not configured)" */
```

Expand to:
```c
log_message(LOGLEVEL_INFO, __FILE__, __LINE__, fmt, ##__VA_ARGS__)
```

### Dropping `__func__`

DAWN and MIRAGE's `log_message` signature takes `const char *func` but **never formats it into the output**. DAWN explicitly discards it (`(void)func;`); MIRAGE captures it without using it. Removing it:

- Removes several KB of dead `__func__` symbols from `.rodata` (the compiler emits a unique `static const char[]` per enclosing function; these can't be folded)
- Removes one argument register/stack push per call site (~3,185 DAWN + ~406 MIRAGE sites)
- Simplifies the signature

`file:line` is sufficient for source navigation — IDEs jump directly to the exact line.

### Console Output Format

```
[INFO] 01:23:45.678 mqtt_comms.c:125:     Connected to broker
[WARN] 01:23:46.012 modem.c:87:           Signal strength low
[ERR ] 01:23:47.345 at_command.c:200:     Serial port timeout
```

Timestamp + level + file:line + padding to 45 chars + message. ANSI color wraps the entire line. **INFO and WARNING go to `stdout`; ERROR goes to `stderr`** (standard Unix convention, matches STAT/ECHO/MIRAGE).

### Syslog Output Format

```
[mqtt_comms.c:125] Connected to broker
```

No timestamp (syslog adds its own). No colors.

### Log Levels

```c
typedef enum {
   LOGLEVEL_INFO = LOG_INFO,
   LOGLEVEL_WARNING = LOG_WARNING,
   LOGLEVEL_ERROR = LOG_ERR,
} log_level_t;
```

Maps directly to syslog values for zero-cost syslog dispatch. Requires `#include <syslog.h>` in `logging.h`.

### API

```c
/* Initialization */
int init_logging(const char *filename, int mode);  /* LOG_TO_CONSOLE or LOG_TO_FILE */
int init_syslog(const char *ident);                /* Switch to syslog mode */
void close_logging(void);

/* Console suppression (for TUI/headless modes) */
void logging_suppress_console(int suppress);

/* Core logging function (called via macros) */
void log_message(log_level_t level, const char *file, int line, const char *fmt, ...);
```

`init_logging` keeps the historical `(filename, mode)` argument order used by DAWN, STAT, and ECHO. Modes: `LOG_TO_CONSOLE = 0`, `LOG_TO_FILE = 1`, `LOG_TO_SYSLOG = 2` (reserved constant; use `init_syslog()` to enter that mode).

## File Structure

Each repo gets **byte-identical** files:
```
include/logging.h    # Canonical header
src/logging.c        # Canonical implementation
```

Paths vary per repo (MIRAGE uses `include/util/logging.h`, DAWN uses `common/include/logging.h`) but file *contents* must match exactly. Verify with `diff` or `sha256sum`.

### DAWN-Specific Bridge (separate files)

DAWN's `common/` library uses `DAWN_LOG_*` macros with a callback abstraction so common-library code doesn't directly depend on the daemon's logging. The bridge that wires `DAWN_LOG_*` → `log_message` is kept **outside** the canonical `logging.c` to preserve byte-identity across repos:

```
common/include/logging_bridge.h   # DAWN-only bridge API
common/src/logging_bridge.c       # Bridge implementation
common/include/logging_common.h   # DAWN_LOG_* macros (unchanged)
common/src/logging_common.c       # Callback dispatch (unchanged)
```

The daemon calls `logging_bridge_install()` after `init_logging()` in `src/dawn.c`.

## Migration Plan

### Phase 1: DAWN (largest effort; must go first)

1. **Write canonical `common/include/logging.h` and `common/src/logging.c`** with the feature set above.
2. **Extract the DAWN-only bridge** from the old `common/src/logging.c` into new `common/src/logging_bridge.c` (+ `common/include/logging_bridge.h`). Call `logging_bridge_install()` from `src/dawn.c` right after `init_logging()`.
3. **Rename macros repo-wide** via search/replace:
   - `\bLOG_INFO\s*\(` → `OLOG_INFO(`
   - `\bLOG_WARNING\s*\(` → `OLOG_WARNING(`
   - `\bLOG_ERROR\s*\(` → `OLOG_ERROR(`
   - `LOG_DEBUG` is already the canonical name — no rename needed.
   - `LOG_CREDENTIAL_STATUS` is already canonical — no rename needed.
   - **Must be scoped**: exclude `common/logging_common.h` (where `DAWN_LOG_*` macros expand using the literal symbols `DAWN_LOG_INFO/WARNING/ERROR` inside `#define` bodies), and exclude third-party/webrtc subtrees.
4. **Verify direct `log_message` callers** — old signature was `(level, file, line, func, fmt, ...)`, new is `(level, file, line, fmt, ...)`. Grep for direct calls; the only one in DAWN is the bridge callback, which is being rewritten anyway.
5. **Build and test**: `cmake --preset debug && make -C build-debug -j8`, run all 13 unit test suites.
6. **Commit**.

### Phase 2: STAT, ECHO, MIRAGE (parallel, straightforward)

Once canonical files are committed in DAWN, copy them to the other three repos in parallel:

- **STAT**: Already uses `OLOG_*`. Drop-in replacement. Timestamps now appear in console output. `init_logging` arg order unchanged.
- **ECHO**: Same as STAT. Already has timestamps.
- **MIRAGE**: Uses `LOG_*`. Requires macro rename (same search/replace pass as DAWN). Adds timestamps to output.

Each: replace files → build → run tests → commit.

### Search/Replace Safety

The macro rename is mechanical but must not touch:
- `#define DAWN_LOG_INFO` / `DAWN_LOG_WARNING` / `DAWN_LOG_ERROR` in `common/include/logging_common.h` (these reference the enum values `DAWN_LOG_INFO` etc., not the `LOG_INFO` macro)
- String literals containing `LOG_INFO` or `LOG_ERROR` (if any — must grep first)
- Third-party subtrees: `whisper.cpp/`, `webrtc-audio-processing/`, `build*/`
- Syslog constants `LOG_INFO` / `LOG_WARNING` / `LOG_ERR` used as enum initializers in `logging.h` (not followed by `(`)

The `\b<name>\s*\(` regex anchors on a function/macro call site (word boundary + name + optional whitespace + open paren), which excludes all of the above. Verified by grep preview before replacement.

## Verification

1. Build all four repos after replacing logging files.
2. Run all unit tests (DAWN: 13 suites, ECHO: 4 suites, STAT: existing suites).
3. Visual check: console output shows timestamps in all repos.
4. Syslog check: `journalctl -u oasis-echo` and `journalctl -u oasis-stat` show properly formatted messages.
5. `sha256sum include/logging.h src/logging.c` across repos — must be byte-identical.

## Open Items

- `LOG_TO_SYSLOG = 2` is defined as a constant but `init_logging(filename, LOG_TO_SYSLOG)` is not a supported entry point (callers use `init_syslog()` directly). Keep the constant for source-level compatibility with STAT/ECHO where it was defined but unused.
