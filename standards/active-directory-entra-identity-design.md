# Active Directory + Microsoft Entra ID — Design d'identité hybride UnreadLines

> **Objectif**
> Définir l'architecture de domaines et la stratégie d'identité hybride entre Active Directory Domain Services (AD DS), Microsoft Entra ID et Microsoft 365 : quel domaine porte quoi, comment le suffixe UPN est configuré et synchronisé, et dans quel ordre déployer.
>
> **Périmètre** : ce document couvre uniquement l'architecture AD DS / DNS / Entra ID et la synchronisation. Les règles de nommage elles-mêmes (format des comptes utilisateurs/admin, comptes de service, groupes, OU, serveurs) sont centralisées dans `server-naming-convention.md`, référencé ci-dessous à chaque fois que c'est pertinent.

---

## 1. Architecture cible

| Domaine | Rôle |
|---|---|
| `corp.unreadlines.com` | Domaine AD DS interne (infrastructure) |
| `id.unreadlines.com` | Namespace d'identité / authentification (UPN) |
| `unreadlines.com` | Domaine public / messagerie |
| `<tenant>.onmicrosoft.com` | Domaine natif du tenant Microsoft |

```text
corp.unreadlines.com     → Infrastructure Active Directory
id.unreadlines.com       → Identité / authentification (UPN)
unreadlines.com          → Communication / adresse e-mail publique
<tenant>.onmicrosoft.com → Domaine natif Microsoft Entra / M365
```

Le domaine AD reste un namespace technique interne ; il n'est pas nécessaire que les utilisateurs se connectent avec `user@corp.unreadlines.com`. C'est un **Alternative UPN suffix** (`id.unreadlines.com`) qui porte l'identité de connexion, indépendamment du nom DNS du domaine AD.

**Exemple de référence** (repris dans tout le document — format de compte détaillé dans `server-naming-convention.md` §5) :

| Attribut | Valeur |
|---|---|
| Nom | Jean Dupont |
| `sAMAccountName` | `u783476512` |
| `userPrincipalName` (AD et Entra) | `u783476512@id.unreadlines.com` |
| `mail` | `jean.dupont@unreadlines.com` |

---

## 2. Pourquoi séparer identité de connexion et adresse e-mail

Une convention du type `prenom.nom@unreadlines.com` comme identifiant de connexion est facile à reconstruire à partir d'informations publiques (site web, LinkedIn, signatures d'e-mail), ce qui facilite le username enumeration, le password spraying, le credential stuffing et le phishing ciblé.

Avec `u783476512@id.unreadlines.com`, le login n'est plus directement déductible du nom et prénom.

**⚠️ Ce n'est qu'une mesure de défense en profondeur.** Elle ne remplace pas : MFA, passkeys/FIDO2, Windows Hello for Business, Conditional Access, Smart Lockout, Microsoft Entra ID Protection, Password Protection, politiques de mots de passe, séparation des comptes privilégiés (voir §7).

Le format exact de l'identifiant (préfixe `u`/`a`/`c` + 9 chiffres) et la taxonomie complète des types de comptes sont définis dans `server-naming-convention.md` §5.

---

## 3. Est-ce une bonne pratique Microsoft ?

Il faut distinguer **support Microsoft** et **recommandation Microsoft par défaut**.

Microsoft recommande généralement un UPN simple, souvent aligné avec l'adresse de messagerie principale (`jean.dupont@unreadlines.com` pour l'UPN comme pour le SMTP). Le design UnreadLines — UPN `u783476512@id.unreadlines.com` distinct du SMTP `jean.dupont@unreadlines.com` — est un **choix d'architecture volontaire**, pas la recommandation par défaut.

Il reste supporté, à condition que :

- `id.unreadlines.com` soit un domaine vérifié dans Entra ID ;
- `userPrincipalName` reste correctement renseigné dans AD et synchronisé tel quel vers Entra ;
- les applications utilisées soient testées avec cette séparation UPN / SMTP (voir §8.1) ;
- l'organisation documente clairement la différence entre login et adresse e-mail.

