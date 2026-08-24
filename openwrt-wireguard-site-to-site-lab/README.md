# VMware Workstation Pro + OpenWrt Network Lab — PC1 and PC2 with a WireGuard Tunnel

This document describes, **from scratch**, how to set up a network lab spread across **two physical PCs**, each hosting an **OpenWrt x86-64** router/firewall inside **VMware Workstation Pro** on Windows.

It is designed to be reproducible: you can delete the VMs, start again from the original OpenWrt images, and redo every step to validate the procedure end to end, on both PCs.

Each PC hosts an isolated LAB network (`192.168.20.0/25` on PC1, `192.168.20.128/25` on PC2), routed to the Internet by its own OpenWrt over Wi-Fi. The two LABs are then linked together by a **WireGuard tunnel** set up directly between the two WAN addresses, which works around a limitation of the VMware bridge over Wi-Fi (explained in section 19).

---

## 1. Lab objective

```text
- One OpenWrt router per PC, with its own WAN leg (Wi-Fi/Freebox) and its
  own LAN leg (private VMware network).
- One /25 LAB subnet per PC, routed and NATed to the Internet by its own
  OpenWrt.
- No LAB VM directly bridged or on VMware NAT: everything goes through
  OpenWrt.
- The two LABs linked together via a WireGuard tunnel between the two
  WANs, so VMs on both PCs can reach each other.
```

---

## 2. Target architecture

```text
                                       +----------------------------------+
                                       |             Internet             |
                                       +----------------------------------+
                                                        |
                                       +----------------------------------+
                                       |             Freebox              |
                                       |          192.168.1.254           |
                                       +----------------------------------+
                                                        |
                                       +----------------------------------+
                                       |        Home Wi-Fi network        |
                                       |          192.168.1.0/24          |
                                       +----------------------------------+
                                                         |
                    +------------------------------------+------------------------------------+
                    |                                                                         |
+--------------------------------------+                                  +--------------------------------------+
|                 PC1                  |                                  |                 PC2                  |
|          Wi-Fi 192.168.1.29          |                                  |         Wi-Fi 192.168.1.119          |
+--------------------------------------+                                  +--------------------------------------+
                    |                                                                         |
+--------------------------------------+                                  +--------------------------------------+
|        VMware Workstation Pro        |                                  |        VMware Workstation Pro        |
+--------------------------------------+                                  +--------------------------------------+
                    |                                                                         |
 VMnet0 (Bridged) + VMnet2 (Host-only)                                     VMnet0 (Bridged) + VMnet2 (Host-only)
                    |                                                                         |
+--------------------------------------+                                  +--------------------------------------+
|            U01PARVMFWL01             |                                  |            U01PARVMFWL02             |
|            OpenWrt x86-64            |                                  |            OpenWrt x86-64            |
|                                      |<================================>|                                      |
|        WAN  192.168.1.250/24         |       WireGuard tunnel wg0       |        WAN  192.168.1.251/24         |
|          wg0  10.10.10.1/30          |          10.10.10.0/30           |          wg0  10.10.10.2/30          |
|        LAN  192.168.20.126/25        |                                  |        LAN  192.168.20.254/25        |
+--------------------------------------+                                  +--------------------------------------+
                    |                                                                         |
+--------------------------------------+                                  +--------------------------------------+
|       LAB1  (192.168.20.0/25)        |                                  |      LAB2  (192.168.20.128/25)       |
+--------------------------------------+                                  +--------------------------------------+
                    |                                                                         |
+--------------------------------------+                                  +--------------------------------------+
|              Example VM              |                                  |              Example VM              |
|            192.168.20.41             |                                  |            192.168.20.131            |
+--------------------------------------+                                  +--------------------------------------+
```

Each OpenWrt acts as the gateway, NAT, and firewall for its own LAB. The WireGuard tunnel only carries traffic **between** the two LABs — each LAB's Internet access goes through its own OpenWrt's WAN masquerading (section 14), independently of the tunnel.

---

## 3. Full addressing plan

| Element                            | PC1 / FWL01            | PC2 / FWL02             |
| ------------------------------------ | ------------------------ | -------------------------- |
| Router VM name                     | `U01PARVMFWL01`         | `U01PARVMFWL02`           |
| Windows host's Wi-Fi                | `192.168.1.29`          | `192.168.1.119`           |
| OpenWrt WAN                        | `192.168.1.250/24`      | `192.168.1.251/24`        |
| WAN gateway (Freebox)               | `192.168.1.254`         | `192.168.1.254`           |
| OpenWrt LAN (LAB gateway)           | `192.168.20.126/25`     | `192.168.20.254/25`       |
| LAB network                        | `192.168.20.0/25`       | `192.168.20.128/25`       |
| Windows host on VMnet2              | `192.168.20.1/25`       | `192.168.20.129/25`       |
| Example LAB server VM               | `192.168.20.41/25`      | `192.168.20.131/25`       |
| WireGuard tunnel (wg0 interface)    | `10.10.10.1/30`         | `10.10.10.2/30`           |
| WireGuard listening port            | `51820/udp`             | `51820/udp`               |

