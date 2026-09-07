# Interview Preparation

> ## DRAFT — USE ONLY AFTER LIVE PACKET TRACER EXECUTION IS COMPLETED AND VERIFIED.
>
> These answers are written in the first person and past tense ("I built", "I configured", "I broke", "I fixed", "I verified") as interview wording for use **after** the lab has actually been executed. Nothing below should be spoken aloud until the build, the seven incidents and the evidence capture have genuinely been performed in Packet Tracer.

---

Answers pitched at **Data Center Technician / NOC Technician / Network Technician I** level. The goal is to sound like someone who has actually built and broken a network, not someone reciting a textbook.

**General advice:** answer the question asked, give a concrete example from this project when you can, and stop. Over-explaining reads as uncertainty. If you do not know something, say so and say what you would check.

---

## Project explanations

### 30-second version

> I built a simulated data center network in Cisco Packet Tracer with three switches, four VLANs, and six endpoints across two racks. I configured VLANs and access ports, 802.1Q trunks, inter-VLAN routing on a Layer 3 switch, DHCP, DNS, and an access control list. Then I broke it seven different ways — wrong VLAN, trunk misconfiguration, DHCP failure, DNS failure, an ACL blocking legitimate traffic, a shut-down port, and a documentation mismatch — and diagnosed and fixed each one, writing up a full incident report every time. I also produced the physical documentation side: rack elevations, an asset inventory, a switch-port map, a cable map, and a server deployment checklist.

### 2-minute version

> The project targets the physical and network deployment side of data center technician work.
>
> The network is three Cisco switches: a 3650 acting as the Layer 3 core, and two 2960 access switches, one per simulated rack. Four VLANs — servers, management, operations, and an untrusted test VLAN — each with its own /24 subnet. The third octet matches the VLAN ID, so an IP address tells you immediately which VLAN it belongs to.
>
> The core does inter-VLAN routing using SVIs, one per VLAN, acting as the default gateways. It also runs the IOS DHCP server for the operations VLAN and holds the access control list. The two access switches are pure Layer 2 with a single management SVI each so they can be administered.
>
> The trunks were the part I was most careful about validating. Rather than just running `show interfaces trunk` and calling it done, I temporarily moved a host onto the opposite switch in the server VLAN with **no default gateway configured**, so a successful ping could only be explained by both trunks actually carrying that VLAN — routing couldn't have rescued it. Then I pruned the VLAN from one trunk and confirmed the ping failed while same-switch traffic kept working. That contrast is the diagnostic signature of a trunk fault.
>
> On the documentation side I built rack elevations for both racks, an asset inventory, a switch-port map, and a cable map with a labeling convention where both ends of every cable carry the same label, and the same label goes into the switch port description. So `show interfaces description` tells you what's connected without walking the floor.
>
> Then seven incidents. The one I found most interesting was the ACL incident, because I diagnosed it almost entirely with ping results before touching an ACL command — the client could reach its gateway, could reach the server VLAN, but not the management VLAN, and a host in a different VLAN could reach management fine. So routing worked, the destination was healthy, and the failure depended on the source. That combination only means filtering.
>
> I should be clear it's a Packet Tracer simulation, not production experience. I haven't terminated fiber or racked physical hardware — the cabling and transceiver material in the repo is documented understanding, and it's labelled that way.

---

## Core networking concepts

### What is a switch?

A switch connects devices within a network and forwards traffic based on **MAC addresses**, operating at Layer 2. It learns which MAC address lives on which port by reading the source address of incoming frames, builds a MAC address table, and forwards frames only to the port where the destination lives — rather than flooding every port like a hub would.

In this project the two 2960s are access switches; every server and workstation plugs into one of them.

### What is a router?

A router connects **different networks** and forwards traffic based on **IP addresses**, operating at Layer 3. It reads the destination IP, consults a routing table, and forwards toward the correct network.

**Switch vs router in one line:** a switch moves traffic *within* a network using MAC addresses; a router moves traffic *between* networks using IP addresses.

**The nuance worth adding:** a Layer 3 switch does both. My core switch does normal Layer 2 switching but also has `ip routing` enabled with an SVI per VLAN, so it routes between VLANs in switching hardware. That's how most data center aggregation is done.

