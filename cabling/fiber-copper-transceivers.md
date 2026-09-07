# Cabling, Media & Transceiver Concepts

> **Scope disclaimer — read first.**
>
> This document demonstrates **conceptual knowledge only**. All network links in this project were built in Cisco Packet Tracer, which simulates cabling. No physical cable was terminated, no fiber was spliced or polished, no transceiver was seated, and no patch panel was punched down as part of this project. Nothing here is a claim of hands-on physical cabling experience.
>
> Where this lab uses copper because Packet Tracer offers nothing else, that is stated plainly rather than described as fiber.

---

## 1. Copper media

### Cat5e, Cat6, Cat6A

| Standard | Max speed | Max distance at that speed | Typical use |
|---|---|---|---|
| Cat5e | 1 Gbps | 100 m | Legacy access layer, still common for workstations |
| Cat6 | 1 Gbps (10 Gbps to 55 m) | 100 m at 1 Gbps | Standard server and workstation patching |
| Cat6A | 10 Gbps | 100 m | 10 Gbps server connections, short copper uplinks |

The practical distinction: **Cat6 supports 10 Gbps only over short runs; Cat6A supports it for the full 100 m.** Cat6A achieves this with tighter twist rates and additional shielding to control alien crosstalk — interference between adjacent cables in a dense bundle, which becomes a real problem in a packed rack.

Cat6A is physically thicker and less flexible than Cat6. In a high-density rack that affects bend radius, cable management arm capacity, and how much airflow the bundle blocks.

### The 100 metre limit

All twisted-pair Ethernet is limited to 100 m total channel length — typically 90 m of horizontal solid-core run plus up to 10 m of stranded patch cords at each end. Exceeding it causes intermittent errors, CRC failures, and links that negotiate at lower speeds rather than failing outright. Intermittent problems that appear under load are a classic symptom of an over-length or marginal copper run.

### RJ45

The 8-position 8-contact modular connector used for twisted-pair Ethernet. Four pairs, eight conductors.

- **T568A and T568B** are the two pinout standards. Either works, but **both ends of a cable must use the same standard** unless a crossover is intended. Mixing them accidentally produces a crossover cable, which on modern auto-MDI/MDIX hardware may still work — which is exactly why the mistake goes unnoticed until it hits older equipment.
- **Straight-through** cables connect unlike devices: host to switch, switch to router.
- **Crossover** cables traditionally connected like devices: switch to switch, host to host. Modern Ethernet devices commonly support Auto-MDI/MDIX and negotiate this automatically, so straight-through frequently works. This lab uses crossover for switch-to-switch links to clearly represent the traditional convention.

---

## 2. Fiber optic media

### Multimode vs single-mode

| Property | Multimode (MMF) | Single-mode (SMF) |
|---|---|---|
| Core diameter | 50 µm (OM3/OM4/OM5) | ~9 µm |
| Light source | VCSEL laser / LED (850 nm) | Laser (1310 / 1550 nm) |
| Typical distance | 100–550 m depending on grade | 10 km to 80 km+ |
| Relative optics cost | Lower | Higher |
| Jacket colour convention | Aqua (OM3/OM4), lime green (OM5) | Yellow |
| Typical use | Within a data center — rack to rack, row to row | Between buildings, campus, long haul |

**The core reason for the difference.** Multimode has a wide core, so light enters at multiple angles ("modes") and different rays take slightly different path lengths. Over distance those rays arrive spread out in time — *modal dispersion* — and the signal blurs until it is unreadable. Single-mode's narrow core permits essentially one path, eliminating modal dispersion and allowing far greater distance.

**The practical rule:** inside a data center, multimode. Between buildings, single-mode.

**Grades matter.** OM3 supports 10 Gbps to ~300 m; OM4 to ~400 m; OM5 adds wavelength division support. Specifying "multimode" without the grade is not enough on a real order.

### LC connectors

The dominant fiber connector in modern data centers. A **small form factor** connector — roughly half the footprint of the older SC — which is why LC dominates in high-density panels where port count per rack unit is the constraint.

Fiber is normally run as a **duplex pair**: one strand transmits, one receives. Duplex LC clips the two ferrules together in a single housing so the pair stays correctly oriented.

Other connectors worth recognising: **SC** (square push-pull, older/legacy), **MPO/MTP** (multi-fiber, 12 or 24 strands in one connector, used for 40G/100G breakout).

### Fiber handling basics

- **Never look into the end of a live fiber.** The light is invisible and can cause permanent retinal damage.
- **Keep dust caps on** whenever a connector is unmated. A single dust particle on a 9 µm core is a significant obstruction, and contamination is the leading cause of fiber link faults.
- **Respect minimum bend radius.** Bending fiber too tightly causes light to escape the core — attenuation, or a broken strand.
- **Clean before mating.** Inspect and clean with proper tools, not a shirt sleeve.

