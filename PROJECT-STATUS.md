# Project Status

An honest record of what was implemented, what is simulated, what is documented only, and what could not be done in this environment.

**Purpose:** this file exists so nothing in the repository overstates what was actually accomplished. If a recruiter, hiring manager or interviewer wants to know exactly what was hands-on versus conceptual, this is the answer.

---

## Legend

| Status | Meaning |
|---|---|
| **Prepared — Packet Tracer execution/evidence pending** | Designed and documented in full. Requires live Cisco Packet Tracer execution and evidence capture |
| **Simulated environment** | Targets a simulation rather than physical hardware |
| **Documented only** | Written up as conceptual knowledge. Not built, not performed |
| **Not implemented** | Deliberately out of scope |
| **Evidence pending** | Documentation complete; live Packet Tracer capture still required |

---

## Network configuration

| Item | Status | Notes |
|---|---|---|
| Topology — 3 switches, 6 endpoints | Prepared — Packet Tracer execution/evidence pending | Cisco Packet Tracer 8.x |
| Device hostnames | Prepared — Packet Tracer execution/evidence pending | All three switches |
| Port descriptions with cable labels | Prepared — Packet Tracer execution/evidence pending | Every configured interface |
| VLAN creation (10, 20, 30, 40, 99) | Prepared — Packet Tracer execution/evidence pending | All three switches |
| Access port assignment | Prepared — Packet Tracer execution/evidence pending | All seven host ports |
| 802.1Q trunk configuration | Prepared — Packet Tracer execution/evidence pending | Both uplinks, native VLAN 99, explicit allowed lists |
| Trunk end-to-end validation | Prepared — Packet Tracer execution/evidence pending | Test designed to use a host on the opposite switch with no gateway configured |
| `ip routing` on core | Prepared — Packet Tracer execution/evidence pending | CORE-SW-01 |
| SVIs for all four VLANs | Prepared — Packet Tracer execution/evidence pending | CORE-SW-01 |
| Management SVI on access switches | Prepared — Packet Tracer execution/evidence pending | VLAN 20, one per access switch |
| `ip default-gateway` on access switches | Prepared — Packet Tracer execution/evidence pending | Both |
| IOS DHCP pool for VLAN 30 | Prepared — Packet Tracer execution/evidence pending | With excluded range |
| DHCP lease verification | Prepared — Packet Tracer execution/evidence pending | Pool, bindings and client-side `ipconfig /all` |
| DNS service and A records | Prepared — Packet Tracer execution/evidence pending | Packet Tracer DNS service on DC-SERVER-01 |
| Named extended ACL | Prepared — Packet Tracer execution/evidence pending | `UNTRUSTED-TO-MGMT` inbound on Vlan40 |
| ACL bidirectional validation | Prepared — Packet Tracer execution/evidence pending | Blocked and permitted traffic validation prepared |
| Reachability matrix, pre and post ACL | Prepared — Packet Tracer execution/evidence pending | `network-design/reachability-baseline.md` |

---

## Incidents

All seven are fully designed and documented, with fault-injection commands, diagnostic sequences, root-cause analysis and validation tables prepared. Each still requires live Packet Tracer execution and evidence capture.

| Incident | Status | Fault type |
|---|---|---|
| NET-001 Wrong VLAN | Prepared — Packet Tracer execution/evidence pending | Config change on ACCESS-SW-01 Fa0/3 |
| NET-002 Trunk failure | Prepared — Packet Tracer execution/evidence pending | Allowed-VLAN-list change on ACCESS-SW-01 Gi0/1 |
| NET-003 DHCP failure | Prepared — Packet Tracer execution/evidence pending | Pool misconfiguration on CORE-SW-01 |
| NET-004 DNS failure | Prepared — Packet Tracer execution/evidence pending | DNS service disabled on DC-SERVER-01 |
| NET-005 ACL block | Prepared — Packet Tracer execution/evidence pending | Second ACL applied inbound on Vlan30 |
| NET-006 Disabled port | Prepared — Packet Tracer execution/evidence pending | `shutdown` on ACCESS-SW-01 Fa0/2 |
| NET-007 Documentation mismatch (optional) | Prepared — Packet Tracer execution/evidence pending | Cable relocated without updating records |

**Note on NET-005.** The baseline VLAN 40 ACL is applied inbound on `interface Vlan40` and is architecturally incapable of blocking VLAN 30 traffic. The incident therefore models a change-management error — a separate temporary ACL applied to the wrong interface — rather than pretending the baseline ACL caused a fault it could not cause.

---

## Documentation

| Item | Status |
|---|---|
| Rack elevations, two racks | Complete (documentation exercise) |
| Asset inventory CSV | Complete |
| Switch-port map CSV | Complete |
| Cable map CSV | Complete |
| VLAN plan | Complete |
| IP address plan | Complete |
| Reachability baseline | Complete |
| New server deployment checklist | Complete |
| Troubleshooting runbook | Complete |
| Seven incident reports | Complete |
| Cabling and transceiver concepts | **Documented only** |
| Interview preparation | Complete |

---

## Physical work — explicitly not performed

**None of the following was done. No claim is made that it was.**

