# UnreadLines — Naming Conventions

> Scope: naming rules for servers/VMs, user and administrator accounts, service accounts, workload identities, security groups, organizational units, cloud identity objects (Entra ID / Intune), and Azure resources across the UnreadLines infrastructure.
> For the Active Directory / Microsoft Entra domain architecture, UPN suffix strategy, and hybrid identity sync design, see `active-directory-entra-identity-design.md` — that document only covers AD DS / DNS / Entra ID design decisions and cross-references this one for every naming pattern.

---

## 1. Entities and sites

Codes used across OUs, groups, servers, delegations, and GPOs — **not** in human account identifiers (see §4):

```text
U00 = Global / cross-site
U01 = France

PAR = Paris
MAR = Marseille
BDX = Bordeaux
```

---

## 2. Server naming convention

### Naming format

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

### Server role codes

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
| `ECN` | Microsoft Entra Connect Sync server |

### Examples

```text
U01PARVMDOM01
U01PARVMDOM02
U01PARVMPKI01
U01PARVMNPS01
U01PARVMWEB01
U01PARVMFWL01
U01PARVMRAP01
U01PARVMECN01
```

### Naming rules

- Use uppercase ASCII letters and numbers.
- Do not use spaces, accents, underscores, or special characters.
- Keep the computer name at 15 characters or fewer for compatibility with legacy NetBIOS-dependent systems.
- Use a stable three-letter site code.
- Increment the final number for servers with the same role and site.
- Do not reuse a retired computer name while its identity may still exist in Active Directory or management systems.
- Record the assigned name, role, site, IP address, and owner in the infrastructure inventory.

This naming convention is an internal infrastructure policy. Microsoft does not require this exact pattern; it is designed to remain readable and compatible with Windows and Active Directory constraints.

---

## 3. Organizational unit (OU) structure

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

## 4. Group naming convention

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

This pattern is for **on-premises Active Directory groups**, scoped by physical site. Cloud-only groups that exist only in Entra ID — with no site to scope to — follow §9 instead.

---

## 5. Account naming — full taxonomy

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

## 6. Open point — numeric ID generation

Not yet defined by this convention:

- **Generation mechanism** for the 9-digit suffix (dedicated script, sequence table, truncated GUID/hash?) and how uniqueness is enforced at account-creation time.
- **Name ↔ identifier mapping**: by default this is just the AD/Entra user object itself (`displayName` + `sAMAccountName` + `mail` on the same record) — no separate table to build or declare, already covered by existing directory governance (RBAC/delegations, the HR/IT data protection register that already exists regardless of this convention). This only becomes a genuinely new artifact if the 9-digit suffix is generated or allocated by a tool/table **external to AD** before account creation — that external store would then need its own owner, access control, and retention policy.

This should be resolved (with an owner and a tool/process) before the convention is applied beyond a pilot group.

---

## 7. PowerShell validation

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

## 8. Rename a Windows server

Run PowerShell as Administrator before installing or promoting the server as a domain controller:

```powershell
Rename-Computer -NewName "U01PARVMDOM01" -Restart
```

Confirm the name after the restart:

```powershell
$env:COMPUTERNAME
```

---

## 9. Cloud identity objects (Entra ID / Intune)

Objects that exist only in the cloud tenant — no on-premises AD counterpart, so §3's OU tree and §4's site-scoped `GG-` groups don't apply to them. This section formalizes the pattern first executed in
`intune-trusted-certificate-profiles/README.md` §6, rather than leaving each new cloud-only lab to invent
its own — the gap this section closes was flagged but deliberately left open in earlier drafts of this
document (real precedent was needed before a pattern was worth locking in).

### 9.1 Cloud-only security groups (Entra ID)

```text
SG-<Entity><Site>-<Workload>-<Purpose>-<Qualifier>
```

| Part | Meaning | Example |
| --- | --- | --- |
| `SG` | **Security Group**, cloud-only (Entra ID) — mirrors what `GG` already means for §4's on-prem groups (AD group-scope prefix, Global Group), just for the cloud console instead of AD | `SG` |
| `<Entity><Site>` | §1 entity/site code, concatenated exactly as §2 (`U01PARVMDOM01`) and §10.1 (`rg-u01par-pki-01`) already do. `U00` alone, with **no city suffix** — Global / cross-site is by definition not tied to one city, so there is nothing to pair it with. A group genuinely tied to one physical location instead pairs entity + city, e.g. `U01PAR`, `U01MAR`, `U01BDX` | `U00` |
| `<Workload>` | The product/service the group scopes access or policy for | `Intune` |
| `<Purpose>` | What the group is for | `TrustedCert` |
| `<Qualifier>` | What distinguishes this group from siblings doing the same job for a different target | `Windows`, `AndroidCorp`, `AndroidBYOD`, `iOS` |

