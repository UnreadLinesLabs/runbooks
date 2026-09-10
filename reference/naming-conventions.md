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

**A server name carries two independent things: which entity administratively owns it, and which city it
physically runs in.** The two don't have to agree. `U01PARVMPKI01` reads as "France's, hosted in Paris";
`U00PARVMPNC01` reads as "the company's, hosted in Paris" — same real hardware in the same city, different
entity because the *service* is scoped differently: a country's own infrastructure vs. something shared by
the whole company regardless of country. A VM's city segment is never `U00` alone — every physical machine
runs somewhere real, so the city segment always names a real city, whichever entity owns the VM. `U00` with
no city at all (§9.1) is reserved for objects with no physical location whatsoever: a cross-site AD/Entra
group (`GG-U00-ADM-Global`), a tenant-wide Intune policy, the `OU=U00` branch itself (§3) — never a VM.

**Renaming an already-built, already-documented server for this is expensive** — every runbook, screenshot
and cross-reference that names it has to be found and updated, and a renamed *folder* also breaks any
published video URL (`context.md` §6). `U01PARVMPKI01`/`02` stay `U01PAR` even though the PKI they run is
company-wide, precisely because `ad-cs-pki-deployment/README.md` already documents them under that name —
the inconsistency is accepted and will be called out as "written before this convention existed" rather
than silently hidden. `U00PARVMPNC01` (still `Planned` in `vm-inventory.md`, backlog item 5) had no runbook
written yet at the time this section was drafted, so it cost nothing to rename to the company-wide entity
code from the start.

