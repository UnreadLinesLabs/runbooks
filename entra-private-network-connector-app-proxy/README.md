# Publish NDES through the Entra Private Network Connector and Application Proxy

This lab exposes `U01PARVMNDS01`'s SCEP endpoint (`ndes-scep-intune-connector`) to a real mobile device
outside the lab network, without opening an inbound port or handing the device a domain credential. It
builds a dedicated server, `U00PARVMPNC01`, running the Microsoft Entra Private Network Connector — an
outbound-only tunnel to Entra ID — and publishes NDES behind it through Entra Application Proxy.

A **separate, dedicated server** for the connector, rather than installing it on `U01PARVMNDS01` itself,
keeps the two roles independent: the connector is company-wide infrastructure that could one day carry
other published applications, while NDES stays scoped to SCEP. `U00` (not `U01`) reflects that — see
`reference/naming-conventions.md` §1. The connector does not join the Active Directory domain: it
authenticates to Entra ID directly and has no on-premises dependency of its own.

Application Proxy's preauthentication is fixed at **Passthrough** — the only mode SCEP's own protocol
allows, not a choice made for this lab. Because Application Proxy publishes one internal URL to one
external URL for the whole application, with no sub-path restriction of its own, this lab also adds an
IIS-side rule on `U01PARVMNDS01` that narrows the externally reachable surface down to the one path a
SCEP client actually needs.

`ndes-scep-intune-connector` left IIS on `U01PARVMNDS01` on plain HTTP — reasonable at the time, since
nothing outside the lab network could reach it yet. Publishing it through Application Proxy changes that:
this lab also issues `U01PARVMNDS01` its own Server Authentication certificate and binds it to IIS, the
same pattern `nps-server-certificate-deployment` already used for `U01PARVMNPS01`, so the connection stays
HTTPS end to end, including on this last internal hop.

## 1. Architecture

```text
A real phone, outside the lab network
      |
      v
Entra Application Proxy — external URL (*.msappproxy.net), Passthrough preauth
      |
      v
U00PARVMPNC01   Entra Private Network Connector           192.168.20.148
      |
      |  outbound HTTPS 443 to Entra ID only
      |  (no inbound port opened on U00PARVMPNC01 or U01PARVMNDS01)
      v
U01PARVMNDS01   NDES (IIS) + Intune Certificate Connector  192.168.20.147
      |  IIS rule (§12) narrows the externally reachable paths to
      |  /certsrv/mscep/mscep.dll (+ /pkiclient.exe variant) and
      |  /CertificateRegistrationSvc/ — everything else on this
      |  site, including /certsrv/mscep_admin, answers 403
      v
U01PARVMPKI02   Issuing CA — signs, per ndes-scep-intune-connector

Not built yet — see §16 "Next step":
      the Intune SCEP certificate profile that actually points a
      mobile device at this lab's new external URL.
```

## 2. Where this lab fits

The certificate and RADIUS chain toward a working `UnreadLines-Mobile` (`YouTube/backlog.md` §2):

```text
1.  Root CA + Issuing CA (PKI)                              — ad-cs-pki-deployment            done
2.  NPS / RADIUS server                                     — radius-nps-deployment           done
3.  NPS server certificate (Server Authentication)          — nps-server-certificate-deployment done
4.  SCEP template + NDES + Intune Certificate Connector     — ndes-scep-intune-connector      done
5.  Entra Private Network Connector + Application Proxy     — this lab                        ← now
6.  Intune trusted-certificate profiles (Root/Issuing)      — intune-trusted-certificate-profiles done
7.  Intune SCEP profile (mobile client certificate)         — video TBD
8.  Intune Wi-Fi EAP-TLS profile                            — video TBD
9.  U01PARVMRAP01 switched to UnreadLines-Mobile            — video TBD
10. Real EAP-TLS handshake, confirmed in NPS logs           — video TBD
```

Item 4 is a hard prerequisite: without a working NDES endpoint there is nothing to publish. Nothing
downstream of this lab (items 7–10) can be tested end to end until it is done, because no mobile device
outside the lab network has a path to the SCEP endpoint before this lab exists.

## 3. Scope and dependencies

This lab covers: building `U00PARVMPNC01`, installing and registering the Entra Private Network
Connector, publishing `U01PARVMNDS01` through Entra Application Proxy with Passthrough preauthentication,
and narrowing the externally reachable path to the SCEP endpoint with an IIS rule.