`SG-` deliberately does not collide with §4's on-prem `GG-` prefix: seeing which prefix a group uses says
which console it lives in (Entra ID vs. on-prem AD) without opening it. Unlike §4's `GG-` groups, the
entity/site code here doesn't mean "the users or machines physically at this site" — it means "the scope
this cloud object belongs to." A tenant-wide MDM policy still belongs to an entity (`U00` = the whole
company, cross-site), so it keeps the code instead of dropping it: every naming pattern in this document
draws from the same §1 codes, and a cloud-only group being platform-scoped rather than building-scoped
isn't a reason to break that consistency.

Executed example (4 groups, one per platform, `intune-trusted-certificate-profiles/README.md` §6 — `U00`
because these groups scope Intune-managed devices tenant-wide, not by site):

```text
SG-U00-Intune-TrustedCert-Windows
SG-U00-Intune-TrustedCert-AndroidCorp
SG-U00-Intune-TrustedCert-AndroidBYOD
SG-U00-Intune-TrustedCert-iOS
```

### 9.2 Intune configuration and compliance profiles

```text
<Category> - <Subject> - <Qualifier>
```

Title Case with spaces and a hyphen separator — Intune's own console convention, not the kebab-case or
ALL-CAPS used above, because these names are read directly by an admin inside the Intune admin center, not
typed as a CLI or DNS identifier the way a server or account name is.

Executed example: `Trusted Certificate - UnreadLines Root CA - Windows`
(`intune-trusted-certificate-profiles/README.md` §3).

### 9.3 App registrations and service principals

Reserved — not yet used by any lab. Follow §5.2's existing `app-*` / `mi-*` prefixes
(`app-terraform-prod`, `mi-webapp-prod`) rather than inventing a new one: an app registration or managed
identity is the same kind of object whether it is described in the account taxonomy or listed here, so one
prefix pair covers both.

---

## 10. Azure resource naming (Azure Resource Manager)

Not yet in use — the `unreadlines` tenant has no Azure subscription yet (see `context.md` §6's note on
Azure Cloud Shell requiring one). Documented ahead of the first real deployment so that lab isn't the one
that has to invent this convention under pressure.

### 10.1 Pattern

```text
<resource-abbr>-<entity><site>-<workload>-<NN>
```

Reuses §1's entity and site codes the same way §2's server convention does — a resource group or virtual
network is still a UnreadLines infrastructure object, and the point of §1 is that every naming convention
in this document draws from the same site codes rather than each inventing its own.

Example: `rg-u01par-pki-01` for a resource group.

### 10.2 Resource type abbreviations

Taken from Microsoft's Cloud Adoption Framework recommended abbreviations (§11), not invented locally —
using the same abbreviation everyone else's Azure documentation and tooling already expects reduces the
chance of a reader (or an AI assistant) misreading a resource type from its name.

| Resource type | Abbreviation |
| --- | --- |
| Management group | `mg` |
| Resource group | `rg` |
| Virtual network | `vnet` |
| Subnet | `snet` |
| Network security group | `nsg` |
| Public IP address | `pip` |
| Load balancer (internal / external) | `lbi` / `lbe` |
| Azure Firewall | `afw` |
| Azure Bastion | `bas` |
| Virtual machine | `vm` |
| Storage account | `st` |
| Key Vault | `kv` |
| Log Analytics workspace | `log` |
| Application Insights | `appi` |
| App Service | `app` |
| App Service plan | `asp` |
| Azure Container Registry | `cr` |
| Managed identity | `id` |

### 10.3 Platforms that don't allow hyphens

Storage accounts and Azure Container Registry names are lowercase alphanumeric only, no hyphens, and
capped at 24 and 50 characters respectively. For these two, concatenate the same parts without separators
instead: `<resource-abbr><entity><site><workload><NN>`, trimmed to fit the limit — e.g. `stu01parpki01`.

### 10.4 Environment

No environment segment (`prod`/`dev`/`lab`) — deliberately, the same way §5's account taxonomy carries no
site code. Every UnreadLines lab is a single environment; adding a segment that's always the same value
for every resource that will ever exist here would only add width, not information. Revisit this if a
second, genuinely separate Azure environment (a true production tenant, say) is ever stood up.

---

## 11. References

- [Abbreviation recommendations for Azure resources — Cloud Adoption Framework, Microsoft Learn](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
- [Define your naming convention — Cloud Adoption Framework, Microsoft Learn](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
