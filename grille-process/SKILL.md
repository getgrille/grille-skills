---
name: grille-process
description: "Use when running executables, build tools, or CLI commands on Windows via Grille process_run. WHEN: 'run cargo build', 'run git', 'execute xcopy', 'run npm', 'compile', 'build the project', 'run a script'. DO NOT USE WHEN: 'run PowerShell', 'Get-*', 'run cmdlet' (use grille-powershell); 'run inside container' (use grille-docker)."
---

## Overview

This skill covers `process_run` — Grille's tool for spawning Windows executables directly, without a shell intermediary. Apply it whenever Claude needs to run build tools, version control commands, deployment scripts, or any allowlisted binary on the local machine.

`process_run` is distinct from `ps_run` (PowerShell cmdlets) and `docker_exec` (commands inside containers). Use this tool for native Windows executables.

## Tools

- `process_run` — Execute an allowlisted Windows executable with arguments. Captures stdout and stderr. 30-second default timeout, 300-second maximum. Output capped at 10 MB. **Python stdout is automatically decoded as UTF-8** — no `sys.stdout.reconfigure()` needed in inline `-c` scripts.

## Patterns

**Always include `.exe` (or `.cmd`) in the executable name:**
```
✅ "cargo.exe"
✅ "git.exe"
✅ "npm.cmd"
✅ "az.cmd"
❌ "cargo"    ← will fail — no shell to resolve the extension
❌ "git"      ← will fail
```

**Standard build workflow:**
```
grille:process_run
  executable="cargo.exe"
  args=["build", "--release"]
  working_dir="C:\\dev\\myapp"
```

**Git operations:**
```
grille:process_run
  executable="git.exe"
  args=["log", "--oneline", "-10"]
  working_dir="C:\\dev\\myapp"
```
Git for Windows deadlock prevention is baked into Grille internally (`stdin: null`
prevents MSYS2 PTY initialization from hanging). No action needed by Claude.

**Deploy — copy built binary:**
```
grille:process_run
  executable="xcopy.exe"
  args=["C:\\dev\\myapp\\target\\release\\myapp.exe",
        "C:\\Users\\<user>\\AppData\\Local\\MyApp\\", "/Y"]
```

## Constraints

- **Do not attempt to use `cmd.exe`, `powershell.exe` (5.1), `wscript.exe`, `cscript.exe`, `mshta.exe`, `regsvr32.exe`, `rundll32.exe`, `certutil.exe`, or `msiexec.exe`.** These LOLBins are hardcoded denials in Grille — they cannot be unblocked by any configuration. Use `ps_run` for PowerShell 7 (`pwsh.exe`) cmdlets instead.
- **Do not omit the `.exe` or `.cmd` extension.** Grille spawns executables via `tokio::process::Command` without a shell. Shell extension resolution (`.cmd`, `.bat`, `.ps1`) does not apply. The extension must be explicit.
- **Do not attempt `git push` via Grille.** Git push requires SSH agent access, which is inaccessible from a Grille subprocess context. Push manually from a terminal.

⛔ **`git push` cannot work via Grille. STOP — push manually from a terminal window.**
- **Do not use `process_run` for PowerShell cmdlets.** Use `ps_run` (the powershell module). `process_run` has `powershell.exe` on its LOLBin deny list.

⛔ **For PowerShell cmdlets, STOP — use `ps_run` via the grille-powershell skill. Calling `process_run` with `powershell.exe` is a hardcoded denial and will always fail.**
- **Executable must be in `process_allowed` in `grille.toml`.** If a binary is not in the allowlist, the call is denied and logged. Check `grille_info` or `grille.toml` to see the current allowlist.
- **Arguments are passed as a list, never shell-concatenated.** Do not construct shell pipelines or use shell metacharacters (`|`, `>`, `&&`). Each argument is a discrete array element.

## Examples

**Example 1: Build and check for errors**
```
grille:process_run
  executable="cargo.exe"
  args=["build", "--release"]
  working_dir="C:\\dev\\myapp"

→ Check stdout/stderr for "error[" lines
→ On success: Finished release in Xs
```

**Example 2: Check git status before a commit**
```
grille:process_run
  executable="git.exe"
  args=["status", "--short"]
  working_dir="C:\\dev\\myapp"

→ Review modified/untracked files
→ Follow with: git.exe add / git.exe commit
→ Push manually from terminal (SSH agent not accessible via Grille)
```
