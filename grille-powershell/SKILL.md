---
name: grille-powershell
description: "Use when running PowerShell cmdlets or pipeline expressions on Windows via Grille ps_run. WHEN: 'run Get-Process', 'run PowerShell', 'run cmdlet', 'Test-NetConnection', 'ps_run', 'Get-Service'. DO NOT USE WHEN: 'run cargo', 'run git', 'run npm' (use grille-process); 'start service', 'stop service' (use grille-services)."
---

## Overview

This skill covers `ps_run` — Grille's PowerShell execution tool. Apply it when Claude needs to run PowerShell cmdlets for system inspection, network testing, data formatting, or object pipeline operations that are not available as direct Win32 tools.

`ps_run` uses PowerShell 5.1 by default (cold start ~1.5s). It enforces Constrained Language Mode (CLM) and an explicit cmdlet allowlist configured per role. It is not a general-purpose script runner — it is a controlled cmdlet executor.

## Tools

- `ps_run` — Execute a PowerShell cmdlet or pipeline. Subject to CLM and the `allow_cmdlets` list in the active role's `powershell_role` config. Maximum execution: 60 seconds.

## Patterns

**Allowed cmdlets in the `developer_ps` role:**
`Get-*`, `Select-Object`, `Where-Object`, `Sort-Object`, `Format-List`, `Format-Table`,
`Out-String`, `Test-NetConnection`, `Test-Path`, `Measure-Object`, `Group-Object`,
`Compare-Object`. Module: `NetTCPIP`.

**Read-only inspection — the primary use case:**
```
grille:ps_run
  script="Get-Process | Where-Object { $_.CPU -gt 10 } | Select-Object Name, CPU, Id"
```

**Network connectivity check:**
```
grille:ps_run
  script="Test-NetConnection -ComputerName <remote-host> -Port 5432"
```

**File system test:**
```
grille:ps_run
  script="Test-Path 'C:\\dev\\grille\\target\\release\\grille.exe'"
```

**Format output for readability:**
```
grille:ps_run
  script="Get-Service | Where-Object Status -eq 'Running' | Format-Table -AutoSize | Out-String"
```

## Constraints

- **Do not attempt cmdlets outside the allowlist.** CLM blocks .NET type access, COM objects, and XAML. Denied cmdlets return an error — they are not silently ignored. If you need a cmdlet that isn't allowed, use `grille:service_*` tools (for services), `grille:registry_*` tools (for registry), or `grille:process_run` (for executables) instead.
- **Do not use `ps_run` for service mutations (`Start-Service`, `Stop-Service`, `Restart-Service`).** These are explicitly denied in `deny_cmdlets`. Use `grille:service_start`, `grille:service_stop`, `grille:service_restart` instead.

⛔ **`Start-Service` and `Stop-Service` are blocked in ps_run. STOP — use `service_start` / `service_stop` / `service_restart` via the grille-services skill.**
- **Do not use `$` characters in passwords or secret values passed through ps_run.** PowerShell interpolates `$` in double-quoted strings. Use single-quoted strings, or better, use the secrets module to avoid passing credentials through PowerShell at all.
- **Do not invoke `az` without the `.cmd` extension.** The Azure CLI on Windows is installed as `az.cmd`. Without a shell, `az` alone fails to resolve. Use `grille:process_run executable="az.cmd"` for Azure CLI calls — not `ps_run`.
- **Do not expect `ps_run` to resolve `.cmd` or `.bat` extensions.** PowerShell via Grille does not invoke a shell. Batch files and `.cmd` scripts must be run via `process_run`, not `ps_run`.
- **Do not use `ps_run` for file writes.** CLM blocks `[System.IO.File]::WriteAllText(...)` and similar .NET patterns. Use `fs_write_file` or `fs_str_replace` for file writes.

⛔ **File writes via PowerShell .NET types are blocked by CLM. STOP — use `fs_write_file` or `fs_str_replace` via the grille-filesystem skill.**

## Examples

**Example 1: Find processes consuming significant memory**
```
grille:ps_run
  script="Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 Name, Id, @{N='MB';E={[math]::Round($_.WorkingSet64/1MB,1)}} | Format-Table"
```

**Example 2: Check if a TCP port is reachable before a SQL connection**
```
grille:ps_run
  script="Test-NetConnection -ComputerName <remote-host> -Port 5432 -InformationLevel Detailed"

→ TcpTestSucceeded: True = proceed with sql_query
→ TcpTestSucceeded: False = check Docker container status with docker_ps
```
