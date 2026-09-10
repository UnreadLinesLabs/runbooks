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
      |  IIS rule (§12) narrows the externally reachable path to
      |  /certsrv/mscep/mscep.dll — everything else on this site
      |  answers 403 when reached through the published URL
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
| Externally reachable path (after §12's IIS rule) | `/certsrv/mscep/mscep.dll` only |
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

## 6. Build `U00PARVMPNC01`

**On `U00PARVMPNC01`:**

Clone the Windows Server 2025 template per `prepare-windows-ubuntu-templates/README.md`, then rename and
address it:

```powershell
Rename-Computer -NewName "U00PARVMPNC01" -Restart
```

After the restart, confirm the name and set a static IP on Subnet 2:

```powershell
$env:COMPUTERNAME

New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.20.148 -PrefixLength 25 -DefaultGateway 192.168.20.254
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.20.41
```

Leave the machine in its default workgroup — it is not domain-joined (§4).

## 7. Publish the NDES server authentication template — on `U01PARVMPKI02`

`U01PARVMNDS01`'s IIS site has run on plain HTTP since `ndes-scep-intune-connector` — fine while nothing
outside the lab network could reach it. Publishing it through Application Proxy changes that: this
template gives it its own certificate so the internal hop stays HTTPS too.

Duplicate the built-in `Web Server` template — it already defaults to Server Authentication only and
`Supply in the request` for its Subject Name, so there is nothing to change on either of those tabs; this
is the opposite case from the SCEP template in `ndes-scep-intune-connector`, which had to swap `Web
Server`'s Application Policy for Client Authentication:

```text
Template display name : NDES Server Authentication
Template name          : NDESServerAuthentication
```

**On the Security tab, replace whatever broad `Enroll` permission the duplicate inherited with `Read` +
`Enroll` for `U01PARVMNDS01$` only** — this certificate identifies exactly one server; no other principal
needs to request it.

Publish it: `certsrv.msc` → `UnreadLines Issuing CA` → **Certificate Templates** → **New** → **Certificate
Template to Issue** → select `NDES Server Authentication`.

## 8. Enroll the certificate and bind it to IIS — on `U01PARVMNDS01`

In **IIS Manager**, select the server node, then **Server Certificates → Create Domain Certificate**. This
wizard requests directly against `U01PARVMPKI02` using whichever published template the server's computer
account can enroll for — with §7 published and scoped to `U01PARVMNDS01$` alone, it resolves to `NDES
Server Authentication` without having to name it explicitly. Enter:

- **Common name**: `u01parvmnds01.corp.unreadlines.com`
- **Organization** / **Organizational unit** / **City** / **State** / **Country**: any value — the
  Subject Name tab is display-only for this template (§7); nothing here changes what the certificate
  authorizes.

Bind it to the default site:

```powershell
$cert = Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.Subject -like "*u01parvmnds01.corp.unreadlines.com*" }

New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -IPAddress "*"
(Get-WebBinding -Name "Default Web Site" -Protocol https).AddSslCertificate($cert.Thumbprint, "My")
```

Confirm the SCEP endpoint now answers over HTTPS internally, from `U01PARVMNDS01` itself:

```powershell
Invoke-WebRequest https://u01parvmnds01.corp.unreadlines.com/certsrv/mscep/mscep.dll -UseBasicParsing |
    Select-Object StatusCode
```

## 9. Install and register the Private Network Connector — on `U00PARVMPNC01`

1. In the Entra admin center, go to **Enterprise applications → Application Proxy** and select
   **Download connector service** — this is the same installer Microsoft Learn documents for both
   Application Proxy and Private Access (§3).
2. Copy the installer to `U00PARVMPNC01` and run it.
3. When prompted, sign in with the Application Administrator account from §5. **Internet Explorer
   Enhanced Security Configuration can block this sign-in screen** — Microsoft's own documentation
   flags this; disable it temporarily on `U00PARVMPNC01` if the screen doesn't render.
4. If `U00PARVMPNC01` sits behind an outbound proxy, run the connector's
   `ConfigureOutBoundProxy.ps1` script afterward to point it at that proxy — not needed on this lab's
   network, which reaches the internet directly through `FWL02`.
5. Let the installer finish; it registers the connector against the `unreadlines` tenant automatically.

## 10. Verify the connector is Active — in the Entra admin center

In **Enterprise applications → Application Proxy → Connectors** (or **Global Secure Access → Connectors
→ Private network connectors** — the same object surfaces in both blades, §3), confirm `U00PARVMPNC01`
shows **Active**. Do not continue to §11 until it does — a connector that isn't active yet cannot proxy
anything published against it.

## 11. Publish NDES through Application Proxy — in the Entra admin center

1. **Enterprise applications → New application → On-premises application.**
2. **Name**: `App Proxy - NDES SCEP - Mobile` (§4).
3. **Internal URL**: the root URL of `U01PARVMNDS01`, e.g. `https://u01parvmnds01.corp.unreadlines.com/`
   — the NDES root, not a sub-path. Microsoft's own procedure for this exact scenario publishes the
   root; this screen has no option to publish only a sub-path (the limitation the intro above already
   notes — §12 is where it's actually addressed).
4. **External URL**: leave the tenant default (`*.msappproxy.net`) — no custom domain configured.
5. **Pre-authentication**: **Passthrough**. Not selectable as anything else for this app — SCEP cannot
   complete any interactive sign-in step.
6. **Connector group**: default (`U00PARVMPNC01` is currently the only connector).
7. Save, then copy the generated external URL — needed for §13 and for the Intune SCEP profile in the
   next lab (§16).

## 12. Harden the published endpoint — on `U01PARVMNDS01`

Application Proxy always maps the whole internal URL to the whole external URL (§11) — restricting the
externally reachable surface to the one path SCEP needs has to happen on the NDES server itself, in IIS.

1. Install the **URL Rewrite** module if it isn't already present (§5).
2. In IIS Manager, open the site hosting NDES, then **URL Rewrite → Add Rule(s) → Blocking Rule**.
3. Block every request whose path does **not** match the SCEP endpoint:
   - Pattern: `^certsrv/mscep/mscep\.dll`
   - Requested URL: **Does Not Match the Pattern**
   - Action: **Abort Request** (or **Custom Response**, status `403`)

   Equivalent rule in `web.config`:

   ```xml
   <rule name="Restrict external access to SCEP endpoint only" stopProcessing="true">
     <match url="^certsrv/mscep/mscep\.dll" negate="true" />
     <action type="AbortRequest" />
   </rule>
   ```

4. **This rule applies to every request IIS receives on this site**, including from inside the lab
   network — confirm nothing else on `U01PARVMNDS01` depends on another path on the same site/binding
   before enabling it. `ndes-scep-intune-connector/README.md` builds this server for SCEP alone, so
   nothing else should be sharing the site — but verify on the live server rather than assuming it.

## 13. Test the published endpoint end to end

**From a machine outside the lab network** (a real phone, or a laptop on a different network — testing
from inside the lab network proves nothing about the path this lab actually builds):

1. Browse to the external URL from §11's root (no path). Expect **403 Forbidden** — proof that §12's rule
   is in effect, not just that the app is reachable.
2. Browse to the external URL with `/certsrv/mscep/mscep.dll` appended. Expect the NDES SCEP status page
   (HTTP 200) — the same validation step Microsoft's own procedure uses, here also confirming §12 didn't
   block the one path it's supposed to allow.
3. Back in the Entra admin center, re-check §10 — the connector should still show **Active** after
   handling a real external request.

## 14. Expected final state

- `U00PARVMPNC01` runs the Entra Private Network Connector, shown **Active** in the Entra admin center.
- `U01PARVMNDS01` serves its SCEP endpoint over HTTPS internally, under its own `NDES Server
  Authentication` certificate (§7–§8).
- `App Proxy - NDES SCEP - Mobile` is published, Passthrough preauthentication, internal URL pointed at
  `U01PARVMNDS01`'s root.
- From outside the lab network, the published external URL returns **403** at its root and **200** at
  `/certsrv/mscep/mscep.dll` only.
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

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