### What is a VLAN?

A VLAN is a logical division of a physical switch into separate broadcast domains. Two ports in different VLANs on the same physical switch cannot communicate at Layer 2 at all — as if they were plugged into two different switches with no cable between them.

**Why they're used:** segmentation for security, containing broadcast traffic, and organising the network by function rather than by physical location.

**Concrete example:** I have VLAN 10 for servers, VLAN 20 for management, VLAN 30 for workstations, and VLAN 40 for untrusted test devices. That separation is what lets me apply an ACL blocking VLAN 40 from reaching VLAN 20.

### What is an access port?

A port that belongs to exactly one VLAN and carries untagged traffic for that VLAN only. Endpoints — servers, workstations, printers — connect to access ports. The endpoint has no idea VLANs exist.

```
switchport mode access
switchport access vlan 10
```

I set the mode explicitly rather than leaving the port on dynamic negotiation, so a host-facing port can never negotiate itself into a trunk.

### What is a trunk port?

A port that carries **multiple VLANs** over a single physical link, using 802.1Q tagging to keep them separate.

**Access port carries one VLAN, trunk carries many.** In my network there is one cable between each access switch and the core, but four VLANs need to cross it — that link is a trunk.

### What is 802.1Q?

The standard for VLAN tagging on trunk links. It inserts a 4-byte tag into the Ethernet frame header containing the VLAN ID, so the receiving switch knows which VLAN each frame belongs to. The tag is added when the frame enters the trunk and stripped when it leaves.

**The exception is the native VLAN** — frames in the native VLAN cross the trunk untagged. That's why the native VLAN has to match on both ends. I set mine to VLAN 99, an otherwise unused VLAN, so stray untagged traffic doesn't land in VLAN 1 with unconfigured ports.

### What is an SVI?

A Switched Virtual Interface — a virtual Layer 3 interface representing an entire VLAN on a switch. It's not a physical port; it's the router's presence inside that VLAN.

`interface Vlan10` with IP 192.168.10.1 becomes the default gateway for every host in VLAN 10.

**One useful detail:** an SVI only comes up when the VLAN exists *and* at least one port in it is up — either an access port with a live device or a trunk carrying that VLAN. So an SVI stuck down usually means nothing is active in that VLAN, which is a diagnostic rather than a bug.

### What is a default gateway?

The address a host sends traffic to when the destination is not on its own subnet.

**How the host decides:** it compares the destination IP against its own IP and subnet mask.
- Destination on my subnet → send directly to that host's MAC address, no router involved
- Destination off my subnet → send to the default gateway and let the router handle it

**Why that matters diagnostically:** the gateway is never consulted for local traffic. A host with a wrong or missing gateway can still ping everything on its own subnet and nothing beyond it. That's exactly the symptom in my DHCP incident — the pool handed out a gateway address that didn't exist, so clients could reach each other but nothing else.

### What is a MAC address table?

The table a switch builds mapping MAC addresses to switch ports. The switch learns entries by reading the source MAC of every incoming frame and noting which port it arrived on.

```
show mac address-table
```

**Why it matters for this job:** The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation.

I used it in my documentation-mismatch incident: a server had been moved to a different port without any records being updated, and cross-referencing the learned MAC against the server's NIC MAC proved where it actually was.

### What is DHCP?

Dynamic Host Configuration Protocol — automatically assigns IP configuration to clients so it doesn't have to be set by hand.

**It delivers four things:** IP address, subnet mask, default gateway, and DNS server. Any one of them being wrong causes a different problem.

In my lab, workstations use DHCP; servers stay static, because server addresses need to be predictable for DNS records, monitoring and documentation.

### Explain DORA

The four-message DHCP exchange:

| Step | Message | Who | What |
|---|---|---|---|
| **D** | Discover | Client | Broadcast — "is there a DHCP server?" Client has no IP yet, so source is 0.0.0.0 |
| **O** | Offer | Server | "Here's an address you can have" — with mask, gateway, DNS, lease time |
| **R** | Request | Client | Broadcast — "I accept" — broadcast so other servers that offered know to withdraw |
| **A** | Acknowledge | Server | "Confirmed, it's yours for the lease duration" |

