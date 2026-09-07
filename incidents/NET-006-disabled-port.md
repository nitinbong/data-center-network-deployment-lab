# NET-006 — Server Link Down Due to Administratively Disabled Switch Port

| Field | Value |
|---|---|
| **Incident ID** | NET-006 |
| **Priority** | P2 — single simulated server offline |
| **Category** | Layer 1 / switch port administrative state |
| **Affected Device** | DC-SERVER-02 (SRV-A01-002) |
| **Affected Switch** | ACCESS-SW-01 (SW-A01-001, RACK-A01 U42) |
| **Affected Port** | Fa0/2 |
| **Affected VLAN** | 10 (SERVERS) |
| **Detected by** | Monitoring alert — server unreachable |
| **Status** | Prepared — execution/evidence pending |

---

## Issue

The switch port serving DC-SERVER-02 was administratively shut down, dropping the server's network link entirely. The server itself remained powered and healthy but was completely isolated from the network.

## Initial symptoms

- DC-SERVER-02 unreachable from every host on the network
- DC-SERVER-02 could not reach its own default gateway
- **Link LED dark** on both the server NIC and the switch port
- DC-SERVER-01 and DC-SERVER-03, on the same switch and VLAN, entirely unaffected
- No cabling work had been performed

**The distinguishing symptom is the dark link light.** Every other incident in this project featured a green link with a logical fault above it. This one is different: the physical link is genuinely down, which points straight at Layer 1 and eliminates VLAN, IP, routing, DHCP, DNS and ACL from consideration entirely.

## Fault injection (for reproduction)

Run on ACCESS-SW-01:

```
enable
configure terminal
interface FastEthernet0/2
 shutdown
 exit
end
```

## Investigation

**Step 1 — physical/link state.** Link LED dark at both ends. This is the top of the troubleshooting runbook and it stops the investigation from wandering: nothing above Layer 1 can work while the link is down, so no VLAN, IP, or routing check is worth running yet.

**Step 2 — switch-side port state.** `show interfaces status` on ACCESS-SW-01 showed Fa0/2 in state `disabled`.

The distinction between the possible states is what makes this quick:

| Status | Meaning | Cause |
|---|---|---|
| `connected` | Link up, forwarding | Healthy |
| `notconnect` | No link detected | Cable unplugged/faulty, far-end device down, wrong port |
| `disabled` | Administratively shut down | Someone issued `shutdown` on the port |
| `err-disabled` | Disabled by a protection feature | Port security violation, BPDU guard, link flap |

`disabled` is decisive: this is not a cable problem, not a failed NIC, and not a protection feature. Somebody shut the port.

**Step 3 — confirm with the interface detail.** `show interface FastEthernet0/2` showed the line protocol state as administratively down.

**Step 4 — confirm in the configuration.** `show running-config interface FastEthernet0/2` showed the `shutdown` command present on the interface. Fault confirmed in configuration, not just in operational state.

**Step 5 — verify nothing else changed.** The port description and `switchport access vlan 10` were intact. The only difference from the known-good baseline was the `shutdown` line.

**Step 6 — check for wider impact.** `show interfaces status` confirmed Fa0/1 and Fa0/3 were still `connected`. Scope limited to a single port.

## Commands and checks used

```
show interfaces status                              (ACCESS-SW-01)
show interface FastEthernet0/2                      (ACCESS-SW-01)
show running-config interface FastEthernet0/2       (ACCESS-SW-01)
show mac address-table                              (ACCESS-SW-01)
ping 192.168.10.1        (from DC-SERVER-02 - fails)
ping 192.168.10.12       (from DC-SERVER-01 - fails)
```

`show mac address-table` was checked deliberately: DC-SERVER-02's MAC had aged out and no longer appeared on Fa0/2. A device that cannot transmit cannot be learned. That absence corroborates a link-level failure rather than a configuration issue higher up.

## Evidence

**Before — `show interfaces status` showing Fa0/2 disabled:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show interface FastEthernet0/2` showing administratively down:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show running-config interface FastEthernet0/2` showing the shutdown command:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — ping from DC-SERVER-02 to gateway failing:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — topology screenshot showing the red link on Fa0/2:**
```
[ REFERENCE SCREENSHOT: NET-006-before-fail.png ]
```

## Root cause

The `shutdown` command was present on `ACCESS-SW-01 FastEthernet0/2`, placing the port in an administratively down state. A shut port does not establish a physical link regardless of cabling or the far-end device's condition.

The cable, the NIC, the VLAN assignment, the server's IP configuration and all upstream network infrastructure were correct and unchanged. The single fault was the administrative state of one switch port.

## Resolution

```
enable
configure terminal
interface FastEthernet0/2
 no shutdown
 exit
end
copy running-config startup-config
```

The port transitions through spanning-tree listening and learning states before forwarding. On a real Cisco switch this takes roughly 30 seconds unless PortFast is configured; in Packet Tracer it is faster but the amber-to-green transition is still visible. **A port that has just come up is not immediately passing traffic** — waiting for green before declaring the fix successful avoids a false negative on the validation ping.

## Validation

| Check | Command | Expected | Actual |
|---|---|---|---|
| Port administratively up | `show interfaces status` | Fa0/2 `connected` | |
| Link protocol up | `show interface Fa0/2` | up / line protocol up | |
| Shutdown removed from config | `show running-config interface Fa0/2` | No `shutdown` line | |
| VLAN still correct | `show interfaces status` | VLAN 10 | |
| Description intact | `show interfaces description` | Original description present | |
| MAC relearned | `show mac address-table` | DC-SERVER-02 MAC on Fa0/2, VLAN 10 | |
| Gateway reachable | `ping 192.168.10.1` from DC-SERVER-02 | Success | |
| Same-VLAN peer reachable | `ping 192.168.10.11` from DC-SERVER-02 | Success | |
| Inter-VLAN reachable | `ping 192.168.30.1` from DC-SERVER-02 | Success | |
| Reachable from NOC | `ping 192.168.10.12` from NOC-PC-01 | Success | |
| Name resolution | `ping dc-server-02.dclab.local` from NOC-PC-01 | Success | |

**After — `show interfaces status` showing Fa0/2 connected:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — `show mac address-table` showing the MAC relearned:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — ping success from DC-SERVER-02:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Preventive action

1. **Check administrative state before dispatching anyone to the floor.** `show interfaces status` distinguishes `disabled` from `notconnect` in one line. A `disabled` port is fixed from the CLI in seconds; sending a technician to check a cable that was never the problem wastes time.
2. **Never leave a port shut without a description explaining why.** A port shut for a decommission with no note is indistinguishable from a port shut by mistake.
3. **Include port state in change verification.** If a change window involves shutting ports, verifying they are all back up belongs in the change closure steps.
4. **Investigate `err-disabled` differently.** An err-disabled port was shut *by the switch*, not by a person. `no shutdown` alone will not fix it — the triggering condition (port security violation, BPDU guard, link flap) must be resolved first, or the port will simply shut again.

## Lessons learned

This incident is the cleanest illustration of why the troubleshooting runbook starts at the physical layer. The dark link light answered the question before any other command was run.

The broader habit: check the cheapest, fastest, most decisive thing first. `show interfaces status` costs one command and rules out or confirms an entire category of faults. Working upward from Layer 1 means never wasting time on a routing table when the cable is not even linked.

The `notconnect` versus `disabled` distinction is small and genuinely useful. One means "the switch sees nothing on this port" — go look at the cable. The other means "the switch is refusing to bring this port up" — the fix is at the CLI and no physical intervention is needed at all.
