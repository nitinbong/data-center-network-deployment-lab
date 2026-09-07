# Phase 1 — Topology Build, Cabling, Hostnames & Port Descriptions

**Goal:** get every device placed, cabled, named, and labeled. No VLANs yet. At the end of this phase every link should show green and the three servers should ping each other inside default VLAN 1.

**Estimated time:** 45–60 minutes.

---

## 1.1 Place the devices

Open Packet Tracer 8.x and drag in the following. Component names are exactly as they appear in the PT device browser.

| Qty | PT category | Device model | Rename to |
|---|---|---|---|
| 1 | Network Devices → Switches | `3650-24PS` | CORE-SW-01 |
| 2 | Network Devices → Switches | `2960-24TT` | ACCESS-SW-01, ACCESS-SW-02 |
| 3 | End Devices → End Devices | `Server` | DC-SERVER-01, DC-SERVER-02, DC-SERVER-03 |
| 3 | End Devices → End Devices | `PC` | NOC-PC-01, OPS-PC-01, TEST-PC-01 |

If `3650-24PS` is not present in your PT install, use `3560-24PS` instead. Everything in this project works identically on the 3560 — the only difference is interface naming: the 3650 uses `GigabitEthernet1/0/1`, the 3560 uses `GigabitEthernet0/1`. If you use the 3560, substitute accordingly everywhere in this project and note the substitution in your README.

**Rename devices now, not later.** Click the device, go to the **Config** tab, and set the **Display Name** field. The display name is what appears on the topology canvas and in your screenshots. It is separate from the IOS hostname you will set in step 1.4 — set both, and make them match.

### Arrange the canvas

Lay the devices out to match the logical hierarchy, because a screenshot of a tangled topology reads as careless work:

- CORE-SW-01 centered along the top
- ACCESS-SW-01 lower left, ACCESS-SW-02 lower right
- The three servers in a row beneath ACCESS-SW-01
- The three workstations in a row beneath ACCESS-SW-02

---

## 1.2 Add the second NIC to DC-SERVER-01

DC-SERVER-01 needs a second network interface for the dedicated management VLAN.

1. Click DC-SERVER-01 → **Physical** tab.
2. Click the **power switch** on the device graphic to power it off. The module bay will not accept a card while the device is powered on.
3. From the MODULES list on the left, drag **`PT-HOST-NM-1CFE`** into the empty module slot on the chassis graphic.
4. Click the power switch again to power the device back on.

The server now has `FastEthernet0` (the original, built-in NIC) and a second interface from the new module. Check the **Config** tab — you should see two interfaces listed under INTERFACE in the left sidebar. Note which name PT assigned to the new one; it varies slightly by version.

**Documentation mapping:** built-in NIC = NIC1 (data, VLAN 10). Module NIC = NIC2 (management, VLAN 20).

---

## 1.3 Run the cables

Use the **Connections** tool (the lightning bolt icon).

### Host connections — Copper Straight-Through

| From | Port | To | Port |
|---|---|---|---|
| DC-SERVER-01 | NIC1 (FastEthernet0) | ACCESS-SW-01 | Fa0/1 |
| DC-SERVER-02 | FastEthernet0 | ACCESS-SW-01 | Fa0/2 |
| DC-SERVER-03 | FastEthernet0 | ACCESS-SW-01 | Fa0/3 |
| DC-SERVER-01 | NIC2 (module) | ACCESS-SW-02 | Fa0/5 |
| NOC-PC-01 | FastEthernet0 | ACCESS-SW-02 | Fa0/10 |
| OPS-PC-01 | FastEthernet0 | ACCESS-SW-02 | Fa0/11 |
| TEST-PC-01 | FastEthernet0 | ACCESS-SW-02 | Fa0/20 |

### Uplinks — Copper Cross-Over

| From | Port | To | Port |
|---|---|---|---|
| ACCESS-SW-01 | Gi0/1 | CORE-SW-01 | Gi1/0/1 |
| ACCESS-SW-02 | Gi0/1 | CORE-SW-01 | Gi1/0/2 |

**Why cross-over between switches:** traditionally, connections between like devices such as switch-to-switch used crossover cables, because both ends transmit on the same pin pairs. Modern Ethernet devices commonly support Auto-MDI/MDIX, so a straight-through cable may also work automatically. This lab uses crossover here to clearly represent the traditional switch-to-switch connection.

**Port discipline matters.** Connect to the exact ports listed. Every document in this repository references these port numbers, and NET-007 depends on documentation matching reality. If you patch a server into Fa0/4 because it was closer to the mouse pointer, your port map is now wrong and every downstream document inherits the error. This is exactly how real data centers end up with documentation nobody trusts.

