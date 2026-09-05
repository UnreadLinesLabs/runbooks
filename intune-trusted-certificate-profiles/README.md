# Deploy Intune trusted certificate profiles for the UnreadLines CA hierarchy

This runbook makes every Windows, Android Enterprise and iOS/iPadOS device managed by Intune trust the
UnreadLines certification authority hierarchy built in `ad-cs-pki-deployment/README.md`: the offline
`UnreadLines Root CA` (`U01PARVMPKI01`) and the online `UnreadLines Issuing CA` (`U01PARVMPKI02`). It
publishes eight Trusted certificate configuration profiles — one Root CA and one Issuing CA profile per
platform — each scoped to its own dynamic Entra ID device group, rather than assigned to every managed
device.

This is item 7 of the certificate and RADIUS chain toward a working `UnreadLines-Mobile` SSID, described
in `nps-server-certificate-deployment/README.md` §3. That lab issued the server-side certificate NPS
presents during the EAP-TLS handshake; this one is the first step on the client side — before any device
can be handed a client authentication certificate over SCEP (§12), it first has to trust the CA that will
issue it.

## 1. Architecture

```text
U01PARVMPKI01   Root CA, offline
      |  signs
      v
U01PARVMPKI02   Issuing CA          192.168.20.143
      |  both certificates published over HTTP — ad-cs-pki-deployment §12, §15
      v
http://pki.corp.unreadlines.com/
      UnreadLinesRootCA.crt         DER
      UnreadLinesIssuingCA.crt      Base-64 (PEM) — converted to DER in §5
      |
      v
Microsoft Intune — 8 Trusted certificate profiles
      Root CA    -> Windows · Android (Corp) · Android (BYOD) · iOS/iPadOS
      Issuing CA -> Windows · Android (Corp) · Android (BYOD) · iOS/iPadOS
      |  each pair assigned to
      v
Microsoft Entra ID — 4 dynamic device groups
      SG-U00-Intune-TrustedCert-Windows
      SG-U00-Intune-TrustedCert-AndroidCorp
      SG-U00-Intune-TrustedCert-AndroidBYOD
      SG-U00-Intune-TrustedCert-iOS
      |  membership rule matches devices by OS and ownership
      v
Enrolled devices trust the UnreadLines CA hierarchy
```

## 2. Scope and dependencies

This runbook covers publishing the eight Trusted certificate profiles and the four dynamic groups that
scope them. It depends on `ad-cs-pki-deployment/README.md` reaching Checkpoint 2 (§18): both CAs
issuing, both certificates published and reachable over HTTP.

It does **not** enroll devices into Intune, issue a client authentication certificate, or activate
`UnreadLines-Mobile`. Device enrollment is assumed already done for at least one device per platform —
verifying it is out of scope here. Issuing a client certificate to a device is the Intune SCEP profile
covered in §12, which itself needs NDES and the Intune Certificate Connector — neither exists yet. This
runbook adds no VM and changes no existing one, so it carries no "Update the infrastructure inventory"
section.

## 3. Target configuration

| Item | Value |
| --- | --- |
| Tenant | `unreadlines` |
| Root CA certificate | `http://pki.corp.unreadlines.com/UnreadLinesRootCA.crt` — DER, used as-is |
| Issuing CA certificate | `http://pki.corp.unreadlines.com/UnreadLinesIssuingCA.crt` — Base-64 (PEM), converted to DER in §5 |
| Profile type | Trusted certificate (per-platform template) |
| Windows destination store — Root CA profile | Computer certificate store — Root (default) |
| Windows destination store — Issuing CA profile | Computer certificate store — Intermediate |

**Dynamic Entra ID groups (4) — created first, in §6:**

