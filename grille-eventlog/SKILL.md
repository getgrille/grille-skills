---
name: grille-eventlog
description: "Use when querying Windows Event Log for system, application, or security events. WHEN: 'event log', 'eventlog_query', 'windows events', 'check for errors in event log', 'service crash events', 'Grille security events'. DO NOT USE WHEN: reading Grille tool call history (use grille-audit); reading log files on disk (use grille-filesystem)."
---

## Overview

This skill covers Grille's Windows Event Log module — a single tool for querying the Win32 Event Log via direct EvtQuery API calls. Apply it when Claude needs to correlate system events with application behavior, investigate service failures, read Grille's own security event emissions, or perform SIEM-style queries on local machine events.

Grille queries the Event Log directly via Win32 APIs — no PowerShell subprocess, no cold start. Supported channels: System, Application, Security (requires `eventlog_allow_securitylog = true` AND Windows administrator privileges).

## Tools

- `eventlog_query` — Query Windows Event Log entries. Filter by channel, source, level (Error/Warning/Information), time range (hours), and event ID. Results capped at 200 entries.

## Grille's Own Event Emissions

Grille writes structured events to Windows Event Log for all security-relevant actions. These are queryable via `eventlog_query source="Grille"`:

| Event ID | Trigger |
|----------|---------|
| 1000 | Grille server started |
| 1001 | Grille server stopped |
| 1002 | Config reloaded |
| 1003 | Config permission warning |
| 1006 | Secret resolution failed |
| 2002 | Process execution denied (not in allowlist) |
| 2003 | Path access denied |
| 2004 | LOLBin blocked |
| 2005 | Path traversal attempt blocked |

Every denied `ps_kill` attempt (future tool) will emit event 2006 — queryable for SIEM detection.

## Patterns

**Query recent Grille security events:**
```
grille:eventlog_query
  channel="Application"
  source="Grille"
  level="Warning"
  hours=24
```

**Investigate a service failure:**
```
grille:eventlog_query
  channel="System"
  source="Service Control Manager"
  level="Error"
  hours=2
```

**Check for application errors in the last hour:**
```
grille:eventlog_query
  channel="Application"
  level="Error"
  hours=1
```

**Correlate a specific event by ID:**
```
grille:eventlog_query
  channel="Application"
  source="Grille"
  event_id=2002
  hours=48
→ All process execution denials in the last 48 hours
```

**Read system startup events:**
```
grille:eventlog_query
  channel="System"
  event_id=6005
  hours=72
→ Event 6005 = "The Event log service was started" = system boot
```

## Constraints

- **The Security channel requires both `eventlog_allow_securitylog = true` in grille.toml AND Windows administrator privileges.** Even with both flags set, the current Grille process must be running elevated. If either condition is missing, the query is denied with a clear error message.
- **Only System, Application, and Security channels are supported.** Custom channels (e.g. `Microsoft-Windows-PowerShell/Operational`) are not exposed in v1. This limits surface area and prevents probing of sensitive provider channels.
- **Results are capped at 200 entries.** For high-volume channels, narrow the time range or add source/level/event_id filters. There is no pagination — if 200 entries are returned, there may be more within the time window.
- **`eventlog_query` is read-only, always.** There are no write, clear, or export operations. The Event Log is an observability surface only.
- **The `source` parameter is sanitized to `[a-zA-Z0-9 \-_.]`.** Do not include special characters in the source filter — they will be stripped or cause the query to fail.

## Examples

**Example 1: SIEM-style check — has Grille blocked anything suspicious recently?**
```
grille:eventlog_query
  channel="Application"
  source="Grille"
  level="Warning"
  hours=168   ← last 7 days

→ Review for 2002 (process denied), 2004 (LOLBin blocked), 2005 (path traversal)
→ Any of these in production = review the session audit log for context
```

**Example 2: Diagnose a Docker service that failed to start**
```
grille:docker_ps
→ Container "rekn-api" in "Exit 1" state

grille:eventlog_query
  channel="Application"
  level="Error"
  hours=1
→ Find error entries from the application source

grille:docker_logs container="rekn-api" tail=50
→ Cross-reference with container log output
```
