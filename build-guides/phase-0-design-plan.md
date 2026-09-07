# Data Center Network Deployment, Switching & Cabling Operations Lab
## Phase 0 — Design Plan

**Project 2 of 2** · Companion to *Data Center Infrastructure & Linux Operations Home Lab*
**Platform:** Cisco Packet Tracer 8.x
**Career target:** Data Center Technician · Deployment Technician · NOC Technician · Network Technician I

> This is a hands-on simulated lab built in Cisco Packet Tracer. It is not production enterprise experience. No physical cabling, rack installation, or fiber termination work is claimed.

---

## 1. Scope and boundaries

**This project covers** the physical and Layer 2/3 deployment side of data center technician work: switching, VLANs, trunking, inter-VLAN routing, DHCP/DNS, basic ACLs, rack elevation and port documentation, cabling and transceiver concepts, server deployment procedure, and network incident response.

**This project deliberately does not cover** anything already demonstrated in Project 1: Linux administration, SSH server configuration, Nginx, users and permissions, host firewalls, storage/RAID, Bash or Python automation, or any host-side incident.

**Explicitly out of scope** (kept out to stay entry-level appropriate): OSPF/EIGRP/BGP, spanning-tree tuning, EtherChannel, HSRP/VRRP, port security beyond basics, QoS, VXLAN/EVPN, or anything CCNP-tier.

---

## 2. Topology

```
                        +---------------------------+
                        |       CORE-SW-01          |
                        |  Cisco 3650 (Layer 3)     |
                        |  SVIs · DHCP · ACL        |
                        +------+-------------+------+
                        Gi1/0/1              Gi1/0/2
                          (trunk)            (trunk)
                             |                   |
             +---------------+                   +----------------+
             |                                                    |
      +------+-------+                                    +-------+------+
      | ACCESS-SW-01 |                                    | ACCESS-SW-02 |
      | Cisco 2960   |                                    | Cisco 2960   |
      | TOR RACK-A01 |                                    | TOR RACK-A02 |
      +--+---+---+---+                                    +--+---+---+---+
     Fa0/1 Fa0/2 Fa0/3                              Fa0/5 Fa0/10 Fa0/11 Fa0/20
       |     |     |                                   |     |      |      |
   SERVER SERVER SERVER                            SRV-01 NOC-PC OPS-PC TEST-PC
     -01   -02    -03                               mgmt    -01    -01    -01
   VLAN10 VLAN10 VLAN10                            VLAN20 VLAN30 VLAN30 VLAN40
```

### Device inventory

| Hostname | Model | Role | Rack | RU |
|---|---|---|---|---|
| CORE-SW-01 | Cisco 3650-24PS | Layer 3 core — gateways, DHCP, ACL | RACK-A02 | U40 |
| ACCESS-SW-01 | Cisco 2960-24TT | Top-of-rack access switch, compute rack | RACK-A01 | U42 |
| ACCESS-SW-02 | Cisco 2960-24TT | Top-of-rack access switch, core/ops rack | RACK-A02 | U42 |
| DC-SERVER-01 | PT Server | Simulated server + DNS service | RACK-A01 | U40 |
| DC-SERVER-02 | PT Server | Simulated server | RACK-A01 | U39 |
| DC-SERVER-03 | PT Server | Simulated server | RACK-A01 | U38 |
| NOC-PC-01 | PT PC | NOC monitoring workstation | NOC room | — |
| OPS-PC-01 | PT PC | Operations workstation | NOC room | — |
| TEST-PC-01 | PT PC | Untrusted / staging test host | NOC room | — |

---

## 3. Design decisions and rationale

### 3.1 Layer-3 switch instead of router-on-a-stick

`CORE-SW-01` is a Cisco 3650 running `ip routing`, with a switched virtual interface (SVI) per VLAN acting as the default gateway. It also serves DHCP and hosts the ACL.

**Why not router-on-a-stick:** a router with 802.1Q subinterfaces is the classic textbook exercise, but in a real data center the aggregation/core layer is typically a Layer 3 switch. Using the 3650 is both closer to production practice and simpler to build — one device, four SVIs, no subinterface encapsulation to misconfigure. Router-on-a-stick is also less representative of typical production data-center aggregation designs and introduces a single uplink bottleneck, since all inter-VLAN traffic must traverse one physical link twice.