| Item | Status | Why |
|---|---|---|
| Physical rack mounting | Not implemented | No physical hardware |
| Physical cable installation | Not implemented | No physical hardware |
| Fiber termination, splicing or polishing | Not implemented | No physical hardware |
| Transceiver (SFP/SFP+/QSFP) installation | Not implemented | Not modelled in Packet Tracer |
| Patch panel punch-down | Not implemented | No physical hardware |
| Cable tracing with toner and probe | Not implemented | No physical hardware |
| Physical label application | Not implemented | Labeling convention documented only |
| Optical power measurement | Not implemented | Not modelled in Packet Tracer |
| Airflow, blanking panel, PDU work | Not implemented | Documented as design intent only |

The content in `cabling/fiber-copper-transceivers.md` demonstrates **conceptual understanding**. It is not a record of work performed.

---

## Environment limitations

Constraints imposed by Cisco Packet Tracer that shaped the design.

| Limitation | Impact | How it was handled |
|---|---|---|
| 2960 uplink ports are copper only | Inter-rack trunk cannot be fiber | Simulated as Cat6A. Real data centers may use multimode fiber, single-mode fiber, DAC/AOC, or other media and optics depending on distance, bandwidth and design. |
| No transceiver modelling | Cannot demonstrate optics, `show interfaces transceiver`, or optical power | Documented conceptually and labelled as such |
| 2960 host ports are FastEthernet | Server ports are `Fa0/x`, not `Gi0/x` as in the original outline | Real port numbering used throughout. Documented in the design plan |
| `nslookup` support varies by version | May be unavailable | Hostname ping used as the resolution test. Noted where the command could not be run |
| No BMC/iDRAC/iLO simulation | True out-of-band management not possible | VLAN 20 documented as a *dedicated management VLAN*, explicitly not OOB |
| Simplified IOS feature set | Some production commands unavailable or behave differently | Scope kept to features that genuinely work in Packet Tracer |
| No physical layer error injection | Cannot simulate CRC errors, dirty optics, or marginal cable runs | Discussed conceptually in the runbook, not demonstrated |

---

## Deliberately out of scope

Excluded to keep the project at entry-level data center technician depth rather than drifting into network engineering.

| Topic | Reason |
|---|---|
| Spanning-tree tuning, root bridge election, PortFast/BPDU guard | CCNA/CCNP depth, beyond the target role |
| EtherChannel / link aggregation | Not typical entry-level technician work |
| HSRP / VRRP first-hop redundancy | Design-level topic |
| Dynamic routing (OSPF, EIGRP, BGP) | Not needed — all networks are directly connected |
| Port security, 802.1X | Security engineering scope |
| QoS | Not relevant to the target role |
| VXLAN, EVPN, spine-leaf fabric | Advanced data center networking |
| IPv6 | Would double the addressing work for no added relevance |
| NIC teaming / bonding | Would require host OS work, which is Project 1 territory |
| A/B redundant network paths | Documented as a concept. Implementing it needs dual switches per rack and FHRP |
| SSH access to switches | Deliberately omitted — Project 1 covers SSH in depth |

---

## Relationship to Project 1

Project 2 minimizes overlap with Project 1 and focuses on complementary switching, network deployment, rack/port documentation and network troubleshooting skills.

| Project 1 covered | Project 2 covers |
|---|---|
| Linux administration | Cisco IOS switch configuration |
| Static IPv4 on Linux hosts | VLAN design and inter-VLAN routing |
| SSH server configuration and troubleshooting | 802.1Q trunking |
| Nginx and service troubleshooting | DHCP and DNS services on network devices |
| Linux users and permissions | Access control lists |
| Host firewall fundamentals | Switch-port troubleshooting |
| Storage, filesystems, RAID | Rack elevation and asset inventory |
| CPU/RAM/disk monitoring | Cable mapping and labeling |
| Bash and Python automation | Cabling, media and transceiver concepts |
| Server-side incidents (SSH, service, disk, permissions) | Network incidents (VLAN, trunk, DHCP, DNS, ACL, port, documentation) |

Project 2 minimizes overlap with Project 1 and focuses on complementary switching, network deployment, rack/port documentation and network troubleshooting skills.

---

## Evidence status

**Evidence pending.** All documentation, configurations, incident reports and runbooks are complete. The following require live Packet Tracer execution and capture:

- [ ] `packet-tracer/data-center-network.pkt` — the built simulation
- [ ] `packet-tracer/data-center-network-baseline.pkt` — known-good rollback copy
- [ ] `topology/network-topology.png` — topology screenshot
- [ ] `screenshots/` — 12–15 baseline captures plus 2 per incident (failure and fix), roughly 25–30 total
- [ ] `configurations/*.txt` Section B — actual `show running-config` output
- [ ] `network-design/reachability-baseline.md` — matrices filled from real ping results
- [ ] Incident report evidence blocks — actual command output pasted in place of `[ PASTE ACTUAL OUTPUT ]` markers

**No output in this repository has been fabricated.** Every location requiring real evidence is marked with an explicit placeholder rather than filled with plausible-looking text. See `MANUAL-EXECUTION-CHECKLIST.md` for the exact steps.

Until those placeholders are replaced, this repository is a complete design and documentation package with the simulation evidence outstanding. It should not be presented as fully complete until they are filled.
