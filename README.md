# grille-skills

**17 skills for every Grille module. Works with Claude Code, Codex CLI, Cursor, Gemini CLI, and Windsurf.**

<!-- mcp-name: io.github.getgrille/grille -->

---

Azure MCP manages your subscription. Grille manages your machine.

Grille is a security-first MCP server for Windows — local filesystem, processes, Docker, SQL, secrets, services, registry, and remote execution, all under declarative RBAC with a full audit trail. This repository contains the SKILL.md plugin for the open skills standard, so Claude and other agents understand exactly how to use every Grille module.

---

## Install

**Claude Code:**
```bash
npx skills add getgrille/grille-skills
```

**Cursor, Gemini CLI, Windsurf, Codex CLI:**
Add this repository to your skills configuration. All agents that support the open SKILL.md standard (released Dec 2025) can load these files directly.

**Manual:**
Clone this repo and point your agent's skills path at the root directory. Each subdirectory contains a single `SKILL.md` file named for its module.

---

## Skills

| Skill | Trigger | Tools Covered |
|-------|---------|---------------|
| [grille-filesystem](./grille-filesystem/SKILL.md) | "read file", "write file", "edit file", "search codebase", "find file", "search in file", "list directory", "copy file", "delete file", "diff", "append to" | `fs_read_file`, `fs_write_file`, `fs_str_replace`, `fs_append_file`, `fs_list_directory`, `fs_find`, `fs_search`, `fs_copy_file`, `fs_stat`, `fs_diff`, `fs_create_directory`, `fs_delete_file`, `fs_delete_directory` |
| [grille-process](./grille-process/SKILL.md) | "run cargo build", "run git", "execute xcopy", "run npm", "compile", "build the project" | `process_run` |
| [grille-powershell](./grille-powershell/SKILL.md) | "run Get-Process", "run cmdlet", "Test-NetConnection", "Get-Service", "ps_run" | `ps_run` |
| [grille-docker](./grille-docker/SKILL.md) | "docker ps", "start container", "stop container", "docker logs", "compose up", "docker exec" | `docker_ps`, `docker_logs`, `docker_inspect`, `docker_images`, `docker_stats`, `docker_start`, `docker_stop`, `docker_compose_*`, `docker_exec` |
| [grille-sql](./grille-sql/SKILL.md) | "sql query", "select from", "insert into", "run migration", "describe table", "sql transaction" | `sql_query`, `sql_execute`, `sql_begin`, `sql_commit`, `sql_rollback`, `sql_list_tables`, `sql_describe` |
| [grille-secrets](./grille-secrets/SKILL.md) | "secret not resolving", "configure AKV", "secret_ref", "{{secret:}}", "credential error" | Transparent (no direct tools — resolved via `{{secret:}}` refs) |
| [grille-system](./grille-system/SKILL.md) | "start of session", "is Grille healthy", "check system state", "OS version", "disk space", "RAM usage", "any errors this session", "grille_diagnose", "grille_health", "grille_info", "grille_session_stats" | `grille_diagnose`, `grille_info`, `grille_health`, `grille_session_stats`, `grille_reload_config` |
| [grille-sysinfo](./grille-sysinfo/SKILL.md) | "OS version", "uptime", "disk space", "RAM", "installed software", "sys_summary", "sys_drives", "sys_software", "check disk before build" | `sys_summary`, `sys_drives`, `sys_software` |
| [grille-audit](./grille-audit/SKILL.md) | "did that write succeed", "check audit log", "what did Claude do", "any errors", "verify the action" | `grille_audit` |
| [grille-services](./grille-services/SKILL.md) | "service status", "start service", "stop service", "restart service", "is service running" | `service_list`, `service_get`, `service_start`, `service_stop`, `service_restart` |
| [grille-remote](./grille-remote/SKILL.md) | "ssh_run", "remote_copy", "deploy to server", "run on remote machine", "copy file to remote" | `ssh_run`, `remote_copy` |
| [grille-registry](./grille-registry/SKILL.md) | "read registry", "write registry", "HKCU", "HKLM", "registry_read_value" | `registry_read_value`, `registry_list_key`, `registry_set_value`, `registry_delete_value` |
| [grille-eventlog](./grille-eventlog/SKILL.md) | "event log", "windows events", "check for errors in event log", "Grille security events" | `eventlog_query` |
| [grille-processes](./grille-processes/SKILL.md) | "what's running", "list processes", "kill process", "process tree", "wait for process", "CPU usage", "memory usage", "what spawned this" | `ps_list`, `ps_kill`, `ps_tree`, `ps_wait` |
| [grille-git](./grille-git/SKILL.md) | "git status", "commit history", "git log", "show diff", "commit changes", "switch branch", "fetch from remote", "stash changes" | `git_status`, `git_log`, `git_diff`, `git_branches`, `git_show`, `git_stash_list`, `git_commit`, `git_checkout`, `git_stash`, `git_stash_pop`, `git_fetch` |
| [grille-networking](./grille-networking/SKILL.md) | "what process is holding port", "is port open", "check if port is open", "my IP address", "ping host", "is host reachable", "DNS lookup", "resolve hostname", "active connections", "network adapters", "is the API up", "check health endpoint", "query REST API on localhost", "who owns this domain", "when does this domain expire", "domain registration", "whois", "nameservers" | `net_connections`, `net_adapters`, `net_ping`, `net_dns_lookup`, `net_port_check`, `net_http_get`, `net_whois` |
| [grille-security](./grille-security/SKILL.md) | "security model", "what can Claude not do", "prompt injection", "audit guarantees", "enterprise evaluation" | Reference skill — no direct tools |
| [grille-windows-env](./grille-windows-env/SKILL.md) | executable not found, CRLF mismatch, `fs_str_replace` failing, `$` mangling, `grille_reload_config` not working | Cross-cutting — applies to all modules |

