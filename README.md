# Data Center Network Deployment, Switching & Cabling Operations Lab

A hands-on simulated data center network built in Cisco Packet Tracer, covering Layer 2 switching, VLAN design, 802.1Q trunking, inter-VLAN routing, DHCP, DNS, access control lists, rack and switch-port documentation, server deployment procedure, and structured network incident response.

**Project design and implementation package prepared for Cisco Packet Tracer. Live execution and evidence capture pending.**

> ### Scope disclaimer
>
> **This is a hands-on simulated lab, not production enterprise experience.**
>
> All network devices are simulated in Cisco Packet Tracer. No physical cabling, fiber termination, rack installation, or transceiver work was performed. Sections covering physical media, transceivers, patch panels and A/B redundancy demonstrate **conceptual understanding only** and are labelled as such throughout.

---

## Why this project

This project targets the physical and network deployment side of data center technician work — the day-to-day tasks of connecting servers to switches, assigning VLANs, verifying ports, documenting racks and cable runs, and troubleshooting connectivity when something breaks.

**Roles this maps to:** Data Center Technician · Data Center Operations Technician · Deployment Technician · Server Technician · Infrastructure Technician · NOC Technician · Network Technician I

**Relationship to Project 1.** This is the second of two portfolio projects. Project 1 (*Data Center Infrastructure & Linux Operations Home Lab*) covered the host and operating system side: Linux administration, SSH, service troubleshooting, permissions, storage and RAID, monitoring, and automation. Project 2 minimizes overlap with Project 1 and focuses on complementary switching, network deployment, rack/port documentation and network troubleshooting skills.

---

## Network to be built

A three-switch network with four VLANs, routed at a Layer 3 core, serving six endpoints across two simulated racks.

```
                        +---------------------------+
                        |       CORE-SW-01          |
                        |  Cisco 3650-24PS (L3)     |
                        |  SVIs . DHCP . ACL        |
                        +------+-------------+------+
                        Gi1/0/1              Gi1/0/2
                     802.1Q trunk         802.1Q trunk
                             |                   |
      +----------------------+                   +----------------------+
      | ACCESS-SW-01 (2960)  |                   | ACCESS-SW-02 (2960)  |
      | RACK-A01 U42         |                   | RACK-A02 U42         |
      +--+---+---+-----------+                   +--+---+---+---+-------+
     Fa0/1 Fa0/2 Fa0/3                      Fa0/5 Fa0/10 Fa0/11 Fa0/20
       |     |     |                           |     |      |      |
   DC-SRV DC-SRV DC-SRV                    DC-SRV NOC-PC OPS-PC TEST-PC
     -01    -02    -03                     -01 N2   -01    -01    -01
   VLAN10 VLAN10 VLAN10                    VLAN20 VLAN30 VLAN30 VLAN40
```

### VLAN design

| VLAN | Name | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| 10 | SERVERS | 192.168.10.0/24 | 192.168.10.1 | Simulated server data NICs |
| 20 | MANAGEMENT | 192.168.20.0/24 | 192.168.20.1 | Switch management IPs, dedicated server management NIC |
| 30 | OPERATIONS | 192.168.30.0/24 | 192.168.30.1 | NOC and operations workstations (DHCP) |
| 40 | UNTRUSTED | 192.168.40.0/24 | 192.168.40.1 | Test hosts, ACL-restricted |
| 99 | NATIVE | — | — | Native VLAN on trunks, no hosts |

---

## Technologies

| Area | Detail |
|---|---|
| Simulation platform | Cisco Packet Tracer 8.x |
| Core switch | Cisco Catalyst 3650-24PS (Layer 3) |
| Access switches | Cisco Catalyst 2960-24TT (Layer 2) |
| Switching | VLANs, access ports, 802.1Q trunking, native VLAN, MAC address tables |
| Routing | Layer 3 switching via SVIs, connected routes |
| Services | Cisco IOS DHCP server, Packet Tracer DNS service |
| Security | Named extended IPv4 access control lists |
| Documentation | Markdown, CSV asset and cable registers |

