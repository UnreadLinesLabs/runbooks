# Add a wired interconnect between the two lab routers

This runbook adds a second, wired path between the two LAB subnets built in
[`openwrt-wireguard-site-to-site/README.md`](../openwrt-wireguard-site-to-site/README.md), alongside the
WireGuard tunnel that runbook already carries over Wi-Fi. It does not touch that tunnel's configuration:
it adds a direct Ethernet cable between the two physical hosts, a new interface on each OpenWrt router, a
point-to-point subnet and static routes, a dedicated firewall zone, and a small operating procedure to
switch which of the two paths carries inter-LAB traffic.

The reason to add it is measured, not assumed. Diagnosing an intermittent DNS failure between
`U01PARVMECN01` (Subnet 2) and `U01PARVMDOM01` (Subnet 1) led to a live capture on both `U01PARVMFWL01`
and `U01PARVMFWL02`, then an `iperf3` UDP test between the two `wg0` tunnel addresses: a real **5.3% UDP
packet loss** at only 5 Mbit/s, with the `wg0` interface counters themselves showing zero errors or drops
on both routers. WireGuard, the routers, and the AD DNS server were all cleared first — the loss traced to
the home Wi-Fi that both PCs share for their WAN uplink, aggravated (though not entirely explained) by the
Freebox having auto-selected a crowded DFS channel. This runbook reproduces that measurement as its
baseline (section 5), then repeats it once the wired link is in place (section 11), so the improvement is
a number, not a claim.

Only one of the two paths carries inter-LAB traffic at a time. Switching between them is a documented,
reversible operation (section 10) — not a one-way migration, and not two paths racing each other on
different route metrics. Wi-Fi keeps doing exactly what it does today, carrying each router's own Internet
access; this runbook only changes how the two LAB subnets reach each other.

## 1. Architecture

```text
                    +----------------------------------+
                    |             Internet              |
                    +----------------------------------+
                                     |
                    +----------------------------------+
                    |             Freebox               |
                    |          192.168.1.254            |
                    +----------------------------------+
                                     |
       +-----------------------------+-----------------------------+
       |  Wi-Fi (WAN, unchanged)                    Wi-Fi (WAN, unchanged)  |
+--------------------------------------+                +--------------------------------------+
|                 PC1                  |                |                 PC2                  |
+--------------------------------------+                +--------------------------------------+
|            U01PARVMFWL01             |                |            U01PARVMFWL02             |
|                                       |                |                                       |
|  wg0   10.10.10.1/30  (existing) <===|==Wi-Fi tunnel==>|  wg0   10.10.10.2/30  (existing)     |
|  eth3  10.20.20.1/30  (new)     <====|===new cable====>|  eth3  10.20.20.2/30  (new)          |
|                                       |                |                                       |
|        LAN  192.168.20.126/25        |                |        LAN  192.168.20.254/25        |
+--------------------------------------+                +--------------------------------------+
                    |                                                         |
+--------------------------------------+                +--------------------------------------+
|       LAB1  (192.168.20.0/25)        |                |      LAB2  (192.168.20.128/25)       |
+--------------------------------------+                +--------------------------------------+
```

Only one of `wg0` or the new wired interface is enabled on each router at any given time — section 10
covers the switch. Both ends use `eth3` for the new link, so the two routers read the same in every
command below — section 7 explains how that alignment is achieved on `U01PARVMFWL01`, which does not
naturally reach a fourth NIC on its own.

## 2. Scope and dependencies

This runbook assumes `openwrt-wireguard-site-to-site/README.md` is fully built and its section 22
end-to-end test (a LAB1 VM pinging a LAB2 VM over `wg0`) already passes.

It adds:

- a physical Ethernet cable directly between PC1 and PC2
- a new bridged VMware network on each PC, bound to a free physical Ethernet adapter
- a third or fourth NIC on `U01PARVMFWL01` and `U01PARVMFWL02` (section 7 explains the numbering)
- a `/30` point-to-point subnet and static routes between the two LAN prefixes
- a dedicated `wired` firewall zone, mirroring the pattern already used for the `wg` zone
- a documented, reversible procedure to switch which path is active (section 10)

