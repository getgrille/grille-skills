---
name: grille-windows-env
description: "Use when working with Grille on Windows to understand cross-cutting environmental constraints. WHEN: executable not found, CRLF mismatch, fs_str_replace failing, dollar sign mangling, grille_reload_config not working, two-filesystem confusion. DO NOT USE WHEN: you need module-specific tool guidance — load the relevant module skill instead."
---

## Overview

This skill documents Windows-specific environmental truths that apply across all Grille modules. Apply it proactively — these constraints are not module-specific, and forgetting them causes silent failures. For full detail on executable naming and CRLF patterns, see `references/`.

---

## Executables: Always Use the Extension

Grille spawns executables without a shell — `.exe` and `.cmd` do not resolve automatically.

Quick reference: `cargo.exe`, `git.exe`, `npm.cmd`, `az.cmd`, `xcopy.exe`, `docker.exe`, `node.exe`, `python.exe`.

⛔ **`az` and `az.exe` both fail — the Azure CLI is `az.cmd`. This error is silently swallowed by the auth chain.** See [`references/executable-naming.md`](./references/executable-naming.md) for the full table.

---

## CRLF: Use Single-Line Anchors in `fs_str_replace`

Windows files use CRLF (`\r\n`). `fs_str_replace` fails when `old_str` spans line boundaries in a CRLF file. Use a unique single-line anchor — never span a line boundary.

⛔ **Multi-line `old_str` on Windows files will silently fail to match. Use a unique single-line string.** See [`references/crlf-patterns.md`](./references/crlf-patterns.md) for the full pattern including append-only document strategy.

---

## Dollar Signs in Passwords

`$` is interpolated by both PowerShell and Docker Compose. Use alphanumeric-only passwords in all demo and dev environments.

---

## Key File Paths

| Resource | Path |
|----------|------|
| Binary (MSIX install) | `C:\Users\<user>\AppData\Local\Grille\grille.exe` |
| Config | `C:\Users\<user>\AppData\Roaming\Grille\grille.toml` |
| Audit logs | `C:\Users\<user>\AppData\Roaming\Grille\logs\` |
| Source | `C:\dev\<project>\` |

---

## `grille_reload_config`: What It Does and Does Not Do

Hot-swaps existing config values from disk. Does **not** register new SQL connections, new secrets providers, or new remote hosts — those require a full Claude Desktop restart.

The "no changes detected" response is a known bug — the reload still executes correctly.

---

## Two Filesystems: The Critical Distinction

⛔ **`create_file` and `str_replace` (Anthropic sandbox tools) do NOT write to real Windows disk. They write to a sandbox that mimics Windows paths but is completely separate. For any write to `C:\`, always use `grille:fs_write_file` or `grille:fs_str_replace`, then verify with `grille:fs_stat`.**

The failure mode is silent — sandbox writes return success while the real file is untouched.

---

## Git and SSH

- `git push` cannot work via Grille — SSH agent is inaccessible from subprocess context. Push manually.
- `git.exe` requires the `.exe` suffix; Git for Windows deadlock is prevented internally by Grille.

---

## Machines

Configure your machines — laptop, desktop, or any deployment target — in `grille.toml`. Each machine runs its own Grille instance with its own config. Config is machine-local; changes on one machine do not affect another.
