---
name: grille-audit
description: "Use when verifying tool calls completed, reviewing recent Grille activity, or diagnosing failures. WHEN: 'did that write succeed', 'check audit log', 'what did Claude do', 'recent tool calls', 'grille_audit', 'any errors', 'verify the action'. DO NOT USE WHEN: looking for system events (use grille-eventlog); discovering config (use grille_info)."
---

## Overview

This skill covers `grille_audit` — Grille's always-available audit log reader. Apply it whenever Claude needs to confirm that a tool call actually completed, investigate a failure that returned an ambiguous result, or review security events such as denied operations, LOLBin blocks, and path traversal attempts.

The audit log is a JSONL file written locally at `%APPDATA%\Grille\logs\grille-audit-YYYY-MM-DD.jsonl`. It is append-only and tamper-evident. Every tool call is logged — successful or not.

## Tools

- `grille_audit` — Read recent audit log entries. Supports filtering by tool name, result (ok/error), and time range. Returns structured entries with timestamp, call ID, tool name, role, result, duration, and (for errors) error details.

## Patterns

**Verify a write succeeded (use after every fs write):**
```
grille:grille_audit tool="fs_str_replace" limit=5
→ Confirm most recent entry shows result="ok" for the target path
→ Cross-check with fs_stat for file size and mtime
```

**Diagnose a tool failure:**
```
grille:grille_audit tool="sql_query" limit=10
→ Find the failing call by timestamp
→ Read error message — Grille errors are directive (they say what to do next)
→ Note: error messages name the problem but do not enumerate config details
   Use grille_info for config discovery, not error messages
```

**Check for recent denied operations:**
```
grille:grille_audit result="error" limit=20
→ Review any DENY entries for pattern (wrong path, wrong executable, etc.)
→ All denied ps_kill attempts (future) also appear here AND in Windows Event Log
```

**Review a full session's activity:**
```
grille:grille_audit limit=50
→ Ordered newest-first by default
→ Use to build a summary of what Claude did in the session
```

**Filter to secrets-related events:**
```
grille:grille_audit tool="sql_query" limit=20
→ Look for secret_resolution entries in results
→ Secret refs (e.g. "work/db:rekn") are logged — secret values are never logged
```

## Constraints

- **Do not claim a tool call succeeded based solely on the tool's return value.** Always confirm with `grille_audit` for writes to important files or multi-step operations. The audit log is the authoritative record of what happened.
- **Do not use `grille_audit` as a substitute for `grille_info` for config discovery.** Audit entries show what happened — they deliberately do not enumerate config details (allowlists, host lists, connection names) per SD-015. Use `grille_info` to see configured resources.
- **Audit log entries show secret refs, never secret values.** A log entry for a SQL call may show `secret_ref = "work/db:rekn"` — that is the opaque reference token. The resolved password value never appears in the log.
- **The audit log is local and user-scoped.** Entries from the desktop machine's Grille instance are not visible from the laptop, and vice versa. Each machine maintains its own audit log.
- **Retention is 30 days by default** (configurable in `grille.toml` `[logging]` section). Entries older than the retention window are pruned automatically.

## Examples

**Example 1: Confirm a journal entry was successfully appended**
```
grille:grille_audit tool="fs_str_replace" limit=3

Expected output:
{
  "ts": "2026-05-05T10:23:45Z",
  "tool": "fs_str_replace",
  "args": {"path": "C:\\dev\\grille-docs\\02_DevJournal.md"},
  "result": "ok",
  "duration_ms": 4
}

→ result="ok" + path matches + timestamp is recent = write confirmed
Then also: grille:fs_stat path="C:\\dev\\grille-docs\\02_DevJournal.md"
→ Verify size increased
```

**Example 2: Investigate why a process call was denied**
```
grille:grille_audit result="error" limit=10

→ Find entry: tool="process_run", result="denied"
   error: "Executable 'az' is not in process_allowed"

→ Diagnosis: missing .cmd extension
→ Fix: use "az.cmd" not "az" as the executable name
```
