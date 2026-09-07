# NET-003 — DHCP Clients Failing to Obtain Valid Addressing

| Field | Value |
|---|---|
| **Incident ID** | NET-003 |
| **Priority** | P2 — all VLAN 30 workstations affected, servers unaffected |
| **Category** | DHCP service misconfiguration |
| **Affected Device** | NOC-PC-01, OPS-PC-01 |
| **Affected Switch** | ACCESS-SW-02 (SW-A02-001) |
| **Affected Port** | Fa0/10, Fa0/11 |
| **Affected VLAN** | 30 (OPERATIONS) |
| **DHCP server** | CORE-SW-01, pool `OPS-POOL` |
| **Detected by** | Workstation users unable to reach network after reboot |
| **Status** | Prepared — execution/evidence pending |

---

## Issue

The DHCP pool on CORE-SW-01 was misconfigured with an incorrect `default-router` value. Clients continued to receive an IP address and subnet mask, but the gateway handed out did not exist on the VLAN 30 network. Workstations could communicate within their own subnet but could reach nothing beyond it.

## Initial symptoms

- NOC-PC-01 and OPS-PC-01 obtained addresses in the correct range (192.168.30.101+)
- Both could ping each other successfully
- Neither could reach 192.168.10.11, 192.168.20.2, or any other subnet
- Neither could resolve hostnames — DNS lives at 192.168.10.11, off-subnet
- `ipconfig /all` showed a default gateway that did not match the VLAN 30 SVI

**Why this is deceptive:** the DHCP transaction *succeeded*. There was no APIPA address, no "DHCP request failed" message, no obvious service outage. The client received four values and three of them were correct. Only the gateway was wrong, and a wrong gateway produces the exact symptom of "works locally, fails everywhere else" — which is easily mistaken for a routing or trunk problem.

## Fault injection (for reproduction)

Run on CORE-SW-01:

```
enable
configure terminal
ip dhcp pool OPS-POOL
 default-router 192.168.30.254
 exit
end
```

Then on NOC-PC-01 and OPS-PC-01:

```
ipconfig /release
ipconfig /renew
```

**Alternative faults that produce different, equally valid scenarios** (pick one; the injection above is the documented one):

| Fault | Command | Resulting symptom |
|---|---|---|
| Wrong network statement | `network 192.168.31.0 255.255.255.0` | Client gets an address in the wrong subnet entirely |
| Exhausted pool via over-broad exclusion | `ip dhcp excluded-address 192.168.30.1 192.168.30.254` | Client gets APIPA 169.254.x.x — no addresses available |
| Missing DNS server | `no dns-server` | IP connectivity fine, name resolution fails |

## Investigation

**Step 1 — client-side facts first.** `ipconfig /all` on NOC-PC-01 showed a valid IP and mask in the expected range, a DNS server of 192.168.10.11, and a default gateway of 192.168.30.254.

The address being in the correct range immediately rules out several possibilities: the client reached the DHCP server, the server responded, the pool is not exhausted, and the VLAN is correct. DHCP is working — it is handing out wrong information.

**Step 2 — verify the gateway is the problem.** Ping 192.168.30.254 from NOC-PC-01: fail. Ping 192.168.30.1 (the actual SVI): success. The real gateway is reachable; the one the client was told to use is not.

**Step 3 — pool state.** `show ip dhcp pool` on CORE-SW-01 confirmed addresses were being leased, ruling out exhaustion.

**Step 4 — bindings.** `show ip dhcp binding` showed both client MACs with active leases in the correct range.

**Step 5 — pool configuration.** `show running-config | section dhcp` revealed `default-router 192.168.30.254` where the design specifies `192.168.30.1`. Fault located.

**Step 6 — rule out routing.** Confirmed the VLAN 30 SVI was up and the routing table intact, so nothing below DHCP was contributing.

## Commands and checks used

```
ipconfig /all                              (NOC-PC-01, OPS-PC-01)
ping 192.168.30.1                          (client - real gateway, succeeds)
ping 192.168.30.254                        (client - issued gateway, fails)
show ip dhcp pool                          (CORE-SW-01)
show ip dhcp binding                       (CORE-SW-01)
show running-config | section dhcp         (CORE-SW-01)
show ip interface brief                    (CORE-SW-01)
ipconfig /release
ipconfig /renew
```

## Evidence

**Before — `ipconfig /all` on NOC-PC-01 showing wrong gateway:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show ip dhcp pool` on CORE-SW-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show ip dhcp binding` on CORE-SW-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `show running-config | section dhcp`:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — off-subnet ping failing from NOC-PC-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Root cause

The `default-router` parameter in DHCP pool `OPS-POOL` was set to `192.168.30.254`, an address with no device assigned to it. The VLAN 30 gateway is the SVI at `192.168.30.1`.

Clients received a technically complete DHCP lease containing an unreachable gateway. Since a host sends all off-subnet traffic to its default gateway, and that gateway did not respond to ARP, every off-subnet packet was dropped at the client before it ever reached the network.

DHCP itself, the VLAN, the trunk, the routing table and the SVI were all functioning correctly. The single fault was one incorrect value in the pool.

## Resolution

```
enable
configure terminal
ip dhcp pool OPS-POOL
 default-router 192.168.30.1
 exit
end
copy running-config startup-config
```

Then force clients to acquire the corrected lease:

```
ipconfig /release
ipconfig /renew
```

**A configuration fix alone does not resolve the incident.** Existing clients hold the bad lease until it expires or is manually renewed. The renew step is part of the fix, not an optional extra.

## Validation

| Check | Command | Expected | Actual |
|---|---|---|---|
| Pool corrected | `show running-config \| section dhcp` | `default-router 192.168.30.1` | |
| Lease reissued | `show ip dhcp binding` | Fresh leases for both clients | |
| Client has correct gateway | `ipconfig /all` | Gateway 192.168.30.1 | |
| Client has correct DNS | `ipconfig /all` | DNS 192.168.10.11 | |
| Gateway reachable | `ping 192.168.30.1` | Success | |
| Off-subnet reachable | `ping 192.168.10.11` | Success | |
| Name resolution | `ping dc-server-01.dclab.local` | Success | |
| Management reachable | `ping 192.168.20.2` | Success | |

**After — `ipconfig /all` on NOC-PC-01:**
```
[ PASTE ACTUAL OUTPUT ]
```

**After — `show ip dhcp binding`:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Preventive action

1. **Validate all four DHCP values after any pool change** — IP, mask, gateway, DNS. Three correct values and one wrong one still produces an outage.
2. **Test from a client, not just from the switch.** `show ip dhcp pool` looked healthy throughout this incident. Only `ipconfig /all` on an actual client exposed the fault.
3. **Force a lease renewal after every pool change** so clients pick up the new values immediately rather than at an unpredictable lease expiry.
4. **Keep the authoritative pool values in `network-design/ip-address-plan.md`** so the correct gateway can be confirmed without guessing.

## Lessons learned

DHCP failure is not one symptom, it is several, and they point in different directions:

| Symptom | Meaning |
|---|---|
| Client has 169.254.x.x (APIPA) | No DHCP response reached the client — wrong VLAN, service down, pool exhausted, or missing `ip helper-address` across a router |
| Client has an address in the wrong subnet | Wrong `network` statement in the pool |
| Client has an address but no off-subnet access | Wrong or missing `default-router` — this incident |
| Client has connectivity but cannot resolve names | Wrong or missing `dns-server` |

Reading which of these applies takes seconds with `ipconfig /all` and immediately determines where to look. That client-side check should come before any switch-side investigation.
