# Network Troubleshooting Runbook

**Audience:** Data Center Technician, NOC Technician, Deployment Technician
**Scope:** switch, VLAN, port, addressing, routing, DHCP, DNS and ACL faults on the network described in this repository

---

## The method

Work **bottom-up**, from the physical layer toward the application layer, and stop at the first check that fails. Each step is cheaper and faster than the one after it, and each one eliminates an entire category of possible causes.

The reason this order matters: it is entirely possible to spend twenty minutes reading a routing table for a server whose cable was never plugged in. Checking the link light first costs five seconds and would have ended the investigation immediately.

```
 1. Physical / link state
 2. Identify the correct switch and port
 3. Verify administrative state
 4. Verify VLAN
 5. Verify IP address and subnet mask
 6. Verify default gateway
 7. Verify trunk
 8. Verify routing
 9. Verify DHCP
10. Verify DNS
11. Verify ACL
12. Retest
13. Document the resolution
```

---

## Before you start — characterise the failure

Two minutes of ping tests will usually name the fault before a single `show` command is run.

| Test | Result | What it eliminates |
|---|---|---|
| Ping own gateway | Success | Layer 1, VLAN, local IP config are all fine |
| Ping own gateway | Fail | Problem is local — port, VLAN, IP, mask, or cable |
| Ping host in same VLAN, same switch | Success | Switch and VLAN are forwarding correctly |
| Ping host in same VLAN, different switch | Fail (while local succeeds) | **Trunk problem** |
| Ping host in a different VLAN | Fail (while gateway succeeds) | Routing or ACL |
| Ping one remote VLAN OK, another fails | — | **ACL**, not routing |
| Ping by IP works, by name fails | — | **DNS**, not network |
| Ping by IP and by name both fail | — | Network. DNS is irrelevant until connectivity returns |

**Also ask:** how many devices are affected? One device points at that device's port or configuration. Every device on one switch points at the switch or its uplink. Every device in one VLAN points at the VLAN, its SVI, or the trunk. Every device everywhere points at the core.

---

## Step 1 — Physical / link state

**What to check**

- Link LED on the device NIC and the switch port
- `show interfaces status` on the switch

**Why**

Nothing above Layer 1 can work while the link is down. This is the fastest, cheapest, most decisive check available.

**Good result:** Link LED lit (solid or blinking); port shows `connected`.
**Bad result:** LED dark; port shows `notconnect`, `disabled`, or `err-disabled`.

**Interpreting the port status field**

| Status | Meaning | Next action |
|---|---|---|
| `connected` | Link up and forwarding | Go to Step 2 |
| `notconnect` | Switch sees no link | Check cable seating, cable integrity, far-end device power, correct port |
| `disabled` | Administratively shut down | Go to Step 3. Fixed from the CLI — no floor visit needed |
| `err-disabled` | Shut by a protection feature | Find the trigger (port security, BPDU guard, link flap) before `no shutdown`, or it will shut again |

**Also check** speed and duplex in the same output. A gigabit NIC showing 100 Mbps half-duplex suggests a cable, connector, or negotiation fault. Check `show interface <port>` for rising CRC and input errors, which point at bad cabling or a failing transceiver.

**Next:** link up → Step 2. Link down → resolve the physical or administrative cause and retest.

---

## Step 2 — Identify the correct switch and port

**What to check**

```
show interfaces description
show mac address-table
```

Compare against `cabling/switch-port-map.csv`.

**Why**

Every subsequent step assumes you are looking at the right port. If the documentation is wrong, you will correctly configure a port that has nothing on it and conclude the network is broken.

**Good result:** the device's NIC MAC appears in the MAC address table on the port the documentation specifies.
**Bad result:** the MAC appears on a different port, or does not appear at all.

The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation. Documentation records an intention; the live table shows current Layer 2 attachment. When they disagree, investigate and correct the documentation — see incident NET-007.

**If the MAC does not appear at all:** the device has not transmitted, which usually means the link is down (return to Step 1) or the device is powered off. MAC entries also age out after a period of silence, so generate traffic from the device and re-check.

**Next:** correct port confirmed → Step 3.

---

## Step 3 — Verify administrative state

**What to check**

```
show interfaces status
show interface <port>
show running-config interface <port>
```

**Why**

A port can be perfectly cabled and still refuse to link because someone shut it, or because a protection feature shut it.

**Good result:** no `shutdown` in the running config; interface and line protocol both up.
**Bad result:** `shutdown` present; status `disabled` or `err-disabled`.

**Fix for an administratively shut port:**

```
configure terminal
interface <port>
 no shutdown
 exit
end
```

Wait for the port to reach forwarding state before testing. A port that has just come up is not immediately passing traffic — testing too early produces a false negative.