| Group | Description | Membership rule |
| --- | --- | --- |
| `SG-U00-Intune-TrustedCert-Windows` | Windows devices enrolled in Intune — targeted by the two Windows Trusted certificate profiles (§7). | `(device.deviceOSType -eq "Windows")` |
| `SG-U00-Intune-TrustedCert-AndroidCorp` | Corporate-owned Android Enterprise devices — targeted by the Android (Corp) Trusted certificate profiles (§8). | `(device.deviceOSType -startsWith "Android") -and (device.deviceOwnership -eq "Company")` |
| `SG-U00-Intune-TrustedCert-AndroidBYOD` | Personally-owned (BYOD) Android Enterprise devices — targeted by the Android (BYOD) Trusted certificate profiles (§9). | `(device.deviceOSType -startsWith "Android") -and (device.deviceOwnership -eq "Personal")` |
| `SG-U00-Intune-TrustedCert-iOS` | iPhone and iPad devices — targeted by the iOS/iPadOS Trusted certificate profiles (§10). | `(device.deviceOSType -eq "iPad") -or (device.deviceOSType -eq "iPhone")` |

**Profiles (8) — created in §7–§10, each assigned to the group above that matches its platform:**

| Profile | Description (profile Basics tab) | Assigned group |
| --- | --- | --- |
| `Trusted Certificate - UnreadLines Root CA - Windows` | UnreadLines Root CA (U01PARVMPKI01) — Computer certificate store, Root. | `SG-U00-Intune-TrustedCert-Windows` |
| `Trusted Certificate - UnreadLines Issuing CA - Windows` | UnreadLines Issuing CA (U01PARVMPKI02) — Computer certificate store, Intermediate. | `SG-U00-Intune-TrustedCert-Windows` |
| `Trusted Certificate - UnreadLines Root CA - Android (Corp)` | UnreadLines Root CA (U01PARVMPKI01) — Android Enterprise, corporate-owned devices. | `SG-U00-Intune-TrustedCert-AndroidCorp` |
| `Trusted Certificate - UnreadLines Issuing CA - Android (Corp)` | UnreadLines Issuing CA (U01PARVMPKI02) — Android Enterprise, corporate-owned devices. | `SG-U00-Intune-TrustedCert-AndroidCorp` |
| `Trusted Certificate - UnreadLines Root CA - Android (BYOD)` | UnreadLines Root CA (U01PARVMPKI01) — Android Enterprise, personally-owned (BYOD) devices. | `SG-U00-Intune-TrustedCert-AndroidBYOD` |
| `Trusted Certificate - UnreadLines Issuing CA - Android (BYOD)` | UnreadLines Issuing CA (U01PARVMPKI02) — Android Enterprise, personally-owned (BYOD) devices. | `SG-U00-Intune-TrustedCert-AndroidBYOD` |
| `Trusted Certificate - UnreadLines Root CA - iOS/iPadOS` | UnreadLines Root CA (U01PARVMPKI01) — iOS/iPadOS devices. | `SG-U00-Intune-TrustedCert-iOS` |
| `Trusted Certificate - UnreadLines Issuing CA - iOS/iPadOS` | UnreadLines Issuing CA (U01PARVMPKI02) — iOS/iPadOS devices. | `SG-U00-Intune-TrustedCert-iOS` |

## 4. Prerequisites

- `ad-cs-pki-deployment/README.md` Checkpoint 2 (§18) reached: both CAs issuing, both certificates
  published and reachable at `http://pki.corp.unreadlines.com/`.
- At least one device already enrolled in Intune for each of the four platforms targeted here —
  enrollment itself is out of scope (§2).
- Microsoft Entra ID P1 or higher in the `unreadlines` tenant — dynamic group membership rules require
  it.
- Intune Administrator (or Global Administrator) role in the tenant.
- A way to convert a Base-64 (PEM) certificate to DER — `openssl`, or Windows' built-in `certutil`. Used
  once, in §5.

No lab VM is touched in this runbook. Every action happens in the Microsoft Entra admin center, the
Microsoft Intune admin center, or on your own administrative workstation for §5's conversion — each
section's opening line says which, so no separate machine marker is used.