**The detail worth adding:** DISCOVER is a broadcast, and routers don't forward broadcasts. So the DHCP server has to be in the client's VLAN, or the router interface needs `ip helper-address` to relay requests to a server elsewhere. In my lab the core switch is both router and DHCP server with an SVI directly in VLAN 30, so no helper is needed — but a missing helper-address is a very common real-world DHCP failure.

**Quick diagnostic:** if a client has a 169.254.x.x address, that's APIPA. It asked and got no answer — wrong VLAN, no pool, pool exhausted, or missing helper-address.

### What is DNS?

Domain Name System — translates hostnames into IP addresses.

**The reason it matters for troubleshooting** is that it sits above IP connectivity, so you can separate the two with one pair of pings:

| ping by IP | ping by name | Conclusion |
|---|---|---|
| Works | Works | Healthy |
| Works | Fails | **DNS problem** — network path is fine |
| Fails | Fails | **Network problem** — DNS is irrelevant until connectivity works |

That's my NET-004 incident. Users reported the servers as down; the servers were completely fine and only name resolution had failed. Being able to make that call in thirty seconds stops the whole team troubleshooting the wrong layer.

### What is an ACL?

An Access Control List — a set of rules permitting or denying traffic based on source, destination, protocol and port.

Mine denies the untrusted VLAN from reaching the management VLAN and permits everything else:

```
deny ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255
permit ip any any
```

**Three things that explain most ACL mistakes:**

1. **Top-down, first match wins.** Order matters — a broad permit above a specific deny makes the deny unreachable.
2. **There's an implicit `deny any` at the end.** An ACL with only deny statements blocks everything. That explicit `permit ip any any` is mandatory.
3. **Direction is relative to the interface.** Applied inbound on VLAN 40, it can only filter traffic *sourced from* VLAN 40.

That third point is why my ACL incident had to be a separately misapplied ACL on VLAN 30 — the baseline ACL on VLAN 40 physically cannot block VLAN 30's traffic.

---

## Practical / hands-on questions

### How do you locate a server's switch port?

Working from most to least preferred:

1. **Read the cable label.** If labeling is disciplined, that's the whole job. My convention puts the same label on both ends: `A01-SRV001-NIC1__A01-SW001-Fa0-1`.
2. **Check the port map documentation.** `switch-port-map.csv` in my repo.
3. **Check the switch port descriptions** — `show interfaces description` returns what's documented as connected.
4. **Check the MAC address table.** `show mac address-table`, then compare the learned MAC against the server's NIC MAC. The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation.
5. **Shut/no-shut the port and watch the link LED** — effective but it drops the link, so only on a host confirmed out of service.
6. **Toner and probe** for unlabelled copper, or a visual fault locator for fiber.

I'd start at 1 and drop to 4 to confirm. Method 4 is what caught a server that had been moved without the records being updated in my NET-007 incident.

### What does a link light tell you?

| State | Meaning | What to do |
|---|---|---|
| **Off / dark** | No physical link — Layer 1 problem | Check cable seating, cable integrity, correct port, far-end device powered, port not shut |
| **Solid on** | Link up, no traffic passing | Physical layer is fine. Move up: VLAN, IP, gateway |
| **Blinking** | Link up and traffic flowing | Physical layer confirmed working |
| **Amber** | Vendor-dependent — often speed, disabled port, or STP not yet forwarding | Check `show interfaces status` for the real answer |

**The key inference:** a dark light means stop and fix Layer 1, because nothing above it can work. A lit light means Layer 1 is fine and the problem is above it.

**But don't trust the LED alone.** `show interfaces status` is authoritative and gives you port state, speed, duplex and VLAN in one line. The LED tells you about the physical link; the switch tells you about the configuration.

### How would you troubleshoot a server with no network connectivity?

Bottom-up, stopping at the first failed check. Each step is cheaper than the next and eliminates a whole category of causes.