### 3.2 Access switch model and port numbering

Access switches are Cisco 2960-24TT. In Packet Tracer this model provides:

- `FastEthernet0/1` through `FastEthernet0/24` — host-facing access ports
- `GigabitEthernet0/1` and `GigabitEthernet0/2` — uplink ports

The original project outline used `Gi0/1` for server connections. On a real 2960 those Gigabit ports are the uplinks, not host ports. **Servers are therefore documented on `Fa0/x` and trunks on `Gi0/1`.** This matches the actual hardware rather than the sketch, and every document in the repo uses the real port numbering.

### 3.3 Two racks instead of one

Splitting compute (RACK-A01) from core/operations (RACK-A02) creates a genuine inter-rack uplink to document, which is the natural place to discuss fiber, transceivers, and A/B redundancy.

**Important honesty constraint:** the 2960 uplink ports in Packet Tracer are copper. The simulated link is therefore Cat6A. Real data centers may use multimode fiber, single-mode fiber, DAC/AOC, or other media and optics depending on distance, bandwidth and design. The cabling document will label the simulated link as copper, and no fiber run will be claimed as built.

### 3.4 Dedicated server management NIC

`DC-SERVER-01` receives a second NIC (PT-HOST-NM-1CFE module) patched to `ACCESS-SW-02 Fa0/5` on VLAN 20. All three switches also carry a VLAN 20 management SVI/IP.

**Why:** it mirrors real DC practice — production data traffic and management traffic on separate NICs, separate switch ports, and separate VLANs. It also gives incident NET-005 (ACL blocking legitimate management traffic) real traffic to break, instead of a contrived scenario.

**Terminology limit:** this is a *dedicated management NIC on a dedicated management VLAN*, not true out-of-band management. Real OOB management uses a physically separate baseboard controller (iDRAC, iLO, BMC) on an isolated network that stays reachable when the host operating system is down. Nothing in this lab reproduces that. The design demonstrates in-band management separation only, and no document in this repository will claim otherwise.

### 3.5 Where each service lives

| Service | Runs on | Serves |
|---|---|---|
| Inter-VLAN routing | CORE-SW-01 (SVIs) | All VLANs |
| DHCP | CORE-SW-01 (IOS DHCP pool) | VLAN 30 only |
| DNS | DC-SERVER-01 (PT DNS service) | All VLANs |
| ACL | CORE-SW-01, applied to VLAN 40 SVI | VLAN 40 restrictions |

DHCP is scoped to VLAN 30 only. Servers keep static addressing — which is itself the correct DC answer to "why isn't this server on DHCP?"

---

## 4. VLAN plan

| VLAN ID | Name | Subnet | Gateway (SVI) | Purpose |
|---|---|---|---|---|
| 10 | SERVERS | 192.168.10.0/24 | 192.168.10.1 | Simulated server data NICs |
| 20 | MANAGEMENT | 192.168.20.0/24 | 192.168.20.1 | Switch management IPs, dedicated server management NIC |
| 30 | OPERATIONS | 192.168.30.0/24 | 192.168.30.1 | NOC and operations workstations (DHCP) |
| 40 | UNTRUSTED | 192.168.40.0/24 | 192.168.40.1 | Test / unvetted devices, ACL-restricted |
| 99 | NATIVE | — | — | Native VLAN on trunks (unused for hosts) |

**VLAN 99 note:** setting an unused VLAN as native on trunks is a standard hygiene practice — it avoids untagged traffic landing in VLAN 1. This will be configured but not dwelt on; it is one line per trunk.

---

## 5. IP address plan

