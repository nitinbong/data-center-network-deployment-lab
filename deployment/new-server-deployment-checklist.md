# New Server Network Deployment Checklist

**Purpose:** standard procedure for bringing a new server onto the network, from rack arrival to closed ticket.

**Applies to:** Data Center Technician, Deployment Technician, Server Technician.

**Rule:** every box is verified, not assumed. "It should be VLAN 10" is not verification. `show interfaces status` showing VLAN 10 is verification.

---

## Deployment record header

| Field | Value |
|---|---|
| Ticket / change number | |
| Server hostname | |
| Asset ID | |
| Technician | |
| Date | |
| Change window | |

---

## Section 1 — Physical placement

- [ ] **Verify rack ID** matches the ticket (e.g. `RACK-A01`). Confirm against the physical rack label, not just the floor plan.
- [ ] **Verify rack-unit position** matches the assigned RU (e.g. `U40`). Confirm the intended U is actually free before mounting.
- [ ] **Verify asset / server ID** on the chassis asset tag matches the ticket (e.g. `SRV-A01-004`).
- [ ] **Verify rails are correctly seated** and the chassis is fully secured before releasing weight.
- [ ] **Verify airflow** — front-to-back orientation correct, blanking panels in place in adjacent empty units.
- [ ] **Verify power** — both PSUs connected, ideally to A and B feeds.

**Do not proceed until the server is physically where the documentation says it is.** Everything downstream inherits this.

---

## Section 2 — Network physical connection

- [ ] **Verify NIC** — identify which physical NIC is being used (NIC1 data, NIC2 management). Record the NIC's MAC address; it is needed in Section 6.
- [ ] **Verify assigned switch** matches the port map (e.g. `ACCESS-SW-01`).
- [ ] **Verify assigned switch port** matches the port map (e.g. `Fa0/4`). Confirm the port is genuinely unused before patching into it.
- [ ] **Verify cable type** is correct for the run (Cat6 / Cat6A copper, or fiber with matching transceivers at both ends).
- [ ] **Apply cable label at both ends**, matching the documented convention:
      `<SRC-RACK>-<SRC-DEVICE>-<PORT>__<DST-RACK>-<DST-DEVICE>-<PORT>`
- [ ] **Verify cable label** is legible and identical at both ends before dressing the cable.
- [ ] **Dress the cable** through the management arm / vertical manager. Respect bend radius.

---

## Section 3 — Link verification

- [ ] **Verify physical link LED** on both the server NIC and the switch port. Dark LED = Layer 1 problem, stop and resolve before continuing.
- [ ] **Verify switch-side link state:**
      ```
      show interfaces status
      ```
      Port must read `connected`. `notconnect` means no link. `disabled` or `err-disabled` means the port is administratively down or has been shut by a protection feature.
- [ ] **Verify speed and duplex** negotiated correctly in the same output. A 1 Gbps NIC showing 100 Mbps half-duplex indicates a cable or negotiation fault.
- [ ] **Verify no error counters incrementing:**
      ```
      show interface <port>
      ```
      Look at input errors, CRC and collisions. Rising CRC errors point at a bad cable, connector, or transceiver.

---

## Section 4 — Layer 2 configuration

- [ ] **Verify VLAN assignment** matches the design:
      ```
      show vlan brief
      show interfaces status
      ```
      The port must appear under the correct VLAN. This is the single most common deployment error and the basis of incident NET-001.
- [ ] **Verify port mode** is access, not trunk or dynamic:
      ```
      show running-config interface <port>
      ```
      Expect `switchport mode access` and `switchport access vlan <id>`.
- [ ] **Verify port description** is set and matches the cable label. An undescribed port is an undocumented port.

---

## Section 5 — Layer 3 configuration

- [ ] **Verify IP address** assigned on the server matches the IP address plan.
- [ ] **Verify subnet mask** is correct for the VLAN (`255.255.255.0` in this design). A wrong mask produces the classic "can reach some things but not others" symptom.
- [ ] **Verify default gateway** matches the VLAN's SVI (`.1` in each subnet).
- [ ] **Verify DNS server** is set (`192.168.10.11` in this design).
- [ ] **Verify no IP conflict** — confirm the address is not already in use and is not inside a DHCP dynamic range.

---

## Section 6 — Connectivity testing

- [ ] **Ping the default gateway** from the server:
      ```
      ping 192.168.10.1
      ```
      This is the first and most important test. Failure means the problem is local — VLAN, IP, mask, or Layer 1. Do not test anything further until this passes.
- [ ] **Ping a known host in the same VLAN.** Confirms Layer 2 forwarding within the VLAN.
- [ ] **Ping a host in a different VLAN.** Confirms routing and the gateway are working.
- [ ] **Test name resolution:**
      ```
      ping <hostname>.dclab.local
      ```
      If the IP ping works and the hostname ping fails, the fault is DNS, not the network.
- [ ] **Test the specific connectivity the server actually requires** — the application ports, the backup target, the monitoring server. General ping success does not prove the server can do its job.

---

## Section 7 — Switch-side verification

- [ ] **Verify the MAC address table** shows the server on the expected port and VLAN:
      ```
      show mac address-table
      ```
      Cross-check the learned MAC against the NIC MAC recorded in Section 2.

      **This confirms the right physical device is on the right physical port.** The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation. If the MAC appears on a different port than expected, the cable is not where the paperwork says it is.
- [ ] **Record the confirmed switch port** for documentation.

---

## Section 8 — Documentation

- [ ] **Update `cabling/switch-port-map.csv`** — device, interface, switch, port, mode, VLAN, IP, cable type, cable label, purpose.
- [ ] **Update `cabling/cable-map.csv`** — new cable ID, both endpoints, cable type, length.
- [ ] **Update `rack-design/asset-inventory.csv`** — asset ID, device name, model, rack, RU, VLAN, management IP, role, status.
- [ ] **Update `rack-design/rack-elevation.md`** — the RU is no longer empty.
- [ ] **Update `network-design/ip-address-plan.md`** — the address is no longer free.
- [ ] **Add DNS record** if the server needs name resolution.
- [ ] **Verify switch port description** on the device matches the updated documentation.

**The documentation and the switch must agree.** Three sources — the port map CSV, the port description on the switch, and the MAC address table — should all tell the same story. Where they disagree, something was changed without being recorded, which is exactly the fault reproduced in NET-007.

---

## Section 9 — Handover and closure

- [ ] **Confirm the server owner can reach the server** as expected.
- [ ] **Add the server to monitoring** if applicable.
- [ ] **Record all verification evidence** in the ticket — port, VLAN, IP, MAC, and the ping results.
- [ ] **Update the deployment ticket** with the completed checklist.
- [ ] **Close the ticket**, or hand back to the requester with the confirmed details.

---

## Rollback

If the deployment fails validation and cannot be resolved within the change window:

- [ ] Administratively shut the switch port to prevent a half-configured host generating traffic
- [ ] Remove or clearly mark the patch cord
- [ ] Revert any switch configuration changes made
- [ ] Restore documentation to its pre-change state
- [ ] Record what failed and at which step in the ticket

Leaving a partially deployed server live and undocumented is worse than backing it out cleanly.

---

## Quick reference — deployment verification commands

| Check | Command |
|---|---|
| Port up, speed, duplex, VLAN | `show interfaces status` |
| Port errors | `show interface <port>` |
| VLAN membership | `show vlan brief` |
| Port configuration | `show running-config interface <port>` |
| Device actually on this port | `show mac address-table` |
| Port description | `show interfaces description` |
| Gateway reachable | `ping <gateway>` from the server |
| Name resolution | `ping <hostname>` from the server |
