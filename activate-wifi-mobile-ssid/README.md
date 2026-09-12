# Activate the UnreadLines-Mobile SSID on U01PARVMRAP01

This lab switches `U01PARVMRAP01` from `UnreadLines-Guest` (WPA2-Personal, `hostapd-wifi-access-point/README.md`) to `UnreadLines-Mobile` — WPA2-Enterprise, 802.1X EAP-TLS, authenticated against the RADIUS server built in `radius-nps-deployment/README.md` and `nps-server-certificate-deployment/README.md`. Two things happen here, both required: a new dedicated network (`VMnet4`) with its own firewall zone and DHCP scope on `U01PARVMFWL02`, and the `hostapd` config swap itself. Everything upstream of `hostapd` — the certificate template, NDES, the Application Proxy path, the Intune SCEP and Wi-Fi profiles — is already built and published; this lab is the last piece of infrastructure standing between that chain and a real handshake.

## 1. Architecture

```text
                              Internet
                                 |
                        U01PARVMFWL02 - eth1
                         192.168.1.251/24
                                 |
                             WAN / NAT
                                 |
                 +----------------------+------------------------+
                 |                      |                        |
              VMnet2                 VMnet3                   VMnet4
        LAN 192.168.20.128/25   192.168.21.0/25          192.168.21.128/25
                 |             (UnreadLines-Guest,        (UnreadLines-Mobile,
        +--------+--------+    offline while                this lab)
        |                 |    Mobile is active)                  |
 RAP01 ens33        FWL02 eth0                            +-------+-------+
192.168.20.145/25   br-lan .254                           |               |
  (management)                                      RAP01 ens38    FWL02 eth4
        |                                          (bridge member) 192.168.21.129/25
        |                                                 |
        |                                             br-mobile
        |                                                 |
        |                                          wlx00c0cab23594
        |                                                 |
        |                                              hostapd
        |                                                 |
        |                                     Phone (Intune-managed,
        |                                      EAP-TLS client cert)
        |                                  DHCP 192.168.21.178-228
        |
        |          Subnet 2                                Subnet 1
        |     192.168.20.128/25                       192.168.20.0/25
        +---- U01PARVMRAP01 ------ existing tunnel ---- U01PARVMNPS01
              192.168.20.145        (FWL02 <-> FWL01)     192.168.20.46
              RADIUS client (NAS)                         RADIUS server
              — unchanged by this lab, see §2
```

`VMnet4` is a third dedicated VMware host-only network, following exactly the pattern `VMnet3` already set: not an 802.1Q VLAN (`hostapd-wifi-access-point/README.md` §11 has the diagnosis for why), one `/25` per SSID, its own firewall zone and DHCP scope on `FWL02`. The RADIUS conversation between `RAP01` and `NPS01` does not travel over `VMnet4` at all — it rides `RAP01`'s existing management path (`ens33` → `FWL02` → the FWL01↔FWL02 tunnel → `NPS01`), exactly as it already does today, because that is the IP `NPS01` already has on file for this NAS (§2).

## 2. Scope and dependencies

This lab builds the infrastructure `UnreadLines-Mobile` needs on the network and access-point side:

- A new VMware host-only network, `VMnet4` (`192.168.21.128/25`), attached to both `RAP01` and `FWL02`.
- `RAP01`: a new bridge (`br-mobile`) carrying the new interface, ready to receive the Wi-Fi radio.
- `FWL02`: the `VMnet4` interface, a dedicated firewall zone, and a DHCP scope for Wi-Fi clients.
- `hostapd` switched from `UnreadLines-Guest` to `UnreadLines-Mobile`: WPA2-Enterprise, `ieee8021x=1`, `wpa_key_mgmt=WPA-EAP`, pointed at `U01PARVMNPS01`.

It deliberately does **not**:

- Touch the certificate mapping or the `GG-U01-PAR-WiFi-Mobile` AD group — both already correct as of `intune-scep-certificate-profile/README.md` (step 7 of the Wi-Fi chain). Nothing here changes who is allowed to authenticate.
- Change how `RAP01` talks RADIUS to `NPS01`. The RADIUS client (NAS) `NPS01` already has on file is `RAP01`'s management IP, `192.168.20.145` (`radius-nps-deployment/README.md` §3) — that traffic already crosses the existing FWL01↔FWL02 tunnel today, unrestricted, and continues to do so unchanged. `VMnet4`'s firewall zone cannot filter that conversation and isn't asked to: it never sees it. `VMnet4` carries only the Wi-Fi client segment.
- Open any forwarding rule out of `VMnet4`. Unlike `UnreadLines-Guest`'s zone (which forwards to `wan`), `UnreadLines-Mobile`'s zone starts `forward='REJECT'` with no exception added. A device can associate and get a DHCP lease once EAP-TLS succeeds, but reaches nothing beyond `FWL02` itself until a future lab defines what internal or Internet access managed devices actually get. That is a deliberate scope boundary, not an oversight — see §11.
- Validate a real EAP-TLS handshake in the NPS logs, or run the negative-test matrix (wrong group, revoked certificate, rogue AP, …). That is the next lab in the chain (§11), and it is the point where the certificate mapping and AD group from step 7 are verified for the first time against a real client certificate.
- Build `UnreadLines-Corp`, the workstation-facing enterprise SSID on the same naming plan (`hostapd-wifi-access-point/README.md` §15). Same pattern, later, `VMnet5`.