It does **not** cover:

- NDES itself, the `Intune SCEP Mobile User` template, or the Intune Certificate Connector — already
  built in `ndes-scep-intune-connector/README.md`. This lab only exposes what that one built.
- The Intune SCEP certificate profile that actually targets a group of mobile devices with this lab's new
  external URL — a later lab (item 7, §2 above).
- Microsoft Entra Private Access / Global Secure Access. It shares the same connector family — Microsoft
  Learn: "Application proxy uses the same connector as Microsoft Entra Private Access, the Microsoft
  Entra private network connector" — but it is a different product built for authenticated, per-user
  Zero Trust access, and it does not offer SCEP's required Passthrough (no preauthentication) mode. This
  lab uses classic Application Proxy application publishing, not Private Access.
- A real certificate request from a phone. This lab's success condition is narrower: the endpoint is
  reachable from outside the lab network, scoped to exactly the path SCEP needs, and the connector
  reports healthy — not that a device has enrolled.

## 4. Target configuration

| Item | Value |
| --- | --- |
| Connector server | `U00PARVMPNC01`, `192.168.20.148/25`, Subnet 2 |
| OS | Windows Server 2025 |
| Domain-joined | No — standalone; the connector authenticates directly to Entra ID |
| Connector software | Microsoft Entra Private Network Connector (the current, unified connector — Microsoft's own naming history: this was formerly the separate "Application Proxy connector") |
| Published server | `U01PARVMNDS01`, `192.168.20.147/25` (built in `ndes-scep-intune-connector`) |
| Application Proxy application (display name) | `App Proxy - NDES SCEP - Mobile` — proposed here, following the `<Category> - <Subject> - <Qualifier>` Title Case pattern `naming-conventions.md` §9.2 already uses for Intune profiles; not yet formalized as its own subsection of §9, since this is the first Application Proxy app this project has published |
| Internal URL | `https://u01parvmnds01.corp.unreadlines.com/` — the NDES root, per Microsoft's own published procedure (§11) |
| External URL | Tenant default (`*.msappproxy.net`) — no custom domain configured for this project |
| Pre-authentication | Passthrough — the only mode SCEP's protocol allows; mandatory, not a choice made for this lab |
| Externally reachable paths (after §12's IIS rule) | `/certsrv/mscep/mscep.dll` (and its `/pkiclient.exe` variant) plus `/CertificateRegistrationSvc/` — the latter kept open because the Intune Certificate Connector calls it on the same site; `/certsrv/mscep_admin` stays blocked |
| NDES IIS certificate — duplicated from | built-in `Web Server` template |
| NDES IIS certificate — template display name | `NDES Server Authentication` |
| NDES IIS certificate — template name (internal) | `NDESServerAuthentication` |
| NDES IIS certificate — EKU | Server Authentication only (`Web Server`'s own default — nothing to change) |
| NDES IIS certificate — Subject | `CN=u01parvmnds01.corp.unreadlines.com`, SAN DNS name = same |

## 5. Prerequisites

- `ndes-scep-intune-connector/README.md` reached its checkpoint: `U01PARVMNDS01` running, issuing from
  `Intune SCEP Mobile User`, Certificate Connector confirmed **Active** — `reference/vm-inventory.md`
  §2 already reflects this.
- `U00PARVMPNC01` built per `prepare-windows-ubuntu-templates/README.md`, renamed per
  `reference/naming-conventions.md` §2 and §8 — Windows Server 2025, workgroup (not domain-joined),
  static IP `192.168.20.148/25`, gateway `192.168.20.254`, nothing else installed on it yet.
- An account with the **Application Administrator** role (or higher) on the `unreadlines` tenant, to
  register the connector and publish the application — a narrower role than the Intune/Global
  Administrator used for the Certificate Connector in `ndes-scep-intune-connector`.
- **The `unreadlines` tenant licensed for Microsoft Entra ID P1 or higher** — Application Proxy requires
  it. Not yet confirmed for this tenant; verify under the Entra admin center's licensing blade before
  starting, rather than assuming it is already in place.
- Outbound HTTPS (443) from `U00PARVMPNC01` to Entra ID. No inbound port is opened on either
  `U00PARVMPNC01` or `U01PARVMNDS01` by this lab.
- Administrative access on `U00PARVMPNC01` (§9), `U01PARVMPKI02` (§7), and `U01PARVMNDS01` (§8, §12).
- IIS's **URL Rewrite** module available on `U01PARVMNDS01` for §12 — not installed by
  `ndes-scep-intune-connector`, since NDES itself doesn't need it.
- `UnreadLinesRootCA.crt` and `UnreadLinesIssuingCA.crt` already copied onto `U00PARVMPNC01` at
  `C:\UnreadLines\` — the project's standard Level 2 staging location for exactly this kind of manual
  transfer (`ad-cs-pki-deployment/README.md` §2). Needed in §6 to install the `UnreadLines Root CA` and
  `UnreadLines Issuing CA` trust that a domain-joined machine would otherwise get automatically, since
  this one isn't.

## 6. Build `U00PARVMPNC01`

**On `U00PARVMPNC01`:**

Clone the Windows Server 2025 template per `prepare-windows-ubuntu-templates/README.md`, then rename and
address it:

```powershell
Rename-Computer -NewName "U00PARVMPNC01" -Restart
```

After the restart, confirm the name and check the actual adapter name before addressing it — it isn't
always `Ethernet0`, depending on how the template's NIC was named:

```powershell
$env:COMPUTERNAME
Get-NetAdapter
```

Set a static IP on Subnet 2, substituting the real adapter name from `Get-NetAdapter` above for
`<AdapterName>`:

```powershell
New-NetIPAddress -InterfaceAlias "<AdapterName>" -IPAddress 192.168.20.148 -PrefixLength 25 -DefaultGateway 192.168.20.254
Set-DnsClientServerAddress -InterfaceAlias "<AdapterName>" -ServerAddresses 192.168.20.41
```

Leave the machine in its default workgroup — it is not domain-joined (§4).

**Install the corporate CA trust by hand.** A domain-joined machine gets `UnreadLines Root CA` and
`UnreadLines Issuing CA` in its trust stores automatically, via Group Policy — `U00PARVMPNC01` never
will, since it deliberately isn't domain-joined (§4). Without this, the connector has no way to trust
`U01PARVMNDS01`'s certificate (issued by `UnreadLines Issuing CA`, §7–§8) when it forwards published
requests to it, and the app would fail with a backend TLS trust error despite everything else in this lab
being configured correctly. Both certificates are already staged at `C:\UnreadLines\` on this machine
(§5) — the project's standard Level 2 staging location for a manual transfer like this one
(`ad-cs-pki-deployment/README.md` §2) — so install them straight from there, no download needed. No
format conversion either, unlike the DER requirement Intune's profiles have
(`intune-trusted-certificate-profiles/README.md` §5):

```powershell
Import-Certificate -FilePath "C:\UnreadLines\UnreadLinesRootCA.crt" -CertStoreLocation Cert:\LocalMachine\Root
Import-Certificate -FilePath "C:\UnreadLines\UnreadLinesIssuingCA.crt" -CertStoreLocation Cert:\LocalMachine\CA
```

Confirm both landed in the right store before moving on:

```powershell
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -like "*UnreadLines Root CA*"
Get-ChildItem Cert:\LocalMachine\CA   | Where-Object Subject -like "*UnreadLines Issuing CA*"
```

## 7. Publish the NDES server authentication template — on `U01PARVMPKI02`

`U01PARVMNDS01`'s IIS site has run on plain HTTP since `ndes-scep-intune-connector` — fine while nothing
outside the lab network could reach it. Publishing it through Application Proxy changes that: this
template gives it its own certificate so the internal hop stays HTTPS too.

1. `certtmpl.msc` → right-click the built-in `Web Server` template → **Duplicate Template**.

2. **Compatibility tab — leave the wizard's default.** `Web Server`'s own built-in Compatibility level
   (`Windows Server 2003` / `Windows XP/Server 2003`) is what the duplication wizard starts from, the
   same floor `ndes-scep-intune-connector` (§6 there) had to force its SCEP template back down to. Here
   there's nothing to force — it's already where it needs to be, and it lands on Legacy CSP either way
   (confirmed on the Cryptography tab, step 7 below).

3. **General tab:**

   ```text
   Template display name : NDES Server Authentication
   Template name          : NDESServerAuthentication
   ```

   Leave **Publish certificate in Active Directory** unchecked, same reasoning as `Web Server` itself:
   an IIS server certificate has no AD object of its own to publish against.

4. **Subject Name tab — nothing to change.** `Web Server` already defaults to *Supply in the request*,
   which is what §8's IIS wizard needs: it lets that wizard set the Common Name to
   `u01parvmnds01.corp.unreadlines.com` itself rather than pulling it from an AD computer object field.

5. **Extensions tab — nothing to change.** `Web Server`'s built-in Application Policy is already `Server
   Authentication` only — the correct EKU for a certificate that just has to prove `U01PARVMNDS01`'s own
   identity to a browser or to Application Proxy. This is the opposite case from the SCEP template, which
   had to swap this same built-in EKU out for Client Authentication.

6. **Request Handling tab — nothing to change.** `Web Server`'s default (`Purpose: Signature and
   encryption`, private key not exportable) is exactly what a normal HTTPS binding needs — it supports
   both the RSA key-exchange cipher suites (encryption) and the signature-based ones (ECDHE), unlike the
   SCEP template, which narrowed this to `Signature` only because a client-auth certificate never
   decrypts anything. Leave **Allow private key to be exported** unchecked.

7. **Cryptography tab — confirm, don't just glance at it:**

   ```text
   Provider Category   : Legacy Cryptographic Service Provider
   Providers           : Microsoft RSA SChannel Cryptographic Provider ✓ (checked)
   Minimum key size     : 2048
   ```

   Legacy CSP is correct and expected here — `Microsoft RSA SChannel Cryptographic Provider` is the
   standard provider IIS/Schannel itself uses for a TLS server certificate, not a compromise. If this
   tab instead shows **Key Storage Provider**, the Compatibility tab (step 2) was raised above `Web
   Server`'s own default — go back and confirm it's still unchanged.

8. **Security tab — remove the broad default, grant only the account that needs it:**

   ```text
   Authenticated Users
       Enroll     : Not granted   ← remove, do not leave inherited from Web Server

   U01PARVMNDS01$ (the NDES server's own computer account)
       Read       : Allow
       Enroll     : Allow
   ```

   This certificate identifies exactly one server; no other computer account needs to request it.

9. Publish it: `certsrv.msc` → `UnreadLines Issuing CA` → **Certificate Templates** → **New** →
   **Certificate Template to Issue** → select `NDES Server Authentication`.

10. **If enrollment fails with `The requested certificate template is not supported by this CA`
    (`0x80094800`, `CERTSRV_E_UNSUPPORTED_CERT_TYPE`), check which enrollment method produced it before
    assuming anything is wrong on the CA:**

    - **From IIS Manager's Server Certificates → Create Domain Certificate wizard: this error is
      expected, not a bug to chase.** That wizard does not let you pick a template at all — it always
      requests the built-in `Web Server` template internally, regardless of what CA path or friendly
      name you type in. `ad-cs-pki-deployment/README.md` never published `Web Server` as an issuable
      template on `UnreadLines Issuing CA` (only a temporary `PKI Validation` template, since deleted),
      so any request from this wizard is rejected outright — the same result whichever order the server
      name and CA name are typed in. This is exactly why §8 uses `certlm.msc` instead: it's the only one
      of the two that lets you explicitly select `NDES Server Authentication` rather than the hardcoded
      `Web Server`.
    - **From `certlm.msc` with `NDES Server Authentication` explicitly selected (§8):** this points at a
      real gap on the CA. Confirm on `U01PARVMPKI02`:

      ```powershell
      certutil -CATemplates
      ```

      Look for `NDESServerAuthentication` (the internal name, §7 step 3) in the list. If it's
      **missing**, step 9 above didn't actually take — repeat it. If it's **present** but enrollment
      still fails, the CA service has not refreshed its cached template list yet (it polls Active
      Directory periodically, not instantly, after a template is newly published); force the refresh:

      ```powershell
      Restart-Service CertSvc
      ```

      Then retry the enrollment on `U01PARVMNDS01` (§8).

## 8. Enroll the certificate and bind it to IIS — on `U01PARVMNDS01`

**Do not use IIS Manager's Server Certificates → Create Domain Certificate wizard for this step — it
cannot request a custom template at all.** That wizard has no template picker; it always submits its
request for the built-in `Web Server` template, no matter what CA path or friendly name is typed into it.
`ad-cs-pki-deployment/README.md` never published `Web Server` as issuable on `UnreadLines Issuing CA`, so
a request from this wizard is rejected with `The requested certificate template is not supported by this
CA` (`0x80094800`) regardless — see §7 step 10. Use the Certificates MMC snap-in instead, which lets you
pick `NDES Server Authentication` explicitly and discovers the CA through Active Directory rather than a
typed hostname:

1. `certlm.msc` opens it directly — or, if that command isn't available, `mmc.exe` → **File → Add/Remove
   Snap-in…** → **Certificates** → **Computer account** → **Local computer** → **Finish**. Either way,
   confirm the console is scoped to **Computer account**, not **My user account** — a certificate
   requested under the wrong scope won't match the `U01PARVMNDS01$` Read/Enroll grant from §7 and won't
   show the template at all.
2. **Personal** → right-click → **All Tasks → Request New Certificate...**
3. **Next** → **Next** (Active Directory Enrollment Policy).
4. Check **NDES Server Authentication** in the list — it appears automatically because `U01PARVMNDS01$`
   has `Read` + `Enroll` on it (§7). A note reading *"More information is required to enroll for this
   certificate"* appears under it — this is expected, since the template's Subject Name is *Supply in the
   request* (§7):
   - Click the note → **Subject** tab:
     ```text
     Subject name
     Type  : Common name          Value : u01parvmnds01.corp.unreadlines.com
     Type  : Country               Value : FR
     Type  : State                 Value : Île-de-France
     Type  : City                  Value : Paris
     Type  : Organization          Value : UnreadLines
     Type  : Organizational unit   Value : UnreadLines Labs

     Alternative name
     Type  : DNS
     Value : u01parvmnds01.corp.unreadlines.com
     ```
     Add each Subject name row one at a time (**Add** after each), then the Alternative name row, then
     **OK**. The SAN entry matters as much as the Common Name — most clients (and Chrome specifically)
     validate the SAN, not the CN, so a certificate missing it will still bind in IIS but fail TLS
     validation for anything checking the hostname properly. Country/State/City/Organization/
     Organizational unit are cosmetic here (nothing in this lab validates them), but filling them in
     keeps the Subject line consistent and readable when this certificate turns up next to others in
     the store — `Organization` is the fictional company (`UnreadLines`) itself; `Organizational unit`
     is `UnreadLines Labs`, the project producing these runbooks, kept distinct from the company the same
     way `reference/naming-conventions.md` §3 keeps that name out of the real AD `OU=` tree.
5. **Enroll** → **Finish**.

**If `NDES Server Authentication` does not appear in step 4's list at all**, check, in this order:

- It is actually published on `UnreadLines Issuing CA` — §7 step 9 (`certsrv.msc` → Certificate
  Templates → New → Certificate Template to Issue), confirmed with `certutil -CATemplates` on
  `U01PARVMPKI02` (§7 step 10).
- `U01PARVMNDS01$` has both **Read** and **Enroll** on the template's Security tab (§7 step 8) — not just
  one of the two, and not a group that doesn't actually include the computer account.
- The console is scoped to **Computer account**, not **My user account** (step 1 above) — the single most
  common reason the template silently doesn't show up even when everything above is correct.

The certificate lands in `Cert:\LocalMachine\My` — this enrollment path doesn't reliably set a friendly
name, so give it one: find it (Subject `u01parvmnds01.corp.unreadlines.com`, issued by `UnreadLines
Issuing CA`), right-click → **Properties** → **General** tab → **Friendly name**: `NDES Server
Authentication`. This is what makes it easy to pick out in the IIS binding dropdown below, rather than
having to recognize it by expiry date among any other certificates on this store.

Create the HTTPS binding, then attach the certificate through IIS Manager (site → **Bindings…** → **Add**
or **Edit** the `https`/`443` binding) rather than PowerShell's `AddSslCertificate` — that method
frequently fails on this exact step with `A specified logon session does not exist (0x80070520)`, a known
COM/CNG quirk unrelated to anything specific to this lab:

Check first, and only create the binding if it isn't already there — IIS bindings are unique per
protocol/IP/port across every site on the server, so running `New-WebBinding` against one that already
exists (most likely from an earlier attempt at this same step) throws `Cannot add duplicate collection
entry of type 'binding' ... 'https, *:443:'` instead of just no-opping:

```powershell
if (-not (Get-WebBinding -Name "Default Web Site" -Protocol https)) {
    New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -IPAddress "*"
}
```

Either way — whether this just created the binding or it already existed — move on to attaching the
certificate.

**In IIS Manager:** `Default Web Site` → **Bindings...** → select the `https`/`443` binding →
**Edit...** → **SSL certificate**: choose `NDES Server Authentication` (the friendly name from above) →
**OK**. This binds the certificate through the same in-process path IIS Manager always uses, sidestepping
whatever breaks the PowerShell method.

Confirm the SCEP endpoint now answers over HTTPS internally, from `U01PARVMNDS01` itself. **A bare request
to `mscep.dll` with no SCEP operation is expected to return `403 Forbidden` — that's NDES's normal response
to a request that isn't an actual SCEP operation, not a fault** — Microsoft's own troubleshooting guide
states this `403` explicitly means the SCEP URL is functioning correctly when called without SCEP
parameters (§17). It still proves the DNS name resolves, the
certificate is accepted, the HTTPS binding works, and IIS reaches the `mscep.dll` extension — everything
this step is meant to confirm. To get a real `200`, ask for an actual SCEP operation instead:

```powershell
$uri = "https://u01parvmnds01.corp.unreadlines.com/certsrv/mscep/mscep.dll?operation=GetCACaps&message=ca"
(Invoke-WebRequest -Uri $uri -UseBasicParsing).StatusCode
```

Expect `200`.

## 9. Install and register the Private Network Connector — on `U00PARVMPNC01`

1. In the Entra admin center, go to **Enterprise applications → Private Network Connectors** and select
   **Download connector service** — this is the same installer Microsoft Learn documents for both
   Application Proxy and Private Access (§3); the blade is named after the unified connector, not after
   Application Proxy specifically, which is why it doesn't sit under an "Application Proxy" label.
2. Copy the installer to `U00PARVMPNC01` and run it.
3. When prompted, sign in with the Application Administrator account from §5. **Internet Explorer
   Enhanced Security Configuration can block this sign-in screen** — Microsoft's own documentation
   flags this; disable it temporarily on `U00PARVMPNC01` if the screen doesn't render.
4. If `U00PARVMPNC01` sits behind an outbound proxy, run the connector's
   `ConfigureOutBoundProxy.ps1` script afterward to point it at that proxy — not needed on this lab's
   network, which reaches the internet directly through `FWL02`.
5. Let the installer finish; it registers the connector against the `unreadlines` tenant automatically.

## 10. Verify the connector is Active — in the Entra admin center

In **Enterprise applications → Private Network Connectors** (or **Global Secure Access → Connectors →
Private network connectors** — the same object surfaces in both blades, §3), confirm `U00PARVMPNC01`
shows **Active**. Do not continue to §11 until it does — a connector that isn't active yet cannot proxy
anything published against it.

## 11. Publish NDES through Application Proxy — in the Entra admin center

1. **Enterprise applications → New application → Add an on-premises application** (there is no separate
   "On-premises application" tile to click first — this button on the New application page opens the
   form directly). If that button isn't there, use the alternate path instead: **Create your own
   application** → **Configure Application Proxy for secure remote access to an on-premises application**.
2. **Name**: `App Proxy - NDES SCEP - Mobile` (§4).
3. **Internal URL**: the root URL of `U01PARVMNDS01`, e.g. `https://u01parvmnds01.corp.unreadlines.com/`
   — the NDES root, not a sub-path. Microsoft's own procedure for this exact scenario publishes the
   root; this screen has no option to publish only a sub-path (the limitation the intro above already
   notes — §12 is where it's actually addressed).
4. **External URL**: leave the tenant default suffix as offered (`*.msappproxy.net` — shown as two parts,
   a prefix box and a fixed suffix dropdown; accept whatever prefix the form proposes) — no custom domain
   configured for this project. **Application segments**: leave empty — that's for publishing several
   internal apps behind one wildcard app, not used here.
5. **Pre Authentication**: change the dropdown from its default (**Microsoft Entra ID**) to
   **Passthrough** — it does not default to this, so it has to be picked explicitly. Passthrough is the
   only mode that actually works for this app: SCEP cannot complete an interactive sign-in step, so
   leaving the default in place would make the published endpoint unreachable by any real SCEP client.
6. **Connector group**: the default group `U00PARVMPNC01` registered into — the tenant may label it with
   a region suffix (e.g. `Default - Europe`) rather than plain `Default`; either way, leave whatever the
   dropdown already shows selected, since `U00PARVMPNC01` is currently the only connector in it.
   **SSL Certificate**: shown as "No SSL certificate required" — informational only (the tenant default
   `*.msappproxy.net` domain manages its own certificate), nothing to configure here.
7. Save, then copy the generated external URL — needed for §13 and for the Intune SCEP profile in the
   next lab (§16).

## 12. Harden the published endpoint — on `U01PARVMNDS01`

Application Proxy always maps the whole internal URL to the whole external URL (§11) — restricting the
externally reachable surface to the one path SCEP needs has to happen on the NDES server itself, in IIS.

1. **Install the URL Rewrite module** — it is not part of the IIS role and `ndes-scep-intune-connector`
   never installed it (§5). Download and install it silently:

   ```powershell
   Invoke-WebRequest -Uri "https://download.microsoft.com/download/1/2/8/128E2E22-C1B9-44A4-BE2A-5859ED1D4592/rewrite_amd64_en-US.msi" `
       -OutFile "$env:TEMP\rewrite_amd64_en-US.msi"

   Start-Process msiexec.exe -ArgumentList "/i `"$env:TEMP\rewrite_amd64_en-US.msi`" /quiet /norestart" -Wait
   ```

   If that URL has moved, get the current one from
   [iis.net/downloads/microsoft/url-rewrite](https://www.iis.net/downloads/microsoft/url-rewrite) — the
   filename (`rewrite_amd64_en-US.msi`) should stay the same across revisions.

   Confirm it installed and IIS picked it up (requires the `WebAdministration` module, already present
   with the IIS role):

   ```powershell
   Import-Module WebAdministration
   Get-WebGlobalModule -Name "RewriteModule"
   ```

   Expect one row back, not an empty result — an empty result means the MSI installed but IIS wasn't
   restarted to pick it up; run `iisreset` and check again.

2. In IIS Manager, open the site hosting NDES, then **URL Rewrite → Add Rule(s) → Blocking Rule**.
3. Block every request whose path does **not** match one of the two paths this server actually needs to
   keep answering:
   - Pattern: `^(certsrv/mscep/mscep\.dll(/pkiclient\.exe)?|CertificateRegistrationSvc/.*)$`
   - Requested URL: **Does Not Match the Pattern**
   - Action: **Abort Request** (or **Custom Response**, status `403`)

   Equivalent rule in `web.config`:

   ```xml
   <rule name="Restrict external access to SCEP endpoint only" stopProcessing="true">
     <match url="^(certsrv/mscep/mscep\.dll(/pkiclient\.exe)?|CertificateRegistrationSvc/.*)$" negate="true" />
     <action type="AbortRequest" />
   </rule>
   ```

   Three corrections against the first draft of this pattern, worth noting since they came from a real
   review rather than the first live test:
   - **Anchored at both ends** (`^...$`) — without the trailing `$`, anything starting with
     `certsrv/mscep/mscep.dll` would match, `mscep.dll-malicious` included, defeating the rule entirely.
   - **`/pkiclient.exe` variant included** — some SCEP clients (routers, VPN appliances following the
     original Cisco/Verisign SCEP CGI convention) request `.../mscep/mscep.dll/pkiclient.exe` rather than
     `mscep.dll` alone; NDES answers both, so the rule has to allow both too.
   - **`CertificateRegistrationSvc/` added to the allow list.** This is a second, separate IIS-hosted
     endpoint the Intune Certificate Connector calls to validate requests — and it lives on the *same*
     site as `mscep.dll`. A rule scoped to `mscep.dll` alone doesn't just narrow what's reachable from the
     internet: it blocks **every** request IIS receives on this site regardless of where it came from,
     including the connector's own local calls to `CertificateRegistrationSvc` — so the original, narrower
     pattern risked breaking certificate issuance entirely, not just tightening external exposure.
     `/certsrv/mscep_admin` — the NDES admin page that can reveal an enrollment challenge password — is
     deliberately **not** added to this list: this project's Intune Certificate Connector uses dynamic,
     per-request challenges (§`ndes-scep-intune-connector`), so `mscep_admin` isn't part of the working
     flow, and it's exactly the kind of path this hardening step exists to keep off the published URL in
     the first place.

4. **This rule applies to every request IIS receives on this site**, including from inside the lab
   network — confirm nothing else on `U01PARVMNDS01` depends on another path on the same site/binding
   before enabling it. `ndes-scep-intune-connector/README.md` builds this server for SCEP and the
   Certificate Connector alone, so nothing else should be sharing the site — but verify on the live server
   rather than assuming it, and re-test actual certificate issuance (not just the URL checks in §13) after
   enabling this rule, since `CertificateRegistrationSvc` traffic won't show up in an outside-in test.

## 13. Test the published endpoint end to end

**From a machine outside the lab network** (a real phone, or a laptop on a different network — testing
from inside the lab network proves nothing about the path this lab actually builds):

1. Browse to the external URL from §11's root (no path). Expect **403 Forbidden** — proof that §12's rule
   is in effect, not just that the app is reachable.
2. Append `/certsrv/mscep/mscep.dll` with no query string and browse to that. Expect **403 Forbidden** here
   too — this is NDES's own normal response to a request that isn't an actual SCEP operation (§8), so on
   its own it does **not** distinguish "reachable, working endpoint" from "still blocked by §12's rule".
   To actually prove the path through, request a real SCEP operation instead and expect **200**:

   ```powershell
   $uri = "<external URL from §11>/certsrv/mscep/mscep.dll?operation=GetCACaps&message=ca"
   (Invoke-WebRequest -Uri $uri -UseBasicParsing).StatusCode
   ```
3. Back in the Entra admin center, re-check §10 — the connector should still show **Active** after
   handling a real external request.

## 14. Expected final state

- `U00PARVMPNC01` runs the Entra Private Network Connector, shown **Active** in the Entra admin center,
  and trusts `UnreadLines Root CA`/`UnreadLines Issuing CA` even though it isn't domain-joined (§6).
- `U01PARVMNDS01` serves its SCEP endpoint over HTTPS internally, under its own `NDES Server
  Authentication` certificate (§7–§8).
- `App Proxy - NDES SCEP - Mobile` is published, Passthrough preauthentication, internal URL pointed at
  `U01PARVMNDS01`'s root.
- From outside the lab network, the published external URL returns **403** at its root, **403** on a bare
  `/certsrv/mscep/mscep.dll` request (expected NDES behavior, §8), and **200** on a real SCEP operation
  (`?operation=GetCACaps&message=ca`); `/certsrv/mscep_admin` also returns **403**, and
  `CertificateRegistrationSvc` stays reachable for the Certificate Connector's own use.
- No inbound port is open on `U00PARVMPNC01` or `U01PARVMNDS01` — the entire path from the internet to
  `U01PARVMNDS01` is the connector's outbound tunnel.

**What this does not yet prove:** no mobile device has actually requested a certificate through this
path. No Intune SCEP profile targets a device at this new external URL yet — that's the next lab (§16).

## 15. Update the infrastructure inventory

Per `reference/naming-conventions.md` §2, update `reference/vm-inventory.md`'s `U00PARVMPNC01` row:
change `Status` from `Planned` to `Running`, and record the published external URL and the display name
from §11 in its notes.

## 16. Next step — Intune SCEP profile

`U01PARVMNDS01` now has a real, internet-reachable — but narrowly scoped — path. The next lab creates the
Intune SCEP certificate profile (Certificate type `User`, SAN = UPN, Client Authentication only) and
points its **SCEP Server URLs** field at this lab's external URL. Only after that lab does an actual
Intune-managed phone have both a policy telling it to enrol and a network path to reach the endpoint
built here.

## 17. References

- [Use Microsoft Entra application proxy with a Network Device Enrollment Service (NDES) server — Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/app-proxy/app-proxy-protect-ndes)
- [Understand Microsoft Entra application proxy connectors — Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/app-proxy/application-proxy-conceptual-connectors)
- [Microsoft Entra private network connectors — Microsoft Learn](https://learn.microsoft.com/en-us/entra/global-secure-access/concept-connectors)
- [Add an on-premises application for remote access through Application Proxy — Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/app-proxy/application-proxy-add-on-premises-application)
- [Create Blocking Rules for URL Rewrite Module — Microsoft Learn / IIS.net](https://learn.microsoft.com/en-us/iis/extensions/url-rewrite-module/creating-blocking-rules-for-url-rewrite-module)
- [URL Rewrite Module 2.1 — download page, IIS.net](https://www.iis.net/downloads/microsoft/url-rewrite)
- [Troubleshoot managed device to NDES communication in Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/mem/intune/certificates/troubleshoot-scep-certificate-device-to-ndes) — source for §8/§13's `403` on a bare `mscep.dll` request being expected behavior.

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
