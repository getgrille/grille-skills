# grille-skills

**14 skills for every Grille module. Works with Claude Code, Codex CLI, Cursor, Gemini CLI, and Windsurf.**

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
| [grille-filesystem](./grille-filesystem/SKILL.md) | "read file", "write file", "edit file", "search codebase", "find file", "list directory", "copy file", "delete file", "diff", "append to" | `fs_read_file`, `fs_write_file`, `fs_str_replace`, `fs_append_file`, `fs_list_directory`, `fs_find`, `fs_copy_file`, `fs_stat`, `fs_diff`, `fs_create_directory`, `fs_delete_file`, `fs_delete_directory` |
| [grille-process](./grille-process/SKILL.md) | "run cargo build", "run git", "execute xcopy", "run npm", "compile", "build the project" | `process_run` |
| [grille-powershell](./grille-powershell/SKILL.md) | "run Get-Process", "run cmdlet", "Test-NetConnection", "Get-Service", "ps_run" | `ps_run` |
| [grille-docker](./grille-docker/SKILL.md) | "docker ps", "start container", "stop container", "docker logs", "compose up", "docker exec" | `docker_ps`, `docker_logs`, `docker_inspect`, `docker_images`, `docker_stats`, `docker_start`, `docker_stop`, `docker_compose_*`, `docker_exec` |
| [grille-sql](./grille-sql/SKILL.md) | "sql query", "select from", "insert into", "run migration", "describe table", "sql transaction" | `sql_query`, `sql_execute`, `sql_begin`, `sql_commit`, `sql_rollback`, `sql_list_tables`, `sql_describe` |
| [grille-secrets](./grille-secrets/SKILL.md) | "secret not resolving", "configure AKV", "secret_ref", "{{secret:}}", "credential error" | Transparent (no direct tools — resolved via `{{secret:}}` refs) |
| [grille-audit](./grille-audit/SKILL.md) | "did that write succeed", "check audit log", "what did Claude do", "any errors", "verify the action" | `grille_audit` |
| [grille-services](./grille-services/SKILL.md) | "service status", "start service", "stop service", "restart service", "is service running" | `service_list`, `service_get`, `service_start`, `service_stop`, `service_restart` |
| [grille-remote](./grille-remote/SKILL.md) | "ssh_run", "remote_copy", "deploy to server", "run on remote machine", "copy file to remote" | `ssh_run`, `remote_copy` |
| [grille-registry](./grille-registry/SKILL.md) | "read registry", "write registry", "HKCU", "HKLM", "registry_read_value" | `registry_read_value`, `registry_list_key`, `registry_set_value`, `registry_delete_value` |
| [grille-eventlog](./grille-eventlog/SKILL.md) | "event log", "windows events", "check for errors in event log", "Grille security events" | `eventlog_query` |
| [grille-security](./grille-security/SKILL.md) | "security model", "what can Claude not do", "prompt injection", "audit guarantees", "enterprise evaluation" | Reference skill — no direct tools |
| [grille-windows-env](./grille-windows-env/SKILL.md) | executable not found, CRLF mismatch, `fs_str_replace` failing, `$` mangling, `grille_reload_config` not working | Cross-cutting — applies to all modules |

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
