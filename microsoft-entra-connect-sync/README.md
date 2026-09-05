# Deploy Microsoft Entra Connect and synchronize Active Directory with Microsoft Entra ID

This runbook builds `U01PARVMECN01`, a dedicated Windows Server running Microsoft Entra Connect Sync,
and turns on the first live synchronization between the `corp.unreadlines.com` Active Directory domain
and the `unreadlines` Microsoft Entra tenant. It relies on
`active-directory-domain-controller/README.md` for the AD DS forest itself, and on the hybrid identity
design already decided in `reference/active-directory-entra-identity-design.md` — this runbook does not
repeat that design, it verifies the two prerequisites it depends on (the alternative UPN suffix in AD,
the verified and Primary `id.unreadlines.com` domain in Entra) and builds on top of them.

Password Hash Synchronization is the authentication method: the option Microsoft recommends by default,
with no extra agent and no dependency on on-premises availability for cloud sign-in. Synchronization is
scoped, through OU filtering, to standard user accounts (`u...`) only — AD admin accounts (`a...`),
service accounts and gMSA stay out of Entra, consistent with the admin-tier separation
`reference/naming-conventions.md` §5.3 already defines. That filtering has a real limit, stated where it
matters below (§2): it only works as long as admin accounts are not placed in the same OU as standard
users.

## 1. Architecture

```text
   Subnet 1 - 192.168.20.0/25                    Subnet 2 - 192.168.20.128/25
+----------------------------+               +--------------------------------+
| U01PARVMDOM01              |               | U01PARVMECN01                  |
| 192.168.20.41              |               | 192.168.20.146                 |
| AD DS / DNS                |               | Microsoft Entra Connect Sync   |
| corp.unreadlines.com       |               +---------------+----------------+
+--------------+-------------+                               |
               |                                              |
   U01PARVMFWL01 == WireGuard tunnel OR wired link == U01PARVMFWL02
        192.168.20.126        (one active at a time)     192.168.20.254 (NAT to internet)
               |                                              |
               +----------------- LDAP (cross-subnet) --------+
                                                                |
                                                    HTTPS 443, outbound only,
                                                    via FWL02 NAT
                                                                |
                                                                v
                                        Microsoft Entra tenant "unreadlines"
                                   id.unreadlines.com    Verified, Primary (UPN)
                                   unreadlines.com       Verified (mail / SMTP)
                                   unreadlines.onmicrosoft.com   native domain
```

`U01PARVMECN01` sits in Subnet 2, on the same host and behind the same router as the PKI and web
distribution servers, not next to the domain controller. That is deliberate, not an oversight: it
reaches `U01PARVMDOM01` over the same FWL01↔FWL02 cross-subnet path every other Subnet 2 domain-joined
server already uses for LDAP and DNS (`ad-cs-pki-deployment/README.md` §1), and it reaches Microsoft
outbound through `U01PARVMFWL02`'s existing NAT to the home Wi-Fi/Freebox uplink — no new routing or
firewall work on either OpenWrt router.

That cross-subnet path is now one of two, per `openwrt-wired-site-to-site/README.md`: the original
WireGuard tunnel over Wi-Fi, or a wired Ethernet interconnect added later — only one active at a time,
switchable without touching this lab's config either way (`reference/vm-inventory.md` records which is
currently active). If Entra Connect Sync ever shows intermittent sync failures or delays reaching
`U01PARVMDOM01` for LDAP/DNS, that's worth checking before assuming an Entra Connect problem: the
Wi-Fi tunnel measured real packet loss from radio interference in this lab (`openwrt-wired-site-to-site/
README.md` §5), and switching to the wired link (§10 there) removes it from the equation entirely.

## 2. Scope and dependencies

This runbook builds `U01PARVMECN01`, installs Microsoft Entra Connect in custom mode, configures
Password Hash Synchronization, scopes synchronization to a single OU through domain/OU filtering, runs
the first synchronization cycle, and validates that a pilot user appears correctly in Microsoft Entra
with the expected `@id.unreadlines.com` sign-in name.

It does **not** design the UPN/domain strategy — that decision is already made and documented in
`reference/active-directory-entra-identity-design.md`, which this runbook only verifies (§6). It does
not configure Conditional Access, break-glass account exclusions, password writeback, or Seamless SSO —
all tracked as open points in that design document (§8) and picked up in "Next step" (§12) rather than
here. It does not extend synchronization beyond the pilot OU, and it does not restructure the OU tree:
it assumes the tree in `reference/naming-conventions.md` §3 already exists as documented.

