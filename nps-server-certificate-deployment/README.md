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
information`, Subject name format `Common name`, and `DNS name` checked under "Include this information in
alternate subject name". The Subject name format dropdown shows as greyed out ("Control is disabled due to
compatibility settings") — the Compatibility tab's CSP/provider selection locks it, so there's nothing to pick
here even if a different format were wanted. That's fine: EAP-TLS clients validate the server certificate against
its Subject *Alternative* Name, not the CN, and the SAN's `DNS name` — built from `U01PARVMNPS01`'s AD computer
object — is already checked. Confirm it's still checked; there's nothing to enable.

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
    Autoenroll : Allow
```

`U01PARVMNPS01` doesn't appear by default in the **Select Users, Computers, Service Accounts, or Groups** dialog —
click **Object Types...** and check **Computers** first, then type `U01PARVMNPS01` and **Check Names**.

Unlike `PKI Validation` in Phase 7 — a throwaway template deliberately kept to manual-only enrollment —
`NPS Server Authentication` is a production server certificate that has to keep working unattended. Without
`Autoenroll`, nobody notices it's about to expire until `UnreadLines-Mobile` starts failing EAP-TLS handshakes one
day, which is a bad way to find out. `Autoenroll` lets `U01PARVMNPS01` renew this certificate on its own before
`NotAfter`, with no one having to repeat §2 manually.

`Autoenroll` on the template's ACL is necessary but **not sufficient** on its own — it only controls whether the
CA lets `U01PARVMNPS01` autoenroll, not whether the computer actually tries to. That second half is Group Policy:
`Certificate Services Client - Auto-Enrollment`, scoped to `U01PARVMNPS01` — built in §5 below, after the
certificate itself is enrolled and bound. Until §5 is done, granting `Autoenroll` here is necessary groundwork,
but renewal is still effectively manual.

Before publishing, review the full ACL once more and confirm no other broad group (`Domain Computers`,
`Everyone`, or anything else carried over from the duplication) still holds `Enroll` or `Autoenroll` beyond what's
listed above — the same caution Phase 7 called out for its own template.

Publish it: `certsrv.msc` → `UnreadLines Issuing CA` → **Certificate Templates** → **New** → **Certificate
Template to Issue** → select `NPS Server Authentication`.

## 2. Enroll the Certificate — on `U01PARVMNPS01`