## 5. Convert the Issuing CA certificate to DER

On your own administrative workstation — no lab VM is involved in this step. Download both certificates
from the distribution point: `http://pki.corp.unreadlines.com/UnreadLinesRootCA.crt`
and `http://pki.corp.unreadlines.com/UnreadLinesIssuingCA.crt`.

`UnreadLinesRootCA.crt` needs no conversion — `Export-Certificate -Type CERT`
(`ad-cs-pki-deployment/README.md` §10) always produces DER, and Intune's Trusted certificate profiles
require DER on every platform, Windows included. `UnreadLinesIssuingCA.crt` is Base-64 (PEM) despite the
`.crt` extension — `certreq -retrieve` (`ad-cs-pki-deployment/README.md` §14) returns Base-64 by
default — and Intune rejects it, or behaves unpredictably, if it is uploaded as-is.

Confirm which is which before trusting the extension: a DER file opens as binary; a Base-64 file starts
with `-----BEGIN CERTIFICATE-----` when opened as text.

Convert:

```bash
openssl x509 -in UnreadLinesIssuingCA.crt -inform PEM -out UnreadLinesIssuingCA-der.crt -outform DER
```

*— or, without `openssl`, on Windows:*

```text
certutil -decode UnreadLinesIssuingCA.crt UnreadLinesIssuingCA-der.crt
```

Confirm the result:

```bash
openssl x509 -in UnreadLinesIssuingCA-der.crt -inform DER -noout -subject -issuer
```

Expect `subject=...CN = UnreadLines Issuing CA` and `issuer=...CN = UnreadLines Root CA`. Keep both
`UnreadLinesRootCA.crt` and `UnreadLinesIssuingCA-der.crt` at hand — every profile in §7–§10 uploads one
of these two files, never the original Base-64 `UnreadLinesIssuingCA.crt`.

## 6. Create the four dynamic Entra ID groups

Names follow `reference/naming-conventions.md` §9.1, `SG-<Entity><Site>-<Workload>-<Purpose>-<Qualifier>`
— the first real execution of that cloud-only-object pattern, now the reference precedent for future
labs. `U00` (§1, Global / cross-site) is used here because these groups scope Intune-managed devices
tenant-wide, not by physical site.

In the Microsoft Entra admin center: `entra.microsoft.com` → **Groups** → **All groups** → **New group**.
Repeat the whole procedure below four times, once per row of the table further down.

**Basics, in the New Group panel:**

```text
Group type        : Security          (default — leave as is)
Group name        : <see table below>
Group description : <see table below>
Membership type   : Dynamic Device    (dropdown: Assigned / Dynamic User / Dynamic Device)
```

Leave **Microsoft Entra roles can be assigned to the group** off, and **Owners** empty — this group only
exists for its dynamic rule and its Intune assignment, nobody needs to own it individually.

Selecting **Dynamic Device** replaces **Members** with a required **Dynamic device members** field
showing a single link, **Add dynamic query**. Click it: it opens a full-page **Dynamic membership rules**
panel with two tabs, **Configure Rules** (default) and **Validate Rules**.

Build the rule with the table under **Configure Rules** — its columns are **And/Or**, **Property**,
**Operator**, **Value** — never with the **Rule syntax** box underneath. That box only ever shows the
syntax the builder produced; click its **Edit** link to type into it directly only if you like living
dangerously — it is easy to leave in a state that looks valid and isn't, where the builder can't produce
an invalid rule.

For the row's first condition: click the **Property** field (a searchable dropdown — confirmed present
in the live list, alongside `deviceCategory`, `deviceManufacturer`, `deviceModel` and others) and pick
`deviceOSType` or `deviceOwnership`; click **Operator** and pick `Equals` or `Starts With`; type the
**Value** by hand — it is a free-text box, not a picker, so it is up to you to type it exactly as below.
For a second condition on the same group, click **+ Add expression** below the table: a new row appears
with its own **And/Or** dropdown as its first cell — set it to **And** or **Or** per the table, then fill
Property/Operator/Value the same way as the first row.