It does not touch: the existing WireGuard keys, peers or `wg0` configuration; either router's WAN/Wi-Fi
configuration or masquerading; or any LAB VM's own network settings. Nothing in this runbook keeps both
paths active at the same time — that would need a routing-metric decision between two routes to the same
prefix, and this lab deliberately avoids it by keeping the two paths mutually exclusive instead.

It does not add VLANs. The new link is a dedicated point-to-point cable between exactly two ports, not a
shared trunk — there is nothing on it to separate with a tag. `hostapd-wifi-access-point/README.md` §11
already found that VMware Workstation doesn't reliably relay 802.1Q-tagged frames on a host-only network,
which is why that lab uses a dedicated VMware network per SSID instead of VLAN tagging; the same reasoning
applies here.

## 3. Target configuration

| Item | `U01PARVMFWL01` (PC1) | `U01PARVMFWL02` (PC2) |
| --- | --- | --- |
| New physical link | direct Ethernet cable between PC1 and PC2 | (same cable) |
| VMware network for the link | new bridged network (`VMnet6`), bound to PC1's free physical Ethernet adapter | new bridged network (`VMnet6`), bound to PC2's free physical Ethernet adapter |
| New interface on the router | `eth3` (see section 7 — a placeholder NIC keeps the numbering aligned with `FWL02`) | `eth3` |
| Point-to-point address | `10.20.20.1/30` | `10.20.20.2/30` |
| Route to the remote LAB | `192.168.20.128/25 via 10.20.20.2` | `192.168.20.0/25 via 10.20.20.1` |
| Firewall zone | `wired` | `wired` |
| Existing WireGuard interface | `wg0` — untouched, disabled while `eth3` is active | `wg0` — untouched, disabled while `eth3` is active |

`VMnet6` is a name, not a shared network: each PC configures its own Virtual Network Editor
independently, so `VMnet6` on PC1 and `VMnet6` on PC2 never actually meet — using the same number on both
just keeps the documentation and the LuCI/`ip addr` output easy to read side by side.

## 4. Prerequisites

