# Active Directory + Microsoft Entra ID — UnreadLines hybrid identity design

> **Purpose**
> Define the domain architecture and the hybrid identity strategy across Active Directory Domain Services (AD DS), Microsoft Entra ID and Microsoft 365: which domain carries what, how the UPN suffix is configured and synced, and in which order to deploy.
>
> **Scope**: this document covers only the AD DS / DNS / Entra ID architecture and synchronization. The naming rules themselves (user/admin account format, service accounts, groups, OUs, servers) are centralized in `naming-conventions.md`, referenced below wherever relevant.

---

## 1. Target architecture

| Domain | Role |
|---|---|
| `corp.unreadlines.com` | Internal AD DS domain (infrastructure) |
| `id.unreadlines.com` | Identity / authentication namespace (UPN) |
| `unreadlines.com` | Public domain / messaging |
| `<tenant>.onmicrosoft.com` | Native Microsoft tenant domain |

```text
corp.unreadlines.com     → Active Directory infrastructure
id.unreadlines.com       → Identity / authentication (UPN)
unreadlines.com          → Communication / public email address
<tenant>.onmicrosoft.com → Native Microsoft Entra / M365 domain
```

The AD domain stays an internal technical namespace; there is no need for users to sign in with `user@corp.unreadlines.com`. An **Alternative UPN suffix** (`id.unreadlines.com`) carries the sign-in identity, independently of the AD domain's DNS name.

**Reference example** (used throughout this document — detailed account format in `naming-conventions.md` §5):

| Attribute | Value |
|---|---|
| Name | Jean Dupont |
| `sAMAccountName` | `u783476512` |
| `userPrincipalName` (AD and Entra) | `u783476512@id.unreadlines.com` |
| `mail` | `jean.dupont@unreadlines.com` |

---

## 2. Why separate the sign-in identity from the email address

A convention such as `firstname.lastname@unreadlines.com` as the sign-in identifier is easy to reconstruct from public information (website, LinkedIn, email signatures), which makes username enumeration, password spraying, credential stuffing and targeted phishing easier.

With `u783476512@id.unreadlines.com`, the login can no longer be derived directly from the first and last name.

**⚠️ This is only a defense-in-depth measure.** It does not replace: MFA, passkeys/FIDO2, Windows Hello for Business, Conditional Access, Smart Lockout, Microsoft Entra ID Protection, Password Protection, password policies, or separation of privileged accounts (see §7).

The exact identifier format (`u`/`a`/`c` prefix + 9 digits) and the full account taxonomy are defined in `naming-conventions.md` §5.

---

## 3. Is this a Microsoft best practice?

A distinction has to be made between **Microsoft support** and **Microsoft's default recommendation**.

Microsoft generally recommends a simple UPN, often aligned with the primary email address (`jean.dupont@unreadlines.com` for both the UPN and SMTP). The UnreadLines design — UPN `u783476512@id.unreadlines.com` distinct from the SMTP address `jean.dupont@unreadlines.com` — is a **deliberate architecture choice**, not the default recommendation.

It remains supported, provided that:

- `id.unreadlines.com` is a verified domain in Entra ID;
- `userPrincipalName` stays correctly populated in AD and is synced as-is to Entra;
- the applications in use are tested against this UPN / SMTP separation (see §8.1);
- the organization clearly documents the difference between login and email address.

Present it as: *"Microsoft generally recommends a simple user identity, often aligned with the primary SMTP address. UnreadLines deliberately chooses a separate identity namespace so that the sign-in identifier cannot be derived directly from the public email address."* — not as an official Microsoft recommendation.

---

## 4. Attributes: do not confuse `sAMAccountName`, UPN, `mail` and `proxyAddresses`

| Attribute | Example | Role |
|---|---|---|
| `sAMAccountName` | `u783476512` | Legacy AD identifier (`UNREADLINES\u783476512`) |
| `userPrincipalName` | `u783476512@id.unreadlines.com` | Modern sign-in identity; normal source of the Entra UPN via Entra Connect |
| `mail` | `jean.dupont@unreadlines.com` | Email address associated with the user |
| `proxyAddresses` | `SMTP:jean.dupont@unreadlines.com`<br>`smtp:j.dupont@unreadlines.com` | `SMTP:` (uppercase) = primary address; `smtp:` (lowercase) = secondary aliases |

