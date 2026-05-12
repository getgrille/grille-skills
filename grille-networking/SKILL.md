---
name: grille-networking
description: "Use when diagnosing network state or connectivity on the local Windows machine via Grille. WHEN: 'what process is holding port 8080', 'what is my IP address', 'is megatron reachable', 'ping host', 'resolve hostname', 'DNS lookup', 'active connections', 'network adapters', 'what is the Tailscale IP', 'what external connections is my app making', 'why can my container not reach the database', 'net_connections', 'net_adapters', 'net_ping', 'net_dns_lookup'. DO NOT USE WHEN: 'run ping.exe or curl.exe' (use grille-process); 'HTTP request or API call' (Tier 2 net_http_get not yet implemented); 'firewall rules' (Tier 3, not implemented); 'network on a remote machine' (use grille-remote for ssh_run)."
---

## Overview

This skill covers Grille's Networking module — Tier 1 read-only diagnostics, 4 tools. All use native Windows APIs (iphlpapi) — no subprocess, no elevation required, structured output. Covers the most common developer and IT network questions without leaving Claude's context.

Requires `"networking"` in the active role's modules list.

## Tools

- `net_connections` — Active TCP and UDP connections with local/remote address, state, PID, and process name. Answers: "What process is holding port 8080?", "What is my app connecting to?", "Why can't my container reach the database?". Covers IPv4.
- `net_adapters` — Network interfaces with friendly name, description, MAC address, IP addresses (IPv4 and IPv6), and up/down status. Answers: "What is my IP?", "What is the Tailscale IP?", "Which adapters are active?".
- `net_ping` — ICMP echo via `IcmpSendEcho` (no raw socket privileges needed). Returns RTT min/avg/max and packet loss. Answers: "Is megatron reachable?", "What is the latency to this host?".
- `net_dns_lookup` — Forward DNS resolution via system resolver (`getaddrinfo`). Returns all resolved IPv4 and IPv6 addresses. Answers: "What does megatron.local resolve to?", "Has this DNS change propagated?".

## Patterns

**Find what is holding a port:**
```
grille:net_connections filter_port=8080
→ Proto, local, remote, state, PID, process name
→ Empty = nothing listening on that port
```

**Find all connections for a specific process:**
```
grille:net_connections filter_process="node"
→ All TCP/UDP entries where process name contains "node"
```

**See all external connections (hide loopback LISTEN):**
```
grille:net_connections protocol="tcp" include_listening=false
→ ESTABLISHED + other active states only
```

**Get IP addresses on this machine:**
```
grille:net_adapters
→ All UP adapters with IP/MAC — find Tailscale IP, LAN IP, etc.

grille:net_adapters include_down=true
→ Include disconnected adapters too
```

**Check reachability and latency:**
```
grille:net_ping host="megatron.local"
→ 4 pings, RTT min/avg/max ms, packet loss %

grille:net_ping host="100.95.128.36" count=1 timeout_ms=500
→ Single fast reachability check
```

**Resolve a hostname:**
```
grille:net_dns_lookup host="megatron.local"
→ All IPs the system resolver returns

grille:net_dns_lookup host="api.github.com"
→ Verify DNS propagation or CDN routing
```

**Diagnose container connectivity:**
```
grille:net_connections filter_process="docker"
→ See what Docker is connecting to

grille:net_connections filter_port=5432
→ Who is connected to Postgres? Is the container reaching it?

grille:net_ping host="localhost"
grille:net_dns_lookup host="host.docker.internal"
```

## Output formats

**net_connections:**
```
[OK] net connections — 142 connections

Proto  Local                        Remote                       State          PID  Process
---------
TCP    0.0.0.0:8080                 *:*                          LISTEN        4821  node.exe
TCP    127.0.0.1:5432               127.0.0.1:54312              ESTABLISHED   1204  postgres.exe
```

**net_adapters:**
```
[OK] net adapters — 3 adapters

Ethernet (UP)
  Description: Intel(R) Ethernet Connection
  MAC:         A4-BB-6D-12-34-56
  IP:          192.168.1.42, fe80::1

Tailscale (UP)
  MAC:         (none)
  IP:          100.95.128.10
```

**net_ping:**
```
[OK] net ping — megatron.local (100.95.128.36)

  Sent: 4  Received: 4  Lost: 0 (0% loss)
  RTT min/avg/max: 2/3/5 ms
```

**net_dns_lookup:**
```
[OK] net dns lookup — 'megatron.local' → 1 address

  100.95.128.36
```

## ⛔ Common mistakes

- **Expecting HTTP fetch** — `net_http_get` is Tier 2 and not yet implemented. Use `process_run` with `curl.exe` if allowed.
- **Remote machine networking** — these tools only see the local machine. For megatron's connections, use `grille:ssh_run` with `ss` or `netstat`.
- **IPv6 connections** — `net_connections` currently covers IPv4 only. IPv6 TCP/UDP table (Tier 1 extension) is deferred.
- **Expecting firewall rules** — `net_firewall_list` is Tier 3, not yet implemented.