**Fix for err-disabled:** identify and resolve the trigger first. `no shutdown` alone will not hold. See incident NET-006.

**Next:** port up and forwarding → Step 4.

---

## Step 4 — Verify VLAN

**What to check**

```
show vlan brief
show interfaces status
show running-config interface <port>
```

Compare against `network-design/vlan-plan.md`.

**Why**

A port in the wrong VLAN places the device in a different broadcast domain from its own subnet. Everything looks physically healthy and nothing works. This is the single most common entry-level network fault.

**Good result:** the port appears under the correct VLAN in `show vlan brief`, and the VLAN column in `show interfaces status` matches the port map.
**Bad result:** the port is in a different VLAN, or still in default VLAN 1 because it was never assigned.

**Fix:**

```
configure terminal
interface <port>
 switchport mode access
 switchport access vlan <correct-id>
 exit
end
```

**Also verify the VLAN exists on this switch.** A port assigned to a VLAN that is not in the VLAN database will show as inactive. `show vlan brief` lists every VLAN the switch knows about.

See incident NET-001.

**Next:** VLAN correct → Step 5.

---

## Step 5 — Verify IP address and subnet mask

**What to check**

On the host: `ipconfig /all`, or the Packet Tracer IP Configuration panel.
Compare against `network-design/ip-address-plan.md`.

**Why**

The right VLAN with the wrong subnet is the same failure as the wrong VLAN. Both leave the host unable to reach its gateway.

**Good result:** IP is inside the VLAN's subnet, mask is correct (255.255.255.0 throughout this design).
**Bad result:**

| Symptom | Meaning |
|---|---|
| Address is `169.254.x.x` | APIPA. DHCP was requested and no server answered. Jump to Step 9 |
| Address is in the wrong subnet | Static misconfiguration, or a DHCP pool with the wrong `network` statement |
| Mask is wrong | Host miscalculates which destinations are local. Produces "reaches some things, not others" |
| Address duplicates another host | IP conflict. Check whether a static address was taken from inside a DHCP range |

**Next:** addressing correct → Step 6.

---

## Step 6 — Verify default gateway

**What to check**

On the host: `ipconfig /all` for the gateway value, then:

```
ping <gateway>
```

**Why**

A host sends all off-subnet traffic to its default gateway. A wrong, missing, or unreachable gateway means the host can reach its own subnet and nothing else. That symptom is easily mistaken for a routing or trunk fault.

**Good result:** gateway matches the SVI for that VLAN (`.1` in each subnet here) and responds to ping.
**Bad result:** gateway is wrong, blank, or does not respond.

**Diagnostic distinction:** if the configured gateway does not respond but the correct SVI address does, the gateway value is wrong — not the network. That is exactly incident NET-003.

**If the gateway does not respond at all**, the SVI may be down. Check on the core:

```
show ip interface brief
```

An SVI shows down when its VLAN has no active ports, or when it was never brought up with `no shutdown`.

**Next:** gateway reachable → the local host is healthy. Continue to Step 7 for cross-switch or inter-VLAN problems.

---

## Step 7 — Verify trunk

**What to check**

```
show interfaces trunk
```

On **both ends** of the link.

**Why**

If the fault only affects traffic between switches — while traffic within a switch is fine — the trunk is the prime suspect. A trunk silently discards any VLAN not on its allowed list. No error, no log, no down interface.

**Good result:**

- Mode `on`, encapsulation `802.1q`, status `trunking`
- Native VLAN matches on both ends (99 in this design)
- The required VLAN appears in "Vlans allowed on trunk"
- The required VLAN also appears in "Vlans in spanning tree forwarding state and not pruned"

**Bad result:** the VLAN is missing from either list, the native VLAN differs between ends, or status reads `not-trunking`.

**The two allowed-VLAN lines mean different things.** The first is what you configured. The second is what is actually forwarding right now. When they disagree, that gap is the fault.

**Fix:**

```
configure terminal
interface <trunk-port>
 switchport trunk allowed vlan add <vlan-id>
 exit
end
```

**Use the `add` keyword.** Without it, the command replaces the entire allowed list rather than appending to it — which is how this fault is usually created in the first place. See incident NET-002.

**Corroborate with the MAC table.** On the far switch, `show mac address-table vlan <id>` should show MACs learned via the trunk port. If a VLAN has active hosts on the other switch but no MACs arriving over the trunk, the trunk is not carrying it.

**Next:** trunk carrying the VLAN → Step 8.

---

## Step 8 — Verify routing

**What to check**

On CORE-SW-01:

```
show ip route
show ip interface brief
```

**Why**

Traffic between VLANs must be routed. In this design that happens on CORE-SW-01 via its SVIs.

**Good result:** four connected (`C`) routes, one per VLAN subnet, each pointing at its SVI. All four SVIs up/up.
**Bad result:** missing routes, or SVIs showing down.

