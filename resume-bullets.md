# Resume Material — Project 2

> ## DRAFT — USE ONLY AFTER LIVE PACKET TRACER EXECUTION IS COMPLETED AND VERIFIED.

---

**Project title for the resume:**
Data Center Network Deployment, Switching & Cabling Operations Lab — Cisco Packet Tracer

---

## Primary bullets (use 4–5)

> Verbs used are deliberate: configured, simulated, mapped, tested, troubleshot, validated, documented. Nothing claims production, enterprise, or physical hardware experience.

- **Configured** a simulated multi-VLAN data center network in Cisco Packet Tracer across one Layer 3 core switch and two access switches, implementing VLAN segmentation, access ports, 802.1Q trunk links with explicit allowed-VLAN lists, and inter-VLAN routing via switched virtual interfaces.

- **Deployed and validated** network services including a Cisco IOS DHCP server with reserved address exclusions and DNS host records, verifying lease delivery of IP, subnet mask, gateway and DNS through `show ip dhcp binding` and client-side `ipconfig /all`.

- **Troubleshot and resolved** seven simulated network incidents — VLAN misassignment, trunk VLAN pruning, DHCP misconfiguration, DNS service failure, misapplied access control list, administratively disabled switch port, and switch-port documentation mismatch — using `show interfaces status`, `show vlan brief`, `show interfaces trunk`, `show mac address-table`, `show ip route` and `show access-lists`, documenting each in a structured incident report with root cause and validation evidence.

- **Documented** data center physical infrastructure including two rack elevations with rack-unit assignments, an asset inventory, a switch-port map, and a cable map using a two-ended labeling convention mirrored in switch port descriptions, enabling device-to-port verification via MAC address table cross-reference.

- **Authored** a 13-step network troubleshooting runbook and a nine-section new server network deployment checklist covering rack and asset verification, cable labeling, link state, VLAN and IP validation, gateway and DNS testing, MAC-table confirmation, and documentation and ticket closure.

---

## Alternate bullets (swap in as needed)

- **Implemented and tested** a named extended IPv4 access control list restricting an untrusted VLAN from reaching the management VLAN, validating both blocked and permitted traffic against a pre-change and post-change reachability matrix to confirm no unintended impact.

- **Mapped** server-to-switch-port relationships across two simulated racks, cross-referencing NIC MAC addresses against switch MAC address tables to verify physical port assignment independently of written documentation.

- **Simulated** end-to-end 802.1Q trunk validation by placing a test host in the server VLAN on a remote access switch with no default gateway configured, isolating trunk behaviour from routing to prove VLAN traversal across both uplinks.

---

## Skills section additions

```
Networking:     Cisco IOS, VLANs, 802.1Q trunking, inter-VLAN routing, SVIs,
                Layer 3 switching, DHCP, DNS, extended ACLs, MAC address tables
Tools:          Cisco Packet Tracer, Cisco CLI
Data Center:    Rack elevation documentation, asset inventory, switch-port mapping,
                cable mapping and labeling, structured cabling concepts,
                copper/fiber media, SFP/SFP+/QSFP transceiver concepts
Operations:     Network incident response, root cause analysis, troubleshooting
                runbooks, server deployment procedures, change documentation
```

---

## Relationship to Project 1

Project 1 bullets cover Linux administration, SSH, Nginx, users and permissions, storage and RAID, monitoring, Bash and Python automation, and host-side incidents. Project 2 minimizes overlap with Project 1 and focuses on complementary switching, network deployment, rack/port documentation and network troubleshooting skills.

| Project 1 territory | Project 2 territory |
|---|---|
| Linux, SSH, Nginx, permissions | Cisco IOS switching |
| Storage, filesystems, RAID | VLANs, trunks, routing |
| Bash and Python scripting | DHCP, DNS, ACLs |
| Host monitoring | Rack, port and cable documentation |
| Server-side incidents | Network incidents |

---

## Claims deliberately not made

These would be false based on what was actually done, and a single follow-up question would expose them.

| Not claimed | Why |
|---|---|
| Professional or production Cisco engineering experience | This is a Packet Tracer simulation |
| Production network administration | No production network was touched |
| Physical fiber installation, termination or splicing | Not performed |
| Physical rack installation or cable pulling | Not performed |
| Transceiver installation | Not modelled in Packet Tracer |
| Patch panel termination | Not performed |
| Enterprise-scale network design | Nine devices in a simulation |
| CCNA/CCNP-level engineering | Deliberately entry-level scope |

---

## How to describe it if asked directly

> It's a hands-on lab I built in Cisco Packet Tracer to develop data center technician skills — switching, VLANs, trunking, routing, DHCP, DNS, ACLs, plus the rack and cable documentation side. The lab includes seven prepared network fault scenarios that I will execute and validate before using this description. It's a simulation, not production experience, and the physical cabling material in it is documented understanding rather than work I performed.

Leading with the honest framing costs nothing and makes everything else more credible.
