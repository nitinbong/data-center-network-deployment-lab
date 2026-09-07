# Phase 2 & 3 — VLAN Creation, Access Ports & Trunk Links

**Prerequisite:** Phase 1 complete. All links green, hostnames and port descriptions set, static IPs assigned.

**Goal of Phase 2:** create the four lab VLANs on all three switches and assign every host port to the correct VLAN as an access port.

**Goal of Phase 3:** carry those VLANs between switches over 802.1Q trunk links, and prove the difference between an access port and a trunk port.

**Estimated time:** 60–75 minutes combined.

---

# PHASE 2 — VLANs and access ports

## 2.1 The concept, before the commands

A switch with no VLANs is one flat broadcast domain — every device hears every broadcast from every other device. VLANs partition that single physical switch into multiple isolated logical switches. Two ports in different VLANs on the same physical switch cannot talk to each other at all without a router, exactly as if they were plugged into two switches with no cable between them.

An **access port** belongs to exactly one VLAN and carries untagged traffic for that VLAN only. Servers, workstations, printers — anything that is an endpoint — connects to an access port. The endpoint has no idea VLANs exist.

That single-VLAN property is what makes NET-001 (wrong VLAN) possible: the cable is fine, the link light is green, the switch port is up, the server's IP configuration is correct — and it still cannot reach anything, because the port was placed in the wrong logical switch.

---

## 2.2 Create the VLANs

VLANs must exist on **every switch that carries them**, including switches where the VLAN has no local ports. CORE-SW-01 has no VLAN 40 access ports, but it needs VLAN 40 defined because it will host that VLAN's gateway and the trunks must carry it.

Run this identical block on **all three switches** — CORE-SW-01, ACCESS-SW-01, and ACCESS-SW-02:

```
enable
configure terminal
!
vlan 10
 name SERVERS
 exit
vlan 20
 name MANAGEMENT
 exit
vlan 30
 name OPERATIONS
 exit
vlan 40
 name UNTRUSTED
 exit
vlan 99
 name NATIVE
 exit
!
end
```

Do not save yet — you will save at the end of the phase.

### Why VLAN 99 exists

VLAN 99 is the native VLAN for the trunk links. Untagged frames arriving on an 802.1Q trunk get placed in the native VLAN. Leaving that as the default (VLAN 1) means stray untagged traffic lands in the same VLAN as the switch's own default management interface. Moving it to an otherwise unused VLAN is standard hygiene. No host will ever be assigned to VLAN 99.

---

## 2.3 Assign access ports

### ACCESS-SW-01

```
enable
configure terminal
!
interface range FastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 10
 exit
!
end
copy running-config startup-config
```

`interface range` applies the same configuration to a block of ports in one pass. On a 48-port switch where forty ports are all server-facing VLAN 10 ports, this is the difference between one command and forty.

### ACCESS-SW-02

```
enable
configure terminal
!
interface FastEthernet0/5
 switchport mode access
 switchport access vlan 20
 exit
!
interface range FastEthernet0/10 - 11
 switchport mode access
 switchport access vlan 30
 exit
!
interface FastEthernet0/20
 switchport mode access
 switchport access vlan 40
 exit
!
end
copy running-config startup-config
```

### CORE-SW-01

CORE-SW-01 has no host-facing access ports in this design — only the two trunk uplinks. Just save what you created:

```
enable
copy running-config startup-config
```

### What `switchport mode access` actually does

Without it, the port stays in dynamic auto/desirable mode and could negotiate itself into a trunk if the far end asks. Explicitly setting `access` mode means the port will only ever be an access port, regardless of what is plugged in. Hardcoding port mode rather than relying on negotiation is standard practice on host-facing ports.

---

## 2.4 Phase 2 validation

### Test 1: VLAN database

On each switch:

```
show vlan brief
```