| Device | Interface | VLAN | IP address | Mask | Gateway | Method |
|---|---|---|---|---|---|---|
| CORE-SW-01 | SVI VLAN 10 | 10 | 192.168.10.1 | /24 | — | Static |
| CORE-SW-01 | SVI VLAN 20 | 20 | 192.168.20.1 | /24 | — | Static |
| CORE-SW-01 | SVI VLAN 30 | 30 | 192.168.30.1 | /24 | — | Static |
| CORE-SW-01 | SVI VLAN 40 | 40 | 192.168.40.1 | /24 | — | Static |
| ACCESS-SW-01 | SVI VLAN 20 | 20 | 192.168.20.2 | /24 | 192.168.20.1 | Static |
| ACCESS-SW-02 | SVI VLAN 20 | 20 | 192.168.20.3 | /24 | 192.168.20.1 | Static |
| DC-SERVER-01 | NIC1 (data) | 10 | 192.168.10.11 | /24 | 192.168.10.1 | Static |
| DC-SERVER-01 | NIC2 (mgmt) | 20 | 192.168.20.11 | /24 | 192.168.20.1 | Static |
| DC-SERVER-02 | NIC1 | 10 | 192.168.10.12 | /24 | 192.168.10.1 | Static |
| DC-SERVER-03 | NIC1 | 10 | 192.168.10.13 | /24 | 192.168.10.1 | Static |
| NOC-PC-01 | NIC | 30 | 192.168.30.101–150 | /24 | 192.168.30.1 | DHCP |
| OPS-PC-01 | NIC | 30 | 192.168.30.101–150 | /24 | 192.168.30.1 | DHCP |
| TEST-PC-01 | NIC | 40 | 192.168.40.50 | /24 | 192.168.40.1 | Static |

### Address reservation ranges

| Range | Use |
|---|---|
| .1 | Gateway (SVI) — always |
| .2 – .9 | Network infrastructure (switch management IPs) |
| .10 – .99 | Statically assigned servers |
| .100 – .199 | DHCP pool |
| .200 – .254 | Reserved / future |

### DHCP scope (VLAN 30)

| Parameter | Value |
|---|---|
| Pool name | OPS-POOL |
| Network | 192.168.30.0 /24 |
| Excluded | 192.168.30.1 – 192.168.30.100 |
| Usable range | 192.168.30.101 – 192.168.30.254 |
| Default gateway | 192.168.30.1 |
| DNS server | 192.168.10.11 |

### DNS records (DC-SERVER-01)

| Hostname | Type | Address |
|---|---|---|
| dc-server-01.dclab.local | A | 192.168.10.11 |
| dc-server-02.dclab.local | A | 192.168.10.12 |
| dc-server-03.dclab.local | A | 192.168.10.13 |
| core-sw-01.dclab.local | A | 192.168.20.1 |

---

## 6. Rack elevations

### RACK-A01 — Compute

| RU | Equipment | Asset ID | Notes |
|---|---|---|---|
| U42 | ACCESS-SW-01 | SW-A01-001 | Top-of-rack access switch |
| U41 | PP-A01-01 | PP-A01-001 | 24-port copper patch panel |
| U40 | DC-SERVER-01 | SRV-A01-001 | Simulated server, DNS service |
| U39 | DC-SERVER-02 | SRV-A01-002 | Simulated server |
| U38 | DC-SERVER-03 | SRV-A01-003 | Simulated server |
| U37 – U01 | (empty) | — | Reserved / blanking panels |

### RACK-A02 — Core and operations

| RU | Equipment | Asset ID | Notes |
|---|---|---|---|
| U42 | ACCESS-SW-02 | SW-A02-001 | Top-of-rack access switch |
| U41 | PP-A02-01 | PP-A02-001 | 24-port copper patch panel |
| U40 | CORE-SW-01 | SW-A02-002 | Layer 3 core switch |
| U39 – U01 | (empty) | — | Reserved / blanking panels |

**Workstations** (NOC-PC-01, OPS-PC-01, TEST-PC-01) are located in the NOC room and patched back to RACK-A02 through patch panel PP-A02-01. They occupy no rack units.

**Convention note:** rack units are numbered bottom-up (U01 at the floor, U42 at the top). Placing the switch at the top of the rack is standard top-of-rack design: it keeps server-to-switch patch cords short, keeps cable management clean and contained within the rack, and makes server-to-switch-port mapping easier to trace visually.

---

## 7. Switch-port map