### Watch the link lights

After each cable, watch the small triangle at each end:

- **Amber/orange** — the port is up physically but STP is running listening/learning. Normal, resolves in seconds.
- **Green** — link up and forwarding.
- **Red** — link down. Wrong cable type, wrong port, or the device is powered off.

This is the simulated version of what you read off a real switch faceplate. Link LED off means a physical layer problem — cable, transceiver, or port. Link LED on with no traffic means the physical layer is fine and the problem is higher up. That distinction is the first branch of the troubleshooting runbook.

---

## 1.4 Configure hostnames and port descriptions

Open the **CLI** tab on each switch and enter the following. When prompted with `Continue with configuration dialog? [yes/no]:` answer `no`.

### CORE-SW-01

```
enable
configure terminal
hostname CORE-SW-01
no ip domain-lookup
banner motd #Authorized access only. CORE-SW-01 / RACK-A02 U40#
!
interface GigabitEthernet1/0/1
 description TRUNK to ACCESS-SW-01 Gi0/1 -- RACK-A01 TOR uplink
 exit
interface GigabitEthernet1/0/2
 description TRUNK to ACCESS-SW-02 Gi0/1 -- RACK-A02 TOR uplink
 exit
!
end
copy running-config startup-config
```

Press Enter at the `Destination filename [startup-config]?` prompt.

### ACCESS-SW-01

```
enable
configure terminal
hostname ACCESS-SW-01
no ip domain-lookup
banner motd #Authorized access only. ACCESS-SW-01 / RACK-A01 U42#
!
interface FastEthernet0/1
 description DC-SERVER-01 NIC1 DATA | A01-SRV001-NIC1__A01-SW001-Fa0-1
 exit
interface FastEthernet0/2
 description DC-SERVER-02 NIC1 DATA | A01-SRV002-NIC1__A01-SW001-Fa0-2
 exit
interface FastEthernet0/3
 description DC-SERVER-03 NIC1 DATA | A01-SRV003-NIC1__A01-SW001-Fa0-3
 exit
interface GigabitEthernet0/1
 description UPLINK to CORE-SW-01 Gi1/0/1
 exit
!
end
copy running-config startup-config
```

### ACCESS-SW-02

```
enable
configure terminal
hostname ACCESS-SW-02
no ip domain-lookup
banner motd #Authorized access only. ACCESS-SW-02 / RACK-A02 U42#
!
interface FastEthernet0/5
 description DC-SERVER-01 NIC2 MGMT | A01-SRV001-NIC2__A02-SW001-Fa0-5
 exit
interface FastEthernet0/10
 description NOC-PC-01 | A02-NOC001-NIC1__A02-SW001-Fa0-10
 exit
interface FastEthernet0/11
 description OPS-PC-01 | A02-OPS001-NIC1__A02-SW001-Fa0-11
 exit
interface FastEthernet0/20
 description TEST-PC-01 UNTRUSTED | A02-TST001-NIC1__A02-SW001-Fa0-20
 exit
interface GigabitEthernet0/1
 description UPLINK to CORE-SW-01 Gi1/0/2
 exit
!
end
copy running-config startup-config
```

### What `no ip domain-lookup` does

Without it, any mistyped command gets interpreted as a hostname and the switch stalls for 30+ seconds trying to resolve it via broadcast DNS. It is the first line most engineers type on a lab switch. Worth being able to explain in an interview.

### Why port descriptions are the point of this phase

The descriptions embed the cable label from the design document. Six months later, `show interfaces description` tells the next technician what is plugged into every port without anyone walking the floor with a flashlight. This is the switch-side half of the documentation story — the CSV port map is the other half, and the two must agree.

---

## 1.5 Assign static IP addresses

VLANs do not exist yet, so every port is in default VLAN 1. Assigning the server IPs now gives you a meaningful Phase 1 test.

On each device: **Desktop** tab → **IP Configuration** → select **Static**.

| Device | Interface | IP address | Subnet mask | Gateway | DNS |
|---|---|---|---|---|---|
| DC-SERVER-01 | NIC1 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 | 192.168.10.11 |
| DC-SERVER-01 | NIC2 | 192.168.20.11 | 255.255.255.0 | 192.168.20.1 | — |
| DC-SERVER-02 | FastEthernet0 | 192.168.10.12 | 255.255.255.0 | 192.168.10.1 | 192.168.10.11 |
| DC-SERVER-03 | FastEthernet0 | 192.168.10.13 | 255.255.255.0 | 192.168.10.1 | 192.168.10.11 |
| TEST-PC-01 | FastEthernet0 | 192.168.40.50 | 255.255.255.0 | 192.168.40.1 | 192.168.10.11 |

