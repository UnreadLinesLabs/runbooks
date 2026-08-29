# UnreadLines — Virtual Machine Inventory

> Scope: the authoritative list of every virtual machine deployed across the UnreadLines Labs infrastructure — name, role, network placement, IP address, gateway, and status. Naming follows `server-naming-convention.md`; the two-subnet / two-OpenWrt lab topology is defined in `openwrt-wireguard-site-to-site-lab/README.md`.
>
> Update this file whenever a VM is created, renamed, re-IPed, retired, or its role changes — per `server-naming-convention.md` §2 ("Record the assigned name, role, site, IP address, and owner in the infrastructure inventory").

---

## 1. Hypervisor hosts (not VMs)

| Host | Role | Wi-Fi (home network) | VMnet2 (lab bridge) |
| --- | --- | --- | --- |
| PC1 | Hosts Subnet 1 lab (`FWL01`, `DOM01`, `ADM01`) | `192.168.1.29` | `192.168.20.1/25` |
| PC2 | Hosts Subnet 2 lab (`FWL02`, `PKI01`, `PKI02`, `WEB01`, `RAP01`) | `192.168.1.119` | `192.168.20.129/25` |

## 2. Virtual machines

| VM name | Role | Subnet | IPv4 | Gateway | OS | Domain-joined | Status | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `U01PARVMFWL01` | Router / firewall (OpenWrt) | Subnet 1 LAN / home Wi-Fi WAN | LAN `192.168.20.126/25` · WAN `192.168.1.250/24` | Freebox `192.168.1.254` | OpenWrt | N/A | Running | NAT + firewall for Subnet 1; WireGuard endpoint to `FWL02` |
| `U01PARVMFWL02` | Router / firewall (OpenWrt) | Subnet 2 LAN / home Wi-Fi WAN | LAN `192.168.20.254/25` · WAN `192.168.1.251/24` · VMnet3 `192.168.21.1/25` (`eth2`) | Freebox `192.168.1.254` | OpenWrt | N/A | Running | NAT + firewall for Subnet 2; WireGuard endpoint to `FWL01`; dedicated `VMnet3` uplink (`eth2`) to `RAP01` for the Wi-Fi segment — see §3 and the hostapd Wi-Fi AP lab §9 (802.1Q VLAN tagging over `VMnet2` was tried and abandoned; `VMnet3` is a separate host-only network, not a VLAN) |
| `U01PARVMDOM01` | AD DS / DNS | Subnet 1 (`192.168.20.0/25`) | `192.168.20.41/25` | `192.168.20.126` | Windows Server | Yes (DC) | Running | Forest/domain `corp.unreadlines.com` |
| `U01PARVMADM01` | Administration / clean validation client | Subnet 1 (`192.168.20.0/25`) | `192.168.20.45/25` | `192.168.20.126` | Windows | Yes | Running | PKI validation client |
| `U01PARVMNPS01` | Network Policy Server (RADIUS) | Subnet 1 (`192.168.20.0/25`) | `192.168.20.46/25` | `192.168.20.126` | Windows Server | Yes | Planned | RADIUS for `UnreadLines-Mobile` (802.1X EAP-TLS); RADIUS client is `RAP01` (Subnet 2), reachable via the FWL01↔FWL02 WireGuard tunnel; NPS server certificate pending — see the `radius-nps-deployment` lab |
| `U01PARVMPKI01` | Offline Standalone Root CA | Subnet 2 (`192.168.20.128/25`) | `192.168.20.142/25` | `192.168.20.254` | Windows Server | No (standalone) | Off except controlled operations | Never expose beyond console/mRemoteNG during signing |
| `U01PARVMPKI02` | Online Enterprise Issuing CA | Subnet 2 (`192.168.20.128/25`) | `192.168.20.143/25` | `192.168.20.254` | Windows Server | Yes | Running | Issues user/computer/server certificates |
| `U01PARVMWEB01` | HTTP CRL/AIA Web Distribution Point | Subnet 2 (`192.168.20.128/25`) | `192.168.20.144/25` | `192.168.20.254` | Windows Server | Yes | Running | Publishes `pki.corp.unreadlines.com` |
| `U01PARVMRAP01` | Wireless Access Point (`hostapd`, USB Wi-Fi passthrough) | Subnet 2 (`192.168.20.128/25`); second NIC on `VMnet3` | `192.168.20.145/25` (management, `ens33`); no IP on `ens37`/`VMnet3` | `192.168.20.254` | Ubuntu/Debian | No | Running | SSID#1 (`UnreadLines-Guest`, WPA2-PSK + DHCP) validated end-to-end: association, DHCP lease, Internet. Bridges the Wi-Fi radio to `VMnet3` (`br-vlan10`, member `ens37`) — a dedicated VMware network, not a VLAN trunk. `UnreadLines-Mobile` (802.1X EAP-TLS) pending the NPS server certificate — see the hostapd Wi-Fi AP lab §13 and the `radius-nps-deployment` lab |

## 3. Wi-Fi client networks (behind `RAP01` / `FWL02` — not VMs)

Each SSID gets its own dedicated VMware host-only network between `RAP01` and `FWL02` (`VMnet3`, `VMnet4`, …) — **not** an 802.1Q VLAN. An earlier design tried VLAN-tagging both SSIDs over the existing `VMnet2` trunk; it was abandoned after testing showed VMware Workstation doesn't relay tagged frames between VMs on that host-only network. See the hostapd Wi-Fi AP lab §9 for the full diagnosis.

| Network | Purpose | Subnet | Gateway/DHCP | Status |
| --- | --- | --- | --- | --- |
| `VMnet3` | SSID#1 — WPA2/WPA3-Personal (`UnreadLines-Guest`) | `192.168.21.0/25` | `FWL02` (`eth2`, `192.168.21.1`) | Active |
| `VMnet4` (planned) | `UnreadLines-Mobile` — WPA2/WPA3-Enterprise (802.1X EAP-TLS) | `192.168.21.128/25`* | `FWL02` (new interface, not yet created) | Reserved, not yet active |

*`hostapd-wifi-access-point-lab/README.md` §13 gives `192.168.22.0/25` as its own example subnet for this network — reconcile the two before `VMnet4` is actually built, this table hasn't been updated to match yet.

Full deployment detail: see the hostapd Wi-Fi AP lab, the `radius-nps-deployment` lab (RADIUS/NPS server), and, for the future certificate-based SSID, `README-Intune-SCEP-WiFi-EAP-TLS.md`.

## 4. Next free addresses

| Subnet | Range | Next free |
| --- | --- | --- |
| Subnet 1 (`192.168.20.0/25`) | `.1`–`.126` | `.47` |
| Subnet 2 (`192.168.20.128/25`) | `.129`–`.254` | `.146` |
| SSID#1 / `VMnet3` (`192.168.21.0/25`) | `.1`–`.126` | static range `.1`–`.9` (`.1` = `FWL02` `eth2`, `.2` = `RAP01` `br-vlan10`, optional/diagnostic); DHCP pool `.50`–`.99` (see hostapd lab) |
| `UnreadLines-Mobile` / `VMnet4` (`192.168.21.128/25`, reserved — see note above) | `.129`–`.254` | static range `.129`–`.137`; DHCP pool `.178`–`.228` (reserved, not yet built) |