| Device | Interface | Switch | Port | Mode | VLAN | IP | Cable | Purpose |
|---|---|---|---|---|---|---|---|---|
| DC-SERVER-01 | NIC1 | ACCESS-SW-01 | Fa0/1 | access | 10 | 192.168.10.11 | Cat6 | Server data |
| DC-SERVER-02 | NIC1 | ACCESS-SW-01 | Fa0/2 | access | 10 | 192.168.10.12 | Cat6 | Server data |
| DC-SERVER-03 | NIC1 | ACCESS-SW-01 | Fa0/3 | access | 10 | 192.168.10.13 | Cat6 | Server data |
| DC-SERVER-01 | NIC2 (mgmt) | ACCESS-SW-02 | Fa0/5 | access | 20 | 192.168.20.11 | Cat6 | Dedicated server management NIC |
| NOC-PC-01 | NIC | ACCESS-SW-02 | Fa0/10 | access | 30 | DHCP | Cat6 | NOC workstation |
| OPS-PC-01 | NIC | ACCESS-SW-02 | Fa0/11 | access | 30 | DHCP | Cat6 | Operations workstation |
| TEST-PC-01 | NIC | ACCESS-SW-02 | Fa0/20 | access | 40 | 192.168.40.50 | Cat6 | Untrusted test host |
| ACCESS-SW-01 | Gi0/1 | CORE-SW-01 | Gi1/0/1 | trunk | 10,20,30,40 | — | Cat6A * | Uplink RACK-A01 → core |
| ACCESS-SW-02 | Gi0/1 | CORE-SW-01 | Gi1/0/2 | trunk | 10,20,30,40 | — | Cat6A * | Uplink RACK-A02 → core |

\* Simulated as copper in Packet Tracer. Real data centers may use multimode fiber, single-mode fiber, DAC/AOC, or other media and optics depending on distance, bandwidth and design. See the cabling document.

### Cable labeling convention

Format: `<SOURCE-RACK>-<SOURCE-DEVICE>-<PORT>__<DEST-RACK>-<DEST-DEVICE>-<PORT>`

Example: `A01-SRV001-NIC1__A01-SW001-Fa0-1`

Both ends of every cable carry the same label. This is what makes cable tracing possible without a toner.

---

## 8. ACL policy

Kept deliberately minimal — one named extended ACL, one application point.

| Rule | Source | Destination | Action | Reason |
|---|---|---|---|---|
| 1 | 192.168.40.0/24 | 192.168.20.0/24 | deny | Untrusted hosts must never reach management infrastructure |
| 2 | any | any | permit | Everything else unrestricted |

**Applied:** inbound on `interface Vlan40` on CORE-SW-01.

**Not restricted:** VLAN 30 (Operations) → VLAN 10 (Servers) is permitted with no ACL at all. That is a required business flow.

**Incident hook (NET-005):** the baseline ACL above is applied inbound on VLAN 40, so it can only ever filter traffic *sourced from* VLAN 40. It cannot affect VLAN 30 → VLAN 20 traffic, and the incident will not pretend otherwise.

Instead, NET-005 simulates a change-management error: a **second, temporary ACL is deliberately created and applied inbound on `interface Vlan30`**, as if a technician had applied an untrusted-host restriction to the wrong interface. That ACL denies VLAN 30 → VLAN 20 traffic and permits everything else. The observable result is that NOC-PC-01 loses access to switch management addresses while VLAN 30 → VLAN 10 server traffic keeps working normally. After troubleshooting, the misapplied ACL is removed from VLAN 30 and the interface returned to the baseline state.

That asymmetry — one destination unreachable while routing and every other destination is fine — is the fingerprint of a filtering problem rather than a routing or Layer 2 problem.

---

## 9. Incident catalogue

Six incidents, none overlapping Project 1. Each is broken in Packet Tracer, observed live, then fixed and re-verified. **Outputs will not be fabricated** — command output goes into the report only after it has actually been run.

| ID | Title | Injected fault | Primary diagnostic commands |
|---|---|---|---|
| NET-001 | Wrong VLAN | DC-SERVER-03 port set to VLAN 30 instead of 10 | `show vlan brief`, `show interfaces status`, `show mac address-table` |
| NET-002 | Trunk failure | VLAN 30 pruned from ACCESS-SW-02 uplink | `show interfaces trunk`, `show vlan brief` |
| NET-003 | DHCP failure | DHCP pool gateway/exclusion misconfigured | `show ip dhcp pool`, `show ip dhcp binding`, `show run \| section dhcp` |
| NET-004 | DNS failure | DNS service stopped / record removed on DC-SERVER-01 | `ping` by IP vs by name, `ipconfig /all`, `nslookup` |
| NET-005 | ACL block | Temporary ACL misapplied inbound on VLAN 30, blocking VLAN 30 → VLAN 20 | `show access-lists`, `show ip interface Vlan30`, `show run interface Vlan30` |
| NET-006 | Disabled switch port | `shutdown` applied to DC-SERVER-02's access port | `show interfaces status`, `show interface Fa0/2`, `show running-config` |
| NET-007 *(optional)* | Documentation mismatch | Server moved to a different port, docs not updated | `show mac address-table`, port map cross-check |

