# Grille — Secrets Privacy Contract Reference

## The Contract

A secret must exist only in:
1. The vault (encrypted at rest — DPAPI or AKV)
2. Grille's process memory briefly during one operation
3. The target system's process memory

A secret must **never** appear in: Claude's conversation context, the MCP protocol payload, the conversation transcript, or any log Claude can read.

## Five Enforcement Mechanisms

**1. Pre-resolution allowlist check**
Before any secret is resolved, Grille checks whether the calling tool is authorized for that secret pattern. A prompt injection calling `sql_query` cannot resolve a secret that `sql_query` isn't allowed to use. The confused-deputy attack surface is zero.

**2. `Zeroizing<Vec<u8>>` storage**
Secret bytes are stored as `Zeroizing<Vec<u8>>` — memory is zeroed on drop. A `String` may be copied on heap reallocation before the old copy is zeroed; raw bytes do not. The `SecretValue` type has no `Debug` or `Display` impl — it cannot appear in logs or panics.

**3. Post-execution output scan**
After every tool call, Grille scans the output for any resolved secret bytes. Any match is replaced with the `{{secret:...}}` placeholder before the result is returned to Claude. Accidental leakage is caught at the boundary.

**4. `#[serde(skip)]` on redact entries**
The `redact_entries` field on `ToolCallResult` carries resolved values from tool execution back to the redaction pipeline. It is `#[serde(skip)]` — it never crosses the MCP wire to Claude under any circumstances.

**5. Audit log records refs, never values**
The audit log entry for a SQL call shows `secret_ref = "work/db:rekn"` — the opaque reference token. The resolved password value is never written to the audit log.

## Provider Summary

| Provider | Backend | Scope | Auth |
|----------|---------|-------|------|
| `local` | Windows Credential Manager (DPAPI) | Machine-local, user-scoped | User's Windows login |
| `work` | Azure Key Vault (REST, no SDK) | Cross-machine, team-shareable | `az login` (default_chain) or service principal |

## AKV Design Decision

The AKV backend calls the REST API directly — no Azure SDK. Rationale: the SDK adds 40+ transitive dependencies to a security tool's attack surface. AKV needs exactly three endpoints (AAD token, GET secret, PUT secret). Direct REST is auditable — every HTTP call is visible in source. (SD-018)

## SQL URL Injection Pattern

For Postgres connections with `secret_ref`:
```
grille.toml:  postgres://user@host/db  +  secret_ref = "work/db:rekn"
At call time:  postgres://user:RESOLVED_PASSWORD@host/db
```
The URL with placeholder lives in `grille.toml` — only the password is secret.