`U01PARVMNDS01` (backlog item 4) took the same path first — reserved as `U00PARVMNDS01` under this same
reasoning on 2026-09-05 — and then reverted on 2026-09-09, once its runbook
(`ndes-scep-intune-connector/README.md`) was already written and real infrastructure construction had
begun: the server is `U01PARVMNDS01` again. `vm-inventory.md`, `YouTube/backlog.md` and the repository
index (`runbooks/README.md`) were corrected in that same pass; every remaining reference to
`U00PARVMNDS01` — in that runbook itself, `notes/wifi-mobile-certificate-chain.md`,
`notes/entra-private-network-connector-app-proxy.md` and `configure-kds-root-key-for-gmsa/README.md`
— was missed at the time and corrected only on 2026-09-10, during a cleaning pass. Don't assume a
"corrected" note here means every file was actually touched — verify.

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
| `NDS` | Network Device Enrollment Service (NDES / SCEP) |
| `PNC` | Microsoft Entra Private Network Connector |

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
U01PARVMNDS01
U01PARVMPNC01
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
    ├── OU=U00                 (Global / cross-site)
    │   ├── OU=Accounts        (a-accounts only — see below)
    │   ├── OU=Groups          (groups scoped to the whole company, incl. GG-U00-ADM-Global — see below)
    │   ├── OU=Servers         (servers/computer objects managed at company level, not one city)
    │   └── OU=ServiceAccounts (service accounts/gMSA used company-wide)
    └── OU=U01                 (France)
        ├── OU=Groups          (groups scoped to France, incl. every city's GG-U01-<city>-ADM-* — see below)
        ├── OU=Servers         (servers/computer objects managed at country level, not one city)
        ├── OU=ServiceAccounts (service accounts/gMSA scoped to France as a whole)
        ├── OU=PAR  (Accounts, Computers, Groups, Servers, ServiceAccounts)
        ├── OU=MAR  (Accounts, Computers, Groups, Servers, ServiceAccounts)
        └── OU=BDX  (Accounts, Computers, Groups, Servers, ServiceAccounts)
```

`OU=UnreadLines` is a single top-level OU holding everything below it, kept separate from AD's built-in containers (`CN=Users`, `CN=Computers`, `CN=System`, ...) — it was renamed from an initial `UnreadLines Labs` to match the company name used everywhere else (`UnreadLines Root CA`, `UnreadLines Issuing CA`, the `corp.unreadlines.com` domain itself); "Labs" is the name of the project/channel producing these runbooks, not a name that belongs inside the fictional company's own AD.

**`OU=U00` and `OU=U01` are siblings at the same level, for the same reason they're siblings in §1**: every direct child of `OU=UnreadLines` is an entity code, so reading the top of the tree tells you which entity (not which purpose) an object belongs to before you go one level deeper — exactly the way `OU=U01` branches into its cities (`PAR`/`MAR`/`BDX`), `OU=U00` branches into its own children. That symmetry is what keeps the tree extensible without a redesign: a second country becomes a sibling of `OU=U01` with the same shape underneath, not a reason to restructure anything.

**`Groups`, `Servers` and `ServiceAccounts` exist at every entity level — global (`OU=U00`), country (`OU=U01`), and city (`OU=PAR`/`MAR`/`BDX`)** — a group, a managed server or a service account can legitimately be scoped to exactly one city, to a whole country regardless of city, or to the whole company regardless of country, and each scope gets its own OU so it is visible from the path alone, not just from the name (a service account used by an application that runs for every French site, not just Paris, belongs in `OU=U01/ServiceAccounts`, not in one city's). Each `ServiceAccounts` OU is flat and does not get re-split by entity underneath itself — the `U00`/`U01`/city branching happens exactly once, at the top of the tree; there is no `OU=U00/ServiceAccounts/U01`. In practice `OU=U00/ServiceAccounts` mostly ends up holding accounts for infrastructure that is inherently company-wide by function, not by hosting location: AD DS/replication, the PKI (`ad-cs-pki-deployment`), NDES/Intune Certificate Connector, Microsoft Entra Connect — one shared forest, one shared CA hierarchy, one shared tenant, regardless of which city's hardware the VM (`U01PARVMPKI01`, `U01PARVMNDS01`, `U01PARVMECN01`, ...) happens to run on (§1's `U00` clarification above already covers why the VM name itself still carries `U01PAR`). A service account tied to something genuinely city-specific instead — a file server used only by the Bordeaux office, say — goes in that city's own `ServiceAccounts`.

**`Accounts` and `Computers` are the two exceptions: city-only**, except that `OU=U00/Accounts` also exists, for a different reason than the other `U00`/`U01` OUs above. A person always works out of one physical office, and a workstation (like a server, §2) always runs in one physical city — there is no "France, no particular city" flavor of either, so neither gets a country-level OU, and `Computers` doesn't exist at `U00` either. `OU=U00/Accounts`, by contrast, is not a "global computer/workstation"-style exception — it exists because **exactly one account type has no city to begin with**: an `a…` account's scope lives entirely in its `GG-*-ADM-*` group memberships (§4), which can span one city, all of France, or the whole company and can change without the account moving — so unlike a `u…` account, it was never going to sit in any one city's `Accounts` OU. Putting every `a…` account in `OU=U00/Accounts` also buys tier isolation for free: a GPO or a delegation scoped to `OU=U01/*/Accounts` or `OU=U01/*/Computers` never reaches an administrator account, and a helpdesk delegation over a city's `Accounts` OU carries no rights over `OU=U00/Accounts`. Naming this OU `Accounts` rather than `Users` matters here too: `u…` and `a…` sit in an OU of the same name at two different points in the tree, which matches the vocabulary §5 already uses (`Account naming`, `Account type matrix`) and gives Microsoft Entra Connect's OU-based sync filtering a clean line to draw, since every `a…` account is now outside every city's `Accounts` OU by construction, not merely by convention.

A city's own `Groups` OU holds only its non-`ADM` groups (`GG-U01-PAR-Users`, `GG-U01-PAR-IT`, `GG-U01-PAR-Helpdesk`) — **an `ADM` group is never placed in the `Groups` OU of the OU subtree it delegates rights over; it goes one level up instead.** `GG-U01-PAR-ADM-Servers` and `GG-U01-PAR-ADM-Workstations` delegate rights inside `OU=PAR`, so they live in `OU=U01/Groups`, not `OU=PAR/Groups`: whoever only holds the rights those groups grant has no rights over `OU=U01` at all, and so cannot touch `OU=U01/Groups` to add themselves, or anyone else, to their own delegation group. The same rule pushes one level higher again — whoever can manage `OU=U01/Groups` (and so, indirectly, every city's `ADM` membership) needs delegated rights on `OU=U01` itself, and that delegation has to be granted from `OU=U00`, one level up from `OU=U01`. `GG-U00-ADM-Global` is where that chain ends: it lives in `OU=U00/Groups`, at the top of the custom tree, so there is no further OU to push it up into — its own membership has to be anchored by AD's built-in tier-0 protection (`Domain Admins`/`Enterprise Admins`), not by another custom group, because nothing in this tree sits above `OU=U00`. The same one-level-up placement applies to any other group that itself grants delegated rights (`GG-U01-PAR-Helpdesk` included, if it is ever wired into an actual OU delegation rather than just used for ticketing/routing) — a plain membership or workload group, which grants no rights of its own, has no such constraint and stays at the level matching its own scope, same as `ServiceAccounts`.

A user can physically sit in `OU=Accounts,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com` while having a login that is independent of the site (see §4). A new country adds a sibling of `OU=U01` (e.g. `OU=U02`) with its own `Groups`/`Servers`/`ServiceAccounts` plus the same five-OU pattern under each of its cities; `OU=U00` does not change shape when a country is added.

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

**The site segment is omitted, not replaced by a placeholder, whenever a group has no single city to scope to.** `GG-U00-ADM-Global` already does this: `GG-<Entity>-<Purpose>`, no site, because `U00` by definition never has one. The same omission applies one level down, for a group scoped to all of France but not to one particular city — `GG-U01-ADM-<Purpose>`, not `GG-U01-<some invented city>-ADM-<Purpose>`. Nothing new is introduced for that case; it is the same rule already used for `U00`, applied at `U01` instead — no example exists yet because no such group has been needed, but the pattern is settled: a site segment appears only when the group is genuinely scoped to one city, and disappears rather than being faked when it isn't.

This is the opposite of §2's server names, where the city segment is *never* omitted, `U00` included (§1): a VM always runs on real hardware somewhere, so it always carries a real city, whichever entity administratively owns it (`U00PARVMPNC01`). A group can legitimately have no single place to point to; a physical machine cannot.

Where these groups live in the OU tree: a plain group in the `Groups` OU matching its own scope (city, country, or company-wide); an `ADM` group one level *above* the OU subtree it delegates rights over instead — see §3.

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