For DC-SERVER-01, use the interface dropdown at the top of the IP Configuration window to switch between NIC1 and NIC2.

**Leave NOC-PC-01 and OPS-PC-01 set to DHCP.** They will fail to get an address right now — that is expected and correct. They are waiting on the DHCP pool you build in Phase 5.

The gateway addresses point at SVIs that do not exist yet. That is fine. Configuring a gateway does not require it to be reachable; the host just will not be able to leave its subnet until Phase 4.

---

## 1.6 Validation — run these and record the results

### Test 1: link status on both access switches

```
show interfaces status
```

Expected on ACCESS-SW-01: `Fa0/1`, `Fa0/2`, `Fa0/3`, and `Gi0/1` show status `connected` and VLAN `1`. Every other port shows `notconnect`.

Expected on ACCESS-SW-02: `Fa0/5`, `Fa0/10`, `Fa0/11`, `Fa0/20`, and `Gi0/1` show `connected`, VLAN `1`.

Everything sitting in VLAN 1 is the correct Phase 1 result. VLAN assignment is Phase 2.

### Test 2: port descriptions took

```
show interfaces description
```

Every configured port should show your description text. If a description is missing, you configured it on the wrong interface.

### Test 3: MAC address table

```
show mac address-table
```

You may see few or no dynamic entries yet — a switch only learns a MAC after that device transmits a frame. Run Test 4 first, then run this again. Watching entries appear after traffic flows is the clearest demonstration of how MAC learning actually works, and it is worth a screenshot.

### Test 4: same-subnet, same-switch connectivity

From DC-SERVER-01 → **Desktop** → **Command Prompt**:

```
ping 192.168.10.12
ping 192.168.10.13
```

Both should succeed. All three servers are in the same subnet on the same switch in the same VLAN, so this works with no routing at all — the switch just forwards frames by MAC address.

Then, from **DC-SERVER-02** (192.168.10.12) → **Desktop** → **Command Prompt**:

```
ping 192.168.20.11
```

This should **fail**. DC-SERVER-02 is in 192.168.10.0/24 and the target management NIC is in 192.168.20.0/24. Crossing between subnets requires a router, and no SVI exists yet. A failure here is a pass for Phase 1.

**Why the test is run from DC-SERVER-02 and not from DC-SERVER-01:** 192.168.20.11 is one of DC-SERVER-01's own interfaces. A host pinging an address assigned to itself is normally answered by its own IP stack without any packet ever reaching the network, so that test would prove nothing about routing either way. Testing from a genuinely separate host is the only way to actually exercise the path.

### Test 5: re-check the MAC table

Go back to ACCESS-SW-01 and run `show mac address-table` again. You should now see dynamic entries for the server MACs mapped to Fa0/1, Fa0/2, Fa0/3, all in VLAN 1.

Cross-check one entry: pick a server, open its **Config** tab, read its MAC address, and confirm the switch learned it on the port your documentation says it should be on. **That cross-check is the core skill of this entire project** — it is how a technician confirms which physical device is on which physical port without touching a cable.

---

## 1.7 Screenshots to capture before moving on

Capture only what `screenshots/README.md` lists for this phase. The project targets roughly 25–30 screenshots in total (12–15 baseline, 2 per incident), so capture the clearest single view of each capability rather than every command you run.


Save the file as `packet-tracer/data-center-network.pkt`.

---

## 1.8 Report back before Phase 2

Paste the actual output of the following, exactly as it appears — do not clean it up:

1. `show interfaces status` from ACCESS-SW-01
2. `show interfaces status` from ACCESS-SW-02
3. `show mac address-table` from ACCESS-SW-01
4. The result of `ping 192.168.10.12` from DC-SERVER-01 (expected: success)
5. The result of `ping 192.168.20.11` from DC-SERVER-02 (expected: failure)
6. The interface name PT assigned to DC-SERVER-01's second NIC

Also flag anything that did not behave as described above — a red link, a port that stayed `notconnect`, a ping that failed when it should have worked. Those are worth diagnosing now rather than inheriting into Phase 2, and a real fault you actually hit is better portfolio material than a scripted one.

No incident report or command output goes into this repository until it has been run and observed. Phase 2 (VLAN creation and access port assignment) starts once you post these results.
