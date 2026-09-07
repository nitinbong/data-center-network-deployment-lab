# Topology Description

## Logical topology

```
                        +---------------------------+
                        |       CORE-SW-01          |
                        |  Cisco 3650-24PS (L3)     |
                        |  RACK-A02 U40             |
                        |  SVIs . DHCP . ACL        |
                        +------+-------------+------+
                        Gi1/0/1              Gi1/0/2
                     802.1Q trunk         802.1Q trunk
                    VLANs 10,20,30,40    VLANs 10,20,30,40
                     native VLAN 99       native VLAN 99
                             |                   |
             +---------------+                   +----------------+
             |                                                    |
      +------+-------+                                    +-------+------+
      | ACCESS-SW-01 |                                    | ACCESS-SW-02 |
      | 2960-24TT    |                                    | 2960-24TT    |
      | RACK-A01 U42 |                                    | RACK-A02 U42 |
      +--+---+---+---+                                    +--+---+---+---+
     Fa0/1 Fa0/2 Fa0/3                              Fa0/5 Fa0/10 Fa0/11 Fa0/20
       |     |     |                                   |     |      |      |
   DC-SRV DC-SRV DC-SRV                            DC-SRV NOC-PC OPS-PC TEST-PC
     -01    -02    -03                             -01 N2   -01    -01    -01
   VLAN10 VLAN10 VLAN10                            VLAN20 VLAN30 VLAN30 VLAN40
   .10.11 .10.12 .10.13                            .20.11  DHCP   DHCP  .40.50
```

## Layers

**Core / Layer 3 — CORE-SW-01.** A Cisco 3650 with `ip routing` enabled. Hosts one SVI per VLAN, which serve as the default gateways. Also runs the IOS DHCP server for VLAN 30 and holds the VLAN 40 access control list. Has no host-facing access ports; its only connections are the two trunk uplinks.

**Access / Layer 2 — ACCESS-SW-01 and ACCESS-SW-02.** Cisco 2960 top-of-rack switches. All endpoints connect here on access ports. Each has a single VLAN 20 management SVI so it is reachable for administration. Neither performs inter-VLAN routing; CORE-SW-01 performs that function.

**Endpoints.** Three servers, three workstations, and one server management NIC.

## Physical distribution

| Rack | Devices |
|---|---|
| RACK-A01 | ACCESS-SW-01 (U42), PP-A01-01 (U41), DC-SERVER-01 (U40), DC-SERVER-02 (U39), DC-SERVER-03 (U38) |
| RACK-A02 | ACCESS-SW-02 (U42), PP-A02-01 (U41), CORE-SW-01 (U40) |
| NOC room | NOC-PC-01, OPS-PC-01, TEST-PC-01 — patched via PP-A02-01 to ACCESS-SW-02 |

## Traffic paths

| Path | Route taken | Routing required |
|---|---|---|
| DC-SERVER-01 to DC-SERVER-02 | ACCESS-SW-01 only | No — same VLAN, same switch |
| NOC-PC-01 to DC-SERVER-01 | ACCESS-SW-02 to CORE-SW-01 (routed VLAN 30 to VLAN 10) to ACCESS-SW-01 | Yes |
| NOC-PC-01 to ACCESS-SW-01 mgmt IP | ACCESS-SW-02 to CORE-SW-01 (routed VLAN 30 to VLAN 20) | Yes |
| TEST-PC-01 to VLAN 20 | Blocked inbound at `interface Vlan40` by ACL `UNTRUSTED-TO-MGMT` | Denied |
| DC-SERVER-01 NIC1 to DC-SERVER-01 NIC2 | ACCESS-SW-01 to CORE-SW-01 (routed VLAN 10 to VLAN 20) to ACCESS-SW-02 | Yes — the two NICs are in different VLANs |

## Cross-switch dependency

Every VLAN 10 host sits on ACCESS-SW-01, and every VLAN 30 and 40 host sits on ACCESS-SW-02. Any communication between a workstation and a server therefore crosses **both trunks and the core**. That dependency is what makes trunk faults (NET-002) immediately visible: intra-switch traffic keeps working while inter-switch traffic stops entirely.

## Topology screenshot

`topology/network-topology.png` — full Packet Tracer canvas showing all nine devices with all links in the green (up) state. Captured from the live simulation.

## Deliberate scope limits

Single uplink per access switch. No spanning-tree tuning, EtherChannel, NIC teaming, or first-hop redundancy (HSRP/VRRP). No A/B redundant paths. These are documented as concepts in `cabling/fiber-copper-transceivers.md` but not implemented — the project targets entry-level data center technician skills, not network engineering design.
