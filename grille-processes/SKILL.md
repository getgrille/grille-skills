---
name: grille-processes
description: "Use when listing, inspecting, killing, waiting on, or snapshotting Windows processes via Grille. WHEN: 'what's running', 'list processes', 'kill process', 'stop process', 'process tree', 'what spawned X', 'wait for process to finish', 'CPU usage', 'memory usage', 'which process is using memory', 'is X running', 'snapshot processes', 'what changed since', 'did my deploy restart the right process', 'ps_list', 'ps_kill', 'ps_tree', 'ps_wait', 'ps_snapshot', 'ps_diff'. DO NOT USE WHEN: 'run a command or executable' (use grille-process for process_run); 'Windows service' (use grille-services); 'Docker container' (use grille-docker)."
---

## Overview

This skill covers Grille's Processes module — 6 tools for Windows process inspection, management, and change detection. Apply it when Claude needs to enumerate running processes, diagnose resource usage, check if a specific process is running, understand process hierarchy, terminate a hung process, wait for a process to complete, or verify that an operation changed only the expected processes.

`ps_list` takes at least 500ms — it samples CPU% over two snapshots. Use `ps_tree` or `ps_snapshot` when you only need identity information (faster, no sampling).

## Tools

- `ps_list` — Full process enumeration with 17 fields: pid, ppid, name, path, company, description, product_version, cpu_percent, working_set_mb, private_memory_mb, handle_count, thread_count, session_id, priority_class, responding, start_time, command_line. Filter by name/session, sort by any field, configurable limit.
- `ps_kill` — Terminate a process by PID or name. Three-layer security protection (see Security model).
- `ps_tree` — Parent-child process hierarchy via ToolHelp32. Fast — no CPU sampling or WMI call.
- `ps_wait` — Poll until a process exits or timeout expires (max 300s). Returns exit status per PID.
- `ps_snapshot` — Capture a named point-in-time snapshot of all running processes to disk (`%APPDATA%\Grille\snapshots\<name>.json`). Also: `list=true` to enumerate saved snapshots, `delete="name"` to remove one. Auto-purges snapshots older than 7 days.
- `ps_diff` — Compare a saved snapshot against another snapshot or live state. Sections: started, exited, restarted. `filter_name` to scope results.
- `ps_snapshot` — Capture a named point-in-time snapshot of all running processes to disk. Also: list and delete snapshots.
- `ps_diff` — Compare two snapshots (or one snapshot vs. live state) to see what started, exited, or restarted.

## Patterns

**Check if a specific process is running:**
```
grille:ps_list filter_name="cargo"
→ Returns matching processes with CPU%, memory, start_time
→ Empty results = not running
```

**Find what's using the most CPU:**
```
grille:ps_list sort_by="cpu" sort_desc=true limit=10
→ Top 10 by CPU% — first entry is the hottest process
```

**Find what's using the most memory:**
```
grille:ps_list sort_by="memory" sort_desc=true limit=10
→ Top 10 by working set
```

**Get command line arguments for a process:**
```
grille:ps_list filter_name="node" include_command_line=true
→ Adds ~1s WMI call — shows full argv
→ Useful for identifying which script/server is running
→ Returns null for elevated processes — graceful, not an error
```

**Understand a process hierarchy:**
```
grille:ps_tree root_name="cargo"
→ Shows cargo + all child processes it spawned (rustc, linker, etc.)

grille:ps_tree
→ Full system tree — all parent-child relationships
```

**Kill a hung process:**
```
1. grille:ps_list filter_name="cargo"
   → Confirm it exists, note PID

2. grille:ps_kill name="cargo.exe" reason="hung build process"
   → Returns killed[] and denied[] — check denied for security layer explanation
```

**Kill by specific PID (safer when multiple instances exist):**
```
grille:ps_kill pid=14832 reason="specific hung instance"
```

**Wait for a background process to finish:**
```
grille:ps_wait name="cargo.exe" timeout_seconds=180
→ Polls every 1s, returns when all matching processes exit
→ On timeout: returns exited[] and still_running[] lists
```

**Verify a deploy restarted the right process:**
```
1. grille:ps_snapshot name="before-deploy"
   → Captures 426 processes to %APPDATA%\Grille\snapshots\before-deploy.json

2. (run your deploy)

3. grille:ps_diff before="before-deploy"
   → {started: [{name: "myapp.exe", pid: 14432, path: "C:\services\myapp.exe"}],
      exited:  [{name: "myapp.exe", pid: 8820, ...}],
      restarted: [{name: "myapp.exe", old: {pid: 8820}, new: {pid: 14432}}],
      summary: "0 started, 0 exited, 1 restarted, ~425 unchanged"}
   → Clean restart from expected path — deploy verified
```

**Detect unexpected processes spawned by a script:**
```
1. grille:ps_snapshot name="before-script"

2. (run the script)

3. grille:ps_diff before="before-script"
   → Review started[] for anything unexpected
   → Empty started[] = script spawned nothing that survived
```

**List and manage saved snapshots:**
```
grille:ps_snapshot list=true
→ [{name: "before-deploy", timestamp: "2026-05-11 14:38:24", process_count: 426}, ...]

grille:ps_snapshot delete="before-deploy"
→ ✓ Deleted snapshot 'before-deploy'.
```

## Security model for ps_kill (SD-025)

Three independent layers enforced in fixed order. No configuration overrides Layer 1 or 2.