---

## 3. Transceivers

A transceiver is a hot-swappable module that converts electrical signals inside the switch to optical (or copper) signals on the cable. Because it is modular, one switch port can run copper, short-range fiber, or long-range fiber depending on which module is fitted.

| Form factor | Typical speed | Notes |
|---|---|---|
| **SFP** | 1 Gbps | Small Form-factor Pluggable. The original density-oriented module |
| **SFP+** | 10 Gbps | Same physical size as SFP, higher speed. Most common DC server/uplink optic |
| **SFP28** | 25 Gbps | Same footprint again, used for 25G server connections |
| **QSFP+** | 40 Gbps | "Quad" SFP — four 10G lanes in one module |
| **QSFP28** | 100 Gbps | Four 25G lanes. Standard for modern spine/leaf uplinks |

### Common variants you will see stencilled on a module

| Marking | Meaning |
|---|---|
| `10GBASE-SR` | Short range, multimode, up to ~300–400 m |
| `10GBASE-LR` | Long range, single-mode, up to 10 km |
| `1000BASE-T` | Copper SFP with an RJ45 port |
| `DAC` | Direct Attach Copper — a fixed cable with transceivers moulded on both ends. Cheap and low-latency for very short in-rack runs (1–7 m). Cannot be re-terminated |

### Operational points that matter on the floor

- **Transceivers must match at both ends.** An SR module on one end and an LR on the other will not link, or will link with errors. Wavelength and mode must match.
- **The transceiver must match the fiber.** An SR (multimode) optic on single-mode fiber will not work correctly.
- **Vendor coding is real.** Many switches reject third-party optics unless explicitly allowed. "The optic is fine, the switch refuses it" is a genuine failure mode.
- **A transceiver is a field-replaceable part**, and swapping a suspect one is a standard early troubleshooting step for a link that will not come up or shows errors.

### Packet Tracer limitation

The Cisco 2960-24TT uplink ports simulated in this lab are fixed copper. Real data centers may use multimode fiber, single-mode fiber, DAC/AOC, or other media and optics depending on distance, bandwidth and design. Packet Tracer does not model transceiver insertion, optical power levels, or `show interfaces transceiver` output. Everything in this section is documented understanding — none of it was physically performed or simulated in this project.

---

## 4. NIC, TX/RX and link LEDs

### NIC

The Network Interface Card is the server's connection to the network. Points that matter operationally:

- A server may have **multiple NICs** for different purposes — production data, management, storage, backup. DC-SERVER-01 in this lab has two: NIC1 on VLAN 10 for data, NIC2 on VLAN 20 for management.
- Each NIC has its own **MAC address**, and that MAC is what appears in the switch's MAC address table. This is how a technician maps a physical server to a physical switch port.
- Multiple NICs may be **bonded/teamed** for redundancy or throughput. Not implemented in this lab.

### TX/RX

**TX** is transmit, **RX** is receive. Every link needs both directions.

On fiber, TX and RX are physically separate strands, so **they must be crossed**: the TX of one device connects to the RX of the other. Duplex LC connectors handle this automatically when correctly polarised. A fiber pair connected TX-to-TX is the most common fiber installation error, and the symptom is a link that never comes up despite both optics being healthy.

On copper, the pairs perform both roles and auto-MDI/MDIX handles the crossing electronically.

### Link LEDs

The link LED on a NIC or switch port is the fastest diagnostic available, and it answers one question decisively:

| LED state | Meaning | What to check next |
|---|---|---|
| **Off / dark** | No physical link. Layer 1 problem | Cable seated? Correct port? Port administratively shut? Far-end device powered on? Faulty cable or transceiver? |
| **Solid on** | Link established, no traffic passing | Physical layer is fine. Move up the stack: VLAN, IP, gateway |
| **Blinking** | Link established, traffic flowing | Physical layer confirmed working |
| **Amber / orange** | Varies by vendor. Often speed indication, port disabled, or STP not yet forwarding | Check `show interfaces status` for the authoritative state |

**The key inference:** a dark link light means stop and fix Layer 1 — nothing above it can work. A lit link light means Layer 1 is fine and the problem is above it. That single branch is the top of the troubleshooting runbook, and NET-006 in this project is precisely the case where a port is up on the cabling side but administratively shut down at the switch.

**Do not trust the LED alone.** `show interfaces status` on the switch is authoritative and tells you the port state, speed, duplex and VLAN in one line. An LED tells you about the physical link; the switch tells you about the configuration.

---

## 5. Patch panels

A patch panel is a passive termination point. Permanent structured cabling — run through walls, floors, and overhead trays — terminates on the back of the panel. Short, replaceable patch cords connect the front of the panel to the switch.

