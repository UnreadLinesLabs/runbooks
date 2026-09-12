# Deploy the Intune SCEP certificate profile for mobile client authentication

This runbook hands each managed mobile user a client authentication certificate — the credential
`UnreadLines-Mobile` will eventually check during the EAP-TLS handshake. It is item 7 of the chain
described in `nps-server-certificate-deployment/README.md` §3 (frozen at the time that runbook was
written — `YouTube/backlog.md` §2 is the up-to-date reference for the full chain), and depends on every
piece of infrastructure the previous items already built: the CA hierarchy trusted by devices
(`intune-trusted-certificate-profiles/README.md`), the certificate template and NDES server
(`ndes-scep-intune-connector/README.md`), and the internet-reachable, narrowly-scoped path to that
server (`entra-private-network-connector-app-proxy/README.md`).

Two things are new here that none of those runbooks cover: connecting the on-premises AD group this
profile targets to Entra ID, and configuring the certificate's Subject Alternative Name so NPS can later
apply a *strong* certificate mapping (`KB5014754`) rather than a weak one.

## 1. Architecture

```text
Managed phone — Android Enterprise (Corporate or Personal/BYOD), or iOS/iPadOS
      |  already trusts the UnreadLines CA hierarchy
      |  (intune-trusted-certificate-profiles/README.md)
      v
Microsoft Intune — SCEP certificate profile, one per profile type (this runbook, §7-§9)
      Subject : CN={{UserPrincipalName}}
      SAN     : UPN = {{UserPrincipalName}}
                URI = {{OnPremisesSecurityIdentifier}}   <- strong mapping, KB5014754
      EKU     : Client Authentication only
      |  assigned to
      v
GG-U01-PAR-WiFi-Mobile — on-premises AD group
      synced to Entra ID via this runbook (§5-§6) — was on-prem only before
      |  device requests a certificate over SCEP, through
      v
Entra Application Proxy -> Entra Private Network Connector -> NDES (U01PARVMNDS01)
      (entra-private-network-connector-app-proxy/README.md, ndes-scep-intune-connector/README.md)
      |  signs against
      v
AD CS Issuing CA (U01PARVMPKI02) — IntuneSCEPMobileUser template
      |
      v
Certificate installed on the phone — SAN carries both the UPN and the on-premises SID. Once
UnreadLines-Mobile exists (§11, next step), NPS forwards the authentication as usual; it is the
domain controller that verifies the strong certificate mapping against that SID.
```

## 2. Scope and dependencies

This runbook creates three Intune SCEP certificate profiles — Android Enterprise Corporate, Android
Enterprise Personal (BYOD), and iOS/iPadOS — each issuing a client authentication certificate to members
of `GG-U01-PAR-WiFi-Mobile`. That's three profiles across two platform families, not three platforms:
Corporate and Personal (BYOD) are both Android Enterprise, distinguished only by enrollment mode; the
third profile is iOS/iPadOS. It also does the groundwork that profile's targeting depends on: creating
the `Synced` sub-OU, moving `GG-U01-PAR-WiFi-Mobile` into it, and extending Microsoft Entra Connect's OU
filtering so the group reaches Entra ID.