`RAP01`'s single physical radio (Alfa AWUS1900 / RTL8814AU) cannot run two BSS at once (`hostapd-wifi-access-point/README.md` §5.4) — that is a hardware limit, not a scoping choice. `UnreadLines-Guest` goes offline for the duration this SSID is active; switching back means restoring its `hostapd.conf`, not covered here.

## 3. Target configuration

| Item | Value |
| --- | --- |
| Wi-Fi network | `VMnet4` (new, dedicated), `192.168.21.128/25` |
| `RAP01` → `VMnet4` NIC | `ens38` (no IP; bridge member) — confirm the actual name with `ip -br link` |
| `RAP01` bridge | `br-mobile` (new; separate from `UnreadLines-Guest`'s `br-vlan10`) |
| `RAP01` diagnostic address | `192.168.21.130/25` on `br-mobile` (optional, same pattern as `hostapd-wifi-access-point/README.md` §5.5) |
| `FWL02` → `VMnet4` NIC | `eth4`, `192.168.21.129/25` — confirm the actual name with `ip -br link` |
| SSID | `UnreadLines-Mobile`, WPA2-Enterprise (802.1X, EAP-TLS) |
| `wpa_key_mgmt` | `WPA-EAP` |
| RADIUS server | `U01PARVMNPS01`, `192.168.20.46`, auth UDP `1812` / acct UDP `1813` |
| RADIUS shared secret | `<RadiusSharedSecret>` — must match the secret already on `NPS01`'s RADIUS client entry for `RAP01` (`radius-nps-deployment/README.md`); never commit the real value |
| Client DHCP pool | `192.168.21.178`–`192.168.21.228`, 12h lease |
| Firewall zone (`FWL02`) | `wifi_mobile` — `input='ACCEPT'`, `output='ACCEPT'`, `forward='REJECT'`, no forwarding rule added (§2) |

WPA3-Enterprise (mandatory PMF, `WPA-EAP-SHA256`) is not configured here, for the same reason `UnreadLines-Guest` shipped as WPA2-Personal only despite the "WPA2/WPA3" naming plan in `hostapd-wifi-access-point/README.md` §15: it is a later hardening step, not exercised in this lab.

## 4. Prerequisites

- `U01PARVMRAP01` deployed and reachable, with `UnreadLines-Guest` currently live (`hostapd-wifi-access-point/README.md`).
- `U01PARVMNPS01` up, domain-joined, with the `UnreadLines-Mobile - EAP-TLS` Network Policy configured and its RADIUS client (NAS) entry for `RAP01` already in place (`radius-nps-deployment/README.md`).
- `U01PARVMNPS01`'s server certificate issued and bound to that Network Policy (`nps-server-certificate-deployment/README.md`).
- The Intune SCEP and Wi-Fi EAP-TLS profiles published and targeting `GG-U01-PAR-WiFi-Mobile` (`intune-scep-certificate-profile/README.md`, `intune-wifi-eap-tls-profile/README.md`).
- `192.168.21.128/25` confirmed free, and its static range (`.129`–`.137`) not already allocated — check `reference/vm-inventory.md` §4 at time of writing.
- The RADIUS shared secret already configured on `NPS01`'s client entry for `RAP01`, available to type into `hostapd.conf` — not stored in this repository (§3).

**Not required here**: any change to Active Directory, the AD group, or certificate templates — all already done (steps 4–8 of the Wi-Fi chain).

## 5. Create `VMnet4` in VMware Workstation

Same process as `VMnet3` (`hostapd-wifi-access-point/README.md` §6), a third dedicated host-only network:

```text
Edit → Virtual Network Editor → Add Network → VMnet4
```

Configure it:

```text
Type         : Host-only
Host adapter : disabled
VMware DHCP  : disabled
Subnet IP    : 192.168.21.128
Subnet mask  : 255.255.255.128
```

Click **Apply**. DHCP and the host adapter are disabled for the same reasons as `VMnet3`: `FWL02` is the only DHCP server for this segment, and Windows has no business on the Wi-Fi segment.

**Attach it to both VMs** (shut down `RAP01` and `FWL02` first):

```text
VM Settings → Add → Network Adapter → Custom: VMnet4
```

Don't touch the existing `VMnet2`/`VMnet3` adapters on either VM.

**After both VMs boot, identify the new interface on each:**

```bash
ip -br link
```

This lab assumes `ens38` on `RAP01` and `eth4` on `FWL02` — substitute your actual names if they differ.

## 6. Configure `RAP01`: the `VMnet4` interface

Edit the same Netplan file `hostapd-wifi-access-point/README.md` §5.5 already put in place — add `ens38` and a second bridge alongside the existing `ens37`/`br-vlan10`, without touching either:

**On `U01PARVMRAP01`:**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      match:
        macaddress: 00:0c:29:68:b7:1e   # unchanged — your VM's actual MAC
      set-name: ens33
      addresses:
        - 192.168.20.145/25
      routes:
        - to: default
          via: 192.168.20.254
      nameservers:
        addresses:
          - 192.168.20.41
        search: []
    ens37:
      dhcp4: false
    ens38:
      dhcp4: false
  bridges:
    br-vlan10:
      interfaces: [ens37]
      dhcp4: false
      addresses: [192.168.21.2/25]
    br-mobile:
      interfaces: [ens38]
      dhcp4: false
      # Optional, diagnostic only — same rationale as br-vlan10's address,
      # see hostapd-wifi-access-point/README.md §5.5. Not needed for RADIUS:
      # that conversation still rides ens33 (§2).
      addresses: [192.168.21.130/25]
```

```bash
sudo netplan generate
sudo netplan apply
ip a
```

Expect `ens38` up with no IP, and `br-mobile` up at `192.168.21.130/25` (or with no address if you dropped the diagnostic one). `br-vlan10` and `ens37` are untouched — `UnreadLines-Guest` keeps working until `hostapd` is actually switched (§8).

## 7. Configure `U01PARVMFWL02`: interface, firewall zone, DHCP

SSH to `FWL02` (`192.168.20.254`). All of this is on `eth4` (§5) — a new, independent UCI section, alongside `wifi_ssid1` (`UnreadLines-Guest`), not a replacement of it.

**On `U01PARVMFWL02`, for §7.1 through §7.3:**

### 7.1 Interface

```bash
uci set network.wifi_ssid2='interface'
uci set network.wifi_ssid2.device='eth4'
uci set network.wifi_ssid2.proto='static'
uci set network.wifi_ssid2.ipaddr='192.168.21.129'
uci set network.wifi_ssid2.netmask='255.255.255.128'

uci commit network
/etc/init.d/network reload
```

Verify:

```bash
ip -br addr show dev eth4
ip route show dev eth4
```

Expect `eth4` at `192.168.21.129/25` and a kernel route for `192.168.21.128/25` via `eth4`.

### 7.2 Firewall zone

```bash
uci set firewall.wifi_mobile='zone'
uci set firewall.wifi_mobile.name='wifi_mobile'
uci set firewall.wifi_mobile.network='wifi_ssid2'
uci set firewall.wifi_mobile.input='ACCEPT'
uci set firewall.wifi_mobile.output='ACCEPT'
uci set firewall.wifi_mobile.forward='REJECT'

uci commit firewall
/etc/init.d/firewall restart
```

No forwarding rule is added here — unlike `UnreadLines-Guest`'s `wifi_wan` rule, `wifi_mobile` forwards nowhere yet (§2). `input='ACCEPT'` keeps `FWL02` itself reachable for diagnostics on this segment; tighten it once a real access policy exists.

### 7.3 DHCP and DNS

```bash
uci set dhcp.wifi_ssid2='dhcp'
uci set dhcp.wifi_ssid2.interface='wifi_ssid2'
uci set dhcp.wifi_ssid2.start='50'
uci set dhcp.wifi_ssid2.limit='51'
uci set dhcp.wifi_ssid2.leasetime='12h'

uci add_list dhcp.wifi_ssid2.dhcp_option='3,192.168.21.129'
uci add_list dhcp.wifi_ssid2.dhcp_option='6,192.168.21.129'

uci commit dhcp
/etc/init.d/dnsmasq restart
```

`start='50'` is an offset from the network's base address (`192.168.21.128`), so the pool runs `192.168.21.178`–`192.168.21.228` (`limit='51'`) — matching the range already reserved in `reference/vm-inventory.md` §4. `dhcp_option` `3` (gateway) and `6` (DNS) both point at `FWL02` itself, same pattern as `UnreadLines-Guest`.

Watch it live once a client attempts to join:

```bash
logread -f | grep -i dnsmasq
cat /tmp/dhcp.leases
```

## 8. Switch `hostapd` to `UnreadLines-Mobile`

**This takes `UnreadLines-Guest` offline** — the single radio can only run one BSS (§2).

**On `U01PARVMRAP01`:**

```bash
sudo systemctl stop hostapd
sudo nano /etc/hostapd/hostapd.conf
```

Replace its contents:

```text
interface=wlx00c0cab23594
bridge=br-mobile
driver=nl80211

ssid=UnreadLines-Mobile
country_code=FR

hw_mode=g
channel=6

ieee80211n=1
wmm_enabled=1

auth_algs=1
ieee8021x=1

wpa=2
wpa_key_mgmt=WPA-EAP
rsn_pairwise=CCMP

auth_server_addr=192.168.20.46
auth_server_port=1812
auth_server_shared_secret=<RadiusSharedSecret>

acct_server_addr=192.168.20.46
acct_server_port=1813
acct_server_shared_secret=<RadiusSharedSecret>
```

`bridge=br-mobile` is the one line that actually performs the swap — it moves the radio from `UnreadLines-Guest`'s bridge to this SSID's. `auth_server_shared_secret` (and `acct_server_shared_secret`, same value) must match exactly what `NPS01`'s RADIUS client entry for `RAP01` already has (`radius-nps-deployment/README.md`) — store the real value outside this repository, never commit it. `/etc/default/hostapd`'s `DAEMON_CONF` already points at this same file (`hostapd-wifi-access-point/README.md` §7) — nothing to change there.

Start it in the foreground first, the same way as the initial deployment:

```bash
sudo hostapd -dd /etc/hostapd/hostapd.conf
```

Look for:

```text
wlx00c0cab23594: interface state COUNTRY_UPDATE->ENABLED
wlx00c0cab23594: AP-ENABLED
```

`Ctrl-C`, then restart as a service:

```bash
sudo systemctl restart hostapd
sudo systemctl status hostapd
```

```bash
bridge link show
# expect wlx00c0cab23594 AND ens38 as members of br-mobile, both "state forwarding"
```

## 9. Expected final state

This lab stops short of a real client join — that is §11 / the next lab in the chain. What must be true here:

**On `U01PARVMRAP01`:**

```bash
systemctl status hostapd
# active (running), AP-ENABLED for UnreadLines-Mobile

journalctl -u hostapd -n 20
# no repeated RADIUS timeout / retransmission errors talking to 192.168.20.46
```

```bash
bridge link show
# wlx00c0cab23594 and ens38 both members of br-mobile
```

**On `FWL02`:**

```bash
uci show network.wifi_ssid2
uci show firewall.wifi_mobile
uci show dhcp.wifi_ssid2
ip -br addr show dev eth4
# eth4 at 192.168.21.129/25
```

A device without a valid client certificate that tries to join will be rejected during the EAP exchange — expected, not a fault to chase down here; a real, certificate-bearing device may already succeed at this point since every upstream piece is in place, but confirming that end to end, including the negative cases, is deliberately deferred to §11.

## 10. Update the infrastructure inventory

In `reference/vm-inventory.md`:

- §2 — `U01PARVMRAP01`'s row: `UnreadLines-Mobile` now built and active; `UnreadLines-Guest` offline while it is (hardware limit, §2 of this document).
- §3 — the Wi-Fi client networks table: `VMnet4` status from "Reserved, not yet active" to "Active".
- §4 — next free addresses: `.129` (`FWL02` `eth4`) and `.130` (`RAP01` `br-mobile`, diagnostic) now used from the `.129`–`.137` static range; the `.178`–`.228` DHCP pool now built, not just reserved.

## 11. Next step — validating the real handshake

This lab makes `UnreadLines-Mobile` live and reachable, with every upstream dependency (certificate chain, NPS, Intune profiles) already in place — but it does not prove any of it works together. The next lab in the chain does that: a real EAP-TLS handshake confirmed in the NPS logs, plus an explicit negative-test matrix (wrong group, revoked certificate, expired certificate, rogue AP presenting the wrong server certificate, and more — not just the case that works). That is where the certificate mapping and the `GG-U01-PAR-WiFi-Mobile` group from `intune-scep-certificate-profile/README.md` get their first real test against an actual client certificate.

## 12. References

- [hostapd documentation](https://w1.fi/hostapd/)
- [OpenWrt — DHCP and DNS (dnsmasq)](https://openwrt.org/docs/guide-user/base-system/dhcp)
- [OpenWrt — firewall zones](https://openwrt.org/docs/guide-user/firewall/firewall_configuration)
- [Netplan — bridges](https://netplan.readthedocs.io/en/stable/netplan-yaml/#properties-for-device-type-bridges)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