**Why bother:**

- **Structured cable is not moved.** Only the short patch cords are handled, so the permanent infrastructure is not stressed by repeated re-patching.
- **Damage is cheap to repair.** A failed patch cord is replaced in seconds; a failed in-wall run is a project.
- **Cable management is cleaner.** All runs land in one predictable place instead of dozens of long cables snaking to individual switch ports.
- **Ports are labelled and numbered**, making tracing a matter of reading rather than following a cable by hand.

In this project, PP-A01-01 and PP-A02-01 sit at U41 in each rack, directly below the top-of-rack switch, so patch cords between panel and switch are only inches long.

---

## 6. Cable labeling

Every cable is labelled **at both ends**, with the same label. An unlabelled cable is an unknown cable, and an unknown cable is one nobody is willing to unplug.

**Convention used in this project:**

```
<SOURCE-RACK>-<SOURCE-DEVICE>-<PORT>__<DEST-RACK>-<DEST-DEVICE>-<PORT>
```

**Example:**

```
A01-SRV001-NIC1__A01-SW001-Fa0-1
```

Read directly off the label: rack A01, server 001, NIC 1, connecting to rack A01, switch 001, port Fa0/1. No lookup required, no tracing required.

The same labels appear in `cable-map.csv`, in `switch-port-map.csv`, and in the switch port descriptions themselves (`show interfaces description`). Three independent sources that must agree. When they disagree, something was changed without documentation — which is exactly the fault reproduced in NET-007.

---

## 7. Cable tracing

Finding which physical cable corresponds to which logical connection. Methods, in order of preference:

1. **Read the label.** If labeling is disciplined, this is the entire job.
2. **Check the switch port description.** `show interfaces description` returns what the documentation says is connected.
3. **Check the MAC address table.** `show mac address-table` returns which MAC is learned on which port. Compare against the device's actual NIC MAC. The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation.
4. **Port shut/no-shut test.** Administratively shut the port and watch which link LED goes dark. Effective, but it drops the link — only acceptable on a host confirmed out of service or during a maintenance window.
5. **Toner and probe.** A tone generator on one end, an inductive probe used to find the matching cable in a bundle. Standard for unlabelled copper.
6. **Visual fault locator.** A red laser injected into a fiber, visible at the far end. The fiber equivalent of a toner.

**Method 3 is the one this project exercises most.** The MAC address table is a key live source for identifying Layer 2 attachment and should be cross-checked with interface status, endpoint MAC addresses, port descriptions and documentation.

---

## 8. A/B network redundancy

In a production data center, critical servers are connected to **two independent network paths**:

- **NIC1 → Switch A** (A-side)
- **NIC2 → Switch B** (B-side)

The two switches are separate physical devices, ideally in separate racks, on **separate power feeds (A/B PDUs)**. The design intent is that no single failure — one switch, one PDU, one uplink, one cable, one transceiver — takes a server offline.

This extends beyond the network: dual power supplies to A and B feeds, dual uplinks from each switch, and often dual paths all the way to the core.

**What this lab does and does not have.** This project does **not** implement A/B redundancy. Each device has a single network path. DC-SERVER-01's second NIC is a *dedicated management interface on a separate VLAN* — it is a management separation design, not a redundancy design, and it would not keep the server online if ACCESS-SW-01 failed.

Redundancy is documented here as understanding. Implementing it properly would require NIC teaming, dual switches per rack, and first-hop redundancy such as HSRP or VRRP — all deliberately out of scope for an entry-level project.

---

## 9. Quick reference

| Question | Answer |
|---|---|
| Copper max distance? | 100 m for all twisted-pair Ethernet |
| Cat6 vs Cat6A? | Cat6A does 10 Gbps for the full 100 m; Cat6 only to ~55 m |
| Multimode vs single-mode? | Multimode = short, inside the DC, wide core, cheaper optics. Single-mode = long haul, narrow core |
| Which fiber colour? | Aqua = OM3/OM4 multimode. Yellow = single-mode |
| Most common DC fiber connector? | LC, usually duplex |
| SFP vs SFP+? | Same size, 1 Gbps vs 10 Gbps |
| What is QSFP? | Quad SFP — four lanes in one module. QSFP+ = 40G, QSFP28 = 100G |
| What is a DAC? | Direct Attach Copper — fixed cable with integrated optics, for very short in-rack runs |
| Link light off means? | Layer 1 problem. Fix it before checking anything else |
| Link light solid, no traffic? | Physical layer fine. Check VLAN, IP, gateway |
| How do I find a server's switch port? | MAC address table, cross-checked against the port map |
| Why label both ends? | So the cable can be identified from either end without tracing |
