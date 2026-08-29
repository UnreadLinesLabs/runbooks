# Deploy a Windows NPS RADIUS Server for 802.1X Enterprise Wi-Fi

This lab deploys `U01PARVMNPS01`, a dedicated Windows Server running the **Network Policy Server (NPS)** role — Microsoft's RADIUS implementation. It will authenticate `UnreadLines-Mobile`, the WPA2/WPA3-Enterprise (802.1X EAP-TLS) SSID that `hostapd-wifi-access-point-lab/README.md` §13 plans for domain-managed, Intune-enrolled mobile devices — the first of two enterprise SSIDs on that naming plan (`UnreadLines-Corp`, for domain-joined workstations, comes later and is out of scope here). §13 defers building it to "once RADIUS/NPS exists" — that server is `U01PARVMNPS01`, built here.

NPS is installed as its **own dedicated server**, not on the domain controller: `active-directory-domain-controller/README.md` explicitly reserves `U01PARVMDOM01` for AD DS/DNS only and defers NPS, PKI, and other roles to dedicated servers as the infrastructure grows.

## Scope and Dependencies

This lab covers everything needed to stand up NPS as a working RADIUS server and validate it end-to-end at the protocol level: role installation, AD registration, the RADIUS client (NAS) definition, the AD group and Network Policy that will govern `UnreadLines-Mobile` access, firewall rules, and a `radtest` connectivity check.

It does **not** activate `UnreadLines-Mobile` on `U01PARVMRAP01`. That requires a certificate for NPS with the Server Authentication EKU, issued by `U01PARVMPKI02` — and `ad-cs-pki-deployment/README.md` deliberately stops short of publishing production certificate templates and autoenrollment ("Only after this checkpoint can production certificate templates, permissions, auto-enrollment, and other integrations begin — all of that is out of scope for this runbook."). Issuing that template, enrolling the NPS certificate, and swapping `hostapd` to `UnreadLines-Mobile` is the explicit "Next step" at the end of this document, not duplicated here. `UnreadLines-Corp` (the workstation-facing enterprise SSID on the same hostapd naming plan) is a separate, later effort — nothing here builds it.

## Target Configuration

| Item | Value |
| --- | --- |
| Server name | `U01PARVMNPS01` |
| Role | Network Policy Server (RADIUS) |
| Subnet | Subnet 1 (`192.168.20.0/25`) |
| IPv4 address | `192.168.20.46/25` |
| Default gateway | `192.168.20.126` (`U01PARVMFWL01`) |
| DNS server | `192.168.20.41` (`U01PARVMDOM01`) |
| AD domain | `corp.unreadlines.com` |
| Domain-joined | Yes |
| RADIUS client (NAS) | `U01PARVMRAP01` — `192.168.20.145` (Subnet 2) |
| Authentication port | UDP `1812` |
| Accounting port | UDP `1813` |
| Target SSID | `UnreadLines-Mobile` (WPA2/WPA3-Enterprise, 802.1X EAP-TLS) — see `hostapd-wifi-access-point-lab/README.md` §13 |
| Wi-Fi access group | `GG-U01-PAR-WiFi-Mobile` |

`RAP01` sits in Subnet 2 (`192.168.20.128/25`) while NPS sits in Subnet 1 — this is intentional, not an oversight. `openwrt-wireguard-site-to-site-lab/README.md` already establishes and verifies full routing between the two LAN subnets over the FWL01↔FWL02 WireGuard tunnel (§21, "Full end-to-end verification"), so RADIUS traffic from `RAP01` reaches `NPS01` the same way any other cross-subnet traffic already does. No additional OpenWrt routing or firewall change is required: the existing `lan ↔ wg` forwarding rules on both routers already accept it.

## Prerequisites