**The most common cause of no routing at all:** `ip routing` was never enabled. Without it the 3650 is a Layer 2 switch with IP addresses on it — the SVIs come up, each VLAN can reach its own SVI, and nothing routes between them. Verify with:

```
show running-config | include ip routing
```

**If a single SVI is down**, its VLAN has no active ports. An SVI needs the VLAN to exist and at least one port in it to be up — either an access port with a live device or a trunk carrying that VLAN.

**Useful confirmation from the client side:**

```
tracert <destination>
```

The first hop should be the source VLAN's gateway. If the trace dies at the first hop, the problem is between the host and its gateway. If it reaches the gateway and stops, the problem is on the router.

**Next:** routing confirmed → Step 9 for addressing problems, Step 10 for name problems, Step 11 for selective blocking.

---

## Step 9 — Verify DHCP

**What to check**

Client side first:

```
ipconfig /all
```

Then on the DHCP server (CORE-SW-01):

```
show ip dhcp pool
show ip dhcp binding
show running-config | section dhcp
```

**Why**

DHCP delivers four things — address, mask, gateway, DNS. Any one being wrong produces a different symptom, and reading which symptom applies tells you where to look.

**Reading the client state**

| Client shows | Meaning | Where to look |
|---|---|---|
| `169.254.x.x` (APIPA) | No DHCP response reached the client | Wrong VLAN, pool not configured, pool exhausted, or no `ip helper-address` across a router boundary |
| Address in the wrong subnet | Pool `network` statement is wrong | Pool configuration |
| Address correct, no off-subnet access | `default-router` is wrong or missing | Pool configuration — see NET-003 |
| Connectivity fine, names fail | `dns-server` is wrong or missing | Pool configuration, then Step 10 |

**Server-side checks**

`show ip dhcp pool` gives total addresses, leased addresses and the current index. If leased equals total, the pool is exhausted.
`show ip dhcp binding` lists active leases with client MACs — useful for confirming a specific client actually got an address.

**Broadcast note.** DHCP DISCOVER is a broadcast, and routers do not forward broadcasts. The server must be in the client's VLAN, or the VLAN's router interface must carry `ip helper-address` pointing at the server. In this lab CORE-SW-01 is both router and DHCP server with an SVI directly in VLAN 30, so no helper is required — but a missing helper-address is a very common real-world cause of DHCP failure.

**After any pool change, force renewal:**

```
ipconfig /release
ipconfig /renew
```

Existing clients hold their old lease until it expires. The renewal is part of the fix.

**Next:** valid lease with all four values correct → Step 10.

---

## Step 10 — Verify DNS

**What to check**

```
ping <ip-address>
ping <hostname>
ipconfig /all
```

Then inspect the DNS service on DC-SERVER-01 (Services → DNS).

**Why**

DNS sits above IP connectivity. Separating the two takes one pair of pings and eliminates the entire network from consideration.

| `ping <IP>` | `ping <hostname>` | Conclusion |
|---|---|---|
| Works | Works | Healthy |
| Works | Fails | **DNS problem.** Network path is fine |
| Fails | Fails | **Network problem.** Return to Step 1. DNS is irrelevant until connectivity works |

**When it is DNS, check in this order:**

1. Does the client have the right DNS server address? (`ipconfig /all` — expect 192.168.10.11)
2. Can the client reach the DNS server? (`ping 192.168.10.11`)
3. Is the DNS service actually running? (DC-SERVER-01 → Services → DNS)
4. Does the specific record exist and point at the right address?

Steps 1 and 2 use the client only. If both pass and names still fail, the fault is on the server. See incident NET-004.

**Note:** a server that responds to ping is not necessarily running its services. Host-level monitoring would have shown DC-SERVER-01 green throughout NET-004 while name resolution was entirely down.

**Next:** name resolution confirmed → Step 11.

---

## Step 11 — Verify ACL

**What to check**

On CORE-SW-01:

```
show access-lists
show ip interface <interface>
show running-config | section access-list
```

**Why**

An ACL blocks specific traffic while leaving everything else working. The signature is selective failure with routing intact.

**When to suspect an ACL:**

- The source can reach one remote VLAN but not another
- The destination is reachable from a *different* source VLAN
- Routing table and SVIs are all healthy
- The failure depends on **who is asking**, not on the path

**Good result:** only the intended ACLs are present, applied to the intended interfaces in the intended direction.
**Bad result:** an unexpected ACL, an ACL on the wrong interface, or the wrong direction.

**Three rules that explain most ACL faults:**