- `openwrt-wireguard-site-to-site/README.md` built and verified end to end (its section 22).
- One free physical Ethernet port on each PC. Since each PC already reaches the Internet over Wi-Fi
  (the WireGuard runbook's section 7.4), the onboard Ethernet port is normally unused and available for
  this link.
- A standard Ethernet cable. No crossover cable is needed — every NIC involved here auto-negotiates
  MDI/MDI-X. A switch works too instead of a direct cable (see section 6) — same config either way.
- Local administrator rights on both Windows hosts — creating a VMware network and adding a VM network
  adapter both require them.
- `iperf3` on both routers, for the baseline and the verification test:

  **On `U01PARVMFWL01` and `U01PARVMFWL02`:**

  ```sh
  apk update
  apk add iperf3
  ```

## 5. Measure the baseline over the Wi-Fi tunnel

Capture the current state before wiring anything, so section 11's result is a comparison, not a claim.

**On `U01PARVMFWL01`:**

```sh
iperf3 -s
```

**On `U01PARVMFWL02`:**

```sh
iperf3 -c 10.10.10.1 -u -b 5M -t 30
```

Record the receiver-side line from the summary (`Lost/Total Datagrams`) — this lab's own baseline,
captured the same way, was:

| Test | Result |
| --- | --- |
| `ping` (LAB2 VM to `192.168.20.41`, 20 packets) | 0-5% loss, jitter 11-365 ms depending on Wi-Fi channel conditions |
| `iperf3 -u -b 5M -t 30` (`U01PARVMFWL02` → `U01PARVMFWL01` over `wg0`) | **5.3% loss** (748/14119 datagrams), `wg0` interface counters clean (0 errors/drops both sides) |

Also worth a quick check, from any LAB2 host:

```powershell
ping 192.168.20.41 -n 20
```

## 6. Wire the two PCs and create the VMware bridged network

Physically connect the Ethernet cable between a free physical Ethernet port on PC1 and one on PC2.

A direct cable is the simplest option for two PCs, but nothing here requires it: a small unmanaged
switch works identically — connect each PC's free port to the switch instead of to each other. The
`VMnet6` bridging config below and everything from section 7 onward is exactly the same either way, and
a switch is the natural choice if the PCs aren't close enough for one cable or if a third device is
ever added to this link later (section 14).

On **each** PC, in `Edit → Virtual Network Editor → Add Network → VMnet6`:

| Parameter | Value |
| --- | --- |
| Type | Bridged |
| Bridged to | this PC's specific physical Ethernet adapter (avoid "Automatic") |

Binding to a specific adapter matters here for the same reason it mattered for the Wi-Fi bridge in the
WireGuard runbook's section 7.4: with "Automatic", VMware can silently bridge to the wrong NIC if the host
has more than one.

## 7. Add the wired interface to `U01PARVMFWL01` and `U01PARVMFWL02`

OpenWrt numbers NICs in the order they exist in the VM's hardware, starting at `eth0`. The two routers
don't start from the same count — `U01PARVMFWL01` has two existing NICs (LAN, WAN), `U01PARVMFWL02` has
three (LAN, WAN, and the `VMnet3` uplink to `U01PARVMRAP01` added in `hostapd-wifi-access-point/README.md`)
— so reaching `eth3` on both takes a different number of adapters on each.

**On `U01PARVMFWL02`:** add one network adapter — this is its fourth NIC, `eth3`.

```text
Network Adapter → Custom: Specific virtual network → VMnet6
Connected / Connect at power on : Enabled
```

**On `U01PARVMFWL01`:** add two network adapters, in order — the goal is to reach `eth3` here too, so the
numbering reads the same on both routers in every command that follows. The first is a deliberate,
unused placeholder that occupies the `eth2` slot; the second is the real link.

```text
Network Adapter (3rd)  → not connected to any network — leave "Connected" and
                          "Connect at power on" both unchecked
Network Adapter (4th)  → Custom: Specific virtual network → VMnet6
                          Connected / Connect at power on : Enabled
```

The placeholder gets no `uci` configuration at all — it exists only to occupy `eth2` so the real link
becomes `eth3`, matching `U01PARVMFWL02`. Leaving an intentionally unused adapter is unusual enough to
call out on screen and in this note, so a future reader doesn't mistake it for a leftover.

Confirm the result on each router before continuing — don't assume the count above stays accurate if
either VM's hardware changes later:

```sh
ip addr
```

The new link's interface has no address yet; the placeholder on `FWL01` shows up with no carrier and no
address either, which is expected.

## 8. Configure the point-to-point addressing and static routes

**On `U01PARVMFWL01`:**

```sh
uci set network.wired='interface'
uci set network.wired.device='eth3'
uci set network.wired.proto='static'
uci set network.wired.ipaddr='10.20.20.1'
uci set network.wired.netmask='255.255.255.252'
uci commit network

uci add network route
uci set network.@route[-1].interface='wired'
uci set network.@route[-1].target='192.168.20.128'
uci set network.@route[-1].netmask='255.255.255.128'
uci set network.@route[-1].gateway='10.20.20.2'
uci commit network

/etc/init.d/network restart
```

**On `U01PARVMFWL02`:**

```sh
uci set network.wired='interface'
uci set network.wired.device='eth3'
uci set network.wired.proto='static'
uci set network.wired.ipaddr='10.20.20.2'
uci set network.wired.netmask='255.255.255.252'
uci commit network

uci add network route
uci set network.@route[-1].interface='wired'
uci set network.@route[-1].target='192.168.20.0'
uci set network.@route[-1].netmask='255.255.255.128'
uci set network.@route[-1].gateway='10.20.20.1'
uci commit network

/etc/init.d/network restart
```

Verify the interface and route came up on both sides. `wired` is the UCI logical interface name, not
the Linux device name — check the actual device (`eth3`) or ask `netifd` for the logical interface's
status:

```sh
ip addr show eth3
ifstatus wired
```

Don't test with `ping` yet — `eth3`/`wired` isn't in any firewall zone at this point, so OpenWrt drops
the traffic by default even though the addressing and route above are already correct. That test comes
at the end of section 9, once the zone exists.

## 9. Configure the firewall zone for the wired link

The same `lan ↔ zone` forwarding pattern already used for the `wg` zone in
`openwrt-wireguard-site-to-site/README.md` §21.4, applied to the new `wired` interface instead. No
masquerading and no WAN-facing rule are needed — this interface is never attached to the `wan` zone.

**On `U01PARVMFWL01` and `U01PARVMFWL02`:**

```sh
uci add firewall zone
uci set firewall.@zone[-1].name='wired'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci add_list firewall.@zone[-1].network='wired'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wired'
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wired'
uci set firewall.@forwarding[-1].dest='lan'
uci commit firewall

/etc/init.d/firewall restart
```

Now the link can actually be tested end to end:

```sh
ping -c 20 10.20.20.1   # from FWL02
ping -c 20 10.20.20.2   # from FWL01
```

## 10. Switch between the wired link and the Wi-Fi tunnel

Exactly one of `wg0` or `wired` is enabled on each router at any time. Run the same mode on **both**
routers together — mixing modes between the two ends leaves the two LABs unable to reach each other at
all, since each end would be listening on a different path.

**Switch to the wired link** (on `U01PARVMFWL01` and `U01PARVMFWL02`):

```sh
uci set network.wg0.disabled='1'
uci set network.wired.disabled='0'
uci commit network
/etc/init.d/network restart
```

**Switch back to the Wi-Fi tunnel** (on `U01PARVMFWL01` and `U01PARVMFWL02`):

```sh
uci set network.wired.disabled='1'
uci set network.wg0.disabled='0'
uci commit network
/etc/init.d/network restart
```

Neither command removes any configuration — `disabled='1'` just stops that interface from coming up.
Switching back later is the same two lines in reverse.

Verify which path is actually active after switching:

```sh
ip route | grep 192.168.20
wg show          # only meaningful once wg0 is enabled again
```

## 11. Verify the improvement

Repeat the exact tests from section 5, now with the wired link active.

**On `U01PARVMFWL01`:**

```sh
iperf3 -s
```

**On `U01PARVMFWL02`:**

```sh
iperf3 -c 10.20.20.1 -u -b 5M -t 30
```

And from a LAB2 host:

```powershell
ping 192.168.20.41 -n 20
```

| Test | Before (`wg0` / Wi-Fi) | After (`eth3` / wired) |
| --- | --- | --- |
| `ping` (continuous) | 0-5% loss, jitter 11-365 ms | 0% loss, 3-10 ms, no timeouts |
| `iperf3 -u -b 5M -t 30` | 5.3% loss | 0% loss (0/12950 datagrams), jitter 0.158 ms |

## 12. Expected final state

```text
ip addr show <new interface>          : shows 10.20.20.1/30 (FWL01) or 10.20.20.2/30 (FWL02)
ip route | grep 192.168.20            : the remote LAB prefix via the currently active path only
ping between a LAB1 and a LAB2 VM     : succeeds, near-zero loss, with the wired link active
wg show (wired link active)           : no interface — wg0 is disabled
wg show (Wi-Fi tunnel active)         : a recent handshake, as before
```

## 13. Update the infrastructure inventory

`reference/vm-inventory.md` records IP addressing per VM. Once this lab is actually built, add the new
point-to-point address to the `U01PARVMFWL01` and `U01PARVMFWL02` rows (or a dedicated table, following
the pattern already used there for the Wi-Fi client networks in its section 3) — not before, since the
inventory should reflect what is actually deployed rather than what is planned.

## 14. Next step — extend the wired backbone

Nothing currently depends on this. If a third PC or subnet is ever added to the lab, the same pattern —
a dedicated point-to-point cable and interface, no VLANs, one active path at a time — extends directly.
The reserved `UnreadLines-Mobile` and `UnreadLines-Corp` Wi-Fi segments (`VMnet4`, `VMnet5`, still
unbuilt per `reference/vm-inventory.md` §3) are unrelated to this wired link and unaffected by it.

## 15. References

- [Build an OpenWrt network lab with a WireGuard site-to-site tunnel](../openwrt-wireguard-site-to-site/README.md)
- [Deploy a Linux `hostapd` Wi-Fi access point](../hostapd-wifi-access-point/README.md)
- [Virtual machine inventory](../reference/vm-inventory.md)
- [OpenWrt — network configuration](https://openwrt.org/docs/guide-user/network/network_configuration)

---

*Part of [UnreadLines Labs](https://youtube.com/@unreadlineslabs) — real-world enterprise infrastructure, identity, and security labs, documented the way nobody else bothers to.*