À présenter comme : *« Microsoft recommande généralement une identité utilisateur simple, souvent alignée avec l'adresse SMTP principale. UnreadLines choisit volontairement un namespace d'identité séparé afin de ne pas rendre l'identifiant de connexion directement déductible depuis l'adresse e-mail publique. »* — pas comme une recommandation Microsoft officielle.

---

## 4. Attributs : ne pas confondre `sAMAccountName`, UPN, `mail` et `proxyAddresses`

| Attribut | Exemple | Rôle |
|---|---|---|
| `sAMAccountName` | `u783476512` | Identifiant legacy AD (`UNREADLINES\u783476512`) |
| `userPrincipalName` | `u783476512@id.unreadlines.com` | Identité moderne de connexion ; source normale de l'UPN Entra via Entra Connect |
| `mail` | `jean.dupont@unreadlines.com` | Adresse e-mail associée à l'utilisateur |
| `proxyAddresses` | `SMTP:jean.dupont@unreadlines.com`<br>`smtp:j.dupont@unreadlines.com` | `SMTP:` (majuscules) = adresse principale ; `smtp:` (minuscules) = alias secondaires |

---

## 5. Domaines à configurer dans Microsoft Entra

Avant toute synchronisation AD → Entra, le tenant doit contenir a minima :

```text
<tenant>.onmicrosoft.com   (déjà présent nativement)
unreadlines.com
id.unreadlines.com
```

L'ordre n'est pas interchangeable : `unreadlines.com` doit être ajouté et vérifié **avant** `id.unreadlines.com`, puisque ce dernier dépend du domaine racine déjà vérifié pour se vérifier automatiquement.

> ⚠️ Ne pas confondre avec `admin.microsoft.com`, qui est le **Microsoft 365 admin center** — un portail différent. La gestion des domaines Entra ID se fait sur `entra.microsoft.com`.