| Group name | Group description | Rule, as entered in the builder |
| --- | --- | --- |
| `SG-U00-Intune-TrustedCert-Windows` | Windows devices enrolled in Intune — targeted by the two Windows Trusted certificate profiles (§7). | `(device.deviceOSType -eq "Windows")` |
| `SG-U00-Intune-TrustedCert-AndroidCorp` | Corporate-owned Android Enterprise devices — targeted by the Android (Corp) Trusted certificate profiles (§8). | `(device.deviceOSType -startsWith "Android") -and (device.deviceOwnership -eq "Company")` |
| `SG-U00-Intune-TrustedCert-AndroidBYOD` | Personally-owned (BYOD) Android Enterprise devices — targeted by the Android (BYOD) Trusted certificate profiles (§9). | `(device.deviceOSType -startsWith "Android") -and (device.deviceOwnership -eq "Personal")` |
| `SG-U00-Intune-TrustedCert-iOS` | iPhone and iPad devices — targeted by the iOS/iPadOS Trusted certificate profiles (§10). | `(device.deviceOSType -eq "iPad") -or (device.deviceOSType -eq "iPhone")` |

This is the same four rules as §3, entered field by field instead of as one string — the builder
assembles the identical `(device.deviceOSType -eq "Windows")`-style syntax underneath, visible read-only
in the **Rule syntax** box once the row is filled in.

Once every row for that group is filled in, click **Save** in the command bar at the top of the
**Dynamic membership rules** panel — this returns to the **New Group** panel with the rule attached.
Click **Create** there to create the group, then start over for the next one.

Once all four exist, reopen each one (**Groups** → search its name → **Dynamic membership rules**) and
confirm the table still reads exactly as above — this is the same check §11 asks for again at the end.

⚠️ Dynamic membership evaluation is not instant: a newly created group's **Members** tab can stay empty
for several minutes, sometimes longer, even against devices that are already enrolled and match the
rule. An empty **Members** tab right after creation is not by itself evidence the rule is wrong.

## 7. Deploy the Windows profiles

Profile names follow `reference/naming-conventions.md` §9.2 — Title Case with spaces, Intune's own
console convention, not the kebab-case or ALL-CAPS used for servers and accounts elsewhere in this repo.

In the Microsoft Intune admin center: `intune.microsoft.com` → **Devices** → **Configuration** →
**Create** → **New policy**. Every profile in this runbook goes through the same wizard shape — five
tabs, **Basics** → **Configuration settings** → **Scope tags** → **Assignments** → **Review + create** —
moved through with **Next** and finished with **Create** on the last tab. This section spells out every
tab once; §8–§10 only call out what changes.

```text
Platform      : Windows 10 and later
Profile type  : Templates → Trusted certificate
```

Click **Create** to open the wizard.

**Root CA profile — Basics tab:**

```text
Name          : Trusted Certificate - UnreadLines Root CA - Windows
Description   : UnreadLines Root CA (U01PARVMPKI01) — Computer certificate store, Root.
```

**Next** → **Configuration settings tab:**

```text
Certificate file    : UnreadLinesRootCA.crt   (the browser's native file picker — select it yourself, §5)
Destination store   : Computer certificate store - Root      (already the default — leave it)
```

**Next** → **Scope tags tab:** leave at the default (`Default`) — this lab doesn't use scope tags.
**Next** → **Assignments tab:** under **Included groups**, click **Add groups**, type
`SG-U00-Intune-TrustedCert-Windows` in the search box, select it, **Select**. **Next** → **Review + create
tab:** check the summary matches the above, then **Create**.