Breakdown of the global LAB network `192.168.20.0/24`:

```text
192.168.20.0/24
        |
        +-- LAB PC1 : 192.168.20.0/25    GW 192.168.20.126
        |
        +-- LAB PC2 : 192.168.20.128/25  GW 192.168.20.254
```

PC1's LAB network:

```text
Network     : 192.168.20.0/25       Broadcast     : 192.168.20.127
Netmask     : 255.255.255.128       Usable range  : .1 - .126
Gateway     : 192.168.20.126
```

PC2's LAB network:

```text
Network     : 192.168.20.128/25     Broadcast     : 192.168.20.255
Netmask     : 255.255.255.128       Usable range  : .129 - .254
Gateway     : 192.168.20.254
```

---

## 4. Why use OpenWrt as the router VM

OpenWrt provides, in a very lightweight VM: IPv4/IPv6 routing, a stateful firewall, NAT and masquerading, port forwarding, 802.1Q VLANs, DHCP and DNS, static routes, WireGuard, the LuCI web interface, SSH administration, package management, and the ability to add routing protocols and other services.

For this lab, a VM with **1 vCPU and 512 MB of RAM** is more than enough, on each of the two PCs.

---

## 5. Download and convert the OpenWrt image

This step is identical on PC1 and PC2: each PC needs its own `.vmdk` file, produced the same way.

### 5.1 Download the image

The current stable branch is **OpenWrt 25.12**, current version **25.12.5**. For VMware x86-64, download the **BIOS/non-EFI** variant:

```text
openwrt-25.12.5-x86-64-generic-ext4-combined.img.gz
```

Official download:

```text
https://downloads.openwrt.org/releases/25.12.5/targets/x86/64/
```

Verify the SHA-256 published on the download page:

```powershell
Get-FileHash .\openwrt-25.12.5-x86-64-generic-ext4-combined.img.gz -Algorithm SHA256
```

> If a newer stable version exists by the time the lab is rebuilt, use the latest stable release and adjust the file names in the following steps.

### 5.2 Decompress the image

With 7-Zip: right-click the `.img.gz` file → `7-Zip` → `Extract Here`. You get an `.img` file, a raw disk image.

### 5.3 Convert IMG → VMDK with StarWind V2V Converter

VMware Workstation Pro uses `.vmdk` disks. The official OpenWrt documentation describes a conversion with `qemu-img`; this lab uses **StarWind V2V Converter** instead, to avoid installing QEMU.

```text
1. Source image location      → Local file
2. Source image               → openwrt-25.12.5-x86-64-generic-ext4-combined.img
3. Destination image location → Local file
4. Destination image format   → VMDK / VMware image
5. Type                       → VMware Workstation growable image
6. Destination file           → U01PARVMFWL01.vmdk   (or U01PARVMFWL02.vmdk on PC2)
7. Run the conversion
```

The **growable** mode allocates disk space as it is used. Repeat this conversion on PC2 to produce `U01PARVMFWL02.vmdk`.

---

## 6. Prepare the VMware networks (on each PC)

The goal is for **VMware to do neither NAT nor DHCP in OpenWrt's place** on the LAB network. This section must be done on both PC1 and PC2, with each PC's own values.

### 6.1 Recap of VMware networks

```text
VMnet0 = Bridged     → link to the physical network / Freebox
VMnet2 = Host-only   → private network for the LAB VMs (created for this lab)
VMnet8 = VMware NAT  → not used here
```

**VMnet2** is deliberately created instead of reusing VMnet8, which is normally associated with VMware's own NAT.

### 6.2 Create VMnet2

In `Edit → Virtual Network Editor → Add Network → VMnet2`, then configure:

| Parameter                                          | PC1                 | PC2                    |
| ----------------------------------------------------- | ---------------------- | -------------------------- |
| Type                                                | Host-only             | Host-only                  |
| Subnet IP                                           | `192.168.20.0`        | `192.168.20.128`           |
| Subnet mask                                         | `255.255.255.128`     | `255.255.255.128`          |
| Connect a host virtual adapter to this network      | Enabled               | Enabled                    |
| Use local DHCP service                              | Disabled              | Disabled                   |
| NAT VMware                                          | Disabled              | Disabled                   |

`Host-only` does not mean the VMs will be cut off from the Internet: VMnet2 simply provides a private virtual Ethernet switch between the Windows host, OpenWrt's LAN leg, and the lab VMs. OpenWrt then handles the routing to the Internet.

### 6.3 Configure Windows' VMnet2 interface

On Windows (`Win + R` → `ncpa.cpl`), open the properties of `VMware Network Adapter VMnet2` and configure IPv4:

