# NET-002 — Required VLAN Missing From Trunk

| Field | Value |
|---|---|
| **Incident ID** | NET-002 |
| **Priority** | P1 — all cross-switch VLAN 10 traffic failing |
| **Category** | Layer 2 / 802.1Q trunk misconfiguration |
| **Affected Device** | All VLAN 10 hosts, when accessed from another switch |
| **Affected Switch** | ACCESS-SW-01 (SW-A01-001) |
| **Affected Port** | Gi0/1 (trunk uplink to CORE-SW-01 Gi1/0/1) |
| **Affected VLAN** | 10 (SERVERS) |
| **Detected by** | User reports of server unreachability from operations workstations |
| **Status** | Prepared — execution/evidence pending |

---

## Issue

VLAN 10 was removed from the allowed VLAN list on the ACCESS-SW-01 uplink trunk. All VLAN 10 traffic crossing between switches was silently discarded at the trunk, isolating the server VLAN from every other switch in the network while leaving intra-switch server traffic fully operational.

## Initial symptoms

- NOC-PC-01 and OPS-PC-01 (VLAN 30, on ACCESS-SW-02) could not reach any server in VLAN 10
- DC-SERVER-01, DC-SERVER-02 and DC-SERVER-03 could still ping each other normally
- No server could reach its default gateway at 192.168.10.1
- All link lights were green; no port was down
- `show vlan brief` on ACCESS-SW-01 showed VLAN 10 present with all three server ports correctly assigned

**The diagnostic signature:** devices in the same VLAN on the **same switch** communicate normally, while devices in the same VLAN on **different switches** cannot reach each other at all. That contrast points at the trunk and nothing else — access ports, VLAN membership, IP addressing and cabling are all provably fine.

## Fault injection (for reproduction)

Run on ACCESS-SW-01:

```
enable
configure terminal
interface GigabitEthernet0/1
 switchport trunk allowed vlan 20,30,40,99
 exit
end
```

Note that this is not an exotic command — it is what happens when a technician intends `switchport trunk allowed vlan add <id>` and omits the `add` keyword, replacing the entire list instead of appending to it.

## Investigation

**Step 1 — physical/link state.** All ports `connected`, all links green. Layer 1 ruled out.

**Step 2 — access ports and VLAN membership.** `show vlan brief` on ACCESS-SW-01 confirmed VLAN 10 exists and Fa0/1–Fa0/3 are assigned to it. Access-layer configuration correct.

**Step 3 — characterise the failure.** Built a quick reachability picture:

| From | To | Same switch? | Result |
|---|---|---|---|
| DC-SERVER-02 | DC-SERVER-01 | Yes | Success |
| DC-SERVER-01 | 192.168.10.1 (SVI on core) | No | Fail |
| NOC-PC-01 | DC-SERVER-01 | No | Fail |

Everything crossing a switch boundary failed. Everything local succeeded. This immediately narrows the fault to the path between switches.

**Step 4 — trunk.** `show interfaces trunk` on ACCESS-SW-01 showed VLAN 10 absent from both "Vlans allowed on trunk" and "Vlans in spanning tree forwarding state and not pruned". Fault located.

**Step 5 — confirm from the other side.** `show interfaces trunk` on CORE-SW-01 showed VLAN 10 still allowed on Gi1/0/1. The mismatch confirmed the fault was on the ACCESS-SW-01 side of the link.

**Step 6 — corroborate with the MAC table.** `show mac address-table vlan 10` on CORE-SW-01 showed no MAC addresses learned via Gi1/0/1, despite three active servers in that VLAN. No frames were crossing the trunk.

## Commands and checks used

```
show interfaces trunk                    (ACCESS-SW-01 and CORE-SW-01)
show vlan brief                          (ACCESS-SW-01)
show interfaces status                   (ACCESS-SW-01)
show mac address-table vlan 10           (CORE-SW-01)
show running-config interface GigabitEthernet0/1
ping 192.168.10.11    (from DC-SERVER-02 - local, succeeds)
ping 192.168.10.1     (from DC-SERVER-01 - crosses trunk, fails)
```

## Evidence

**Before — `show interfaces trunk` on ACCESS-SW-01 (VLAN 10 absent):**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show vlan brief` on ACCESS-SW-01 (VLAN 10 present, ports assigned):**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — cross-switch ping failing:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — local intra-switch ping still succeeding:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show mac address-table vlan 10` on CORE-SW-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Root cause

The allowed VLAN list on `ACCESS-SW-01 Gi0/1` was set to `20,30,40,99`, omitting VLAN 10. An 802.1Q trunk forwards only VLANs present in its allowed list; frames tagged with any other VLAN are discarded at the trunk without logging an error.

Because VLAN 10 remained correctly configured on the access ports and in the VLAN database, every local check appeared healthy. The fault existed only on the inter-switch path.

**Contributing factor:** `switchport trunk allowed vlan <list>` replaces the entire list rather than appending to it. Using the command without the `add` keyword during a change is a well-known cause of exactly this outage.

## Resolution

```
enable
configure terminal
interface GigabitEthernet0/1
 switchport trunk allowed vlan 10,20,30,40,99
 exit
end
copy running-config startup-config
```

## Validation

| Check | Command | Expected | Actual |
|---|---|---|---|
| VLAN 10 allowed on trunk | `show interfaces trunk` (ACCESS-SW-01) | 10,20,30,40,99 listed | |
| VLAN 10 forwarding, not pruned | `show interfaces trunk` | VLAN 10 in forwarding list | |
| Trunk matches on both ends | `show interfaces trunk` (CORE-SW-01) | Identical allowed list | |
| Native VLAN consistent | `show interfaces trunk` | 99 on both ends | |
| Server reaches gateway | `ping 192.168.10.1` from DC-SERVER-01 | Success | |
| Cross-switch reachability | `ping 192.168.10.11` from NOC-PC-01 | Success | |
| MAC learning across trunk | `show mac address-table vlan 10` (CORE-SW-01) | VLAN 10 MACs on Gi1/0/1 | |

**After — `show interfaces trunk` on ACCESS-SW-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — cross-switch ping succeeding:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Preventive action

1. **Always use the `add` keyword.** `switchport trunk allowed vlan add <id>` appends. The bare form replaces the entire list and is the direct cause of this class of outage.
2. **Verify both ends after any trunk change.** `show interfaces trunk` on both switches, comparing allowed lists and native VLANs.
3. **Include a cross-switch reachability test in every trunk change window.** A local ping proves nothing about a trunk.
4. **Document the intended allowed VLAN list** in `network-design/vlan-plan.md` so the correct value can be restored without guesswork.

## Lessons learned

"Works locally, fails remotely" is the fastest fault classification available. Before running a single command, comparing an intra-switch test against an inter-switch test narrows the problem to the trunk or the routing path.

The other lesson is that a trunk fault is silent. There is no error message, no down interface, no log entry — just discarded frames. Only `show interfaces trunk` reveals it, which is why it belongs in the standard verification sequence rather than being reached for only when someone suspects a trunk.
