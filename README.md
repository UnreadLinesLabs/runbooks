# infra-cloud-labs
Cloud and infrastructure labs — hypervisors, networking, Linux/Windows systems

## Labs

- [Prepare Windows and Ubuntu virtual machines as templates](prepare-windows-ubuntu-templates/README.md) — create reusable Windows and Ubuntu golden images for cloning and deployment.
- [Deploy the first Active Directory domain controller](active-directory-domain-controller/README.md) — create the first AD forest, domain controller, and DNS service.
- [Deploy a Linux hostapd Wi-Fi access point with VLAN-segmented SSIDs](hostapd-wifi-access-point-lab/README.md) — build `U01PARVMRAP01`, bridge two SSIDs over 802.1Q VLANs into `FWL02`, and hand each SSID its own DHCP scope and firewall zone.
- [Deploy a Windows NPS RADIUS server for 802.1X Enterprise Wi-Fi](radius-nps-deployment/README.md) — build `U01PARVMNPS01`, register it in AD, and configure the RADIUS client/Network Policy that will authenticate `UnreadLines-Mobile`.
- [Issue the NPS server certificate and activate EAP-TLS for `UnreadLines-Mobile`](nps-server-certificate-deployment/README.md) — publish a Server Authentication template on `U01PARVMPKI02`, enroll it on `U01PARVMNPS01`, and bind it into the `UnreadLines-Mobile - EAP-TLS` Network Policy.

## Standards

- [Server naming convention](standards/server-naming-convention.md) — define consistent names for virtual infrastructure servers.
- [Active Directory + Microsoft Entra ID — hybrid identity design](standards/active-directory-entra-identity-design.md) — domain architecture, UPN suffix strategy, and Entra Connect sync design.
- [Virtual machine inventory](standards/vm-inventory.md) — authoritative list of every lab VM: name, role, IP, gateway, and status.