| Parameter    | PC1                  | PC2                     |
| -------------- | ---------------------- | --------------------------- |
| IP address   | `192.168.20.1`        | `192.168.20.129`            |
| Netmask      | `255.255.255.128`     | `255.255.255.128`           |
| Gateway      | empty                  | empty                       |
| DNS          | empty                  | empty                       |

This Windows interface must have **no gateway at all**: the PC's Internet connection must keep using its normal Wi-Fi connection.

### 6.4 Configure the VMware bridge onto Wi-Fi

In `Edit → Virtual Network Editor → VMnet0`:

```text
VMnet0
Type       : Bridged
Bridged to : the PC's specific physical Wi-Fi adapter (avoid "Automatic")
```

On Windows (`ncpa.cpl` → Wi-Fi adapter → Properties), check that the `VMware Bridge Protocol` component is ticked. This is essential: if the bridge isn't properly tied to the Wi-Fi adapter, OpenWrt can show its WAN interface as `UP` while still being unable to reach the Freebox.

Do this identically on PC1 and PC2, each bridging to its own Wi-Fi adapter.

---

## 7. Create the OpenWrt VMs (FWL01 and FWL02)

On each PC: `Create a New Virtual Machine → Custom (advanced) → I will install the operating system later`.

| Parameter    | PC1                  | PC2                     |
| -------------- | ---------------------- | --------------------------- |
| VM name      | `U01PARVMFWL01`       | `U01PARVMFWL02`             |
| Guest OS     | Linux                  | Linux                       |
| Version      | Other Linux 64-bit     | Other Linux 64-bit          |
| CPU          | 1 vCPU                 | 1 vCPU                      |
| RAM          | 512 MB                 | 512 MB                      |
| Firmware     | BIOS                   | BIOS                        |

The **BIOS** firmware matches the non-EFI `generic-ext4-combined` image used in this tutorial.

### 7.1 Disk

Choose `Use an existing virtual disk`, then select the matching `.vmdk` (`U01PARVMFWL01.vmdk` on PC1, `U01PARVMFWL02.vmdk` on PC2). If VMware asks `Convert` or `Keep Existing Format`, choose **`Keep Existing Format`**.

### 7.2 CD/DVD drive

OpenWrt boots directly from the VMDK, no ISO is needed. Uncheck `Connect at power on` on the CD/DVD drive (`VM Settings → CD/DVD`), to avoid VMware messages about an unavailable SATA/CD-ROM device.

---

## 8. Configure the VMs' network adapters

The order of the adapters is **deliberate and matters on first boot**, identically on both VMs.

### NIC 1 — LAB LAN

```text
Network Adapter 1 → Custom: Specific virtual network → VMnet2
Connected / Connect at power on : Enabled
```

Will normally become `eth0`, used by OpenWrt as the LAN.

### NIC 2 — physical network / Freebox

```text
Network Adapter 2 → Bridged (or Custom: VMnet0 if VMnet0 was explicitly bound to Wi-Fi)
Connected / Connect at power on : Enabled
```

Will normally become `eth1`, used as the WAN.

### Why put VMnet2 on NIC 1?

A fresh OpenWrt x86 install normally uses its first interface as the LAN and boots with a default LAN configuration on `192.168.1.1`. By putting **VMnet2 on the first interface**, the default LAN and any DHCP it runs stay confined to the lab's virtual network, and don't conflict with the physical network `192.168.1.0/24` on first boot.

---

## 9. First boot and LAN configuration

Start the VM. After boot, you should get `root@OpenWrt:~#`. The console uses a US keyboard layout and BusyBox may not support certain command variants (`ip -br`, for instance) — use the plain forms `ip addr show` and `uci show network`.

Configure the LAN with the address specific to each router:

**On FWL01 (PC1):**

```sh
uci set network.lan.ipaddr='192.168.20.126'
uci set network.lan.netmask='255.255.255.128'
uci commit network
/etc/init.d/network restart
```

**On FWL02 (PC2):**

```sh
uci set network.lan.ipaddr='192.168.20.254'
uci set network.lan.netmask='255.255.255.128'
uci commit network
/etc/init.d/network restart
```

Check with `ip addr show br-lan` that the expected address appears.

---

## 10. Access the router over SSH

The Windows host already has its IP on VMnet2 (`192.168.20.1` on PC1, `192.168.20.129` on PC2). From PowerShell:

```powershell
ping 192.168.20.126   # PC1
ping 192.168.20.254   # PC2
```

If the ping succeeds, connect over SSH from Windows:

```powershell
ssh root@192.168.20.126   # PC1
ssh root@192.168.20.254   # PC2
```

Once connected, set a root password if this hasn't already been done:

```sh
passwd
```

From this point on, all configuration can be done by copy/pasting over SSH, on both routers.

---

## 11. Configure the hostname

**FWL01:**

```sh
uci set system.@system[0].hostname='U01PARVMFWL01'
uci commit system
/etc/init.d/system reload
```

