# Deploy the first Active Directory domain controller

This procedure deploys the first domain controller and the first forest for a professional lab environment.

The procedure is written for a fresh Windows Server VM. It must not be performed on a server that is already joined to another domain, is already a domain controller, or hosts unrelated application roles.

## 1. Architecture

```text
                  U01PARVMFWL01 — OpenWrt router
                  default gateway and upstream DNS
                          192.168.20.126
                                   |
                     Subnet 1 — 192.168.20.0/25
                                   |
                            U01PARVMDOM01
                            192.168.20.41
              +--------------------------------------------+
              |  AD DS — first forest of the environment   |
              |  Forest / domain : corp.unreadlines.com    |
              |  NetBIOS         : UNREADLINES             |
              |                                            |
              |  DNS — authoritative for the AD zone,      |
              |  forwards everything else upstream         |
              +--------------------------------------------+
```

The DNS client setting on this server changes once during the build, and only that setting:

```text
before installing AD DS/DNS ....  192.168.20.126   (upstream resolver)
after  installing AD DS/DNS ....  192.168.20.41  (itself)
```

## 2. Scope and dependencies

This runbook builds the **first** domain controller and the first forest: it installs AD DS on
`U01PARVMDOM01`, creates `corp.unreadlines.com`, stands up the DNS service that comes with it, and
validates resolution inside and outside the domain.

It does **not** add a second domain controller, configure Sites and Services, create the organizational
unit structure, or join any client to the domain. It also deliberately installs nothing else here: PKI,
NPS/RADIUS, Entra Connect and application roles each belong on a dedicated server as the infrastructure
grows — `ad-cs-pki-deployment/README.md` and `radius-nps-deployment/README.md` are built on separate
machines for exactly that reason.

Naming for the accounts, groups and OUs this forest will hold is defined in
`reference/server-naming-convention.md`. The hybrid identity design that sits on top of it — including
the `unreadlines.com` UPN suffix mentioned below — is in
`reference/active-directory-entra-identity-design.md`.

## 3. Target configuration

| Item | Value |
| --- | --- |
| Server name | `U01PARVMDOM01` |
| AD forest and domain | `corp.unreadlines.com` |
| NetBIOS domain name | `UNREADLINES` |
| IPv4 address | `192.168.20.41` |
| Prefix length | `/25` |
| Subnet mask | `255.255.255.128` |
| Default gateway | `192.168.20.126` |
| DNS forwarder | `192.168.20.126` |

> **Note on the addressing.** This domain controller was first built when the lab was a single flat
> `192.168.20.0/24` network with no router of its own, and the recording made at that time shows that
> plan — a `/24` prefix and `192.168.20.2` as gateway and resolver. The lab has since been split into
> two `/25` subnets behind a pair of OpenWrt routers
> (`openwrt-wireguard-site-to-site/README.md`). The values in the table above are the current ones:
> a `/25` prefix, with `U01PARVMFWL01` (`192.168.20.126`) as both default gateway and upstream DNS.
> They are what the rest of this repository and `reference/vm-inventory.md` agree on — follow them
> rather than the older figures if the two ever disagree.

Use the DNS name `unreadlines.com` later as an alternate UPN suffix for Microsoft Entra integration. Do not use it as the AD forest name if the public DNS zone is used by other services.

## 4. Prerequisites

- A supported, fully updated Windows Server installation.
- A fresh VM with a unique virtual network identity.
- Local Administrator access.
- A static IPv4 address reserved for this server.
- The server name approved according to the infrastructure naming convention.
- The server is not joined to an existing domain.
- The VM snapshot policy is understood before installing AD DS.

**Every command in this runbook runs on `U01PARVMDOM01`.**

## 5. Initial manual preparation

1. Configure the final IPv4 address and subnet mask manually:
    - IPv4 address: `192.168.20.41`
    - Subnet mask: `255.255.255.128` (`/25`)
    - Default gateway: `192.168.20.126`
    - Temporary DNS server before AD DS/DNS installation: `192.168.20.126`
2. Configure the final computer name manually: `U01PARVMDOM01`.
3. Enable Remote Desktop manually so the server can be administered through mRemoteNG.
4. Connect to the server with mRemoteNG and continue the remaining steps in an elevated PowerShell session.

The IPv4 address is final from the beginning. Only the client DNS setting changes after AD DS/DNS installation: it changes from the upstream DNS server `192.168.20.126` to the local DNS service `192.168.20.41`.

### 5.1 Verify the hostname and network configuration

Before installing AD DS, verify the manually configured hostname, IP address, subnet prefix, gateway, and DNS server:

```powershell
$ActiveInterface = Get-NetAdapter |
    Where-Object Status -eq "Up" |
    Select-Object -First 1

$Configuration = Get-NetIPConfiguration `
    -InterfaceIndex $ActiveInterface.ifIndex

[PSCustomObject]@{
    ComputerName = $env:COMPUTERNAME
    Interface    = $ActiveInterface.Name
    IPv4Address  = ($Configuration.IPv4Address.IPAddress -join ", ")
    PrefixLength = ($Configuration.IPv4Address.PrefixLength -join ", ")
    Gateway      = ($Configuration.IPv4DefaultGateway.NextHop -join ", ")
    DnsServers   = ($Configuration.DnsServer.ServerAddresses -join ", ")
}
```

Confirm these values before continuing:

```text
ComputerName : U01PARVMDOM01
IPv4Address  : 192.168.20.41
PrefixLength : 24
Gateway      : 192.168.20.126
DnsServers   : 192.168.20.126
```

### 5.2 Rename the server with PowerShell

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

## 6. Functional levels and AD schema

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

## 7. Install Active Directory Domain Services

```powershell
Install-WindowsFeature `
    -Name AD-Domain-Services `
    -IncludeManagementTools
```

## 8. Create the first forest and domain controller

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

## 9. Configure the domain controller DNS client

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

## 10. Configure the DNS forwarder

Add the upstream DNS server as a forwarder:

```powershell
$Forwarder = Get-DnsServerForwarder |
    Where-Object IPAddress -eq "192.168.20.126"

if (-not $Forwarder) {
    Add-DnsServerForwarder `
        -IPAddress "192.168.20.126" `
        -PassThru
}
```

Confirm the forwarder:

```powershell
Get-DnsServerForwarder
```

## 11. Validate the deployment

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

## 12. Expected final state

```text
Computer name : U01PARVMDOM01
AD domain     : corp.unreadlines.com
NetBIOS name  : UNREADLINES
IPv4 address  : 192.168.20.41/25
Gateway       : 192.168.20.126
DNS client    : 192.168.20.41
DNS forwarder : 192.168.20.126
```

Do not install the PKI, Entra Connect, NPS, DHCP, or application roles on this domain controller. Add dedicated servers as the infrastructure grows.

## 13. Update the infrastructure inventory

Per `reference/server-naming-convention.md` §2, record the assigned name, role, site, IP address and
owner in `reference/vm-inventory.md`. `U01PARVMDOM01` is already listed there as the AD DS/DNS server
for Subnet 1 — confirm the row still matches what was actually deployed.

## 14. References

- [Install Active Directory Domain Services — Microsoft Learn](https://learn.microsoft.com/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-)
- [`Install-ADDSForest` — Microsoft Learn](https://learn.microsoft.com/powershell/module/addsdeployment/install-addsforest)
- [Forest and domain functional levels — Microsoft Learn](https://learn.microsoft.com/windows-server/identity/ad-ds/active-directory-functional-levels)
- [DNS forwarders — Microsoft Learn](https://learn.microsoft.com/windows-server/networking/dns/quickstart-install-configure-dns-server)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
