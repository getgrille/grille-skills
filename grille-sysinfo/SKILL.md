---
name: grille-sysinfo
description: "Use for machine hardware and OS state: OS version, uptime, RAM, disk space, installed software. WHEN: 'what OS is this', 'how much disk space', 'RAM usage', 'what's installed', 'sys_summary', 'sys_drives', 'sys_software', 'check disk before build'. DO NOT USE WHEN: you need full session health (use grille_diagnose — it includes drives and RAM); querying processes (use grille-processes); querying Windows Event Log (use grille-eventlog)."
---

## Overview

This skill covers the `sys_info` module — three read-only tools that return machine hardware and OS state via native Win32 APIs. No WMI, no subprocess, no elevation required.

**Tools covered:** `sys_summary`, `sys_drives`, `sys_software`

---

## Tool Selection

| Goal | Tool |
|------|------|
| OS version, uptime, CPU, RAM overview | `sys_summary` |
| Per-drive disk space (free/used/total) | `sys_drives` |
| Installed software inventory | `sys_software` |

**Note:** `grille_diagnose` includes drives and RAM in its consolidated output — use that at session start. Use `sys_drives` / `sys_summary` directly when you need just that data mid-session without the full diagnostic snapshot.

---

## sys_summary

Returns a snapshot of the local machine:

- OS display version (e.g. `Windows 11 24H2 (Build 26200)`) — via `RtlGetVersion`, not `GetVersionExW`
- Uptime (days/hours/minutes/seconds)
- Hostname
- CPU count and architecture
- RAM total, available, usage %

```
sys_summary()   # no arguments
```

---

## sys_drives

Returns per-drive disk usage for all fixed and network drives:

- Drive letter, volume label, filesystem type (NTFS/ReFS/FAT32)
- Total, free, used space
- Usage % with warning flag at 90%+
- Drive type (Fixed, Network, Removable, etc.)

```
sys_drives()   # no arguments
```

**When to use:** Before a large build, file copy, or deploy to confirm there's enough headroom.

---

## sys_software

Returns the installed software inventory from the Windows registry Uninstall keys (HKLM 64-bit, HKLM 32-bit, HKCU). Each entry includes display name, version, publisher, install date, and install location where available.

```
sys_software()   # no arguments
```

**Notes:**
- Only returns entries with a display name — filters out internal Windows components
- Covers both MSI-installed and MSIX/Store apps visible in registry
- Does not cover portable apps (no installer, no registry entry)