---

## Implementation scope

### Switching and VLANs

The lab is designed to configure five VLANs across three switches with descriptive naming. Every host port is assigned as an explicit access port with `switchport mode access`, rather than relying on dynamic negotiation. Port descriptions embed the cable label on every configured interface, so `show interfaces description` alone identifies what is connected to each port.

Validation will use `show vlan brief`, `show interfaces status` and `show mac address-table` — including cross-checking learned MAC addresses against device NIC addresses to confirm which physical device sits on which physical port.

### Trunking

The implementation configures 802.1Q trunk links between each access switch and the core, with VLAN 99 as native and an explicit allowed VLAN list rather than the permissive default.

The planned validation uses a host temporarily placed on the opposite access switch in VLAN 10, with **no default gateway configured**, so a successful ping can only be explained by both trunks carrying the VLAN. VLAN 10 is then deliberately pruned from one trunk to confirm the ping fails while same-switch traffic continues working — the defining signature of a trunk fault.

### Inter-VLAN routing

The implementation enables `ip routing` on the core and configures one SVI per VLAN as its default gateway. Live execution will verify four connected routes and confirm routing with `tracert`, showing the SVI as the first hop.

Each access switch is given a single VLAN 20 management SVI and `ip default-gateway`, making both switches reachable for administration. The access switches do not perform inter-VLAN routing; CORE-SW-01 performs that function.

### DHCP

The implementation includes an IOS DHCP pool serving VLAN 30, with an excluded range protecting the gateway and static/infrastructure addresses. Live execution will verify leases through `show ip dhcp pool`, `show ip dhcp binding` and client-side `ipconfig /all`, confirming all four delivered values: address, mask, gateway and DNS.

### DNS

The implementation includes Packet Tracer DNS on DC-SERVER-01 with A records for all servers and the core switch management address. Live execution will verify resolution from DHCP clients across VLAN boundaries. The diagnostic distinction between IP connectivity failure and name resolution failure is documented.

### Access control

The baseline design applies a named extended ACL denying the untrusted VLAN access to the management VLAN, permitting everything else. Validation covers **both halves** — that the intended traffic is blocked, and that all other traffic still passes — using a pre-change and post-change reachability matrix to confirm nothing is blocked unintentionally.

### Physical and deployment documentation

Produced rack elevations for two racks with rack unit assignments, an asset inventory, a switch-port map, a cable map with a consistent two-ended labeling convention, VLAN and IP address plans, and a reachability baseline.

Wrote a nine-section new server network deployment checklist covering rack ID, rack unit, asset ID, NIC, switch, port, cable label, link state, VLAN, IP, mask, gateway, DNS, gateway ping, required connectivity testing, MAC table verification, documentation updates and ticket closure.

### Incident response

Seven network incidents are prepared for live injection, diagnosis, resolution and validation in Packet Tracer. Each has a structured report covering symptoms, investigation steps, commands used, root cause, resolution, validation table and preventive actions.

| ID | Incident | Fault | Key diagnostic |
|---|---|---|---|
| [NET-001](incidents/NET-001-wrong-vlan.md) | Wrong VLAN | Access port in VLAN 30 instead of VLAN 10 | `show interfaces status` |
| [NET-002](incidents/NET-002-trunk-failure.md) | Trunk failure | VLAN pruned from allowed list | `show interfaces trunk` |
| [NET-003](incidents/NET-003-dhcp-failure.md) | DHCP failure | Wrong `default-router` in the pool | `ipconfig /all` |
| [NET-004](incidents/NET-004-dns-failure.md) | DNS failure | DNS service disabled | Ping by IP vs by name |
| [NET-005](incidents/NET-005-acl-block.md) | ACL block | Temporary ACL applied to the wrong interface | `show ip interface Vlan30` |
| [NET-006](incidents/NET-006-disabled-port.md) | Disabled port | Port administratively shut down | `show interfaces status` |
| [NET-007](incidents/NET-007-documentation-mismatch.md) | Documentation mismatch | Server moved without updating records | `show mac address-table` |