---

## Tool Reference

Every Grille tool exposes a human-readable title in the MCP manifest for display in Claude Desktop and other clients.

| Tool | Title |
|------|-------|
| `fs_read_file` | Grille · Filesystem · Read File |
| `fs_write_file` | Grille · Filesystem · Write File |
| `fs_str_replace` | Grille · Filesystem · Surgical Edit |
| `fs_append_file` | Grille · Filesystem · Append File |
| `fs_list_directory` | Grille · Filesystem · List Directory |
| `fs_find` | Grille · Filesystem · Find Files |
| `fs_copy_file` | Grille · Filesystem · Copy File |
| `fs_stat` | Grille · Filesystem · File Info |
| `fs_diff` | Grille · Filesystem · Diff |
| `fs_create_directory` | Grille · Filesystem · Create Directory |
| `fs_delete_file` | Grille · Filesystem · Delete File |
| `fs_delete_directory` | Grille · Filesystem · Delete Directory |
| `fs_search` | Grille · Filesystem · Search in File |
| `process_run` | Grille · Process · Run Executable |
| `ps_run` | Grille · PowerShell · Run Cmdlet |
| `docker_ps` | Grille · Docker · List Containers |
| `docker_logs` | Grille · Docker · Container Logs |
| `docker_inspect` | Grille · Docker · Inspect Container |
| `docker_images` | Grille · Docker · List Images |
| `docker_stats` | Grille · Docker · Container Stats |
| `docker_start` | Grille · Docker · Start Container |
| `docker_stop` | Grille · Docker · Stop Container |
| `docker_exec` | Grille · Docker · Exec in Container |
| `docker_compose_ps` | Grille · Docker · Compose Status |
| `docker_compose_up` | Grille · Docker · Compose Up |
| `docker_compose_down` | Grille · Docker · Compose Down |
| `docker_compose_restart` | Grille · Docker · Compose Restart |
| `sql_query` | Grille · SQL · Query |
| `sql_execute` | Grille · SQL · Execute |
| `sql_begin` | Grille · SQL · Begin Transaction |
| `sql_commit` | Grille · SQL · Commit |
| `sql_rollback` | Grille · SQL · Rollback |
| `sql_list_tables` | Grille · SQL · List Tables |
| `sql_describe` | Grille · SQL · Describe Table |
| `service_list` | Grille · Services · List |
| `service_get` | Grille · Services · Get Status |
| `service_start` | Grille · Services · Start |
| `service_stop` | Grille · Services · Stop |
| `service_restart` | Grille · Services · Restart |
| `ssh_run` | Grille · Remote · SSH Run |
| `remote_copy` | Grille · Remote · Copy File |
| `registry_list_key` | Grille · Registry · List Key |
| `registry_read_value` | Grille · Registry · Read Value |
| `registry_set_value` | Grille · Registry · Set Value |
| `registry_delete_value` | Grille · Registry · Delete Value |
| `eventlog_query` | Grille · Event Log · Query |
| `ps_list` | Grille · Processes · List |
| `ps_kill` | Grille · Processes · Kill |
| `ps_tree` | Grille · Processes · Tree |
| `ps_wait` | Grille · Processes · Wait |
| `ps_snapshot` | Grille · Processes · Snapshot |
| `ps_diff` | Grille · Processes · Diff |
| `git_status` | Grille · Git · Status |
| `git_log` | Grille · Git · Log |
| `git_diff` | Grille · Git · Diff |
| `git_branches` | Grille · Git · Branches |
| `git_show` | Grille · Git · Show |
| `git_stash_list` | Grille · Git · Stash List |
| `git_commit` | Grille · Git · Commit |
| `git_checkout` | Grille · Git · Checkout |
| `git_stash` | Grille · Git · Stash |
| `git_stash_pop` | Grille · Git · Stash Pop |
| `git_fetch` | Grille · Git · Fetch |
| `net_connections` | Grille · Networking · Connections |
| `net_adapters` | Grille · Networking · Adapters |
| `net_ping` | Grille · Networking · Ping |
| `net_dns_lookup` | Grille · Networking · DNS Lookup |
| `net_port_check` | Grille · Networking · Port Check |
| `net_http_get` | Grille · Networking · HTTP GET |
| `net_whois` | Grille · Networking · WHOIS |
| `sys_summary` | Grille · System Info · Summary |
| `sys_drives` | Grille · System Info · Drives |
| `sys_software` | Grille · System Info · Installed Software |
| `grille_info` | Grille · System · Info |
| `grille_diagnose` | Grille · System · Diagnose |
| `grille_health` | Grille · System · Health |
| `grille_audit` | Grille · System · Audit Log |
| `grille_reload_config` | Grille · System · Reload Config |
| `grille_session_stats` | Grille · System · Session Stats |

