---
name: grille-system
description: "Use for Grille server health, session diagnostics, and machine state. WHEN: 'start of session', 'is Grille healthy', 'check system state', 'what tools are available', 'OS version', 'disk space', 'RAM usage', 'any errors', 'how many calls this session', 'grille_diagnose', 'grille_health', 'grille_info', 'grille_session_stats', 'secrets doctor', 'check secrets', 'is my AKV reachable', 'secret provider health'. DO NOT USE WHEN: querying the Windows Event Log in depth (use grille-eventlog); reviewing tool call history (use grille-audit)."
---

## Overview

This skill covers the always-available Grille system tools — no module needs to be enabled. Use these at the start of every session to confirm Grille is healthy and the machine is in a good state before starting work.

**Tools covered:** `grille_diagnose`, `grille_info`, `grille_health`, `grille_session_stats`, `grille_reload_config`, `grille_secrets_doctor`

---

## Tool Selection

| Goal | Tool |
|------|------|
| Start-of-session health check | `grille_diagnose` |
| What version / role / modules are active | `grille_info` |
| Is something stuck or slow right now | `grille_health` |
| How many calls this session, per-tool breakdown | `grille_session_stats` |
| Apply grille.toml changes without restart | `grille_reload_config` |
| Are my secret providers reachable and auth working | `grille_secrets_doctor` |

**Default:** when in doubt at the start of a session, run `grille_diagnose` — it covers everything in one call.

---

## grille_diagnose

The primary start-of-session tool. Returns a consolidated snapshot in one call:

- **Server** — version, role, client, tool count, session call count + start time, in-flight calls
- **System** — OS display version (e.g. `Windows 11 24H2`), hostname, uptime
- **RAM** — total, available, usage %
- **Drives** — per-drive table (fixed + network), free/used/total, warning flag at 90%+ use
- **Grille Errors** — last 5 WARN/ERROR entries from the current session's audit log (config warnings excluded)
- **Windows Errors** — last 3 Error-level events from System + Application logs (last 24h)

```
grille_diagnose()   # no arguments
```

**When to use:**
- Always at the start of a session
- When something feels wrong or slow
- Before a build or deploy to confirm disk space and RAM headroom

**Notes:**
- Always available — no module required
- Windows Errors section is silently omitted if Event Log access is denied
- Session scoping uses the `grille_startup` audit marker emitted on every Grille start

---

## grille_info

Returns version, active role, enabled modules, and total tool count. Lighter than `grille_diagnose` — use when you only need to check what's available, not full machine state.

```
grille_info()   # no arguments
```

---

## grille_health

Returns live runtime state: watchdog status, in-flight tool calls with elapsed time, connected client info, and log file locations. Use when a call feels stuck or you want to see what's currently running.

```
grille_health()   # no arguments
```

---

## grille_session_stats

Returns a structured breakdown of the current session: duration, total calls, per-tool call counts, error count, most active module. Reads from the audit log — no overhead beyond file I/O.

```
grille_session_stats()                     # current session
grille_session_stats(last_n_sessions=3)    # last 3 completed sessions
```

---

## grille_secrets_doctor

Health check for all configured secret providers. Reports reachability, authentication status, configured secret ref patterns, and secret count per provider. Use when a tool fails with a secret resolution error to diagnose the root cause.

```
grille_secrets_doctor()   # no arguments
```

**Output per provider:**
- Provider name, type, vault URL (if AKV)
- Configured ref patterns and which tools they're allowed for
- Status: `✓ reachable (N secrets found)` or `✗ auth failed — ...` or `✗ unreachable — ...`

**When to use:**
- A `sql_query` or `ssh_run` fails with a secret resolution error
- After rotating credentials in AKV or CredMan
- When setting up a new machine to verify provider connectivity

---

## grille_reload_config

Applies changes to `grille.toml` without restarting Claude Desktop. Changes to `fs_allowed_paths`, `process_allowed`, `active_role`, and `fs_denied_paths` take effect immediately. Enabling new capability modules still requires a restart.

```
grille_reload_config()   # no arguments
```

**Note:** Returns "no changes detected" even when changes were applied — this is a known cosmetic issue, not a failure.

---

## Patterns

**Start every session with:**
```
grille_diagnose()
```

**Switching roles mid-session:**
1. Edit `grille.toml` → change `active_role`
2. `grille_reload_config()`
3. Verify with `grille_info()`

**Session is slow or a call is not returning:**
```
grille_health()   # shows in-flight calls + elapsed time
```
