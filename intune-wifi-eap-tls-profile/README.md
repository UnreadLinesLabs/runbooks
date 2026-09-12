# Deploy the Intune Wi-Fi EAP-TLS profile for UnreadLines-Mobile

This runbook builds the Wi-Fi profile that finally tells managed phones how to join
`UnreadLines-Mobile`: SSID, security type, and — critically — which certificate to present and which
server to trust when it does. It is item 8 of the chain described in
`nps-server-certificate-deployment/README.md` §3 (frozen at the time that runbook was written —
`YouTube/backlog.md` §2 is the up-to-date reference for the full chain), and it is a pure composition
step: every certificate it references already exists, from `intune-trusted-certificate-profiles/README.md`
(item 6, the Root/Issuing trust) and `intune-scep-certificate-profile/README.md` (item 7, the client
authentication certificate). Nothing here issues a new certificate.

Two settings carry the actual security weight, and both are easy to get wrong by accepting Intune's own
defaults without looking. **Root certificate for server validation** selects the Root CA trust profile as
the anchor of that verification — never the Issuing CA profile in this specific field. That doesn't leave
a gap: the Issuing CA already reaches every platform through its own, separate Trusted certificate profile
(`Trusted Certificate - UnreadLines Issuing CA - <platform>`, `intune-trusted-certificate-profiles/README.md`)
— this field only ever names the top of the chain because Intune resolves the intermediate from that same
trust deployment, not because the intermediate is missing from the device. The **RADIUS server names**
field is the second load-bearing setting: it pins the profile to the exact DNS name on `U01PARVMNPS01`'s
server certificate (`nps-server-certificate-deployment/README.md` §7-§8), rather than leaving devices to
accept any server whose certificate happens to chain to the trusted root. Skipping that second setting is
what would let a rogue access point — broadcasting the same SSID, holding any certificate issued by the
same CA, say a decommissioned lab server's — pass validation.

## 1. Architecture

```text
Managed phone — Android Enterprise (Corporate or Personal/BYOD), or iOS/iPadOS
      |  already trusts the UnreadLines CA hierarchy
      |  (intune-trusted-certificate-profiles/README.md)
      |  already holds a client authentication certificate
      |  (intune-scep-certificate-profile/README.md)
      v
Microsoft Intune — Wi-Fi profile, one per profile type (this runbook, §5-§7)
      SSID                : UnreadLines-Mobile
      Security type       : Enterprise, EAP-TLS
      Client certificate  : the matching SCEP profile from item 7
      Root certificate    : the matching Root CA trust profile from item 6 — never Issuing
      RADIUS server names : exact SAN DNS entry from U01PARVMNPS01's server certificate
      |  assigned to
      v
GG-U01-PAR-WiFi-Mobile — same group as item 7, already synced to Entra ID
      |  device attempts to join, through
      v
RAP01 (802.1X authenticator) — not yet switched onto UnreadLines-Mobile (item 9, next)
      |
      v
U01PARVMNPS01 (RADIUS / NPS) — presents its Server Authentication certificate
      (nps-server-certificate-deployment/README.md)
      |
      v
Phone validates: issued by the Root CA (trust profile above)? SAN matches a name from
"RADIUS server names" above? -- if both hold, phone presents its own client certificate.
Full handshake validated end to end in item 10, not here.
```

## 2. Scope and dependencies

