# NET-001 — Server on Incorrect VLAN

| Field | Value |
|---|---|
| **Incident ID** | NET-001 |
| **Priority** | P2 — single simulated server isolated, no wider impact |
| **Category** | Layer 2 / VLAN misconfiguration |
| **Affected Device** | DC-SERVER-03 (SRV-A01-003) |
| **Affected Switch** | ACCESS-SW-01 (SW-A01-001, RACK-A01 U42) |
| **Affected Port** | Fa0/3 |
| **Correct VLAN** | 10 (SERVERS) |
| **Misconfigured VLAN** | 30 (OPERATIONS) |
| **Detected by** | Post-deployment connectivity validation |
| **Status** | Prepared — execution/evidence pending |

---

## Issue

DC-SERVER-03 was reported as unreachable following a port change. The server's access port on ACCESS-SW-01 had been assigned to VLAN 30 instead of VLAN 10, placing the server in a different broadcast domain from its configured IP subnet.

## Initial symptoms

- DC-SERVER-03 could not reach its default gateway at 192.168.10.1
- DC-SERVER-03 could not reach DC-SERVER-01 or DC-SERVER-02 in the same subnet
- The switch port link light was **on** and `show interfaces status` showed the port `connected`
- No cabling had been changed and the server's own IP configuration was unmodified

**The misleading part:** every physical indicator looked healthy. Link up, correct cable, correct port, correct IP on the host. This is the signature of a Layer 2 logical fault rather than a physical one.

## Fault injection (for reproduction)

Run on ACCESS-SW-01:

```
enable
configure terminal
interface FastEthernet0/3
 switchport access vlan 30
 exit
end
```

## Investigation

Followed the standard troubleshooting order from `runbooks/network-troubleshooting-runbook.md`.

**Step 1 — physical/link state.** Link LED on, port shows `connected`. Layer 1 ruled out.

**Step 2 — identify the port.** `show mac address-table` confirmed DC-SERVER-03's MAC was learned on Fa0/3, matching `switch-port-map.csv`. Correct port confirmed — the server is where the documentation says it is.

**Step 3 — VLAN.** `show interfaces status` showed Fa0/3 in VLAN **30**. The port map specifies VLAN **10**. Fault located.

**Step 4 — confirm the mismatch.** `show vlan brief` showed Fa0/3 listed under OPERATIONS rather than SERVERS.

**Step 5 — rule out addressing.** The server's IP (192.168.10.13/24, gateway 192.168.10.1) was correct and unchanged. The address is right; the VLAN it sits in is wrong. A host in VLAN 30 with a 192.168.10.x address has no path to 192.168.10.1, because the VLAN 10 SVI is not reachable from VLAN 30 at Layer 2 and the host will never ARP successfully for its gateway.

## Commands and checks used

```
show interfaces status
show vlan brief
show mac address-table
show running-config interface FastEthernet0/3
ping 192.168.10.1          (from DC-SERVER-03)
ping 192.168.10.11         (from DC-SERVER-03)
```

## Evidence

> **Paste actual Packet Tracer output below. Do not summarise or retype from memory.**

**Before — `show interfaces status` on ACCESS-SW-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show vlan brief` on ACCESS-SW-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show mac address-table` on ACCESS-SW-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — ping from DC-SERVER-03 to 192.168.10.1:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Root cause

Switch port `ACCESS-SW-01 Fa0/3` was configured with `switchport access vlan 30` instead of `switchport access vlan 10`. The server was therefore placed in the OPERATIONS broadcast domain while retaining a SERVERS subnet IP address, leaving it unable to reach its gateway or any host in its own subnet.

The physical layer, cabling, port assignment, and host IP configuration were all correct. The single fault was the VLAN membership of the access port.

## Resolution

```
enable
configure terminal
interface FastEthernet0/3
 switchport access vlan 10
 exit
end
copy running-config startup-config
```

## Validation

| Check | Command | Expected | Actual |
|---|---|---|---|
| Port VLAN corrected | `show interfaces status` | Fa0/3 in VLAN 10 | |
| VLAN database | `show vlan brief` | Fa0/3 under SERVERS | |
| MAC learned in correct VLAN | `show mac address-table` | DC-SERVER-03 MAC in VLAN 10 on Fa0/3 | |
| Gateway reachable | `ping 192.168.10.1` | Success | |
| Same-VLAN peer reachable | `ping 192.168.10.11` | Success | |
| Inter-VLAN reachable | `ping 192.168.30.1` | Success | |
| Name resolution | `ping dc-server-01.dclab.local` | Success | |

**After — `show interfaces status`:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — ping from DC-SERVER-03 to 192.168.10.1:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Preventive action

1. **Verify VLAN as part of deployment.** `deployment/new-server-deployment-checklist.md` Section 4 requires `show vlan brief` and `show interfaces status` confirmation before a deployment is closed. This fault would have been caught there.
2. **Maintain port descriptions.** The description on Fa0/3 records the intended VLAN. Comparing `show interfaces description` against `show interfaces status` surfaces the mismatch immediately.
3. **Keep the port map authoritative.** `cabling/switch-port-map.csv` is the reference for intended VLAN membership; any change must update it in the same change window.
4. **Treat "link up but no connectivity" as a Layer 2 configuration fault** until proven otherwise. It is the single most common entry-level network incident.

## Lessons learned

A green link light proves the physical layer is working and nothing more. The most common data center network fault is not a bad cable — it is a correct cable in a correctly working port that has been placed in the wrong logical network. `show interfaces status` answers both questions in one line: is the port up, and which VLAN is it in.