**A filtering limit to know before relying on it**: `reference/naming-conventions.md` §3 places every
human account — standard users (`u...`) and AD admins (`a...`) alike — in the same site `Users` OU;
only the naming convention (§5) tells them apart, not the OU tree. Domain/OU filtering (§8 below) can
therefore only keep admin accounts out of Entra as long as none actually exist inside that OU yet, which
is the case at the time of writing. If an `a...` account is ever created inside a site's `Users` OU,
OU filtering stops being sufficient and the exclusion has to move to an attribute-based Microsoft Entra
Connect sync rule (filtering on the `sAMAccountName` prefix) instead — `reference/active-directory-
entra-identity-design.md` §8.2 already flags this exact gap. Revisit this the day a real `a...` account
is created.

This lab is independent of the `UnreadLines-Mobile` Wi-Fi/RADIUS chain (`radius-nps-deployment/README.md`,
`nps-server-certificate-deployment/README.md`) — nothing there depends on it, and it depends on nothing
there.

## 3. Target configuration

| Item | Value |
| --- | --- |
| Server name | `U01PARVMECN01` |
| Role | Microsoft Entra Connect Sync |
| Subnet | Subnet 2 (`192.168.20.128/25`) |
| IPv4 address | `192.168.20.146/25` |
| Default gateway | `192.168.20.254` (`U01PARVMFWL02`) |
| DNS server | `192.168.20.41` (`U01PARVMDOM01`) |
| AD domain | `corp.unreadlines.com` |
| Domain-joined | Yes |
| Microsoft Entra tenant | `unreadlines` |
| UPN source attribute | `userPrincipalName` (already `<sAMAccountName>@id.unreadlines.com` on pilot accounts) |
| Authentication method | Password Hash Synchronization |
| Synchronization scope | Domain/OU filtering — one site `Users` OU, standard accounts (`u...`) only |

## 4. Prerequisites

- `corp.unreadlines.com` AD DS is healthy and `U01PARVMDOM01` is reachable
  (`active-directory-domain-controller/README.md`).
- `unreadlines.com` and `id.unreadlines.com` are added and **Verified** in Microsoft Entra, and
  `id.unreadlines.com` is set as the **Primary** domain — done already; reverified live in §6.
- `id.unreadlines.com` is added as an alternative UPN suffix in AD Domains and Trusts on
  `corp.unreadlines.com` — done already; reverified live in §6.
- At least one pilot AD user already has `userPrincipalName` set to
  `<sAMAccountName>@id.unreadlines.com`, per `reference/active-directory-entra-identity-design.md` §6.2.
  This runbook does not perform that migration, only depends on it.
- A Windows Server 2025 VM for `U01PARVMECN01`, fully updated, with local Administrator access.
- A static IPv4 address reserved for this server: `192.168.20.146/25`.
- The server name approved per `reference/naming-conventions.md` §2 (`ECN` role code).
- An AD account with Enterprise Admin rights, or an equivalent delegation, for the wizard's on-premises
  Active Directory connection.
- A Microsoft Entra **Hybrid Identity Administrator** (or Global Administrator) account for the
  wizard's cloud connection.
- Outbound HTTPS (`443`) from Subnet 2 to the internet through `U01PARVMFWL02`'s NAT — required for the
  installation wizard and the ongoing sync service to reach Microsoft's endpoints.

## 5. Initial manual preparation

1. Configure the final IPv4 address and subnet manually:
    - IPv4 address: `192.168.20.146`
    - Prefix length: `/25` (subnet mask `255.255.255.128`)
    - Default gateway: `192.168.20.254`
    - DNS server: `192.168.20.41`
2. Configure the final computer name manually: `U01PARVMECN01`.
3. Enable Remote Desktop manually so the server can be administered through mRemoteNG.
4. Connect to the server with mRemoteNG and continue the remaining steps in an elevated PowerShell
   session.

### 5.1 Verify the network configuration

**On `U01PARVMECN01`:**

```powershell
$ActiveInterface = Get-NetAdapter |
    Where-Object Status -eq "Up" |
    Select-Object -First 1

Get-NetIPConfiguration -InterfaceIndex $ActiveInterface.ifIndex |
    Select-Object InterfaceAlias, IPv4Address, IPv4DefaultGateway, DNSServer
```

Confirm these values before continuing:

```text
IPv4Address       : 192.168.20.146/25
IPv4DefaultGateway: 192.168.20.254
DNSServer         : 192.168.20.41
```

