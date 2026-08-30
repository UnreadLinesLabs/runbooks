# Deploy a Linux `hostapd` Wi-Fi access point

This lab deploys `U01PARVMRAP01`, a Linux VM that turns a USB Wi-Fi adapter into an 802.11 access point. `RAP01` does not route, NAT, or hand out DHCP — it is a pure Layer 2 bridge between the radio and a dedicated VMware network into `U01PARVMFWL02`, which owns routing, DHCP, and the firewall for the SSID's segment.

**Hardware limitation, confirmed by testing:** the Alfa AWUS1900 (Realtek RTL8814AU) in this lab cannot run two simultaneous BSS/SSID on its single radio — both the in-kernel `rtw88_8814au` driver and the out-of-tree DKMS driver (`morrownr/8814au`) refuse to bring up a second `hostapd` interface (`Device or resource busy`, then `Failed to create interface: -19 (No such device)`). So this lab builds **one SSID only**, end to end. A second SSID (WPA2/WPA3-Enterprise, EAP-TLS, once a RADIUS server exists) is a separate, later piece of work. See `ad-cs-pki-deployment/README.md` for the PKI that architecture depends on; the Intune/SCEP certificate delivery it also needs is not yet covered by a runbook here. If a second physical Wi-Fi adapter ever becomes available, both SSIDs could instead run truly concurrently (one `hostapd` instance per adapter).

**Design note:** the SSID's traffic rides a second, dedicated VMware network (`VMnet3`), untagged — not an 802.1Q VLAN over the existing `VMnet2`. A VLAN-tagged version of this lab was tried first and didn't work in this environment; see §11 for the diagnosis.

## 1. Architecture

```text
                              Internet
                                 |
                        U01PARVMFWL02 - eth1
                         192.168.1.251/24
                                 |
                             WAN / NAT
                                 |
                 +---------------+---------------+
                 |                               |
              VMnet2                          VMnet3
        LAN 192.168.20.128/25          Wi-Fi 192.168.21.0/25
                 |                               |
        +--------+--------+             +--------+--------+
        |                 |             |                 |
 RAP01 ens33        FWL02 eth0    RAP01 ens37       FWL02 eth2
192.168.20.145/25   br-lan .254         |          192.168.21.1/25
  (management)                          |
                                    br-vlan10
                                         |
                                  wlx00c0cab23594
                                         |
                                     hostapd
                                         |
                                   Phone / laptop
                             DHCP 192.168.21.50-99
```

`VMnet2` (Subnet 2, existing) and `VMnet3` (new, dedicated to this lab) are two separate VMware Workstation host-only networks — not two VLANs on the same wire. `RAP01` and `FWL02` each get a second virtual NIC on `VMnet3`; nothing else in the lab is attached to it. `RAP01`'s Wi-Fi radio is bridged to its `VMnet3` NIC (`ens37`) via `br-vlan10`; `FWL02` terminates `VMnet3` directly on `eth2`, with its own IP, DHCP, and firewall zone. No 802.1Q tag is involved anywhere in this design.

## 2. Scope and dependencies

This runbook turns `U01PARVMRAP01` into a working Wi-Fi access point: an Ubuntu/Debian VM with a USB
Wi-Fi adapter passed through, running `hostapd`, bridging its radio onto a dedicated VMware network
(`VMnet3`) that `U01PARVMFWL02` routes, filters and serves DHCP for. It ends with `UnreadLines-Guest`
— WPA2/WPA3-Personal — validated end to end: association, DHCP lease, Internet access.

It builds **one SSID only**, and that is a hardware limit rather than a scoping choice: the adapter used
here cannot run two simultaneous BSS on its single radio. §11 has the diagnosis. The enterprise SSIDs
planned in §15 need a RADIUS server — `radius-nps-deployment/README.md` — and client certificates, which
depend on `ad-cs-pki-deployment/README.md`.

