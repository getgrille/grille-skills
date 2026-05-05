# Grille — Threat Model Reference

## The Core Invariant

> **The human controls the permission boundary. Claude operates within it. Claude never expands it.**

Every security property in Grille flows from this invariant.

## RBAC Evaluation Order

For every tool call, evaluated in this order — first match wins:

1. Module enabled in active role? → DENY if not
2. Hardcoded global deny (LOLBins, system dirs, security processes) → DENY always, no override
3. Explicit deny rule (fs_denied_paths, etc.) → DENY
4. Explicit allow rule (fs_allowed_paths, process_allowed, etc.) → ALLOW
5. Default → **DENY** (fail closed)

## Prompt Injection Risk

Grille enforces scope, not intent. An indirect prompt injection — malicious instructions embedded in a file, web page, or database record Claude reads — can instruct Claude to call Grille tools within their permitted scope. Grille cannot detect this at the tool layer.

**What Grille mitigates:**
- Tight allowlists: injection can only use tools and paths the human explicitly configured
- No self-elevation: injection cannot bootstrap to greater access (SD-001, SD-005)
- CLM on PowerShell: blocks sandbox escape chains (SD-003)
- Error messages never enumerate config: injection cannot use error responses for reconnaissance (SD-015)
- Audit log: post-incident forensics always available

**What Grille cannot mitigate:**
- Network egress via Anthropic's API itself (always-permitted `api.anthropic.com` — "Claudy Day" pattern)
- Malicious tool chaining within permitted scope
- Post-incident prevention (the log supports forensics, not prevention of within-scope actions)

**Recommended mitigations for users:**
1. Scope `registry_allowed_keys` and `fs_allowed_paths` tightly — least privilege is the primary defense
2. Do not store secrets where Claude can reach them
3. Review `grille_audit` after sessions involving untrusted content (web browsing, third-party repos, user-supplied files)
4. Use separate, narrower roles for sessions processing untrusted data

**CVE Context:**
- CVE-2025-68143/44/45: Anthropic's Git MCP server allowed RCE via injected README content chained with Filesystem MCP server
- CVE-2026-21852: Claude Code API traffic redirected via `ANTHROPIC_BASE_URL` to leak API keys
- "Claudy Day": injected payload used permitted `api.anthropic.com` to exfiltrate conversation history

Full analysis: `SecurityDecisions.md` SD-007, `09_SecurityHardening.md`

## Self-Elevation Is Impossible

- `grille_set_role` does not exist — rejected by design (SD-001)
- `grille_reload_config` reads disk only — never writes `grille.toml`
- Error messages never enumerate configured resources (SD-015)
- No tool can change `active_role`, add modules, or expand `fs_allowed_paths`