Confirm domain resolution before joining:

```powershell
Resolve-DnsName -Name "corp.unreadlines.com" -Server "192.168.20.41"
```

### 5.2 Join the domain

**On `U01PARVMECN01`:**

```powershell
Add-Computer `
    -DomainName "corp.unreadlines.com" `
    -Credential (Get-Credential) `
    -NewName "U01PARVMECN01" `
    -Restart
```

Reconnect after the restart using a domain account with local Administrator rights on this server. A
local (pre-domain-join) `Administrator` session is not sufficient beyond this point.

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
Name                       = U01PARVMECN01
Domain                     = corp.unreadlines.com
PartOfDomain               = True
Test-ComputerSecureChannel = True
```

Do not continue to installation before this is verified.

## 6. Verify the existing hybrid identity configuration

This is preparation already done before this runbook — `reference/active-directory-entra-identity-
design.md` §5–6 — reverified here so the two prerequisites Entra Connect depends on are shown, not just
assumed.

### 6.1 The alternative UPN suffix in Active Directory

**On `U01PARVMDOM01`:**

**GUI**: *Active Directory Domains and Trusts* → right-click the domain root → *Properties* →
*Alternative UPN suffixes* tab.

**PowerShell**:

```powershell
Get-ADForest | Select-Object -ExpandProperty UPNSuffixes
```

Expected result:

```text
id.unreadlines.com
```

### 6.2 The custom domains in Microsoft Entra

[Microsoft Entra admin center](https://entra.microsoft.com) → *Identity* → *Domain names* → *Custom
domain names*.

Expected result:

| Name | Status | Federated |
| --- | --- | --- |
| `id.unreadlines.com` | Verified | No |
| `unreadlines.com` | Verified | No |
| `unreadlines.onmicrosoft.com` | Available | No |

Click `id.unreadlines.com` and confirm:

```text
Type            : Custom
Status          : Verified
Federated       : No
Primary domain  : Yes
```

The tenant's *Overview* page also shows a **Microsoft Entra Connect** status widget — it reads
**Disabled** at this point. It flips to **Enabled** once §9 below runs its first cycle; that flip is
part of this runbook's own expected final state (§10), not something to troubleshoot here.

## 7. Install Microsoft Entra Connect

**On `U01PARVMECN01`:**

1. Download the latest Microsoft Entra Connect Sync installer from the **Microsoft Entra admin
   center** — not the Microsoft Download Center, which Microsoft stopped using for new Entra Connect
   Sync releases (see §13). In [Microsoft Entra admin center](https://entra.microsoft.com), left-hand
   menu, under **Entra ID** → **Entra Connect** → **Connect Sync** tab, the status reads **Microsoft
   Entra Connect sync: Not installed** with a **Download Microsoft Entra Connect Sync on Get Started >
   Manage tab** shortcut — follow it to the **Get started** tab's **Manage** sub-tab and download the
   installer from there. Copy it to `U01PARVMECN01`.
2. Run the installer as Administrator.
3. Accept the license terms and privacy notice.
4. On the **Express Settings** page, click **Customize** instead of using Express Settings — Express
   Settings syncs every OU in the forest with no filtering option, which would defeat the OU scoping
   decided in §2 and §3.

## 8. Configure Microsoft Entra Connect

Still inside the same custom-installation wizard, on `U01PARVMECN01`:

1. **Install required components** — leave the defaults (a local SQL Server Express LocalDB instance is
   fine for this lab's volume). Click **Install**.
2. **User sign-in** — select **Password Hash Synchronization**, leave **Enable single sign-on**
   unchecked (out of scope, see §12), click **Next**.
3. **Connect to Microsoft Entra ID** — sign in with the Hybrid Identity Administrator (or Global
   Administrator) cloud account from §4.
4. **Connect your directories** — click **Add Directory**, then **AD forest account**, and provide the
   Enterprise Admin (or delegated) credentials from §4 for `corp.unreadlines.com`.
5. **Microsoft Entra sign-in configuration** — confirm `userPrincipalName` is selected as the attribute
   used for the Microsoft Entra username; do not switch it to `mail` — the whole point of this design is
   keeping the two separate (`reference/active-directory-entra-identity-design.md` §2 and §7).
6. **Domain and OU filtering** — select **Sync selected domains and OUs**, expand
   `corp.unreadlines.com` → `UnreadLines` → `U01`, and check only the `Users` OU under the site actually
   holding pilot accounts (`PAR` at the time of writing). Leave `Workstations`, `Servers`, `Groups` and
   `ServiceAccounts` unchecked, and leave `MAR`/`BDX` unchecked until §12 extends the pilot. See the
   filtering limit called out in §2 before relying on this step to keep `a...` accounts out.
7. **Uniquely identifying your users** — leave the default (`ObjectGUID` as the source anchor).
8. **Filter users and devices** — leave **Synchronize all users and devices** (the OU filtering in step
   6 is the only scoping mechanism used here).
9. **Optional features** — leave everything unchecked. Password writeback, Group writeback and the
   other optional features are explicitly out of scope (§2, §12).
10. **Ready to configure** — leave **Start the synchronization process when configuration completes**
    checked, click **Install**, then **Exit** once it finishes.

## 9. Run and validate the initial synchronization

**On `U01PARVMECN01`:**

The installer already started the first cycle (§8, step 10). Confirm it completed without errors before
checking the result in Entra:

```powershell
Get-ADSyncConnectorRunStatus
```

Expect no connector reporting a running or failed state. If nothing is currently running and a previous
cycle was clean, trigger a fresh delta cycle explicitly:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

In [Microsoft Entra admin center](https://entra.microsoft.com) → *Identity* → *Users* → *All users*,
find the pilot user from §4 and confirm:

```text
User principal name : <sAMAccountName>@id.unreadlines.com
Source               : Windows Server AD
On-premises sync enabled : Yes
```

Return to the tenant *Overview* page (§6.2) and confirm the **Microsoft Entra Connect** widget now
reads **Enabled**.

## 10. Expected final state

```text
Computer name         : U01PARVMECN01
AD domain             : corp.unreadlines.com
IPv4 address          : 192.168.20.146/25
Gateway               : 192.168.20.254
DNS client            : 192.168.20.41
Role                  : Microsoft Entra Connect Sync
Sign-in method        : Password Hash Synchronization
Sync scope            : OU-filtered — one site Users OU, standard accounts (u...) only
Microsoft Entra Connect status : Enabled (was Disabled before this runbook)
Pilot user            : synced, UserPrincipalName ends in @id.unreadlines.com,
                         Source = Windows Server AD
