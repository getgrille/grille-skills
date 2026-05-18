---
name: grille-firewall
description: "Use when reading Windows Firewall rules or the active firewall profile. WHEN: 'why is port X closed', 'what firewall rules exist for port X', 'is there a firewall rule blocking postgres', 'show me inbound block rules', 'what is the active firewall profile', 'what is the default inbound policy', 'firewall_rules', 'firewall_profile', 'is the firewall blocking X'. PAIR WITH grille-networking when net_port_check returns CLOSED and you need to explain why. DO NOT USE WHEN: modifying firewall rules (read-only only); remote machine firewall (local machine only); IPSec / connection security rules (not exposed)."
---

## Overview

This skill covers Grille's Firewall module — 2 read-only tools via the Windows Firewall COM interface (`INetFwPolicy2`). No elevation required. Local machine only.

Requires `"firewall"` in the active role's modules list.

## Tools

- `firewall_rules` — List and filter Windows Firewall rules. Supports filtering by direction, action, enabled state, port, protocol, profile, and name substring. Returns up to 500 rules per call.
- `firewall_profile` — Get the active network profile(s) (Domain/Private/Public), firewall enabled state, and default inbound/outbound actions for each profile.

## Key diagnostic pattern

This is the primary use case — explaining why a port is closed:

```
Step 1: net_port_check host="localhost" port=5432
→ Status: CLOSED

Step 2: firewall_rules direction="inbound" port=5432
→ Matches rules with local port 5432, or program-specific rules with Any ports

Step 3: firewall_profile
→ Active Profile: Private | Default Inbound: Block

Claude explains:
"Port 5432 is closed. No explicit allow rule exists for it on inbound TCP/5432.
The Private profile has a default Block inbound policy — all ports are closed
unless explicitly allowed. Start PostgreSQL and ensure an inbound allow rule
exists for port 5432 on the Private profile."
```

**Why firewall_profile matters:** A default Block inbound policy means there will
be no explicit deny rule for a closed port — the port is just not mentioned. Without
`firewall_profile`, the absence of a block rule looks like the port *should* be open.

## Patterns

**Find rules for a specific port:**
```
grille:firewall_rules direction="inbound" port=5432
→ All enabled inbound rules that cover port 5432 (explicit or Any-port)

grille:firewall_rules direction="inbound" port=5432 protocol="tcp"
→ Narrow to TCP only
```

**Find all block rules:**
```
grille:firewall_rules action="block"
→ All enabled block rules (both directions)

grille:firewall_rules direction="inbound" action="block"
→ Inbound block rules only
```

**Check active profile and defaults:**
```
grille:firewall_profile
→ Active profile(s), firewall enabled, default inbound/outbound action per profile
```

**Find rules for a specific application:**
```
grille:firewall_rules name_contains="postgres"
→ Rules whose name contains "postgres" (case-insensitive)

grille:firewall_rules name_contains="docker"
→ Docker-related firewall rules
```

**Check a specific network profile:**
```
grille:firewall_rules direction="inbound" profile="public" action="allow"
→ What is explicitly allowed inbound on the Public profile?
```

**See disabled rules (for auditing):**
```
grille:firewall_rules enabled=false
→ All disabled rules — useful for checking what was turned off
```

## Output formats

**firewall_profile:**
```
Windows Firewall Profile

Active Profile(s): Private, Public

-- Private Profile --
  Firewall Enabled:         true
  Default Inbound Action:   Block
  Default Outbound Action:  Allow

-- Public Profile --
  Firewall Enabled:         true
  Default Inbound Action:   Block
  Default Outbound Action:  Allow

Inactive Profiles:
  Domain -- Enabled: true  |  Inbound: Block  |  Outbound: Allow
```

**firewall_rules:**
```
Windows Firewall Rules — 3 matched (of 668 total)

Name:         PostgreSQL Server
Direction:    Inbound  |  Action: Allow  |  Enabled: true
Profile(s):   Private
Protocol:     TCP  |  Local Ports: 5432  |  Remote Ports: Any
Program:      C:\Program Files\PostgreSQL\16\bin\postgres.exe

Name:         Node.js JavaScript Runtime
Direction:    Inbound  |  Action: Block  |  Enabled: true
Profile(s):   Private, Public
Protocol:     TCP  |  Local Ports: *  |  Remote Ports: *
Program:      C:\program files\nodejs\node.exe

Name:         Core Networking - IPv6 (IPv6-In)
Direction:    Inbound  |  Action: Allow  |  Enabled: true
Profile(s):   Domain, Private, Public
Protocol:     Other  |  Local Ports: Any  |  Remote Ports: Any
Program:      System
```

## Port filter behaviour

`port=N` matches rules where the local ports field covers N:
- Exact match: `5432`
- Wildcard: `*` or `Any` — matches any port (most program rules fall here)
- Range: `8000-8999` — matches if N is in range
- List: `80,443,8080` — matches if N is in the list

This means a query for `port=5432` will return program-specific rules with
`Local Ports: Any` — they *could* apply to port 5432. Use `protocol="tcp"` to
narrow if you get too many results.

## Security constraints (SD-044)

- **Read-only** — no rule creation, modification, or deletion
- **Local machine only** — INetFwPolicy2 reflects local policy only
- **IPSec excluded** — connection security rules (INetFwRules3) are not loaded
- **No elevation required** — standard user can read all firewall rules

## Common mistakes

- **Expecting empty results to mean "port is open"** — a Block default inbound
  policy closes all ports not explicitly allowed. Always call `firewall_profile`
  alongside `firewall_rules` to get the full picture.
- **Too many results without filters** — 668 rules on a typical Windows machine.
  Always filter by `direction` and `port` or `name_contains` in practice.
- **port filter returns program rules with Any ports** — this is correct behaviour.
  A program rule with `Local Ports: Any` applies to all ports including the one
  you searched for. Focus on the Action and Program fields to interpret them.
- **Remote machine** — these tools only see the local machine's firewall.
  For a remote machine, use `grille:ssh_run` with `netsh advfirewall` commands.
- **Modifying rules** — not supported. Use `netsh advfirewall firewall` via
  `grille:ps_run` if you need to add or modify rules (requires admin).