This runbook creates three Intune Wi-Fi profiles — Android Enterprise Corporate, Android Enterprise
Personal (BYOD), and iOS/iPadOS — each configuring `UnreadLines-Mobile` as an EAP-TLS network and
assigning it to `GG-U01-PAR-WiFi-Mobile`. Same platform split as item 7, for the same underlying reason:
Intune ties a Wi-Fi profile's first setting to a platform choice
([Create a Wi-Fi profile for devices in Microsoft Intune](https://learn.microsoft.com/en-us/intune/device-configuration/templates/configure-wifi)),
so one profile object cannot cover Android and iOS at once, and Android Enterprise itself splits
Corporate-owned from Personally-owned into separate profile types at creation.

**No Windows profile is created here**, for the same reason item 7 created none: no Windows client
certificate exists to reference, and this chain deliberately keeps the "Client Authentication + SAN=UPN"
pattern to mobile devices only (`notes/wifi-mobile-certificate-chain.md` §7.2).

This runbook does **not**:

- Switch `RAP01` onto `UnreadLines-Mobile`, create `VMnet4`, or configure the restrictive firewall zone
  and DHCP scope on `FWL02` — that's item 9 (`YouTube/backlog.md` §2).
- Validate a real EAP-TLS handshake in the NPS logs, negative test included — that's item 10.
- Issue or reissue any certificate — every certificate referenced here already exists.

Depends on: `intune-trusted-certificate-profiles/README.md` (Root CA trust profiles per platform),
`intune-scep-certificate-profile/README.md` (client certificate profiles, and `GG-U01-PAR-WiFi-Mobile`
already synced to Entra ID), `nps-server-certificate-deployment/README.md` (the NPS server certificate
whose SAN this runbook pins).

This runbook adds no VM and changes no existing one, so it carries no "Update the infrastructure
inventory" section.

## 3. Target configuration

| Item | Value |
|---|---|
| Tenant | `unreadlines` |
| Network name (Apple) / SSID | `UnreadLines-Mobile` for both — the display name and the real broadcast name are the same string in this lab |
| Wi-Fi type / security type | Enterprise — EAP-TLS (the access point itself runs WPA2/WPA3-Enterprise; see item 9) |
| EAP type | `EAP-TLS` |
| Authentication method (client certificate) | `Certificates` — the matching SCEP profile from item 7 |
| Root certificate for server validation | The matching Root CA trust profile from item 6 — never Issuing (see the note above) |
| RADIUS / certificate server names | `U01PARVMNPS01.corp.unreadlines.com` — confirmed live off the certificate itself, §4, never guessed from the VM name |
| Identity privacy (outer identity) | `anonymous` — a fixed, non-identifying value sent before the TLS tunnel is up; separate concern from the SAN UPN/SID inside the certificate (§5-§7) |
| Deployment channel | iOS/iPadOS only (Android Enterprise has no such field) — `User channel`, because the linked SCEP certificate is a `User` certificate type (§3 of `intune-scep-certificate-profile/README.md`). Cannot be changed after the profile is deployed — a wrong choice means recreating the profile, not editing it |
| Targeted AD group | `GG-U01-PAR-WiFi-Mobile` — the same synced on-premises AD *user* group as item 7's three SCEP profiles, consistent with the `User` certificate type they issue (Microsoft's own guidance is to assign the trust, SCEP, and Wi-Fi profiles to the same population) |
| Connect automatically | Enabled where the setting is offered — confirm live, §6 |
| Hidden network | Disabled — `UnreadLines-Mobile` broadcasts its SSID |

**Three profiles — created in §5-§7:**

| Profile | Platform / profile type | Client certificate (item 7) | Root certificate (item 6) |
|---|---|---|---|
| `Wi-Fi - UnreadLines-Mobile - Android (Corp)` | Android Enterprise, Corporate-owned, fully managed | `SCEP Certificate - UnreadLines Mobile User - Android (Corp)` | `Trusted Certificate - UnreadLines Root CA - Android (Corp)` |
| `Wi-Fi - UnreadLines-Mobile - Android (BYOD)` | Android Enterprise, Personally-owned devices with work profile | `SCEP Certificate - UnreadLines Mobile User - Android (BYOD)` | `Trusted Certificate - UnreadLines Root CA - Android (BYOD)` |
| `Wi-Fi - UnreadLines-Mobile - iOS/iPadOS` | iOS/iPadOS | `SCEP Certificate - UnreadLines Mobile User - iOS/iPadOS` | `Trusted Certificate - UnreadLines Root CA - iOS/iPadOS` |

## 4. Prerequisites

- `intune-trusted-certificate-profiles/README.md`: all eight Trusted certificate profiles deployed and
  assigned — this runbook's **Root certificate for server validation** field selects from them.
- `intune-scep-certificate-profile/README.md`: all three SCEP profiles deployed, `GG-U01-PAR-WiFi-Mobile`
  synced to Entra ID, and at least one device of each platform family already holding an issued
  certificate — needed to test §8. Confirm specifically that each issued certificate carries: the SAN
  UPN, the SAN URI `{{OnPremisesSecurityIdentifier}}` required for the strong certificate mapping
  (`KB5014754`), the Client Authentication EKU, and `Certificate type: User` — this Wi-Fi profile assumes
  all four and creates none of them; if any is missing, that's a gap in item 7, not something to patch
  here.
- **Confirm `GG-U01-PAR-WiFi-Mobile` is still the on-premises AD *user* group synced to Entra ID that item
  7 populated** — not a device group. A `User`-type certificate (previous bullet) only ever lands on a
  device through a *user* assignment; targeting a device group here would silently break that link.
- **Read the exact RADIUS server name off the real certificate — do not type it from memory or guess it
  from the VM name.**

  **On `U01PARVMNPS01`:**

  ```powershell
  Get-ChildItem Cert:\LocalMachine\My |
      Where-Object {
          $_.Issuer -like "*UnreadLines Issuing CA*" -and
          $_.NotAfter -gt (Get-Date) -and
          $_.HasPrivateKey -and
          $_.EnhancedKeyUsageList.ObjectId -contains "1.3.6.1.5.5.7.3.1"
      } |
      Select-Object Subject, DnsNameList, Thumbprint, NotBefore, NotAfter
  ```

  Filtering on the Server Authentication EKU (`1.3.6.1.5.5.7.3.1`), a valid `NotAfter`, and a present
  private key narrows this to the certificates `U01PARVMNPS01` could actually present — but more than one
  can still match if a renewal already ran (`nps-server-certificate-deployment/README.md` §8 flags this as
  possible). Settle it against the certificate actually in use rather than assuming the newest one:

  ```text
  nps.msc → Policies → Network Policies → UnreadLines-Mobile - EAP-TLS → Properties
      → Constraints → Authentication Methods
      → Microsoft: Smart Card or other certificate → Edit...
  ```

  Note the thumbprint shown there and match it against the command's output above — that certificate's
  `DnsNameList` is the value that goes into §5-§7's server-name field.

  **Confirmed against this lab's tenant (12/09/2026)** — a single, unambiguous match, no renewal collision
  to resolve:

  ```text
  Subject     : CN=U01PARVMNPS01.corp.unreadlines.com
  DnsNameList : {U01PARVMNPS01.corp.unreadlines.com}
  ```

  `U01PARVMNPS01.corp.unreadlines.com` is the value used in §5-§7 below. Re-run the command rather than
  trusting this output once the certificate has renewed — the thumbprint check against `nps.msc` above is
  what catches that, not this snapshot.
- An account with the Intune **Policy and Profile Manager** built-in role (or an equivalent custom RBAC
  role scoped to creating and assigning device configuration profiles) on the `unreadlines` tenant — full
  Intune/Global Administrator is broader than this task needs.

No lab VM is touched directly beyond reading the certificate and Network Policy on `U01PARVMNPS01` in this
Prerequisites step; every configuration action from here on happens in the Microsoft Intune admin center.

## 5. Deploy the Android Enterprise Corporate Wi-Fi profile

**Microsoft Intune admin center → Devices → Configuration → Create → New policy.**

Platform **Android Enterprise**, profile type **Corporate-owned, fully managed** → **Templates** →
**Wi-Fi**.

- **Basics**: name it `Wi-Fi - UnreadLines-Mobile - Android (Corp)`, description "EAP-TLS Wi-Fi profile
  for UnreadLines-Mobile, Android Enterprise Corporate devices."
- **Configuration settings**:
  - **SSID**: `UnreadLines-Mobile`.
  - **Connect automatically**: `Enable`.
  - **Hidden network**: `Disable`.
  - **Wi-Fi type**: `Enterprise`.
  - **EAP type**: `EAP-TLS`.
  - **Radius server name**: `U01PARVMNPS01.corp.unreadlines.com` — confirmed live in §4 against the
    certificate actually bound to the Network Policy, not typed from the VM name.
  - **Root certificate for server validation**: `Trusted Certificate - UnreadLines Root CA - Android
    (Corp)` — never the Issuing CA profile.
  - **Authentication method**: `Certificates`, then select `SCEP Certificate - UnreadLines Mobile User -
    Android (Corp)`.
  - **Identity privacy (outer identity)**: `anonymous`. This is a different concern from the SAN UPN/SID
    inside the certificate (§3 of `intune-scep-certificate-profile/README.md`): the outer identity is
    what's sent in the clear before the TLS tunnel protecting the real identity is up, so a fixed,
    non-identifying value avoids leaking the real UPN at that stage. Confirm during §8/item 10 that the
    `UnreadLines-Mobile - EAP-TLS` Network Policy doesn't key off this value for anything — it shouldn't,
    but this is the first lab where the field is actually set to something other than blank.
- **Assignments**: **Included groups** → `GG-U01-PAR-WiFi-Mobile`.
- **Review + create**.

No **Deployment channel** field exists on this profile type — that setting is iOS/iPadOS-only (§7).

## 6. Deploy the Android Enterprise Personal (BYOD) Wi-Fi profile

**Microsoft Intune admin center → Devices → Configuration → Create → New policy.**

Platform **Android Enterprise**, profile type **Personally-owned devices with work profile** →
**Templates** → **Wi-Fi**.

- **Basics**: name it `Wi-Fi - UnreadLines-Mobile - Android (BYOD)`, description "EAP-TLS Wi-Fi profile
  for UnreadLines-Mobile, Android Enterprise Personal (BYOD) devices."
- **Configuration settings** — same fields and order as §5; only the certificate references change:
  - **SSID**: `UnreadLines-Mobile`.
  - **Connect automatically**: select `Enable` if the setting is offered for this profile type in the
    current Intune console — Microsoft's own field-availability notes scope some Wi-Fi settings to
    profile types where "Intune controls the entire device," a condition that doesn't describe a
    personally-owned work profile the way it describes the Corporate profile in §5. Record what's
    actually offered here rather than assuming §5's screen repeats exactly.
  - **Hidden network**: `Disable`.
  - **Wi-Fi type**: `Enterprise`.
  - **EAP type**: `EAP-TLS`.
  - **Radius server name**: same value as §5.
  - **Root certificate for server validation**: `Trusted Certificate - UnreadLines Root CA - Android
    (BYOD)`.
  - **Authentication method**: `Certificates` → `SCEP Certificate - UnreadLines Mobile User - Android
    (BYOD)`.
  - **Identity privacy (outer identity)**: `anonymous`, same reasoning as §5.
- **Assignments**: **Included groups** → `GG-U01-PAR-WiFi-Mobile`.
- **Review + create**.

No **Deployment channel** field exists on this profile type either (§5's note applies here too).

## 7. Deploy the iOS/iPadOS Wi-Fi profile

**Microsoft Intune admin center → Devices → Configuration → Create → New policy.**

Platform **iOS/iPadOS** → **Templates** → **Wi-Fi**.

- **Basics**: name it `Wi-Fi - UnreadLines-Mobile - iOS/iPadOS`, description "EAP-TLS Wi-Fi profile for
  UnreadLines-Mobile, iOS/iPadOS devices."
- **Configuration settings** — iOS/iPadOS names some of these fields differently from Android Enterprise,
  and adds one setting Android doesn't have at all:
  - **Network name**: `UnreadLines-Mobile` — the name shown to the user when browsing available
    networks. On this platform it's a distinct field from **SSID**, which is the network's real broadcast
    name; both are set to the same string here.
  - **SSID**: `UnreadLines-Mobile`.
  - **Connect automatically**: `Enable`.
  - **Hidden network**: `Disable`.
  - **Security type**: `WPA/WPA2 - Enterprise` — the current Intune documentation for this platform lists
    `WPA - Enterprise` and `WPA/WPA2 - Enterprise` only, no separate WPA3-Enterprise option. This Intune
    setting doesn't fully determine the radio-side security either way: the actual WPA2/WPA3 transition
    behavior for `UnreadLines-Mobile` is configured on `RAP01` itself, in item 9, independently of what
    this field is called here.
  - **EAP type**: `EAP-TLS`.
  - **Certificate server names**: same value as §5-§6 — plural field name on this platform, accepts more
    than one entry if ever needed, but only `U01PARVMNPS01`'s applies here.
  - **Root certificate for server validation**: `Trusted Certificate - UnreadLines Root CA - iOS/iPadOS`.
  - **Certificates**: `SCEP Certificate - UnreadLines Mobile User - iOS/iPadOS`.
  - **Deployment channel**: `User channel`. The linked SCEP profile issues a `User`-type certificate, and
    Microsoft's guidance is unconditional on this point: user certificates require the user channel,
    device certificates require the device channel. **This cannot be changed after the profile is
    deployed** — picking the wrong one means creating a new profile from scratch, not editing this one.
  - **Identity privacy (outer identity)**: `anonymous`, same reasoning as §5.
- **Assignments**: **Included groups** → `GG-U01-PAR-WiFi-Mobile`.
- **Review + create**.

## 8. Verify the deployment

**Devices → Configuration** lists all three profiles. For each, confirm:

- **Root certificate for server validation** points at the matching platform's Root CA trust profile
  (§3) — never Issuing.
- The RADIUS / certificate server name reads exactly `U01PARVMNPS01.corp.unreadlines.com` — §4's
  confirmed value, not a value typed from memory.
- On the iOS/iPadOS profile only: **Deployment channel** reads `User channel` — this is the one setting
  in this runbook that can't be fixed by editing the profile if it's wrong (§7).
- **Assignments** shows `GG-U01-PAR-WiFi-Mobile` as an Included group, and that group is still the synced
  on-premises AD *user* group from item 7 — not a device group.

If a device already enrolled and holds both its Root CA trust and its client certificate (items 6-7),
its Wi-Fi profile's device status should read **Succeeded** rather than **Pending** or **Error** — Intune
can push and apply a Wi-Fi profile even before `RAP01` broadcasts this SSID (item 9), since applying the
configuration doesn't require the network to be reachable yet.

**What this does not prove:** that a phone can actually join `UnreadLines-Mobile` — `RAP01` isn't
switched onto that SSID yet (item 9), and confirming a real EAP-TLS handshake, negative test included, is
item 10's job. This runbook's success condition is narrower: three correctly scoped Wi-Fi profiles exist,
each referencing the right root trust, the right client certificate, and the right RADIUS server name,
assigned to the right group.

## 9. Next step — switch `RAP01` to `UnreadLines-Mobile`

Phones now have both halves of what EAP-TLS needs — a client certificate (item 7) and the network
configuration to use it (this runbook) — but `RAP01` still isn't broadcasting `UnreadLines-Mobile`.
That's item 9: `VMnet4`, a restrictive firewall zone and DHCP scope on `FWL02`, and the `hostapd`
configuration itself (SSID, WPA2/WPA3-Enterprise, `auth_server_addr` pointing at `U01PARVMNPS01`).

## 10. References

- [Create a Wi-Fi profile for devices in Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/intune/device-configuration/templates/configure-wifi)
- [Add Wi-Fi settings for Android Enterprise devices in Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/intune/device-configuration/templates/ref-wifi-settings-android-enterprise)
- [Configure Wi-Fi settings for Apple devices in Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/intune/device-configuration/templates/ref-wifi-settings-apple)
- [Microsoft Intune built-in roles reference — Microsoft Learn](https://learn.microsoft.com/en-us/intune/intune-service/fundamentals/role-based-access-control-reference)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
