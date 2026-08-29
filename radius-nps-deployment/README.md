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

`NPAS` does not include the AD PowerShell module (`Get-ADGroupMember`, `New-ADGroup`, etc.) — don't install `RSAT-AD-PowerShell` here just to get it. `U01PARVMDOM01` already has the `ActiveDirectory` module natively, as a domain controller, so every AD-native command in this lab (§2, §3, §8) is run there instead — the same split `ad-cs-pki-deployment/README.md` uses for its own GPO steps ("Switch servers for this part — the `GroupPolicy` module isn't necessarily installed on `U01PARVMPKI02`. Use `U01PARVMDOM01`.").

## 2. Register NPS in Active Directory

On `U01PARVMNPS01`, registering the server authorizes it to read dial-in/network-access properties from user and computer accounts in this domain, by adding its computer account to the built-in `RAS and IAS Servers` group:

```powershell
netsh ras add registeredserver
```

**On `U01PARVMDOM01`**, confirm:

```powershell
Get-ADGroupMember -Identity "RAS and IAS Servers" |
    Select-Object Name, objectClass
```

`U01PARVMNPS01` must appear in the output.

## 3. Create the Wi-Fi Mobile Access Group

**On `U01PARVMDOM01`.** Following the group naming convention (`standards/server-naming-convention.md` §4), create a dedicated security group to scope who is allowed to authenticate on `UnreadLines-Mobile`, rather than granting the Network Policy to a broad built-in group. The name mirrors the SSID it governs, the same way `hostapd-wifi-access-point-lab/README.md` §13 names `UnreadLines-Corp` for a future, separate workstation group:

```powershell
New-ADGroup `
    -Name "GG-U01-PAR-WiFi-Mobile" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=Groups,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com" `
    -Description "Members permitted to authenticate on UnreadLines-Mobile (802.1X EAP-TLS)"
```

Leave the group empty for now — there is no real `UnreadLines-Mobile` user or device to add yet. Membership only starts mattering once a pilot mobile is actually enrolled in Intune and needs Wi-Fi access, which is later work (`infrastructure/plan-wifi-eap-tls-scep-intune.md`, video 13), not part of this lab. When that day comes, adding a member looks like this — replace `<SamAccountName>` with a real existing account or computer name, never run it with a placeholder value:

```powershell
Add-ADGroupMember -Identity "GG-U01-PAR-WiFi-Mobile" -Members "<SamAccountName>"
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

Under `Policies`, NPS has two similarly-named nodes — **Connection Request Policies** and **Network Policies**. This step is the second one. §5 already confirmed the default Connection Request Policy needs no changes; don't create a new Connection Request Policy here.

```text
nps.msc → Policies → Network Policies → New
```

**Page 1 — Specify Network Policy Name and Connection Type:**

```text
Policy name                    : UnreadLines-Mobile - EAP-TLS
Type of network access server  : Unspecified
```

Leave it on **Unspecified**, not "Remote Access Server (VPN-Dial up)" or "Remote Desktop Gateway" — the wizard's own hint text says why: `RAP01` is an 802.1X-authenticating wireless access point, which is exactly the case the hint tells you to pick Unspecified for.

**Page 2 — Specify Conditions.** Each condition is added one at a time via the **Add...** button, not typed directly:

```text
Add... → NAS IPv4 Address → Add...
    Enter the IP address of the network access server : 192.168.20.145
    → OK

Add... → Windows Groups → Add...
    → Add Groups...
        Enter the object name to select : GG-U01-PAR-WiFi-Mobile
        → Check Names → OK
    → OK
```

`Windows Groups` sits under the "User or Machine Groups" category in the condition list — search for "group" in the Add Condition dialog's filter box if it isn't immediately visible.

**Page 3 — Specify Access Permission:**

```text
Access granted
```

**Page 4 — Configure Authentication Methods.** There is no literal "EAP-TLS" checkbox in this UI — it's called **Microsoft: Smart Card or other certificate**:

```text
Add... → Microsoft: Smart Card or other certificate → OK
```

Remove every other method already checked by the wizard's defaults (including MS-CHAP v2) — `UnreadLines-Mobile` is certificate-based, not password-based, and leaving a weaker method enabled would let NPS silently fall back to it.

**Page 5 — Configure Constraints.** Two of the five sub-pages are worth actually setting, not skipping:

```text
NAS Port Type
    Wireless - IEEE 802.11    : checked (only this one)
```

`NAS IPv4 Address` (§ Page 2) already scopes this policy to `RAP01` specifically, but adding `NAS Port Type` tightens it further: this policy only ever matches an actual 802.11 wireless association from that NAS, not some other RADIUS-speaking use of the same IP added later. This is the same condition the earlier, undocumented NPS attempt used (`infrastructure/analyse-pki-intune-nps.md` §5.7: "condition NAS Port Type = Wireless IEEE 802.11").

```text
Session Timeout
    Set the maximum session time to    : 8 hours
```

Without a session timeout, an accepted client's connection is trusted indefinitely once it's on — a certificate revoked five minutes after that client connected has no effect until the client happens to disconnect and reassociate on its own. A session timeout forces periodic reauthentication (a fresh EAP-TLS handshake, which re-checks the certificate against the current CRL), bounding how long a revoked-but-still-connected client can stay on `UnreadLines-Mobile`. 8 hours is a reasonable default for a workday device; tighten it later if the threat model calls for it.

Leave the rest at their defaults, and here's why each one doesn't apply:

```text
Idle Timeout             : not meaningful for Wi-Fi the way it is for dial-up/VPN — skip
Called Station ID        : would pin the policy to RAP01's specific radio MAC; too
                            brittle for a lab (swap the USB adapter, break the policy)
Day and time restrictions: no business-hours requirement for this lab — skip
```

**Page 6 — Configure Settings.** Genuinely nothing to change here for this lab — but worth knowing why each section is being skipped rather than just clicking through:

```text
RADIUS Attributes → Standard        : not needed. This is where Tunnel-Type /
                                       Tunnel-Medium-Type / Tunnel-Private-Group-ID
                                       would go for RADIUS-assigned dynamic VLANs
                                       per client — but hostapd-wifi-access-point-lab
                                       abandoned per-client VLANs entirely (§9): each
                                       SSID gets its own dedicated VMware network, not
                                       a shared one split by RADIUS-assigned VLAN. That
                                       mechanism doesn't apply to this topology.
RADIUS Attributes → Vendor Specific : not needed — RAP01 was declared with vendor
                                       "RADIUS Standard" in §4, not a vendor requiring
                                       custom VSAs.
Network Access Protection           : NAP is deprecated/removed on current Windows
                                       Server — ignore this section if it even appears.
Routing and Remote Access           : Multilink/BAP, IP Filters, Encryption, IP
                                       Settings are all RRAS/VPN/dial-up settings —
                                       irrelevant since Page 1 set "Type of network
                                       access server" to Unspecified for a wireless AP,
                                       not a VPN or dial-up server.
```

**Finish.**

**After the wizard closes**, check the policy's position in the `Network Policies` list — it must sit above the built-in "NPS Deny All" catch-all policy, or NPS never evaluates it. Right-click it → **Move Up** if needed.

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

**On `U01PARVMNPS01`.** Export the full NPS configuration for review and as a backup baseline, and confirm the role is installed:

```powershell
netsh nps show config

Get-WindowsFeature -Name NPAS
```

**On `U01PARVMDOM01`.** Confirm the AD registration and the access group are both in place:

```powershell
Get-ADGroupMember -Identity "RAS and IAS Servers" | Select-Object Name
Get-ADGroup -Identity "GG-U01-PAR-WiFi-Mobile"
```

### Connectivity and Shared-Secret Test (`radtest`)

