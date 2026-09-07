# Manual Execution Checklist

Everything in this repository that requires you to operate Cisco Packet Tracer and capture real evidence.

**The rule this project follows:** no command output, screenshot, or test result in this repository has been invented. Every place that needs real evidence carries an explicit placeholder. This file lists them.

---

## A. What is already complete

No further work needed on these. They are finished documentation, not placeholders.

| File | Status |
|---|---|
| `README.md` | Complete |
| `PROJECT-STATUS.md` | Complete |
| `interview-prep.md` | Complete |
| `resume-bullets.md` | Complete |
| `build-guides/` — all five phase guides | Complete |
| `topology/topology-description.md` | Complete |
| `rack-design/rack-elevation.md` | Complete |
| `rack-design/asset-inventory.csv` | Complete |
| `cabling/switch-port-map.csv` | Complete |
| `cabling/cable-map.csv` | Complete |
| `cabling/fiber-copper-transceivers.md` | Complete |
| `network-design/vlan-plan.md` | Complete |
| `network-design/ip-address-plan.md` | Complete |
| `configurations/*.txt` — Section A reference configs | Complete |
| `deployment/new-server-deployment-checklist.md` | Complete |
| `runbooks/network-troubleshooting-runbook.md` | Complete |
| `incidents/NET-001` through `NET-007` — structure, analysis, commands, root cause, preventive actions | Complete |
| `screenshots/README.md` — manifest and naming standard | Complete |

---

## B. What you must perform in Packet Tracer

Work through the build guides in order. Each contains the exact commands.

### B1 — Build the network

| Step | Guide | Deliverable |
|---|---|---|
| Place devices, add second NIC to DC-SERVER-01, run cables, set hostnames and port descriptions, assign static IPs | `build-guides/phase-1-topology-and-cabling.md` | Working topology, all links green |
| Create VLANs, assign access ports | `build-guides/phase-2-3-vlans-and-trunks.md` (Phase 2) | VLANs 10/20/30/40/99 on all switches |
| Configure trunks, run the end-to-end trunk test | `build-guides/phase-2-3-vlans-and-trunks.md` (Phase 3) | Both trunks carrying all VLANs |
| Enable `ip routing`, create SVIs, add management IPs to access switches | `build-guides/phase-4-svis-and-routing.md` | Inter-VLAN routing working |
| Configure DHCP pool for VLAN 30 | `build-guides/phase-5-6-7-dhcp-dns-acl.md` (Phase 5) | Two live leases |
| Configure DNS service and four A records | `build-guides/phase-5-6-7-dhcp-dns-acl.md` (Phase 6) | Name resolution working |
| Configure and apply the baseline ACL | `build-guides/phase-5-6-7-dhcp-dns-acl.md` (Phase 7) | VLAN 40 blocked from VLAN 20 |

**Three things to watch for during the build:**

1. **If your Packet Tracer lacks the 3650**, use the 3560. Everything works identically, but interface naming changes from `GigabitEthernet1/0/1` to `GigabitEthernet0/1`. Substitute throughout and note it in the README.
2. **`switchport trunk encapsulation dot1q`** — Use `switchport trunk encapsulation dot1q` only if the selected Packet Tracer switch model supports/requires it. If IOS rejects the command, omit it and configure `switchport mode trunk` directly.
3. **Return every temporary test change.** TEST-PC-01 must end in VLAN 40 at 192.168.40.50. OPS-PC-01 must end on DHCP, not static.

### B2 — Save the known-good baseline

Before injecting any incident:

- [ ] `copy running-config startup-config` on all three switches
- [ ] File → Save As → `packet-tracer/data-center-network-baseline.pkt` (never modified again — this is your rollback)
- [ ] File → Save As → `packet-tracer/data-center-network.pkt` (working copy)

### B3 — Run the incidents

Each incident file contains a **Fault injection (for reproduction)** section with the exact commands. For each of the seven:

1. Capture the healthy "before" state if not already captured
2. Inject the fault
3. Capture the failure evidence
4. Run the diagnostic commands and capture the output
5. Apply the resolution
6. Capture the "after" evidence
7. Complete the validation table in the report

Order matters for one of them: **NET-005 must be reverted completely**, including deleting the `TEMP-RESTRICT` ACL, and you must confirm the baseline `UNTRUSTED-TO-MGMT` ACL survives intact.

### B4 — Restore to baseline

After all incidents:

- [ ] Verify TEST-PC-01 is in VLAN 40 at 192.168.40.50 with gateway 192.168.40.1
- [ ] Verify OPS-PC-01 and NOC-PC-01 are on DHCP with leases in the .101+ range
- [ ] Verify `show access-lists` shows only `UNTRUSTED-TO-MGMT`
- [ ] Verify `show ip interface Vlan40` shows the ACL applied inbound
- [ ] Verify `show ip interface Vlan30` shows **no** inbound ACL
- [ ] Verify all trunk allowed lists read `10,20,30,40,99`
- [ ] Verify DC-SERVER-03 is on Fa0/3 in VLAN 10 (undo NET-007 if you ran it)
- [ ] Verify all ports show `connected` — no leftover `shutdown`
- [ ] Verify the DHCP pool `default-router` reads `192.168.30.1`
- [ ] Verify the DNS service is On with all four records
- [ ] `copy running-config startup-config` on all three switches
- [ ] Save as `packet-tracer/data-center-network.pkt`

---

## C. Evidence you must return

### C1 — Packet Tracer files

- [ ] `packet-tracer/data-center-network.pkt`
- [ ] `packet-tracer/data-center-network-baseline.pkt`

### C2 — Topology image

- [ ] `topology/network-topology.png` — full canvas, nine devices, all links green

### C3 — Configuration captures

Run `show running-config` on each switch and paste the complete output into **Section B** of:

- [ ] `configurations/core-switch.txt`
- [ ] `configurations/access-switch-01.txt`
- [ ] `configurations/access-switch-02.txt`

If the captured output disagrees with the Section A reference, the capture is correct and the reference needs updating.

### C4 — Baseline screenshots (12–15)

Full list with filenames and contents in `screenshots/README.md`. The target is proof of each capability, not exhaustive coverage:

| Coverage | Approx. count |
|---|---|
| Topology, port status, port descriptions | 3 |
| VLANs and MAC table | 2 |
| Trunk status and end-to-end trunk proof | 2 |
| SVIs, routing table, inter-VLAN ping | 3 |
| DHCP bindings and client `ipconfig /all` | 2 |
| DNS resolution | 1 |
| ACL blocked and permitted, plus ops access unaffected | 2 |

### C5 — Incident screenshots (~14–18)

Two per incident — the failure and the successful fix — plus a diagnostic capture only where it genuinely adds value. For NET-004 and NET-006 the diagnosis is usually visible in the failure capture itself, so a third screenshot is unnecessary.

**Total target across C4 and C5: roughly 25–30 screenshots.**

### C6 — Command output for incident reports

Every `[ PASTE ACTUAL OUTPUT ]` marker in the seven incident files. To find them all:

```bash
grep -rn "PASTE ACTUAL OUTPUT" incidents/
```

### C7 — Reachability matrices

Fill in `network-design/reachability-baseline.md`:

- [ ] Matrix A — pre-ACL, every cell PASS or FAIL from a real ping
- [ ] Matrix B — post-ACL, same
- [ ] Name resolution baseline table

Only the TEST-PC-01 row's VLAN 20 columns should differ between the two matrices. Any other difference means the ACL is over-scoped.

### C8 — Validation tables in incident reports

Each incident report has a validation table with an empty **Actual** column. Fill it from real test results.

---

## D. Final quality check

Run before publishing. Several of these are automatable:

```bash
# Any remaining unfilled evidence placeholders
grep -rn "PASTE ACTUAL OUTPUT\|AWAITING ACTUAL CAPTURE\|RECORD ACTUAL\|PASTE THE UPDATED" .

# Stale temporary IP that should not appear anywhere
grep -rn "192.168.30.150" .

# Confirm the corrected temp IP is used
grep -rn "192.168.30.50" .

# Any leftover TODO markers
grep -rni "todo\|fixme\|xxx" --include="*.md" .
```

Manual checks:

- [ ] Every IP address in the docs matches `network-design/ip-address-plan.md`
- [ ] Every VLAN number matches `network-design/vlan-plan.md`
- [ ] Every switch port matches `cabling/switch-port-map.csv`
- [ ] Markdown links in `README.md` all resolve
- [ ] No claim of physical cabling, fiber work, or production experience anywhere
- [ ] TEST-PC-01 documented and actually in VLAN 40
- [ ] OPS-PC-01 documented and actually on DHCP
- [ ] No ACL left in an incident state
- [ ] No port left administratively shut
- [ ] `PROJECT-STATUS.md` evidence checklist updated once captures are in

---

## E. Order of operations

```
1. Build phases 1-7 following the build guides
2. Capture the 12–15 baseline screenshots as you go
3. Capture show running-config into configurations/*.txt
4. Fill Matrix A and Matrix B in reachability-baseline.md
5. Save both .pkt files
6. Run NET-001 through NET-007, capturing 2 screenshots each (failure and fix), plus a diagnostic capture only where useful
7. Paste command output into the incident reports
8. Fill the validation tables
9. Restore to baseline and verify section B4
10. Run the section D quality checks
11. Update PROJECT-STATUS.md evidence checklist
12. Push to GitHub
```

Capturing screenshots as you go is far less painful than rebuilding the network later because one capture was missed.