It also does **not** use 802.1Q VLAN tagging, although an earlier design did. §11 explains why that was
abandoned in favour of one dedicated VMware network per SSID, and `reference/vm-inventory.md` §3 records
the resulting per-SSID network plan.

## 3. Target configuration

| Item | Value |
| --- | --- |
| VM name | `U01PARVMRAP01` |
| Role | Wireless Access Point (`hostapd`), single SSID |
| Management network | Subnet 2 / `VMnet2` (`192.168.20.128/25`) |
| Management IP | `192.168.20.145/25` |
| Management gateway | `192.168.20.254` (`FWL02`) |
| Management DNS | `192.168.20.41` (AD DNS) |
| Wi-Fi network | `VMnet3` (new, dedicated), `192.168.21.0/25` |
| RAP01 → VMnet3 NIC | `ens37` (no IP; bridge member) |
| FWL02 → VMnet3 NIC | `eth2`, `192.168.21.1/25` |
| Wi-Fi hardware | Alfa AWUS1900 (Realtek RTL8814AU), USB passthrough — single BSS only, see §5.4 |
| SSID#1 | `UnreadLines-Guest`, WPA2-Personal |
| SSID#1 client pool | `192.168.21.50`–`192.168.21.99`, 12h lease |

Update `reference/vm-inventory.md` once this is deployed, if any of these values change.

## 4. Prerequisites

- `U01PARVMFWL02` already deployed and routing Subnet 2 (see `openwrt-wireguard-site-to-site/README.md`).
- An Alfa AWUS1900 (Realtek RTL8814AU) USB Wi-Fi adapter, with AP mode confirmed via `iw list` (§5.4).
- VMware Workstation configured to pass that USB device through to `RAP01` (Removable Devices → the adapter → Connect to the virtual machine).
- `192.168.20.145` and `192.168.21.0/25` confirmed free (checked against `reference/vm-inventory.md` §4 at time of writing).

## 5. Prepare the Ubuntu/Debian VM (`RAP01`)

Follow `prepare-windows-ubuntu-templates/README.md` for the base install and cleanup, then apply the lab-specific configuration below instead of that guide's generic example values.

### 5.1 Quick recovery commands

If a cloned Ubuntu VM boots without working SSH, the following commands usually restore the service:

```bash
sudo ssh-keygen -A
sudo mkdir -p /run/sshd
sudo chmod 755 /run/sshd
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh
```

### 5.2 Hostname

```bash
sudo hostnamectl set-hostname U01PARVMRAP01
sudo nano /etc/hostname     # confirm it reads U01PARVMRAP01
sudo nano /etc/hosts        # update the 127.0.1.1 line to match
sudo reboot
```

### 5.3 USB Wi-Fi passthrough

1. Shut down the VM.
2. Plug the USB Wi-Fi adapter into the host.
3. Start the VM, then in VMware Workstation: **VM → Removable Devices → \<adapter name\> → Connect (Disconnect from Host)**.
   If SSH isn't reachable after this boot (common right after cloning a template), reconnect via the VMware console and run §5.1 first.
4. Install `iw` — not present on a minimal install, and needed for every wireless check in this lab:

```bash
sudo apt update
sudo apt install iw -y
```

5. Inside the VM, confirm the adapter is visible:

```bash
lsusb
ip link show
iw dev
```

If `lsusb` shows only the VMware virtual mouse/USB hub and root hubs, the adapter is still attached to the host — go back to **VM → Removable Devices** and connect it.

**VMware Workstation does not keep USB passthrough across a VM power cycle.** Every time the VM is shut down, rebooted, or reverted to a snapshot, reconnect the adapter through **VM → Removable Devices** before expecting it to show up in `lsusb`.

### 5.4 Driver — the in-kernel driver is enough

This lab's adapter is an Alfa AWUS1900, Realtek **RTL8814AU** chipset (`lsusb`: `0bda:8813`). Recent kernels (this lab: `7.0.0-30-generic`) ship an in-kernel `rtw88_8814au` driver that binds this chipset natively, with no build step:

```bash
lsusb                # confirm the adapter enumerates: 0bda:8813 Realtek RTL8814AU
ip link show          # look for wlx00c0cab23594 (or wlan0)
iw dev
```

If a wireless interface appears, you're done. Confirm AP mode is listed:

```bash
iw list | grep -A 6 "Supported interface modes"
```

`AP` should be in the list. (`interface combinations are not supported` is expected and harmless — that only matters for running two BSS at once, which this lab doesn't do.)

The regulatory domain (country) does not need to be set at the driver/kernel level for this lab — `hostapd.conf` sets `country_code=FR` directly (§7), which `hostapd` applies via `nl80211` on every start, regardless of which driver is loaded. That's simpler and more reliable than trying to persist an `iw reg set` change or a DKMS modprobe option across reboots.

**Only if no wireless interface appears at all**, fall back to the out-of-tree DKMS driver (a separate kernel module, `8814au`, replacing `rtw88_8814au` — not needed in this lab since the in-kernel driver already worked):

```bash
sudo apt update
sudo apt install -y dkms git build-essential linux-headers-$(uname -r) rfkill

git clone https://github.com/morrownr/8814au.git
cd 8814au
sudo ./install-driver.sh
sudo reboot
```

Reconnect the USB device (§5.3), then confirm `lsmod | grep 8814au` and `iw dev` show the driver and interface.

### 5.5 Network configuration (Netplan)

`RAP01` needs two things configured here: its existing management NIC (`ens33`, unchanged from before), and the new `VMnet3` NIC (`ens37`) enslaved to a bridge that `hostapd` will also attach the Wi-Fi radio to. Add the `VMnet3` adapter to the VM first (§6) — `ens37` won't exist until you do.

A cloned/autoinstalled Ubuntu VM already ships `/etc/netplan/00-installer-config.yaml`, generated by `subiquity` and pinned to the management NIC's MAC address (`match:` / `set-name:`). Edit it in place rather than adding a second netplan file — keep the `match`/`set-name` block subiquity wrote for `ens33`, and add `ens37` and the bridge on top:

```yaml
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      match:
        macaddress: 00:0c:29:68:b7:1e   # match your VM's actual MAC — see `ip a`
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
  bridges:
    br-vlan10:
      interfaces: [ens37]
      dhcp4: false
      # Optional, for diagnostics only — lets RAP01 itself ping FWL02 on the
      # Wi-Fi segment while troubleshooting. Safe to remove once validated;
      # RAP01 doesn't need an address here to do its job (§11 / Security).
      addresses: [192.168.21.2/25]
```

`br-vlan10` is just a name (kept from the earlier VLAN design, §11) — it's a plain, untagged bridge. `ens37` has no IP of its own; the bridge is the Layer 2 meeting point between `ens37` (towards `FWL02`) and `wlx00c0cab23594` (towards the Wi-Fi client), which `hostapd` attaches automatically when it starts (§7).

```bash
sudo netplan generate
sudo netplan apply
ip a
```

You should see `ens33` at `192.168.20.145/25`, `ens37` up with no IP, and `br-vlan10` up at `192.168.21.2/25` (or with no IP if you dropped the diagnostic address).

```bash
ping -c 3 192.168.20.254     # FWL02, via the management IP (VMnet2)
ping -c 3 192.168.20.41      # AD DNS
```

## 6. Create `VMnet3` in VMware Workstation

`VMnet3` is a second, dedicated host-only network — separate from `VMnet2` — carrying only the Wi-Fi segment.

**Create the network:**

```text
Edit → Virtual Network Editor → Add Network → VMnet3
```

Configure it:

```text
Type         : Host-only
Host adapter : disabled
VMware DHCP  : disabled
Subnet IP    : 192.168.21.0
Subnet mask  : 255.255.255.128
```

Click **Apply**.

DHCP is disabled because `FWL02` is the only DHCP server for this segment (§8.3). The host adapter is disabled to keep Windows off the Wi-Fi segment entirely and avoid address conflicts.

**Attach it to both VMs** (shut down `RAP01` and `FWL02` first):

```text
VM Settings → Add → Network Adapter → Custom: VMnet3
```

Do this on both `RAP01` and `FWL02`. Don't touch the existing adapters already on `VMnet2`.

**After both VMs boot, identify the new interface on each:**

```bash
ip -br link
```

In this lab: `ens37` on `RAP01`, `eth2` on `FWL02`. Substitute your actual names if they differ — a VM with several NICs doesn't always number them the same way twice.

## 7. Install and configure `hostapd`

```bash
sudo apt update
sudo apt install hostapd -y
sudo systemctl unmask hostapd
```

`/etc/hostapd/hostapd.conf` — one physical radio, one BSS, bridged to the Wi-Fi segment:

```text
interface=wlx00c0cab23594
bridge=br-vlan10
driver=nl80211

ssid=UnreadLines-Guest
country_code=FR

hw_mode=g
channel=6

ieee80211n=1
wmm_enabled=1

auth_algs=1

wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase=<CHANGE-ME — store outside version control>
rsn_pairwise=CCMP
```

Store the real `wpa_passphrase` outside this repository — a local secrets file or vault, referenced here by placeholder only (8–63 characters for WPA/WPA2). Substitute your adapter's actual interface name if different from `wlx00c0cab23594` (check with `ip link show`).

`driver=nl80211` tells `hostapd` to control the radio through the kernel's modern netlink-based wireless API, rather than the old, deprecated `wext` interface. It's the value used with any `mac80211`-based driver, which is exactly what `rtw88_8814au` (§5.4) is — so this is effectively the only correct choice here, not a pick between alternatives. `country_code=FR` sets the regulatory domain on every start, independent of the driver (§5.4).

Point the service at this config — `/etc/default/hostapd` is an environment file that tells systemd which config to load at boot, not the Wi-Fi config itself:

```bash
sudo nano /etc/default/hostapd
```

Uncomment `DAEMON_CONF` and set it to the config path; leave `DAEMON_OPTS` commented out:

```text
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

(The `deprecated` warning at the top of the file is a generic Debian/Ubuntu notice — `DAEMON_CONF` still works fine and is what this lab uses.) Save (`Ctrl-O`, `Enter`), exit (`Ctrl-X`).

The first time, run `hostapd` in the foreground before trusting the service — errors are much easier to read this way:

```bash
sudo systemctl stop hostapd
sudo hostapd -dd /etc/hostapd/hostapd.conf
```

Look for:

```text
wlx00c0cab23594: interface state COUNTRY_UPDATE->ENABLED
wlx00c0cab23594: AP-ENABLED
```

If instead you see `Could not read interface wlx00c0cab23594 flags: No such device` / `nl80211 driver initialization failed`, the adapter simply isn't present right now — almost always the USB passthrough was lost (§5.3, every VM reboot loses it). Check `lsusb` / `iw dev`, reconnect via **VM → Removable Devices** if needed, and try again.

`Ctrl-C` to stop, then enable it as a service:

```bash
sudo systemctl enable --now hostapd
sudo systemctl status hostapd
```

One warning is expected and harmless every time the service starts:

- `Referenced but unset environment variable evaluates to an empty string: DAEMON_OPTS` — `/etc/default/hostapd` only sets `DAEMON_CONF`; `DAEMON_OPTS` is an optional variable the unit references and leaves empty. No action needed.

Verify it came up bridged correctly:

```bash
journalctl -u hostapd -f
# expect: wlx00c0cab23594: AP-ENABLED

bridge link show
# expect wlx00c0cab23594 AND ens37 as members of br-vlan10, both "state forwarding"
```

`journalctl -u hostapd -f` follows continuously — `Ctrl-C` to get back to a prompt.

## 8. Configure `U01PARVMFWL02`: Wi-Fi interface, firewall, DHCP

SSH to `FWL02` (`192.168.20.254`). All of this is on `eth2` (§6) — no VLAN subinterface, no trunk.

### 8.1 Interface

```bash
uci set network.wifi_ssid1='interface'
uci set network.wifi_ssid1.device='eth2'
uci set network.wifi_ssid1.proto='static'
uci set network.wifi_ssid1.ipaddr='192.168.21.1'
uci set network.wifi_ssid1.netmask='255.255.255.128'

uci commit network
/etc/init.d/network reload
```

Verify:

```bash
ip -br addr show dev eth2
ip route show dev eth2
```

Expect `eth2` at `192.168.21.1/25` and a kernel route for `192.168.21.0/25` via `eth2`.

### 8.2 Firewall zone

```bash
uci set firewall.wifi='zone'
uci set firewall.wifi.name='wifi'
uci set firewall.wifi.network='wifi_ssid1'
uci set firewall.wifi.input='ACCEPT'
uci set firewall.wifi.output='ACCEPT'
uci set firewall.wifi.forward='REJECT'

uci set firewall.wifi_wan='forwarding'
uci set firewall.wifi_wan.src='wifi'
uci set firewall.wifi_wan.dest='wan'

uci commit firewall
/etc/init.d/firewall restart
```

`input='ACCEPT'` makes testing against `192.168.21.1` easy for the lab; harden it once validated (§14).

### 8.3 DHCP and DNS

```bash
uci set dhcp.wifi_ssid1='dhcp'
uci set dhcp.wifi_ssid1.interface='wifi_ssid1'
uci set dhcp.wifi_ssid1.start='50'
uci set dhcp.wifi_ssid1.limit='50'
uci set dhcp.wifi_ssid1.leasetime='12h'

uci add_list dhcp.wifi_ssid1.dhcp_option='3,192.168.21.1'
uci add_list dhcp.wifi_ssid1.dhcp_option='6,192.168.21.1'

uci commit dhcp
/etc/init.d/dnsmasq restart
```

`dhcp_option 3` (gateway) and `6` (DNS) both point at `FWL02` itself (`192.168.21.1`) — Wi-Fi clients use `FWL02` as their resolver, which forwards upstream, rather than the AD DNS server used on the management network.

Watch it live while a client connects:

```bash
logread -f | grep -i dnsmasq
cat /tmp/dhcp.leases
```

Expected sequence: `DHCPDISCOVER` → `DHCPOFFER` → `DHCPREQUEST` → `DHCPACK`, all tagged `(eth2)`.

## 9. Verification

From a phone or laptop, connect to `UnreadLines-Guest` with the configured passphrase.

On `RAP01`, confirm the client associated:

```bash
iw dev wlx00c0cab23594 station dump
# expect: authorized: yes / authenticated: yes / associated: yes
```

On the client, confirm a lease in `192.168.21.0/25` and a working default route:

```text
Expected: IP in 192.168.21.50–.99, gateway 192.168.21.1, DNS 192.168.21.1
ping 192.168.21.1     # FWL02
ping 1.1.1.1           # Internet, through FWL02's existing NAT
```

Then open any website to confirm DNS resolution end-to-end.

## 10. Troubleshooting quick reference

**Phone won't associate to the SSID at all** — confirm `hostapd` is actually up and `AP-ENABLED` (§7), then check for a stuck association:

```bash
iw dev wlx00c0cab23594 station dump
```

**Phone associates but never gets an IP** — check bridge membership first, it's the most common cause and takes one command:

```bash
bridge link show
# must show BOTH:
#   ens37             master br-vlan10 state forwarding
#   wlx00c0cab23594   master br-vlan10 state forwarding
```

Either side can silently fall out of the bridge on its own — not just after a reboot, but during normal operation (observed in this lab: the client authenticated and completed the WPA handshake fine, then kept disassociating on inactivity and retrying, because `wlx00c0cab23594` had dropped out of `br-vlan10` and no traffic — including the client's own DHCP requests — could reach `ens37`/`FWL02` at all). Whichever one is missing, re-add it directly:

```bash
sudo ip link set wlx00c0cab23594 master br-vlan10   # if the radio fell out
sudo ip link set ens37 master br-vlan10               # if the VMnet3 NIC fell out
```

(`sudo netplan apply` also works for `ens37`, since it's declared in Netplan — but the direct `ip link set` above is faster and works for either interface.)

**`ip link set ... master br-vlan10` fails with `Cannot find device`** — the interface isn't just out of the bridge, it's gone from the system entirely. Confirm with `dmesg | tail -40`: a line like `usb 3-2: USB disconnect, device number N` means the adapter genuinely dropped off the USB bus, not a config issue. Reconnect it (§5.3: **VM → Removable Devices**), then restart `hostapd` — it doesn't reliably pick up a newly re-appeared interface on its own:

```bash
lsusb                          # confirm the Realtek adapter is back
sudo systemctl restart hostapd
bridge link show               # confirm both members again
```

Spontaneous USB disconnects like this (unrelated to a VM reboot) have shown up more than once in this lab and were never fully root-caused — switching `rtw_switch_usb_mode` and changing the VM's virtual USB controller (3.1 → 2.0) didn't stop them. Treat it as a known rough edge of USB passthrough under VMware Workstation: reconnect and restart `hostapd` when it happens, rather than something to permanently fix.

If membership is fine but you still see nothing get through, walk the path from the radio outward:

```bash
# on RAP01, at the radio
sudo tcpdump -i wlx00c0cab23594 -nne 'port 67 or port 68 or arp'
# on RAP01, at the VMnet3 NIC
sudo tcpdump -i ens37 -nne 'port 67 or port 68 or arp'
```

**DHCP request reaches `FWL02` but no lease is handed out** — check the DHCP config took effect:

```bash
logread -f | grep -i dnsmasq
cat /tmp/dhcp.leases
uci show dhcp.wifi_ssid1
```

**Client gets an IP but no Internet** — test in order:

```text
ping 192.168.21.1     # should always work — FWL02 itself
ping 1.1.1.1            # tests NAT/routing to the WAN
```

If the first works but not the second, check the firewall zone and forwarding rule exist:

```bash
uci show firewall | grep -E "wifi|wifi_ssid1"
# expect a zone 'wifi' and a forwarding wifi -> wan
```

**Internet works by IP but not by hostname** — confirm the DHCP-advertised DNS server:

```bash
uci show dhcp.wifi_ssid1
# expect: dhcp_option='6,192.168.21.1'
```

## 11. Why not 802.1Q VLAN tagging

An earlier version of this lab carried the SSID's traffic as VLAN 10, 802.1Q-tagged, over the existing `VMnet2` trunk shared with the rest of Subnet 2 (`ens33.10` on `RAP01`, `eth0.10` on `FWL02`). It was abandoned after hands-on testing showed it doesn't work in this environment, and the diagnosis is worth recording so it isn't retried blind later.

Symptom: the phone associated to the SSID and completed the WPA handshake, but never received a DHCP lease — it just span on "obtaining IP address."

Diagnosis, hop by hop with `tcpdump`:

1. `wlx00c0cab23594` (the radio) — the client's `DHCPDISCOVER` was there, retransmitting normally.
2. `br-vlan10` and `ens33.10` (`RAP01`'s own bridge and VLAN subinterface) — a single capture on `-i any -e` showed the same frame crossing all four of `RAP01`'s interfaces in order: `wlx00c0cab23594` (in) → `br-vlan10` (bridged) → `ens33.10` (out, tagged) → `ens33` (out, onto the wire). The frame left the VM correctly tagged for VLAN 10.
3. `eth0` on `FWL02` (the raw physical port, before any VLAN decoding) — **nothing arrived**, not even an untagged frame.

Basic, untagged connectivity between the two VMs on `VMnet2` was confirmed working throughout (`ping` from `RAP01`'s management IP to `FWL02` succeeded every time). So the fault was specific to 802.1Q-tagged frames on this particular VMware Workstation host-only network: they leave the sending VM correctly tagged, but the virtual switch doesn't relay them to the peer VM. Disabling VLAN offload on the sending NIC (`ethtool -K ens33 txvlan off rxvlan off`) was tried as a possible fix and didn't change the outcome either.

Rather than keep chasing a VMware Workstation internals issue, the design was changed to two separate host-only networks instead of one trunk — `VMnet2` for the existing LAN, `VMnet3` dedicated to the Wi-Fi segment (§6). Untagged Ethernet between VMs on the same host-only network is exactly what Workstation is built to do well, and it's what this revision of the lab uses throughout.

## 12. Interfaces summary

**`RAP01`**

| Interface | Role | Address |
| --- | --- | --- |
| `ens33` | Management / `VMnet2` | `192.168.20.145/25` |
| `ens37` | `VMnet3` bridge member | none |
| `br-vlan10` | Wi-Fi segment bridge | `192.168.21.2/25` (optional, diagnostic) |
| `wlx00c0cab23594` | Wi-Fi radio (AP) | none |

**`FWL02`**

| Interface | Role | Address |
| --- | --- | --- |
| `br-lan` / `eth0` | LAN / `VMnet2` | `192.168.20.254/25` |
| `eth1` | WAN | `192.168.1.251/24` |
| `eth2` | Wi-Fi segment / `VMnet3` | `192.168.21.1/25` |
| `wg0` | WireGuard (unrelated to this lab) | `10.10.10.2/30` |

## 13. Checklist

* [ ] `VMnet3` created in VMware Workstation (host-only, VMware DHCP disabled, `192.168.21.0/255.255.255.128`).
* [ ] `VMnet3` adapter added to both `RAP01` and `FWL02`; `ens37` / `eth2` identified.
* [ ] `U01PARVMRAP01` created, hostname set, `192.168.20.145/25` confirmed via `ip a`.
* [ ] USB Wi-Fi adapter passed through and visible (`lsusb`, `iw dev`).
* [ ] `ens37` / `br-vlan10` present in Netplan, applied, no IP conflicts.
* [ ] `hostapd` running, `AP-ENABLED`, both `ens37` and `wlx00c0cab23594` members of `br-vlan10`.
* [ ] `FWL02`: `eth2` at `192.168.21.1/25`.
* [ ] `FWL02` firewall: zone `wifi` forwards to `wan`.
* [ ] `FWL02` DHCP active on `wifi_ssid1`, gateway and DNS both advertised as `192.168.21.1`.
* [ ] SSID#1 tested end-to-end: association, DHCP lease, `ping 192.168.21.1`, `ping 1.1.1.1`, a real website.
* [ ] `reference/vm-inventory.md` updated with the real IPs/status now that this is deployed.

## 14. Security

- Never commit the real `wpa_passphrase` or any WireGuard private key to this repository.
- WPA2 passphrase: 8–63 characters, stored outside version control.
- `firewall.wifi.input='ACCEPT'` is convenient for lab testing; tighten it once the lab is validated, if `RAP01`/`FWL02` don't need to be reachable directly from Wi-Fi clients.
- The diagnostic address on `br-vlan10` (`192.168.21.2/25`, §5.5) is optional — remove it if `RAP01` doesn't need to be reachable on the Wi-Fi segment itself.

## 15. Next steps — enterprise SSIDs (mobile, then workstations)

This adapter can only run **one SSID at a time** (§5.4) — that constraint doesn't go away once EAP-TLS is added, it just moves: whichever SSID is configured in `hostapd.conf` is the only one live, the others are offline until swapped in. So the eventual naming plan, following the `UnreadLines-Guest` pattern already in use:

| SSID | Purpose | Auth | Order |
| --- | --- | --- | --- |
| `UnreadLines-Guest` | Personal devices, no domain trust | WPA2-Personal | Built, live today |
| `UnreadLines-Mobile` | Domain-managed mobile devices (Intune-enrolled) | WPA2/WPA3-Enterprise, EAP-TLS | Built first, once RADIUS/NPS exists |
| `UnreadLines-Corp` | Domain-joined workstations | WPA2/WPA3-Enterprise, EAP-TLS | Later — same pattern, reserved for now |

(Names above are a proposal, not yet finalized — adjust freely if a different scheme fits the fleet better.)

### Addressing plan for the Wi-Fi segments

One `/25` per SSID, allocated sequentially from `192.168.21.0`, mirroring the way the LAB network splits
`192.168.20.0/24` into two `/25` subnets:

| VMware network | Subnet | SSID | Status |
| --- | --- | --- | --- |
| `VMnet3` | `192.168.21.0/25` | `UnreadLines-Guest` | In service |
| `VMnet4` | `192.168.21.128/25` | `UnreadLines-Mobile` | To build |
| `VMnet5` | `192.168.22.0/25` | `UnreadLines-Corp` | Reserved |

`reference/vm-inventory.md` §3 is the authoritative record of this table.

### Why each SSID gets its own network, even though only one runs at a time

The single-radio limit means `UnreadLines-Guest` and `UnreadLines-Mobile` are never live simultaneously,
so one shared network would carry the traffic just as well. They still get one each, because a VMware
network here is not just a wire — it is a **firewall zone and a DHCP scope on `FWL02`**, and the two
SSIDs do not deserve the same ones. `UnreadLines-Guest` carries unmanaged personal devices and is
allowed out to the Internet and nowhere else. `UnreadLines-Mobile` carries Intune-managed devices
authenticated by certificate, which have business reaching internal resources.

Sharing one network would mean rewriting `FWL02`'s zone and DHCP configuration at every SSID swap,
in both directions, instead of changing a single `bridge=` line in `hostapd.conf`. It would also put
guest and managed devices in the same broadcast domain — the exact arrangement an enterprise SSID
exists to avoid.

And the constraint is temporary: adding a second physical Wi-Fi adapter makes both SSIDs concurrent, at
which point separate segments stop being a design preference and become a requirement. Building them
separately now costs nothing that the swap does not already cost.

### Building it

Building `UnreadLines-Mobile` follows the same pattern as SSID#1 — a new, dedicated VMware network
(`VMnet4`) rather than a VLAN (§11), on `192.168.21.128/25`:

1. Create `VMnet4` and attach it to `RAP01` and `FWL02`, same process as §6.
2. Add the new interface to `RAP01`'s Netplan and to `FWL02` (§5.5 / §8), same pattern as SSID#1 — its own firewall zone and DHCP scope, not a copy of the guest one.
3. `sudo systemctl stop hostapd` on `RAP01`; edit `/etc/hostapd/hostapd.conf` to point at `UnreadLines-Mobile`, switch to `ieee8021x=1` / `wpa_key_mgmt=WPA-EAP` with the RADIUS IP/shared secret, and change `bridge=br-vlan10` to the new bridge; `sudo systemctl start hostapd`. `UnreadLines-Guest` goes offline while it's active.
4. Update `reference/vm-inventory.md` accordingly.

`UnreadLines-Corp` (workstations) is the same build, later — `VMnet5` on `192.168.22.0/25`, same swap-in mechanism. Both remain single-BSS constrained until a second physical Wi-Fi adapter is added, at which point any two of these three SSIDs could run concurrently.

The PKI that work depends on is `ad-cs-pki-deployment/README.md`; its certificate and Intune side is not yet covered by a runbook here.

## 16. References

- [hostapd documentation](https://w1.fi/hostapd/)
- [OpenWrt — DHCP and DNS (dnsmasq)](https://openwrt.org/docs/guide-user/base-system/dhcp)
- [OpenWrt — firewall zones](https://openwrt.org/docs/guide-user/firewall/firewall_configuration)
- [Netplan — bridges](https://netplan.readthedocs.io/en/stable/netplan-yaml/#properties-for-device-type-bridges)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