**Before running anything: this test will end in `Access-Reject`, on every correctly-configured attempt, and that's the correct outcome — not a failure to troubleshoot.** `UnreadLines-Mobile - EAP-TLS` (§6) only accepts EAP-TLS from a `Wireless - IEEE 802.11` NAS port type; `radtest` speaks plain PAP over a generic NAS port and satisfies neither, by design. There is no version of this command that returns `Access-Accept` against that policy — real EAP-TLS access only starts working once the certificate work in "Next step" is done. The point of this test isn't the top-line result, it's what's underneath it: whether the *reason* for the reject is the authentication method (good — everything else works) or something else entirely (an actual problem). Read the whole subsection below before running the command, so an `Access-Reject` on screen doesn't read as something broken.

Because `UnreadLines-Mobile` itself can't authenticate yet, validate the RADIUS path independently with `radtest` from a Linux host that can reach `192.168.20.46` — `U01PARVMRAP01` is the natural place to run this, since it's the actual RADIUS client and it already sits in Subnet 2. Use a **real** domain account (its actual `sAMAccountName` and password) — never the `u783476512` placeholder from the naming-convention examples, which doesn't exist in AD and will always be rejected as an unknown identity regardless of everything else being correct.

By default `radtest` sends `NAS-IP-Address` as whatever the local machine's hostname resolves to (`127.0.1.1` on a typical Ubuntu `/etc/hosts` — not `RAP01`'s real management IP), which won't match this policy's `NAS IPv4 Address` condition (§6). Pass `RAP01`'s real address explicitly as the trailing `nasname` argument:

```bash
sudo apt install -y freeradius-utils

# radtest <user> <password> <radius-server> <nas-port> <shared-secret> <ppp-hint> <nasname>
radtest <real-domain-account> '<real-password>' 192.168.20.46 0 '<generated secret>' '' 192.168.20.145
```

What this test actually proves is narrower than "accepted or not", and worth checking precisely:

- **No reply at all (timeout)** → the shared secret doesn't match on one side, or the request never reached NPS (routing/firewall). Check `netsh nps show config` and the secret configured in §4.
- **A reply, and `radclient`/`radtest` doesn't complain about it** → the shared secret is correct. If the secret were wrong, `radclient` explicitly says so ("invalid Response Authenticator! (Shared secret is incorrect.)") instead of just showing the reject — its absence here is itself the confirmation.
- **`Access-Reject`, and the NPS Security event log names the reason** — check it on `U01PARVMNPS01`:

  ```powershell
  Get-WinEvent -LogName Security |
      Where-Object Id -eq 6273 |
      Select-Object -First 1 -ExpandProperty Message
  ```

  A reason naming the authentication method or NAS port type (not an unknown user or an unrecognized RADIUS client) confirms the whole chain up to that point — secret, NAS recognition, account resolution — is correct, and the only remaining gap is the certificate.

  Reference result from this lab's own run:

  ```text
  User → Security ID              : S-1-5-21-... (a real SID — not S-1-0-0 / NULL SID,
                                     the specific failure analyse-pki-intune-nps.md hit
                                     with an unresolved Entra UPN)
  RADIUS Client → Friendly Name   : U01PARVMRAP01
  NAS → NAS IPv4 Address          : 192.168.20.145
  Network Policy Name             : UnreadLines-Mobile - EAP-TLS   (not "NPS Deny All")
  Authentication Type / EAP Type  : PAP / -
  Reason Code                     : 66
  Reason                          : The user attempted to use an authentication method
                                     that is not enabled on the matching network policy.
  ```

  That combination — real account resolved, correct RADIUS client and NAS IP, the right Network Policy matched, and a reject that names only the authentication method — is the complete, correct outcome at this stage. Nothing further to fix here; proceed to "Next step."

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
radtest (PAP)      : Access-Reject, with a signed reply (no "invalid Response
                     Authenticator" warning) and a 6273 event citing method/NAS
                     port type, not an unknown user — this is the correct
                     outcome against an EAP-TLS-only policy, not a failure
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
