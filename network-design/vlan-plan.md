# VLAN Plan

**Network:** Data Center Network Deployment Lab
**Layer 3 gateway device:** CORE-SW-01 (Cisco 3650, `ip routing` enabled)

---

## VLAN table

| VLAN ID | Name | Subnet | Gateway (SVI) | Purpose |
|---|---|---|---|---|
| 10 | SERVERS | 192.168.10.0/24 | 192.168.10.1 | Simulated server data NICs |
| 20 | MANAGEMENT | 192.168.20.0/24 | 192.168.20.1 | Switch management IPs, dedicated server management NIC |
| 30 | OPERATIONS | 192.168.30.0/24 | 192.168.30.1 | NOC and operations workstations (DHCP) |
| 40 | UNTRUSTED | 192.168.40.0/24 | 192.168.40.1 | Test / unvetted devices, ACL-restricted |
| 99 | NATIVE | n/a | none | Native VLAN on 802.1Q trunks. No hosts assigned. |

---

## VLAN membership by switch

### ACCESS-SW-01 (RACK-A01)

| Port | Mode | VLAN | Connected device |
|---|---|---|---|
| Fa0/1 | access | 10 | DC-SERVER-01 NIC1 |
| Fa0/2 | access | 10 | DC-SERVER-02 NIC1 |
| Fa0/3 | access | 10 | DC-SERVER-03 NIC1 |
| Gi0/1 | trunk | 10,20,30,40,99 | CORE-SW-01 Gi1/0/1 |

VLAN 20 is defined on this switch and carried on the trunk for the management SVI, but has no local access ports.

### ACCESS-SW-02 (RACK-A02)

| Port | Mode | VLAN | Connected device |
|---|---|---|---|
| Fa0/5 | access | 20 | DC-SERVER-01 NIC2 (management) |
| Fa0/10 | access | 30 | NOC-PC-01 |
| Fa0/11 | access | 30 | OPS-PC-01 |
| Fa0/20 | access | 40 | TEST-PC-01 |
| Gi0/1 | trunk | 10,20,30,40,99 | CORE-SW-01 Gi1/0/2 |

### CORE-SW-01 (RACK-A02)

| Port | Mode | VLAN | Connected device |
|---|---|---|---|
| Gi1/0/1 | trunk | 10,20,30,40,99 | ACCESS-SW-01 Gi0/1 |
| Gi1/0/2 | trunk | 10,20,30,40,99 | ACCESS-SW-02 Gi0/1 |

No host-facing access ports. All four VLANs exist here to host their SVIs.

---

## Design notes

**Why VLAN 99 as native.** Untagged frames arriving on an 802.1Q trunk are placed in the native VLAN. Leaving that as the default VLAN 1 means stray untagged traffic lands in the same VLAN as unconfigured ports. Moving the native VLAN to an otherwise unused ID is standard hygiene. Native VLAN must match on both ends of a trunk or the switches will log a mismatch.

**Why servers are not on DHCP.** Server addresses must be stable and predictable — DNS records, firewall rules, monitoring targets and documentation all reference them. DHCP is used only for workstations, where addresses are disposable.

**Why management is a separate VLAN.** Separating management traffic from production data traffic means a broadcast storm or misconfiguration in the server VLAN does not cost you access to the switches themselves. It also creates a clean boundary to apply access control against, which is what the VLAN 40 ACL enforces.

**Terminology limit.** VLAN 20 is a *dedicated management VLAN*, not true out-of-band management. Real OOB uses a physically separate baseboard controller (iDRAC, iLO, BMC) on an isolated network reachable even when the host OS is down. This lab demonstrates in-band management separation only.
