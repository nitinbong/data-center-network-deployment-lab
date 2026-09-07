# Rack Elevation

Two simulated racks. Rack units are numbered **bottom-up**: U01 at the floor, U42 at the top.

> **Scope note.** These elevations are a documentation exercise built around a Cisco Packet Tracer simulation. No physical racking, mounting, or cabling was performed.

---

## RACK-A01 — Compute

**Location:** Row A, Position 01
**Function:** Simulated server compute
**Power:** 2x PDU (A/B feed) — documented as design intent, not simulated

| RU | Equipment | Asset ID | Type | Notes |
|---|---|---|---|---|
| U42 | ACCESS-SW-01 | SW-A01-001 | Cisco 2960-24TT | Top-of-rack access switch |
| U41 | PP-A01-01 | PP-A01-001 | 24-port copper patch panel | Cat6 terminations |
| U40 | DC-SERVER-01 | SRV-A01-001 | 1U server | Simulated server + DNS service |
| U39 | DC-SERVER-02 | SRV-A01-002 | 1U server | Simulated server |
| U38 | DC-SERVER-03 | SRV-A01-003 | 1U server | Simulated server |
| U37–U01 | — | — | — | Reserved / blanking panels |

```
 U42  [ ACCESS-SW-01           ]  <- TOR switch
 U41  [ PP-A01-01              ]  <- patch panel
 U40  [ DC-SERVER-01           ]
 U39  [ DC-SERVER-02           ]
 U38  [ DC-SERVER-03           ]
 U37  [                        ]
  ..  [   reserved / blanking  ]
 U01  [                        ]
```

---

## RACK-A02 — Core and Operations

**Location:** Row A, Position 02
**Function:** Network core and operations aggregation

| RU | Equipment | Asset ID | Type | Notes |
|---|---|---|---|---|
| U42 | ACCESS-SW-02 | SW-A02-001 | Cisco 2960-24TT | Top-of-rack access switch |
| U41 | PP-A02-01 | PP-A02-001 | 24-port copper patch panel | Cat6 terminations |
| U40 | CORE-SW-01 | SW-A02-002 | Cisco 3650-24PS | Layer 3 core — SVIs, DHCP, ACL |
| U39–U01 | — | — | — | Reserved / blanking panels |

```
 U42  [ ACCESS-SW-02           ]  <- TOR switch
 U41  [ PP-A02-01              ]  <- patch panel
 U40  [ CORE-SW-01             ]  <- Layer 3 core
 U39  [                        ]
  ..  [   reserved / blanking  ]
 U01  [                        ]
```

---

## Non-racked equipment

| Device | Location | Patched via | Landing port |
|---|---|---|---|
| NOC-PC-01 | NOC room | PP-A02-01 | ACCESS-SW-02 Fa0/10 |
| OPS-PC-01 | NOC room | PP-A02-01 | ACCESS-SW-02 Fa0/11 |
| TEST-PC-01 | NOC room | PP-A02-01 | ACCESS-SW-02 Fa0/20 |

Workstations occupy no rack units. Their cable runs terminate on the RACK-A02 patch panel and are patched from the panel to the switch.

---

## Inter-rack link

| From | To | Simulated medium | Real-world equivalent |
|---|---|---|---|
| ACCESS-SW-01 Gi0/1 (RACK-A01) | CORE-SW-01 Gi1/0/1 (RACK-A02) | Cat6A copper | Varies by design — see note below |

The Cisco 2960-24TT uplink ports in Packet Tracer are copper only, so the simulated inter-rack run is Cat6A. Real data centers may use multimode fiber, single-mode fiber, DAC/AOC, or other media and optics depending on distance, bandwidth and design. See `cabling/fiber-copper-transceivers.md`.

---

## Design rationale

**Top-of-rack switching.** Placing the access switch at U42 keeps server-to-switch patch cords short, keeps cable management contained within the rack rather than running horizontally across the row, and makes server-to-switch-port mapping easy to trace visually.

**Patch panel directly below the switch.** A one-U gap between panel and switch means patch cords are inches long and can be dressed cleanly. Structured cabling terminates on the panel; only short patch cords touch the switch itself. If a run is damaged, the panel port is replaced without disturbing the switch.

**Blanking panels in empty units.** Empty rack units allow hot exhaust air to recirculate to the cold intake side. Blanking panels force air through the equipment instead of around it. This is documented as intent — airflow is not simulated in Packet Tracer.

**Compute separated from core.** Splitting servers into RACK-A01 and network core into RACK-A02 creates a clear inter-rack uplink to document and keeps the failure domain of a rack-level power or cooling event limited.
