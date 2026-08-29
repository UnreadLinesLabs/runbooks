# UnreadLines — Infrastructure & Identity Naming Conventions

> Scope: naming rules for servers/VMs, user and administrator accounts, service accounts, workload identities, security groups, and organizational units across the UnreadLines infrastructure.
> For the Active Directory / Microsoft Entra domain architecture, UPN suffix strategy, and hybrid identity sync design, see `active-directory-entra-identity-design.md` — that document only covers AD DS / DNS / Entra ID design decisions and cross-references this one for every naming pattern.

---

## 1. Entities and Sites

Codes used across OUs, groups, servers, delegations, and GPOs — **not** in human account identifiers (see §4):

```text
U00 = Global / cross-site
U01 = France

PAR = Paris
MAR = Marseille
BDX = Bordeaux
```

---

## 2. Server Naming Convention

### Naming Format

```text
[A][NN][CITY]VM[ROLE][NN]
```

Example:

```text
U01PARVMDOM01
```

### Components

| Part | Meaning | Example |
| --- | --- | --- |
| `A` | Organization or environment code | `U` |
| `NN` | Site or entity number | `01` |
| `CITY` | Three-letter site or city code | `PAR` |
| `VM` | Virtual machine identifier | `VM` |
| `ROLE` | Three-letter server role | `DOM` |
| `NN` | Server sequence number | `01` |

### Server Role Codes

| Code | Role |
| --- | --- |
| `DOM` | Active Directory Domain Controller |
| `PKI` | Public Key Infrastructure |
| `NPS` | Network Policy Server |
| `DNS` | DNS Server |
| `WEB` | Web Server |
| `SQL` | SQL Server |
| `FSV` | File Server |
| `FWL` | Router / Firewall (OpenWrt) |
| `RAP` | Radio Access Point (Wi-Fi, hostapd) |

### Examples

```text
U01PARVMDOM01
U01PARVMDOM02
U01PARVMPKI01
U01PARVMNPS01
U01PARVMWEB01
U01PARVMFWL01
U01PARVMRAP01
```

### Naming Rules

- Use uppercase ASCII letters and numbers.
- Do not use spaces, accents, underscores, or special characters.
- Keep the computer name at 15 characters or fewer for compatibility with legacy NetBIOS-dependent systems.
- Use a stable three-letter site code.
- Increment the final number for servers with the same role and site.
- Do not reuse a retired computer name while its identity may still exist in Active Directory or management systems.
- Record the assigned name, role, site, IP address, and owner in the infrastructure inventory.

This naming convention is an internal infrastructure policy. Microsoft does not require this exact pattern; it is designed to remain readable and compatible with Windows and Active Directory constraints.

---

## 3. Organizational Unit (OU) Structure

Site and entity codes (§1) structure the OU tree; they are not carried into human account names.

```text
corp.unreadlines.com
└── OU=UnreadLines
    └── OU=U01
        ├── OU=PAR  (Users, Workstations, Servers, Groups, ServiceAccounts)
        ├── OU=MAR  (Users, Workstations, Servers, Groups, ServiceAccounts)
        └── OU=BDX  (Users, Workstations, Servers, Groups, ServiceAccounts)
```

`OU=UnreadLines` is a single top-level OU holding everything below it, kept separate from AD's built-in containers (`CN=Users`, `CN=Computers`, `CN=System`, ...) — it was renamed from an initial `UnreadLines Labs` to match the company name used everywhere else (`UnreadLines Root CA`, `UnreadLines Issuing CA`, the `corp.unreadlines.com` domain itself); "Labs" is the name of the project/channel producing these runbooks, not a name that belongs inside the fictional company's own AD.

A user can physically sit in `OU=Users,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com` while having a login that is independent of the site (see §4).

---

## 4. Group Naming Convention

```text
GG-U01-PAR-Users
GG-U01-PAR-IT
GG-U01-PAR-Helpdesk
GG-U01-PAR-ADM-Servers
GG-U01-PAR-ADM-Workstations
GG-U01-MAR-ADM-Servers
GG-U01-BDX-ADM-Workstations
GG-U00-ADM-Global
```

Groups — not the account name — carry the geographic/organizational scope. When responsibilities change, membership changes; the account name never does:

```powershell
Remove-ADGroupMember GG-U01-MAR-ADM-Servers a783476512
Add-ADGroupMember    GG-U01-BDX-ADM-Servers a783476512
```

---

## 5. Account Naming — Full Taxonomy

**Principle**: human account names never encode entity or site. Administrative scope is carried entirely by group membership (§4) and delegation, so an administrator can change scope without being renamed. A convention encoding the site directly in the account (e.g. `U01PARA7834767`) was considered and rejected for this reason.

### 5.1 Human accounts — opaque identifier format

```text
[1-letter prefix] + [9 digits]

u123456789   Standard user
a123456789   AD / on-prem administrator
c123456789   Cloud administrator
```

The 9 digits must be stable, non-sequential, and non-guessable — not derived from name, site, country, department, or arrival year. Avoid `jdupont`, `jean.dupont`, `u01-par-jdupont`, `2026-it-001`, or an HR employee number that is already visible elsewhere.

