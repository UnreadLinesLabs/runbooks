# Create a Group Managed Service Account (gMSA)

A Group Managed Service Account (gMSA) is Active Directory's answer to a Windows service needing
credentials without a human maintaining a static password: the domain controllers generate and rotate
the password automatically, and only the hosts explicitly authorized for it can ever retrieve it.
`reference/naming-conventions.md` §5.4 makes gMSA (`gmsa-*$`) the default choice for any Windows service
that supports it, ahead of a classic `svc-*` account with a manually managed password.

This runbook is generic: creating and installing a gMSA is the same procedure whatever the future host or
role — `gmsa-backup$`, `gmsa-monitor$` and `gmsa-sql$` are the other names already reserved by that same
convention, not yet built. The worked example throughout is `gmsa-ndes$`, authorized only on
`U01PARVMNDS01`, the NDES / Intune Certificate Connector host built by
`ndes-scep-intune-connector/README.md`.

Depends on `configure-kds-root-key-for-gmsa/README.md` — a KDS root key must already exist in the forest
before any gMSA can be created.

## 1. Architecture

```text
corp.unreadlines.com

  U01PARVMDOM01 (AD DS, DNS — KDS root key already created:
                  configure-kds-root-key-for-gmsa/README.md)
        │
        │  New-ADServiceAccount / Install-ADServiceAccount
        ▼
  gmsa-ndes$ (AD service account object — its password is held only by AD, never stored anywhere else)
        │
        │  PrincipalsAllowedToRetrieveManagedPassword = U01PARVMNDS01$
        ▼
  U01PARVMNDS01 — the only host allowed to ask a domain controller for gmsa-ndes$'s current password
```

## 2. Scope and dependencies

Creates and installs one gMSA end to end, from the AD object to a confirmed working install on its target
host. Does not cover what the account is actually used for once installed — the NDES service and Intune
Certificate Connector configuration is `ndes-scep-intune-connector/README.md`.

Depends on `configure-kds-root-key-for-gmsa/README.md`.

## 3. Target configuration

| Item | Value |
| --- | --- |
| Account name | `gmsa-ndes$` |
| Host allowed to retrieve the password | `U01PARVMNDS01` |
| Organizational unit | `OU=ServiceAccounts,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com` |

## 4. Prerequisites

- A KDS root key already created and effective in the forest — `configure-kds-root-key-for-gmsa/README.md`
  §7.
- `U01PARVMNDS01` already exists as a domain-joined computer object.
- The Active Directory PowerShell module, available on `U01PARVMDOM01` and on any RSAT-enabled admin
  host, including `U01PARVMADM01`.

## 5. Decide what is allowed to retrieve the password

`-PrincipalsAllowedToRetrieveManagedPassword` accepts a security group or a computer object directly. A
group earns its place once more than one host needs the same account; for a single host like
`U01PARVMNDS01`, naming a group for it adds a layer of indirection with nothing to scope —
`reference/naming-conventions.md` §4 groups exist for site/delegation scoping, not for a single-membership
wrapper. This runbook authorizes `U01PARVMNDS01$` directly, and only switches to a dedicated group if a
second host is ever added.

## 6. Create the gMSA account

**On `U01PARVMDOM01`:**

```powershell
New-ADServiceAccount -Name "gmsa-ndes" `
    -DNSHostName "gmsa-ndes.corp.unreadlines.com" `
    -PrincipalsAllowedToRetrieveManagedPassword "U01PARVMNDS01$" `
    -Path "OU=ServiceAccounts,OU=PAR,OU=U01,OU=UnreadLines,DC=corp,DC=unreadlines,DC=com"
```

`-Name` never carries the trailing `$` — Active Directory adds it to `sAMAccountName` automatically
(`gmsa-ndes$`, matching `reference/naming-conventions.md` §5.4).

## 7. Install the gMSA on the target host

**On `U01PARVMNDS01`:**

```powershell
Install-ADServiceAccount -Identity "gmsa-ndes"
```

Requires the Active Directory PowerShell RSAT feature on `U01PARVMNDS01`
(`Install-WindowsFeature RSAT-AD-PowerShell` if not already present).

## 8. Verify

**On `U01PARVMNDS01`:**

```powershell
Test-ADServiceAccount -Identity "gmsa-ndes"
```

`True` confirms `U01PARVMNDS01` can retrieve and use `gmsa-ndes$`'s current password. `False` most often
means the KDS root key from `configure-kds-root-key-for-gmsa/README.md` is not effective yet, or
`U01PARVMNDS01$` is missing from `-PrincipalsAllowedToRetrieveManagedPassword`.

## 9. Expected final state

- `Get-ADServiceAccount gmsa-ndes` exists in `OU=ServiceAccounts,OU=PAR,OU=U01,OU=UnreadLines`.
- `Test-ADServiceAccount -Identity "gmsa-ndes"` returns `True` when run on `U01PARVMNDS01`.
- No other host can retrieve `gmsa-ndes$`'s password.

## 10. Next step — use the account

`ndes-scep-intune-connector/README.md` picks this account up to run the NDES service and the Intune
Certificate Connector.

## 11. References

- [Group Managed Service Accounts overview — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/group-managed-service-accounts-overview)
- [Manage Group Managed Service Accounts — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
