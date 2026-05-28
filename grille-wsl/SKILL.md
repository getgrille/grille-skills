---
name: grille-wsl
description: "Use when running commands inside a WSL (Windows Subsystem for Linux) distribution. WHEN: 'run in WSL', 'run in Linux', 'run in Ubuntu', 'wsl_run', 'check my WSL environment', 'run a Linux command', 'run python in WSL', 'run cargo in WSL distro'. DO NOT USE WHEN: running Windows executables locally (use grille-process); running commands on remote SSH hosts (use grille-ssh); PowerShell on the local machine (use grille-powershell)."
---

## Overview

This skill covers Grille's wsl module — run allowlisted commands inside a WSL distribution on the local Windows machine. Apply it when Claude needs to execute Linux-native commands in a WSL environment.

The command is spawned as:
```
wsl.exe [--distribution <distro>] [--cd <working_dir>] -- <command> [args...]
```

No shell intermediary — arguments are passed directly. Shell operators (`|`, `&&`, `;`) are not supported in a single `wsl_run` call. For pipelines, use multiple calls where the output of one informs the next, or run a self-contained script file.

Allowed commands are configured per-role in `grille.toml` under `wsl_allowed_commands`. The first token of `command` must match an entry in that list.

**WSL path note:** `working_dir` is a Linux path inside the WSL namespace (e.g. `/home/user/project`). Windows `fs_allowed_paths` do not apply here.

## Tools

- `wsl_run` — Run an allowed command inside a WSL distribution. Returns stdout and stderr. Non-zero exit code is surfaced as an error.

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `command` | ✅ Yes | Executable to run, e.g. `ls`, `python3`, `cargo`. Must be in `wsl_allowed_commands`. |
| `args` | No | Arguments as an array, e.g. `["-la", "/home/user"]`. |
| `distro` | No | WSL distribution name, e.g. `Ubuntu`, `Debian`. Omit for the default distribution. |
| `working_dir` | No | Linux working directory, e.g. `/home/user/project`. |

## Patterns

**List files in a WSL directory:**
```
grille:wsl_run
  command="ls"
  args=["-la", "/home/user/project"]
```

**Run a Python script in WSL:**
```
grille:wsl_run
  command="python3"
  args=["analyze.py"]
  working_dir="/home/user/scripts"
```

**Run a command in a specific distribution:**
```
grille:wsl_run
  command="uname"
  args=["-a"]
  distro="Ubuntu"
```

**Check what distributions are installed (via wsl.exe from process_run):**
If wsl_run itself is failing and you need to diagnose WSL availability, use
`process_run` with `executable="wsl.exe"` and `args=["--list", "--verbose"]`
(requires `wsl.exe` in `process_allowed`).

## Configuration

```toml
[roles.developer]
modules = ["wsl", ...]

[roles.developer]
wsl_allowed_commands = [
    "ls",
    "cat",
    "python3",
    "cargo",
    "git",
    "bash",      # only if you need full shell access — be deliberate
]
```

**Tip:** Keep the allowlist minimal. If you only need to run a specific script, allow `python3` or `bash` rather than every possible command. `bash` gives broader access — only add it if your use case requires it.

## Output limits

- **stdout:** 1 MB cap (truncated with note if exceeded)
- **stderr:** 4 KB cap
- **Timeout:** 120 seconds

## Error patterns

| Error message | Cause | Fix |
|---|---|---|
| `Not in the wsl_allowed_commands list` | Command not in config allowlist | Add the command to `wsl_allowed_commands` in grille.toml, then `grille_reload_config` |
| `wsl.exe not found` | WSL not installed | Run `wsl --install` in an elevated terminal |
| Non-zero exit code | Command failed inside WSL | Check stderr output in the error body |
