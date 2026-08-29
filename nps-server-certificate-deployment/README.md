# Issue the NPS Server Certificate — Activate EAP-TLS for `UnreadLines-Mobile`

This lab publishes `U01PARVMPKI02`'s first production certificate template — the **Server Authentication**
template for `U01PARVMNPS01` — enrolls it, and binds it into the Network Policy Server configuration. This is
exactly item 1 and 2 of the "Next Step" section at the end of `radius-nps-deployment/README.md`: NPS is fully
built and validated at the RADIUS protocol level (`radtest` PAP passes), but the `UnreadLines-Mobile - EAP-TLS`
Network Policy created there has no certificate to present, so any real EAP-TLS handshake still fails at the TLS
layer. It is also the first template published after `ad-cs-pki-deployment/README.md`'s Checkpoint 2, which
explicitly stopped short of "production certificate templates, permissions, auto-enrollment, and other
integrations" — that work starts here.

## Mutual Authentication in EAP-TLS

`UnreadLines-Mobile` is EAP-TLS end-to-end: both sides of the handshake authenticate with a certificate, not just
the client. Until this lab, `U01PARVMNPS01` has nothing to present, so any real association attempt fails at the
TLS layer before AD authentication is ever reached — this is exactly the gap the "Next Step" note in
`radius-nps-deployment/README.md` describes.

At the protocol level, the device first has to convince itself it's actually talking to the corporate RADIUS
server, not an impersonator broadcasting the same SSID:

```text
Mobile device
    |
    | 802.1X association attempt on UnreadLines-Mobile
    v
U01PARVMRAP01 (802.1X authenticator, forwards to RADIUS)
    |
    v
U01PARVMNPS01 (RADIUS / NPS, EAP-TLS)
    |
    | presents its Server Authentication certificate
    v
Mobile device verifies:
    - certificate valid (not expired, not revoked)?
    - issued by `UnreadLines Issuing CA`?
    - name matches the server it's connecting to (`U01PARVMNPS01`'s SAN — see §1's Subject Name tab)?
    |
    v
If OK → EAP-TLS continues to client authentication
```

Two different certificates carry the two different EKUs of that same handshake, and they come from two different
places in this project:

```text
U01PARVMNPS01
    |
    +-- Server Authentication certificate ← this lab
            "I am the legitimate UnreadLines-Mobile RADIUS server."

Mobile device (later — Intune/SCEP)
    |
    +-- Client Authentication certificate
            "I am this specific AD user/device, permitted in GG-U01-PAR-WiFi-Mobile."
```

Put together, the full handshake this lab is one half of looks like:

```text
                 1. Server authentication
Mobile device  <----------------------------  U01PARVMNPS01
                 Server Authentication certificate (this lab)

                 2. Client authentication
Mobile device  ---------------------------->  U01PARVMNPS01
                 Client Authentication certificate (Intune/SCEP, later)
                            |
                            v
                    Strong certificate mapping
                            |
                            v
                    AD user/device identity → GG-U01-PAR-WiFi-Mobile
```

This lab closes out the top half only. The bottom half — issuing a Client Authentication certificate to a real
mobile device — needs the SCEP user template, NDES, and the Intune Certificate Connector, none of which exist
yet.

## Where This Lab Fits

`infrastructure/plan-wifi-eap-tls-scep-intune.md` lays out the full video sequence; the certificate/RADIUS chain
that leads to a working `UnreadLines-Mobile` is:

```text
1.  Root CA + Issuing CA (PKI)                        — ad-cs-pki-deployment            done
2.  NPS / RADIUS server                               — radius-nps-deployment           done
3.  NPS server certificate (Server Authentication)    — this lab                        ← now
4.  SCEP user certificate template                    — video TBD
5.  NDES server + Intune Certificate Connector        — video TBD
6.  Entra Private Network Connector + App Proxy       — video TBD
7.  Intune trusted-certificate profiles (Root/Issuing) — video TBD
8.  Intune SCEP profile (mobile client certificate)   — video TBD
9.  Intune Wi-Fi EAP-TLS profile                      — video TBD
10. RAP01 switched to UnreadLines-Mobile              — video TBD
11. Real EAP-TLS handshake, confirmed in NPS logs     — video TBD
```

Microsoft Entra Connect / hybrid identity (video TBD) runs in parallel, not in this chain — it fixes the `NULL SID`
failure mode documented in `infrastructure/analyse-pki-intune-nps.md`, but nothing in this lab or the next few
videos depends on it being done first.

## Scope and Dependencies

This lab covers publishing the template on the Issuing CA, enrolling the certificate on `U01PARVMNPS01`, and
binding it to the existing Network Policy's EAP-TLS constraint.

It does **not** activate `UnreadLines-Mobile` end-to-end. That still needs, per `radius-nps-deployment/README.md`'s
"Next Step" section (items 3–6, later videos): a client-authentication certificate issued to each permitted
device via Intune/SCEP, `U01PARVMRAP01` switched over from `UnreadLines-Guest` to `UnreadLines-Mobile`
(`hostapd-wifi-access-point-lab/README.md` §13), and a real 802.1X handshake confirmed in the NPS event log.
None of that is in scope here — this lab only removes the one blocker specific to the server side.

