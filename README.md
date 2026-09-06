# UnreadLines Labs — Runbooks

Reproducible, step-by-step enterprise infrastructure, identity, and security labs, built on the
fictional company **UnreadLines** (`unreadlines.com`). Every runbook is written to be followed end to
end without watching the matching video.

House rules for writing one are in [CONTRIBUTING.md](CONTRIBUTING.md).

## Runbooks

Grouped by theme. Within each group, entries are ordered by dependency — later runbooks build on
earlier ones. `openwrt-wireguard-site-to-site` is the foundation: it defines the two subnets every
other runbook places its servers in.

### Platform and virtualization

- [Create a Broadcom account](create-broadcom-account/README.md) — register on the Broadcom Support Portal and verify access by downloading a free VMware product.
- [Prepare Windows and Ubuntu virtual machines as templates](prepare-windows-ubuntu-templates/README.md) — strip machine-specific identity from a base VM and turn it into a reusable golden image.

### Networking

- [Build an OpenWrt network lab with a WireGuard site-to-site tunnel](openwrt-wireguard-site-to-site/README.md) — run an OpenWrt router on each of two physical PCs under VMware Workstation Pro, and join their `/25` LAN subnets over WireGuard.
- [Deploy a Linux `hostapd` Wi-Fi access point](hostapd-wifi-access-point/README.md) — build `U01PARVMRAP01`, bridge its radio onto a dedicated VMware network, and hand the SSID its own DHCP scope and firewall zone on `FWL02`.
- [Add a wired interconnect between the two lab routers](openwrt-wired-site-to-site/README.md) — measure a real packet-loss problem on the Wi-Fi-bridged WireGuard tunnel, then add a switchable, wired alternative path between the two LAB subnets.

### Identity and PKI

- [Deploy the first Active Directory domain controller](active-directory-domain-controller/README.md) — create the `corp.unreadlines.com` forest, its first domain controller, and the DNS service that comes with it.
- [Create the KDS root key for Group Managed Service Accounts (gMSA)](configure-kds-root-key-for-gmsa/README.md) — create and verify the forest's KDS root key, the one-time prerequisite every gMSA depends on.
- [Create a Group Managed Service Account (gMSA)](create-gmsa-account/README.md) — create and install a gMSA with least-privilege password retrieval, using `gmsa-ndes$` on `U00PARVMNDS01` as the worked example.
- [Deploy Microsoft Entra Connect and synchronize Active Directory with Microsoft Entra ID](microsoft-entra-connect-sync/README.md) — build `U01PARVMECN01`, verify the hybrid identity design already in place, and turn on Password Hash Sync between `corp.unreadlines.com` and the `unreadlines` Entra tenant.
- [Deploy an AD CS PKI](ad-cs-pki-deployment/README.md) — stand up an offline Standalone Root CA, an online Enterprise Issuing CA, and an independent HTTP CRL/AIA distribution point, validated end to end with a real leaf certificate.

### Network access control

- [Deploy a Windows NPS RADIUS server for 802.1X enterprise Wi-Fi](radius-nps-deployment/README.md) — build `U01PARVMNPS01`, register it in AD, and configure the RADIUS client and Network Policy that will authenticate `UnreadLines-Mobile`.
- [Issue the NPS server certificate and activate EAP-TLS for `UnreadLines-Mobile`](nps-server-certificate-deployment/README.md) — publish a Server Authentication template on `U01PARVMPKI02`, enroll it on `U01PARVMNPS01`, and bind it into the `UnreadLines-Mobile - EAP-TLS` Network Policy.

### Device management

- [Deploy Intune trusted certificate profiles for the UnreadLines CA hierarchy](intune-trusted-certificate-profiles/README.md) — publish Root and Issuing CA trust profiles for Windows, Android Enterprise (Corporate and BYOD) and iOS/iPadOS, each scoped to its own dynamic Entra ID group.
- [Deploy the SCEP certificate template, NDES, and the Intune Certificate Connector](ndes-scep-intune-connector/README.md) — build `U00PARVMNDS01`, publish the `Intune SCEP Mobile User` template on `U01PARVMPKI02`, and install the Certificate Connector that lets Intune request certificates from it.

## Reference

Conventions, design decisions and inventories that cut across every runbook above. Check them before
inventing a name or an address.

- [Naming conventions](reference/naming-conventions.md) — naming for servers, accounts, groups, organizational units, cloud identity objects (Entra ID / Intune) and Azure resources.
- [Active Directory + Microsoft Entra ID — hybrid identity design](reference/active-directory-entra-identity-design.md) — domain architecture, UPN suffix strategy, and Entra Connect sync design.
- [Virtual machine inventory](reference/vm-inventory.md) — the authoritative list of every lab VM: name, role, IP, gateway and status.

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