1. **Link light and `show interfaces status`.** Is the port `connected`, `notconnect`, or `disabled`? `disabled` means someone shut it — that's a CLI fix, no floor visit needed.
2. **Confirm I'm looking at the right port.** `show mac address-table` cross-checked against the port map.
3. **Check VLAN.** `show vlan brief` and `show interfaces status`. Wrong VLAN is the most common fault and looks perfectly healthy physically.
4. **Check IP and subnet mask** on the host. A 169.254 address means DHCP failed.
5. **Ping the default gateway.** If that fails, the problem is local — VLAN, IP, mask, or Layer 1. If it succeeds, the host is healthy and the problem is beyond it.
6. **If it's cross-switch, check the trunk.** `show interfaces trunk` — is the VLAN in the allowed list?
7. **If it's inter-VLAN, check routing.** `show ip route` on the core, and confirm `ip routing` is actually enabled.
8. **Check DHCP and DNS** if the symptoms point there.
9. **Check ACLs** if the failure is selective — reaching one destination but not another.
10. **Retest, including the things that were already working**, then document.

**The two-minute shortcut** before running any command: ping the gateway, ping something in the same VLAN on the same switch, ping something in the same VLAN on a different switch, ping something in another VLAN, and ping by name versus by IP. That pattern usually names the fault category before I've logged into anything.

### How would you deploy a new server onto the network?

I built a checklist for exactly this. In summary:

**Physical:** verify rack ID, rack unit, asset ID against the ticket. Confirm rails seated, airflow orientation correct, both PSUs connected.

**Connection:** identify the NIC and record its MAC. Verify the assigned switch and port from the port map, confirm the port is genuinely unused, use the correct cable type, and label both ends with the same label before dressing the cable.

**Link:** check the link LED at both ends. `show interfaces status` must show `connected`. Confirm speed and duplex negotiated correctly, and no error counters climbing.

**Layer 2:** confirm VLAN with `show vlan brief` and `show interfaces status`. Confirm port mode is access. Set the port description to match the cable label.

**Layer 3:** verify IP, mask, gateway and DNS against the address plan. Confirm no IP conflict and that the address isn't inside a DHCP range.

**Test:** ping the gateway first — that's the decisive local check. Then a host in the same VLAN, then a host in a different VLAN, then a hostname to test DNS, then whatever specific connectivity the server actually needs. General ping success doesn't prove it can do its job.

**Verify on the switch:** `show mac address-table` to confirm the server's MAC appears on the port the documentation says. That's the proof the right device is on the right port.

**Document:** update the port map, cable map, asset inventory, rack elevation and IP plan. Add the DNS record.

**Close:** record the evidence in the ticket — port, VLAN, IP, MAC, ping results — and close it.

**The point I'd make:** every step is *verified*, not assumed. "It should be VLAN 10" isn't verification; `show interfaces status` showing VLAN 10 is.

---

## Cabling and media

### Copper vs fiber?

| | Copper | Fiber |
|---|---|---|
| Signal | Electrical over twisted pair | Light through glass |
| Distance | 100 m maximum | Hundreds of metres to tens of kilometres |
| Interference | Susceptible to EMI | Immune |
| Cost | Cheaper cable and NICs | More expensive optics |
| Typical use | Server to top-of-rack, workstations | Rack to rack, row to row, between buildings |

**Practical rule:** copper for short runs inside a rack, fiber for anything longer or between racks and buildings.

**Honest note about my lab:** the inter-rack uplink is Cat6A copper because Packet Tracer's 2960 uplink ports are copper only. Real data centers may use multimode fiber, single-mode fiber, DAC/AOC, or other media and optics depending on distance, bandwidth and design. I've documented it that way rather than claiming fiber I didn't use.

### Cat6 vs Cat6A?

Cat6 handles 1 Gbps for the full 100 m but only reaches 10 Gbps over about 55 m. Cat6A does 10 Gbps for the full 100 m, using tighter twists and extra shielding to control alien crosstalk — interference between adjacent cables in a dense bundle, which is a real problem in a packed rack.

Cat6A is thicker and stiffer, which affects bend radius and how much airflow the bundle blocks.

### Single-mode vs multimode?

| | Multimode | Single-mode |
|---|---|---|
| Core diameter | 50 µm | ~9 µm |
| Distance | 100–550 m | 10 km to 80 km+ |
| Optics cost | Lower | Higher |
| Jacket colour | Aqua (OM3/OM4) | Yellow |
| Use | Inside the data center | Between buildings, campus, long haul |

