# Deploy the First Active Directory Domain Controller

This procedure deploys the first domain controller and the first forest for a professional lab environment.

The procedure is written for a fresh Windows Server VM. It must not be performed on a server that is already joined to another domain, is already a domain controller, or hosts unrelated application roles.

## Target Configuration

| Item | Value |
| --- | --- |
| Server name | `U01PARVMDOM01` |
| AD forest and domain | `corp.unreadlines.com` |
| NetBIOS domain name | `UNREADLINES` |
| IPv4 address | `192.168.20.41` |
| Prefix length | `/24` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `192.168.20.2` |
| DNS forwarder | `192.168.20.2` |

Use the DNS name `unreadlines.com` later as an alternate UPN suffix for Microsoft Entra integration. Do not use it as the AD forest name if the public DNS zone is used by other services.

## Prerequisites

- A supported, fully updated Windows Server installation.
- A fresh VM with a unique virtual network identity.
- Local Administrator access.
- A static IPv4 address reserved for this server.
- The server name approved according to the infrastructure naming convention.
- The server is not joined to an existing domain.
- The VM snapshot policy is understood before installing AD DS.

## Initial Manual Preparation

1. Configure the final IPv4 address and subnet mask manually:
    - IPv4 address: `192.168.20.41`
    - Subnet mask: `255.255.255.0` (`/24`)
    - Default gateway: `192.168.20.2`
    - Temporary DNS server before AD DS/DNS installation: `192.168.20.2`
2. Configure the final computer name manually: `U01PARVMDOM01`.
3. Enable Remote Desktop manually so the server can be administered through mRemoteNG.
4. Connect to the server with mRemoteNG and continue the remaining steps in an elevated PowerShell session.

The IPv4 address is final from the beginning. Only the client DNS setting changes after AD DS/DNS installation: it changes from the upstream DNS server `192.168.20.2` to the local DNS service `192.168.20.41`.

### Rename the Server with PowerShell

If the computer name was not configured manually, rename the server before installing AD DS:

```powershell
Rename-Computer `
    -NewName "U01PARVMDOM01" `
    -Restart
```

After the restart, confirm the name:

```powershell
$env:COMPUTERNAME
```

Do not rename the server after it has been promoted to a domain controller.

## Functional Levels and AD Schema

These three concepts must not be confused:

- **Forest functional level** enables forest-wide Active Directory features.
- **Domain functional level** enables domain-wide Active Directory features.
- **AD schema version** defines the attributes and object classes available in the forest. It is extended by Windows Server or application installations and is shared by the entire forest.

The functional level is not a cosmetic label for the Windows Server version. Choose the highest functional level supported by the Windows Server version that will be used by every domain controller in the forest. Do not select Windows Server 2012 merely because older documentation uses that value.

Before promotion, record the operating system version and review the functional levels supported by that version:

```powershell
Get-ComputerInfo |
    Select-Object WindowsProductName, WindowsVersion, OsBuildNumber

Get-Help Install-ADDSForest -Parameter ForestMode
Get-Help Install-ADDSForest -Parameter DomainMode
```

For a production design, decide this level before deploying the first domain controller. Raising functional levels is a planned forest-wide or domain-wide change and can restrict support for older domain controllers. The level cannot be used to make an older operating system provide newer AD DS capabilities.

The schema is forest-wide and should be extended only after testing application prerequisites, checking replication health, and confirming that a tested System State backup is available.

Run PowerShell as Administrator.

## 1. Install Active Directory Domain Services

```powershell
Install-WindowsFeature `
    -Name AD-Domain-Services `
    -IncludeManagementTools
```

## 2. Create the First Forest and Domain Controller

The following command creates the first forest, installs DNS, and promotes the server as the first domain controller.

Enter the DSRM password interactively. Do not place it directly in a script or document.

```powershell
$DSRMPassword = Read-Host `
    -AsSecureString `
    -Prompt "Enter the Directory Services Restore Mode password"

Install-ADDSForest `
    -DomainName "corp.unreadlines.com" `
    -DomainNetbiosName "UNREADLINES" `
    -InstallDNS:$true `
    -SafeModeAdministratorPassword $DSRMPassword `
    -Force
```

The server will restart automatically. Sign in with the domain Administrator account after the restart.

## 3. Configure the Domain Controller DNS Client

Discover the active interface again:

```powershell
$InterfaceAlias = Get-NetAdapter |
    Where-Object Status -eq "Up" |
    Select-Object -First 1 -ExpandProperty Name

$InterfaceAlias
```

Configure the domain controller to use its own DNS service:

```powershell
Set-DnsClientServerAddress `
    -InterfaceAlias $InterfaceAlias `
    -ServerAddresses "192.168.20.41"
```

Do not use `127.0.0.1` as the documented DNS client address. The fixed server address makes the configuration explicit and easier to troubleshoot.

## 4. Configure the DNS Forwarder

Add the upstream DNS server as a forwarder:

```powershell
$Forwarder = Get-DnsServerForwarder |
    Where-Object IPAddress -eq "192.168.20.2"

if (-not $Forwarder) {
    Add-DnsServerForwarder `
        -IPAddress "192.168.20.2" `
        -PassThru
}
```

Confirm the forwarder:

```powershell
Get-DnsServerForwarder
```

## 5. Validate the Deployment

Confirm the domain controller identity:

```powershell
Get-ADDomainController -Identity $env:COMPUTERNAME |
    Select-Object HostName, IPv4Address, Site, IsGlobalCatalog
```

Validate the domain controller:

```powershell
dcdiag /v
```

Validate DNS-specific tests:

```powershell
dcdiag /test:DNS /v
```

Check the forest and domain:

```powershell
Get-ADForest |
    Select-Object Name, ForestMode

Get-ADDomain |
    Select-Object DNSRoot, NetBIOSName, DomainMode
```

Test internal and external DNS resolution:

```powershell
Resolve-DnsName -Name "corp.unreadlines.com" -Server "192.168.20.41"
Resolve-DnsName -Name "microsoft.com" -Server "192.168.20.41"
```

## Expected Final State

```text
Computer name : U01PARVMDOM01
AD domain     : corp.unreadlines.com
NetBIOS name  : UNREADLINES
IPv4 address  : 192.168.20.41/24
Gateway       : 192.168.20.2
DNS client    : 192.168.20.41
DNS forwarder : 192.168.20.2
```

Do not install the PKI, Entra Connect, NPS, DHCP, or application roles on this domain controller. Add dedicated servers as the infrastructure grows.