## Target Configuration

| Item | Value |
| --- | --- |
| CA | `U01PARVMPKI02` (`UnreadLines Issuing CA`) |
| Duplicated from | built-in `RAS and IAS Server` template |
| Template display name | `NPS Server Authentication` |
| Template name (internal) | `NPSServerAuthentication` |
| EKU | Client Authentication + Server Authentication (inherited unchanged from `RAS and IAS Server`) |
| Enrollment scope | `U01PARVMNPS01` computer account only |
| Enrolled on | `U01PARVMNPS01` |
| Bound to | Network Policy `UnreadLines-Mobile - EAP-TLS` (`radius-nps-deployment/README.md` §6) |

## Prerequisites

- `U01PARVMPKI02` is up and reachable, `ad-cs-pki-deployment/README.md` Checkpoint 2 reached.
- `U01PARVMNPS01` is built, domain-joined, and registered in `RAS and IAS Servers` (`radius-nps-deployment/README.md`
  §1–§2).
- The Network Policy `UnreadLines-Mobile - EAP-TLS` already exists with `Microsoft: Smart Card or other
  certificate` as its only authentication method (`radius-nps-deployment/README.md` §6) — it just has nothing to
  bind yet.
- Administrative access on both `U01PARVMPKI02` and `U01PARVMNPS01`.

## 1. Publish the Server Authentication Template — on `U01PARVMPKI02`

`certtmpl.msc` → right-click the built-in `RAS and IAS Server` template → **Duplicate Template**.

**General tab:**

```text
Template display name : NPS Server Authentication
Template name          : NPSServerAuthentication
```

Leave **Compatibility** at whatever level the wizard defaults to — nothing about this template needs a specific
compatibility floor beyond what `RAS and IAS Server` already assumes.

**Subject Name tab — nothing to change.** This is exactly why `RAS and IAS Server` is the standard starting point
for a RADIUS server certificate: duplicating it already carries over `Build from this Active Directory
information`, Subject Name Format `None`, and `DNS name` checked under Alternate subject name. The Subject
Alternative Name built from `U01PARVMNPS01`'s AD computer object is what an EAP-TLS client actually validates the
server certificate against later — confirm it's still checked, but there's nothing to enable.

**Extensions tab — nothing to change.** `Application Policies` already lists both `Client Authentication` and
`Server Authentication`, inherited from the built-in template. NPS only needs `Server Authentication`, but the
extra EKU is the built-in template's own design and isn't worth stripping out for this lab.

**Security tab — this is the actual scoping work.** Registering `U01PARVMNPS01` in AD (`radius-nps-deployment/README.md`
§2, `netsh ras add registeredserver`) already made it a member of the built-in `RAS and IAS Servers` group, and
duplicating `RAS and IAS Server` carries over that group's inherited `Enroll` right — which would scope
enrollment to *any* current or future RAS/IAS server, not to `U01PARVMNPS01` specifically. Remove that inherited
breadth and grant enrollment to the one computer account instead, the same restrictive pattern
`ad-cs-pki-deployment/README.md` Phase 7 used for the `PKI Validation` template:

```text
Authenticated Users
    Read       : Allow
    Enroll     : Not granted
    Autoenroll : Not granted

RAS and IAS Servers (built-in group)
    Enroll     : Not granted   ← remove, do not leave inherited
    Autoenroll : Not granted

U01PARVMNPS01 (computer account)
    Read       : Allow
    Enroll     : Allow
    Autoenroll : Not granted
```

`U01PARVMNPS01` doesn't appear by default in the **Select Users, Computers, Service Accounts, or Groups** dialog —
click **Object Types...** and check **Computers** first, then type `U01PARVMNPS01` and **Check Names**.
`Autoenroll` is deliberately left ungranted: this lab enrolls manually (§2 below), matching how the original
"Next Step" note phrased it (`certlm.msc` or `Get-Certificate`), not GPO-driven autoenrollment. Before publishing,
review the full ACL once more and confirm no other broad group (`Domain Computers`, `Everyone`, or anything else
carried over from the duplication) still holds `Enroll` or `Autoenroll` — the same caution Phase 7 called out for
its own template.

Publish it: `certsrv.msc` → `UnreadLines Issuing CA` → **Certificate Templates** → **New** → **Certificate
Template to Issue** → select `NPS Server Authentication`.

## 2. Enroll the Certificate — on `U01PARVMNPS01`

