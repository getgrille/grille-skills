# Grille — Binary and Runtime Properties Reference

## Distribution Model

- **Single compiled Rust binary** — no Node.js, no Python, no `npx`, no package execution at runtime
- **~10MB release binary** — no framework overhead; Rust compiles to native code
- **~8MB idle memory** — vs 40-80MB for Python-based MCP servers, 50-100MB for Node
- **~20ms startup** — deterministic, no runtime initialization
- **Single `.exe`** — drop anywhere and run; zero install dependencies

## Why Rust for a Security Tool

| Concern | Choice | Outcome |
|---------|--------|---------|
| Runtime attack surface | None | No interpreter process to compromise |
| Memory safety | Compile-time | No buffer overflows from path strings |
| Windows APIs | `windows-rs` | Microsoft-maintained bindings, no wrappers |
| Distribution | Single binary | No supply chain of npm/pip packages at runtime |

## License Verification

- **Ed25519 signatures** — offline verification; private key never embedded in binary
- **Trial and perpetual keys** — verified by signature alone, no network call
- **Annual keys** — LemonSqueezy activation on first use, 30-day cache, 7-day grace on network failure
- **No call-home on every startup** — the only outbound calls are activation (once) and AKV token refresh (if configured)

## Audit Log Properties

- Local JSONL at `%APPDATA%\Grille\logs\grille-audit-YYYY-MM-DD.jsonl`
- Every tool call logged — successful, failed, and denied
- Append-only; daily rotation; 30-day retention by default (configurable 1 day–10 years)
- Retention duration is **not tier-gated** — same design at every license tier (SD-020)
- Not transmitted anywhere — Grille does not send audit data to any external service

## Windows Event Log Emission

Grille writes structured events for security-relevant actions:

| Event ID | Trigger | SIEM Signal |
|----------|---------|-------------|
| 1000 | Server started | — |
| 1001 | Server stopped | — |
| 1002 | Config reloaded | — |
| 2002 | Process execution denied | ⚠ Review if unexpected |
| 2004 | LOLBin blocked | 🚨 Investigate |
| 2005 | Path traversal blocked | 🚨 Investigate |
| Future 2006 | `ps_kill` of protected process denied | 🚨 Alert |

These events are queryable via `grille:eventlog_query source="Grille"` and by any SIEM connected to Windows Event Log.
