---
name: grille-remote
description: "Use when running commands or transferring files on remote SSH hosts configured in grille.toml. WHEN: 'ssh_run', 'remote_copy', 'deploy to server', 'run on remote machine', 'copy file to remote', 'check remote machine'. DO NOT USE WHEN: managing local Docker (use grille-docker); local file operations (use grille-filesystem)."
---

## Overview

This skill covers Grille's remote module — SSH-based access to named remote machines. Apply it when Claude needs to execute commands on a remote server, deploy files, or inspect a remote system. All remote operations target named hosts defined in `grille.toml` — not arbitrary IP addresses or hostnames.

Remote hosts and their connection details (IP, user, identity file, allowed commands, upload/download paths) are configured per-machine in `grille.toml` under `[[roles.<role>.remote_hosts]]`.

## Tools

- `ssh_run` — Execute an allowlisted command on a named remote SSH host. Returns stdout and stderr.
- `remote_copy` — Copy a file to or from a named remote host via SCP. Supports upload (local → remote) and download (remote → local).

## Patterns

**Execute a command on a remote host:**
```
grille:ssh_run
  host="<remote-host>"
  command="docker ps"
```

**Check disk space on the remote:**
```
grille:ssh_run
  host="<remote-host>"
  command="df -h /opt/myapp"
```

**Upload a compiled binary:**
```
grille:remote_copy
  host="<remote-host>"
  direction="upload"
  local_path="C:\\dev\\myapp\\target\\release\\myapp.exe"
  remote_path="/tmp/myapp"
```

**Download a log file:**
```
grille:remote_copy
  host="<remote-host>"
  direction="download"
  remote_path="/var/log/myapp/app.log"
  local_path="C:\\Users\\<user>\\Downloads\\app.log"
```

**Chain: deploy and restart a service:**
```
1. grille:remote_copy (upload binary to /tmp/)
2. grille:ssh_run command="sudo cp /tmp/myapp /opt/myapp/myapp"
3. grille:ssh_run command="sudo systemctl restart myapp"
4. grille:ssh_run command="systemctl status myapp"
```

## Constraints

- **Do not use IP addresses or hostnames as the `host` parameter.** `host` must be a name from `grille.toml`'s `remote_hosts` list, not a raw IP or hostname. Grille resolves all connection details from the named host definition.

⛔ **`host` must be a named host from grille.toml, not an IP address or hostname. Raw IPs will be rejected. Use the configured host name (e.g. `"my-server"`).**

- **Commands must be in the `allowed_commands` list for the target host.** Each remote host has its own `allowed_commands` allowlist in `grille.toml`. Commands not in the list are denied at the Grille layer before any SSH connection is made. Check `grille.toml` or `grille_info` to see what is configured.
- **Remote upload paths must be in `allowed_upload_paths`.** Files cannot be uploaded to arbitrary paths — only paths configured under `allowed_upload_paths` for the target host.
- **Remote download paths must be in `allowed_download_paths`.** Read access is scoped to paths configured under `allowed_download_paths` for the target host.
- **`ssh_run` timeout is 60 seconds.** For long-running remote commands, consider running them in the background (`command &`) or breaking the work into shorter steps.
- **`remote_copy` uses SCP** — it transfers individual files, not directories. For directory transfers, tar the directory first with `ssh_run`, then copy the tarball.

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
grille:remote_copy
  host="<remote-host>"
  direction="upload"
  local_path="C:\\dev\\myapp\\target\\release\\myapp.exe"
  remote_path="/tmp/myapp"

grille:ssh_run host="<remote-host>" command="sudo cp /tmp/myapp /opt/myapp/myapp"
grille:ssh_run host="<remote-host>" command="sudo chmod +x /opt/myapp/myapp"
grille:ssh_run host="<remote-host>" command="sudo systemctl restart myapp"
grille:ssh_run host="<remote-host>" command="journalctl -u myapp -n 20 --no-pager"
→ Confirm clean startup in logs
```
