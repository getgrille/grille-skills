---
name: grille-docker
description: "Use when managing Docker containers, images, or Compose services on Windows via Grille. WHEN: 'docker ps', 'start container', 'stop container', 'docker logs', 'compose up', 'compose down', 'docker exec', 'container status'. DO NOT USE WHEN: 'run SQL query' (use grille-sql); 'run command on remote machine' (use grille-remote)."
---

## Overview

This skill covers Grille's Docker module — 12 tools for container lifecycle management, log access, resource monitoring, and controlled command execution inside containers. Apply it whenever Claude needs to start, stop, inspect, or interact with Docker containers and Compose stacks on the local Windows machine.

Grille's Docker tools call the Docker CLI via subprocess (not the Docker Engine API directly). Docker Desktop must be running.

## Tools

- `docker_ps` — List containers with status, ports, and uptime.
- `docker_logs` — Fetch container log output. Supports `tail` (line count) parameter.
- `docker_inspect` — Full container details: image, mounts, network, health status, environment.
- `docker_images` — List local images with tag and size.
- `docker_stats` — CPU and memory snapshot for running containers.
- `docker_start` — Start a stopped container.
- `docker_stop` — Gracefully stop a running container.
- `docker_compose_ps` — Status of all services in a Compose project.
- `docker_compose_up` — Start Compose services (always detached).
- `docker_compose_down` — Stop and remove Compose services and networks.
- `docker_compose_restart` — Restart one or more Compose services without recreating them.
- `docker_exec` — Execute an allowlisted command inside a running container.

## Patterns

**Check container health before connecting:**
```
1. grille:docker_ps          ← confirm container is running
2. grille:docker_inspect     ← check health status and port bindings
3. grille:sql_query or grille:ssh_run ← proceed with work
```

**Bring up a Compose stack:**
```
grille:docker_compose_up
  project_dir="C:\\dev\\grille-demo"
  services=["postgres", "api"]    ← omit for all services
```

**Tail recent logs for debugging:**
```
grille:docker_logs
  container="rekn-postgres"
  tail=50
```

**Execute a command inside a container:**
```
grille:docker_exec
  container="rekn-postgres"
  command="psql"
  args=["-U", "postgres", "-c", "SELECT count(*) FROM users;"]
```
Only the executable (`command`) is allowlist-checked. Args are unrestricted — complex
flags, `-c "..."` expressions, and multi-part arguments all work without any extra config.

**Restart a single service after a config change:**
```
grille:docker_compose_restart
  project_dir="C:\\dev\\rekn"
  services=["api"]
```

## Constraints

- **Do not use `$` characters in passwords passed to Docker Compose.** Docker Compose performs variable interpolation on `$` in `.env` files and inline values. A password like `P@ss$word` will have `$word` expanded (likely to empty string), silently corrupting the credential. Use alphanumeric passwords in all Compose-managed environments.
- **`docker_exec` allowlist covers the executable only.** Each container's permitted executables are configured in `grille.toml` under `docker_exec_allowed`. Args are unrestricted — any flags or `-c "..."` expressions valid for that executable work as-is. Check `grille.toml` or `grille_info` for configured allowlists.
- **`docker_compose_up` always runs detached.** There is no interactive or attached mode. Log output is accessible via `docker_logs` after startup.
- **Do not attempt Docker Engine API calls directly.** Grille's Docker module uses the Docker CLI (`docker.exe`) via subprocess. A native Docker Engine API module is deferred pending demand. All current tools are CLI-backed.
- **`docker_stop` is graceful — it sends SIGTERM and waits.** For containers that are hung and not responding to SIGTERM, use `docker_inspect` first to understand state. Forced kill is not exposed in v1.
- **Docker Desktop must be running.** If Docker Desktop is not started, all docker_* calls will fail with a connection error. Check with `docker_ps` first.

## Examples

**Example 1: Verify the database is healthy before running migrations**
```
grille:docker_ps
→ Confirm rekn-postgres is "Up X minutes (healthy)"

grille:docker_exec
  container="rekn-postgres"
  command="pg_isready"
  args=["-U", "postgres"]
→ "localhost:5432 - accepting connections"

grille:sql_query
  connection="rekn"
  query="SELECT version()"
→ Proceed with migrations
```

**Example 2: Debug a failing API container**
```
grille:docker_compose_ps
  project_dir="C:\\dev\\rekn"
→ Identify which service is in "Exit 1" state

grille:docker_logs
  container="rekn-api"
  tail=100
→ Read error output to diagnose

grille:docker_compose_restart
  project_dir="C:\\dev\\rekn"
  services=["api"]
→ Restart after fix
```