**Why the difference:** multimode's wide core lets light enter at multiple angles, so different rays take slightly different path lengths and arrive spread out in time. That's modal dispersion, and it blurs the signal over distance. Single-mode's narrow core allows essentially one path, so it goes much further.

**Rule of thumb:** inside the data center, multimode. Between buildings, single-mode.

### What are SFP, SFP+ and QSFP?

Transceivers — hot-swappable modules that convert the switch's electrical signals to optical (or copper) signals on the cable. Because they're modular, one switch port can run copper, short-range fiber or long-range fiber depending on which module is fitted.

| Form factor | Speed |
|---|---|
| SFP | 1 Gbps |
| SFP+ | 10 Gbps |
| SFP28 | 25 Gbps |
| QSFP+ | 40 Gbps — "quad", four 10G lanes |
| QSFP28 | 100 Gbps — four 25G lanes |

**Markings you'll see:** `SR` is short range multimode, `LR` is long range single-mode, `1000BASE-T` is a copper SFP with an RJ45 port. `DAC` is Direct Attach Copper — a fixed cable with optics moulded on both ends, cheap and low-latency for very short in-rack runs.

**Operationally:** the transceivers must match at both ends, and must match the fiber type. An SR optic on single-mode fiber won't work. Many switches also reject third-party optics, so "the optic is fine, the switch refuses it" is a genuine failure mode. A transceiver is a field-replaceable part, so swapping a suspect one is a standard early troubleshooting step.

### What is a patch panel?

A passive termination point. Permanent structured cabling — through walls, floors and overhead trays — terminates on the back of the panel. Short, replaceable patch cords connect the front of the panel to the switch.

**Why:** the permanent cabling is never handled, so it isn't stressed by repeated re-patching. A failed patch cord is a two-second replacement; a failed in-wall run is a project. And every port is numbered, so tracing becomes reading rather than following a cable by hand.

### What is A/B redundancy?

Connecting critical servers to two independent network paths — NIC1 to switch A, NIC2 to switch B — on separate physical switches, ideally in separate racks, on separate power feeds. The intent is that no single failure takes a server offline.

**Being honest about my lab:** I did *not* implement this. Each device has a single path. My server's second NIC is a dedicated management interface on a separate VLAN — that's management separation, not redundancy. It wouldn't keep the server online if the access switch failed. Real A/B would need NIC teaming, dual switches per rack, and first-hop redundancy like HSRP or VRRP, which I kept out of scope.

---

## Explaining the incidents

Pick one and tell it as a story: symptom, what you checked, what it turned out to be, how you verified the fix.

### NET-001 — Wrong VLAN

> A server couldn't reach anything, but the link light was on and `show interfaces status` showed the port connected. The server's own IP config was correct and nothing had been recabled. I confirmed the right port with the MAC address table, then `show interfaces status` showed the port in VLAN 30 when the port map said VLAN 10. The server had a 192.168.10.x address sitting in the operations broadcast domain, so it could never even ARP for its gateway. Fixed with `switchport access vlan 10` and verified with a gateway ping and the MAC table showing the server relearned in VLAN 10.
>
> **The lesson:** a green link light proves Layer 1 and nothing more. Wrong VLAN is the most common entry-level fault and every physical indicator looks perfect.

### NET-002 — Trunk failure

> Workstations couldn't reach any server, but the three servers could ping each other fine. That contrast — works locally, fails across switches — points straight at the trunk. `show interfaces trunk` showed VLAN 10 missing from the allowed list on the access switch uplink. I also confirmed it on the core: no VLAN 10 MACs were being learned via that trunk, despite three active servers.
>
> Root cause was `switchport trunk allowed vlan` used without the `add` keyword, which replaces the entire list instead of appending. Fixed by restoring the full list.
>
> **The lesson:** trunk faults are silent. No error, no down interface, no log entry — just discarded frames. Only `show interfaces trunk` reveals it.

### NET-003 — DHCP failure

> Workstations were getting addresses in the right range and could ping each other, but couldn't reach anything off-subnet and couldn't resolve names. `ipconfig /all` showed a default gateway of 192.168.30.254 — an address with nothing on it. The real gateway is .1.
>
> The pool's `default-router` was wrong. Three of the four delivered values were correct, which is what made it deceptive — DHCP appeared to be working. Fixed the pool, then released and renewed on the clients, because a config fix alone doesn't help clients holding the bad lease.
>
> **The lesson:** check the client first. `show ip dhcp pool` looked healthy throughout. Only `ipconfig /all` exposed it.