**FWL02:**

```sh
uci set system.@system[0].hostname='U01PARVMFWL02'
uci commit system
/etc/init.d/system reload
```

---

## 12. Configure the WAN toward the Freebox

**FWL01:**

```sh
uci set network.wan='interface'                  # creates the "wan" section (type interface)
uci set network.wan.device='eth1'                 # binds the WAN to eth1, the adapter bridged onto Wi-Fi
uci set network.wan.proto='static'                # fixed IP instead of DHCP (needed for the WireGuard tunnel later)
uci set network.wan.ipaddr='192.168.1.250'
uci set network.wan.netmask='255.255.255.0'
uci set network.wan.gateway='192.168.1.254'       # the Freebox
uci -q delete network.wan.dns                     # clears the DNS list first (-q = no error if it doesn't exist yet)
uci add_list network.wan.dns='192.168.1.254'
uci add_list network.wan.dns='1.1.1.1'
uci commit network                                # writes the changes to /etc/config/network
/etc/init.d/network restart                       # applies the new config
```

**FWL02:**

```sh
uci set network.wan='interface'                  # creates the "wan" section (type interface)
uci set network.wan.device='eth1'                 # binds the WAN to eth1, the adapter bridged onto Wi-Fi
uci set network.wan.proto='static'                # fixed IP instead of DHCP (needed for the WireGuard tunnel later)
uci set network.wan.ipaddr='192.168.1.251'
uci set network.wan.netmask='255.255.255.0'
uci set network.wan.gateway='192.168.1.254'       # the Freebox
uci -q delete network.wan.dns                     # clears the DNS list first (-q = no error if it doesn't exist yet)
uci add_list network.wan.dns='192.168.1.254'
uci add_list network.wan.dns='1.1.1.1'
uci commit network                                # writes the changes to /etc/config/network
/etc/init.d/network restart                       # applies the new config
```

---

## 13. Basic network checks

Run on **each** router:

```sh
ip addr show eth1        # should show the router's WAN IP
ip addr show br-lan      # should show the router's LAN IP
ip route
```

Expected routing table on FWL01:

```text
default via 192.168.1.254 dev eth1
192.168.1.0/24 dev eth1 scope link src 192.168.1.250
192.168.20.0/25 dev br-lan scope link src 192.168.20.126
```

Expected routing table on FWL02:

```text
default via 192.168.1.254 dev eth1
192.168.1.0/24 dev eth1 scope link src 192.168.1.251
192.168.20.128/25 dev br-lan scope link src 192.168.20.254
```

Test in this order on each router:

```sh
ping -c 3 192.168.1.254   # VMware bridge + Freebox network OK
ping -c 3 1.1.1.1         # Internet routing OK
ping -c 3 openwrt.org     # DNS OK
```

---

## 14. Firewall and NAT (masquerading)

OpenWrt provides, by default on both routers, a standard policy:

```text
zone lan : input ACCEPT, output ACCEPT, forward ACCEPT
zone wan : input REJECT, output ACCEPT, forward DROP, masq = 1
forwarding lan → wan
```

Check on each router:

```sh
uci show firewall | grep -E "zone|forwarding|masq|network"
sysctl net.ipv4.ip_forward     # should return = 1
```

Masquerading is the outbound NAT: connections initiated from a LAB VM (e.g. `192.168.20.41`) leave with source `192.168.1.250` (or `.251` on PC2) as far as the Freebox is concerned. The Freebox therefore doesn't need to know about the `192.168.20.0/25` (or `.128/25`) network for Internet access.

```text
VM 192.168.20.41 → OpenWrt LAN .126 → OpenWrt WAN .250 → NAT (source becomes .250) → Freebox → Internet
```

Never remove masquerading from the WAN zone, or you'll cut off that LAB's Internet access.

---

## 15. Accessing LuCI

From Windows: `http://192.168.20.126` (PC1) or `http://192.168.20.254` (PC2).

If LuCI isn't installed, once the WAN and Internet are working:

```sh
apk -U add luci
/etc/init.d/uhttpd enable
/etc/init.d/uhttpd restart
```

> OpenWrt **25.12 and later use `apk`** as the package manager; documentation using `opkg` applies to OpenWrt 24.10 and earlier.

In LuCI, check `Network → Interfaces` (LAN and WAN with the right addresses) then `Network → Firewall` (LAN → WAN forwarding and WAN masquerading active). It's recommended **not** to allow LuCI or SSH from the WAN zone: administration should stay accessible only from each lab's LAN.

---

## 16. LAB DHCP: an explicit choice to make

VMware DHCP must stay disabled on VMnet2, on both PCs. From there, two options, either one, identical on PC1 and PC2:

**Option A — static addressing / DHCP elsewhere** (e.g. a Windows domain server): disable OpenWrt's LAN DHCP in LuCI (`Network → Interfaces → LAN → DHCP Server → Ignore interface`), to avoid two competing DHCP servers.

