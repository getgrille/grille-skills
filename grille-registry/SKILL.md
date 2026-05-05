---
name: grille-registry
description: "Use when reading or writing Windows Registry values via Grille. WHEN: 'read registry', 'write registry', 'registry_read_value', 'registry_set_value', 'list registry key', 'HKCU', 'HKLM'. DO NOT USE WHEN: reading files (use grille-filesystem); reading Windows Event Log (use grille-eventlog)."
---

## Overview

This skill covers Grille's Windows Registry module — 4 tools for registry inspection and modification. Apply it when Claude needs to read application configuration from the registry, set registry values for application settings, or inspect system configuration within the scoped key allowlist.

Registry access is RBAC-gated at two levels: module enablement (`registry_read` / `registry_write`) and a key prefix allowlist (`registry_allowed_keys`). Current allowlist: `HKCU\Software` and `HKLM\SOFTWARE`. HKLM is read-only in all configurations.

## Tools

- `registry_list_key` — List subkeys and values under a registry key. Returns value names, types, and data.
- `registry_read_value` — Read a single named value from a registry key.
- `registry_set_value` — Write a named value to a registry key. HKCU only — HKLM writes are always denied.
- `registry_delete_value` — Delete a named value from a registry key. Irreversible within the session.

## Patterns

**Inspect before writing — always:**
```
1. grille:registry_read_value
     key="HKCU\\Software\\MyApp"
     value_name="ConnectionString"
   → Confirm current value and type before overwriting

2. grille:registry_set_value
     key="HKCU\\Software\\MyApp"
     value_name="ConnectionString"
     value_type="REG_SZ"
     value_data="new value"
```

**Explore a key's contents:**
```
grille:registry_list_key key="HKCU\\Software\\MyApp"
→ Lists all subkeys and values with names, types, and data
```

**Read a machine GUID (example: trial fingerprint):**
```
grille:registry_read_value
  key="HKLM\\SOFTWARE\\Microsoft\\Cryptography"
  value_name="MachineGuid"
```

## Constraints

- **Do not attempt to write to HKLM.** HKLM writes are denied in code for all roles, regardless of configuration. `registry_set_value` only operates on HKCU. This is a hardcoded protection — it cannot be configured away.

⛔ **HKLM writes are a hardcoded denial — not a config issue. STOP — only HKCU values can be written.**
- **`registry_delete_value` is irreversible within the session.** There is no undo and no trash. Confirm intent explicitly before executing. Only delete values you have verified exist and are no longer needed.
- **Do not attempt to access hardcoded denied keys.** The following key prefixes are blocked regardless of allowlist configuration: `HKLM\SAM`, `HKLM\SECURITY`, `HKLM\SYSTEM\...\LSA`, `HKLM\SYSTEM\...\SECUREPIPESERVERS`, `HKLM\...\Winlogon`, `HKLM\...\ProfileList`. These protections are compiled into the binary.
- **Key paths must be within `registry_allowed_keys`.** Current configured allowlist: `HKCU\Software` and `HKLM\SOFTWARE`. Access to keys outside these prefixes is denied. Use `grille_info` to see the current allowlist.
- **REG_BINARY writes are not supported in v1.** Grille can read REG_BINARY (returned as hex dump) but cannot write it. All other common types are supported: REG_SZ, REG_EXPAND_SZ, REG_DWORD, REG_QWORD, REG_MULTI_SZ.
- **Do not attempt key creation or key deletion.** Only value-level operations are exposed in v1. `registry_create_key` and `registry_delete_key` do not exist — creating a key requires setting a value within a new key path using an OS-level implicit create, which is not supported.

## Examples

**Example 1: Read and update an application's log level**
```
grille:registry_read_value
  key="HKCU\\Software\\MyApp\\Logging"
  value_name="Level"
→ Current: "Info"

grille:registry_set_value
  key="HKCU\\Software\\MyApp\\Logging"
  value_name="Level"
  value_type="REG_SZ"
  value_data="Debug"

grille:registry_read_value
  key="HKCU\\Software\\MyApp\\Logging"
  value_name="Level"
→ Confirm: "Debug"
```

**Example 2: Inspect all values for a software package**
```
grille:registry_list_key key="HKLM\\SOFTWARE\\Git"
→ Returns: InstallPath (REG_SZ), Version (REG_SZ), ...
→ Use InstallPath to confirm git.exe location
```