Confirm all five VLANs exist with the correct names, and that ports appear under the correct VLAN. On ACCESS-SW-01, `Fa0/1`, `Fa0/2`, `Fa0/3` should be listed under VLAN 10. On ACCESS-SW-02, `Fa0/5` under VLAN 20, `Fa0/10` and `Fa0/11` under VLAN 30, `Fa0/20` under VLAN 40.

**Read the unassigned ports too.** Every port you did not configure is still sitting in VLAN 1. That is correct and expected, and noticing it is the habit that catches a miscabled server later.

### Test 2: port-level VLAN confirmation

```
show interfaces status
```

The VLAN column should now show `10`, `20`, `30`, `40` instead of `1` on your configured ports. This is the single fastest command for answering "what VLAN is this port in?" — one line per port, VLAN and link state together.

### Test 3: same-VLAN connectivity still works

From DC-SERVER-01:

```
ping 192.168.10.12
ping 192.168.10.13
```

Both should still succeed. All three servers moved into VLAN 10 together, so they remain in the same broadcast domain.

### Test 4: cross-VLAN isolation

From **OPS-PC-01** (VLAN 30) — it has no IP yet since DHCP is not built, so temporarily set it to static `192.168.30.50 / 255.255.255.0 / gateway 192.168.30.1` for this test (`.50` sits outside the future DHCP range of `.101`–`.254`).

```
ping 192.168.10.11
```

This must **fail**. VLAN 30 and VLAN 10 are separate broadcast domains with no routing between them yet. Failure here proves your VLAN isolation is genuinely working — and it is the exact behavior you will remove in Phase 4.

Set OPS-PC-01 back to DHCP when done.

### Test 5: MAC address table now shows VLANs

```
show mac address-table
```

On ACCESS-SW-01, the server MAC entries should now show VLAN `10` instead of `1`. The MAC table is per-VLAN — that column is how the switch knows which logical switch a device lives on.

---

## 2.5 Phase 2 screenshots

Capture only what `screenshots/README.md` lists for this phase. The project targets roughly 25–30 screenshots in total (12–15 baseline, 2 per incident), so capture the clearest single view of each capability rather than every command you run.


---

# PHASE 3 — Trunk links

## 3.1 The concept, before the commands

An access port carries **one** VLAN. A trunk port carries **many**.

Physically there is one cable between ACCESS-SW-01 and CORE-SW-01, but traffic from four different VLANs must cross it. 802.1Q solves this by inserting a 4-byte VLAN tag into each frame's header as it leaves the trunk, and stripping it on the far side. The tag is how the receiving switch knows which VLAN a frame belongs to.

Frames in the **native VLAN** cross the trunk untagged — that is the one exception, and it is why native VLAN must match on both ends of a trunk.

The practical consequence, and the basis of NET-002: a trunk has an **allowed VLAN list**. If a VLAN is not on that list, its frames are silently discarded at the trunk. Devices on the same switch keep working perfectly; devices reaching each other *across* switches break. That symptom — works locally, fails between switches — points straight at the trunk.

---

## 3.2 Configure the trunks

### ACCESS-SW-01

```
enable
configure terminal
!
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,99
 exit
!
end
copy running-config startup-config
```

### ACCESS-SW-02

```
enable
configure terminal
!
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,99
 exit
!
end
copy running-config startup-config
```

### CORE-SW-01

Use `switchport trunk encapsulation dot1q` only if the selected Packet Tracer switch model supports/requires it. If IOS rejects the command, omit it and configure `switchport mode trunk` directly.

```
enable
configure terminal
!
interface range GigabitEthernet1/0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,30,40,99
 exit
!
end
copy running-config startup-config
```

Use `switchport trunk encapsulation dot1q` only if the selected Packet Tracer switch model supports/requires it. If IOS rejects the command, omit it and configure `switchport mode trunk` directly. If you are using a 3560 instead of a 3650, use `interface range GigabitEthernet0/1 - 2`.

### A note on the allowed list