**Option B — OpenWrt provides DHCP**: set a range that doesn't conflict with the static addresses, for example `192.168.20.80 - 192.168.20.120` on PC1 (adapt the range for PC2 within `192.168.20.128/25`). The distributed gateway must be the local router's LAN IP.

In an Active Directory environment, domain clients should use the domain's DNS rather than a public DNS directly.

---

## 17. Configure the LAB VMs

On each PC, every LAB VM uses **a single network adapter**, on `VMnet2` (never `Bridged` nor `NAT/VMnet8` — OpenWrt is the sole router to the outside world).

Example on PC1:

```text
IP         : 192.168.20.41
Netmask    : 255.255.255.128
Gateway    : 192.168.20.126
DNS        : depends on the lab
```

Example on PC2:

```text
IP         : 192.168.20.131
Netmask    : 255.255.255.128
Gateway    : 192.168.20.254
DNS        : depends on the lab
```

---

## 18. Full connectivity tests

**From the LAB1 example VM (`192.168.20.41`):**

```powershell
ping 192.168.20.126     # LAN gateway
ping 192.168.1.250      # OpenWrt WAN
ping 192.168.1.254      # Freebox
ping 1.1.1.1            # Internet without DNS
nslookup openwrt.org    # DNS
tracert 1.1.1.1         # the first hop should be 192.168.20.126
```

**From the LAB2 example VM (`192.168.20.131`):**

```powershell
ping 192.168.20.254     # LAN gateway
ping 192.168.1.251      # OpenWrt WAN
ping 192.168.1.254      # Freebox
ping 1.1.1.1            # Internet without DNS
nslookup openwrt.org    # DNS
tracert 1.1.1.1         # the first hop should be 192.168.20.254
```

Expected logical path (LAB1 example):

```text
VM 192.168.20.x → 192.168.20.126 (OpenWrt LAN) → 192.168.1.250 (OpenWrt WAN + NAT) → 192.168.1.254 (Freebox) → Internet
```

Expected logical path (LAB2 example):

```text
VM 192.168.20.13x → 192.168.20.254 (OpenWrt LAN) → 192.168.1.251 (OpenWrt WAN + NAT) → 192.168.1.254 (Freebox) → Internet
```

At this stage, each LAB has independent Internet access. What remains is linking the two LABs together — the subject of the following sections.

---

## 19. Why direct routing between LAB1 and LAB2 doesn't work natively

A natural attempt is to add, on each OpenWrt, a static route to the remote network via the other router's WAN IP (`192.168.20.128/25 via 192.168.1.251` on FWL01, and the reverse on FWL02). This route is correct from Linux's point of view — `ip route get` confirms the right next hop — and the standard firewall (LAN→WAN forwarding, `Allow-Ping` rule) doesn't block anything.

Yet this traffic doesn't get through. Diagnosis (`tcpdump` captures on both OpenWrt routers plus Wireshark on the destination host's physical Wi-Fi adapter) shows that:

```text
FWL02 does send the packet toward 192.168.20.126, next hop 192.168.1.250   : OK
The packet is visible on the destination PC1's physical Wi-Fi adapter     : OK
tcpdump on FWL01's eth1 never sees the packet                            : KO
```

The comparative test settles it:

```text
ping -I 192.168.20.254 -c 3 192.168.1.250     → OK   (destination = the VM's own WAN IP)
ping -I 192.168.20.254 -c 3 192.168.20.126    → KO   (destination = a network behind the VM)
```

**Cause**: on a Wi-Fi interface, VMware cannot do true L2 bridging (802.11 only allows a single MAC address associated per radio). The "VMware Bridge Protocol" simulates the bridge through a form of proxy-ARP/MAC translation, but it only relays traffic whose IP matches exactly that of the bridged VM (the one observed via ARP). A packet routed toward a network located **behind** the VM, with a different destination IP, simply never reaches the vNIC — it's lost between the physical Wi-Fi adapter and the OpenWrt interface.

This behavior was confirmed independently of the network configuration itself: a temporary workaround routing traffic at L3 through both Windows hosts (before being fully removed) confirmed that OpenWrt's routing, both subnets, and the firewall rules all worked correctly — only the direct path through the Wi-Fi bridge for routed traffic is the problem.

The only path that reliably works through this bridge is the one already validated: **traffic addressed directly to the other VM's WAN IP**. Hence the chosen solution: a tunnel that encapsulates inter-LAB traffic inside a session whose outer IP addresses are precisely the two WAN IPs.

---

## 20. WireGuard tunnel between FWL01 and FWL02

The WireGuard tunnel encapsulates all `192.168.20.0/25 ↔ 192.168.20.128/25` traffic inside a UDP session addressed directly between the two WANs:

```text
FWL01 WAN 192.168.1.250  ⇄  FWL02 WAN 192.168.1.251
```

