# NET-005 — Misapplied ACL Blocking Legitimate Management Access

| Field | Value |
|---|---|
| **Incident ID** | NET-005 |
| **Priority** | P1 — operations team locked out of all network management |
| **Category** | Access control list misconfiguration / change error |
| **Affected Device** | NOC-PC-01, OPS-PC-01 |
| **Affected Switch** | CORE-SW-01 (SW-A02-002) |
| **Affected Interface** | `interface Vlan30` (inbound) |
| **Affected VLAN** | 30 (OPERATIONS) — source. 20 (MANAGEMENT) — blocked destination |
| **Detected by** | NOC unable to reach switch management addresses |
| **Status** | Prepared — execution/evidence pending |

---

## Issue

A temporary access control list intended to restrict untrusted hosts was applied inbound on `interface Vlan30` instead of the intended interface. The ACL denied all traffic from the OPERATIONS VLAN to the MANAGEMENT VLAN, locking the NOC and operations workstations out of every switch management address while leaving all other connectivity untouched.

## Scope note — this is not the baseline ACL

The production baseline ACL in this design is `UNTRUSTED-TO-MGMT`, applied **inbound on `interface Vlan40`**. Because it is applied inbound on VLAN 40, it can only ever filter traffic sourced *from* VLAN 40. It is architecturally incapable of affecting VLAN 30 traffic.

This incident therefore models a realistic **change-management error**: a second, separate ACL created during a change window and applied to the wrong interface. The baseline ACL is not the fault and remains in place throughout.

## Initial symptoms

- NOC-PC-01 could not ping 192.168.20.1, 192.168.20.2, 192.168.20.3, or 192.168.20.11
- NOC-PC-01 could still ping 192.168.10.11, 192.168.10.12 and 192.168.10.13 normally
- Name resolution continued to work — DNS is in VLAN 10, unaffected
- `ping core-sw-01.dclab.local` (which resolves to 192.168.20.1) failed, while `ping dc-server-01.dclab.local` succeeded
- Routing table on CORE-SW-01 unchanged, all four SVIs up
- TEST-PC-01's expected VLAN 40 restrictions still behaved correctly

**The diagnostic signature:** routing is provably working — the same source host reaches one remote VLAN but not another. A routing or trunk fault would break both. Selective failure by destination subnet, with routing intact, means filtering.

## Fault injection (for reproduction)

Run on CORE-SW-01:

```
enable
configure terminal
!
ip access-list extended TEMP-RESTRICT
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip any any
 exit
!
interface Vlan30
 ip access-group TEMP-RESTRICT in
 exit
!
end
```

This represents a technician intending to apply an untrusted-host restriction, but selecting the wrong source VLAN and the wrong interface.

## Investigation

**Step 1 — physical and Layer 2.** All ports up, VLANs correct, trunks carrying all VLANs. Ruled out.

**Step 2 — client addressing.** `ipconfig /all` on NOC-PC-01 showed a valid DHCP lease with the correct gateway (192.168.30.1) and DNS. Client configuration correct.

**Step 3 — gateway reachable.** `ping 192.168.30.1` succeeded. The client can reach its own gateway, so the local path is healthy.

**Step 4 — characterise which destinations fail.**

| From NOC-PC-01 to | VLAN | Result |
|---|---|---|
| 192.168.30.1 | 30 (own gateway) | Success |
| 192.168.10.11 | 10 | Success |
| 192.168.10.12 | 10 | Success |
| 192.168.20.1 | 20 | **Fail** |
| 192.168.20.2 | 20 | **Fail** |
| 192.168.20.3 | 20 | **Fail** |

Routing works. One specific destination subnet is unreachable while another remote subnet is fine. This is not a routing fault — a broken route to VLAN 20 would also break traffic from VLAN 10, and it did not.

**Step 5 — verify VLAN 20 is healthy from elsewhere.** From DC-SERVER-01 (VLAN 10), `ping 192.168.20.2` succeeded. VLAN 20 and its devices are fine. The problem is specific to traffic *originating* in VLAN 30.

That combination — destination healthy, source-specific failure — points directly at an inbound filter on the source VLAN's interface.

**Step 6 — routing table.** `show ip route` on CORE-SW-01 showed all four connected routes present. Routing formally ruled out.

**Step 7 — access lists.** `show access-lists` revealed an unexpected ACL named `TEMP-RESTRICT` containing a deny from 192.168.30.0/24 to 192.168.20.0/24, with a non-zero match counter. Fault located.

**Step 8 — confirm application point.** `show ip interface Vlan30` showed `Inbound access list is TEMP-RESTRICT`. The ACL was applied to the operations VLAN, not the untrusted VLAN.

**Step 9 — confirm the baseline is intact.** `show ip interface Vlan40` still showed `UNTRUSTED-TO-MGMT` correctly applied. The production policy was never the problem.

## Commands and checks used

```
show access-lists                        (CORE-SW-01)
show ip interface Vlan30                 (CORE-SW-01)
show ip interface Vlan40                 (CORE-SW-01)
show ip route                            (CORE-SW-01)
show running-config | section access-list
ipconfig /all                            (NOC-PC-01)
ping 192.168.30.1     (own gateway - success)
ping 192.168.10.11    (VLAN 10 - success)
ping 192.168.20.2     (VLAN 20 - fail)
ping 192.168.20.2     (from DC-SERVER-01 in VLAN 10 - success)
```

