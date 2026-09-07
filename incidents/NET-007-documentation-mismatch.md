# NET-007 — Server Relocated Without Documentation Update (Optional)

| Field | Value |
|---|---|
| **Incident ID** | NET-007 |
| **Priority** | P3 — no service impact, documentation integrity issue |
| **Category** | Documentation / asset management |
| **Affected Device** | DC-SERVER-03 (SRV-A01-003) |
| **Affected Switch** | ACCESS-SW-01 (SW-A01-001) |
| **Documented Port** | Fa0/3 |
| **Actual Port** | Fa0/4 |
| **Affected VLAN** | 10 (SERVERS) |
| **Detected by** | Routine audit of port map against MAC address table |
| **Status** | Prepared — execution/evidence pending |

> **Optional incident.** Included because it exercises the single most important verification skill in data center technician work — proving where a device physically is, rather than trusting a spreadsheet.

---

## Issue

DC-SERVER-03 was moved from `Fa0/3` to `Fa0/4` on ACCESS-SW-01 during unrelated work. The port map, cable map and port descriptions were not updated. The server functioned normally, so nothing alerted — but every documentation source now pointed at the wrong port.

## Initial symptoms

**There were none.** That is the entire point of this incident.

- DC-SERVER-03 fully reachable, all connectivity normal
- No monitoring alert, no user complaint, no failed service
- `cabling/switch-port-map.csv` stated Fa0/3
- The port description on Fa0/3 still read `DC-SERVER-03 NIC1 DATA`
- Fa0/3 showed `notconnect`; Fa0/4 showed `connected` with no description

**Why this matters despite zero impact.** Documentation drift is invisible until the moment it is needed, and that moment is always an outage. A technician told to shut Fa0/3 to isolate DC-SERVER-03 would shut a dead port and wonder why nothing changed. Worse, if another server were later patched into Fa0/3, shutting it would take down the wrong host during an incident.

## Fault injection (for reproduction)

1. In Packet Tracer, delete the cable between DC-SERVER-03 and `ACCESS-SW-01 Fa0/3`
2. Connect DC-SERVER-03 to `ACCESS-SW-01 Fa0/4` with a Copper Straight-Through cable
3. Configure the new port:

```
enable
configure terminal
interface FastEthernet0/4
 switchport mode access
 switchport access vlan 10
 exit
end
```

Deliberately leave the port description off Fa0/4 and leave the stale description on Fa0/3. Do not update any documentation.

## Investigation

**Step 1 — audit trigger.** Comparing `cabling/switch-port-map.csv` against live switch state during a routine check.

**Step 2 — port state versus documentation.** `show interfaces status` on ACCESS-SW-01:

| Port | Documented device | Actual state |
|---|---|---|
| Fa0/1 | DC-SERVER-01 | connected, VLAN 10 |
| Fa0/2 | DC-SERVER-02 | connected, VLAN 10 |
| Fa0/3 | DC-SERVER-03 | **notconnect** |
| Fa0/4 | (unassigned) | **connected, VLAN 10** |

A documented port with nothing on it, and an undocumented port with something on it. The discrepancy is visible in a single command.

**Step 3 — identify what is actually on Fa0/4.** `show mac address-table` returned the MAC learned on Fa0/4.

**Step 4 — match the MAC to a device.** Compared the learned MAC against DC-SERVER-03's NIC MAC from its Config tab. Match confirmed.

**This is the key verification step.** The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation. Cross-referencing a learned MAC against a device's NIC address is what confirms attachment rather than inferring it.

**Step 5 — check the descriptions.** `show interfaces description` showed the stale description still on Fa0/3 and none on Fa0/4. The switch's own documentation was wrong in both directions.

**Step 6 — verify the server is otherwise correctly configured.** VLAN 10, correct IP, gateway reachable, name resolution working. No service fault — purely a records problem.

## Commands and checks used

```
show interfaces status                   (ACCESS-SW-01)
show mac address-table                   (ACCESS-SW-01)
show interfaces description              (ACCESS-SW-01)
show running-config interface FastEthernet0/3
show running-config interface FastEthernet0/4
```

