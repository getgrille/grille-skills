---
name: grille-security
description: "Use when evaluating Grille's security model, understanding permission boundaries, or assessing Grille for enterprise deployment. WHEN: 'how does Grille handle security', 'what can Claude not do', 'security model', 'prompt injection', 'audit guarantees', 'enterprise evaluation'. DO NOT USE WHEN: you need to use a specific tool — load the relevant module skill instead."
---

## Overview

Grille is a security-first MCP server for Windows. Its central invariant is:

> **The human controls the permission boundary. Claude operates within it. Claude never expands it.**

This skill is written for enterprise security evaluators and security-conscious developers. For usage patterns, see the individual module skills. For deep-dive detail, see the `references/` subdirectory.

---

## The Permission Model

Grille uses declarative, fail-closed RBAC. Every capability is disabled by default. Claude sees only tools permitted by the active role in `grille.toml` — a file the user authors and owns.

RBAC evaluation order (first match wins): module enabled? → hardcoded global deny → explicit deny rule → explicit allow rule → **DENY** (fail closed). See [`references/threat-model.md`](./references/threat-model.md) for the full evaluation sequence and prompt injection analysis.

---

## Hardcoded Protections

These are compiled into the binary. They cannot be configured away, overridden by `grille.toml`, or bypassed by any role.

**Filesystem:**
- `C:\Windows` and all system directories: hardcoded deny
- Bare drive roots (`C:\`, `D:\`): hardcoded deny
- All paths canonicalized and symlink-resolved before allowlist checks

**Process execution:**
- LOLBin deny list: `cmd.exe`, `powershell.exe` (5.1), `wscript.exe`, `cscript.exe`, `mshta.exe`, `regsvr32.exe`, `rundll32.exe`, `certutil.exe`, `msiexec.exe`
- No shell intermediary; arguments as array (no injection surface)

**Registry:**
- `HKLM\SAM`, `HKLM\SECURITY`, `HKLM\SYSTEM\...\LSA`, and related credential/policy keys: hardcoded deny on read and write
- HKLM writes: denied in code for all roles

**PowerShell:**
- Constrained Language Mode enforced by default: blocks .NET type access, COM objects, XAML

**Process kill (planned `ps_kill`):**
- `MsMpEng.exe`, `lsass.exe`, `csrss.exe`, `SecurityHealthService.exe`, `SenseIr.exe`, and other endpoint security processes: hardcoded deny
- **Grille cannot be used by Claude to disable your endpoint security.** This is a binary guarantee, not a policy.

---

## Self-Elevation Is Impossible

- `grille_set_role` does not exist — rejected by design (SD-001)
- `grille_reload_config` reads disk only — never writes `grille.toml`
- Error messages never enumerate configured resources — injection cannot use error responses for reconnaissance (SD-015)

---

## Secrets Privacy Contract

A secret never appears in Claude's conversation context, the MCP payload, or any log. Five enforcement mechanisms: pre-resolution allowlist check, `Zeroizing<Vec<u8>>` storage, post-execution output scan, `#[serde(skip)]` on redact entries, and audit log records refs only. See [`references/secrets-privacy.md`](./references/secrets-privacy.md) for the full contract and provider details.

---

## Audit Trail

Every tool call logged locally at `%APPDATA%\Grille\logs\`. All denials logged. `ps_kill` denials (future) emit to both Grille audit log and Windows Event Log for SIEM detection. Retention is not tier-gated — same design at every license tier (SD-020). Not transmitted anywhere.

---

## Binary and Runtime Properties

Single compiled Rust binary. ~8MB idle. ~20ms startup. No Node.js, no Python, no package execution at runtime. Ed25519 license verification is offline for trial and perpetual keys. See [`references/binary-properties.md`](./references/binary-properties.md) for full runtime properties and Windows Event Log emission table.

---

## What Grille Does Not Guarantee

- Network egress via Anthropic's API itself — the "Claudy Day" pattern bypasses any local control
- Malicious tool chaining within permitted scope — Grille enforces boundaries, not intent
- Post-incident prevention — the audit log supports forensics, not prevention of within-scope actions