**Issuing CA profile** — same **Create** → **New policy**, same platform and profile type. **Basics:**

```text
Name          : Trusted Certificate - UnreadLines Issuing CA - Windows
Description   : UnreadLines Issuing CA (U01PARVMPKI02) — Computer certificate store, Intermediate.
```

**Configuration settings:**

```text
Certificate file    : UnreadLinesIssuingCA-der.crt   (from §5 — not the original Base-64 file)
Destination store   : Computer certificate store - Intermediate   ← change this; it defaults to Root
```

**Scope tags:** default. **Assignments:** **Add groups** → same `SG-U00-Intune-TrustedCert-Windows` group
as the Root CA profile. **Review + create** → **Create**.

The certificate file picker is the browser's native file dialog and cannot be driven from a
browser-automated session — select the file yourself when prompted.

After saving, each profile's own detail page reports "Platform: Windows 8.1 and later" even though
"Windows 10 and later" was selected at creation — that is how Intune labels this profile type
internally, not a sign anything was picked wrong.

## 8. Deploy the Android Enterprise Corporate profiles

Still in the Microsoft Intune admin center, same **Devices** → **Configuration** → **Create** →
**New policy** flow and the same five-tab wizard as §7:

```text
Platform      : Android Enterprise
Profile type  : Fully Managed, Dedicated, and Corporate-Owned Work Profile → Trusted certificate
```

Android Enterprise splits by enrollment/ownership at this profile-type selection step, not with a
checkbox further down inside the wizard — confirm the enrollment type here before clicking **Create**;
picking the wrong one means abandoning the profile and starting over, there is no way to switch it later.

**Root CA profile — Basics:**

```text
Name          : Trusted Certificate - UnreadLines Root CA - Android (Corp)
Description   : UnreadLines Root CA (U01PARVMPKI01) — Android Enterprise, corporate-owned devices.
```

**Configuration settings:**

```text
Certificate file  : UnreadLinesRootCA.crt
```

This platform has no **Destination store** field — the Configuration settings tab only asks for the
certificate file. **Scope tags:** default. **Assignments:** **Add groups** →
`SG-U00-Intune-TrustedCert-AndroidCorp`. **Review + create** → **Create**.

**Issuing CA profile** — same platform and profile type. **Basics:**

```text
Name          : Trusted Certificate - UnreadLines Issuing CA - Android (Corp)
Description   : UnreadLines Issuing CA (U01PARVMPKI02) — Android Enterprise, corporate-owned devices.
```

**Configuration settings:**

```text
Certificate file  : UnreadLinesIssuingCA-der.crt
```

**Scope tags:** default. **Assignments:** same `SG-U00-Intune-TrustedCert-AndroidCorp` group as the Root
CA profile. **Review + create** → **Create**.

## 9. Deploy the Android Enterprise Personal (BYOD) profiles

Same wizard again, in the same admin center, with the other Android Enterprise profile type:

```text
Platform      : Android Enterprise
Profile type  : Personally-Owned Work Profile → Trusted certificate
```

**Root CA profile — Basics:**

```text
Name          : Trusted Certificate - UnreadLines Root CA - Android (BYOD)
Description   : UnreadLines Root CA (U01PARVMPKI01) — Android Enterprise, personally-owned (BYOD) devices.
```

**Configuration settings:**

```text
Certificate file  : UnreadLinesRootCA.crt
```

No **Destination store** field on this platform either. **Scope tags:** default. **Assignments:**
**Add groups** → `SG-U00-Intune-TrustedCert-AndroidBYOD`. **Review + create** → **Create**.

**Issuing CA profile — Basics:**

```text
Name          : Trusted Certificate - UnreadLines Issuing CA - Android (BYOD)
Description   : UnreadLines Issuing CA (U01PARVMPKI02) — Android Enterprise, personally-owned (BYOD) devices.
```

**Configuration settings:**

```text
Certificate file  : UnreadLinesIssuingCA-der.crt
```

