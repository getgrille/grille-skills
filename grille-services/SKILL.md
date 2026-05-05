---
name: grille-services
description: "Use when listing, inspecting, starting, stopping, or restarting Windows services via Grille. WHEN: 'service status', 'start service', 'stop service', 'restart service', 'is service running', 'service_list', 'service_get'. DO NOT USE WHEN: 'run PowerShell Get-Service' (use grille-powershell for read-only PS queries); 'Docker container' (use grille-docker)."
---

## Overview

This skill covers Grille's Windows Services module — 5 tools for service lifecycle management. Apply it when Claude needs to check service state, enumerate running services, or control service lifecycle as part of a deployment, diagnostic, or maintenance workflow.

Service mutations (`service_start`, `service_stop`, `service_restart`) require Windows administrator elevation. Grille returns a clear error if the current process lacks sufficient privilege — it does not silently fail.

## Tools

- `service_list` — Enumerate Windows services. Supports filtering by name pattern and status.
- `service_get` — Get detailed status for a single service: state, startup type, binary path, dependencies.
- `service_start` — Start a stopped service. Requires admin elevation.
- `service_stop` — Stop a running service. Requires admin elevation.
- `service_restart` — Stop then start a service. Requires admin elevation.

## Patterns

**Always check state before mutation:**
```
1. grille:service_get service="MyApp"
   → Confirm current state (Running / Stopped / Pending)
2. Only then call service_start / service_stop / service_restart
```

**List services by status:**
```
grille:service_list status="Running"
grille:service_list status="Stopped"
grille:service_list name_pattern="grille*"   ← wildcard filter
```

**Restart a service after a config change:**
```
1. grille:service_get service="MyApp"
   → Confirm it is Running

2. grille:service_restart service="MyApp"

3. grille:service_get service="MyApp"
   → Confirm it returned to Running state
```

**Diagnose a service that won't start:**
```
1. grille:service_get service="MyApp"
   → Check binary path — does the executable exist?
   → Check dependencies — are required services running?

2. grille:eventlog_query source="MyApp" hours=1
   → Read Windows Event Log for startup errors

3. grille:fs_stat path="<binary path from service_get>"
   → Confirm the executable is present
```

## Constraints

- **Do not attempt `service_start/stop/restart` if Grille is not running with administrator privileges.** These calls require SeServiceLogonRight or equivalent. Grille returns a clear, actionable error message — it does not partially complete. If elevation is needed, the operation must be performed manually or via an elevated terminal.

⛔ **Service mutations require admin elevation. If Grille is not elevated, STOP — the user must perform this manually from an elevated terminal.**
- **Do not attempt to stop security services.** Grille's immutable deny list blocks stopping endpoint security processes (Windows Defender, Sense, etc.) regardless of configuration or role. These denials are logged to both the Grille audit log and Windows Event Log.

⛔ **Stopping Defender, lsass, csrss, or other security processes is a hardcoded denial. It will fail, be logged, and emit a Windows Event Log entry. Do not attempt it.**
- **Do not assume a service name — verify with `service_list` first.** Service display names and service names differ (e.g., display name "Windows Update" vs service name "wuauserv"). Use `service_list name_pattern=` to find the exact name before calling `service_get`.
- **`service_stop` is graceful.** It sends a stop control and waits for the service to reach the Stopped state. Services with long shutdown routines may take several seconds. If the stop times out, the error message indicates the remaining state.

## Examples

**Example 1: Verify a service started correctly after deployment**
```
grille:service_get service="GrilleWebhook"
→ {state: "Running", startup_type: "Automatic", pid: 12345}
→ Confirmed running — no action needed
```

**Example 2: Restart a service and verify recovery**
```
grille:service_get service="nginx"
→ state: "Stopped"  ← unexpected

grille:eventlog_query source="nginx" hours=2
→ Read crash reason

grille:service_start service="nginx"

grille:service_get service="nginx"
→ state: "Running"  ← recovered
```