### Troubleshooting methodology

Built a [13-step troubleshooting runbook](runbooks/network-troubleshooting-runbook.md) working bottom-up from the physical layer: link state, port identification, administrative state, VLAN, IP and mask, gateway, trunk, routing, DHCP, DNS, ACL, retest, document.

Each step explains what to check, why it matters, what a good and bad result look like, and where to go next. The runbook includes a fault-characterisation table that narrows most problems to a category before a single `show` command is run.

---

## Skills targeted / demonstrated after execution

**Cisco switching** — VLAN creation and naming, access port configuration, port descriptions, `interface range`, MAC address table interpretation, switch management SVIs

**802.1Q trunking** — trunk configuration, native VLAN, allowed VLAN lists, trunk verification, diagnosing VLANs missing from trunks

**Layer 3 switching** — `ip routing`, SVI configuration, connected routes, default gateway behaviour, `ip default-gateway` on Layer 2 switches

**Network services** — IOS DHCP pools with exclusions, DORA process, DNS records, lease verification and renewal

**Access control** — named extended ACLs, wildcard masks, interface application and direction, match counters, validating both permit and deny behaviour

**Physical infrastructure documentation** — rack elevations, asset inventory, switch-port mapping, cable mapping and labeling conventions

**Cabling and media concepts** — Cat6/Cat6A, single-mode and multimode fiber, LC connectors, SFP/SFP+/QSFP transceivers, TX/RX, link LED interpretation, patch panels, cable tracing methods, A/B redundancy *(conceptual)*

**Deployment procedure** — structured server network deployment checklist with verification at every step

**Incident response** — structured diagnosis, root cause analysis, validated resolution, preventive action, professional incident reporting

---

## Lessons learned

**A green link light proves Layer 1 and nothing else.** Five of the seven incident scenarios in this project are built around a perfectly healthy physical link with a logical fault above it. The most common data center network problem is not a bad cable — it is a working cable in a working port that has been placed in the wrong logical network.

**Characterising a failure often names its cause.** Comparing a same-switch ping against a cross-switch ping identifies a trunk fault before any command is run. Comparing ping-by-IP against ping-by-name separates DNS from the network. Comparing which source VLANs can reach a destination isolates an ACL from a routing problem. Two minutes of structured testing beats twenty minutes of unstructured command output.

**The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation.** Documentation records intentions and can be stale, and port descriptions can be wrong, so the live table is what confirms where a device is actually attached.

**Validation means testing both halves of a change.** An ACL that blocks the intended traffic but also blocks something else is not a working ACL. Fixing an incident by removing the wrong rule is not a fix. Every change in this project is designed to be validated against a known-good reachability baseline, not just against the symptom that was reported.

**Documentation drift is invisible until it costs you.** NET-007 had zero service impact and would never have generated an alert — but a technician acting on that documentation during a real outage would have shut down the wrong port. Updating records in the same change window as the work is the only version of documentation discipline that actually holds.

---

## Repository structure