**Scope tags:** default. **Assignments:** same `SG-U00-Intune-TrustedCert-AndroidBYOD` group as the Root
CA profile. **Review + create** → **Create**.

## 10. Deploy the iOS/iPadOS profiles

Same admin center, same wizard:

```text
Platform      : iOS/iPadOS
Profile type  : Templates → Trusted certificate
```

**Root CA profile — Basics:**

```text
Name          : Trusted Certificate - UnreadLines Root CA - iOS/iPadOS
Description   : UnreadLines Root CA (U01PARVMPKI01) — iOS/iPadOS devices.
```

**Configuration settings:**

```text
Certificate file  : UnreadLinesRootCA.crt
```

**Scope tags:** default. **Assignments:** **Add groups** → `SG-U00-Intune-TrustedCert-iOS`. **Review +
create** → **Create**.

**Issuing CA profile — Basics:**

```text
Name          : Trusted Certificate - UnreadLines Issuing CA - iOS/iPadOS
Description   : UnreadLines Issuing CA (U01PARVMPKI02) — iOS/iPadOS devices.
```

**Configuration settings:**

```text
Certificate file  : UnreadLinesIssuingCA-der.crt
```

**Scope tags:** default. **Assignments:** same `SG-U00-Intune-TrustedCert-iOS` group as the Root CA
profile. **Review + create** → **Create**.

Like Android, this platform has no **Destination store** field — Apple's trust store doesn't distinguish
Root from Intermediate at profile creation the way Windows does.

## 11. Verify the deployment

**Devices** → **Configuration** lists all eight profiles. For each, open its assignment and device status
and confirm:

- The assigned group matches §3's profile table — one group assigned to exactly two profiles, that
  platform's Root and Issuing pair.
- Reopening both Windows profiles still shows **Intermediate** as the Issuing CA profile's destination
  store — the one setting most likely to have silently stayed at its Root default if §7 was rushed.

**Entra ID** → **Groups** — reopen each of the four dynamic groups and confirm **Membership rule** still
reads exactly as recorded in §6; an edit that appeared to save in the raw syntax box can fail to persist.

If a platform already has an enrolled device, its group's member count and its profiles' device status
populate immediately — check for **Succeeded** rather than **Pending** or **Error**. A platform with no
device enrolled yet legitimately shows a zero-member group and profiles with nothing to report: that is
not a failure, there is simply nothing yet to evaluate the rule against.

**What this does not prove:** that any specific device has actually installed the certificate — that
needs a device already enrolled and checked in at least once after assignment. This runbook's success
condition is narrower: eight correctly scoped profiles exist, each pointing at the right file, in the
right store, assigned to the right dynamic group.

## 12. Next step — Intune SCEP profile

These eight Trusted certificate profiles only make devices trust the UnreadLines CA hierarchy — none of
them hand a device a certificate it can prove its own identity with. That is item 8 of the chain
described in `nps-server-certificate-deployment/README.md` §3: an Intune SCEP profile, backed by NDES
and the Intune Certificate Connector (items 4–5 of that same chain, also not yet built), issuing a client
authentication certificate to each device permitted onto `UnreadLines-Mobile`. Nothing further down that
chain can be deployed correctly until this runbook's trust relationship exists first.

## 13. References

- [Configure and deploy trusted certificate profiles — Microsoft Learn](https://learn.microsoft.com/en-us/mem/intune/protect/certificates-configure)
- [Windows Trusted certificate profile settings — Microsoft Learn](https://learn.microsoft.com/en-us/mem/intune/protect/certificates-trusted-root)
- [Choose an Android Enterprise enrollment scenario — Microsoft Learn](https://learn.microsoft.com/en-us/mem/intune/enrollment/android-enterprise-scenarios-choose)
- [Dynamic membership rules for groups — Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
- [Export-Certificate — Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/pki/export-certificate)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
