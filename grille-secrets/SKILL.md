---
name: grille-secrets
description: "Use when understanding Grille's credential resolution, diagnosing secrets failures, or configuring CredMan/AKV providers. WHEN: 'secret not resolving', 'credential error', 'configure AKV', 'set up secret', 'secret_ref', '{{secret:}}'. DO NOT USE WHEN: you simply need to run a SQL query — secrets resolve transparently; consult this skill only when resolution fails."
---

## Overview

This skill covers Grille's secrets management system. There are no direct "secrets" tools exposed to Claude — secrets are resolved transparently at tool call time using reference syntax in `grille.toml`. Apply this skill when Claude needs to understand why a tool call is failing due to a missing or denied secret, when configuring a new credential, or when diagnosing secrets-related issues.

Grille enforces a strict privacy contract: secrets never appear in Claude's conversation context, the MCP payload, the conversation transcript, or any log Claude can read. A secret exists only in the vault (encrypted at rest), Grille's process memory briefly during one operation, and the target system's process memory.

## How Secrets Work

**Reference syntax in `grille.toml`:**
```toml
[[roles.developer.sql_connections]]
name = "rekn"
url = "postgres://user@host/db"
secret_ref = "work/db:rekn"        ← resolved at call time from AKV provider "work"
```

**Resolution flow:**
1. Claude calls `sql_query connection="rekn"`
2. Grille finds `secret_ref = "work/db:rekn"` on the connection config
3. Grille checks the per-tool allowlist: is `sql_query` allowed to resolve `db:*` secrets?
4. If yes: Grille resolves the secret from the configured provider (CredMan or AKV)
5. Secret value injected into the connection URL — never into Claude's context
6. Output scanned post-execution; any accidental leakage replaced with `{{secret:...}}` placeholder

**Reference shorthand:**
```
{{secret:db:rekn}}           ← uses default provider
{{secret:local/db:rekn}}     ← explicit provider "local" (Windows Credential Manager)
{{secret:work/db:rekn}}      ← explicit provider "work" (Azure Key Vault)
```

## Providers

**`local` — Windows Credential Manager (DPAPI)**
- Secrets stored in Windows Credential Manager under `grille:<name>` prefix
- Encrypted with the user's DPAPI-derived key — tied to the Windows account
- Managed via `grille secrets set/list/delete/rotate/test` CLI subcommands
- Best for: local development credentials, database passwords on the local machine

**`work` — Azure Key Vault (REST API, no SDK)**
- Calls AKV REST API directly — no Azure SDK dependency
- Auth: `default_chain` (env vars → `az login` cached token) for interactive use
- Token cached in memory with 50-minute window; refreshed automatically
- Secret values cached with configurable TTL (default 60s); negative cache 10s for 404s
- Best for: shared team credentials, CI/CD secrets, credentials used across machines

## Patterns

**Diagnose a secrets resolution failure:**
```
1. grille:grille_audit tool="sql_query"
   → Look for "secret_resolution_failed" entries
   → Note: the audit entry shows the secret ref token, never the value

2. Check if the secret exists:
   grille secrets test --provider work db:rekn   ← from terminal
   or: grille secrets list --provider work

3. Verify the per-tool allowlist in grille.toml:
   [roles.developer.secrets_providers.work.allowed_tools]
   "db:*" = ["sql_query", "sql_execute", ...]
```

**Add a new CredMan secret:**
```
From terminal: grille secrets set db:myapp-dev
→ Enter password at prompt (never pass in command line)
```

**Add a new AKV secret:**
```
From terminal: grille secrets set --provider work db:myapp-prod
→ Writes to Azure Key Vault at grille-akv-1.vault.azure.net
→ AKV name: "db-myapp-prod" (colon replaced with dash)
```

## Constraints

- **Do not pass secret values directly in tool arguments.** Never pass passwords, tokens, or API keys as inline values to any Grille tool. Use the `secret_ref` mechanism in `grille.toml` and let Grille resolve them.
- **Do not add a new secrets provider to `grille.toml` and expect `grille_reload_config` to register it.** New provider registrations require a full Claude Desktop restart. `grille_reload_config` hot-swaps existing provider config (e.g., cache TTL changes) but does not initialize new provider instances.
- **Do not attempt to read resolved secret values from tool output.** Grille's output redaction pipeline replaces any resolved secret value appearing in tool output with `{{secret:...}}`. This is by design — the value never reaches Claude's context.
- **`az login` is required for AKV `default_chain` auth.** If the Azure CLI token has expired, AKV secret resolution will fail. Re-run `az login` from a terminal to refresh. The token lasts approximately 1 hour for the AKV scope.
- **Windows Credential Manager secrets are user-scoped and machine-local.** CredMan secrets set on the laptop are not available on the desktop, and vice versa. For cross-machine credentials, use the AKV provider.

## Examples

**Example 1: Understand why a SQL connection is failing**
```
1. grille:docker_exec container="rekn-postgres" command=["pg_isready"]
   → Confirm the database is accepting connections

2. grille:grille_audit
   → Check for "secret_resolution_failed" or "secret_allowlist_denied" events

3. If secret failed: verify grille.toml has the correct secret_ref and allowed_tools config
4. If allowlist denied: add the tool to the "db:*" allowed_tools list
5. grille:grille_reload_config  ← config-only changes hot-reload; new providers do not
```
