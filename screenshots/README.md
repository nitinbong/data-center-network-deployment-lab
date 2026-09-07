# Screenshot Evidence Manifest

All screenshots are captured from live Cisco Packet Tracer sessions. Nothing in this directory is generated, edited, or reconstructed.

**Target: roughly 25–30 screenshots total.** The aim is proof, not exhaustive coverage. One clear capture that shows a state is worth more than five that repeat it.

- 12–15 baseline screenshots
- 2 per incident — the failure and the successful fix
- A diagnostic screenshot only where it genuinely adds value

---

## Baseline evidence (12–15)

| # | Filename | Contents | Phase |
|---|---|---|---|
| 01 | `01-topology-complete.png` | Full canvas, nine devices, all links green | 1 |
| 02 | `02-interfaces-status-access-sw-01.png` | `show interfaces status` — port state, speed, VLAN in one view | 1 |
| 03 | `03-interfaces-description-access-sw-01.png` | `show interfaces description` — port documentation on the device | 1 |
| 04 | `04-vlan-brief-access-sw-02.png` | `show vlan brief` — VLAN database and port membership | 2 |
| 05 | `05-mac-address-table.png` | `show mac address-table` with VLAN column populated | 2 |
| 06 | `06-trunk-status-core-sw-01.png` | `show interfaces trunk` — both uplinks, native VLAN, allowed list | 3 |
| 07 | `07-trunk-endtoend-ping-success.png` | Cross-switch VLAN 10 ping proving both trunks carry the VLAN | 3 |
| 08 | `08-ip-interface-brief-core.png` | All four SVIs up/up | 4 |
| 09 | `09-ip-route-core.png` | Four connected routes | 4 |
| 10 | `10-intervlan-ping-success.png` | VLAN 30 host reaching a VLAN 10 server | 4 |
| 11 | `11-dhcp-bindings.png` | `show ip dhcp binding` — both leases | 5 |
| 12 | `12-ipconfig-all-dhcp-client.png` | `ipconfig /all` — IP, mask, gateway and DNS all delivered | 5 |
| 13 | `13-dns-resolution-success.png` | Ping by hostname succeeding | 6 |
| 14 | `14-acl-blocked-and-permitted.png` | TEST-PC-01 blocked from VLAN 20, still reaching VLAN 10 | 7 |
| 15 | `15-acl-ops-management-access-ok.png` | NOC-PC-01 management access unaffected by the ACL | 7 |

**Optional extras** if a capture is particularly clear or you want the evidence: pre-routing cross-VLAN isolation failure, `show ip dhcp pool`, the DNS service records panel, or `show access-lists` with match counters.

---

## Incident evidence (2 per incident, plus diagnostics only where valuable)

| Incident | Failure | Fix | Diagnostic (only if it adds value) |
|---|---|---|---|
| NET-001 | `NET-001-before-fail.png` | `NET-001-after-fix.png` | `NET-001-diag.png` — `show interfaces status` showing the wrong VLAN |
| NET-002 | `NET-002-before-fail.png` | `NET-002-after-fix.png` | `NET-002-diag.png` — `show interfaces trunk` with the VLAN missing |
| NET-003 | `NET-003-before-fail.png` | `NET-003-after-fix.png` | `NET-003-diag.png` — `ipconfig /all` showing the wrong gateway |
| NET-004 | `NET-004-before-fail.png` | `NET-004-after-fix.png` | Usually unnecessary — the failure capture showing ping-by-IP working alongside ping-by-name failing is the diagnosis |
| NET-005 | `NET-005-before-fail.png` | `NET-005-after-fix.png` | `NET-005-diag.png` — `show ip interface Vlan30` showing the misapplied ACL |
| NET-006 | `NET-006-before-fail.png` | `NET-006-after-fix.png` | Usually unnecessary — `show interfaces status` showing `disabled` can be in the failure capture |
| NET-007 | `NET-007-before-fail.png` | `NET-007-after-fix.png` | `NET-007-diag.png` — `show mac address-table` showing the device on the undocumented port |

**Combine where you can.** For NET-002, a single capture showing the failed cross-switch ping next to the successful same-switch ping is stronger evidence than two separate screenshots, and it counts as one.

---

## Final state (1–2)

| Filename | Contents |
|---|---|
| `99-final-baseline-restored.png` | Post-incident verification that the network is back to known-good |

---

## Capture standards

- Capture the **entire CLI window** including the command that produced the output. Output with no visible command proves nothing.
- Capture the **device name** in frame where possible.
- Use **PNG**. Do not crop so tightly that context is lost.
- Do not edit, annotate over, or reconstruct output. If a capture was missed, re-run the step.