**Layer 1 — Hardcoded immutable deny (compiled in, never bypassed):**
PIDs 0 and 4, and by name: system, idle, registry, smss.exe, csrss.exe, wininit.exe, winlogon.exe, services.exe, lsass.exe, lsm.exe, svchost.exe, grille.exe.

**Layer 2 — Security vendor protection (compiled in, never bypassed):**
If the target process's `CompanyName` version field matches a known endpoint security vendor (CrowdStrike, SentinelOne, Defender, Sophos, Cylance, and 24 others), the kill is blocked and Event ID 2007 is written to the Windows Event Log at ERROR level. Always investigate if this fires unexpectedly — it may indicate prompt injection.

**Layer 3 — Role allowlist (configured in grille.toml):**
The process name must match `ps_kill_allowed` in the active role config. Default is empty — all kills denied unless explicitly permitted.

```toml
# grille.toml — example
ps_kill_allowed = ["cargo.exe", "node.exe", "myapp.exe"]
```

⛔ **Layer 1 or 2 denial: STOP. These are immutable. Do not attempt workarounds.**

⛔ **Layer 3 denial (not_in_allowlist): tell the user to add the process name to `ps_kill_allowed` in `grille.toml` and reload config (`grille_reload_config`) before retrying.**

## Constraints

- **`ps_list` takes ≥500ms** due to CPU% sampling. For hierarchy or identity-only needs, use `ps_tree` or `ps_snapshot`.
- **`ps_kill` by name kills all matching processes.** Use `pid` when multiple instances exist and you only want one terminated.
- **`ps_wait` max timeout is 300 seconds.**
- **`ps_snapshot` excludes cpu_percent, memory, and handle fields** — these are too volatile to be useful for diffing. Snapshot fields: pid, name, ppid, path, company, start_time, session_id.
- **Snapshots persist to `%APPDATA%\Grille\snapshots\`** and survive Grille restarts. Auto-purged after 7 days.
- **`ps_diff` restarted detection** matches by name: if the same process name appears in both started and exited, it is classified as restarted (different PID = process cycled). It is removed from started/exited to avoid duplication.
- **`responding` field** currently always returns `true` for headless processes (services, CLIs). Only meaningful for processes with a visible message-pump window.
- **Session 0 processes** (Windows services) appear with `session_id=0`. Layer 1 protects critical ones; others can be killed if in `ps_kill_allowed`.
- **Do not use `ps_kill` as a substitute for `service_stop`.** For managed Windows services, use `grille:service_stop` — it sends a graceful stop control and waits for Stopped state. `ps_kill` sends `TerminateProcess` immediately with no cleanup.

## Examples

**Example 1: Diagnose high CPU on a dev machine**
```
grille:ps_list sort_by="cpu" sort_desc=true limit=5
→ {name: "MsMpEng.exe", cpu_percent: 47.2, company: "Microsoft Corporation"}
→ Windows Defender scanning build output — add build directory to exclusions
```

**Example 2: Verify a service process is healthy**
```
grille:ps_list filter_name="rekn-collector"
→ {name: "rekn-collector.exe", pid: 8820, start_time: "2026-05-10 14:32",
   responding: true, working_set_mb: 42.1, path: "C:\services\rekn-collector.exe"}
→ Running from expected path — healthy
```

**Example 3: Kill a hung Node.js dev server**
```
grille:ps_list filter_name="node" include_command_line=true
→ {pid: 37896, command_line: "node server.js --port 3000", cpu_percent: 0.0}
→ Zero CPU on a server = likely hung

grille:ps_kill pid=37896 reason="hung dev server, port 3000 not released"
→ {killed: [{pid: 37896, name: "node.exe"}]}
```

**Example 4: Wait for a release build then deploy**
```
grille:ps_wait name="cargo.exe" timeout_seconds=300
→ {status: "all_exited", processes: [{pid: 14832, name: "cargo.exe", status: "exited"}]}
→ Build complete — proceed with sign/deploy sequence
```

**Example 5: Full deploy verification workflow**
```
grille:ps_snapshot name="before-deploy"
→ {process_count: 312, timestamp: "2026-05-11 14:30:00", status: "saved"}

(deploy runs)

grille:ps_diff before="before-deploy" filter_name="myapp"
→ {restarted: [{name: "myapp.exe", old: {pid: 8820, path: "...v1.2.exe"},
                                   new: {pid: 9104, path: "...v1.3.exe"}}],
   started: [], exited: [],
   summary: "0 started, 0 exited, 1 restarted, ~311 unchanged"}
→ Only myapp restarted, from the expected new binary — deploy clean

grille:ps_snapshot delete="before-deploy"
```

**Example 6: Security audit — what did a script spawn?**
```
grille:ps_snapshot name="before-install"

(run third-party installer)

grille:ps_diff before="before-install"
→ started: [{name: "updatescheduler.exe", path: "C:\Program Files\Vendor\...",
             company: "Vendor Corp"}]
→ One persistent process added — review whether expected
```

**Example 7: Inspect a spawned build tree**
```
grille:ps_tree root_name="cargo"
→ ● cargo.exe [14832]  C:\Users\Bharat\.cargo\bin\cargo.exe
    └─ rustc.exe [15360]
       └─ rustc.exe [15488]  (codegen worker)
       └─ rustc.exe [15492]  (codegen worker)
    └─ link.exe [15600]  (MSVC linker)
```