**It does not change any assignment on the eight Trusted certificate profiles from
`intune-trusted-certificate-profiles/README.md`.** Microsoft recommends deploying the Trusted Root
certificate profile and the SCEP profile "to the same groups" — the reason being that a device must
already hold the Root CA certificate before it can be handed a SCEP-issued one. Adding
`GG-U01-PAR-WiFi-Mobile` (a synced *user* group) onto the trust profiles would mix it with their existing
`SG-U00-Intune-TrustedCert-*` groups, which are *device* groups scoped by platform and ownership only —
a different targeting mechanism built for a broader purpose (every managed device of that platform trusts
the CA hierarchy, independently of Wi-Fi Mobile eligibility, per lab 6's own scope). The requirement is
already satisfied without touching lab 6 at all: a device only ever receives the SCEP profile once it is
enrolled and managed, and any enrolled device of a targeted platform is, by construction of `SG-U00-
Intune-TrustedCert-*`'s membership rule, already covered by the matching trust profile. Same coverage,
no shared group object needed — see §4's Prerequisites for how this is confirmed rather than assumed.

**No Windows profile is created here.** A Windows SCEP profile using this exact Subject/SAN shape
(`CN={{UserPrincipalName}}`, Client Authentication only) would put a second, unrelated flow onto the same
"Client Authentication + SAN=UPN" pattern already used for phones — precisely the collision
`notes/wifi-mobile-certificate-chain.md` §7.2 flags and deliberately defers, on the condition that only
one flow uses that pattern. Restricting this lab to mobile-platform profile types only (no Windows, no
macOS profile exists to assign) is also what satisfies `notes/wifi-mobile-certificate-chain.md` §5's "no
PC reuse" requirement: a device filter object is unnecessary, because a PC has no matching profile type
to receive in the first place. If `UnreadLines-Corp` (idea, not engaged — `YouTube/backlog.md` §3) is
ever built on the same certificate pattern, §7.2's condition is met and the OID/EKU marker it describes
becomes a real prerequisite, not before.

This runbook does **not** create the Wi-Fi EAP-TLS profile (item 8), activate `RAP01` on
`UnreadLines-Mobile` (item 9), or validate a real EAP-TLS handshake in the NPS logs (item 10) — see
`YouTube/backlog.md` §2 for the full chain.

Depends on: `ad-cs-pki-deployment/README.md` (Issuing CA issuing), `ndes-scep-intune-connector/README.md`
(Certificate Connector confirmed **Active**), `entra-private-network-connector-app-proxy/README.md`
(published external URL reachable), `intune-trusted-certificate-profiles/README.md` (device trust already
in place), `radius-nps-deployment/README.md` §8 (`GG-U01-PAR-WiFi-Mobile` already created, on-prem only),
and `microsoft-entra-connect-sync/README.md` (sync pipeline already running, extended here).

This runbook adds no VM and changes no existing one, so it carries no "Update the infrastructure
inventory" section.

## 3. Target configuration

| Item | Value |
| --- | --- |
| Tenant | `unreadlines` |
| Targeted AD group | `GG-U01-PAR-WiFi-Mobile` (on-prem, synced to Entra ID in §6) |
| Certificate type | `User` |
| Subject name format | `CN={{UserPrincipalName}}` |
| SAN — UPN | `{{UserPrincipalName}}` |
| SAN — URI (strong mapping, `KB5014754`) | `{{OnPremisesSecurityIdentifier}}` |
| Extended Key Usage | Client Authentication only |
| Key usage | Digital Signature only — matches `IntuneSCEPMobileUser`'s `Signature`-only Request Handling (`ndes-scep-intune-connector/README.md` §6) |
| Key size | 2048-bit RSA, non-exportable |
| Hashing algorithm | SHA-256 |
| Certificate validity period | `1 year` (lab choice) — Microsoft does not recommend a specific value; its own guidance is only that this field must not exceed the template's own validity (verify live on `U01PARVMPKI02`, §4) and should plan for at least 5 days |
| Renewal threshold | 20% — Microsoft's own worked example in the SCEP profile documentation, no stronger recommendation given |
| SCEP Server URLs | `https://AppProxyNDESSCEPMobile-unreadlines.msappproxy.net/certsrv/mscep/mscep.dll` |

**Three profiles — created in §7-§9:**

| Profile | Root Certificate (Intune field, §4) | Assigned group |
| --- | --- | --- |
| `SCEP Certificate - UnreadLines Mobile User - Android (Corp)` | `Trusted Certificate - UnreadLines Root CA - Android (Corp)` | `GG-U01-PAR-WiFi-Mobile` |
| `SCEP Certificate - UnreadLines Mobile User - Android (BYOD)` | `Trusted Certificate - UnreadLines Root CA - Android (BYOD)` | `GG-U01-PAR-WiFi-Mobile` |
| `SCEP Certificate - UnreadLines Mobile User - iOS/iPadOS` | `Trusted Certificate - UnreadLines Root CA - iOS/iPadOS` | `GG-U01-PAR-WiFi-Mobile` |

The **Root Certificate** field always references the top-level Trusted Root profile, never the Issuing
CA's — the field validates the chain from the root down, and Intune resolves the intermediate separately
from the same trust deployment.

**Sub-OU created in §5:**

| Item | Value |
| --- | --- |
| Distinguished name | `OU=Synced,OU=Groups,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com` |
| Holds | `GG-U01-PAR-WiFi-Mobile` only, for now |

## 4. Prerequisites

- `ndes-scep-intune-connector/README.md` §12: the Certificate Connector confirmed **Active** in
  **Tenant administration → Connectors and tokens → Certificate connectors**.
- `entra-private-network-connector-app-proxy/README.md` §14: the published external URL returns `200` on
  a real SCEP operation (`?operation=GetCACaps&message=ca`).
- `intune-trusted-certificate-profiles/README.md`: all eight Trusted certificate profiles deployed; at
  least one Android Enterprise (Corporate and BYOD) and one iOS/iPadOS device already enrolled, for §10.
- **Confirm, don't assume, that `SG-U00-Intune-TrustedCert-*` still covers every device this lab targets**
  (`intune-trusted-certificate-profiles/README.md` §6) — each dynamic group's membership rule matches by
  platform and ownership only, with no dependency on Wi-Fi Mobile eligibility, so any enrolled device of
  the right platform qualifies. If that rule was ever narrowed since lab 6, re-open this prerequisite
  before relying on it.
- `GG-U01-PAR-WiFi-Mobile` exists on-prem (`radius-nps-deployment/README.md` §8) and its intended members
  are already in it — this runbook does not populate its membership.
- **Verify live on `U01PARVMPKI02`** (`certtmpl.msc` → `IntuneSCEPMobileUser` → General tab) what
  validity period the template actually allows, before setting §3's Intune-side value — it must not
  exceed the template's own.
- Domain Admins, or a delegation covering `OU=Groups,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,
  DC=com`, on `U01PARVMDOM01` — needed for §5.
- Hybrid Identity Administrator (or equivalent) on `U01PARVMECN01` — needed for §6.
- Intune Administrator (or Global Administrator) on the `unreadlines` tenant — needed for §7-§9.

No lab VM is touched directly in this runbook beyond `U01PARVMDOM01` (§5) and `U01PARVMECN01` (§6); every
other action happens in the Microsoft Intune admin center or the Microsoft Entra admin center — each
section's opening line says which.

## 5. Create the `Synced` sub-OU and move `GG-U01-PAR-WiFi-Mobile`

**On `U01PARVMDOM01`:**

```powershell
New-ADOrganizationalUnit -Name "Synced" `
  -Path "OU=Groups,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com"

Move-ADObject -Identity "CN=GG-U01-PAR-WiFi-Mobile,OU=Groups,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com" `
  -TargetPath "OU=Synced,OU=Groups,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com"
```

Confirm the move:

```powershell
Get-ADGroup GG-U01-PAR-WiFi-Mobile | Select-Object DistinguishedName
```

Only `GG-U01-PAR-WiFi-Mobile` moves. This sub-OU exists specifically to give Microsoft Entra Connect's OU
filtering (§6) something narrower than the whole `Groups` OU to select — every other group under
`OU=Groups,OU=PAR,...` stays exactly where it is, unsynced.

## 6. Extend Microsoft Entra Connect OU filtering

**On `U01PARVMECN01`:**

1. Open **Microsoft Entra Connect Sync** → **Configure**.
2. Choose **Customize synchronization options**, sign in with a Hybrid Identity Administrator account.
3. Step through to **Domain and OU filtering** — the tree already shows `Users` checked under `PAR`
   (`microsoft-entra-connect-sync/README.md` §8). Expand `PAR` → `Groups`, and check only the new
   `Synced` sub-OU. Leave `Groups` itself, and every other OU, exactly as they were.
4. Continue to **Configure** and let the wizard finish. It starts a sync cycle automatically.

Confirm the group actually reached Entra ID once the cycle completes:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

**Microsoft Entra admin center → Groups** — search `GG-U01-PAR-WiFi-Mobile`. It should now exist as a
synced, on-premises-sourced group. If it doesn't appear after a delta cycle, run a full
`Start-ADSyncSyncCycle -PolicyType Initial` before assuming the filtering step itself failed.

## 7. Deploy the Android Enterprise Corporate SCEP profile

**Microsoft Intune admin center → Devices → Configuration → Create → New policy.**

Platform **Android Enterprise**, profile type **Corporate-owned, fully managed** → **Templates** → **SCEP
certificate**.

- **Basics**: name it `SCEP Certificate - UnreadLines Mobile User - Android (Corp)`.
- **Configuration settings**: every field from §3's target configuration table, **Root Certificate** set
  to `Trusted Certificate - UnreadLines Root CA - Android (Corp)`.
- **Assignments**: **Included groups** → `GG-U01-PAR-WiFi-Mobile`.
- **Review + create**.

The **Corporate-owned, fully managed** profile type technically covers three Android Enterprise
enrollment modes: Fully Managed, Corporate-Owned Work Profile, and Dedicated. This profile still only
serves members of `GG-U01-PAR-WiFi-Mobile` who enroll a Fully Managed or Corporate-Owned Work Profile
device — a **Dedicated** device (kiosk-style, no user affinity) cannot receive it in practice, because
`{{UserPrincipalName}}` has no signed-in user to resolve against. No separate exclusion is needed here;
it follows from the certificate request simply failing to resolve a variable it cannot fill.

## 8. Deploy the Android Enterprise Personal (BYOD) SCEP profile

**Microsoft Intune admin center → Devices → Configuration → Create → New policy.**

Platform **Android Enterprise**, profile type **Personally-owned devices with work profile** →
**Templates** → **SCEP certificate**.

- **Basics**: name it `SCEP Certificate - UnreadLines Mobile User - Android (BYOD)`.
- **Configuration settings**: same values as §7, **Root Certificate** set to `Trusted Certificate -
  UnreadLines Root CA - Android (BYOD)`.
- **Assignments**: **Included groups** → `GG-U01-PAR-WiFi-Mobile`.
- **Review + create**.

## 9. Deploy the iOS/iPadOS SCEP profile

**Microsoft Intune admin center → Devices → Configuration → Create → New policy.**

Platform **iOS/iPadOS** → **Templates** → **SCEP certificate**.

- **Basics**: name it `SCEP Certificate - UnreadLines Mobile User - iOS/iPadOS`.
- **Configuration settings**: same values as §7, **Root Certificate** set to `Trusted Certificate -
  UnreadLines Root CA - iOS/iPadOS`.
- **Assignments**: **Included groups** → `GG-U01-PAR-WiFi-Mobile`.
- **Review + create**.

## 10. Verify the deployment

**Devices → Configuration** lists all three profiles. For each, confirm:

- **Root Certificate** points at the matching platform's Root CA trust profile (§3) — the setting most
  likely to be wrong if a step was rushed.
- **Assignments** shows `GG-U01-PAR-WiFi-Mobile` as an Included group.

If a device already enrolled and its user is a member of `GG-U01-PAR-WiFi-Mobile`, its SCEP profile's
device status should read **Succeeded** rather than **Pending** or **Error**. On that device, open the
issued certificate and confirm both SAN entries are present: the UPN, and a URI entry formatted
`tag:microsoft.com,2022-09-14:sid:<SID>` — the second is the strong-mapping payload, and the specific
point worth checking on every platform actually enrolled.

**What this does not prove:** that NPS accepts this certificate — `UnreadLines-Mobile` doesn't exist yet
(items 8-9), and confirming NPS actually honors the strong mapping is item 10's job, not this runbook's.
This runbook's success condition is narrower: three correctly scoped SCEP profiles exist, each pointing
at the right root trust, assigned to the right group — and, where a device is available, a certificate
with the right Subject/SAN actually lands on it.

## 11. Next step — Intune Wi-Fi EAP-TLS profile

Phones enrolled and permitted can now request a client authentication certificate, but nothing yet tells
them to use it for `UnreadLines-Mobile`. That is item 8 of the chain: an Intune Wi-Fi profile with
`Trusted server CA` set to the Root CA and RADIUS server names explicitly filled in, referencing the SCEP
profiles built here as its client certificate source.

## 12. References

- [Use SCEP certificate profiles with Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/intune/intune-service/protect/certificates-profile-scep)
- [Support tip: Implementing strong mapping in Microsoft Intune certificates — Microsoft Community Hub](https://techcommunity.microsoft.com/blog/intunecustomersuccess/support-tip-implementing-strong-mapping-in-microsoft-intune-certificates/4053376)
- [KB5014754 — Certificate-based authentication changes on Windows domain controllers — Microsoft Support](https://support.microsoft.com/en-us/topic/kb5014754-certificate-based-authentication-changes-on-windows-domain-controllers-ad2c23b0-15d8-4340-a468-4d4f3b188f16)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