The VMware bridge then only ever sees UDP traffic addressed to the WAN IPs it already knows how to relay correctly (section 19) — the routed traffic that was the problem is invisible to it, hidden inside the tunnel.

### 20.1 Package manager and kernel module

On OpenWrt 25.12.5, `opkg` no longer exists: the default package manager is `apk`. The `luci-app-wireguard` package doesn't exist under that name on this branch either — LuCI's WireGuard support comes through `luci-proto-wireguard`, and isn't needed anyway since all configuration is done via `uci`.

The WireGuard kernel module is already built into the x86-64 generic image. Check:

```sh
modprobe wireguard
echo $?     # 0 = module available
```

### 20.2 Installation

On **FWL01** and **FWL02**:

```sh
apk update
apk add wireguard-tools
```

### 20.3 Key generation

On each router, with a file name that identifies the **local** router (to avoid any mix-up when copy/pasting between the two SSH sessions):

```sh
umask 077
wg genkey | tee /etc/wireguard_local.key | wg pubkey > /etc/wireguard_local.pub
cat /etc/wireguard_local.pub
```

Each router needs to obtain the other's **public** key before continuing.

### 20.4 UCI configuration — FWL01

```sh
uci set network.wg0=interface                      # creates the "wg0" section (type interface)
uci set network.wg0.proto='wireguard'                # uses netifd's WireGuard proto script
uci set network.wg0.private_key="$(cat /etc/wireguard_local.key)"  # injects the private key generated in 20.3
uci set network.wg0.listen_port='51820'
uci add_list network.wg0.addresses='10.10.10.1/30'   # wg0's IP INSIDE the tunnel (not the WAN, not the LAN)
uci set network.wg0.mtu='1380'                       # lowered to offset WireGuard's overhead (~60 bytes)

uci add network wireguard_wg0                        # anonymous section, type "wireguard_wg0" = a peer of the wg0 interface
uci set network.@wireguard_wg0[-1].description='FWL02'
uci set network.@wireguard_wg0[-1].public_key='<FWL02_PUBLIC_KEY>'     # authenticates/encrypts traffic to FWL02
uci set network.@wireguard_wg0[-1].endpoint_host='192.168.1.251'       # FWL02's WAN IP — the one path that crosses the VMware bridge (section 19)
uci set network.@wireguard_wg0[-1].endpoint_port='51820'
uci set network.@wireguard_wg0[-1].route_allowed_ips='1'               # auto-adds the routes below toward wg0
uci add_list network.@wireguard_wg0[-1].allowed_ips='10.10.10.2/32'    # FWL02's tunnel IP
uci add_list network.@wireguard_wg0[-1].allowed_ips='192.168.20.128/25' # all of LAB2, routed through the tunnel
uci set network.@wireguard_wg0[-1].persistent_keepalive='25'           # keeps the mapping alive on the VMware bridge side
uci commit network

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WireGuard'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].target='ACCEPT'          # without this, the default WAN policy (REJECT) blocks the handshake

uci add firewall zone
uci set firewall.@zone[-1].name='wg'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci add_list firewall.@zone[-1].network='wg0'        # trusted zone dedicated to the tunnel

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wg'
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wg'
uci set firewall.@forwarding[-1].dest='lan'          # forwarding is directional: both rules are needed
uci commit firewall

/etc/init.d/network restart
/etc/init.d/firewall restart
```

### 20.5 UCI configuration — FWL02

Mirror configuration (same explanations as 20.4, with the values swapped):

```sh
uci set network.wg0=interface                      # creates the "wg0" section (type interface)
uci set network.wg0.proto='wireguard'                # uses netifd's WireGuard proto script
uci set network.wg0.private_key="$(cat /etc/wireguard_local.key)"  # injects the private key generated in 20.3
uci set network.wg0.listen_port='51820'
uci add_list network.wg0.addresses='10.10.10.2/30'   # wg0's IP INSIDE the tunnel (not the WAN, not the LAN)
uci set network.wg0.mtu='1380'                       # lowered to offset WireGuard's overhead (~60 bytes)

uci add network wireguard_wg0                        # anonymous section, type "wireguard_wg0" = a peer of the wg0 interface
uci set network.@wireguard_wg0[-1].description='FWL01'
uci set network.@wireguard_wg0[-1].public_key='<FWL01_PUBLIC_KEY>'     # authenticates/encrypts traffic to FWL01
uci set network.@wireguard_wg0[-1].endpoint_host='192.168.1.250'       # FWL01's WAN IP — the one path that crosses the VMware bridge (section 19)
uci set network.@wireguard_wg0[-1].endpoint_port='51820'
uci set network.@wireguard_wg0[-1].route_allowed_ips='1'               # auto-adds the routes below toward wg0
uci add_list network.@wireguard_wg0[-1].allowed_ips='10.10.10.1/32'    # FWL01's tunnel IP
uci add_list network.@wireguard_wg0[-1].allowed_ips='192.168.20.0/25'  # all of LAB1, routed through the tunnel
uci set network.@wireguard_wg0[-1].persistent_keepalive='25'           # keeps the mapping alive on the VMware bridge side
uci commit network

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WireGuard'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].target='ACCEPT'          # without this, the default WAN policy (REJECT) blocks the handshake

uci add firewall zone
uci set firewall.@zone[-1].name='wg'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci add_list firewall.@zone[-1].network='wg0'        # trusted zone dedicated to the tunnel

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wg'
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wg'
uci set firewall.@forwarding[-1].dest='lan'          # forwarding is directional: both rules are needed
uci commit firewall

/etc/init.d/network restart
/etc/init.d/firewall restart
```