```

## 11. Update the infrastructure inventory

Per `reference/naming-conventions.md` §2, record the assigned name, role, site, IP address and status in
`reference/vm-inventory.md` — already reflected there alongside this lab.

## 12. Next step — extend the sync beyond the pilot

1. Migrate the remaining AD users' UPN to `id.unreadlines.com`, following the dry-run-then-apply pattern
   in `reference/active-directory-entra-identity-design.md` §6.3.
2. Extend the domain/OU filtering in §8 to the `MAR` and `BDX` site `Users` OUs once they hold real
   accounts, and to the `Groups` OU once group-based licensing or Conditional Access needs it.
3. If an `a...` account is ever created inside a site's `Users` OU, replace the OU filtering in §8 with
   an attribute-based Microsoft Entra Connect sync rule that filters on the `sAMAccountName` prefix — the
   limit already flagged in §2.
4. Work through the open points `reference/active-directory-entra-identity-design.md` §8 still lists:
   break-glass account exclusion from every Conditional Access policy (§8.3), the UPN rollback procedure
   (§8.5), and testing the UPN/`mail` separation against the actual SSO/SAML/SCIM applications in use
   (§8.1).
5. Separate effort, surfaced while verifying §6.2 in this lab: the Global Administrator account
   currently used for tenant administration does not follow the opaque `cXXXXXXXXXX` cloud-admin format
   `reference/naming-conventions.md` §5.2 defines — worth a dedicated pass before hybrid identity work
   goes further.

## 13. References

- [Microsoft Entra Connect — Custom installation](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-custom) — the wizard walked through in §7–8.
- [Choose the right authentication method for your Microsoft Entra hybrid identity solution](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/choose-ad-authn) — Password Hash Sync vs. Pass-through Authentication vs. Federation, the decision made in §3.
- [Microsoft Entra Connect Sync: Configure filtering](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-sync-configure-filtering) — domain/OU filtering (§8) and the attribute-based alternative flagged in §2 and §12.
- [Microsoft Entra Connect: Design concepts](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/plan-connect-design-concepts) — referenced in `reference/active-directory-entra-identity-design.md` §12.
- [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history) — confirms the installer is exclusively available from the Microsoft Entra admin center (§7); the Microsoft Download Center no longer carries new releases.

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
