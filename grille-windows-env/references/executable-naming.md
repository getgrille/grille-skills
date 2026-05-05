# Grille — Executable Naming Reference

## The Rule

Grille spawns executables via `tokio::process::Command` without a shell. Shell extension resolution does not apply. Always include the full extension.

## Full Allowlist with Correct Names

| Tool | Correct | Wrong |
|------|---------|-------|
| Rust build | `cargo.exe` | `cargo` |
| Git | `git.exe` | `git` |
| npm | `npm.cmd` | `npm` |
| Azure CLI | `az.cmd` | `az`, `az.exe` |
| xcopy | `xcopy.exe` | `xcopy` |
| Docker | `docker.exe` | `docker` |
| Node | `node.exe` | `node` |
| Python | `python.exe` | `python` |
| Rustup | `rustup.exe` | `rustup` |
| dotnet | `dotnet.exe` | `dotnet` |
| SSH | `ssh.exe` | `ssh` |
| PowerShell 7 | `pwsh.exe` | `pwsh` (via `ps_run`, not `process_run`) |

## Why az.cmd Specifically Matters

The Azure CLI is installed as `az.cmd` — not `az.exe`. Without a shell, `az` and `az.exe` both fail. The error is swallowed by the AKV auth chain fallback, making it appear that the Azure CLI is not installed when it is. Always use `"az.cmd"` explicitly. This is documented in SD-019 and was confirmed in Session 24.

## PowerShell

`powershell.exe` (5.1) is on the LOLBin deny list in `process_run` — it will always fail. For PowerShell cmdlets, use `ps_run` (the `powershell` module). `pwsh.exe` (PowerShell 7) is not on the deny list but is also not in `process_run`; use `ps_run` for all PowerShell work.
