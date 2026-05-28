---
name: grille-networking
description: "Use when diagnosing network state or connectivity on the local Windows machine via Grille, making HTTP requests, looking up domain registration info, checking port ownership, or reading the local DNS cache. WHEN: 'what process is holding port 8080', 'what's on port 5432', 'who has port 443', 'is port 443 open on megatron', 'check if port is open', 'what is my IP address', 'is megatron reachable', 'ping host', 'resolve hostname', 'DNS lookup', 'active connections', 'network adapters', 'what is the Tailscale IP', 'what external connections is my app making', 'why can my container not reach the database', 'DNS cache', 'why is DNS resolving stale', 'is this hostname cached', 'net_connections', 'net_adapters', 'net_ping', 'net_dns_lookup', 'net_port_check', 'net_who_has_port', 'net_dns_cache', 'http get', 'http post', 'trigger webhook', 'call api', 'net_http_get', 'net_http_post', 'who owns this domain', 'when does this domain expire', 'domain registration', 'whois', 'nameservers', 'net_whois'. DO NOT USE WHEN: 'run ping.exe or curl.exe' (use grille-process); 'firewall rules or firewall profile' (use grille-firewall); 'network on a remote machine' (use grille-ssh for ssh_run); 'certificates or cert store' (use grille-certs)."
---

## Overview

This skill covers Grille's Networking module — Tier 1 read-only diagnostics, Tier 2 HTTP and RDAP, 10 tools total. Tier 1 tools use native Windows APIs (iphlpapi/dnsapi) — no subprocess, no elevation required, structured output. Tier 2 tools use `reqwest` for HTTP and RDAP queries.

Requires `"networking"` in the active role's modules list.

## Tools

- `net_connections` — Active TCP and UDP connections with local/remote address, state, PID, and process name. Covers IPv4 and IPv6 (TCP6/UDP6). Answers: "What process is holding port 8080?", "What is my app connecting to?", "Why can't my container reach the database?".
- `net_adapters` — Network interfaces with friendly name, description, MAC address, IP addresses (IPv4 and IPv6), and up/down status. Answers: "What is my IP?", "What is the Tailscale IP?", "Which adapters are active?".
- `net_ping` — ICMP echo via `IcmpSendEcho` (no raw socket privileges needed). Returns RTT min/avg/max and packet loss. Answers: "Is megatron reachable?", "What is the latency to this host?".
- `net_dns_lookup` — Forward DNS resolution via system resolver (`getaddrinfo`). Returns all resolved IPv4 and IPv6 addresses. Answers: "What does megatron.local resolve to?", "Has this DNS change propagated?".
- `net_port_check` — TCP connect test to a specific host:port. Returns OPEN/CLOSED/FILTERED and round-trip latency. Answers: "Is port 5432 open on megatron?", "Is my service listening?", "Can I reach Redis on port 6379?".
- `net_http_get` — HTTP GET to allowlisted endpoints (configured via `net_allowed_endpoints` in grille.toml). Optional request headers with `{{secret:name}}` resolution for credentials. Returns status code, optional response headers, and body (size-capped at 32 KB default, up to 256 KB). Answers: "Is the API returning 200?", "What does this health endpoint return?", "Query this REST endpoint".
- `net_http_post` — HTTP POST to allowlisted endpoints. Same allowlist model as `net_http_get`. Params: `body` (string), `content_type` (default: `application/json`), optional headers with `{{secret:name}}` resolution. Returns status code and response body. Use for: triggering webhooks, calling REST APIs that require POST, hitting internal action endpoints.
- `net_whois` — RDAP domain registration lookup via the IANA bootstrap registry. Returns status flags, creation/expiry dates, nameservers, registrar (name + IANA ID + URL), and registrant (or GDPR redaction note). No allowlist required — IANA is hardcoded infrastructure. Answers: "Who owns getgrille.com?", "When does this domain expire?", "What nameservers is it using?", "Is this domain registered?".

- `net_who_has_port` — Which process owns a port. Focused alternative to net_connections for the common "port is already in use" diagnostic. Params: `port` (required), `protocol=tcp|udp|both` (default both). Includes LISTEN bindings. Answers: "What's on port 5432?", "Why can't I bind to port 8080?", "Is Postgres listening?".

- `net_dns_cache` — Local Windows DNS resolver cache via DnsGetCacheDataTable. Answers: "Why is DNS resolving stale IPs?", "Is this hostname cached?". Filter by `filter_name` (substring) or `filter_type` (A/AAAA/CNAME/etc). Requires DNS Client service running.

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