`switchport trunk allowed vlan 10,20,30,40,99` is more restrictive than the default, which allows all VLANs 1–4094. Explicitly listing allowed VLANs is good practice — it limits broadcast propagation and stops a VLAN created on one switch from unexpectedly flooding across the whole network.

Be careful with the syntax. `switchport trunk allowed vlan 10` **replaces** the entire list with just VLAN 10. To append, you must use `switchport trunk allowed vlan add 10`. Overwriting a trunk's allowed list with a single VLAN is one of the most common real-world outage causes on a change window, and it is precisely the fault NET-002 reproduces.

---

## 3.3 Phase 3 validation

### Test 1: trunk status

On each switch:

```
show interfaces trunk
```

You should see four columns of information. Confirm:

- **Mode** is `on`, **Encapsulation** is `802.1q`, **Status** is `trunking`
- **Native vlan** reads `99` on both ends of each link
- **Vlans allowed on trunk** lists `10,20,30,40,99`
- **Vlans in spanning tree forwarding state and not pruned** lists the VLANs actually passing traffic

The last two lines differ in meaning. "Allowed" is what you configured. "Forwarding and not pruned" is what is genuinely crossing the wire right now. When those two disagree, you have found your problem.

### Test 2: VLANs propagated

```
show vlan brief
```

On the access switches, `Gi0/1` should no longer be listed under any single VLAN. A trunk port does not belong to a VLAN — it carries all of them. **If you still see `Gi0/1` listed under VLAN 1, the trunk did not form.**

### Test 3: true end-to-end trunk test

This is the payoff test, and it is deliberately built so the traffic has no choice but to cross both trunks.

Every VLAN 10 device currently sits on ACCESS-SW-01, so any ping between them stays local and proves nothing about trunking. To create a genuine cross-switch path, temporarily turn TEST-PC-01 into a VLAN 10 endpoint on the *other* switch.

**Step 1 — move the port.** On ACCESS-SW-02:

```
enable
configure terminal
interface FastEthernet0/20
 switchport access vlan 10
 exit
end
show interfaces status
```

Confirm `Fa0/20` now shows VLAN `10`.

**Step 2 — readdress the host.** On TEST-PC-01 → **Desktop** → **IP Configuration** → **Static**:

| Field | Value |
|---|---|
| IP address | 192.168.10.50 |
| Subnet mask | 255.255.255.0 |
| Default gateway | leave blank for this test |
| DNS server | leave blank for this test |

The gateway is intentionally left empty. This test is pure Layer 2 within VLAN 10 — no routing is involved and none should be. If it succeeds with no gateway configured, you have proven the trunk works and not accidentally proven that something else routed around the problem.

**Step 3 — ping across the fabric.** From TEST-PC-01 → **Command Prompt**:

```
ping 192.168.10.11
```

This should **succeed**. Trace the path it must take:

```
TEST-PC-01
  -> ACCESS-SW-02 Fa0/20   (access port, VLAN 10, untagged)
  -> ACCESS-SW-02 Gi0/1    (trunk, frame tagged VLAN 10)
  -> CORE-SW-01 Gi1/0/2    (trunk, tag read)
  -> CORE-SW-01 Gi1/0/1    (trunk, frame re-tagged VLAN 10)
  -> ACCESS-SW-01 Gi0/1    (trunk, tag stripped)
  -> ACCESS-SW-01 Fa0/1    (access port, VLAN 10, untagged)
  -> DC-SERVER-01 NIC1
```

Two trunk links, three switches, one broadcast domain stretched across all of it. A successful reply here is unambiguous evidence that VLAN 10 is traversing both trunks correctly. Screenshot it.

**Step 4 — confirm on the core.** While the ping is running, on CORE-SW-01:

```
show mac address-table vlan 10
```

You should see DC-SERVER-01's MAC learned on `Gi1/0/1` and TEST-PC-01's MAC learned on `Gi1/0/2`. CORE-SW-01 has no VLAN 10 access ports whatsoever, so both entries can only have arrived over trunks.