This opacity is a **defense-in-depth** measure against username enumeration, password spraying, credential stuffing, and targeted phishing — it does not replace MFA, passkeys/FIDO2, Windows Hello for Business, Conditional Access, Smart Lockout, or Entra ID Protection. The security rationale for keeping the login separate from the public e-mail address is detailed in the identity design document.

### 5.2 Account type matrix

| Type | Convention | Example | In AD | In Entra | Mailbox | Usage |
|---|---|---|---|---|---|---|
| Standard user | `uXXXXXXXXXXX` | `u783476512` | Yes | Synced | `firstname.lastname@unreadlines.com` | Workstation, Outlook, Teams, M365, business apps |
| AD / on-prem admin | `aXXXXXXXXXXX` | `a783476512` | Yes | Only if needed | No | ADUC, GPMC, DNS, DHCP, server admin, AD delegations |
| Cloud admin | `cXXXXXXXXXXX` | `c783476512` | Optional | Yes / cloud-only | No | Entra, M365, Exchange Online, Intune, Conditional Access |
| Classic service account | `svc-*` | `svc-backup` | As needed | Rarely | No | Legacy apps that don't support gMSA |
| gMSA | `gmsa-*$` | `gmsa-backup$` | Yes | No | No | Windows services with automatic password management |
| Entra application | `app-*` | `app-terraform-prod` | No | Yes | No | Service principal for CI/CD, automation |
| Managed Identity | `mi-*` | `mi-webapp-prod` | No | Yes | No | Cloud workloads, no stored credentials |
| Break-glass | `bg-*` | `bg-7f29c1` | No | Yes / cloud-only | No | Emergency access, excluded from Conditional Access |
| Guest | external identity | `alice@partner.com` | No | Entra B2B | External | Partners, consultants |

Never grant elevated privileges to a standard user account simply because its owner is also an administrator elsewhere — see the "separation of admin tiers" principle below.

### 5.3 Separation of admin tiers

```text
u783476512 → daily-driver account
a783476512 → AD / on-prem administration
c783476512 → cloud / Entra administration
```

Never let a single account be both the daily-driver identity and Domain Admin / Global Administrator: compromising the daily account must not compromise the whole estate.

### 5.4 Service accounts and workload identities

- **gMSA** (`gmsa-backup$`, `gmsa-monitor$`, `gmsa-sql$`) is the default choice for Windows services that support it: password is managed automatically by AD, no static secret to maintain.
- **Classic service accounts** (`svc-backup`, `svc-monitor`, `svc-vcenter`) only when gMSA isn't supported: no mailbox, no interactive logon unless required, never Domain Admin, least privilege, secrets in a vault, password rotation, restricted to specific machines.
- **Cloud workloads** should use a **Managed Identity** or **Service Principal** (`app-terraform-prod`, `mi-webapp-prod`) rather than a user account (`svc-app@id.unreadlines.com`), to avoid storing user passwords in scripts.

### 5.5 Guests and break-glass

- **External users**: use Microsoft Entra B2B and keep the partner's own identity (`alice@partner-company.com`) rather than creating `alice.consultant@unreadlines.com`. Grant only the groups, apps, and rights actually needed.
- **Break-glass accounts** (`bg-7f29c1@<tenant>.onmicrosoft.com`): cloud-only, on the tenant's native domain so they don't depend on the custom domain, AD sync, federation, or on-prem infrastructure. Must be **explicitly excluded from every Conditional Access policy**, tightly monitored, tested periodically, and covered by a documented usage procedure.

---

## 6. Open Point — Numeric ID Generation

Not yet defined by this convention:

- **Generation mechanism** for the 9-digit suffix (dedicated script, sequence table, truncated GUID/hash?) and how uniqueness is enforced at account-creation time.
- **Name ↔ identifier mapping**: by default this is just the AD/Entra user object itself (`displayName` + `sAMAccountName` + `mail` on the same record) — no separate table to build or declare, already covered by existing directory governance (RBAC/delegations, the HR/IT data protection register that already exists regardless of this convention). This only becomes a genuinely new artifact if the 9-digit suffix is generated or allocated by a tool/table **external to AD** before account creation — that external store would then need its own owner, access control, and retention policy.

This should be resolved (with an owner and a tool/process) before the convention is applied beyond a pilot group.

---

## 7. PowerShell Validation

Validate a proposed server name:

```powershell
$ComputerName = "U01PARVMDOM01"

if ($ComputerName -match '^[A-Z][0-9]{2}[A-Z]{3}VM[A-Z]{3}[0-9]{2}$' -and
    $ComputerName.Length -le 15) {
    Write-Host "Valid server name: $ComputerName"
}
else {
    Write-Error "Invalid server name: $ComputerName"
}
```

Validate a proposed human account identifier:

```powershell
$AccountName = "u783476512"

if ($AccountName -match '^[uac][0-9]{9}$') {
    Write-Host "Valid account name: $AccountName"
}
else {
    Write-Error "Invalid account name: $AccountName"
}
```

## 8. Rename a Windows Server

Run PowerShell as Administrator before installing or promoting the server as a domain controller:

```powershell
Rename-Computer -NewName "U01PARVMDOM01" -Restart
```

Confirm the name after the restart:

```powershell
$env:COMPUTERNAME
```