---

## What Grille Covers

Grille is the local-machine layer. It handles everything that runs on your Windows machine:

- **Filesystem** — read, write, edit, search, diff, copy, soft-delete
- **Process execution** — allowlisted executables, no shell intermediary, LOLBin protection
- **PowerShell** — Constrained Language Mode, cmdlet allowlist, no sandbox escape
- **Docker** — container lifecycle, Compose stacks, exec with per-container allowlists
- **SQL** — PostgreSQL and SQLite, transactions, row-safety limits
- **Secrets** — Windows Credential Manager (DPAPI) and Azure Key Vault, never in Claude's context
- **Audit** — local JSONL log of every tool call, including all denials
- **Windows Services** — start, stop, restart with hardcoded endpoint-security deny list
- **Remote execution** — SSH commands and SCP file transfer to named hosts
- **Registry** — read/write within scoped key allowlists, HKLM writes always denied
- **Windows Event Log** — query system, application, and security channels
- **Processes** — list all running processes with 17 fields (CPU%, memory, vendor, path, command line), kill with three-layer security protection, process tree, wait for exit, snapshot and diff
- **Git** — structured git operations via libgit2: status, log, diff, branches, show, commit, checkout, stash, fetch. 3-12× more token-efficient than `process_run` + `git.exe`.
- **Networking** — active TCP/UDP connections with PID mapping, network adapters, ICMP ping, DNS lookup, TCP port check, allowlist-gated HTTP GET with secret-ref header support, RDAP/WHOIS domain registration lookup via IANA bootstrap. Native Windows iphlpapi, no elevation required.
- **System info** — OS display version (via RtlGetVersion), uptime, RAM usage, per-drive disk space, installed software inventory. Native Win32 APIs — no WMI, no subprocess, no elevation.
- **Diagnostics** — `grille_diagnose` consolidated health snapshot (server, OS, RAM, drives, session errors, Windows Event Log errors in one call); `grille_info`, `grille_health`, `grille_session_stats`. Always available — no module required.

---

## Security Model

Grille is fail-closed. Every capability is disabled by default. Claude sees only tools permitted by the active role in `grille.toml` — a file the user owns.

Key guarantees baked into the binary (not policy — not configurable):
- Claude cannot change its own role or expand its own permissions
- LOLBins (`cmd.exe`, `powershell.exe` 5.1, etc.) are hardcoded denials
- Windows security processes (Defender, lsass, csrss, etc.) cannot be killed
- HKLM registry writes are always denied
- Windows system directories are always denied
- Secrets never appear in Claude's context, the MCP payload, or any log

See [grille-security](./grille-security/SKILL.md) for the full security model, or [getgrille.com/docs/security](https://getgrille.com/docs/security) for the public security reference.

---

## Open Standard Compatibility

These SKILL.md files follow the open skills standard released December 2025. They work with any agent platform that supports the standard:

- ✅ Claude Code
- ✅ Codex CLI
- ✅ Cursor
- ✅ Gemini CLI
- ✅ Windsurf

---

## Documentation

Full documentation at **[getgrille.com/docs](https://getgrille.com/docs)**

- [Installation guide](https://getgrille.com/docs/install)
- [Configuration reference](https://getgrille.com/docs/configuration)
- [Security reference](https://getgrille.com/docs/security)
- [Module reference](https://getgrille.com/docs/modules)

---

## License

Grille is proprietary software. These SKILL.md files are released under MIT for broad compatibility with the skills ecosystem.

Copyright © 2026 Grille Software. All rights reserved.
