# Reachability Baseline

Two connectivity matrices: one captured **before** the VLAN 40 ACL was applied, one **after**. Together they prove the ACL blocked exactly the intended traffic and nothing more.

**Fill these in from actual Packet Tracer ping results.** Use `PASS` or `FAIL`. Do not estimate — an unverified matrix is worse than no matrix, because it will be trusted.

---

## Matrix A — pre-ACL baseline (end of Phase 6)

Captured after inter-VLAN routing, DHCP and DNS were working, and before any ACL existed.

| From ↓ / To → | 192.168.10.1 | 192.168.10.11 | 192.168.20.1 | 192.168.20.2 | 192.168.20.3 | 192.168.30.1 | 192.168.40.1 |
|---|---|---|---|---|---|---|---|
| DC-SERVER-01 (VLAN 10) | | | | | | | |
| DC-SERVER-02 (VLAN 10) | | | | | | | |
| DC-SERVER-03 (VLAN 10) | | | | | | | |
| NOC-PC-01 (VLAN 30, DHCP) | | | | | | | |
| OPS-PC-01 (VLAN 30, DHCP) | | | | | | | |
| TEST-PC-01 (VLAN 40) | | | | | | | |

**Expected:** every cell PASS. No filtering exists at this point.

---

## Matrix B — post-ACL (end of Phase 7)

Captured after `UNTRUSTED-TO-MGMT` was applied inbound on `interface Vlan40`.

| From ↓ / To → | 192.168.10.1 | 192.168.10.11 | 192.168.20.1 | 192.168.20.2 | 192.168.20.3 | 192.168.30.1 | 192.168.40.1 |
|---|---|---|---|---|---|---|---|
| DC-SERVER-01 (VLAN 10) | | | | | | | |
| DC-SERVER-02 (VLAN 10) | | | | | | | |
| DC-SERVER-03 (VLAN 10) | | | | | | | |
| NOC-PC-01 (VLAN 30, DHCP) | | | | | | | |
| OPS-PC-01 (VLAN 30, DHCP) | | | | | | | |
| TEST-PC-01 (VLAN 40) | | | | | | | |

**Expected:** identical to Matrix A except the TEST-PC-01 row, where the three VLAN 20 columns (`.1`, `.2`, `.3`) change from PASS to FAIL. Every other cell must be unchanged.

---

## Name resolution baseline

| From | Command | Expected | Actual |
|---|---|---|---|
| NOC-PC-01 | `ping dc-server-01.dclab.local` | PASS | |
| NOC-PC-01 | `ping dc-server-02.dclab.local` | PASS | |
| NOC-PC-01 | `ping dc-server-03.dclab.local` | PASS | |
| NOC-PC-01 | `ping core-sw-01.dclab.local` | PASS | |
| TEST-PC-01 | `ping dc-server-02.dclab.local` | PASS | |

---

## How to read a changed cell

| Change | Likely cause |
|---|---|
| A cell changed that should not have | ACL is over-scoped, or applied to the wrong interface/direction |
| An entire row went FAIL | Host-level problem: wrong VLAN, wrong IP, wrong gateway, or a down port |
| An entire column went FAIL | Destination-side problem: SVI down, or that device is offline |
| Only cross-switch cells FAIL | Trunk problem — a VLAN is missing from an allowed list |

This table is the fastest way to characterise a fault before touching a single `show` command, and it is the reasoning used throughout the troubleshooting runbook.