### Report template (each incident)

```
Incident ID
Priority
Affected Device / Switch / Port / VLAN
Issue
Symptoms
Investigation
Commands & Checks Used
Root Cause
Resolution
Validation
Preventive Action
```

---

## 10. Troubleshooting order (used in every incident)

1. Check physical / link state
2. Verify switch port (correct port? admin up?)
3. Verify VLAN assignment
4. Verify IP address and subnet mask
5. Verify default gateway
6. Verify trunk (is the VLAN allowed end to end?)
7. Verify routing (SVI up? `ip routing` enabled?)
8. Verify DHCP / DNS
9. Verify ACL
10. Test connectivity
11. Document the result

The order is bottom-up on purpose. It is faster to rule out a dead port in ten seconds than to spend twenty minutes on a routing table for a cable that was never plugged in.

---

## 11. Build order

| Phase | Work | Output |
|---|---|---|
| 1 | Place devices, cable topology, set hostnames and port descriptions | `.pkt` file, topology screenshot |
| 2 | Create VLANs, assign access ports | `show vlan brief`, `show interfaces status`, `show mac address-table` screenshots |
| 3 | Configure trunks with native VLAN 99 and allowed list | `show interfaces trunk` screenshot |
| 4 | Enable `ip routing`, create SVIs, test inter-VLAN pings | Ping success screenshots |
| 5 | Configure IOS DHCP pool for VLAN 30 | DHCP lease screenshot, `show ip dhcp binding` |
| 6 | Configure DNS service and records on DC-SERVER-01 | Successful `nslookup` screenshot |
| 7 | Build and apply the VLAN 40 ACL | ACL permit/deny test screenshots |
| 8 | Capture full baseline, then run NET-001 → NET-006 | Before/after screenshots per incident |
| 9 | Write all documentation, README, resume bullets, interview prep | Complete repo |

Phases 1–7 build a known-good baseline. **The baseline must be captured and saved before any incident is injected** — otherwise there is nothing to compare against, and "after the fix" screenshots prove nothing.

---

## 12. Repository structure

```
data-center-network-deployment-lab/
├── README.md
├── packet-tracer/
│   └── data-center-network.pkt
├── topology/
│   ├── network-topology.png
│   └── topology-description.md
├── rack-design/
│   ├── rack-elevation.md
│   └── asset-inventory.csv
├── cabling/
│   ├── cable-map.csv
│   ├── switch-port-map.csv
│   └── fiber-copper-transceivers.md
├── network-design/
│   ├── vlan-plan.md
│   └── ip-address-plan.md
├── configurations/
│   ├── core-switch.txt
│   ├── access-switch-01.txt
│   └── access-switch-02.txt
├── deployment/
│   └── new-server-deployment-checklist.md
├── incidents/
│   ├── NET-001-wrong-vlan.md
│   ├── NET-002-trunk-failure.md
│   ├── NET-003-dhcp-failure.md
│   ├── NET-004-dns-failure.md
│   ├── NET-005-acl-block.md
│   └── NET-006-disabled-port.md
├── runbooks/
│   └── network-troubleshooting-runbook.md
└── screenshots/
```

### Screenshot naming convention

```
01-topology-complete.png
02-vlan-brief-access-sw-01.png
03-interfaces-status-access-sw-01.png
04-mac-address-table.png
05-trunk-status-core.png
06-intervlan-ping-noc-to-server.png
07-dhcp-lease-ops-pc.png
08-dns-nslookup-success.png
09-acl-test-blocked.png
NET-001-before.png
NET-001-after.png
...
```

---

## 13. Open items to confirm before Phase 1

1. **Layer-3 switch vs router-on-a-stick** — confirm the 3650 approach described in §3.1.
2. **Packet Tracer version** — 8.x is assumed. The 3650 model is needed for SVIs and IOS DHCP; a 3560 also works if that is what is available.
3. **Two racks vs one** — confirm the RACK-A01 / RACK-A02 split described in §3.3.

Once confirmed, Phase 1 begins with the exact device list, cable runs, and the first block of switch configuration commands.
