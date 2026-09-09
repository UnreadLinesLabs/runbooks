# Deploy the SCEP certificate template, NDES, and the Intune Certificate Connector

This lab publishes `U01PARVMPKI02`'s first certificate template aimed at mobile devices, and builds
`U01PARVMNDS01` — a new server running the Network Device Enrollment Service (NDES) role and the
Certificate Connector for Microsoft Intune. Together they let an Intune-managed phone request a
certificate from the on-prem PKI (`ad-cs-pki-deployment`) over SCEP, without ever handing the phone a
domain credential. It sits next to `nps-server-certificate-deployment`, which closed the *server*
half of the `UnreadLines-Mobile` EAP-TLS handshake; this lab is the first step of the *client* half.

NDES and the Certificate Connector are installed on the same host by Microsoft's own design when SCEP
is backed by a local (non-cloud) CA — not a choice made for this lab. What this lab deliberately does
not do is expose that server to the internet: until `U00PARVMPNC01` and Entra Application Proxy exist
(§15), `U01PARVMNDS01` only has to be reachable from inside the lab network, and the Connector only
needs a single outbound path to the Intune service.

A **new**, dedicated server — rather than adding this template to an NDES role already running
elsewhere — is not a stylistic choice either. A single NDES instance holds exactly one set of
`EncryptionTemplate`/`GeneralPurposeTemplate`/`SignatureTemplate` registry values (§8): global to that
server, not scoped per SCEP profile. Pointing an existing NDES server at `Intune SCEP Mobile User`
would silently stop it from issuing whatever it already serves — a production example being a server
already handling SCEP for Windows Autopilot device certificates. A second SCEP use case always needs
its own NDES server — its own Windows Server, IIS, NDES role, and Certificate Connector instance —
never a repoint of one already in service. This is otherwise the standard, Microsoft-documented
procedure (§16); nothing about it is lab-specific beyond the server and template names.

## 1. Architecture

```text
U01PARVMPKI01   Offline Root CA
      |  signs
      v
U01PARVMPKI02   Issuing CA                                192.168.20.143
      |  issues on request, using the "Intune SCEP Mobile User" template
      v
U01PARVMNDS01   NDES (IIS) + Intune Certificate Connector  192.168.20.147
      |
      |  gmsa-ndes$ requests/signs SCEP certificates on the CA's behalf
      |
      +---- outbound HTTPS 443 ---->  Microsoft Intune service
                                       (the Connector polls for pending SCEP
                                       requests — no inbound port is opened
                                       on U01PARVMNDS01 by this lab)

Not built yet — see §15 "Next step":
      U00PARVMPNC01 (Entra Private Network Connector) + Entra Application Proxy,
      the path a real mobile device will use to reach this NDES endpoint from
      outside the lab network.
```

## 2. Where this lab fits

The certificate and RADIUS chain toward a working `UnreadLines-Mobile`
(`YouTube/backlog.md` §2):

```text
1.  Root CA + Issuing CA (PKI)                              — ad-cs-pki-deployment            done
2.  NPS / RADIUS server                                     — radius-nps-deployment           done
3.  NPS server certificate (Server Authentication)          — nps-server-certificate-deployment done
4.  SCEP template + NDES + Intune Certificate Connector     — this lab                        ← now
5.  Entra Private Network Connector + Application Proxy     — video TBD
6.  Intune trusted-certificate profiles (Root/Issuing)      — intune-trusted-certificate-profiles done
7.  Intune SCEP profile (mobile client certificate)         — video TBD
8.  Intune Wi-Fi EAP-TLS profile                            — video TBD
9.  U01PARVMRAP01 switched to UnreadLines-Mobile            — video TBD
10. Real EAP-TLS handshake, confirmed in NPS logs           — video TBD
```

Item 6 was recorded and published ahead of this lab because it only depended on the PKI (item 1), not
on anything built here. This lab is the harder, server-side half of item 4 in that same chain; nothing
downstream of it can be tested end to end until item 5 also exists.

This lab also depends on two general-purpose identity labs that sit outside the `UnreadLines-Mobile`
chain entirely — `configure-kds-root-key-for-gmsa` and `create-gmsa-account` — because the gMSA this
lab uses is provisioned there, not here. See §5.

## 3. Scope and dependencies

This lab covers: publishing and hardening the SCEP certificate template on `U01PARVMPKI02`, building
`U01PARVMNDS01`, installing the NDES role and pointing it at the template, switching NDES to run under
the gMSA, and installing and registering the Intune Certificate Connector.

It does **not** cover:

- Creating the gMSA itself. `gmsa-ndes$` is provisioned by `create-gmsa-account/README.md` — a
  reusable lab, not specific to NDES — which in turn depends on `configure-kds-root-key-for-gmsa/README.md`
  if this is the first gMSA created in this forest. This lab only consumes that account; see §5.
- Exposing `U01PARVMNDS01` to the internet. That is Entra Private Network Connector + Application Proxy
  (§15) — until it exists, no device outside the lab network can reach this SCEP endpoint at all.
- The Intune SCEP certificate profile that actually targets a group of mobile devices — a later lab
  (item 7 above).
- A real certificate request from a phone. This lab's success condition is narrower: the template,
  NDES, and the Connector are correctly configured and the Connector reports healthy — not that a
  device has enrolled.

## 4. Target configuration

| Item | Value |
| --- | --- |
| CA | `U01PARVMPKI02` (`UnreadLines Issuing CA`) |
| Duplicated from | built-in `Web Server` template |
| Template display name | `Intune SCEP Mobile User` |
| Template name (internal) | `IntuneSCEPMobileUser` |
| Application Policy (EKU) | `Client Authentication` only |
| Subject name | Supply in the request (`Web Server`'s own default — nothing to change) |
| Request Handling — Purpose | `Signature` only |
| Private key | Not exportable, 2048-bit RSA |
| NDES / Connector server | `U01PARVMNDS01`, `192.168.20.147/25`, Subnet 2 |
| NDES service account | `gmsa-ndes$` (gMSA, provisioned in `create-gmsa-account/README.md`) |
| Enrolled on | `U01PARVMNDS01` (NDES role, IIS `SCEP` application pool) |

## 5. Prerequisites

- `ad-cs-pki-deployment/README.md` Checkpoint 2 reached: `U01PARVMPKI02` up, reachable, and already
  issuing from at least one production template (`nps-server-certificate-deployment/README.md` did the
  first one).
- `U01PARVMNDS01` built per `prepare-windows-ubuntu-templates/README.md`, renamed and domain-joined per
  `reference/naming-conventions.md` §2 and §8 — Windows Server 2025, nothing else installed on it yet.
- **`gmsa-ndes$` already exists and is installed and tested on `U01PARVMNDS01`** — done in
  `create-gmsa-account/README.md` (`Install-ADServiceAccount` / `Test-ADServiceAccount` returning
  `True`), itself depending on the KDS root key from `configure-kds-root-key-for-gmsa/README.md` if
  this was the first gMSA created in the forest. Nothing in this lab creates or tests the account —
  §9 and §10 only consume it.
- Administrative access on `U01PARVMPKI02` and `U01PARVMNDS01`.
- An account with the Intune Administrator (or Global Administrator) role on the `unreadlines` tenant,
  to register the Certificate Connector.
- Outbound HTTPS (443) from `U01PARVMNDS01` to the Microsoft Intune service endpoints. No inbound port
  is opened on `U01PARVMNDS01` in this lab — see §3.

## 6. Publish the SCEP certificate template — on `U01PARVMPKI02`

1. `certtmpl.msc` → right-click the built-in `Web Server` template → **Duplicate Template**.

2. **Compatibility tab — set this before touching any other tab, it changes what they offer.**

   ```text
   Certification Authority : Windows Server 2003
   Certificate recipient    : Windows XP / Server 2003
   ```

   NDES only works with a certificate template built on a **Legacy Cryptographic Service Provider**
   (CSP) — not the CNG-based Key Storage Provider (KSP) that newer templates default to. The duplication
   wizard on a modern CA suggests a much newer Compatibility level by default (`Windows Server 2016` /
   `Windows 10 - Windows Server 2016` is common) with a "Show resulting changes" checkbox — accepting
   that default silently switches the Cryptography tab's Provider Category to Key Storage Provider,
   which NDES cannot use. Dropping both settings back to `Windows Server 2003` / `Windows XP/Server
   2003` — the same floor the built-in `Web Server` template itself already assumes — is what keeps
   Legacy CSP selected; confirmed on the Cryptography tab in step 7 below.

3. **General tab:**

   ```text
   Template display name : Intune SCEP Mobile User
   Template name          : IntuneSCEPMobileUser
   ```

   Leave **Publish certificate in Active Directory** unchecked — it already is on `Web Server`, and a
   SCEP client certificate has no AD object of its own to publish against.

4. **Subject Name tab — nothing to change.** `Web Server` already has Subject Name set to *Supply in
   the request*, which is exactly what the Intune policy module for NDES requires: it builds the subject
   and SAN (`CN={{UserPrincipalName}}`, SAN `UPN={{UserPrincipalName}}`) itself, from the SCEP profile
   that will be created in a later lab, not from anything stored in AD. This is the reason to duplicate
   `Web Server` rather than `User` — `User` would default to *Build from Active Directory information*
   and need this tab changed by hand.

5. **Extensions tab — this needs to change.** `Web Server`'s built-in Application Policy is `Server
   Authentication`, which is backwards for a client certificate:

   ```text
   Application Policies
       Remove : Server Authentication
       Add    : Client Authentication
   ```

   Never leave both EKUs on this template: a certificate that can also serve as Server Authentication is
   a certificate that could impersonate a server elsewhere in the domain, for a certificate whose only
   intended job is proving a mobile user's identity to NPS.

6. **Request Handling tab — this needs to change too.** `Web Server` defaults to a purpose suited to TLS
   key exchange, not to signing a client authentication handshake:

   ```text
   Purpose                              : Signature
   Allow private key to be exported     : unchecked (confirm — do not enable)
   ```

   Restricting `Purpose` to `Signature` changes what lands in the issued certificate's Key Usage
   extension: `Digital Signature` alone, instead of the `Key Encipherment` + `Digital Signature` pairing
   `Web Server` carries for a server that has to decrypt an RSA key exchange. A client authentication
   certificate only ever signs a challenge to prove key possession during the handshake — it never
   decrypts anything — so the same least-privilege reasoning as the EKU change in step 5 applies to the
   key's own cryptographic capability, not just to the policy that names it.

7. **Cryptography tab — confirm, don't just glance at it:**

   ```text
   Provider Category   : Legacy Cryptographic Service Provider
   Minimum key size     : 2048
   ```

   This is the tab step 2's Compatibility setting was protecting. If Provider Category shows **Key
   Storage Provider** instead, the Compatibility tab is still too high — go back to step 2 rather than
   trying to force a legacy CSP under a KSP-level template, which the UI doesn't cleanly allow.

8. **Security tab — remove the broad default, grant the two accounts that actually need this template:**

   ```text
   Authenticated Users
       Enroll     : Not granted   ← remove, do not leave inherited from Web Server

   gmsa-ndes$ (the NDES service account — see Prerequisites, §5)
       Read       : Allow
       Enroll     : Allow

   <your own admin account, or an Intune Administrators group>
       Read       : Allow
   ```

   The Read-only grant matters on its own: whoever builds the Intune SCEP profile in a later lab has to
   be able to browse to this template from the Intune admin center, and that lookup fails silently
   without it. Skip this line if that person's account is already a member of `Domain Admins` or
   `Enterprise Admins` — both inherit `Read`/`Write`/`Enroll` on every template by default, `Web Server`
   included, so the grant above is only needed for an account outside those two groups. Removing
   `Enroll` from `Domain Admins`/`Enterprise Admins` themselves is out of scope for this lab's security
   requirements (`notes/wifi-mobile-certificate-chain.md` §5 only targets `gmsa-ndes$` and `Authenticated
   Users`) and wouldn't add real protection anyway — `Write` already lets either group re-grant itself
   `Enroll` at any time.

9. Publish it: `certsrv.msc` → `UnreadLines Issuing CA` → **Certificate Templates** → **New** →
   **Certificate Template to Issue** → select `Intune SCEP Mobile User`.

## 7. Install the NDES role — on `U01PARVMNDS01`

Add the role and its IIS dependency:

```powershell
Install-WindowsFeature ADCS-Device-Enrollment -IncludeManagementTools
```

**Configure it with the built-in Application Pool Identity, not the gMSA yet.** The NDES configuration
step does not support a gMSA directly — this is a documented Microsoft limitation, not something to
work around by fiddling with the wizard. Doing it in two steps (this section, then §10) is the
Microsoft-documented order, not a workaround improvised for this lab:

```powershell
Install-AdcsNetworkDeviceEnrollmentService -ApplicationPoolIdentity `
    -CAConfig "U01PARVMPKI02.corp.unreadlines.com\UnreadLines Issuing CA" `
    -RAName "UnreadLines NDES RA" `
    -RACountry "FR" `
    -RACompany "UnreadLines" `
    -SigningProviderName "Microsoft Strong Cryptographic Provider" `
    -SigningKeyLength 2048 `
    -EncryptionProviderName "Microsoft Strong Cryptographic Provider" `
    -EncryptionKeyLength 2048
```

This is the non-interactive equivalent of the **Configure Active Directory Certificate Services...**
wizard's Network Device Enrollment Service page — using it instead of the GUI avoids a wizard screen
that has no gMSA-aware option anyway. Two certificates are requested automatically as part of this step,
under the built-in Application Pool Identity used above: `CEP Encryption` and `Exchange Enrollment
Agent (Offline request)`, both from their own built-in templates. These are NDES's own
registration-authority certificates — used to sign and encrypt the exchange between NDES and the CA —
and are unrelated to `Intune SCEP Mobile User`, which is the template actual client devices will be
issued from (configured next, in §8). Nothing about this lab's scope requires changing which templates
back these two.

Confirm the site bindings look right for a lab with no external hostname yet:

```powershell
Get-Website -Name "Default Web Site" | Select-Object -ExpandProperty Bindings
```

## 8. Point NDES at the SCEP template via the registry — on `U01PARVMNDS01`

The NDES Configuration Wizard in §7 only registers its own RA templates; it has no field for the actual
SCEP client template, which is set separately in the registry:

```powershell
$path = "HKLM:\SOFTWARE\Microsoft\Cryptography\MSCEP"

Set-ItemProperty -Path $path -Name "EncryptionTemplate"     -Value "IntuneSCEPMobileUser"
Set-ItemProperty -Path $path -Name "GeneralPurposeTemplate" -Value "IntuneSCEPMobileUser"
Set-ItemProperty -Path $path -Name "SignatureTemplate"      -Value "IntuneSCEPMobileUser"
```

**Use the template name from §6's General tab, not its display name** — `IntuneSCEPMobileUser`, no
spaces. NDES always expects all three registry values populated, whether or not the underlying
template actually differentiates purposes. All three point at the same template here because `Intune
SCEP Mobile User` was configured for `Signature` only (§6); a template split across separate
encryption/signature purposes would need different values, which this one deliberately avoids.

These three values are also global to this NDES instance, not scoped to one SCEP profile — the reason
this lab builds a brand-new server rather than repointing an existing NDES role (see the note under
§1's introduction on `Architecture`).

Restart IIS so the new mapping takes effect:

```powershell
iisreset
```

## 9. Grant the service account certificate-management rights — on `U01PARVMPKI02`

Needed so the Certificate Connector (§11) can act on revocation requests coming from Intune, not just
issue new certificates:

```text
certsrv.msc → right-click UnreadLines Issuing CA → Properties → Security tab
    → Add: gmsa-ndes$
    → Issue and Manage Certificates : Allow
    → Request Certificates : Allow (already checked by default when adding a new principal here —
      leave it, do not uncheck it)
```

**Leave `Request Certificates` checked — it is not optional.** It is what lets `gmsa-ndes$` submit a
request to the CA at all; it is separate from the template-level `Enroll` right (§6, which says *which*
template it can request) and from `Issue and Manage Certificates` (which governs managing certificates
already issued, not submitting new ones). Once NDES runs as `gmsa-ndes$` (§10), this account is the one
actually submitting the certificate request to the CA on the phone's behalf, using the RA certificates
from §7 — without `Request Certificates` that submission is refused even with the other two rights in
place.

This overall grant is still narrower than Domain Admin or CA Administrator — `gmsa-ndes$` still can't
change the CA's own configuration, only request and manage the certificates it's responsible for.

## 10. Switch the NDES application pool to the gMSA — on `U01PARVMNDS01`

`U01PARVMNDS01` is still running the `SCEP` application pool under the built-in Application Pool
Identity from §7. This section is the documented Microsoft procedure for moving it to `gmsa-ndes$`
afterward — it does **not** redo the RA certificates or the registry mapping from §7–§8; only the
application pool identity and the private-key permissions on the two RA certificates change.

Add the gMSA to the local `IIS_IUSRS` group, now that IIS exists on this host:

```powershell
Add-LocalGroupMember -Group "IIS_IUSRS" -Member "CORP\gmsa-ndes$"
```

Switch the application pool identity:

```text
inetmgr → Application Pools → SCEP → Advanced Settings...
    → Identity → Custom account → CORP\gmsa-ndes$, password fields left blank
```

The password fields greying out on their own, rather than rejecting a blank password, is what confirms
IIS recognized this as a gMSA rather than a mistyped account.

Grant the gMSA **Read** access to the private keys of the two RA certificates §7 requested under the
old identity — without this, NDES can find the certificates but can't use them once the running
identity changes:

```text
certlm.msc → Certificates (Local Computer) → Personal → Certificates
    → CEP Encryption → All Tasks → Manage Private Keys...
        → Add: gmsa-ndes$ → Read : Allow
    → Exchange Enrollment Agent (Offline request) → All Tasks → Manage Private Keys...
        → Add: gmsa-ndes$ → Read : Allow
```

Restart IIS so the new identity and permissions take effect:

```powershell
iisreset
```

Confirm NDES still answers under the new identity before moving on — a blank or error response here
means the private-key permissions above didn't take, not that something is wrong further down the
chain:

```powershell
Invoke-WebRequest "http://localhost/certsrv/mscep/mscep.dll" -UseBasicParsing |
    Select-Object StatusCode
```

Expect `200`. If this fails, re-check the two **Manage Private Keys** grants above before assuming a
registry or CA problem.

## 11. Install and register the Intune Certificate Connector — on `U01PARVMNDS01`

Download `IntuneCertificateConnector.exe` from the Intune admin center (**Tenant administration** →
**Connectors and tokens** → **Certificate connectors** → **Add**), copy it to `U01PARVMNDS01`, and run
it with an account that has local administrator rights on the server.

During setup:

```text
Service account : SYSTEM
```

`SYSTEM` is Microsoft's default and keeps this lab consistent with the least-privilege pattern already
applied to the NDES role itself — the Connector doesn't need a domain identity of its own when the
underlying NDES service (`gmsa-ndes$`) already carries the CA permissions from §9.

Sign in with the Intune Administrator account from §5 when prompted, and confirm the connector is
scoped to SCEP (certificate revocation support is enabled by default and should stay on, matching §9's
grant).

## 12. Verify the connector is healthy — in the Intune admin center

```text
Tenant administration → Connectors and tokens → Certificate connectors
    → U01PARVMNDS01 : Active
```

**Active** confirms the Connector has completed its first successful check-in over the outbound HTTPS
path from §5's prerequisites — not that a certificate has been issued. Give it a few minutes and refresh
the page if it still shows the initial provisioning state.

## 13. Expected final state

- `Intune SCEP Mobile User` exists on `U01PARVMPKI02`, issued, with the ACL from §6 — no broader
  `Enroll` grant left over from `Web Server`.
- `U01PARVMNDS01` runs the NDES role under `gmsa-ndes$` — not the Application Pool Identity from §7 —
  registry-mapped to `IntuneSCEPMobileUser` (§8), confirmed answering over HTTP (§10).
- `gmsa-ndes$` holds **Issue and Manage Certificates** on `U01PARVMPKI02` (§9) and **Read** on both RA
  certificates' private keys (§10).
- The Certificate Connector on `U01PARVMNDS01` shows **Active** in the Intune admin center.

**What this does not yet prove:** no device, inside the lab network or outside it, has requested a
certificate through this path. `U01PARVMNDS01` isn't reachable from outside the lab yet (§3), and no
Intune SCEP profile exists to target a device at it — both are later work (§15).

## 14. Update the infrastructure inventory

Per `reference/naming-conventions.md` §2, update `reference/vm-inventory.md`'s `U01PARVMNDS01` row:
change `Status` from `Planned` to `Running`, and replace the note with something like "NDES role +
Intune Certificate Connector configured, issuing from `Intune SCEP Mobile User` on `U01PARVMPKI02`;
not yet reachable from outside the lab network — see `U00PARVMPNC01`."

## 15. Next step — Entra Private Network Connector + Application Proxy

`U01PARVMNDS01` is fully configured but unreachable from anywhere a real mobile device would be. The
next lab builds `U00PARVMPNC01`, joins it to Entra Private Network Connector, and publishes this NDES
endpoint through Entra Application Proxy with Passthrough pre-authentication — the only pre-auth mode
SCEP's own protocol allows, not a weaker choice made for this lab. Only after that lab does a real
device have a network path to the template and Connector built here.

## 16. References

- [Configure infrastructure to support SCEP certificate profiles with Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/intune/fundamentals/certificates/scep-infrastructure)
- [Install the Certificate Connector for Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/intune/fundamentals/certificates/connector/setup-connector)
- [Use SCEP certificate profiles with Microsoft Intune — Microsoft Learn](https://learn.microsoft.com/en-us/intune/device-configuration/certificates/scep-profiles)
- [Network Device Enrollment Service (NDES) role — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/networking/core-network-guide/cncg/server-certs/install-the-active-directory-certificate-services-role-services)
- [Getting Started with Group Managed Service Accounts — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/getting-started-with-group-managed-service-accounts)
- [Setting up NDES using a Group Managed Service Account (gMSA) — Microsoft Learn archive](https://learn.microsoft.com/en-us/archive/blogs/pki/setting-up-ndes-using-a-group-managed-service-account-gmsa)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
