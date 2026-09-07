# Phase 4 — Layer 3 SVIs & Inter-VLAN Routing

**Prerequisite:** Phases 1–3 complete. VLANs created, access ports assigned, both trunks carrying VLANs 10/20/30/40/99, TEST-PC-01 returned to VLAN 40.

**Goal:** turn CORE-SW-01 into the router for this network. Create a gateway for each VLAN, enable routing between them, and convert the isolation you proved in Phase 2 into controlled reachability.

**Estimated time:** 40–50 minutes.

---

## 4.1 The concept, before the commands

Phase 2 ended with a deliberate failure: OPS-PC-01 in VLAN 30 could not reach a server in VLAN 10. That was correct. VLANs are separate broadcast domains, and a switch operating at Layer 2 has no mechanism to move a frame between them.

Moving traffic between subnets requires a **router** — a device that reads the destination IP address, consults a routing table, and forwards the packet out toward the correct network. A Layer 3 switch does exactly this, in switching hardware, using **SVIs**.

An **SVI** (Switched Virtual Interface) is a virtual Layer 3 interface representing an entire VLAN. `interface Vlan10` is not a physical port — it is the router's presence inside VLAN 10. Give it 192.168.10.1, and every host in VLAN 10 that has 192.168.10.1 configured as its default gateway can now send off-subnet traffic to it.

### What the default gateway actually does

This is one of the most commonly asked entry-level interview questions, so be precise about it.

When a host wants to send a packet, it compares the destination IP against its own IP and subnet mask:

- **Destination is on my subnet** → send the frame directly to that host's MAC address. No router involved.
- **Destination is not on my subnet** → send the frame to the default gateway's MAC address, and let the router figure it out.

The gateway is not consulted for local traffic at all. That is why, in Phase 2, DC-SERVER-01 could still ping DC-SERVER-02 despite having a gateway configured that did not exist — same subnet, gateway never used. And it is why a host with a wrong or missing gateway shows the exact symptom of "can ping things nearby, cannot reach anything else."

### SVI up/down behavior

An SVI comes up only when both conditions are met:

1. The VLAN exists in the VLAN database, and
2. At least one port in that VLAN is up — either an access port with a live device, or a trunk carrying that VLAN

This matters practically. If you create `interface Vlan40` and it stays down, the usual cause is that nothing is active in VLAN 40 yet. That is a diagnostic, not a bug.

---

## 4.2 Enable routing and create the SVIs

On **CORE-SW-01** only:

```
enable
configure terminal
!
ip routing
!
interface Vlan10
 description GATEWAY - VLAN 10 SERVERS
 ip address 192.168.10.1 255.255.255.0
 no shutdown
 exit
!
interface Vlan20
 description GATEWAY - VLAN 20 MANAGEMENT
 ip address 192.168.20.1 255.255.255.0
 no shutdown
 exit
!
interface Vlan30
 description GATEWAY - VLAN 30 OPERATIONS
 ip address 192.168.30.1 255.255.255.0
 no shutdown
 exit
!
interface Vlan40
 description GATEWAY - VLAN 40 UNTRUSTED
 ip address 192.168.40.1 255.255.255.0
 no shutdown
 exit
!
end
copy running-config startup-config
```

### `ip routing` is the line that matters

Without it, the 3650 is a Layer 2 switch with IP addresses on it. The SVIs will come up, you will be able to ping each one from its own VLAN, and nothing will route between them. It is a single command and it is the single most common omission in this kind of build. If inter-VLAN pings fail later, check this first with `show ip route` — on a switch without `ip routing`, that command returns almost nothing useful.

---

## 4.3 Give the access switches management IPs

The access switches are Layer 2 devices. They do not route. But they each need one IP address so a technician at NOC-PC-01 can reach them for management, and so they appear in the management VLAN like real infrastructure.

In this lab, each Layer 2 access switch uses one management SVI on VLAN 20. The access switches do not perform inter-VLAN routing; CORE-SW-01 performs that function. Each access switch also needs a **default gateway** — not for routing transit traffic, but so its own management traffic can reply to hosts in other subnets.

### ACCESS-SW-01

```
enable
configure terminal
!
interface Vlan20
 description MANAGEMENT IP - ACCESS-SW-01
 ip address 192.168.20.2 255.255.255.0
 no shutdown
 exit
!
ip default-gateway 192.168.20.1
!
end
copy running-config startup-config
```

### ACCESS-SW-02

```
enable
configure terminal
!
interface Vlan20
 description MANAGEMENT IP - ACCESS-SW-02
 ip address 192.168.20.3 255.255.255.0
 no shutdown
 exit
!
ip default-gateway 192.168.20.1
!
end
copy running-config startup-config
```

### Why `ip default-gateway` and not `ip route`

On a Layer 2 switch, `ip default-gateway` tells the switch's own management stack where to send replies destined off-subnet. It affects only traffic the switch itself originates or answers — never traffic passing through it. A Layer 3 switch with `ip routing` enabled ignores `ip default-gateway` entirely and uses `ip route 0.0.0.0 0.0.0.0` instead. Knowing which command belongs on which device type is a clean, small piece of knowledge that separates someone who has built a network from someone who has read about one.

**Note on ACCESS-SW-01:** VLAN 20 has no access ports on this switch — DC-SERVER-01's management NIC lands on ACCESS-SW-02. The SVI will still come up because the trunk on `Gi0/1` carries VLAN 20. If it stays down, verify VLAN 20 is in the trunk's allowed list.

---

