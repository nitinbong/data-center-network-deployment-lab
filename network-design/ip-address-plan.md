# IP Address Plan

**Addressing scheme:** RFC 1918 private, /24 per VLAN, third octet matches VLAN ID.

Mapping the third octet to the VLAN ID (VLAN 10 → 192.168.10.0/24) means anyone reading an IP address immediately knows which VLAN it belongs to. On a real floor that shortcut saves time on every ticket.

---

## Address reservation convention

Applied identically to every subnet in this network.

| Range | Use |
|---|---|
| `.1` | Gateway (SVI) — always |
| `.2` – `.9` | Network infrastructure (switch management IPs) |
| `.10` – `.99` | Statically assigned servers and fixed hosts |
| `.100` – `.199` | DHCP dynamic pool |
| `.200` – `.254` | Reserved / future growth |

---

## Assigned addresses

| Device | Interface | VLAN | IP address | Mask | Gateway | DNS | Method |
|---|---|---|---|---|---|---|---|
| CORE-SW-01 | SVI Vlan10 | 10 | 192.168.10.1 | 255.255.255.0 | — | — | Static |
| CORE-SW-01 | SVI Vlan20 | 20 | 192.168.20.1 | 255.255.255.0 | — | — | Static |
| CORE-SW-01 | SVI Vlan30 | 30 | 192.168.30.1 | 255.255.255.0 | — | — | Static |
| CORE-SW-01 | SVI Vlan40 | 40 | 192.168.40.1 | 255.255.255.0 | — | — | Static |
| ACCESS-SW-01 | SVI Vlan20 | 20 | 192.168.20.2 | 255.255.255.0 | 192.168.20.1 | — | Static |
| ACCESS-SW-02 | SVI Vlan20 | 20 | 192.168.20.3 | 255.255.255.0 | 192.168.20.1 | — | Static |
| DC-SERVER-01 | NIC1 (data) | 10 | 192.168.10.11 | 255.255.255.0 | 192.168.10.1 | 192.168.10.11 | Static |
| DC-SERVER-01 | NIC2 (mgmt) | 20 | 192.168.20.11 | 255.255.255.0 | 192.168.20.1 | — | Static |
| DC-SERVER-02 | NIC1 | 10 | 192.168.10.12 | 255.255.255.0 | 192.168.10.1 | 192.168.10.11 | Static |
| DC-SERVER-03 | NIC1 | 10 | 192.168.10.13 | 255.255.255.0 | 192.168.10.1 | 192.168.10.11 | Static |
| NOC-PC-01 | NIC1 | 30 | 192.168.30.101+ | 255.255.255.0 | 192.168.30.1 | 192.168.10.11 | DHCP |
| OPS-PC-01 | NIC1 | 30 | 192.168.30.101+ | 255.255.255.0 | 192.168.30.1 | 192.168.10.11 | DHCP |
| TEST-PC-01 | NIC1 | 40 | 192.168.40.50 | 255.255.255.0 | 192.168.40.1 | 192.168.10.11 | Static |

---

## DHCP scope — VLAN 30

| Parameter | Value |
|---|---|
| Pool name | `OPS-POOL` |
| Network | 192.168.30.0 255.255.255.0 |
| Excluded range | 192.168.30.1 – 192.168.30.100 |
| First address issued | 192.168.30.101 |
| Default router | 192.168.30.1 |
| DNS server | 192.168.10.11 |
| Domain name | dclab.local |

**Temporary test addressing.** Where a build guide requires a temporary static address on OPS-PC-01, use `192.168.30.50`. That address sits inside the excluded range and outside the dynamic pool, so it can never collide with a DHCP lease. OPS-PC-01 must be returned to DHCP after any such test.

---

## DNS records — hosted on DC-SERVER-01

| Hostname | Type | Address |
|---|---|---|
| dc-server-01.dclab.local | A | 192.168.10.11 |
| dc-server-02.dclab.local | A | 192.168.10.12 |
| dc-server-03.dclab.local | A | 192.168.10.13 |
| core-sw-01.dclab.local | A | 192.168.20.1 |

---

## Free address inventory

| VLAN | In use | Next available static | Dynamic pool remaining |
|---|---|---|---|
| 10 | .1, .11, .12, .13 | .14 | n/a |
| 20 | .1, .2, .3, .11 | .12 | n/a |
| 30 | .1, .101, .102 | .50 range free | .103 – .254 |
| 40 | .1, .50 | .51 | n/a |