- `U01PARVMDOM01` (AD DS/DNS) is up and reachable.
- The FWL01↔FWL02 WireGuard tunnel is up and verified (`openwrt-wireguard-site-to-site-lab/README.md`, §20–21).
- `U01PARVMRAP01` is deployed with `UnreadLines-Guest` (WPA2-Personal) live and validated end-to-end. `UnreadLines-Mobile` itself is **not** pre-provisioned — `hostapd-wifi-access-point-lab/README.md` §13's naming plan (`UnreadLines-Guest` live, `UnreadLines-Mobile` built first among the enterprise SSIDs, `UnreadLines-Corp` later) builds it as its own follow-up (a new dedicated VMware network, e.g. `VMnet4`, not a VLAN — see §9) once this lab produces a working RADIUS server. This lab only needs `RAP01`'s existing management IP (`192.168.20.145` on `ens33`) to register it as a RADIUS client — nothing about `UnreadLines-Mobile`'s future network.
- A supported, fully updated Windows Server installation for `U01PARVMNPS01`, with local Administrator access.
- A static IPv4 address reserved for this server: `192.168.20.46/25`.
- The server name approved according to the infrastructure naming convention (`standards/server-naming-convention.md` already lists `U01PARVMNPS01` as the canonical NPS example).
- **Not required for this lab**: a production certificate template on `U01PARVMPKI02`. That gap is tracked in the "Next step" section below and blocks only the final `UnreadLines-Mobile` activation, not anything in this document.

## Initial Manual Preparation

1. Configure the final IPv4 address and subnet manually:
    - IPv4 address: `192.168.20.46`
    - Prefix length: `/25` (subnet mask `255.255.255.128`)
    - Default gateway: `192.168.20.126`
    - DNS server: `192.168.20.41`
2. Configure the final computer name manually: `U01PARVMNPS01`.
3. Enable Remote Desktop manually so the server can be administered through mRemoteNG.
4. Connect to the server with mRemoteNG and continue the remaining steps in an elevated PowerShell session.

### Verify the Network Configuration

```powershell
$ActiveInterface = Get-NetAdapter |
    Where-Object Status -eq "Up" |
    Select-Object -First 1

Get-NetIPConfiguration -InterfaceIndex $ActiveInterface.ifIndex |
    Select-Object InterfaceAlias, IPv4Address, IPv4DefaultGateway, DNSServer
```

Confirm these values before continuing:

```text
IPv4Address       : 192.168.20.46/25
IPv4DefaultGateway: 192.168.20.126
DNSServer         : 192.168.20.41
```

### Join the Domain

```powershell
Add-Computer `
    -DomainName "corp.unreadlines.com" `
    -Credential (Get-Credential) `
    -NewName "U01PARVMNPS01" `
    -Restart
```

Reconnect after the restart using a domain account with local Administrator rights on this server and enough AD rights to modify the built-in `RAS and IAS Servers` group (Domain Admins, or an explicit delegation) — needed for the registration step in §2. `GG-U01-PAR-WiFi-Mobile` (§3) is an access-control group for Wi-Fi clients, not an administrative delegation — it grants nothing here. A local (pre-domain-join) `Administrator` session is not sufficient beyond this point.

Verify the join before continuing:

```powershell
whoami

Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain

Test-ComputerSecureChannel -Verbose
```

Expect:

```text
whoami                     = UNREADLINES\<AdminAccount>
Name                       = U01PARVMNPS01
Domain                     = corp.unreadlines.com
PartOfDomain               = True
Test-ComputerSecureChannel = True
```

Do not continue to role installation before this is verified.

## 1. Install the Network Policy Server Role

```powershell
Install-WindowsFeature `
    -Name NPAS `
    -IncludeManagementTools
```

Confirm:

```powershell
Get-WindowsFeature -Name NPAS
```

## 2. Register NPS in Active Directory

Registering the server authorizes it to read dial-in/network-access properties from user and computer accounts in this domain, by adding its computer account to the built-in `RAS and IAS Servers` group.

```powershell
netsh ras add registeredserver
```

Confirm:

```powershell
Get-ADGroupMember -Identity "RAS and IAS Servers" |
    Select-Object Name, objectClass
```

`U01PARVMNPS01` must appear in the output.

## 3. Create the Wi-Fi Mobile Access Group

Following the group naming convention (`standards/server-naming-convention.md` §4), create a dedicated security group to scope who is allowed to authenticate on `UnreadLines-Mobile`, rather than granting the Network Policy to a broad built-in group. The name mirrors the SSID it governs, the same way `hostapd-wifi-access-point-lab/README.md` §13 names `UnreadLines-Corp` for a future, separate workstation group:

