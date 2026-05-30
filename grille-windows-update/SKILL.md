---
name: grille-windows-update
description: "Use when checking Windows updates on the local machine. WHEN: 'are there any pending updates', 'check Windows Update', 'is this machine patched', 'any security patches pending', 'is KB<number> installed', 'check patch compliance', 'pending Windows updates', 'wu_status', 'Defender definitions up to date', 'missing patches', 'list installed updates', 'what updates are installed', 'show update history', 'when was KB installed', 'what patches have been applied'. DO NOT USE WHEN: installing or downloading updates (this module is read-only); querying installed software/apps (use grille-wmi with Win32_Product)."
---

## Overview

This skill covers Grille's `windows_update` module — query Windows updates via the Windows Update Agent (WUA) COM API. Read-only. No elevation required.

Three modes:
- **Search mode** (`history=false`, default): calls `IUpdateSearcher::Search()` — returns pending, installed, or all updates. No dates.
- **History mode** (`history=true`): calls `IUpdateSearcher::QueryHistory()` — returns install/uninstall history with dates and result codes.
- **Search filter** (`search`): client-side substring filter on title or KB number, works in both modes.
- **Exclude filter** (`exclude`): client-side substring filter that drops matching titles, applied after `search`. Works in both modes.

## Tools

- `wu_status` — Query Windows updates. Pending, installed, all, or history with dates.

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `status` | No | `"pending"` (default), `"installed"`, or `"all"`. Ignored when `history=true`. |
| `history` | No | `true` = use QueryHistory, adds install dates and result codes. Default `false`. |
| `search` | No | Filter by title or KB number. Case-insensitive substring. E.g. `"KB5048685"`, `"Defender"`, `"Cumulative"`. |
| `exclude` | No | Exclude results whose title contains this substring. Case-insensitive. Applied after `search`. E.g. `"Defender"` to skip definition updates, `"WinAppRuntime"` to skip runtime updates. |
| `include_hidden` | No | Include hidden/deferred updates. Default `false`. Only applies when `history=false`. |
| `max_results` | No | Max updates to return. Default 50, max 200. |

## Patterns

**Check pending updates (default):**
```
grille:wu_status
```

**List installed updates:**
```
grille:wu_status
  status="installed"
```

**Full update history with install dates:**
```
grille:wu_status
  history=true
```

**Check whether a specific KB is installed:**
```
grille:wu_status
  status="installed"
  search="KB5048685"
```

**Search history for a specific KB:**
```
grille:wu_status
  history=true
  search="KB5048685"
```

**Search pending updates for Defender:**
```
grille:wu_status
  search="Defender"
```

**Exclude noisy entries from history (recommended default):**
```
grille:wu_status
  history=true
  exclude="WinAppRuntime"
```
When reviewing update history, always pass `exclude="WinAppRuntime"` unless the
user specifically asks about WinAppRuntime updates — they otherwise flood the
history with dozens of near-identical runtime entries.

**Large WSUS environment:**
```
grille:wu_status
  status="installed"
  max_results=200
```

## Output format

Search mode:
```
3 pending update(s)
Severity summary:  1 Critical  1 Important  1 Other

  1. 2024-12 Cumulative Update for Windows 11 [KB5048685]
     Severity: Critical  Size: 312.4 MB
     Categories: Security Updates, Windows

  2. Security Intelligence Update for Microsoft Defender Antivirus [KB2267602]
     Severity: Unspecified  Size: unknown
     Categories: Definition Updates, Microsoft Defender Antivirus
```

History mode:
```
50 history entries (total on machine: 312)

  1. 2024-12 Cumulative Update for Windows 11 [KB5048685]  [2024-12-10]
     Installed  —  Succeeded
     Categories: Security Updates, Windows

  2. Security Intelligence Update for Microsoft Defender Antivirus [KB2267602]  [2024-12-09]
     Installed  —  Succeeded
```

## Configuration

```toml
[roles.developer]
modules = ["windows_update", ...]
```

No additional configuration keys — enable the module and it is ready.

## Notes

- **First call may be slow (10–30s)** if the WUA cache is stale.
- **Size may show as unknown** for driver and definition updates — expected, not an error.
- **WSUS-managed machines** return the policy-approved update set, not the full catalog.
- **History mode filters failures** — ResultCode 4 (Failed) and 5 (Aborted) are excluded by default. Pass `max_results=200` to get a broader view.
- **KB numbers in history** — extracted from the title if not exposed directly by the WUA interface.
- **Combine with `grille_diagnose`** for the full picture: `grille_diagnose` gives OS version/build, `wu_status` gives what's been applied on top.

## Error patterns

| Error | Cause | Fix |
|---|---|---|
| `CoCreateInstance(UpdateSession) failed` | WUA COM server unavailable | Rare on modern Windows; check Windows Update service is running |
| `GetIDsOfNames failed` | Unexpected WUA interface version | Report as a bug |
| `GetTotalHistoryCount returned unexpected type` | WUA returned no history count | Try without `history=true` first |
