# Create the KDS root key for Group Managed Service Accounts (gMSA)

A Group Managed Service Account's password is generated and rotated by the domain controllers
themselves, derived from a domain-wide secret called the KDS root key. Before the very first gMSA can
be created anywhere in the forest, that key has to exist. This runbook creates it once for
`corp.unreadlines.com` and confirms it is actually usable — nothing here creates a gMSA account itself,
that is `create-gmsa-account/README.md`, which depends on this runbook and picks the same domain back
up.

This lab has a single domain controller, `U01PARVMDOM01` (`active-directory-domain-controller/README.md`).
Microsoft's own guidance for creating this key the production way still means waiting for a real
synchronization window before anything depending on it works, whatever the number of domain controllers
— this runbook follows that production path and the real wait, and separately documents the shortcut
Microsoft describes for a disposable single-DC lab, without using it here.

## 1. Architecture

```text
corp.unreadlines.com (single domain controller in this lab)

  U01PARVMDOM01 (AD DS, DNS — active-directory-domain-controller/README.md)
        │
        │  Group Key Distribution Service (KdsSvc)
        ▼
  CN=Master Root Keys,CN=Group Key Distribution Service,CN=Services,CN=Configuration,
  DC=corp,DC=unreadlines,DC=com
        │
        │  replicated forest-wide (Configuration partition) — every domain controller in
        │  every domain of the forest needs this object before any of them can answer a
        │  gMSA password request
        ▼
  Any future host that installs a gMSA (e.g. U00PARVMNDS01 — create-gmsa-account/README.md)
```

## 2. Scope and dependencies

This runbook creates and verifies the forest's KDS root key — the one-time prerequisite every gMSA
depends on. It does not create any gMSA account: that is `create-gmsa-account/README.md`.

Depends on `active-directory-domain-controller/README.md` — a working `corp.unreadlines.com` forest and
its first domain controller.

## 3. Target configuration

| Item | Value |
| --- | --- |
| Forest / domain | `corp.unreadlines.com` |
| Domain controller | `U01PARVMDOM01` |
| Domain controllers in this lab | 1 (single-DC topology — see §6) |

## 4. Prerequisites

- `U01PARVMDOM01` running Windows Server 2025, promoted per `active-directory-domain-controller/README.md`.
- An account that is a member of `Domain Admins` or `Enterprise Admins`.
- PowerShell run as Administrator, 64-bit — the KDS cmdlets are not available in 32-bit PowerShell.
- The Active Directory PowerShell module, already installed with the AD DS role on `U01PARVMDOM01`.

## 5. Check whether a KDS root key already exists

**On `U01PARVMDOM01`:**

```powershell
Get-KdsRootKey
```

An empty result means no key has ever been created in this forest — continue to §6. Any result means one
already exists: do not create a second one. Deleting and re-creating a KDS root key is not something to
do casually — an old key can stay cached and in use on a domain controller until its KDC service is
restarted, which is out of scope for this runbook.

## 6. Create the KDS root key

**In production (multiple domain controllers):**

```powershell
Add-KdsRootKey -EffectiveImmediately
```

Despite its name, this does not make the key usable straight away. Windows still waits for the key to
finish replicating to every domain controller in the forest before handing it out — Microsoft's own
guidance is to allow up to 10 hours before relying on it. Run the command, then move on to §7 once that
window has passed.

**Lab only, single domain controller — never do this in production:**

```powershell
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
```

This backdates the key's effective time by 10 hours instead of waiting for it, which only makes sense
when there is a single domain controller with nothing else to replicate to. On a real multi-DC forest it
skips the exact safety margin the wait exists for.

`U01PARVMDOM01` is this lab's only domain controller, but this runbook follows the production command and
the real wait — §7 shows how to confirm it is done.

## 7. Verify the key is effective

**On `U01PARVMDOM01`:**

```powershell
Get-KdsRootKey | Format-List KeyId, CreationTime, EffectiveTime
```

The key is usable once `EffectiveTime` is no longer in the future. If it still shows a time ahead of the
current one, the wait from §6 is not over — creating or testing a gMSA before that point fails, typically
with "The request is not supported". Microsoft's own documentation also points at event ID 4004 in the
KDS event log as a second confirmation signal (see §10); `Get-KdsRootKey` is the check this runbook relies
on.

## 8. Expected final state

- `Get-KdsRootKey` returns exactly one key for `corp.unreadlines.com`.
- Its `EffectiveTime` is in the past.
- No gMSA has been created yet — that starts in `create-gmsa-account/README.md`.

## 9. Next step — create a gMSA account

This runbook only unlocks the capability. `create-gmsa-account/README.md` creates and installs the first
gMSA that actually uses it, `gmsa-ndes$`.

## 10. References

- [Create a Key Distribution Service (KDS) Root Key — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/create-the-key-distribution-services-kds-root-key)
- [Get-KdsRootKey — Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/kds/get-kdsrootkey)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