```powershell
New-ADGroup `
    -Name "GG-U01-PAR-WiFi-Mobile" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groups,OU=PAR,OU=U01,DC=corp,DC=unreadlines,DC=com" `
    -Description "Members permitted to authenticate on UnreadLines-Mobile (802.1X EAP-TLS)"
```

Add members as needed (users and/or computer accounts, once EAP-TLS is active):

```powershell
Add-ADGroupMember -Identity "GG-U01-PAR-WiFi-Mobile" -Members "u783476512"
```

## 4. Add the RADIUS Client (NAS)

`hostapd` on `U01PARVMRAP01` — not `FWL02` — is the RADIUS client: it is the process that will send Access-Request packets directly to NPS once `UnreadLines-Mobile` is built and active (`hostapd-wifi-access-point-lab/README.md` §13). Generate a strong shared secret and store it in a password vault before this step; it will be needed again in `hostapd.conf`.

```text
nps.msc → NPS (Local) → RADIUS Clients and Servers → RADIUS Clients → New

  Enable this RADIUS client        : checked
  Friendly name                    : U01PARVMRAP01
  Address (IP or DNS)              : 192.168.20.145
  Shared secret                    : <generated secret>
  Vendor name                      : RADIUS Standard
```

Confirm reachability from NPS's side before moving on:

```powershell
Test-NetConnection -ComputerName 192.168.20.145 -Port 22
```

(Any open TCP port on `RAP01` is enough to confirm routing; RADIUS itself is UDP and won't respond to an unsolicited packet, which is expected.)

## 5. Leave the Default Connection Request Policy

A single, local NPS server needs no custom Connection Request Policy: the built-in "Use Windows authentication for all users" policy already routes every request to local Windows/AD authentication. Confirm it's present and unmodified:

```text
nps.msc → Policies → Connection Request Policies
```

## 6. Create the Network Policy for `UnreadLines-Mobile`

```text
nps.msc → Policies → Network Policies → New

  Policy name : UnreadLines-Mobile - EAP-TLS

  Conditions:
    - NAS IPv4 Address    : 192.168.20.145
    - Windows Groups      : GG-U01-PAR-WiFi-Mobile

  Access permission       : Access granted

  Authentication methods  : EAP-TLS only
                            (remove MS-CHAP v2 and any other method —
                            UnreadLines-Mobile is certificate-based, not
                            password-based)

  Constraints             : defaults are fine for the lab

  Order                   : above the built-in "NPS Deny All" catch-all policy
```

This policy is created now so the AD group, the RADIUS client, and the policy logic are all in place and reviewable. It cannot actually admit a client yet — EAP-TLS requires NPS to present a Server Authentication certificate, which does not exist until the "Next step" section below is completed. Attempting a real EAP-TLS handshake against this server today fails at the TLS layer with no valid server certificate to offer, which is expected.

## 7. Open the Windows Firewall for RADIUS

Installing the NPAS role registers the relevant firewall rules, but confirm they are enabled:

```powershell
Get-NetFirewallRule -DisplayGroup "Network Policy Server" |
    Select-Object DisplayName, Enabled

Get-NetFirewallRule -DisplayGroup "Network Policy Server" |
    Enable-NetFirewallRule
```

## 8. Validate the Deployment

Export the full NPS configuration for review and as a backup baseline:

```powershell
netsh nps show config
```

Confirm the role, AD registration, RADIUS client, and policy are all in place:

```powershell
Get-WindowsFeature -Name NPAS
Get-ADGroupMember -Identity "RAS and IAS Servers" | Select-Object Name
Get-ADGroup -Identity "GG-U01-PAR-WiFi-Mobile"
```

### Connectivity and Shared-Secret Test (`radtest`)

Because `UnreadLines-Mobile` itself can't authenticate yet (§6), validate the RADIUS path independently with `radtest` from a Linux host that can reach `192.168.20.46` — `U01PARVMRAP01` is the natural place to run this, since it's the actual RADIUS client and it already sits in Subnet 2:

```bash
sudo apt install -y freeradius-utils

