# SKILL.md — grille-wmi

## What This Skill Does
Query WMI/CIM classes on the local Windows machine for hardware, OS, BIOS, network, and system inventory data. Returns structured results as key-value pairs (single instance) or tables (multiple instances).

## Tool
`wmi_query` — query any WMI class listed in `allowed_classes` (grille.toml).

## Parameters
| Parameter | Required | Description |
|---|---|---|
| `class` | Yes | WMI class name, e.g. `Win32_Processor` |
| `filter` | No | WQL WHERE clause (without `WHERE`), e.g. `DriveType=3` |
| `properties` | No | Array of property names to return (omit = all) |

## Safe Class Set (initial allowlist)
| Class | What It Returns |
|---|---|
| `Win32_OperatingSystem` | OS version, build, uptime, memory stats |
| `Win32_ComputerSystem` | Hostname, domain membership, manufacturer, model, RAM |
| `Win32_Processor` | CPU name, cores, speed, socket, architecture |
| `Win32_PhysicalMemory` | RAM slot layout, capacity per DIMM, speed, manufacturer |
| `Win32_DiskDrive` | Physical disk model, size, interface, firmware |
| `Win32_LogicalDisk` | Drive letters, filesystem, free/total space |
| `Win32_NetworkAdapterConfiguration` | IP address, subnet, gateway, DNS, DHCP |
| `Win32_BIOS` | BIOS version, release date, serial number |
| `Win32_StartupCommand` | Startup entries (useful for security audits) |
| `Win32_BaseBoard` | Motherboard manufacturer, model, serial number |
| `Win32_SystemEnclosure` | Chassis type (desktop/laptop/server), serial number |

## Common Patterns

### Hardware inventory
```
class: Win32_Processor
properties: ["Name", "NumberOfCores", "NumberOfLogicalProcessors", "MaxClockSpeed"]
```

### RAM layout
```
class: Win32_PhysicalMemory
properties: ["BankLabel", "Capacity", "Speed", "Manufacturer"]
```

### Fixed drives only
```
class: Win32_LogicalDisk
filter: DriveType=3
properties: ["DeviceID", "Size", "FreeSpace", "FileSystem", "VolumeName"]
```

### Active network adapters
```
class: Win32_NetworkAdapterConfiguration
filter: IPEnabled=True
properties: ["Description", "IPAddress", "DefaultIPGateway", "DNSServerSearchOrder"]
```

### Security audit: startup entries
```
class: Win32_StartupCommand
```

## Security Constraints (SD-039)
- **Query-only** — method invocations structurally impossible
- **Local machine only** — ComputerName not exposed
- **Deny-by-default** — only classes in `allowed_classes` (grille.toml) work
- **Hardcoded blocks** (cannot be overridden): `Win32_Product` (MSI reconfiguration disaster), `Win32_Process`, `Win32_Service`, `StdRegProv`, all WMI event subscription and persistence classes

## grille.toml Config
```toml
# Add "wmi" to modules list
modules = ["...", "wmi"]

[roles.developer.wmi]
allowed_classes = [
    "Win32_OperatingSystem",
    "Win32_ComputerSystem",
    "Win32_Processor",
    "Win32_PhysicalMemory",
    "Win32_DiskDrive",
    "Win32_LogicalDisk",
    "Win32_NetworkAdapterConfiguration",
    "Win32_BIOS",
    "Win32_StartupCommand",
    "Win32_BaseBoard",
    "Win32_SystemEnclosure",
]
```

## Output Format
- **1 instance**: key-value pairs aligned by property name width
- **Multiple instances**: column table, max 500 rows, columns truncated to 60 chars
- Empty/null properties are omitted from output

## Notes
- Uses native COM/IWbemServices — no PowerShell, no WMIC subprocess
- Typical query latency: 20-80ms
- `Win32_PhysicalMemory` often returns multiple rows (one per DIMM slot)
- `Win32_NetworkAdapterConfiguration` with `IPEnabled=True` filter avoids returning dozens of virtual adapters
