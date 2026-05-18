---
name: grille-ssh
description: "Use when running commands or transferring files on remote SSH hosts configured in grille.toml. WHEN: 'ssh_run', 'remote_copy', 'deploy to server', 'run on remote machine', 'copy file to remote', 'check remote machine', 'ssh into server'. DO NOT USE WHEN: managing local Docker (use grille-docker); local file operations (use grille-filesystem)."
---

## Overview

This skill covers Grille's ssh module — outbound SSH access to named remote machines. Apply it when Claude needs to execute commands on a remote server, deploy files, or inspect a remote system. All operations target named hosts defined in `grille.toml` — not arbitrary IP addresses or hostnames.

"Outbound" is important: Grille's SSH is Claude reaching out *from your machine to a server you configured*. It is not RDP, WinRM, or PowerShell Remoting — there is no inbound attack surface.

Remote hosts and their connection details (IP, user, identity file, allowed commands, upload/download paths) are configured per-machine in `grille.toml` under `[[roles.<role>.ssh_hosts]]`.

## Tools

- `ssh_run` — Execute an allowlisted command on a named remote SSH host. Returns stdout and stderr. Two modes: (1) **single**: `command` + optional `args`; (2) **compound**: `commands` array of full command strings, joined with `&&` and executed in a single SSH call — short-circuits on first failure. Each element's first token is validated against `allowed_commands`. Cannot mix `command` and `commands`.
- `ssh_copy` — Copy a file to or from a named remote host via SCP. Supports upload (local → remote) and download (remote → local).

## Patterns

**Execute a command on a remote host:**
```
grille:ssh_run
  host="<remote-host>"
  command="docker ps"
```

**Compound mode — multiple steps in a single SSH call:**
```
grille:ssh_run
  host="<remote-host>"
  commands=["git fetch origin", "git pull"]
```
Each element's first token is validated against `allowed_commands`. Commands join
with `&&` — execution stops at first failure. Use `working_dir` to set the remote
directory for all steps.

**Compound deploy: copy, restart, verify in one call each:**
```
grille:ssh_copy
  host="<remote-host>" direction="upload"
  local_path="C:\\dev\\myapp\\target\\release\\app" remote_path="/tmp/app"

grille:ssh_run
  host="<remote-host>"
  commands=["sudo cp /tmp/app /opt/myapp/app", "sudo chmod +x /opt/myapp/app", "sudo systemctl restart myapp"]

grille:ssh_run host="<remote-host>" command="journalctl" args=["-u", "myapp", "-n", "20", "--no-pager"]
```

**Upload a compiled binary:**
```
grille:ssh_copy
  host="<remote-host>"
  direction="upload"
  local_path="C:\\dev\\myapp\\target\\release\\myapp.exe"
  remote_path="/tmp/myapp"
```

**Download a log file:**
```
grille:ssh_copy
  host="<remote-host>"
  direction="download"
  remote_path="/var/log/myapp/app.log"
  local_path="C:\\Users\\<user>\\Downloads\\app.log"
```

**Chain: deploy and restart a service:**
```
1. grille:ssh_copy (upload binary to /tmp/)
2. grille:ssh_run command="sudo cp /tmp/myapp /opt/myapp/myapp"
3. grille:ssh_run command="sudo systemctl restart myapp"
4. grille:ssh_run command="systemctl status myapp"
```

## Constraints

- **Do not use IP addresses or hostnames as the `host` parameter.** `host` must be a name from `grille.toml`'s `ssh_hosts` list, not a raw IP or hostname. Grille resolves all connection details from the named host definition.

⛔ **`host` must be a named host from grille.toml, not an IP address or hostname. Raw IPs will be rejected. Use the configured host name (e.g. `"my-server"`).**

- **Commands must be in the `allowed_commands` list for the target host.** In single mode the `command` executable is checked. In compound mode (`commands` array) the first token of each element is checked. Commands not in the list are denied before any SSH connection is made.
- **Do not embed shell operators (`&&`, `|`, `;`) in single-mode `command` or `args`.** Use compound mode (`commands` array) for multi-step operations instead.
- **Remote upload paths must be in `allowed_upload_paths`.** Files cannot be uploaded to arbitrary paths — only paths configured under `allowed_upload_paths` for the target host.
- **Remote download paths must be in `allowed_download_paths`.** Read access is scoped to paths configured under `allowed_download_paths` for the target host.
- **`ssh_run` timeout is 60 seconds.** For long-running remote commands, consider running them in the background (`command &`) or breaking the work into shorter steps.
- **`ssh_copy` uses SCP** — it transfers individual files, not directories. For directory transfers, tar the directory first with `ssh_run`, then copy the tarball.

## Examples

**Example 1: Inspect services on a remote host**
```
grille:ssh_run host="<remote-host>" command="docker ps"
→ See running containers

grille:ssh_run host="<remote-host>" command="df -h /opt/myapp"
→ Check disk usage

grille:ssh_run host="<remote-host>" command="systemctl status myapp"
→ Verify service is active
```

**Example 2: Deploy an updated binary and verify startup**
```
grille:ssh_copy
  host="<remote-host>"
  direction="upload"
  local_path="C:\\dev\\myapp\\target\\release\\myapp.exe"
  remote_path="/tmp/myapp"

grille:ssh_run
  host="<remote-host>"
  commands=["sudo cp /tmp/myapp /opt/myapp/myapp", "sudo chmod +x /opt/myapp/myapp", "sudo systemctl restart myapp"]

grille:ssh_run host="<remote-host>" command="journalctl" args=["-u", "myapp", "-n", "20", "--no-pager"]
→ Confirm clean startup in logs
```