## 4.4 Phase 4 validation

### Test 1: SVI status

On CORE-SW-01:

```
show ip interface brief
```

All four VLAN interfaces should show the correct IP with Status `up` and Protocol `up`.

If any show `down/down`, the VLAN has no active port. VLAN 40's only member is TEST-PC-01 on ACCESS-SW-02 — confirm that host is powered on and its port is up.

### Test 2: routing table

```
show ip route
```

You should see four connected routes, marked `C`, one per subnet, each pointing at its SVI:

```
C  192.168.10.0/24 is directly connected, Vlan10
C  192.168.20.0/24 is directly connected, Vlan20
C  192.168.30.0/24 is directly connected, Vlan30
C  192.168.40.0/24 is directly connected, Vlan40
```

Four connected routes is the proof that routing is enabled and the switch knows about all four networks. No static or dynamic routing protocol is needed — every network is directly attached.

### Test 3: gateway reachability from each VLAN

From **DC-SERVER-01** (VLAN 10):

```
ping 192.168.10.1
```

This is the first check in almost every real "no network connectivity" ticket. If a host cannot reach its own gateway, the problem is local: wrong VLAN, wrong IP, wrong mask, or a Layer 2 fault. Nothing further up the stack is worth checking until this passes.

### Test 4: inter-VLAN routing — the reversal

This is the test that failed in Phase 2 and must now succeed.

Temporarily set **OPS-PC-01** to static `192.168.30.50 / 255.255.255.0 / gateway 192.168.30.1`, then:

The address `.50` is used deliberately because it falls **outside** the DHCP dynamic range that Phase 5 will create (`.101`–`.254`). Borrowing an address from inside a future DHCP pool is how duplicate-address incidents get created.

```
ping 192.168.10.11
```

This should now **succeed**. Screenshot it next to the Phase 2 failure — same source, same destination, same cabling, same VLANs. The only thing that changed is that a router now exists between them. That before/after pair is one of the strongest single pieces of evidence in this portfolio.

Now trace the path:

```
tracert 192.168.10.11
```

You should see 192.168.30.1 as the first hop, then the destination. That first hop is the SVI doing its job.

Leave OPS-PC-01 statically addressed for now — Test 5 uses it.

### Test 5: management VLAN reachability

Run this from **OPS-PC-01 only**, while it is still statically addressed at 192.168.30.50:

```
ping 192.168.20.2
ping 192.168.20.3
```

Both access switches should reply. This confirms two things at once: the access switch management SVIs are up, and their `ip default-gateway` settings are correct — without the gateway the switches would receive the echo request but be unable to route the reply back to VLAN 30.

This is also the exact path that NET-005 will later break, so this successful result is the baseline that incident is measured against. Screenshot it.

**Now return OPS-PC-01 to DHCP.** Desktop → IP Configuration → select **DHCP**. It will fail to obtain an address until Phase 5 builds the pool, which is expected. Do not leave it static — a workstation left with a hardcoded address is one of the most common sources of drift between documentation and reality.

### Test 6: full reachability matrix

Fill this in from actual test results. Leave nothing blank — a matrix with guesses in it is worse than no matrix.

| From ↓ / To → | 192.168.10.1 | 192.168.10.11 | 192.168.20.1 | 192.168.20.2 | 192.168.30.1 | 192.168.40.1 |
|---|---|---|---|---|---|---|
| DC-SERVER-01 (VLAN 10) | | | | | | |
| DC-SERVER-02 (VLAN 10) | | | | | | |
| OPS-PC-01 (VLAN 30, static .50) | | | | | | |
| TEST-PC-01 (VLAN 40) | | | | | | |

At this stage, with no ACL yet, **every cell should be a success.** That is intentional. Phase 7 will introduce the ACL that turns some of these into deliberate failures, and this matrix is the baseline that proves the ACL did what it was supposed to and nothing more.

Save this table into `network-design/reachability-baseline.md`.

---

## 4.5 Phase 4 screenshots

Capture only what `screenshots/README.md` lists for this phase. The project targets roughly 25–30 screenshots in total (12–15 baseline, 2 per incident), so capture the clearest single view of each capability rather than every command you run.


---

## 4.6 Configuration capture

Phase 4 is the first point where the configurations are substantial enough to archive. On each switch:

```
show running-config
```

Copy the full output into the repository:

- `configurations/core-switch.txt`
- `configurations/access-switch-01.txt`
- `configurations/access-switch-02.txt`

Capture these again at the end of Phase 7 once DHCP and the ACL are in place. Keeping a clean pre-incident config archive is what lets you restore a known-good baseline after the incident simulations, and it is genuinely how change control works.

---

## 4.7 Report back before Phase 5

Paste the actual output of:

1. `show ip interface brief` from CORE-SW-01
2. `show ip route` from CORE-SW-01
3. `ping 192.168.10.1` from DC-SERVER-01
4. `ping 192.168.10.11` from OPS-PC-01 (static .50)
5. `tracert 192.168.10.11` from OPS-PC-01 (static .50)
6. `show ip interface brief` from ACCESS-SW-01
7. The completed reachability matrix from Test 6

Flag any SVI that stayed `down/down`, and any cell in the matrix that failed. A failure at this stage is almost always a missing `ip routing`, a VLAN missing from a trunk, or a host with a wrong gateway — all three are worth diagnosing yourself before I explain them, because that diagnosis is the actual skill this project is meant to demonstrate.

Phase 5 (IOS DHCP for VLAN 30, and the DHCP DORA process) starts once you post these.