### Test 4: break the trunk and prove the failure

Same setup, still with TEST-PC-01 in VLAN 10 at 192.168.10.50. This is a rehearsal for NET-002, run deliberately and reverted immediately. **Do not save the configuration during this test.**

On ACCESS-SW-01, remove VLAN 10 from the uplink's allowed list:

```
enable
configure terminal
interface GigabitEthernet0/1
 switchport trunk allowed vlan 20,30,40,99
 exit
end
show interfaces trunk
```

Confirm VLAN 10 no longer appears in the allowed list.

Now repeat the ping from TEST-PC-01:

```
ping 192.168.10.11
```

This must **fail** — request timed out. Screenshot it.

Then, without changing anything else, ping between two servers that are both on ACCESS-SW-01. From DC-SERVER-02:

```
ping 192.168.10.11
```

This still **succeeds**. That traffic never touches a trunk.

**That contrast is the entire diagnostic signature of a trunk fault:** devices in the same VLAN on the same switch communicate normally, while devices in the same VLAN on different switches cannot reach each other at all. Nothing about the access ports, the IP addressing, or the link lights looks wrong. Capture both results side by side — the failed cross-switch ping and the successful local ping are far more convincing together than either alone.

### Test 5: restore and re-verify

Put VLAN 10 back on the trunk:

```
enable
configure terminal
interface GigabitEthernet0/1
 switchport trunk allowed vlan 10,20,30,40,99
 exit
end
show interfaces trunk
```

Repeat the ping from TEST-PC-01 to 192.168.10.11. It should **succeed** again. Screenshot it.

Break, observe, fix, re-verify — that is the full cycle every incident report in this project follows, and you have now run it once end to end.

### Test 6: return TEST-PC-01 to its production state

Do not skip this. TEST-PC-01 belongs in VLAN 40, and leaving it in VLAN 10 would silently invalidate the ACL work in Phase 7 and the port map in your documentation.

On ACCESS-SW-02:

```
enable
configure terminal
interface FastEthernet0/20
 switchport access vlan 40
 exit
end
copy running-config startup-config
```

On TEST-PC-01 → **Desktop** → **IP Configuration** → **Static**:

| Field | Value |
|---|---|
| IP address | 192.168.40.50 |
| Subnet mask | 255.255.255.0 |
| Default gateway | 192.168.40.1 |
| DNS server | 192.168.10.11 |

Verify with `show interfaces status` on ACCESS-SW-02 that `Fa0/20` reads VLAN `40` again, and confirm the port description still matches. Then save on all three switches.

Returning a device to its documented state after testing, and verifying you actually did, is the habit that prevents NET-007 — documentation that no longer matches reality.

---

## 3.4 Phase 3 screenshots

Capture only what `screenshots/README.md` lists for this phase. The project targets roughly 25–30 screenshots in total (12–15 baseline, 2 per incident), so capture the clearest single view of each capability rather than every command you run.


---

## 3.5 Report back before Phase 4

Paste the actual output of:

1. `show vlan brief` from ACCESS-SW-01
2. `show vlan brief` from ACCESS-SW-02
3. `show interfaces status` from ACCESS-SW-02
4. `show interfaces trunk` from CORE-SW-01
5. `show mac address-table vlan 10` from CORE-SW-01 during the end-to-end test
6. The three ping results from TEST-PC-01 to 192.168.10.11 — success, failure with VLAN 10 pruned, success after restore
7. `show interfaces status` from ACCESS-SW-02 confirming Fa0/20 is back on VLAN 40

Flag anything that behaved differently — particularly if a trunk showed status `not-trunking`. If `switchport trunk encapsulation dot1q` was rejected, simply note that and confirm the trunk formed without it. Those are worth resolving before routing goes on top.

Phase 4 (enabling `ip routing`, creating the four SVIs, and turning the isolation you just proved into controlled reachability) starts once you post these.