### 20.6 Why `route_allowed_ips='1'` is enough

This parameter tells `netifd` to automatically add a route to `wg0` for every `allowed_ips` entry on the peer. There's no need for manual static routes toward `wan` (those would point at the bridged path that fails, see section 19): the routes generated here point at `wg0`, which encapsulates the traffic inside the UDP session addressed directly between the two WANs.

### 20.7 Verifying the tunnel

```sh
wg show                 # a recent "latest handshake" should appear on both sides
ping -c 3 10.10.10.2    # from FWL01
ping -c 3 10.10.10.1    # from FWL02
```

### 20.8 Operational notes

```text
- persistent_keepalive='25' keeps the VMware bridge state and any Freebox
  NAT state alive; don't remove it.
- The tunnel doesn't replace WAN masquerading (section 14): that's still
  needed for each LAB's Internet access, the tunnel only carries
  inter-LAB traffic.
```

---

## 21. Full end-to-end verification

```powershell
# from a LAB1 VM, e.g. 192.168.20.41
ping 192.168.20.131

# from a LAB2 VM, e.g. 192.168.20.131
ping 192.168.20.41
```

Expected final state, across the whole lab:

```text
PC1/LAB1 → FWL01 192.168.20.126        : OK
PC2/LAB2 → FWL02 192.168.20.254        : OK
FWL01 WAN ↔ FWL02 WAN                  : OK
LAB1 → Internet via FWL01 (WAN masq.)  : OK
LAB2 → Internet via FWL02 (WAN masq.)  : OK
Tunnel wg0 : 10.10.10.1 ↔ 10.10.10.2   : OK
LAB1 ↔ LAB2 (e.g. .41 ↔ .131)          : OK (via wg0)
```

---

## 22. Back up the OpenWrt configuration

Once each router is validated, take a backup before adding VLANs, extra firewall rules, or inter-site routing.

In LuCI: `System → Backup / Flash Firmware → Generate archive`.

From the CLI, on each router:

```sh
sysupgrade -b /tmp/U01PARVMFWL01-backup.tar.gz   # or U01PARVMFWL02 on PC2
```

Retrieve the file before making any deep changes to the lab.

---

## 23. Best practices and pitfalls to avoid

```text
- Don't put lab VMs on VMnet8 (VMware NAT): that role belongs to OpenWrt.
- Only bridge the OpenWrt VM to Wi-Fi, never the lab VMs themselves.
- Never set a gateway on Windows' VMnet2 interface: the host keeps its
  default route via its own Wi-Fi.
- Don't expose LuCI/SSH from the WAN zone unless truly needed.
- Don't run several DHCP servers (VMware, OpenWrt, Windows) on the same
  segment unless you explicitly mean to.
- Name WireGuard key files after the local router
  (wireguard_local.key), not the remote one, to avoid any mix-up when
  copy/pasting between the two SSH sessions.
- Don't try to solve the inter-LAB problem with simple static routes via
  wan: it will never work over a Wi-Fi bridge (section 19) — the tunnel
  is the solution, not a temporary workaround.
```

---

## 24. Full deployment checklist

### PC1 / FWL01

* [ ] Create `VMnet2` as Host-only, `192.168.20.0/25`, DHCP and NAT disabled.
* [ ] Configure Windows VMnet2 as `192.168.20.1/25`, no gateway.
* [ ] Configure VMnet0 as an explicit bridge to Wi-Fi, check `VMware Bridge Protocol`.
* [ ] Create FWL01 from the OpenWrt VMDK (NIC1 = VMnet2, NIC2 = VMnet0).
* [ ] LAN `192.168.20.126/25`, WAN `192.168.1.250/24`, gateway `192.168.1.254`.
* [ ] Test `192.168.1.254`, `1.1.1.1`, DNS.
* [ ] Check masquerading is on WAN only.
* [ ] LAB1 VMs on VMnet2 only, gateway `192.168.20.126`.

### PC2 / FWL02

