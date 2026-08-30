# Deploy an AD CS PKI

This runbook deploys a Microsoft Active Directory Certificate Services (AD CS) infrastructure with three PKI-role servers: an offline Standalone Root CA, an online Enterprise Issuing CA, and an independent HTTP CRL/AIA Web Distribution Point. The environment also uses `U01PARVMDOM01` for AD DS/DNS and `U01PARVMADM01` as the dedicated administration and clean PKI validation client. Full CRLs only, no Delta CRL, one standardized file convention across the three PKI-role servers, and a real leaf-certificate test before the PKI is considered complete.

## 1. Architecture

```text
                                 Internet
                                    |
                                 Freebox
                              192.168.1.254
                                    |
                       Home Wi-Fi  192.168.1.0/24
                                    |
             +----------------------+----------------------+
             |                                              |
            PC1                                            PC2
     U01PARVMFWL01 (OpenWrt)                        U01PARVMFWL02 (OpenWrt)
     WAN  192.168.1.250/24                           WAN  192.168.1.251/24
     LAN  192.168.20.126/25  <--- inter-site link --->  LAN  192.168.20.254/25
             |                                              |
   Subnet 1 : 192.168.20.0/25                    Subnet 2 : 192.168.20.128/25
             |                                              |
       +-----+-----+                        +---------------+---------------+
       |           |                        |               |               |
U01PARVMDOM01  U01PARVMADM01          U01PARVMPKI01    U01PARVMPKI02    U01PARVMWEB01
AD DS / DNS    Admin / validation     Root CA (offline) Issuing CA      Web Dist. (CRL/AIA)
192.168.20.41/25  192.168.20.45/25   192.168.20.142/25  192.168.20.143/25  192.168.20.144/25

Certificate trust and publication flow:

U01PARVMPKI01 --- signs ---> U01PARVMPKI02 --- issues ---> Users / Computers / Servers
                                    |
                                    v  CRL + certificate
                              U01PARVMWEB01 --- HTTP ---> pki.corp.unreadlines.com
```

PKI01/PKI02/WEB01 sit on Subnet 2 behind `U01PARVMFWL02` (PC2); DOM01/ADM01 sit on Subnet 1 behind `U01PARVMFWL01` (PC1). All cross-subnet traffic (domain join, GPO, AD DNS lookups, SMB CRL publication) rides the inter-site link between the two OpenWrt routers documented in `openwrt-wireguard-site-to-site/README.md`.

## 2. Servers and roles

| Server | Role | IPv4 | Subnet | Gateway |
| --- | --- | --- | --- | --- |
| `U01PARVMPKI01` | Offline Standalone Root CA | `192.168.20.142/25` | Subnet 2 (`192.168.20.128/25`) | `192.168.20.254` |
| `U01PARVMPKI02` | Online Enterprise Issuing CA | `192.168.20.143/25` | Subnet 2 (`192.168.20.128/25`) | `192.168.20.254` |
| `U01PARVMWEB01` | HTTP CRL/AIA Web Distribution Point | `192.168.20.144/25` | Subnet 2 (`192.168.20.128/25`) | `192.168.20.254` |
| `U01PARVMDOM01` | AD DS / DNS | `192.168.20.41/25` | Subnet 1 (`192.168.20.0/25`) | `192.168.20.126` |
| `U01PARVMADM01` | Administration / clean PKI validation client | `192.168.20.45/25` | Subnet 1 (`192.168.20.0/25`) | `192.168.20.126` |

The three PKI-role servers (`PKI01`, `PKI02`, `WEB01`) sit on Subnet 2, `U01PARVMDOM01` and `U01PARVMADM01` on Subnet 1 — matching the two-router OpenWrt lab split (`openwrt-wireguard-site-to-site/README.md`). Traffic between the two subnets crosses that lab's inter-site link.

```text
Domain:                    corp.unreadlines.com
PKI publication hostname:  pki.corp.unreadlines.com
```

```text
U01PARVMPKI01 (Root CA, offline, not domain-joined)
        |
        | signs
        v
U01PARVMPKI02 (Issuing CA, online, domain-joined)
        |
        | issues
        v
Users / Computers / Servers / Services
```

`U01PARVMPKI01` stays outside the domain and is normally powered off. `U01PARVMPKI02` is domain-joined and stays online. `U01PARVMWEB01` is domain-joined but is not a certification authority.

## 3. Scope and dependencies

This runbook builds a complete two-tier PKI: an offline Standalone Root CA (`U01PARVMPKI01`), an online
Enterprise Issuing CA (`U01PARVMPKI02`), and an independent HTTP CRL/AIA distribution point
(`U01PARVMWEB01`). It is finished only once a real leaf certificate has been issued and its full chain
and revocation checking validated from a clean client.

It stops deliberately there. Production certificate templates, permissions and autoenrolment are **out
of scope** — the first of them is published in `nps-server-certificate-deployment/README.md`, which
issues the Server Authentication certificate NPS needs. Intune, SCEP and NDES work is later still.

It assumes `active-directory-domain-controller/README.md` has produced a working `corp.unreadlines.com`
forest, and that the two-subnet topology from `openwrt-wireguard-site-to-site/README.md` is up: the PKI
servers sit in Subnet 2 and the domain controller in Subnet 1, so every domain operation here crosses
the inter-site link.

## 4. Prerequisites

- `U01PARVMDOM01` up, with `corp.unreadlines.com` resolving — `active-directory-domain-controller/README.md`.
- Routing between Subnet 1 and Subnet 2 verified — `openwrt-wireguard-site-to-site/README.md` §22.
- Three Windows Server VMs prepared from `prepare-windows-ubuntu-templates/README.md`, with the static
  addresses listed in §2 reserved and free.
- `U01PARVMADM01` available as a **clean** validation client: it must never have had the Root
  certificate installed by hand, or the Phase 7 test proves nothing.
- Local Administrator on each of the three PKI servers, and Domain Admin rights for the GPO work in
  Phase 6.

## 5. Security rules

- Never transfer the Root CA private key.
- Never place a PFX/P12 file, a CA backup, or an AD CS database on `U01PARVMWEB01`.
- `U01PARVMWEB01` holds only public CA certificates and CRLs — nothing else.
- Keep the Root CA powered off and disconnected except during controlled operations.
- Never rename a CA after installation.
- Never rename the native files inside `CertEnroll`.
- Do not publish production certificate templates or enable auto-enrollment before the final validation (Checkpoint 2).
- Only a temporary, strictly controlled validation template may be used before Checkpoint 2 — see Phase 7.

These rules are not repeated elsewhere in this document — apply them throughout.

## 6. Root CA power and access discipline

The standard `mRemoteNG` connection model used for `U01PARVMPKI02` and `U01PARVMWEB01` should not be read as "the Root CA stays reachable over the network." For `U01PARVMPKI01`:

- Prefer the hypervisor/VM console for administration.
- RDP/mRemoteNG is acceptable only temporarily, during initial deployment or a controlled maintenance operation (Phase 1, Phase 3, Phase 4's signing step).
- Before signing a certificate, publishing a CRL, or renewing anything, check the clock — a wrong clock on an offline VM that's rarely powered on can silently issue a certificate with a bad `NotBefore` or reject a legitimate request:
  ```powershell
  Get-Date
  w32tm /query /status
  ```
  `U01PARVMPKI01` is offline and not domain-joined, so `w32tm /query /status` may correctly report it as unsynchronized — that's expected, not a fault to fix. It's shown for context only. The actual check is comparing `Get-Date`'s output against a trusted external time source (a phone, another server you trust, an internet time reference) — that comparison is what proves the clock is close enough to trust, not the `w32tm` sync state.
- Once the operation is done and the required public files have been transferred off it, disconnect or disable its network connectivity, then power it off.

```text
Controlled Root CA operation
        v
verify required public files were transferred
        v
disconnect / disable Root CA network connectivity
        v
power off U01PARVMPKI01
```

`U01PARVMPKI02` and `U01PARVMWEB01` continue to be administered normally through `mRemoteNG`. This is not repeated at the end of every phase that touches `U01PARVMPKI01` — apply it after Phase 1, Phase 3, and the signing step of Phase 4.

## 7. File and publication conventions

```text
1. AD CS native output              C:\Windows\System32\CertSrv\CertEnroll\
        |  copy, rename to the standard name below
        v
2. Staging / transfer               C:\UnreadLines\
        |  transfer to the destination server
        v
3. HTTP publication (WEB01 only)    C:\inetpub\wwwroot\pki\
        |
        v
   http://pki.corp.unreadlines.com/
```

**Level 1 — `C:\Windows\System32\CertSrv\CertEnroll\`.** AD CS creates and manages its own files here. Never rename or modify the originals.

**Level 2 — `C:\UnreadLines\`.** Staging and transfer only — non-secret, transferable files, the working directory for certificate requests, manual transfers, and validation files:

```text
C:\UnreadLines\
    UnreadLinesRootCA.crt
    UnreadLinesRootCA.crl
    UnreadLinesIssuingCA.req
    UnreadLinesIssuingCA.crt
    PKIValidation.crt
```

The Root CA's CRL is staged here for its manual transfer to `U01PARVMWEB01` (Phase 3). The Issuing CA's CRL is the one exception — it never passes through this directory; AD CS publishes it directly from `CertEnroll` onto `U01PARVMWEB01` (Phase 5).

**Level 3 — `C:\inetpub\wwwroot\pki\`.** The final publication directory on `U01PARVMWEB01`, served by a dedicated IIS site. At first deployment:

```text
C:\inetpub\wwwroot\pki\
    UnreadLinesRootCA.crt
    UnreadLinesRootCA.crl
    UnreadLinesIssuingCA.crt
    UnreadLinesIssuingCA.crl
```

**Backup — kept out of `C:\UnreadLines` entirely.** `C:\UnreadLines` is a staging/transfer area for non-secret files. The Root CA backup contains the private key and must never sit next to it. Use a separate location:

```text
C:\PKI-Backup\
= sensitive data — private key, CA database

C:\UnreadLines\
= public, transferable files only
```

Copy the contents of `C:\PKI-Backup` to protected, offline storage immediately after each backup.

**Certificate extension convention.** `.crt` is the standard extension for CA certificates at the staging and publication levels. A Windows export command may initially produce a `.cer` file — `.cer` and `.crt` can both hold the same X.509 DER content, so this runbook writes certificate exports directly under their `.crt` staging name rather than exporting once and renaming afterward.

## 8. Initial manual preparation — standard procedure

Applies the first time you touch `U01PARVMPKI01`, `U01PARVMPKI02`, `U01PARVMWEB01`, or `U01PARVMADM01`:

```text
Initial Manual Preparation
        v
Verify Hostname and Network Configuration
        v
mRemoteNG
        v
Elevated PowerShell
        v
Role / Domain / PKI configuration
```

Configure manually, in the server console:

1. The final IPv4 address.
2. The subnet mask `/25` (the server's actual subnet — Subnet 1 or Subnet 2, see Architecture).
3. The default gateway (`192.168.20.126` on Subnet 1, `192.168.20.254` on Subnet 2).
4. The DNS server appropriate to the server's role.
5. The final hostname.
6. Remote Desktop.

Then connect with mRemoteNG and continue in an elevated PowerShell session. Each phase below states this server's specific values.

If the hostname wasn't set manually before connecting, `Rename-Computer` is the fallback — never the default path:

```powershell
Rename-Computer -NewName "<TargetHostname>" -Restart
```

**Standard Network Verification** — run after manual preparation and before domain join, AD CS installation, IIS installation, or anything else that depends on the server's identity:

```powershell
$ActiveInterface = Get-NetAdapter |
    Where-Object Status -eq "Up" |
    Select-Object -First 1

$Configuration = Get-NetIPConfiguration `
    -InterfaceIndex $ActiveInterface.ifIndex

[PSCustomObject]@{
    ComputerName = $env:COMPUTERNAME
    Interface    = $ActiveInterface.Name
    IPv4Address  = ($Configuration.IPv4Address.IPAddress -join ", ")
    PrefixLength = ($Configuration.IPv4Address.PrefixLength -join ", ")
    Gateway      = ($Configuration.IPv4DefaultGateway.NextHop -join ", ")
    DnsServers   = ($Configuration.DnsServer.ServerAddresses -join ", ")
}
```

Compare against the phase's Target Configuration. Do not continue until every value matches.

## 9. Phase overview

| Phase | Server | Content |
| --- | --- | --- |
| 1 | `U01PARVMPKI01` | Install the Root CA |
| 2 | `U01PARVMWEB01` (+ `U01PARVMDOM01`) | Prepare the Web Distribution Point |
| 3 | `U01PARVMPKI01` → `U01PARVMWEB01` | Root CA CDP/AIA, generate and publish the Full CRL |
| — | — | **Checkpoint 1** — Root CA publication |
| 4 | `U01PARVMPKI02` → `U01PARVMPKI01` → `U01PARVMPKI02` | Install the Issuing CA |
| 5 | `U01PARVMPKI02` → `U01PARVMWEB01` | Issuing CA CDP/AIA, generate and publish the Full CRL |
| 6 | `U01PARVMDOM01` (+ `U01PARVMADM01`) | Deploy Root trust domain-wide via GPO |
| 7 | `U01PARVMPKI02` (+ `U01PARVMADM01`) | Leaf certificate validation |
| — | — | **Checkpoint 2** — PKI complete |

```text
Manual preparation
        v
Hostname / Network verification
        v
CAPolicy.inf
        v
CA / IIS installation
        v
CDP / AIA configuration
        v
AD CS native output — C:\Windows\System32\CertSrv\CertEnroll
        v
Staging — C:\UnreadLines
        v
Transfer
        v
HTTP repository — C:\inetpub\wwwroot\pki
        v
http://pki.corp.unreadlines.com/
        v
GPO Root Trust
        v
Leaf certificate validation
        v
Checkpoint 2 — PKI ready
```

---

## 10. Phase 1 — Root CA

**Server:** `U01PARVMPKI01`

### 10.1 Target configuration

```text
ComputerName : U01PARVMPKI01
IPv4Address  : 192.168.20.142
PrefixLength : 25
Gateway      : 192.168.20.254
DnsServers   : none — not domain-joined, do not assign the AD DNS server
```

| Item | Value |
| --- | --- |
| CA type | Standalone Root CA |
| CA common name | `UnreadLines Root CA` |
| Key provider | Microsoft Software Key Storage Provider |
| Key length | 4096 bits |
| Hash algorithm | SHA-256 |
| Certificate validity | 20 years |
| Full CRL period | 26 weeks |

### 10.2 Initial manual preparation

Follow the standard procedure above using the Target Configuration values here. `U01PARVMPKI01` stays outside the domain.

### 10.3 Verify hostname and network configuration

Run the Standard Network Verification script and confirm it matches the Target Configuration above.

### 10.4 Create `CAPolicy.inf`

Run this in an elevated PowerShell session on `U01PARVMPKI01`, before installing the AD CS role. The command below writes `C:\Windows\CAPolicy.inf` directly — no manual Notepad step is needed. AD CS reads this file only if it already exists in `%SystemRoot%` at the moment the CA role is installed, so it must be created first:

```powershell
@'
[Version]
Signature="$Windows NT$"

[Certsrv_Server]
RenewalKeyLength=4096
RenewalValidityPeriod=Years
RenewalValidityPeriodUnits=20
CRLPeriod=Weeks
CRLPeriodUnits=26
CRLDeltaPeriod=Days
CRLDeltaPeriodUnits=0

[CRLDistributionPoint]

[AuthorityInformationAccess]
'@ | Set-Content -Path "C:\Windows\CAPolicy.inf" -Encoding ASCII
```

Confirm the file exists and contains exactly this content before continuing:

```powershell
Get-Item "C:\Windows\CAPolicy.inf"
Get-Content "C:\Windows\CAPolicy.inf"
```

`CRLDeltaPeriodUnits=0` explicitly disables Delta CRL publication — this Root CA only ever produces a Full CRL. The `[CRLDistributionPoint]` and `[AuthorityInformationAccess]` sections are intentionally empty, so the Root CA's own self-signed certificate carries no CDP/AIA extensions — a root has nothing to chain to, and a self-referencing CDP/AIA on a trust anchor is meaningless. This does **not** remove CDP/AIA from the PKI as a whole:

```text
Root CA certificate       CDP: none              AIA: none
Issuing CA certificate    CDP: Root CA CRL        AIA: Root CA certificate
Leaf certificate          CDP: Issuing CA CRL     AIA: Issuing CA certificate
```

### 10.5 Install AD CS role and the standalone Root CA

```powershell
Install-WindowsFeature `
    -Name ADCS-Cert-Authority `
    -IncludeManagementTools
```

A server restart is not normally required here. Check the output — `RestartNeeded` is the relevant field:

- `Restart Needed = No` → continue directly.
- `Restart Needed = Yes` → restart the server, then reconnect before continuing.

Do not force a restart when `Restart Needed = No`.

Only then install the Root CA itself:

```powershell
Install-AdcsCertificationAuthority `
    -CAType StandaloneRootCA `
    -CACommonName "UnreadLines Root CA" `
    -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" `
    -KeyLength 4096 `
    -HashAlgorithmName SHA256 `
    -ValidityPeriod Years `
    -ValidityPeriodUnits 20 `
    -Force
```

### 10.6 Enable AD CS auditing

`U01PARVMPKI01` isn't domain-joined, so the Advanced Audit Policy is set locally rather than through a GPO — `AuditFilter` alone isn't enough without it:

```powershell
auditpol /set /subcategory:"Certification Services" /success:enable /failure:enable

certutil -setreg CA\AuditFilter 127

Restart-Service CertSvc
```

Verify both settings now, while the CA is still up — `U01PARVMPKI01` is about to be powered off, so this is the point to confirm and record the audit state, not later:

```powershell
auditpol /get /subcategory:"Certification Services"

certutil -getreg CA\AuditFilter
```

Expect `Certification Services    Success and Failure` and `AuditFilter    127`.

### 10.7 Stage and back up

```powershell
New-Item -Path "C:\UnreadLines" -ItemType Directory -Force
```

Export the Root CA certificate directly under its staging name:

```powershell
$cert = Get-ChildItem `
    -Path "Cert:\LocalMachine\My", "Cert:\LocalMachine\CA" |
    Where-Object Subject -like "*UnreadLines Root CA*" |
    Select-Object -First 1

Export-Certificate `
    -Cert $cert `
    -FilePath "C:\UnreadLines\UnreadLinesRootCA.crt" `
    -Type CERT
```

Capture and record its thumbprint now — this is the reference thumbprint used to verify every later transfer of this certificate, including the comparison in Phase 6:

```powershell
$RootCert = Get-PfxCertificate `
    -FilePath "C:\UnreadLines\UnreadLinesRootCA.crt"

$RootCert |
    Select-Object Subject, Issuer, Thumbprint, NotBefore, NotAfter
```

Record the `Thumbprint` value in the deployment record.

Back up the CA to its own, separate, password-protected location:

```powershell
$BackupPassword = Read-Host -Prompt "CA backup password" -AsSecureString

New-Item -Path "C:\PKI-Backup" -ItemType Directory -Force

Backup-CARoleService `
    -Path "C:\PKI-Backup" `
    -Password $BackupPassword `
    -Force

reg export `
    "HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration" `
    "C:\PKI-Backup\CA-Configuration.reg" `
    /y

Copy-Item `
    "C:\Windows\CAPolicy.inf" `
    "C:\PKI-Backup\CAPolicy.inf" `
    -Force
```

Copy `C:\PKI-Backup` to protected offline storage now. Repeat this same backup after Phase 4's signing step, once the Issuing CA's certificate has been issued — a Root CA backup is worth taking again right after the operation that used it.

Store the backup password separately from the backup itself — anyone who obtains both can restore the Root CA's private key. Once the offline copy is verified, remove the local `C:\PKI-Backup` staging copy from `U01PARVMPKI01` as operational policy allows; the VM stays powered off most of the time, but the backup shouldn't sit indefinitely on its system disk either.

### 10.8 Verification

```powershell
Get-Service CertSvc

certutil -config "localhost\UnreadLines Root CA" -ping
```

Confirm `C:\UnreadLines\UnreadLinesRootCA.crt` exists and that `C:\PKI-Backup` has been copied off the VM.

This phase's controlled operation on `U01PARVMPKI01` is now complete — apply the Root CA Power and Access Discipline sequence from the top of this document:

```text
verify protected backup copy
        v
disconnect / disable network connectivity
        v
power off U01PARVMPKI01
```

`U01PARVMPKI01` comes back on for its next controlled operation — configuring CDP/AIA in Phase 3.

**Next:** Phase 2, on `U01PARVMWEB01` — independent of this phase's timing.

---

## 11. Phase 2 — Web distribution point

**Server:** `U01PARVMWEB01`, with a short step on `U01PARVMDOM01`.

### 11.1 Target configuration

```text
ComputerName : U01PARVMWEB01
IPv4Address  : 192.168.20.144
PrefixLength : 25
Gateway      : 192.168.20.254
DnsServers   : 192.168.20.41
```

### 11.2 Initial manual preparation

Follow the standard procedure above using the Target Configuration values here.

### 11.3 Verify hostname and network configuration

Run the Standard Network Verification script and confirm it matches the Target Configuration above. Confirm domain resolution before joining:

```powershell
Resolve-DnsName -Name "corp.unreadlines.com" -Server "192.168.20.41"
```

### 11.4 Domain join

```powershell
$Credential = Get-Credential

Add-Computer `
    -DomainName "corp.unreadlines.com" `
    -Credential $Credential `
    -Restart
```

After the restart, reconnect to `U01PARVMWEB01` with a domain account that has the required local administrative rights. Do not continue the post-domain-join configuration using the local account that was used before the domain join.

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
Name                       = U01PARVMWEB01
Domain                     = corp.unreadlines.com
PartOfDomain               = True
Test-ComputerSecureChannel = True
```

Do not continue to IIS installation until these checks are correct.

### 11.5 Install IIS and create the dedicated site

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

New-Item -Path "C:\inetpub\wwwroot\pki" -ItemType Directory -Force

Import-Module WebAdministration

New-Website `
    -Name "PKI Distribution" `
    -PhysicalPath "C:\inetpub\wwwroot\pki" `
    -Port 80 `
    -HostHeader "pki.corp.unreadlines.com"

Stop-Website -Name "Default Web Site"
```

A dedicated site means `http://pki.corp.unreadlines.com/UnreadLinesRootCA.crl` maps directly to `C:\inetpub\wwwroot\pki\UnreadLinesRootCA.crl` — no `/pki/` segment, no ambiguity with `Default Web Site`, which is stopped since it isn't used on this server.

### 11.6 Configure MIME types

If both mappings already exist with the correct MIME types, do not add them again — attempting to add an existing `fileExtension` causes IIS to return `Cannot add duplicate collection entry`. The block below checks each extension before adding it, so it's safe to run as-is regardless of what's already mapped:

```powershell
$Mappings = & "$env:SystemRoot\System32\inetsrv\appcmd.exe" `
    list config `
    /section:system.webServer/staticContent

if (-not ($Mappings -match 'fileExtension=\".crl\"')) {
    Add-WebConfigurationProperty `
        -PSPath "IIS:\Sites\PKI Distribution" `
        -Filter "system.webServer/staticContent" `
        -Name "." `
        -Value @{
            fileExtension = ".crl"
            mimeType      = "application/pkix-crl"
        }
}

if (-not ($Mappings -match 'fileExtension=\".crt\"')) {
    Add-WebConfigurationProperty `
        -PSPath "IIS:\Sites\PKI Distribution" `
        -Filter "system.webServer/staticContent" `
        -Name "." `
        -Value @{
            fileExtension = ".crt"
            mimeType      = "application/x-x509-ca-cert"
        }
}
```

Confirm both mappings are present:

```powershell
& "$env:SystemRoot\System32\inetsrv\appcmd.exe" `
    list config `
    /section:system.webServer/staticContent |
    Select-String -Pattern '\.crl|\.crt'
```

Expected output:

```text
<mimeMap fileExtension=".crl" mimeType="application/pkix-crl" />
<mimeMap fileExtension=".crt" mimeType="application/x-x509-ca-cert" />
```

The final HTTP verification for each file still expects `.crl → 200 / application/pkix-crl` and `.crt → 200 / application/x-x509-ca-cert`.

### 11.7 Create the DNS record — on `U01PARVMDOM01`

```powershell
Add-DnsServerResourceRecordA `
    -ZoneName "corp.unreadlines.com" `
    -Name "pki" `
    -IPv4Address "192.168.20.144" `
    -TimeToLive 01:00:00
```

Verify resolution before testing anything over HTTP — a `health.txt` request against a name that doesn't resolve yet proves nothing:

```powershell
Resolve-DnsName -Name "pki.corp.unreadlines.com" -Server "192.168.20.41"
```

### 11.8 Test and remove `health.txt`

Run this section on `U01PARVMWEB01` — the physical file is created and removed there, even if the `Invoke-WebRequest` call itself is optionally repeated from `U01PARVMDOM01` or another domain member:

```powershell
"PKI publication test" | Set-Content "C:\inetpub\wwwroot\pki\health.txt"

Invoke-WebRequest -Uri "http://pki.corp.unreadlines.com/health.txt" -UseBasicParsing
```

Expect HTTP `200` with the body `PKI publication test`. Once confirmed, remove the file — it has no place in the finished repository:

```powershell
Remove-Item -Path "C:\inetpub\wwwroot\pki\health.txt"
```

**Next:** Phase 3, on `U01PARVMPKI01`. `U01PARVMWEB01` now resolves and serves HTTP on its dedicated site — the Root CA's CDP/AIA extensions can safely point at it.

---

## 12. Phase 3 — Root CA CDP/AIA and publication

**Server:** `U01PARVMPKI01`, then transfer to `U01PARVMWEB01`. Requires Phase 2 complete — configure CDP/AIA before signing the Issuing CA's request, since certificates already issued keep the URLs that were active at signing time.

The Root CA certificate itself intentionally contains no CDP or AIA extensions (Phase 1's `CAPolicy.inf`). The CDP/AIA configuration performed in this phase does not modify the Root CA certificate itself — it controls the CDP/AIA extensions placed in certificates *issued by* the Root CA, especially the `UnreadLines Issuing CA` certificate. This configuration must be completed before the Issuing CA's request is signed in Phase 4. Do not skip this phase.

Start the controlled Root CA operation:

1. Power on `U01PARVMPKI01`.
2. Use the hypervisor console, or temporarily restore the approved management connectivity.
3. Verify the Root CA clock before continuing:

```powershell
Get-Date
w32tm /query /status
```

### 12.1 Configure CDP and AIA

```powershell
certsrv.msc
```

Right-click `UnreadLines Root CA` → **Properties** → **Extensions**.

Select **CRL Distribution Point (CDP)**. Remove the default entries. Don't check `Publish Delta CRLs to this location` on either entry below — `CAPolicy.inf` already disabled Delta CRL generation, and the checkbox should stay off to match. Keep the local path enabled and excluded from the issued-certificate extension:

```text
C:\Windows\System32\CertSrv\CertEnroll\<CaName><CRLNameSuffix>.crl
Publish CRLs to this location                : enabled
Include in the CDP extension of issued certs : disabled
```

Add the HTTP location — type it exactly as shown, including the literal `<CRLNameSuffix>` token:

```text
http://pki.corp.unreadlines.com/UnreadLinesRootCA<CRLNameSuffix>.crl
Publish CRLs to this location                : disabled
Include in the CDP extension of issued certs : enabled
```

Select **Authority Information Access (AIA)**. Remove the default entries. Unlike the CDP tab, the AIA tab has no "publish to this location" checkbox — each location only exposes **Include in the AIA extension of issued certificates** and **Include in the online certificate status protocol (OCSP) extension**. Keep the local path present but excluded from both:

```text
C:\Windows\System32\CertSrv\CertEnroll\<ServerDNSName>_<CaName><CertificateName>.crt
Include in the AIA extension of issued certificates : disabled
Include in the OCSP extension                       : disabled
```

Add:

```text
http://pki.corp.unreadlines.com/UnreadLinesRootCA<CertificateName>.crt
Include in the AIA extension of issued certificates : enabled
Include in the OCSP extension                       : disabled
```

Click **Apply**, then **OK**. Restart `CertSvc` before doing anything else — AD CS doesn't pick up CDP/AIA extension changes until the service restarts, and the Issuing CA's certificate is about to be signed with whatever is active right now:

```powershell
Restart-Service CertSvc
```

> **Why `<CRLNameSuffix>` and `<CertificateName>` stay in the URL.** At first deployment both macros expand to nothing, so the published files are simply `UnreadLinesRootCA.crl` and `UnreadLinesRootCA.crt`. They earn their place at the next CA certificate renewal: `<CertificateName>` lets AD CS distinguish generations of the CA's own certificate, and `<CRLNameSuffix>` distinguishes the CRLs tied to each generation's key. Keeping the macros now means a future renewal doesn't require re-touching this configuration or overwriting a generation still referenced by certificates already issued.

### 12.2 Verify the configuration

```powershell
certutil -getreg CA\CRLPublicationURLs
certutil -getreg CA\CACertPublicationURLs
```

### 12.3 Generate and stage the CRL

```powershell
certutil -crl
```

```powershell
Get-ChildItem -Path "C:\Windows\System32\CertSrv\CertEnroll" -Filter "*.crl"

Copy-Item `
    -Path "C:\Windows\System32\CertSrv\CertEnroll\UnreadLines Root CA.crl" `
    -Destination "C:\UnreadLines\UnreadLinesRootCA.crl" `
    -Force
```

Adjust the source filename above to match what `Get-ChildItem` actually returned.

### 12.4 Transfer to `U01PARVMWEB01` and `U01PARVMDOM01`

On `U01PARVMPKI01`, confirm both files exist before transferring anything:

```powershell
Get-Item `
    "C:\UnreadLines\UnreadLinesRootCA.crt", `
    "C:\UnreadLines\UnreadLinesRootCA.crl"
```

**`U01PARVMPKI01` → `U01PARVMWEB01`.** Copy:

```text
C:\UnreadLines\UnreadLinesRootCA.crt
C:\UnreadLines\UnreadLinesRootCA.crl
```

to `U01PARVMWEB01:`

```text
C:\inetpub\wwwroot\pki\UnreadLinesRootCA.crt
C:\inetpub\wwwroot\pki\UnreadLinesRootCA.crl
```

On `U01PARVMWEB01`, confirm both files landed:

```powershell
Get-Item `
    "C:\inetpub\wwwroot\pki\UnreadLinesRootCA.crt", `
    "C:\inetpub\wwwroot\pki\UnreadLinesRootCA.crl"
```

**`U01PARVMPKI01` → `U01PARVMDOM01`.** While `U01PARVMPKI01` is still powered on for this phase, also stage a copy of the certificate — **not the CRL** — on `U01PARVMDOM01` for Phase 6's GPO import. The Root CA is normally powered off right after this phase, and deploying Root trust shouldn't require powering it back on just to fetch a public certificate that's already been exported.

On `U01PARVMDOM01`:

```powershell
New-Item -Path "C:\UnreadLines" -ItemType Directory -Force
```

Copy only:

```text
U01PARVMPKI01:C:\UnreadLines\UnreadLinesRootCA.crt
```

to:

```text
U01PARVMDOM01:C:\UnreadLines\UnreadLinesRootCA.crt
```

On `U01PARVMDOM01`, confirm it landed:

```powershell
Get-Item "C:\UnreadLines\UnreadLinesRootCA.crt"
```

Do not copy the Root CRL to `U01PARVMDOM01` — it has no use there. Do not power off `U01PARVMPKI01` until both transfers above have been completed and verified.

### 12.5 Verification

Run these from `U01PARVMWEB01` or from a domain member using Active Directory DNS — not from `U01PARVMPKI01`. Do not configure permanent AD DNS on `U01PARVMPKI01` just to run this test; it stays without a permanent AD DNS server per its Target Configuration.

```powershell
$Response = Invoke-WebRequest `
    -Uri "http://pki.corp.unreadlines.com/UnreadLinesRootCA.crl" `
    -UseBasicParsing -ErrorAction Stop
$Response.StatusCode
$Response.Headers["Content-Type"]

$Response = Invoke-WebRequest `
    -Uri "http://pki.corp.unreadlines.com/UnreadLinesRootCA.crt" `
    -UseBasicParsing -ErrorAction Stop
$Response.StatusCode
$Response.Headers["Content-Type"]
```

Expect:

```text
UnreadLinesRootCA.crl   → 200 / application/pkix-crl
UnreadLinesRootCA.crt   → 200 / application/x-x509-ca-cert
```

The Phase 3 Root CA operation is complete once:

1. The Root certificate and CRL transfers to `U01PARVMWEB01` and `U01PARVMDOM01` are verified.
2. The HTTP checks above both return `200`.

Only then disconnect / disable `U01PARVMPKI01`'s network connectivity and power it off. It comes back on for its next controlled operation — signing the Issuing CA's request in Phase 4.

**Next:** Checkpoint 1.

---

## 13. Checkpoint 1 — Root CA publication

Do not create the Issuing CA before this checkpoint is fully green:

- Root CA operational.
- Root certificate exported.
- Root backup completed and copied off the VM.
- Full Root CRL generated.
- Root CDP correctly configured.
- Root AIA correctly configured.
- `UnreadLinesRootCA.crl` reachable over HTTP, status `200`.
- `UnreadLinesRootCA.crt` reachable over HTTP, status `200`.
- Root CRL not expired.

---

## 14. Phase 4 — Issuing CA

**Server:** `U01PARVMPKI02`, with a signing step on `U01PARVMPKI01`.

### 14.1 Target configuration

```text
ComputerName : U01PARVMPKI02
IPv4Address  : 192.168.20.143
PrefixLength : 25
Gateway      : 192.168.20.254
DnsServers   : 192.168.20.41
```

| Item | Value |
| --- | --- |
| CA type | Enterprise Subordinate CA |
| CA common name | `UnreadLines Issuing CA` |
| Key provider | Microsoft Software Key Storage Provider |
| Key length | 4096 bits |
| Hash algorithm | SHA-256 |
| Certificate validity | 10 years — set by the Root CA at signing time |
| Full CRL period | 1 day |

### 14.2 Initial manual preparation

Follow the standard procedure above using the Target Configuration values here.

### 14.3 Verify hostname and network configuration

Run the Standard Network Verification script and confirm it matches the Target Configuration above. Confirm domain resolution before joining:

```powershell
Resolve-DnsName -Name "corp.unreadlines.com" -Server "192.168.20.41"
```

### 14.4 Domain join

```powershell
$Credential = Get-Credential

Add-Computer `
    -DomainName "corp.unreadlines.com" `
    -Credential $Credential `
    -Restart
```

Reconnect using the domain account intended to administer the Enterprise Issuing CA — a member of **Enterprise Admins** (or explicitly delegated the required AD CS rights). A local `Administrator` session is not sufficient.

Verify the join and the account before continuing:

```powershell
whoami

Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain

Test-ComputerSecureChannel -Verbose
```

Expect:

```text
whoami                     = UNREADLINES\<AdminAccount>
Name                       = U01PARVMPKI02
Domain                     = corp.unreadlines.com
PartOfDomain               = True
Test-ComputerSecureChannel = True
```

Do not continue to `CAPolicy.inf` / AD CS installation before this is verified.

### 14.5 Create `CAPolicy.inf`

Run this in an elevated PowerShell session on `U01PARVMPKI02`, before installing the AD CS role. The command below writes `C:\Windows\CAPolicy.inf` directly — no manual Notepad step is needed. AD CS reads this file only if it already exists in `%SystemRoot%` at the moment the CA role is installed, so it must be created first:

```powershell
@'
[Version]
Signature="$Windows NT$"

[Certsrv_Server]
RenewalKeyLength=4096
LoadDefaultTemplates=0
'@ | Set-Content -Path "C:\Windows\CAPolicy.inf" -Encoding ASCII
```

Confirm the file exists and contains exactly this content before continuing:

```powershell
Get-Item "C:\Windows\CAPolicy.inf"
Get-Content "C:\Windows\CAPolicy.inf"
```

`LoadDefaultTemplates=0` stops the Enterprise Issuing CA from automatically publishing the built-in default templates the moment it comes online — production templates are configured only after Checkpoint 2.

### 14.6 Install the AD CS role

```powershell
Install-WindowsFeature `
    -Name ADCS-Cert-Authority `
    -IncludeManagementTools
```

A server restart is not normally required here. Check the output — `RestartNeeded` is the relevant field:

- `Restart Needed = No` → continue directly.
- `Restart Needed = Yes` → restart the server before continuing.

Do not force a restart when `Restart Needed = No`.

### 14.7 Generate the Issuing CA certificate request

```powershell
New-Item -Path "C:\UnreadLines" -ItemType Directory -Force

Install-AdcsCertificationAuthority `
    -CAType EnterpriseSubordinateCA `
    -CACommonName "UnreadLines Issuing CA" `
    -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" `
    -KeyLength 4096 `
    -HashAlgorithmName SHA256 `
    -OutputCertRequestFile "C:\UnreadLines\UnreadLinesIssuingCA.req" `
    -Force
```

This reports the installation as incomplete — expected, since the request still needs to be signed by the Root CA. The private key stays on `U01PARVMPKI02`; only the request file leaves this server.

### 14.8 Sign the request — on `U01PARVMPKI01`

Start the controlled Root CA operation:

1. Power on `U01PARVMPKI01`.
2. Use the hypervisor console, or temporarily restore the approved management connectivity.
3. Verify the Root CA clock before continuing:

```powershell
Get-Date
w32tm /query /status
```

On `U01PARVMPKI01`, create the destination directory before transferring anything into it:

```powershell
New-Item -Path "C:\UnreadLines\Requests" -ItemType Directory -Force
```

Transfer only:

```text
U01PARVMPKI02:
    C:\UnreadLines\UnreadLinesIssuingCA.req
```

to:

```text
U01PARVMPKI01:
    C:\UnreadLines\Requests\UnreadLinesIssuingCA.req
```

Confirm the request file landed:

```powershell
Get-Item "C:\UnreadLines\Requests\UnreadLinesIssuingCA.req"
```

Confirm the Root CA's CDP/AIA configuration (Phase 3) is already final — the certificate signed here keeps whatever URLs are active at signing time.

Set the validity this Root CA will apply to the certificate it's about to sign:

```powershell
certutil -setreg CA\ValidityPeriodUnits 10
certutil -setreg CA\ValidityPeriod "Years"

Restart-Service CertSvc
```

Never set this longer than the Root CA's own remaining validity. Then submit the request:

```powershell
certreq `
    -submit `
    -config "localhost\UnreadLines Root CA" `
    "C:\UnreadLines\Requests\UnreadLinesIssuingCA.req" `
    "C:\UnreadLines\UnreadLinesIssuingCA.crt"
```

If `certreq -submit` returns:

```text
Certificate request is pending: Taken Under Submission
```

record the actual **Request ID** it returns — a Standalone Root CA doesn't auto-issue by default, so this is the expected path, not a failure.

On `U01PARVMPKI01`, approve it in the Certification Authority console:

```text
certsrv.msc
  → UnreadLines Root CA
    → Pending Requests
      → select the request
        → Right-click → All Tasks → Issue
```

Only then retrieve the certificate, using the actual numeric Request ID — for example, if the ID returned was `3`:

```powershell
certreq `
    -retrieve `
    -f `
    -config "localhost\UnreadLines Root CA" `
    3 `
    "C:\UnreadLines\UnreadLinesIssuingCA.crt"
```

Do not type `<REQUEST_ID>` literally in PowerShell — replace it with the actual numeric Request ID returned by `certreq -submit`.

Confirm the certificate was retrieved correctly:

```powershell
Get-Item "C:\UnreadLines\UnreadLinesIssuingCA.crt"

certutil -dump "C:\UnreadLines\UnreadLinesIssuingCA.crt"
```

Back up the Root CA again now that it has just signed a subordinate certificate, before disconnecting it. `C:\PKI-Backup` is a temporary staging location — the command below is safe whether the directory already exists or was removed after the previous backup:

```powershell
New-Item -Path "C:\PKI-Backup" -ItemType Directory -Force

$BackupPassword = Read-Host -Prompt "CA backup password" -AsSecureString

Backup-CARoleService `
    -Path "C:\PKI-Backup" `
    -Password $BackupPassword `
    -Force

reg export `
    "HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration" `
    "C:\PKI-Backup\CA-Configuration.reg" `
    /y

Copy-Item `
    "C:\Windows\CAPolicy.inf" `
    "C:\PKI-Backup\CAPolicy.inf" `
    -Force

Get-ChildItem "C:\PKI-Backup" -Recurse
```

On protected, offline storage, keep two distinct generations rather than overwriting the same folder:

```text
RootCA-Backups\
    01-Initial-Installation\
    02-After-IssuingCA-Signing\
```

`01-Initial-Installation` is the initial Root CA state (Phase 1). `02-After-IssuingCA-Signing` is the Root CA state after issuance of the `UnreadLines Issuing CA` certificate, and is the current recovery backup — do not overwrite the initial protected backup with it.

Keep the backup password separate from the backup. `C:\PKI-Backup` on `U01PARVMPKI01` is only temporary staging and may be removed once the protected offline copy has been verified.

On `U01PARVMPKI01`, confirm both files exist:

```powershell
Get-Item `
    "C:\UnreadLines\UnreadLinesIssuingCA.crt", `
    "C:\UnreadLines\UnreadLinesRootCA.crt"
```

Transfer these two files from `C:\UnreadLines` on `U01PARVMPKI01` to `C:\UnreadLines` on `U01PARVMPKI02`:

```text
UnreadLinesIssuingCA.crt
UnreadLinesRootCA.crt
```

On `U01PARVMPKI02`, confirm both landed:

```powershell
Get-Item `
    "C:\UnreadLines\UnreadLinesIssuingCA.crt", `
    "C:\UnreadLines\UnreadLinesRootCA.crt"
```

Verify the Root certificate against the reference thumbprint recorded in Phase 1 before trusting it:

```powershell
Get-PfxCertificate `
    -FilePath "C:\UnreadLines\UnreadLinesRootCA.crt" |
    Select-Object Subject, Issuer, Thumbprint, NotBefore, NotAfter
```

The Phase 4 Root CA operation is complete once:

1. The new Root backup (above) has been copied to protected offline storage.
2. Both certificate transfers to `U01PARVMPKI02` have been verified.

Only then disconnect / disable `U01PARVMPKI01`'s network connectivity and power it off.

### 14.9 Install on `U01PARVMPKI02`

```powershell
Import-Certificate `
    -FilePath "C:\UnreadLines\UnreadLinesRootCA.crt" `
    -CertStoreLocation "Cert:\LocalMachine\Root"
```

Only then install the signed certificate: `certsrv.msc` → right-click `UnreadLines Issuing CA` → **All Tasks** → **Install CA Certificate** → select `C:\UnreadLines\UnreadLinesIssuingCA.crt`.

After installing the certificate, `CertSvc` can remain stopped — start it explicitly and confirm:

```powershell
$CertSvc = Get-Service CertSvc
if ($CertSvc.Status -ne "Running") {
    Start-Service CertSvc
}
Get-Service CertSvc
```

Expect `Status = Running`.

### 14.10 Verification

```powershell
Get-Service CertSvc

certutil -ping

certutil -config "localhost\UnreadLines Issuing CA" -ping
```

The trust chain is now `UnreadLines Root CA` → `UnreadLines Issuing CA`.

**Next:** Phase 5, still on `U01PARVMPKI02`.

---

## 15. Phase 5 — Issuing CA CDP/AIA and publication

**Server:** `U01PARVMPKI02`, then transfer to `U01PARVMWEB01`.

### 15.1 Create the publication share — on `U01PARVMWEB01`

The Issuing CA's Full CRL publishes itself, natively, straight from AD CS into a dedicated share — no script, no scheduled task. Create that share before touching CDP. `CertSvc` runs as `LocalSystem` and accesses the remote SMB share using the `U01PARVMPKI02$` **computer account** — that's the identity presented on the network when it writes to a UNC CDP location.

On `U01PARVMWEB01`:

```text
C:\inetpub\wwwroot\pki
  → Properties → Sharing → Advanced Sharing → Share this folder
    Share name: PKIPublish$
```

**Share permissions.** Remove `Everyone` if present. Add the computer account:

```text
Permissions → Add
  Object Types → Computers
  Location     → corp.unreadlines.com
  Name         → U01PARVMPKI02
  Check Names
```

Then set:

```text
U01PARVMPKI02$
    Read          Allow
    Change        Allow
    Full Control  No

Administrators
    Full Control  Allow
```

**NTFS permissions.**

```text
C:\inetpub\wwwroot\pki
  → Properties → Security → Edit → Add
    Object Types → Computers
    → U01PARVMPKI02
```

Grant:

```text
Modify
Read & execute
List folder contents
Read
Write
Full Control  No
```

Do not remove the existing ACL entries for `SYSTEM`, `Administrators`, or IIS.

Result:

```text
\\U01PARVMWEB01\PKIPublish$
    Share: U01PARVMPKI02$ = Change + Read, Administrators = Full Control
    NTFS:  U01PARVMPKI02$ = Modify
```

Never add `U01PARVMPKI02$` to `Administrators`, `Domain Admins`, or any other privileged group on `U01PARVMWEB01`. If possible, restrict inbound TCP 445 on `U01PARVMWEB01` to `192.168.20.143` for this specific purpose. The real test of this configuration remains `certutil -crl` on `U01PARVMPKI02`, in "Verify the Configuration and Publish" below.

### 15.2 Configure CDP and AIA

Switch back to `U01PARVMPKI02`. Run the following on `U01PARVMPKI02`:

```powershell
certsrv.msc
```

Right-click `UnreadLines Issuing CA` → **Properties** → **Extensions** → **CRL Distribution Point (CDP)**. Remove the default entries, then configure three separate locations.

**A — local, native AD CS output:**

```text
C:\Windows\System32\CertSrv\CertEnroll\<CaName><CRLNameSuffix>.crl
Publish CRLs to this location                 : enabled
Include in the CDP extension of issued certs  : disabled
```

**B — UNC, publish-only, straight onto `U01PARVMWEB01`:**

```text
file://\\U01PARVMWEB01\PKIPublish$\UnreadLinesIssuingCA<CRLNameSuffix>.crl
Publish CRLs to this location                 : enabled
Include in the CDP extension of issued certs  : disabled
```

This entry exists only so AD CS can write the CRL onto `U01PARVMWEB01` — never check "Include in the CDP extension of issued certificates" for it; clients must never be told to fetch a CRL over UNC. It keeps `<CRLNameSuffix>` for the same renewal reason as Phase 3's macros.

**C — HTTP, client-facing:**

```text
http://pki.corp.unreadlines.com/UnreadLinesIssuingCA<CRLNameSuffix>.crl
Publish CRLs to this location                 : disabled
Include in the CDP extension of issued certs  : enabled
```

No Delta CRL anywhere, and no `<DeltaCRLAllowed>` in any of the three.

Select **Authority Information Access (AIA)**. Same pattern as Phase 3 — local and HTTP only. As on the Root CA, the AIA tab has no "publish to this location" checkbox, only **Include in the AIA extension of issued certificates** and **Include in the OCSP extension**:

```text
C:\Windows\System32\CertSrv\CertEnroll\<ServerDNSName>_<CaName><CertificateName>.crt
Include in the AIA extension of issued certificates : disabled
Include in the OCSP extension                       : disabled

http://pki.corp.unreadlines.com/UnreadLinesIssuingCA<CertificateName>.crt
Include in the AIA extension of issued certificates : enabled
Include in the OCSP extension                       : disabled
```

Do not create a UNC AIA entry — the UNC share is used only for CRL publication through CDP, never for AIA.

Click **Apply**, then **OK**.

### 15.3 Set the full CRL period

The Issuing CA revokes daily, so its Full CRL needs a short period — with a short overlap window so a client refreshing near expiry still gets a valid CRL:

```powershell
certutil -setreg CA\CRLPeriodUnits 1
certutil -setreg CA\CRLPeriod "Days"
certutil -setreg CA\CRLOverlapUnits 12
certutil -setreg CA\CRLOverlapPeriod "Hours"
certutil -setreg CA\CRLDeltaPeriodUnits 0

Restart-Service CertSvc

certutil -getreg CA\CRLDeltaPeriodUnits
```

Expect `CRLDeltaPeriodUnits = 0` — Delta CRL publication stays off, same as the Root CA. In `certsrv.msc`'s CDP dialog, don't check any `Publish Delta CRLs to this location` box on any of the three entries in the next step.

### 15.4 Verify the configuration and publish

```powershell
certutil -getreg CA\CRLPublicationURLs
certutil -getreg CA\CACertPublicationURLs
```

Confirm the publication share is reachable before relying on it — this distinguishes an ACL problem from a network/firewall problem if the next command fails:

```powershell
Test-NetConnection -ComputerName "U01PARVMWEB01" -Port 445
```

```powershell
certutil -crl
```

This single command writes the CRL locally **and** publishes it onto `U01PARVMWEB01` through the UNC location configured above — this first run tests the configuration end to end. AD CS automatically republishes the Base CRL again on its own once `CRLPeriod` elapses, no scheduled task involved; the 12-hour overlap isn't a second publication cycle, it just extends how long a CRL stays valid past the next scheduled publication, giving clients a safety margin if a republish is briefly late.

Confirm the file landed on `U01PARVMWEB01`:

```powershell
Get-Item "C:\inetpub\wwwroot\pki\UnreadLinesIssuingCA.crl" |
    Select-Object Name, Length, LastWriteTime
```

Record this `LastWriteTime` as the initial publication baseline — the Post-deployment Operational Check at the end of this document compares against it. Do not run `certutil -crl` again before that check; a further manual run would just reset the baseline you're trying to prove happens on its own.

Then from a domain member:

```powershell
$Response = Invoke-WebRequest `
    -Uri "http://pki.corp.unreadlines.com/UnreadLinesIssuingCA.crl" `
    -UseBasicParsing -ErrorAction Stop
$Response.StatusCode
$Response.Headers["Content-Type"]
```

Expect `200` / `application/pkix-crl`.

### 15.5 Transfer the certificate to `U01PARVMWEB01`

The CRL now publishes itself; the Issuing CA certificate still needs a one-time manual copy for its HTTP AIA location — it isn't part of the UNC CDP mechanism above. Transfer:

```text
U01PARVMPKI02:
    C:\UnreadLines\UnreadLinesIssuingCA.crt
```

to:

```text
U01PARVMWEB01:
    C:\inetpub\wwwroot\pki\UnreadLinesIssuingCA.crt
```

Verify it over HTTP from a domain member — the same symmetry as Phase 3's Root CA test:

```powershell
$Response = Invoke-WebRequest `
    -Uri "http://pki.corp.unreadlines.com/UnreadLinesIssuingCA.crt" `
    -UseBasicParsing -ErrorAction Stop
$Response.StatusCode
$Response.Headers["Content-Type"]
```

Expect `200` / `application/x-x509-ca-cert`.

### 15.6 Configure AD CS auditing — on `U01PARVMPKI02`

```powershell
certutil -setreg CA\AuditFilter 127

Restart-Service CertSvc
```

### 15.7 Create the audit GPO — on `U01PARVMDOM01`

Switch servers for this part — the `GroupPolicy` module isn't necessarily installed on `U01PARVMPKI02`. Use a GPO dedicated to this — do not fold it into `PKI - Trusted Root CA`.

On `U01PARVMDOM01`:

```text
gpmc.msc
  → Domains → corp.unreadlines.com
    → Create a GPO in this domain, and Link it here...
      Name: PKI - Issuing CA Audit
```

**Security Filtering.** Add `U01PARVMPKI02` (`Object Types → Computers`), then remove `Authenticated Users` from Security Filtering.

**Delegation tab.** Confirm (add if missing):

```text
Authenticated Users
    Read                Allow
    Apply Group Policy  No

U01PARVMPKI02
    Read                Allow
    Apply Group Policy  Allow
```

Never set `Deny` on `Authenticated Users` — leaving it at no `Apply Group Policy` right is sufficient and reversible.

Edit the GPO and configure:

```text
Computer Configuration
  -> Policies
    -> Windows Settings
      -> Security Settings
        -> Advanced Audit Policy Configuration
          -> Audit Policies
            -> Object Access
              -> Audit Certification Services
                 Success = Enabled
                 Failure = Enabled
```

Also enable the setting that makes Advanced Audit Policy subcategories authoritative, so a legacy category-level audit policy can't override them:

```text
Computer Configuration
  -> Policies
    -> Windows Settings
      -> Security Settings
        -> Local Policies
          -> Security Options
            -> Audit: Force audit policy subcategory settings
               (Windows Vista or later) to override audit policy
               category settings : Enabled
```

Creating the GPO isn't proof the policy is actually in effect on `U01PARVMPKI02` — apply and verify it there:

```powershell
# on U01PARVMPKI02
gpupdate /target:computer /force

gpresult /scope computer /r

auditpol /get /subcategory:"Certification Services"

certutil -getreg CA\AuditFilter
```

Expect:

```text
PKI - Issuing CA Audit  = Applied
Certification Services  = Success and Failure
AuditFilter              = 127
```

The GPO must apply only to `U01PARVMPKI02` — no other computer should show it as applied.

### 15.8 Back up the Issuing CA

Now that auditing and the audit GPO are in place, the Issuing CA is in its truly final configuration — back it up now, not earlier, so the registry export actually reflects `AuditFilter 127` and everything else configured above rather than a stale pre-audit state:

```powershell
$BackupPassword = Read-Host -Prompt "CA backup password" -AsSecureString

New-Item -Path "C:\PKI-Backup" -ItemType Directory -Force

Backup-CARoleService `
    -Path "C:\PKI-Backup" `
    -Password $BackupPassword `
    -Force

reg export `
    "HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration" `
    "C:\PKI-Backup\CA-Configuration.reg" `
    /y

Copy-Item `
    "C:\Windows\CAPolicy.inf" `
    "C:\PKI-Backup\CAPolicy.inf" `
    -Force
```

`Backup-CARoleService` and the registry export don't capture which certificate templates the Enterprise CA has published — that association lives in AD DS, not in the CA's own database. Record it alongside the rest of the backup:

```powershell
certutil -catemplates > "C:\PKI-Backup\CATemplates.txt"
```

At this point in the deployment the list is normally empty — `LoadDefaultTemplates=0` kept the CA from auto-publishing anything, and no production template has been published yet. `certutil -catemplates` can return `0x80070490 (Element not found)` in that case — that's expected when no template is assigned to the CA yet, not a failed backup step. Capturing it now still establishes the complete backup procedure; once production templates are published post-Checkpoint 2, every future backup of `U01PARVMPKI02` must repeat this step so the template list stays current.

Copy `C:\PKI-Backup` to protected offline storage. Never place it in `C:\UnreadLines` or on `U01PARVMWEB01`.

Store the backup password separately from the backup itself — anyone who obtains both can restore the Issuing CA's private key. `U01PARVMPKI02` stays online, unlike the Root CA, so don't leave the backup sitting indefinitely on its system disk: once the offline copy is verified, remove the local `C:\PKI-Backup` staging copy from `U01PARVMPKI02` as operational policy allows.

Do not wait here for the CRL to republish naturally — continue directly to Phase 6. The unattended-republication check belongs after Checkpoint 2; see the final section of this document.

**Next:** Phase 6, on `U01PARVMDOM01`.

---

## 16. Phase 6 — Deploy root trust via GPO

**Server:** `U01PARVMDOM01`, tested on a domain member. The Root CA is standalone — it isn't automatically trusted by domain computers, and a manual per-machine import doesn't scale.

### 16.1 Verify the root certificate

`UnreadLinesRootCA.crt` was already staged on `U01PARVMDOM01:C:\UnreadLines\` during Phase 3, while `U01PARVMPKI01` was still powered on — there's no need to power the Root CA back on solely to fetch a public certificate that's already been exported. If the file is missing here for some reason, retrieve the already-published public copy from `U01PARVMWEB01:C:\inetpub\wwwroot\pki\UnreadLinesRootCA.crt` instead. Do not power on `U01PARVMPKI01` just to obtain this certificate.

Before creating the GPO, verify it's the right file:

```powershell
Get-PfxCertificate `
    -FilePath "C:\UnreadLines\UnreadLinesRootCA.crt" |
    Select-Object Subject, Issuer, Thumbprint, NotBefore, NotAfter
```

Compare the thumbprint against the reference thumbprint recorded in Phase 1. Do not proceed if it doesn't match.

### 16.2 Create and link the GPO

On `U01PARVMDOM01`:

```text
gpmc.msc
  → Domains → corp.unreadlines.com
    → Create a GPO in this domain, and Link it here...
      Name: PKI - Trusted Root CA
```

Link it at the domain root; do not modify `Default Domain Policy`. Unlike the audit GPO in Phase 5, keep the default **Security Filtering: Authenticated Users** here — every domain computer needs this policy to apply.

### 16.3 Import the root certificate

Edit `PKI - Trusted Root CA` and navigate to:

```text
Computer Configuration
  -> Policies
    -> Windows Settings
      -> Security Settings
        -> Public Key Policies
          -> Trusted Root Certification Authorities
```

Right-click → **Import** → select `C:\UnreadLines\UnreadLinesRootCA.crt`.

Do not import `UnreadLinesIssuingCA.crt` here — only the Root CA certificate belongs in this store.

One domain-linked GPO is enough — Active Directory and SYSVOL replicate it to every Domain Controller, member server, and domain PC. There is no need to import the certificate manually on each Domain Controller. (`certutil -dspublish` is an alternative low-level method, but this GPO is the primary approach in this runbook.)

### 16.4 Prepare `U01PARVMADM01` and verify root trust

`U01PARVMADM01` serves double duty here: it's the member machine that proves the GPO works, and — once verified — it stays prepared as the dedicated, genuinely clean client used for Phase 7's leaf validation.

```text
ComputerName : U01PARVMADM01
IPv4Address  : 192.168.20.45
PrefixLength : 25
Gateway      : 192.168.20.126
DnsServers   : 192.168.20.41
```

Follow the standard Initial Manual Preparation procedure, then join `U01PARVMADM01` to `corp.unreadlines.com` the same way as `U01PARVMWEB01`/`U01PARVMPKI02` (see Phase 2's Domain Join). After the join and reconnect:

```powershell
whoami

Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Domain, PartOfDomain

Test-ComputerSecureChannel -Verbose

Resolve-DnsName "pki.corp.unreadlines.com"

gpupdate /force
gpresult /scope computer /r
```

Expect:

```text
Name                       = U01PARVMADM01
Domain                     = corp.unreadlines.com
PartOfDomain               = True
Test-ComputerSecureChannel = True
pki.corp.unreadlines.com   = 192.168.20.144
```

Then confirm Root trust reached this machine through the GPO:

```powershell
Get-ChildItem -Path "Cert:\LocalMachine\Root" |
    Where-Object Subject -like "*UnreadLines Root CA*" |
    Select-Object Subject, Issuer, Thumbprint
```

Compare the thumbprint against the reference thumbprint recorded in Phase 1. `U01PARVMADM01` is now the designated validation client for Phase 7 and Checkpoint 2.

### 16.5 Verify on a domain controller

Repeat the same check once on a Domain Controller — GPO scope and inheritance should reach Domain Controllers the same as any other domain member, and it is worth confirming rather than assuming:

```powershell
gpupdate /force

gpresult /scope computer /r
```

```powershell
Get-ChildItem -Path "Cert:\LocalMachine\Root" |
    Where-Object Subject -like "*UnreadLines Root CA*" |
    Select-Object Subject, Issuer, Thumbprint
```

**Next:** Phase 7.

---

## 17. Phase 7 — Leaf certificate validation

**Server:** `U01PARVMPKI02` to issue the certificate; validated from `U01PARVMADM01`, prepared and verified as the dedicated, genuinely clean validation client at the end of Phase 6. A clean validation client means, all conditions required:

- `UnreadLines Root CA` is trusted through the Phase 6 GPO.
- `UnreadLines Issuing CA` is absent from `Cert:\CurrentUser\CA`.
- `UnreadLines Issuing CA` is absent from `Cert:\LocalMachine\CA`.
- The URL retrieval cache is cleared before validation.
- `pki.corp.unreadlines.com` resolves correctly.

Windows builds a chain from whatever's already available locally before falling back to AIA, so a machine that already has the Issuing CA's certificate some other way (a prior enrollment, a management agent, any other manual step) would pass a naive test without the AIA URL ever actually being exercised — "never handed to it manually" is not enough to prove this on its own.

`certutil -verify -urlfetch` on the Issuing CA's own certificate only proves the Issuing CA chains to the Root CA — it fetches the Root CA's AIA/CDP, not anything the Issuing CA itself embeds into the certificates it issues:

```text
UnreadLinesIssuingCA.crt
        |
        v  Root AIA / Root CRL
UnreadLines Root CA
```

To actually confirm CDP/AIA on the **Issuing CA's issued certificates** work, issue one real leaf certificate and validate its full chain.

### 17.1 The designated test user

The leaf enrollment and the AIA/CDP validation below must be performed as a non-privileged domain user, never as Domain Admin — using Domain Admin for this is exactly what caused the intermediate-store confusion during this runbook's lab pass.

On `U01PARVMDOM01`, use an existing non-privileged domain user or create a dedicated temporary validation user. This runbook refers to it as the **designated test user** — for example, `No One` (UPN `u026080910@unreadlines.com`, sAMAccountName `u026080910`). The account only needs normal domain logon rights plus the `Read` + `Enroll` permissions granted on `PKI Validation` below; nothing more.

### 17.2 Create a temporary validation template

On `U01PARVMPKI02`: `certtmpl.msc` → `User` → right-click → **Duplicate Template**. On the **General** tab:

```text
Template display name : PKI Validation
Template name          : PKIValidation
```

This is a **user** template — enroll with a user account on `U01PARVMADM01`, not a computer account; don't mix the two.

On the **Subject Name** tab:

```text
Build from this Active Directory information : selected
Subject name format:
    Fully distinguished name
Include e-mail name in subject name:
    disabled
Alternate subject name:
    User principal name (UPN) : enabled
    E-mail name                : disabled
```

The temporary validation certificate does not require the test user's Active Directory mail attribute — leaving an e-mail requirement enabled can cause `0x80094812 CERTSRV_E_SUBJECT_EMAIL_REQUIRED` if that attribute isn't populated.

Duplicating the built-in `User` template copies its existing ACL, which includes `Domain Users: Enroll` — that inherited right has to be explicitly removed, not just left in place alongside a new, narrower entry, or any domain user could still enroll for it. On the **Security** tab, don't remove `Authenticated Users` entirely — the Enterprise CA itself needs `Read` on the template to offer it at all, and `Authenticated Users` is what normally carries that. Set:

```text
Authenticated Users
    Read       : Allow
    Enroll     : Not granted
    Autoenroll : Not granted

Designated test user (a user account used on U01PARVMADM01)
    Read       : Allow
    Enroll     : Allow
    Autoenroll : Not granted
```

Remove `Enroll`/`Autoenroll` from any broad inherited group such as `Domain Users`, `Everyone`, or `Domain Computers`. Do not grant `Autoenroll` permission to any principal — the validation certificate must be requested manually by the designated test user. Before publishing the template, review the full ACL once more and confirm no other broad group still carries `Enroll` or `Autoenroll`; a duplicated template can carry over rights beyond just the ones called out above, depending on what it was duplicated from.

That's restrictive enough for a temporary template — only the designated test identity can enroll.

Publish it: `certsrv.msc` → **Certificate Templates** → **New** → **Certificate Template to Issue** → select `PKI Validation`.

### 17.3 Issue one validation certificate

Log on to `U01PARVMADM01` as the designated test user and open `certmgr.msc` → **Personal** → **All Tasks** → **Request New Certificate** → select `PKI Validation` → **Enroll**.

This client hasn't necessarily used `C:\UnreadLines` before, so create it first:

```powershell
New-Item -Path "C:\UnreadLines" -ItemType Directory -Force
```

Still in the test user's session, export the issued certificate: `certmgr.msc` → **Personal** → **Certificates**. The `Issued To` / `Issued By` columns show the certificate's **Subject** and **Issuer**, not the template name — since the template builds the subject from Active Directory (`Fully distinguished name`), the certificate shows up under the test user's own name (e.g. `No One`), **not** as `PKI Validation`. Identify it by `Issued By: UnreadLines Issuing CA` and `Intended Purposes: Client Authentication` instead of by name — it should be the only certificate in this store.

Right-click it → **All Tasks** → **Export**. Choose:

```text
No, do not export the private key
DER encoded binary X.509 (.CER)
Destination: C:\UnreadLines\PKIValidation.crt
```

> ⚠️ The export wizard appends its own `.cer` extension to whatever you type, even if you already typed `.crt` — the file lands as `C:\UnreadLines\PKIValidation.crt.cer`, not `PKIValidation.crt`. Rename it so the commands later in this phase (`certutil -dump`, `certutil -verify -urlfetch`) find it at the expected path:
> ```powershell
> Rename-Item -Path "C:\UnreadLines\PKIValidation.crt.cer" -NewName "PKIValidation.crt"
> ```

Confirm it landed at the right name and path:

```powershell
Get-Item "C:\UnreadLines\PKIValidation.crt"
```

Do not run `certutil -URL` / `-verify` before this file actually exists.

### 17.4 Validate the full chain

On `U01PARVMADM01`, first confirm the Issuing CA's certificate genuinely isn't already present locally — this is what makes the AIA fetch below meaningful rather than a no-op. `Cert:\CurrentUser\CA` belongs to whichever account is currently logged on: checking it while logged on as a Domain Admin does **not** validate the designated test user's `CurrentUser\CA` store, so run that half of the check from the test user's own session.

`LocalMachine` (any administrative session on `U01PARVMADM01`):

```powershell
Get-ChildItem "Cert:\LocalMachine\CA" |
    Where-Object Subject -like "*UnreadLines Issuing CA*" |
    Select-Object PSParentPath, Subject, Thumbprint
```

`CurrentUser` (in the designated test user's own session):

```powershell
Get-ChildItem "Cert:\CurrentUser\CA" |
    Where-Object Subject -like "*UnreadLines Issuing CA*" |
    Select-Object PSParentPath, Subject, Thumbprint
```

Expect no results from either. If the Issuing CA's certificate is present, remove it — only on `U01PARVMADM01`, never on `U01PARVMPKI02`:

```powershell
Get-ChildItem "Cert:\LocalMachine\CA" |
    Where-Object Subject -like "*UnreadLines Issuing CA*" |
    Remove-Item -Force
```

```powershell
# in the test user's own session
Get-ChildItem "Cert:\CurrentUser\CA" |
    Where-Object Subject -like "*UnreadLines Issuing CA*" |
    Remove-Item -Force
```

Then re-run both checks above and confirm both stores are empty before continuing.

Return to the designated test user's session before continuing — the `LocalMachine\CA` cleanup above needs an administrative session, but everything from here on must run as the designated test user, not as an admin:

```text
ADM01 admin session
    → clean LocalMachine\CA if required

ADM01 test-user session
    → clean/check CurrentUser\CA
    → confirm LocalMachine\CA is empty
    → clear URL cache
    → dump leaf
    → verify -urlfetch
```

The following commands must be executed from the designated test user's session: `certutil -urlcache * delete`, `certutil -dump`, and `certutil -verify -urlfetch` below.

Clear any cached CDP/AIA/CRL results — otherwise a stale cached answer can pass even if the live chain is broken:

```powershell
certutil -urlcache * delete
```

When the cache is already empty, `certutil` may end with `0x80070103 (ERROR_NO_MORE_ITEMS)` — if WinINet and WinHTTP have no remaining cache entries, this is not a blocking error.

Before the final test, confirm the leaf certificate actually carries the expected extensions:

```powershell
certutil -dump "C:\UnreadLines\PKIValidation.crt"
```

Confirm the presence of:

```text
CRL Distribution Points:
    http://pki.corp.unreadlines.com/UnreadLinesIssuingCA.crl
Authority Information Access:
    http://pki.corp.unreadlines.com/UnreadLinesIssuingCA.crt
```

Optional interactive URL inspection:

```powershell
certutil -URL "C:\UnreadLines\PKIValidation.crt"
```

This opens the Retrieval Tools GUI. It's diagnostic only — a blank-looking window is not a failure, and there's nothing that must be clicked here to proceed. The authoritative end-to-end validation for this runbook is the command below; don't get stuck on the GUI:

```powershell
certutil -verify -urlfetch "C:\UnreadLines\PKIValidation.crt"
```

With the Issuing CA absent from the local intermediate stores and the URL cache cleared, a successful `certutil -verify -urlfetch` provides the intended validation of the HTTP AIA/CDP path. This walks the full chain:

```text
PKIValidation.crt
    | CDP
    v
UnreadLinesIssuingCA.crl

PKIValidation.crt
    | AIA
    v
UnreadLinesIssuingCA.crt
    | AIA / CDP
    v
UnreadLinesRootCA.crt / UnreadLinesRootCA.crl
```

Confirm: no CDP/AIA/revocation errors, both CRLs reachable and not expired, chain resolves fully to `UnreadLines Root CA`. A reference result looks like:

```text
Leaf → Issuing AIA       VERIFIED
Leaf → Issuing CRL       VERIFIED
Issuing → Root AIA       VERIFIED
Issuing → Root CRL       VERIFIED
Revocation check         PASSED
Full chain to Root       PASSED

Leaf certificate revocation check passed
CertUtil: -verify command completed successfully.
```

### 17.5 Remove the temporary template

On `U01PARVMPKI02`. Two separate operations — both are required, not just the first:

1. Unpublish it from the Issuing CA: `certsrv.msc` → `UnreadLines Issuing CA` → **Certificate Templates** → right-click `PKI Validation` → **Delete**. This only stops the CA from being able to issue it.
2. Delete the template object itself from Active Directory: `certtmpl.msc` → right-click `PKI Validation` → **Delete**.

After both, no `PKI Validation` template remains published or present in AD. No production template is enabled yet.

**Next:** Checkpoint 2.

---

## 18. Checkpoint 2 — PKI complete

The reference leaf validation for this checklist was performed from `U01PARVMADM01` (Phase 7). The deployment is complete only when all of the following hold:

- Root CA certificate trusted by GPO on a tested member machine and on a tested Domain Controller.
- Root CRL reachable, HTTP `200`.
- Root CA certificate reachable, HTTP `200`.
- Issuing CRL reachable, HTTP `200`.
- Issuing CA certificate reachable, HTTP `200`.
- Correct MIME types (`application/pkix-crl`, `application/x-x509-ca-cert`) confirmed for all four files above.
- Neither CRL expired.
- Issuing CA operational.
- A leaf validation certificate was successfully issued from `U01PARVMADM01` (Phase 7).
- `UnreadLines Issuing CA` confirmed absent from `U01PARVMADM01`'s local intermediate stores before the AIA test.
- `certutil -verify -urlfetch` succeeds on that leaf certificate.
- No CDP/AIA/revocation error anywhere in the chain.
- The full chain reaches `UnreadLines Root CA`.
- The temporary `PKI Validation` template was removed (unpublished from the CA **and** deleted from AD).
- Root CA backup completed and copied to protected offline storage (both generations — initial and post-signing).
- Issuing CA backup completed and copied to protected offline storage.
- `AuditFilter = 127` on both CAs.
- `Audit Certification Services = Success and Failure` verified as applied on `U01PARVMPKI02` (not just configured in the GPO).
- Root CA `Certification Services` audit was verified before `U01PARVMPKI01` was taken offline (Phase 1).

Only after this checkpoint can production certificate templates, permissions, auto-enrollment, and other integrations begin — all of that is out of scope for this runbook.

---

## 19. Post-deployment operational check — after the first automatic CRL cycle

This check is distinct from Checkpoint 2, which can be reached the same day: it confirms native, unattended CRL republication on `U01PARVMPKI02` actually happens on its own, and it requires waiting roughly 24 hours for the first natural `CRLPeriod` to elapse. Checkpoint 2 does not depend on it — perform it once, afterward, as a final confirmation.

Once that time has passed, repeat this check on `U01PARVMWEB01` **without** running `certutil -crl` by hand:

```powershell
Get-Item "C:\inetpub\wwwroot\pki\UnreadLinesIssuingCA.crl" |
    Select-Object Name, Length, LastWriteTime
```

Compare against the initial publication baseline recorded in Phase 5, right after the first `certutil -crl`. The check is demonstrable as: `New LastWriteTime > Initial LastWriteTime`, with no manual `certutil -crl` run in between. That's the real confirmation the native UNC publication works unattended, not just on a manually forced run.

## 20. Update the infrastructure inventory

Per `reference/server-naming-convention.md` §2, record `U01PARVMPKI01`, `U01PARVMPKI02` and
`U01PARVMWEB01` in `reference/vm-inventory.md` — name, role, site, address and owner. All three are
already listed there; confirm each row matches what was actually deployed, in particular that `PKI01`
is recorded as normally powered off.

## 21. References

- [Active Directory Certificate Services overview — Microsoft Learn](https://learn.microsoft.com/windows-server/identity/ad-cs/active-directory-certificate-services-overview)
- [Prepare the `CAPolicy.inf` file — Microsoft Learn](https://learn.microsoft.com/windows-server/networking/core-network-guide/cncg/server-certs/prepare-the-capolicy-inf-file)
- [`certutil` — Microsoft Learn](https://learn.microsoft.com/windows-server/administration/windows-commands/certutil)
- [RFC 5280 — Internet X.509 Public Key Infrastructure Certificate and CRL Profile](https://www.rfc-editor.org/rfc/rfc5280)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