**Étape 1 — Ajouter et vérifier `unreadlines.com` en premier** : [Microsoft Entra admin center](https://entra.microsoft.com) (`entra.microsoft.com`) → *Entra ID → Domain names → Custom domain names → Add* → saisir `unreadlines.com`. Microsoft fournit un enregistrement DNS (généralement TXT) à ajouter chez le fournisseur DNS public, puis *Verify*. Ne pas passer à l'étape 2 tant que ce domaine n'affiche pas *Verified*.

**Étape 2 — Ajouter `id.unreadlines.com` une fois l'étape 1 vérifiée** : même chemin, *Custom domain names → Add* → saisir `id.unreadlines.com`. Comme c'est un sous-domaine d'un domaine racine déjà vérifié dans le même tenant, il se vérifie généralement automatiquement, sans TXT supplémentaire. Résultat attendu :

```text
<tenant>.onmicrosoft.com    Verified
unreadlines.com             Verified
id.unreadlines.com          Verified
```

**Domaine primaire (Primary domain)** : retenu comme `id.unreadlines.com`, car c'est le suffixe proposé par défaut à la création d'un nouvel utilisateur cloud, cohérent avec le namespace d'authentification. Changer le Primary domain ne renomme pas automatiquement les utilisateurs existants.

`unreadlines.com` reste néanmoins un domaine personnalisé vérifié à part entière, utilisé pour M365, Exchange Online, adresses e-mail, groupes, applications — aucune contradiction à avoir `id.unreadlines.com` comme identity namespace et `unreadlines.com` comme SMTP namespace en parallèle.

Documentation : [Gestion des domaines personnalisés Entra](https://learn.microsoft.com/en-us/entra/identity/users/domains-manage)

---

## 6. Suffixe UPN dans Active Directory et migration des comptes

Le domaine AD reste `corp.unreadlines.com` — **il ne faut pas renommer la forêt**. On ajoute simplement `id.unreadlines.com` comme suffixe UPN alternatif.

**GUI** : *Active Directory Domains and Trusts → clic droit → Properties → Alternative UPN suffixes → Add → `id.unreadlines.com`*

**PowerShell** :

```powershell
Set-ADForest -Identity "corp.unreadlines.com" `
    -UPNSuffixes @{Add="id.unreadlines.com"}

# Vérification
Get-ADForest | Select-Object -ExpandProperty UPNSuffixes
```

### 6.1 Ancien suffixe `unreadlines.com`

Ajouter `id.unreadlines.com` ne modifie **aucun compte automatiquement** : `sAMAccountName`, `userPrincipalName`, `mail` et `proxyAddresses` restent inchangés tant qu'on ne les modifie pas explicitement — ce qui permet une migration contrôlée. Conserver temporairement `unreadlines.com` dans la liste des suffixes UPN pendant la transition ; ne pas le supprimer avant d'avoir vérifié tous les comptes et dépendances.

### 6.2 Modifier un utilisateur existant

```powershell
Set-ADUser u783476512 -UserPrincipalName "u783476512@id.unreadlines.com"

Get-ADUser u783476512 -Properties UserPrincipalName,mail |
    Select-Object SamAccountName,UserPrincipalName,mail
```

Résultat attendu :

```text
SamAccountName    : u783476512
UserPrincipalName : u783476512@id.unreadlines.com
mail              : jean.dupont@unreadlines.com
```

**Ne pas modifier l'adresse `mail`** lors du changement d'UPN : le changement concerne l'identité de connexion, pas l'adresse publique `jean.dupont@unreadlines.com`.

### 6.3 Modification en masse — toujours tester en dry-run d'abord

```powershell
$Users = Get-ADUser -Filter 'UserPrincipalName -like "*@unreadlines.com"' `
    -Properties UserPrincipalName

foreach ($User in $Users) {
    $Prefix = $User.UserPrincipalName.Split("@")[0]
    $NewUPN = "$Prefix@id.unreadlines.com"
    Write-Host "$($User.UserPrincipalName) -> $NewUPN"

    # Décommenter seulement après validation du dry-run
    # Set-ADUser $User -UserPrincipalName $NewUPN
}
```

### 6.4 Commandes de contrôle

```powershell
# Lister les utilisateurs
Get-ADUser -Filter * -Properties UserPrincipalName,mail |
    Select-Object SamAccountName,UserPrincipalName,mail

# Filtrer les UPN utilisant encore l'ancien suffixe
Get-ADUser -Filter * -Properties UserPrincipalName |
    Where-Object { $_.UserPrincipalName -like "*@unreadlines.com" } |
    Select-Object SamAccountName,UserPrincipalName

# Lister les utilisateurs déjà sur le nouveau namespace
Get-ADUser -Filter * -Properties UserPrincipalName |
    Where-Object { $_.UserPrincipalName -like "*@id.unreadlines.com" } |
    Select-Object SamAccountName,UserPrincipalName
```

---

## 7. Synchronisation avec Microsoft Entra Connect

La synchronisation n'a pas encore été lancée — les domaines et UPN peuvent donc être préparés avant la première synchronisation.

```text
AD : userPrincipalName = u783476512@id.unreadlines.com
        ↓  Microsoft Entra Connect
Entra : userPrincipalName = u783476512@id.unreadlines.com
```

**Attribut source UPN** : garder `userPrincipalName` (recommandation Microsoft par défaut). Il n'est pas nécessaire d'utiliser l'e-mail comme *Alternate Login ID* — ce serait d'ailleurs incohérent avec l'objectif de ne pas exposer le login via l'adresse publique (voir aussi le point de vigilance §8.4).

**Pourquoi `id.unreadlines.com` doit être vérifié avant la synchronisation** : Entra ID exige que le suffixe UPN synchronisé corresponde à un domaine personnalisé vérifié. Si un compte AD porte `u783476512@id.unreadlines.com` mais que `id.unreadlines.com` n'est pas vérifié dans le tenant, Entra peut substituer le suffixe par le domaine natif (`u783476512@<tenant>.onmicrosoft.com`).

**Quels types de comptes se synchronisent** (taxonomie complète des types dans `server-naming-convention.md` §5) :

| Type | Synchronisation |
|---|---|
| Utilisateur standard (`u…`) | Généralement synchronisé AD → Entra |
| Admin AD (`a…`) | Peut rester on-prem uniquement si aucun besoin cloud |
| Admin Cloud (`c…`) | Peut être créé directement dans Entra, cloud-only |
| Break-glass (`bg-…`) | Doit rester cloud-only, exclu de Conditional Access (§8.3) |
| gMSA | Compte AD technique, pas un utilisateur Entra classique |

---

## 8. Points de vigilance à trancher avant la mise en production

### 8.1 Impact sur les applications tierces et l'Exchange hybride

La séparation UPN ≠ SMTP doit être testée explicitement, pas seulement supposée compatible. Risques connus à vérifier :
- Autodiscover Outlook historique basé sur un SCP qui suppose souvent UPN = e-mail dans certains scénarios hybrides ;
- connecteurs SAML/SCIM tiers qui font l'hypothèse UPN = adresse e-mail pour le provisioning ;
- fédération Teams/Skype legacy ou clients VPN qui préremplissent le login depuis l'adresse e-mail de l'annuaire.

Recommandation : constituer une liste des applications SSO/SAML/SCIM en place et valider explicitement chacune avec le nouveau design avant la généralisation.

### 8.2 Comptes synchronisés vs cloud-only — cohérence avec la taxonomie

S'assurer que les exclusions de synchronisation (admins AD non nécessaires dans le cloud, comptes de service) sont bien reflétées dans le scoping Entra Connect (OU filtering ou règles de synchronisation), pas seulement documentées.

### 8.3 Conditional Access et comptes break-glass

Les comptes break-glass doivent être **explicitement exclus de toutes les policies Conditional Access**. Sans cette exclusion, un break-glass account peut se retrouver bloqué au moment précis où il est censé servir.

### 8.4 Expérience utilisateur du login opaque

Faire taper `u783476512@id.unreadlines.com` (au lieu d'une adresse mémorisable) dégrade l'expérience de connexion sur les surfaces non couvertes par SSO/Windows Hello : portails web tiers, VPN, profils Wi-Fi saisis manuellement, support téléphonique. Le choix de ne pas utiliser l'e-mail comme *Alternate Login ID* est cohérent avec l'objectif de sécurité, mais ce compromis UX devrait être assumé explicitement, avec un plan de mitigation (auto-provisioning via profils MDM/Intune, SSO fédéré partout où c'est possible, gestionnaire de mots de passe d'entreprise).

### 8.5 Procédure de rollback UPN

Actuellement absente : définir le scénario qui déclenche un rollback, la commande (`Set-ADUser -UserPrincipalName <ancien>`), le périmètre concerné, et la re-synchronisation Entra Connect qui doit suivre.

---

## 9. Sécurité des comptes privilégiés et limites de l'UPN opaque

**Comptes privilégiés** (`a…`, `c…`) — protections renforcées à envisager : MFA forte, passkeys/FIDO2, Windows Hello for Business, Conditional Access, Privileged Identity Management, postes d'administration dédiés, restrictions de connexion, absence de boîte mail et de navigation Internet, moindre privilège, journalisation renforcée, surveillance des connexions à risque.

**Ce que l'UPN opaque ne remplace pas.** Rendre le login moins déductible (`u783476512@id.unreadlines.com`) ne protège pas seul contre : vol de session, phishing, AiTM, malware, compromission endpoint, token theft, mauvaises configurations Conditional Access, mots de passe faibles, MFA fatigue, ou compromission d'un administrateur. C'est une couche supplémentaire, pas le contrôle principal.

---

## 10. Checklist et ordre de déploiement

**Vérifications avant Entra Connect :**

- [ ] `corp.unreadlines.com` fonctionne correctement comme domaine AD DS ; DNS et réplication AD sains
- [ ] `unreadlines.com` ajouté et vérifié dans Entra
- [ ] `id.unreadlines.com` ajouté et vérifié dans Entra
- [ ] `id.unreadlines.com` ajouté comme suffixe UPN alternatif AD
- [ ] tous les utilisateurs pilotes utilisent `@id.unreadlines.com`
- [ ] les adresses `mail` restent en `@unreadlines.com`
- [ ] comptes à synchroniser placés dans des OU clairement identifiées ; exclusions définies (admins, comptes de service non nécessaires)
- [ ] groupe pilote d'utilisateurs défini
- [ ] stratégie de matching des éventuels comptes cloud existants vérifiée
- [ ] procédure de rollback UPN documentée (§8.5)
- [ ] break-glass accounts créés et exclus des policies Conditional Access (§8.3)

**Ordre de déploiement :**

```text
01. Vérifier AD DS / DNS
02. Ajouter unreadlines.com dans Entra puis le vérifier
03. Ajouter id.unreadlines.com dans Entra puis le vérifier
04. Définir id.unreadlines.com comme Primary domain (choix retenu)
05. Ajouter id.unreadlines.com comme UPN suffix dans AD
06. Créer/modifier les utilisateurs pilotes : uXXXXXXXXXXX@id.unreadlines.com
07. Vérifier mail = prenom.nom@unreadlines.com
08. Organiser les OU de synchronisation et les exclusions
09. Installer/configurer Microsoft Entra Connect (source UPN = userPrincipalName)
10. Synchroniser le groupe pilote
11. Vérifier les UPN dans Entra, l'authentification, M365, les applications SSO
12. Étendre la synchronisation progressivement
```

---

## 11. Décision d'architecture — résumé

```text
Domaine AD        corp.unreadlines.com
Identité          id.unreadlines.com
E-mail            unreadlines.com
Domaine natif MS  <tenant>.onmicrosoft.com

Utilisateur       u783476512@id.unreadlines.com
E-mail public     jean.dupont@unreadlines.com
```

Cette séparation est volontaire. Elle vise à conserver un domaine AD technique propre, utiliser un namespace d'identité cohérent, ne pas révéler directement le login à partir de l'adresse e-mail publique, et garder une identité stable dans le temps (règles de nommage détaillées dans `server-naming-convention.md`).

```text
QUI est l'utilisateur          →  u783476512  (server-naming-convention.md §5)
COMMENT il s'authentifie       →  u783476512@id.unreadlines.com  (ce document, §6-7)
COMMENT il communique          →  jean.dupont@unreadlines.com
OÙ / QUOI il administre        →  OU + groupes + délégations  (server-naming-convention.md §3-4)
```

Elle doit être accompagnée des contrôles modernes Microsoft d'authentification et de protection des identités (§9) — l'opacité du login n'est qu'une couche parmi d'autres.

---

## 12. Références Microsoft

- [Microsoft Entra Connect — Custom installation](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-custom) — `userPrincipalName` comme attribut normalement utilisé comme UPN ; suffixes UPN = domaines vérifiés dans Entra ; conserver `userPrincipalName` comme source lorsque possible.
- [Microsoft Entra Connect — Design concepts](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/plan-connect-design-concepts) — format `username@domain` ; risque de fallback vers `onmicrosoft.com` si le suffixe n'est pas vérifié.
- [Gestion des domaines personnalisés Entra](https://learn.microsoft.com/en-us/entra/identity/users/domains-manage) — ajout/vérification d'un domaine, gestion du Primary domain, sous-domaines ; changer le Primary domain ne renomme pas les utilisateurs existants.
- [Planification des modifications UPN](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/howto-troubleshoot-upn-changes) — tester les changements UPN, prévoir un rollback, utiliser un périmètre pilote, vérifier le nouveau suffixe avant synchronisation.