* [ ] Create `VMnet2` as Host-only, `192.168.20.128/25`, DHCP and NAT disabled.
* [ ] Configure Windows VMnet2 as `192.168.20.129/25`, no gateway.
* [ ] Configure VMnet0 as an explicit bridge to Wi-Fi, check `VMware Bridge Protocol`.
* [ ] Create FWL02 from the OpenWrt VMDK (NIC1 = VMnet2, NIC2 = VMnet0).
* [ ] LAN `192.168.20.254/25`, WAN `192.168.1.251/24`, gateway `192.168.1.254`.
* [ ] Test `192.168.1.254`, `1.1.1.1`, DNS.
* [ ] Check masquerading is on WAN only.
* [ ] LAB2 VMs on VMnet2 only, gateway `192.168.20.254`.

### Inter-LAB tunnel

* [ ] `apk add wireguard-tools` on both routers.
* [ ] Check the kernel module (`modprobe wireguard`).
* [ ] Generate and exchange the public keys.
* [ ] Apply the `wg0` config + firewall on FWL01 and FWL02.
* [ ] `wg show`: recent handshake on both sides.
* [ ] Ping `10.10.10.1 ↔ 10.10.10.2`.
* [ ] Ping between a LAB1 VM and a LAB2 VM.

---

## 25. Useful diagnostic commands

### OpenWrt — interfaces and routing

```sh
ip addr
ip addr show br-lan
ip addr show eth1
ip route
ip route get 192.168.20.126
ip neigh show dev eth1
```

### OpenWrt — firewall

```sh
uci show firewall
uci show firewall | grep masq
sysctl net.ipv4.ip_forward
```

### OpenWrt — packet captures

```sh
tcpdump -ni eth1 icmp
tcpdump -ni any icmp
tcpdump -eni eth1 icmp     # -e also shows MAC addresses
```

### OpenWrt — WireGuard

```sh
wg show
ip addr show wg0
ip route | grep wg0
```

### Test with a forced source address

```sh
ping -I 192.168.20.254 -c 3 192.168.1.250
ping -I 192.168.20.254 -c 3 192.168.20.126
```

### Windows

```powershell
Get-NetAdapter
Get-NetIPAddress -AddressFamily IPv4
Get-NetAdapterBinding -Name "Wi-Fi" -ComponentID vmware_bridge
Get-NetRoute -AddressFamily IPv4
tracert 192.168.20.131
```

### Wireshark — useful filters

```text
icmp
icmp && ip.addr == 192.168.20.126
icmp && ip.addr == 192.168.20.254
udp.port == 51820
```

---

## 26. Access from the host PCs to the remote LAB

The WireGuard tunnel correctly links the VMs of both LABs together. But by default, the **host PCs themselves** (not the VMs) can reach VMs on their own local LAB, but not those on the remote LAB (SSH, RDP, etc.), even when the tunnel is working perfectly.

### Why

* The LAB VMs use OpenWrt as their default gateway, so they automatically inherit the route to the remote LAB added by `route_allowed_ips='1'` (section 20.6).
* The host PC's `VMware Network Adapter VMnet2` interface, on the other hand, is deliberately configured with **no gateway at all**: this is what lets the host PC keep its normal default route to the Internet via Wi-Fi, without VMnet2 overriding it.
* Result: the host PC knows its own LAB subnet (directly connected) but has no route to the remote LAB subnet.

### Fix: add a persistent static route

On **PC1** (PowerShell, run as administrator):

```powershell
route -p add 192.168.20.128 mask 255.255.255.128 192.168.20.126
```

On **PC2** (PowerShell, run as administrator):

```powershell
route -p add 192.168.20.0 mask 255.255.255.128 192.168.20.254
```

* `-p` makes the route persistent (it survives a reboot).
* The first parameter plus `mask` describe the remote LAB prefix (`/25`).
* The last parameter is the **local** OpenWrt's LAN IP (the gateway into the tunnel).

> ⚠️ Do not use `New-NetRoute ... -PolicyStore PersistentStore`: on some Windows builds, that parameter combination triggers the error `Invalid parameter PolicyStore PersistentStore` (System Error 87). The legacy `route -p` command works reliably and persists the route correctly.

### Verification

```powershell
route print -4 | findstr "192.168.20"
```

The route should show up both in the active routing table and in the "Persistent Routes" section. Once added, the host PC can reach the remote LAB's VMs over SSH/RDP, exactly as the LAB VMs already do via OpenWrt.

---

## References

* OpenWrt — 25.12 stable branch: `https://openwrt.org/releases/25.12/start`
* Official OpenWrt x86-64 images: `https://downloads.openwrt.org/releases/25.12.5/targets/x86/64/`
* OpenWrt on VMware: `https://openwrt.org/docs/guide-user/virtualization/vmware`
* OpenWrt package management: `https://openwrt.org/docs/guide-user/additional-software/managing_packages`
* WireGuard on OpenWrt: `https://openwrt.org/docs/guide-user/services/vpn/wireguard/start`
* StarWind V2V Converter — VMDK conversion: `https://www.starwindsoftware.com/v2v-help/CovertingtoVMDK.html`