### NET-004 — DNS failure

> Users reported the servers were down. I pinged one by IP — instant reply. Pinged the same server by hostname — failed. That single contrast eliminated the entire network path and pointed at name resolution. The client had the right DNS server address and could reach it, so the fault was on the server: the DNS service had been turned off, though all the records were still there.
>
> **The lesson:** when someone says a server is down, ping it by IP first. That one test splits the problem space in half, and communicating "this is DNS, not the network" stops a whole team troubleshooting the wrong layer.

### NET-005 — ACL blocking legitimate traffic

> The NOC lost access to all switch management addresses but could still reach the server VLAN normally. I worked it out almost entirely from ping results: the client reached its own gateway, so the local path was fine. It reached VLAN 10 but not VLAN 20, so routing was working — a routing fault would break both. And a host in VLAN 10 could reach VLAN 20 fine, so the destination was healthy.
>
> That combination means the failure depends on *who is asking*, not on the path — which only happens with filtering. `show access-lists` turned up an unexpected ACL with a climbing match counter, and `show ip interface Vlan30` showed it applied inbound on the operations VLAN. It was meant for the untrusted VLAN — wrong source network and wrong interface.
>
> I removed it, then verified the intended VLAN 40 restriction was still enforced, because fixing an ACL incident by removing the wrong ACL isn't a fix.
>
> **The lesson:** characterise the failure before reaching for tools. Four of those five reasoning steps were pure logic from ping results.

### NET-006 — Disabled switch port

> A server went unreachable and the link light was dark at both ends. That's different from every other incident — the physical link was genuinely down, which eliminates VLAN, IP, routing, DHCP, DNS and ACLs immediately. `show interfaces status` showed the port `disabled`, not `notconnect`, and that distinction is decisive: `notconnect` means the switch sees nothing and you go check the cable; `disabled` means someone ran `shutdown` and it's a CLI fix. `no shutdown`, wait for the port to reach forwarding, then verify.
>
> **The lesson:** check the cheapest, most decisive thing first. And know the difference between `notconnect`, `disabled` and `err-disabled` — err-disabled was shut by the switch itself, so `no shutdown` alone won't hold until you fix the trigger.

### NET-007 — Documentation mismatch

> This one had no symptoms at all, which is the point. During a routine audit I noticed a documented port showing `notconnect` and an undocumented port showing `connected`. The MAC address table confirmed the server had been moved to a different port without any records being updated.
>
> No service impact — but a technician told to shut the documented port to isolate that server would have shut a dead port. Worse, if another server were later patched into it, they'd take down the wrong host during an outage.
>
> **The lesson:** three sources should agree about where a device is — the port map, the port description, and the MAC table. The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation.

---

## Questions you should ask them

Having a couple ready signals genuine interest.

- What does a typical day look like — is it mostly deployments, break-fix, or a mix?
- What's the ticketing and change management process for a port change or a new server deployment?
- How is the physical documentation maintained — DCIM tool, spreadsheets, something custom?
- What's the escalation path when a technician hits something beyond their scope?
- What would you want someone in this role to be independently confident with after ninety days?

---

## Things to be careful about

**Do not overclaim.** If asked whether you've terminated fiber or racked hardware, the answer is no, and that's fine for an entry-level role. Saying "I haven't done that physically, but I understand the concepts and here's what I'd expect" reads far better than a claim that falls apart under one follow-up question.

**Do not oversell the simulation.** Call it a Packet Tracer lab. Interviewers know what Packet Tracer is, and being straight about it makes everything else you say more credible.

**Do not recite.** If you find yourself listing definitions, stop and give an example from the project instead. "A VLAN is a logical broadcast domain" is a textbook line. "I put servers in VLAN 10 and untrusted test devices in VLAN 40 so I could apply an ACL between them" shows you've actually used one.

**Do say what you'd check.** For anything you don't know, "I haven't worked with that, but I'd start by checking X" is a strong answer. It shows method, which is most of what an entry-level technician is hired for.