Cross-referenced against DC-SERVER-03 → Config tab (NIC MAC address) and `cabling/switch-port-map.csv`.

## Evidence

**Before — `show interfaces status` showing Fa0/3 notconnect and Fa0/4 connected:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show mac address-table` showing the server MAC on Fa0/4:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show interfaces description` showing the stale description on Fa0/3:**
```
[ PASTE ACTUAL OUTPUT ]
```

**DC-SERVER-03 NIC MAC address (from the device Config tab):**
```
[ RECORD ACTUAL MAC ]
```

## Root cause

DC-SERVER-03's network cable was relocated from `Fa0/3` to `Fa0/4` without any corresponding update to:

- `cabling/switch-port-map.csv`
- `cabling/cable-map.csv`
- The port description on `Fa0/4`
- The stale port description on `Fa0/3`
- The physical cable label

The move itself was executed correctly at a technical level — the new port was configured with the right VLAN and the server worked. The failure was procedural: a change was made without following the documentation update steps in the deployment checklist.

## Resolution

The correct resolution depends on what the port map is *supposed* to say. Two valid paths:

### Option A — accept the new location as correct

Update every record to match reality.

```
enable
configure terminal
!
interface FastEthernet0/3
 no description
 shutdown
 exit
!
interface FastEthernet0/4
 description DC-SERVER-03 NIC1 DATA | A01-SRV003-NIC1__A01-SW001-Fa0-4
 switchport mode access
 switchport access vlan 10
 exit
!
end
copy running-config startup-config
```

Shutting the now-vacant Fa0/3 prevents an undocumented device from being patched into a port that still looks assigned.

Then update:
- `cabling/switch-port-map.csv` — change DC-SERVER-03's port from Fa0/3 to Fa0/4, update the cable label
- `cabling/cable-map.csv` — update CBL-003's destination port and label
- Physical cable label at both ends

### Option B — restore the documented state

Move the cable back to Fa0/3, remove the Fa0/4 configuration, and leave documentation unchanged.

**Option A was taken here**, since the server was operating normally in its new position and moving a live server's cable to satisfy a spreadsheet introduces risk for no benefit. This project's repository reflects Option A only in this incident report — the baseline `switch-port-map.csv` documents Fa0/3, and the network is returned to that state at the end of incident testing.

## Validation

| Check | Command | Expected | Actual |
|---|---|---|---|
| Description on the correct port | `show interfaces description` | Fa0/4 described, Fa0/3 clear | |
| Port state | `show interfaces status` | Fa0/4 connected VLAN 10 | |
| MAC on the documented port | `show mac address-table` | Server MAC on Fa0/4 | |
| Vacant port secured | `show interfaces status` | Fa0/3 disabled | |
| Connectivity unchanged | `ping 192.168.10.1` from DC-SERVER-03 | Success | |
| Documentation matches switch | Compare CSV to `show interfaces status` | Full agreement | |

**After — `show interfaces description` with corrected descriptions:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — updated `switch-port-map.csv` row:**
```
[ PASTE THE UPDATED CSV LINE ]
```

## Preventive action

1. **Documentation update is part of the change, not a follow-up task.** `deployment/new-server-deployment-checklist.md` Section 8 lists every file that must be updated. A change is not complete until those are done.
2. **Schedule periodic port audits.** Comparing `show interfaces status` and `show mac address-table` against the port map catches drift before it causes an incident. This is a quick, high-value routine task.
3. **Maintain port descriptions on the switch.** They are documentation that lives on the device itself and survives a lost spreadsheet.
4. **Shut and clear unused ports.** A vacant port with a stale description assigned to a server that is not there is an accident waiting to happen.
5. **Label the cable at both ends when it is moved.** The label is the first line of defence and the cheapest to maintain.

## Lessons learned

Three independent sources should agree about where a device is: the port map CSV, the port description on the switch, and the MAC address table. The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation.

The broader point is about trust. Documentation is only useful if it is reliably accurate; documentation that is *sometimes* wrong is arguably worse than none, because it produces confident action based on false information. The habit that prevents this is small — update the record in the same change window as the work — and it is exactly what separates a technician whose documentation the team relies on from one whose documentation nobody checks.
