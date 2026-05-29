---
name: grille-windows-update
description: "Use when checking pending Windows updates on the local machine. WHEN: 'are there any pending updates', 'check Windows Update', 'is this machine patched', 'any security patches pending', 'is KB<number> installed', 'check patch compliance', 'pending Windows updates', 'wu_status', 'Defender definitions up to date', 'missing patches'. DO NOT USE WHEN: installing or downloading updates (this module is read-only); querying installed software (use grille-wmi with Win32_Product); checking update history (not exposed by this module)."
---

## Overview

This skill covers Grille's `windows_update` module — query pending Windows updates via the Windows Update Agent (WUA) COM API. Read-only. No elevation required.

The module calls `IUpdateSearcher::Search("IsInstalled=0")` only. It does not download, install, or modify anything.

## Tools

- `wu_status` — List pending updates. Returns title, severity, KB article numbers, download size, mandatory flag, and categories.

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `include_hidden` | No | Include hidden/deferred updates. Default `false`. |
| `max_results` | No | Max updates to return. Default 50, max 200. |

## Patterns

**Check for pending updates:**
```
grille:wu_status
```

**Include hidden/deferred updates:**
```
grille:wu_status
  include_hidden=true
```

**Check for a large batch (WSUS environment):**
```
grille:wu_status
  max_results=200
```

## Output format

```
3 pending update(s)
Severity summary: 1 Critical  1 Important  1 Other

 1. 2024-12 Cumulative Update for Windows 11 [KB5048685]
    Severity: Critical  Size: 312.4 MB
    Categories: Security Updates, Windows

 2. Security Intelligence Update for Microsoft Defender Antivirus - KB2267602 [KB2267602]
    Severity: Unspecified  Size: unknown
    Categories: Definition Updates, Microsoft Defender Antivirus
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
- **Install history is not exposed** — query is hardcoded to pending updates only.

## Error patterns

| Error | Cause | Fix |
|---|---|---|
| `CoCreateInstance(UpdateSession) failed` | WUA COM server unavailable | Rare on modern Windows; check Windows Update service is running |
| `GetIDsOfNames failed` | Unexpected WUA interface version | Report as a bug |