**Check if a specific port is open:**
```
grille:net_port_check host="megatron.local" port=5432
→ OPEN (2ms) — port is reachable

grille:net_port_check host="localhost" port=6379
→ CLOSED — nothing listening

grille:net_port_check host="external.host.com" port=443 timeout_ms=2000
→ OPEN / FILTERED — use for service reachability checks
```

**HTTP GET to a local or private endpoint:**
```
grille:net_http_get url="http://192.168.1.221:8000/health"
→ 200 OK — service is up

grille:net_http_get url="http://localhost:8000/api/status"
→ Status code + body — check what the API returns

grille:net_http_get url="http://192.168.1.221:8000/api/projects"
  headers={"Authorization": "Bearer {{secret:rekn-dev-key}}"}
→ Authenticated request — secret resolved from Credential Manager, never logged

grille:net_http_get url="http://localhost:8000/api/data"
  include_response_headers=true
→ Full response headers + body — useful for debugging content-type, cache, etc.
```

**HTTP POST — trigger webhook or call REST API:**
```
grille:net_http_post
  url="http://192.168.1.221:8000/api/tasks"
  body='{"title": "Review PR", "priority": "high"}'
→ Creates a task via the API

grille:net_http_post
  url="http://localhost:9000/hooks/deploy"
  body='{"branch": "main"}'
  headers={"X-Hook-Token": "{{secret:deploy-hook-token}}"}
→ Trigger a deploy webhook — token from Credential Manager

grille:net_http_post
  url="http://192.168.1.221:8000/api/login"
  body="username=admin&password=test"
  content_type="application/x-www-form-urlencoded"
→ Form POST — override content_type when not JSON
```

**Look up domain registration:**
```
grille:net_whois domain="getgrille.com"
→ Status, created/expires, nameservers, registrar, registrant

grille:net_whois domain="example.org"
→ 404 from RDAP server → "NOT REGISTERED" message

grille:net_whois domain="*.getgrille.com"
→ Wildcard stripped automatically — looks up "getgrille.com"
```

**Diagnose why a port is closed (firewall correlation):**
```
grille:net_port_check host="localhost" port=5432
→ CLOSED

grille:firewall_rules direction="inbound" port=5432
→ Check for explicit block rules or missing allow rules

grille:firewall_profile
→ Default Inbound: Block — all ports closed unless explicitly allowed

→ See grille-firewall skill for full pattern and interpretation
```

**Diagnose container connectivity:**
```
grille:net_connections filter_process="docker"
→ See what Docker is connecting to

grille:net_port_check host="localhost" port=5432
→ Is Postgres reachable from the host?

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
TCP6   [::]:443                     [::]:*                       LISTEN        1204  nginx.exe
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

**net_port_check:**
```
[OK] net port check — megatron.local:5432

  Status:  OPEN
  Latency: 2ms
```

**net_http_get:**
```
[OK] GET http://192.168.1.221:8000/health — 200 OK (24ms)
Endpoint: rekn-local
Content-Type: application/json

{"status":"ok","version":"1.2.3"}
```

**net_whois:**
```
[OK] net whois — getgrille.com

  Status:        active
  Created:       2024-11-15
  Updated:       2025-03-02
  Expires:       2026-11-15
  Nameservers:   brad.ns.cloudflare.com, uma.ns.cloudflare.com
  Registrar:     Cloudflare, Inc.
  IANA ID:       1910
  Registrar URL: https://rdap.cloudflare.com
  Registrant:    [redacted — GDPR/privacy proxy]

  RDAP server: https://rdap.cloudflare.com
```

## ⛔ Common mistakes

- **Expecting HTTP fetch** — `net_http_get` and `net_http_post` are available for allowlisted private/local endpoints. For public internet URLs, use Claude's native web search — it's more capable and needs no config.
- **`net_whois` for TLDs without RDAP** — some country-code TLDs don't publish RDAP servers in the IANA bootstrap. The tool returns a clear "No RDAP server found for TLD" error in that case.
- **Hitting authenticated endpoints without a token** — `net_http_get` and `net_http_post` support `Authorization` headers via `{{secret:name}}` refs. Store credentials in Windows Credential Manager; never pass raw tokens in args.
- **URL not in allowlist** — `net_http_get` and `net_http_post` will deny any URL not matching a `[[roles.<role>.net_allowed_endpoints]]` entry. Add the endpoint to grille.toml first.
- **Remote machine networking** — these tools only see the local machine. For megatron's connections, use `grille:ssh_run` with `ss` or `netstat`.
- **Expecting firewall rules** — use the `grille-firewall` skill (`firewall_rules`, `firewall_profile`). Pairs naturally with `net_port_check` when a port is CLOSED.
- **Confusing net_connections with net_port_check** — `net_connections` shows what is currently connected on *this* machine; `net_port_check` actively probes whether a remote host:port is reachable.