In an elevated session (the certificate goes into the `LocalMachine` store, not a user's), either the GUI or the
PowerShell equivalent works — both are mentioned in `radius-nps-deployment/README.md`'s "Next Step" note:

```text
certlm.msc → Personal → All Tasks → Request New Certificate
    → Active Directory Enrollment Policy → NPS Server Authentication → Enroll
```

```powershell
Get-Certificate -Template "NPSServerAuthentication" -CertStoreLocation "Cert:\LocalMachine\My"
```

If the template doesn't appear in either path, the CA hasn't propagated the publication yet or the computer
account's `Enroll` right from §1 didn't take — re-check the Security tab on `U01PARVMPKI02` before assuming a
client-side problem.

Confirm it landed and carries the expected issuer and EKU:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
    Where-Object Issuer -like "*UnreadLines Issuing CA*" |
    Select-Object Subject, Issuer, NotAfter, Thumbprint

(Get-ChildItem Cert:\LocalMachine\My |
    Where-Object Issuer -like "*UnreadLines Issuing CA*").EnhancedKeyUsageList
```

Expect `Server Authentication` (and `Client Authentication`) in the EKU list, and a `Subject`/SAN referencing
`U01PARVMNPS01`. As a domain member, `U01PARVMNPS01` already trusts `UnreadLines Issuing CA` the same way every
other domain-joined server in this lab does — `ad-cs-pki-deployment/README.md` Phase 7 already proved the
CDP/AIA publication path works end-to-end from a clean client, so there's no need to repeat that validation here.

## 3. Bind the Certificate to the Network Policy — on `U01PARVMNPS01`

```text
nps.msc → Policies → Network Policies → UnreadLines-Mobile - EAP-TLS → Properties
    → Constraints tab → Authentication Methods
    → select Microsoft: Smart Card or other certificate → Edit...
    → Certificate issued to: <the NPS Server Authentication certificate enrolled in §2>
    → OK → Apply
```

The **Edit...** dialog only lists certificates in `LocalMachine\My` that carry the Server Authentication EKU and
a private key — if more than one shows up, match it by the Subject/Thumbprint confirmed in §2. Restart the NPS
service so nothing keeps the old (absent) binding cached:

```powershell
Restart-Service IAS
```

(`IAS` is the actual Windows service name behind Network Policy Server — a legacy artifact of NPS's RADIUS/IAS
lineage, not a typo.)

## 4. Verify the Binding

Reopen the policy and confirm the certificate selection persisted:

```text
nps.msc → Network Policies → UnreadLines-Mobile - EAP-TLS → Properties
    → Constraints → Authentication Methods → Edit...
```

Check for any service-level error on the restart:

```powershell
Get-WinEvent -LogName System -MaxEvents 20 |
    Where-Object { $_.ProviderName -eq "Service Control Manager" -and $_.Message -like "*IAS*" }
```

**What this does not yet prove:** a real EAP-TLS handshake still won't succeed. `UnreadLines-Mobile` has no
client certificates issued to any device yet, and `U01PARVMRAP01` hasn't switched over from `UnreadLines-Guest`.
Both are later, separate work (see Next Step below) — this lab's success condition is narrower: the Network
Policy now has a valid, correctly-scoped Server Authentication certificate bound to it, where before it had none.

## 5. Update the Infrastructure Inventory

Per `server-naming-convention.md` §2, update `standards/vm-inventory.md`'s `U01PARVMNPS01` row: the note "NPS
server certificate pending" no longer applies — replace it with something like "NPS server certificate issued
(`NPS Server Authentication`, `UnreadLines Issuing CA`) and bound to `UnreadLines-Mobile - EAP-TLS`; SSID
activation still pending `RAP01` switch-over." Leave the `Status` column as-is until `UnreadLines-Mobile` is
actually live — this lab removes one blocker, not all of them.

## Next Step

`radius-nps-deployment/README.md`'s "Next Step" section, items 3–6, remain: issue client-authentication
certificates to permitted devices via Intune/SCEP (`README-Intune-SCEP-WiFi-EAP-TLS.md`'s design), build
`UnreadLines-Mobile` and switch `RAP01` over from `UnreadLines-Guest`
(`hostapd-wifi-access-point-lab/README.md` §13), and confirm a real `Access-Accept` in the NPS event log. Per
`infrastructure/plan-wifi-eap-tls-scep-intune.md`, those land across the videos that follow, after the SCEP user
certificate template and the NDES/Intune Certificate Connector server exist.

## References

- [Deploy client certificates for PEAP and EAP-TLS — Microsoft Learn](https://learn.microsoft.com/windows-server/networking/technologies/nps/nps-deploy-client-certs)
- [Configure NPS server certificates — Microsoft Learn](https://learn.microsoft.com/windows-server/networking/technologies/nps/nps-deploy-server-certs)
- [Certificate Templates overview — Microsoft Learn](https://learn.microsoft.com/windows-server/identity/ad-cs/certificate-templates-overview)
- [Duplicate a Certificate Template — Microsoft Learn](https://learn.microsoft.com/windows-server/identity/ad-cs/manage-certificate-templates)
- [RFC 5216 — The EAP-TLS Authentication Protocol](https://www.rfc-editor.org/rfc/rfc5216)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
