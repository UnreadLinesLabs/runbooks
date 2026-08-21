# Server Naming Convention

This document defines the naming convention for virtual servers used in the infrastructure labs.

## Naming Format

```text
[A][NN][CITY]VM[ROLE][NN]
```

Example:

```text
U01PARVMDOM01
```

## Components

| Part | Meaning | Example |
| --- | --- | --- |
| `A` | Organization or environment code | `U` |
| `NN` | Site or entity number | `01` |
| `CITY` | Three-letter site or city code | `PAR` |
| `VM` | Virtual machine identifier | `VM` |
| `ROLE` | Three-letter server role | `DOM` |
| `NN` | Server sequence number | `01` |

## Server Role Codes

| Code | Role |
| --- | --- |
| `DOM` | Active Directory Domain Controller |
| `PKI` | Public Key Infrastructure |
| `NPS` | Network Policy Server |
| `DNS` | DNS Server |
| `WEB` | Web Server |
| `SQL` | SQL Server |
| `FSV` | File Server |

## Examples

```text
U01PARVMDOM01
U01PARVMDOM02
U01PARVMPKI01
U01PARVMNPS01
U01PARVMWEB01
```

## Naming Rules

- Use uppercase ASCII letters and numbers.
- Do not use spaces, accents, underscores, or special characters.
- Keep the computer name at 15 characters or fewer for compatibility with legacy NetBIOS-dependent systems.
- Use a stable three-letter site code.
- Increment the final number for servers with the same role and site.
- Do not reuse a retired computer name while its identity may still exist in Active Directory or management systems.
- Record the assigned name, role, site, IP address, and owner in the infrastructure inventory.

This naming convention is an internal infrastructure policy. Microsoft does not require this exact pattern; the pattern is designed to remain readable and compatible with Windows and Active Directory constraints.

## PowerShell Validation

Run the following command to validate a proposed name:

```powershell
$ComputerName = "U01PARVMDOM01"

if ($ComputerName -match '^[A-Z][0-9]{2}[A-Z]{3}VM[A-Z]{3}[0-9]{2}$' -and
    $ComputerName.Length -le 15) {
    Write-Host "Valid server name: $ComputerName"
}
else {
    Write-Error "Invalid server name: $ComputerName"
}
```

## Rename a Windows Server

Run PowerShell as Administrator before installing or promoting the server as a domain controller:

```powershell
Rename-Computer -NewName "U01PARVMDOM01" -Restart
```

Confirm the name after the restart:

```powershell
$env:COMPUTERNAME
```