```
data-center-network-deployment-lab/
├── README.md                          This file
├── PROJECT-STATUS.md                  Honest completion and limitation record
├── interview-prep.md                  Interview question preparation
├── resume-bullets.md                  Truthful resume material
├── MANUAL-EXECUTION-CHECKLIST.md      Steps requiring live Packet Tracer work
│
├── build-guides/                      Phase-by-phase build instructions
│   ├── phase-0-design-plan.md
│   ├── phase-1-topology-and-cabling.md
│   ├── phase-2-3-vlans-and-trunks.md
│   ├── phase-4-svis-and-routing.md
│   └── phase-5-6-7-dhcp-dns-acl.md
│
├── packet-tracer/                     Simulation files
│   ├── data-center-network.pkt
│   └── data-center-network-baseline.pkt
│
├── topology/
│   ├── network-topology.png
│   └── topology-description.md
│
├── rack-design/
│   ├── rack-elevation.md
│   └── asset-inventory.csv
│
├── cabling/
│   ├── cable-map.csv
│   ├── switch-port-map.csv
│   └── fiber-copper-transceivers.md
│
├── network-design/
│   ├── vlan-plan.md
│   ├── ip-address-plan.md
│   └── reachability-baseline.md
│
├── configurations/
│   ├── core-switch.txt
│   ├── access-switch-01.txt
│   └── access-switch-02.txt
│
├── deployment/
│   └── new-server-deployment-checklist.md
│
├── incidents/
│   ├── NET-001-wrong-vlan.md
│   ├── NET-002-trunk-failure.md
│   ├── NET-003-dhcp-failure.md
│   ├── NET-004-dns-failure.md
│   ├── NET-005-acl-block.md
│   ├── NET-006-disabled-port.md
│   └── NET-007-documentation-mismatch.md
│
├── runbooks/
│   └── network-troubleshooting-runbook.md
│
└── screenshots/
    └── README.md                      Screenshot manifest and capture standards
```

---

## Technical detail

### Addressing convention

The third octet matches the VLAN ID — VLAN 10 is 192.168.10.0/24, VLAN 20 is 192.168.20.0/24. Reading an IP address immediately tells you which VLAN it belongs to.

Within every subnet: `.1` is the gateway, `.2`–`.9` is network infrastructure, `.10`–`.99` is static hosts, `.100`–`.199` is the DHCP pool, `.200`–`.254` is reserved.

Full detail in [`network-design/ip-address-plan.md`](network-design/ip-address-plan.md).

### Cable labeling convention

```
<SOURCE-RACK>-<SOURCE-DEVICE>-<PORT>__<DEST-RACK>-<DEST-DEVICE>-<PORT>
```

Example: `A01-SRV001-NIC1__A01-SW001-Fa0-1`

The same label appears on both ends of the cable, in `cable-map.csv`, in `switch-port-map.csv`, and in the switch port description. Four sources that must agree.

### Design decisions

**Layer 3 switch rather than router-on-a-stick.** A router with 802.1Q subinterfaces is the classic textbook exercise, but data center aggregation is typically done with a Layer 3 switch. It is also simpler to build and avoids forcing all inter-VLAN traffic through a single physical link twice.

**Two racks rather than one.** Splitting compute from core creates a genuine inter-rack uplink to document, which is the natural place to discuss fiber, transceivers and redundancy.

**Explicit allowed VLAN lists on trunks** rather than the permissive 1–4094 default, limiting broadcast propagation.

**VLAN 99 as native**, so stray untagged traffic does not land in VLAN 1 alongside unconfigured ports.

### Deliberate scope limits

No spanning-tree tuning, EtherChannel, NIC teaming, HSRP/VRRP, port security, QoS, or dynamic routing protocols. Single uplink per access switch, no A/B redundant paths. These are documented as concepts where relevant but not implemented — the project targets entry-level data center technician skills, not network engineering design.

---

## Honest limitations

See [`PROJECT-STATUS.md`](PROJECT-STATUS.md) for a complete record of what was implemented, what is simulated, what is documented-only, and what the Packet Tracer environment could not support.

Summary of the main boundaries:

- All networking is simulated in Packet Tracer. No physical hardware was configured.
- No physical cabling, fiber termination, transceiver installation or rack mounting was performed.
- VLAN 20 is a dedicated management VLAN, not true out-of-band management. Real OOB uses a separate baseboard controller (iDRAC, iLO, BMC) on an isolated network.
- The inter-rack uplink is simulated as Cat6A copper because Packet Tracer's 2960 uplink ports are copper. Real data centers may use multimode fiber, single-mode fiber, DAC/AOC, or other media and optics depending on distance, bandwidth and design.
- A/B network redundancy is documented as a concept and is not implemented.