---

## 5. Domains to configure in Microsoft Entra

Before any AD → Entra sync, the tenant must contain at least:

```text
<tenant>.onmicrosoft.com   (already present natively)
unreadlines.com
id.unreadlines.com
```

The order is not interchangeable: `unreadlines.com` must be added and verified **before** `id.unreadlines.com`, since the latter relies on the already-verified root domain to verify itself automatically.

> ⚠️ Do not confuse this with `admin.microsoft.com`, which is the **Microsoft 365 admin center** — a different portal. Entra ID domain management is done at `entra.microsoft.com`.

**Step 1 — Add and verify `unreadlines.com` first**: [Microsoft Entra admin center](https://entra.microsoft.com) (`entra.microsoft.com`) → *Entra ID → Domain names → Custom domain names → Add* → enter `unreadlines.com`. Microsoft provides a DNS record (usually TXT) to add at the public DNS provider, then *Verify*. Do not move on to step 2 until this domain shows as *Verified*.

**Step 2 — Add `id.unreadlines.com` once step 1 is verified**: same path, *Custom domain names → Add* → enter `id.unreadlines.com`. Because it is a subdomain of a root domain already verified in the same tenant, it usually verifies automatically, with no extra TXT record. Expected result:

```text
<tenant>.onmicrosoft.com    Verified
unreadlines.com             Verified
id.unreadlines.com          Verified
```

**Primary domain**: set to `id.unreadlines.com`, because it is the suffix offered by default when creating a new cloud user, which is consistent with the authentication namespace. Changing the Primary domain does not automatically rename existing users.

`unreadlines.com` nonetheless remains a fully verified custom domain, used for M365, Exchange Online, email addresses, groups and applications — there is no contradiction in running `id.unreadlines.com` as the identity namespace and `unreadlines.com` as the SMTP namespace in parallel.

Documentation: [Managing custom domains in Entra](https://learn.microsoft.com/en-us/entra/identity/users/domains-manage)

---

## 6. UPN suffix in Active Directory and account migration

The AD domain stays `corp.unreadlines.com` — **the forest must not be renamed**. `id.unreadlines.com` is simply added as an alternative UPN suffix.

**GUI**: *Active Directory Domains and Trusts → right-click → Properties → Alternative UPN suffixes → Add → `id.unreadlines.com`*

**PowerShell**:

```powershell
Set-ADForest -Identity "corp.unreadlines.com" `
    -UPNSuffixes @{Add="id.unreadlines.com"}

# Verification
Get-ADForest | Select-Object -ExpandProperty UPNSuffixes
```

### 6.1 Legacy `unreadlines.com` suffix

Adding `id.unreadlines.com` changes **no account automatically**: `sAMAccountName`, `userPrincipalName`, `mail` and `proxyAddresses` stay unchanged until they are explicitly modified — which allows a controlled migration. Keep `unreadlines.com` in the UPN suffix list temporarily during the transition; do not remove it before every account and dependency has been checked.

### 6.2 Modifying an existing user

```powershell
Set-ADUser u783476512 -UserPrincipalName "u783476512@id.unreadlines.com"

Get-ADUser u783476512 -Properties UserPrincipalName,mail |
    Select-Object SamAccountName,UserPrincipalName,mail
```

Expected result:

```text
SamAccountName    : u783476512
UserPrincipalName : u783476512@id.unreadlines.com
mail              : jean.dupont@unreadlines.com
```

**Do not modify the `mail` address** when changing the UPN: the change applies to the sign-in identity, not to the public address `jean.dupont@unreadlines.com`.

### 6.3 Bulk changes — always dry-run first

```powershell
$Users = Get-ADUser -Filter 'UserPrincipalName -like "*@unreadlines.com"' `
    -Properties UserPrincipalName

foreach ($User in $Users) {
    $Prefix = $User.UserPrincipalName.Split("@")[0]
    $NewUPN = "$Prefix@id.unreadlines.com"
    Write-Host "$($User.UserPrincipalName) -> $NewUPN"

    # Uncomment only after validating the dry-run
    # Set-ADUser $User -UserPrincipalName $NewUPN
}
```

### 6.4 Verification commands

```powershell
# List users
Get-ADUser -Filter * -Properties UserPrincipalName,mail |
    Select-Object SamAccountName,UserPrincipalName,mail

# Filter UPNs still using the legacy suffix
Get-ADUser -Filter * -Properties UserPrincipalName |
    Where-Object { $_.UserPrincipalName -like "*@unreadlines.com" } |
    Select-Object SamAccountName,UserPrincipalName

# List users already on the new namespace
Get-ADUser -Filter * -Properties UserPrincipalName |
    Where-Object { $_.UserPrincipalName -like "*@id.unreadlines.com" } |
    Select-Object SamAccountName,UserPrincipalName
```

---

## 7. Synchronization with Microsoft Entra Connect

Sync has not been started yet — the domains and UPNs can therefore be prepared before the first sync run.

```text
AD : userPrincipalName = u783476512@id.unreadlines.com
        ↓  Microsoft Entra Connect
Entra : userPrincipalName = u783476512@id.unreadlines.com
```

**UPN source attribute**: keep `userPrincipalName` (Microsoft's default recommendation). There is no need to use the email address as an *Alternate Login ID* — doing so would in fact conflict with the goal of not exposing the login through the public address (see also §8.4).

**Why `id.unreadlines.com` must be verified before sync**: Entra ID requires the synced UPN suffix to match a verified custom domain. If an AD account carries `u783476512@id.unreadlines.com` but `id.unreadlines.com` is not verified in the tenant, Entra may substitute the suffix with the native domain (`u783476512@<tenant>.onmicrosoft.com`).

**Which account types get synced** (full type taxonomy in `naming-conventions.md` §5):

| Type | Synchronization |
|---|---|
| Standard user (`u…`) | Generally synced AD → Entra |
| AD admin (`a…`) | May stay on-prem only if there is no cloud need |
| Cloud admin (`c…`) | May be created directly in Entra, cloud-only |
| Break-glass (`bg-…`) | Must stay cloud-only, excluded from Conditional Access (§8.3) |
| gMSA | Technical AD account, not a regular Entra user |

---

## 8. Points to settle before going to production

### 8.1 Impact on third-party applications and Exchange hybrid

The UPN ≠ SMTP separation must be tested explicitly, not simply assumed to be compatible. Known risks to check:
- legacy Outlook Autodiscover based on an SCP, which in some hybrid scenarios assumes UPN = email;
- third-party SAML/SCIM connectors that assume UPN = email address for provisioning;
- legacy Teams/Skype federation or VPN clients that pre-fill the login from the directory email address.

Recommendation: build a list of the SSO/SAML/SCIM applications in place and validate each one explicitly against the new design before rolling it out broadly.

### 8.2 Synced vs cloud-only accounts — consistency with the taxonomy

Make sure the sync exclusions (AD admins not needed in the cloud, service accounts) are actually reflected in the Entra Connect scoping (OU filtering or sync rules), not just documented.

### 8.3 Conditional Access and break-glass accounts

Break-glass accounts must be **explicitly excluded from every Conditional Access policy**. Without that exclusion, a break-glass account can end up locked out at the exact moment it is meant to be used.

### 8.4 User experience of the opaque login

Making people type `u783476512@id.unreadlines.com` (instead of a memorable address) degrades the sign-in experience on surfaces not covered by SSO/Windows Hello: third-party web portals, VPN, manually entered Wi-Fi profiles, phone support. Choosing not to use email as an *Alternate Login ID* is consistent with the security objective, but this UX trade-off should be owned explicitly, with a mitigation plan (auto-provisioning through MDM/Intune profiles, federated SSO wherever possible, an enterprise password manager).

### 8.5 UPN rollback procedure

Currently missing: define the scenario that triggers a rollback, the command (`Set-ADUser -UserPrincipalName <previous>`), the scope involved, and the Entra Connect re-sync that must follow.

---

## 9. Privileged account security and the limits of an opaque UPN

**Privileged accounts** (`a…`, `c…`) — hardened protections to consider: strong MFA, passkeys/FIDO2, Windows Hello for Business, Conditional Access, Privileged Identity Management, dedicated admin workstations, sign-in restrictions, no mailbox and no internet browsing, least privilege, enhanced logging, risky sign-in monitoring.

**What an opaque UPN does not replace.** Making the login harder to derive (`u783476512@id.unreadlines.com`) does not on its own protect against: session hijacking, phishing, AiTM, malware, endpoint compromise, token theft, Conditional Access misconfiguration, weak passwords, MFA fatigue, or the compromise of an administrator. It is an additional layer, not the primary control.

---

## 10. Checklist and deployment order

**Checks before Entra Connect:**

- [ ] `corp.unreadlines.com` works correctly as the AD DS domain; DNS and AD replication healthy
- [ ] `unreadlines.com` added and verified in Entra
- [ ] `id.unreadlines.com` added and verified in Entra
- [ ] `id.unreadlines.com` added as an alternative UPN suffix in AD
- [ ] all pilot users use `@id.unreadlines.com`
- [ ] `mail` addresses stay on `@unreadlines.com`
- [ ] accounts to sync placed in clearly identified OUs; exclusions defined (admins, service accounts not needed)
- [ ] pilot user group defined
- [ ] matching strategy for any existing cloud accounts verified
- [ ] UPN rollback procedure documented (§8.5)
- [ ] break-glass accounts created and excluded from Conditional Access policies (§8.3)

**Deployment order:**

```text
01. Verify AD DS / DNS
02. Add unreadlines.com in Entra, then verify it
03. Add id.unreadlines.com in Entra, then verify it
04. Set id.unreadlines.com as the Primary domain (chosen option)
05. Add id.unreadlines.com as a UPN suffix in AD
06. Create/modify the pilot users: uXXXXXXXXXXX@id.unreadlines.com
07. Check mail = firstname.lastname@unreadlines.com
08. Organize the sync OUs and the exclusions
09. Install/configure Microsoft Entra Connect (UPN source = userPrincipalName)
10. Sync the pilot group
11. Check the UPNs in Entra, authentication, M365, SSO applications
12. Extend the sync progressively
```

---

## 11. Architecture decision — summary

```text
AD domain         corp.unreadlines.com
Identity          id.unreadlines.com
Email             unreadlines.com
Native MS domain  <tenant>.onmicrosoft.com

User              u783476512@id.unreadlines.com
Public email      jean.dupont@unreadlines.com
```

This separation is deliberate. It aims to keep a clean technical AD domain, use a consistent identity namespace, avoid revealing the login directly from the public email address, and keep an identity that stays stable over time (naming rules detailed in `naming-conventions.md`).

```text
WHO the user is                →  u783476512  (naming-conventions.md §5)
HOW they authenticate          →  u783476512@id.unreadlines.com  (this document, §6-7)
HOW they communicate           →  jean.dupont@unreadlines.com
WHERE / WHAT they administer   →  OUs + groups + delegations  (naming-conventions.md §3-4)
```

It must be paired with Microsoft's modern authentication and identity protection controls (§9) — login opacity is only one layer among many.

---

## 12. Microsoft references

- [Microsoft Entra Connect — Custom installation](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-custom) — `userPrincipalName` as the attribute normally used as the UPN; UPN suffixes = domains verified in Entra; keep `userPrincipalName` as the source where possible.
- [Microsoft Entra Connect — Design concepts](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/plan-connect-design-concepts) — `username@domain` format; risk of falling back to `onmicrosoft.com` if the suffix is not verified.
- [Managing custom domains in Entra](https://learn.microsoft.com/en-us/entra/identity/users/domains-manage) — adding/verifying a domain, managing the Primary domain, subdomains; changing the Primary domain does not rename existing users.
- [Planning UPN changes](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/howto-troubleshoot-upn-changes) — test UPN changes, plan a rollback, use a pilot scope, verify the new suffix before syncing.

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