## Evidence

**Before — management ping failing from NOC-PC-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — VLAN 10 ping still succeeding from the same host:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show access-lists` on CORE-SW-01 showing TEMP-RESTRICT with match counters:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show ip interface Vlan30` showing the inbound ACL:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show ip route` confirming routing is intact:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Root cause

An access list named `TEMP-RESTRICT`, containing `deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255`, was applied inbound on `interface Vlan30` on CORE-SW-01.

Because the ACL was applied inbound on the OPERATIONS VLAN interface, it was evaluated against every packet entering the router from VLAN 30. Traffic destined for VLAN 20 matched the deny statement and was dropped at the router. Traffic to any other destination matched the subsequent `permit ip any any` and passed normally, which is why VLAN 10 access appeared unaffected.

The intended target of the restriction was the UNTRUSTED VLAN (40), not OPERATIONS (30). Both the source network in the ACL and the interface it was applied to were wrong.

Routing, VLANs, trunking, DHCP, DNS and the baseline `UNTRUSTED-TO-MGMT` ACL were all functioning correctly throughout.

## Resolution

Remove the ACL from the interface, then delete the ACL itself:

```
enable
configure terminal
!
interface Vlan30
 no ip access-group TEMP-RESTRICT in
 exit
!
no ip access-list extended TEMP-RESTRICT
!
end
copy running-config startup-config
```

**Removing the ACL from the interface is the fix.** Deleting the ACL definition afterwards is cleanup — an unused ACL left in the configuration is a hazard, because a future change may reapply it without anyone reading its contents.

### Restore and verify the baseline policy

The intended production policy must be confirmed still in place after the incident:

```
show access-lists
show ip interface Vlan40
```

Expected: `UNTRUSTED-TO-MGMT` present, applied inbound on `interface Vlan40`, with `TEMP-RESTRICT` gone entirely.

## Validation

| Check | Command | Expected | Actual |
|---|---|---|---|
| Faulty ACL removed from interface | `show ip interface Vlan30` | No inbound access list | |
| Faulty ACL deleted | `show access-lists` | `TEMP-RESTRICT` absent | |
| Baseline ACL present | `show access-lists` | `UNTRUSTED-TO-MGMT` present | |
| Baseline ACL correctly applied | `show ip interface Vlan40` | Inbound `UNTRUSTED-TO-MGMT` | |
| NOC reaches core SVI | `ping 192.168.20.1` from NOC-PC-01 | Success | |
| NOC reaches ACCESS-SW-01 | `ping 192.168.20.2` from NOC-PC-01 | Success | |
| NOC reaches ACCESS-SW-02 | `ping 192.168.20.3` from NOC-PC-01 | Success | |
| NOC reaches server mgmt NIC | `ping 192.168.20.11` from NOC-PC-01 | Success | |
| Name resolution restored | `ping core-sw-01.dclab.local` | Success | |
| VLAN 10 access unchanged | `ping 192.168.10.11` from NOC-PC-01 | Success | |
| **Baseline restriction still enforced** | `ping 192.168.20.2` from TEST-PC-01 | **Fail** (correct) | |
| **Baseline permit still working** | `ping 192.168.10.11` from TEST-PC-01 | Success | |

The last two rows are essential. Fixing this incident must not accidentally remove the legitimate VLAN 40 restriction. Confirming the intended policy still works is part of the resolution, not an optional extra.

**After — management access restored from NOC-PC-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — `show access-lists` showing only the baseline ACL:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — baseline VLAN 40 restriction confirmed still enforced:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Preventive action

1. **Verify the application point, not just the ACL contents.** `show ip interface <interface>` confirms which ACL is applied where and in which direction. A correct ACL on the wrong interface is an outage.
2. **Test both halves of every ACL change.** Confirm the intended traffic is blocked *and* that all other traffic still passes. This incident only blocked one thing — and that one thing was the wrong thing.
3. **Use the reachability matrix.** `network-design/reachability-baseline.md` contains the known-good matrix. Re-running it after an ACL change surfaces unintended blocks immediately.
4. **Never leave temporary ACLs in the configuration.** If an ACL is temporary, its removal belongs in the same change record as its creation.
5. **Name ACLs descriptively.** `UNTRUSTED-TO-MGMT` states its purpose; a name like `TEMP-RESTRICT` or `ACL-1` tells a future reader nothing and invites exactly this kind of mistake.

## Lessons learned

The reasoning chain that identified this fault used no ACL-specific knowledge until the final step:

1. The client reaches its own gateway, so the local path is fine
2. The client reaches VLAN 10 but not VLAN 20, so routing is working
3. A host in VLAN 10 reaches VLAN 20 fine, so the destination is healthy
4. Therefore the failure depends on the **source**, not the destination or the path
5. Source-dependent selective blocking with intact routing means a filter

Steps 1–4 are pure logic from ping results. Only step 5 requires knowing what an ACL is. That is the general pattern worth carrying: characterise the failure precisely before reaching for tools, because a well-characterised failure usually names its own cause.