1. **Top-down, first match wins.** Order matters. A broad permit above a specific deny makes the deny unreachable.
2. **Implicit `deny any` at the end.** Every ACL ends with an invisible deny-all. An ACL of only deny statements blocks everything — an explicit `permit ip any any` is mandatory, not optional.
3. **Direction is relative to the interface.** `in` filters traffic entering the switch from that VLAN, so an inbound ACL on VLAN 40 can only ever filter VLAN 40's own traffic.

**Match counters are evidence.** `show access-lists` shows a hit count per line. A deny line with a rising counter is actively dropping traffic right now — that is proof, not inference.

**Verify the application point separately from the contents.** A correct ACL on the wrong interface is an outage. `show ip interface <interface>` states which ACL is applied and in which direction. See incident NET-005.

**Next:** ACL confirmed correct or corrected → Step 12.

---

## Step 12 — Retest

**What to check**

Re-run the tests that originally failed, plus the tests that were working.

**Why**

Confirming the fault is fixed is only half of validation. The other half is confirming the fix did not break something else.

**Minimum retest set:**

- [ ] Ping the default gateway from the affected host
- [ ] Ping a host in the same VLAN
- [ ] Ping a host in a different VLAN
- [ ] Resolve a hostname
- [ ] Verify the switch port state, VLAN and MAC table entry
- [ ] Confirm any intentional restrictions still function

That last item catches the most damaging class of mistake. Fixing an ACL incident by removing the wrong ACL is not a fix — see NET-005, where confirming the VLAN 40 restriction still worked was part of resolution.

**For a wider check**, re-run the reachability matrix in `network-design/reachability-baseline.md` and compare against the known-good baseline. Any cell that changed and should not have is a new problem you just introduced.

---

## Step 13 — Document the resolution

**What to record**

- What was reported, and what was actually wrong
- Which checks were run and what they returned
- The root cause, stated specifically
- The exact change made
- The validation evidence
- Any documentation updated as a result

**Why**

A resolved incident with no record teaches nobody anything and gets rediscovered from scratch next time. A resolved incident with a good record turns one person's twenty minutes into a two-minute fix for the next person.

**Also update the physical documentation** if anything changed: `cabling/switch-port-map.csv`, `cabling/cable-map.csv`, `rack-design/asset-inventory.csv`, port descriptions on the switch, and the physical cable label. Documentation updated in the same change window stays accurate; documentation updated "later" does not. See NET-007.

**Save the configuration.** A fix that is not written to startup-config disappears at the next reload:

```
copy running-config startup-config
```

---

## Fast-path reference

For experienced use — the shortest path from symptom to likely cause.

| Symptom | Most likely cause | First command |
|---|---|---|
| Link light dark | Cable, or port administratively shut | `show interfaces status` |
| Port shows `disabled` | Someone ran `shutdown` | `show running-config interface <port>` |
| Port shows `err-disabled` | Protection feature triggered | `show interface <port>` |
| Link up, no connectivity at all | Wrong VLAN | `show interfaces status` |
| Cannot reach own gateway | VLAN, IP, or mask wrong | `ipconfig /all` then `show vlan brief` |
| Local works, cross-switch fails | Trunk missing a VLAN | `show interfaces trunk` |
| Own subnet works, everything else fails | Wrong or unreachable gateway | `ipconfig /all` |
| No VLAN can reach any other | `ip routing` not enabled | `show ip route` |
| One VLAN unreachable, others fine | ACL | `show access-lists` |
| Destination fine from one VLAN, not another | ACL on the source interface | `show ip interface <vlan>` |
| Client has 169.254.x.x | No DHCP response | `show ip dhcp pool` |
| Client has IP but no off-subnet access | Wrong `default-router` in pool | `show running-config \| section dhcp` |
| Ping by IP works, by name fails | DNS | Check DNS service and records |
| Device not where docs say | Documentation drift | `show mac address-table` |

---

## Command quick reference

| Purpose | Command |
|---|---|
| Port state, speed, duplex, VLAN — all in one line | `show interfaces status` |
| Port errors and counters | `show interface <port>` |
| What is documented as connected where | `show interfaces description` |
| VLAN database and port membership | `show vlan brief` |
| What device is actually on which port | `show mac address-table` |
| MAC entries for one VLAN | `show mac address-table vlan <id>` |
| Trunk mode, native VLAN, allowed VLANs | `show interfaces trunk` |
| SVI addresses and state | `show ip interface brief` |
| Routing table | `show ip route` |
| DHCP pool utilisation | `show ip dhcp pool` |
| Active DHCP leases | `show ip dhcp binding` |
| ACL contents and match counters | `show access-lists` |
| Which ACL is applied to an interface | `show ip interface <interface>` |
| Full or filtered configuration | `show running-config`, `show running-config interface <port>` |
| Client-side addressing | `ipconfig /all` |
| Force a new DHCP lease | `ipconfig /release` then `ipconfig /renew` |
| Path to a destination | `tracert <ip>` |