# radtest <user> <password> <radius-server> <nas-port> <shared-secret>
radtest u783476512 'TestPassword123!' 192.168.20.46 0 '<generated secret>'
```

A domain account that has network dial-in permission allowed (or "Control access through NPS Network Policy" with a matching PAP-permitting test policy) returns `Access-Accept`; a wrong shared secret returns no reply at all (silently dropped, by RADIUS design — check `netsh nps show config` and the shared secret on both ends if that happens). This only proves the RADIUS path, the shared secret, and basic PAP authentication work — it does not validate EAP-TLS, which needs the certificate work below.

## Expected Final State

```text
Computer name      : U01PARVMNPS01
AD domain          : corp.unreadlines.com
IPv4 address       : 192.168.20.46/25
Gateway            : 192.168.20.126
DNS client         : 192.168.20.41
Role               : NPS (Network Policy and Access Services)
Registered in AD   : Yes (RAS and IAS Servers)
RADIUS client      : U01PARVMRAP01 (192.168.20.145)
Network Policy     : UnreadLines-Mobile - EAP-TLS (created, not yet usable)
radtest (PAP)      : Access-Accept
EAP-TLS            : Blocked on NPS server certificate — see Next step
```

## 9. Update the Infrastructure Inventory

Per `server-naming-convention.md` §2, record the assigned name, role, site, IP, and owner in `standards/vm-inventory.md` once deployed (already reflected there alongside this lab).

## Next Step — Issue the NPS Server Certificate and Activate `UnreadLines-Mobile`

1. On `U01PARVMPKI02`, publish a certificate template with the **Server Authentication** EKU (a duplicated `RAS and IAS Server` template is the standard starting point), scoped so only `U01PARVMNPS01` can enroll — this is the "production certificate templates" work `ad-cs-pki-deployment/README.md` explicitly deferred past its Checkpoint 2.
2. Enroll that certificate on `U01PARVMNPS01` (`certlm.msc` or `Get-Certificate`), then bind it to NPS's EAP properties (`nps.msc` → the Network Policy's EAP-TLS constraint → *Edit* → select the new certificate).
3. Since `UnreadLines-Mobile` is EAP-TLS end-to-end (client certificates too, not just the server side), issue a client-authentication certificate to each device or user permitted in `GG-U01-PAR-WiFi-Mobile` — via Intune/SCEP for these Intune-enrolled mobile devices, per `README-Intune-SCEP-WiFi-EAP-TLS.md`'s design.
4. Build `UnreadLines-Mobile` itself and switch `RAP01` over to it by following `hostapd-wifi-access-point-lab/README.md` §13: create the new dedicated VMware network (e.g. `VMnet4` — not a VLAN, see §9), wire it into `RAP01`'s Netplan and `FWL02`, then stop `hostapd` and edit `hostapd.conf` to `ssid=UnreadLines-Mobile`, `ieee8021x=1` / `wpa_key_mgmt=WPA-EAP` with `auth_server_addr=192.168.20.46`, the shared secret from §4 above, and `bridge=` set to that new network's bridge (replacing `br-vlan10`), then restart. `RAP01`'s single radio means `UnreadLines-Guest` goes offline while `UnreadLines-Mobile` is active.
5. Re-run the full 802.1X handshake from a real client and confirm `Access-Accept` in the NPS event log (`Get-WinEvent -LogName Security | Where-Object Id -in 6272,6273`).
6. `UnreadLines-Corp` (domain-joined workstations, same hostapd naming plan) is a later, separate effort — it needs its own RADIUS client entry, its own AD group, and likely its own Network Policy (machine authentication rather than user), not a rename of anything built here.

## References

- [Network Policy Server (NPS) overview — Microsoft Learn](https://learn.microsoft.com/windows-server/networking/technologies/nps/nps-top)
- [RFC 2865 — Remote Authentication Dial In User Service (RADIUS)](https://www.rfc-editor.org/rfc/rfc2865)
- [RFC 2866 — RADIUS Accounting](https://www.rfc-editor.org/rfc/rfc2866)
- [RFC 5216 — The EAP-TLS Authentication Protocol](https://www.rfc-editor.org/rfc/rfc5216)
- [FreeRADIUS `radtest` documentation](https://wiki.freeradius.org/config/Radtest)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