In an elevated session (the certificate goes into the `LocalMachine` store, not a user's), either the GUI or the
PowerShell equivalent works — both are mentioned in `radius-nps-deployment/README.md`'s "Next Step" note. **Use
one, not both.** Neither method warns about or detects the other: each is its own enrollment request, so running
both silently issues two separate, equally valid certificates from the same template — same subject, different
thumbprints — leaving two entries to disambiguate for no reason in §3.

This is the certificate's first, manual issuance — §1 granted `Autoenroll` for *renewals*, it doesn't skip this
initial request, and the GPO that actually makes autoenrollment happen isn't built until §5, after this
certificate already exists. (Re-running this lab later — say, after a revocation — is a different story: at that
point the GPO from §5 already exists and applies, so `U01PARVMNPS01` could pick the certificate back up on its
own at the next Group Policy refresh, `gpupdate /force`, before either command below is even run. Check
`certlm.msc` → Personal → Certificates first in that case — if `NPS Server Authentication` is already there,
that's autoenrollment having done its job; don't enroll again manually on top of it.)

```text
certlm.msc → Personal → All Tasks → Request New Certificate
    → Active Directory Enrollment Policy → NPS Server Authentication → Enroll
```

*— or —*

```powershell
Get-Certificate -Template "NPSServerAuthentication" -CertStoreLocation "Cert:\LocalMachine\My"
```

If the template doesn't appear in either path, the CA hasn't propagated the publication yet or the computer
account's `Enroll` right from §1 didn't take — re-check the Security tab on `U01PARVMPKI02` before assuming a
client-side problem.

If both ended up running anyway, nothing is broken — both certificates are equally valid, just redundant. Compare
`NotBefore` to find the one just issued and bind that one in §3:

```powershell
Get-ChildItem Cert:\LocalMachine\My |
    Where-Object Issuer -like "*UnreadLines Issuing CA*" |
    Select-Object Thumbprint, NotBefore, NotAfter, SerialNumber
```

The other is safe to leave for now; clean it up later by revoking it on `U01PARVMPKI02` (`certsrv.msc` → Issued
Certificates → match the `SerialNumber` above → **Revoke**) and removing it from `Cert:\LocalMachine\My`.

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

## 5. Configure Autoenrollment — on `U01PARVMDOM01`

`Autoenroll: Allow` on the template (§1) only grants `U01PARVMNPS01` the *right* to autoenroll — nothing polls for
it without a GPO enabling the autoenrollment client. This follows the same two-part pattern
`ad-cs-pki-deployment/README.md` already used twice: Phase 6 (Root CA trust) creates a GPO and imports a
certificate, Phase 5 (CA auditing) creates a GPO and flips a policy setting; this is the same shape, but for
autoenrollment.

**Create and scope the GPO.** On `U01PARVMDOM01`:

```text
gpmc.msc
  → Domains → corp.unreadlines.com
    → Create a GPO in this domain, and Link it here...
      Name: PKI - NPS Autoenrollment
```

Unlike `PKI - Trusted Root CA` (deliberately domain-wide — every domain computer needs Root trust), this one
follows the narrower `PKI - Issuing CA Audit` pattern instead: scope it to `U01PARVMNPS01` only, mirroring how
the template's own ACL (§1) restricts enrollment to that one server rather than to `RAS and IAS Servers` at
large.

**Security Filtering.** Add `U01PARVMNPS01` (`Object Types → Computers`), then remove `Authenticated Users` from
Security Filtering.

**Delegation tab.** Confirm (add if missing):

```text
Authenticated Users
    Read                Allow
    Apply Group Policy  No

U01PARVMNPS01
    Read                Allow
    Apply Group Policy  Allow
```

Never set `Deny` on `Authenticated Users` — leaving it at no `Apply Group Policy` right is sufficient and
reversible, the same caution `ad-cs-pki-deployment/README.md` Phase 5 calls out for its own scoped GPO.

**Configure the policy.** Edit `PKI - NPS Autoenrollment`:

```text
Computer Configuration
  -> Policies
    -> Windows Settings
      -> Security Settings
        -> Public Key Policies
          -> Certificate Services Client - Auto-Enrollment
             Configuration Model : Enabled
             Renew expired certificates, update pending
                certificates, and remove revoked certificates : checked
             Update certificates that use certificate templates : checked
```

**Apply and verify — on `U01PARVMNPS01`:**

```powershell
gpupdate /target:computer /force

gpresult /scope computer /r
```

Confirm `PKI - NPS Autoenrollment` appears under "Applied Group Policy Objects", not "Denied Group Policy
Objects" — a Security Filtering or Delegation mistake above shows up here as a silent non-apply, not an error
dialog. Then trigger an autoenrollment cycle rather than waiting for the default interval, and confirm the
certificate already enrolled in §2 is still recognized (this is also what a real renewal will look like, later,
closer to `NotAfter`):

```powershell
certutil -pulse

Get-ChildItem Cert:\LocalMachine\My |
    Where-Object Issuer -like "*UnreadLines Issuing CA*" |
    Select-Object Thumbprint, NotBefore, NotAfter
```

That `-pulse` above doesn't really prove autoenrollment *works* — nothing changes when a valid certificate already
exists, so a broken GPO and a working one look identical at that point. A more convincing test: delete a
certificate and see if it comes back.

The autoenrollment client doesn't remember what it deleted — at every pulse it re-derives, from scratch, which
templates `U01PARVMNPS01` currently has `Autoenroll` rights on (via AD, per §1's ACL) and whether a valid
certificate matching each one currently exists in the store. No matching certificate — deleted, expired, or never
issued, the check doesn't distinguish between those — means it requests a new one. That's the same check a real
renewal relies on later, just triggered by absence instead of an approaching `NotAfter`.

⚠️ Run this against the **duplicate** certificate from §2 if one still exists, not the one bound in `nps.msc`
(§3) — deleting the bound certificate breaks `UnreadLines-Mobile - EAP-TLS`'s EAP-TLS binding immediately, since
`nps.msc` doesn't automatically rebind to a replacement; that would mean redoing §3 by hand afterward just to
recover, not to test anything.

```powershell
Get-ChildItem Cert:\LocalMachine\My |
    Where-Object Issuer -like "*UnreadLines Issuing CA*" |
    Select-Object Thumbprint, NotBefore, NotAfter
# note the Thumbprint of the certificate you're about to delete — NOT the one bound in nps.msc

Remove-Item -Path "Cert:\LocalMachine\My\<Thumbprint-to-delete>" -Force

certutil -pulse

Get-ChildItem Cert:\LocalMachine\My |
    Where-Object Issuer -like "*UnreadLines Issuing CA*" |
    Select-Object Thumbprint, NotBefore, NotAfter
```

A new entry with a **different Thumbprint** confirms autoenrollment actually re-issued the certificate, not just
that the GPO applied without error.

From here on, `U01PARVMNPS01` renews `NPS Server Authentication` on its own before `NotAfter` — no need to repeat
§2 manually unless the certificate is revoked or the template changes.

## 6. Update the Infrastructure Inventory

Per `server-naming-convention.md` §2, update `standards/vm-inventory.md`'s `U01PARVMNPS01` row: the note "NPS
server certificate pending" no longer applies — replace it with something like "NPS server certificate issued
(`NPS Server Authentication`, `UnreadLines Issuing CA`), autoenrolled/renewed via `PKI - NPS Autoenrollment`, and
bound to `UnreadLines-Mobile - EAP-TLS`; SSID activation still pending `RAP01` switch-over." Leave the `Status`
column as-is until `UnreadLines-Mobile` is actually live — this lab removes one blocker, not all of them.

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
