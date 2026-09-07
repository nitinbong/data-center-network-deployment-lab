# Phase 5, 6 & 7 — DHCP, DNS and ACL

**Prerequisite:** Phases 1–4 complete. All four SVIs up, `ip routing` enabled, inter-VLAN pings succeeding, OPS-PC-01 returned to DHCP, TEST-PC-01 back in VLAN 40 at 192.168.40.50.

---

# PHASE 5 — DHCP

## 5.1 The DORA process

DHCP assigns IP configuration automatically through a four-message exchange. Learn it as **DORA**.

| Step | Message | Sent by | Direction | What it carries |
|---|---|---|---|---|
| **D** | DISCOVER | Client | Broadcast | "Is there a DHCP server out there?" Client has no IP yet, so source is 0.0.0.0 and destination is 255.255.255.255 |
| **O** | OFFER | Server | Unicast/broadcast | "Here is an address you can have" — proposed IP, mask, gateway, DNS, lease time |
| **R** | REQUEST | Client | Broadcast | "I accept that offer" — broadcast so any other servers that made offers know to withdraw them |
| **A** | ACKNOWLEDGE | Server | Unicast | "Confirmed, it's yours for the lease duration" — client may now use the address |

### The detail that matters operationally

DISCOVER is a **broadcast**, and routers do not forward broadcasts. That means a DHCP server must either be inside the client's own VLAN, or the router interface for that VLAN must be configured with `ip helper-address` to relay DHCP requests to a server elsewhere.

In this lab neither is a problem: CORE-SW-01 is both the router and the DHCP server, and `interface Vlan30` is directly inside VLAN 30. No relay is needed. But "the DHCP server is in a different VLAN and there is no helper-address" is one of the most common real-world DHCP faults, and being able to name it is worth points in an interview.

### How to spot DHCP failure instantly

A Windows client that cannot reach a DHCP server assigns itself an **APIPA** address in `169.254.x.x`. If you see 169.254 anywhere, the client asked and got no answer. That is not a "wrong IP" problem — it is a "no DHCP response reached me" problem, and the causes are: wrong VLAN, DHCP not configured, pool exhausted, or no helper-address across a router boundary.

---

## 5.2 Configure the DHCP pool

On **CORE-SW-01**:

```
enable
configure terminal
!
ip dhcp excluded-address 192.168.30.1 192.168.30.100
!
ip dhcp pool OPS-POOL
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 192.168.10.11
 domain-name dclab.local
 exit
!
end
copy running-config startup-config
```

### Line by line

| Command | Purpose |
|---|---|
| `ip dhcp excluded-address 192.168.30.1 192.168.30.100` | Reserves `.1`–`.100` so DHCP never hands them out. Protects the gateway and the static/infrastructure range. Configured **globally**, not inside the pool — a common mistake is trying to put it under the pool, where it is not accepted. |
| `ip dhcp pool OPS-POOL` | Creates the pool and enters pool configuration mode |
| `network 192.168.30.0 255.255.255.0` | Defines the scope. Combined with the exclusion, usable addresses start at `.101` |
| `default-router 192.168.30.1` | The gateway handed to clients — the VLAN 30 SVI |
| `dns-server 192.168.10.11` | DC-SERVER-01, configured in Phase 6 |
| `domain-name dclab.local` | Appended to unqualified hostname lookups |

**The first address handed out will be 192.168.30.101.** If a client receives anything below `.101`, the exclusion did not apply.

---

## 5.3 Set the clients to DHCP

On **NOC-PC-01** and **OPS-PC-01**: Desktop → IP Configuration → select **DHCP**.

The status field should change to `DHCP request successful`. If it reads `DHCP request failed` or the address shows `169.254.x.x`, stop and troubleshoot before continuing — do not proceed with a broken pool.

---

## 5.4 Phase 5 validation

### Test 1: pool state

On CORE-SW-01:

```
show ip dhcp pool
```

Check `Total addresses`, `Leased addresses`, and the `Current index`. Leased should equal 2 after both clients obtain addresses.

### Test 2: bindings

```
show ip dhcp binding
```

Two entries, each showing an IP in the `.101`+ range mapped to a client hardware address with a lease expiry. Cross-check the MAC addresses against the client Config tabs — this is the same device-to-address verification skill used with the MAC address table.

### Test 3: client-side configuration

On NOC-PC-01 → Desktop → Command Prompt:

```
ipconfig /all
```

Verify all four values arrived: IP address, subnet mask, default gateway `192.168.30.1`, DNS server `192.168.10.11`. The whole point of DHCP is that all four come from one exchange — a client with an IP but no gateway usually means the pool is missing `default-router`.

### Test 4: connectivity on a leased address

```
ping 192.168.30.1
ping 192.168.10.11
```

Both should succeed. A leased address is only useful if it actually works.

### Test 5: renew

```
ipconfig /release
ipconfig /renew
```

Watch the address return. On CORE-SW-01, `show ip dhcp binding` should show the refreshed lease. This is the exact procedure used to fix NET-003 after the pool is repaired, so run it once here while everything is healthy.

## 5.5 Phase 5 screenshots

Capture only what `screenshots/README.md` lists for this phase. The project targets roughly 25–30 screenshots in total (12–15 baseline, 2 per incident), so capture the clearest single view of each capability rather than every command you run.


---

# PHASE 6 — DNS

## 6.1 The concept

DNS translates names to IP addresses. It sits **above** IP connectivity, and that layering is the whole diagnostic point of this phase.

| Symptom | What it means |
|---|---|
| `ping 192.168.10.12` works, `ping dc-server-02.dclab.local` works | Everything healthy |
| `ping 192.168.10.12` works, `ping dc-server-02.dclab.local` fails | **DNS problem.** The network path is fine; name resolution is broken |
| `ping 192.168.10.12` fails, `ping dc-server-02.dclab.local` fails | **Network problem.** DNS is irrelevant until the path is fixed |

The middle row is NET-004. The single most useful reflex when someone says "the server is down" is to ping it by IP: if the IP works, the network is fine and you are looking at DNS or the application, not connectivity.

## 6.2 Configure DNS on DC-SERVER-01

1. Click **DC-SERVER-01** → **Services** tab → **DNS**
2. Set **DNS Service** to **On**
3. Add each record: type the Name, select Type `A Record`, enter the Address, click **Add**

| Name | Type | Address |
|---|---|---|
| `dc-server-01.dclab.local` | A Record | 192.168.10.11 |
| `dc-server-02.dclab.local` | A Record | 192.168.10.12 |
| `dc-server-03.dclab.local` | A Record | 192.168.10.13 |
| `core-sw-01.dclab.local` | A Record | 192.168.20.1 |

Confirm all four appear in the record list. The DNS service must show **On** — a server with perfect records and the service off is exactly the NET-004 fault.

### DNS server reachability

The DNS server is at 192.168.10.11 in VLAN 10. Clients in VLAN 30 reach it across the inter-VLAN routing built in Phase 4. If routing were broken, DNS would fail too — which is why the runbook checks routing *before* DNS.

## 6.3 Phase 6 validation

### Test 1: clients received the DNS server address

On NOC-PC-01:

```
ipconfig /all
```

DNS server should read `192.168.10.11`, delivered by DHCP.

For the statically-addressed hosts (DC-SERVER-02, DC-SERVER-03, TEST-PC-01), confirm the DNS field is set to `192.168.10.11` in IP Configuration. Set it now if blank.

### Test 2: resolution from a DHCP client

On NOC-PC-01:

```
ping 192.168.10.12
ping dc-server-02.dclab.local
```

Both should succeed. Note in the output that the hostname ping displays the resolved IP — that display is the resolution succeeding.

### Test 3: nslookup

```
nslookup dc-server-02.dclab.local
```

Packet Tracer's `nslookup` support varies by version. If the command is unavailable, say so plainly in your documentation and use hostname ping as the resolution test instead. Do not write up a command you could not run.

### Test 4: resolve every record

From NOC-PC-01, ping all four names:

```
ping dc-server-01.dclab.local
ping dc-server-02.dclab.local
ping dc-server-03.dclab.local
ping core-sw-01.dclab.local
```

`core-sw-01.dclab.local` resolves to 192.168.20.1 — the management SVI — so success also proves a VLAN 30 client can reach the management VLAN. Keep that result: it is the "before" evidence for NET-005.

### Test 5: resolution from VLAN 40

From TEST-PC-01:

```
ping dc-server-02.dclab.local
```

Should succeed. No ACL exists yet. After Phase 7, VLAN 40 will still resolve names and reach VLAN 10, but will be blocked from VLAN 20 — a useful demonstration that an ACL blocks specific traffic, not all traffic.

## 6.4 Phase 6 screenshots

Capture only what `screenshots/README.md` lists for this phase. The project targets roughly 25–30 screenshots in total (12–15 baseline, 2 per incident), so capture the clearest single view of each capability rather than every command you run.


---

# PHASE 7 — Access Control List

## 7.1 The policy

One rule, one application point:

> Hosts in the UNTRUSTED VLAN (40) must not reach the MANAGEMENT VLAN (20). All other traffic is permitted.

The reasoning is straightforward: an unvetted or staging device has no business reaching switch management interfaces or server management NICs. It may still need internet or server access, so everything else stays open.

## 7.2 How ACLs are evaluated

Three rules that explain most ACL mistakes:

1. **Top-down, first match wins.** Evaluation stops at the first matching line. Order is everything — a broad `permit` above a specific `deny` makes the deny unreachable.
2. **Implicit `deny any` at the end.** Every ACL ends with an invisible deny-all. An ACL containing only `deny` statements blocks everything. That is why the explicit `permit ip any any` below is mandatory, not optional.
3. **Direction is relative to the interface.** `in` means traffic entering the switch from that VLAN. Applied inbound on VLAN 40, the ACL can only filter traffic *sourced from* VLAN 40 — it cannot affect VLAN 30's traffic at all. This is exactly why NET-005 has to create a separate misapplied ACL rather than pretending this one caused the fault.

## 7.3 Configure and apply

On **CORE-SW-01**:

```
enable
configure terminal
!
ip access-list extended UNTRUSTED-TO-MGMT
 remark Deny VLAN 40 UNTRUSTED access to VLAN 20 MANAGEMENT
 deny ip 192.168.40.0 0.0.0.255 192.168.20.0 0.0.0.255
 remark Permit all other traffic
 permit ip any any
 exit
!
interface Vlan40
 ip access-group UNTRUSTED-TO-MGMT in
 exit
!
end
copy running-config startup-config
```

### Wildcard masks

ACLs use wildcard masks, not subnet masks. `0.0.0.255` is the inverse of `255.255.255.0`: a `0` bit means "must match", a `1` bit means "don't care". So `192.168.40.0 0.0.0.255` matches any address in 192.168.40.0/24. Getting this backwards is a classic error — `255.255.255.0` as a wildcard would match almost nothing you intended.

## 7.4 Phase 7 validation

### Test 1: the ACL exists and is correct

```
show access-lists
```

Both lines present, deny before permit. After running the tests below, run this again — the match counters next to each line increment as traffic hits them, which is direct evidence the ACL is being evaluated rather than merely configured.

### Test 2: applied to the right interface, right direction

```
show ip interface Vlan40
```

Look for `Inbound access list is UNTRUSTED-TO-MGMT`. If it shows outbound or none, the `ip access-group` line landed wrong.

### Test 3: blocked traffic — the deny works

From **TEST-PC-01** (192.168.40.50, VLAN 40):

```
ping 192.168.20.2
ping 192.168.20.3
ping 192.168.20.11
```

All three must **fail**. Screenshot.

### Test 4: permitted traffic — the permit works

Still from TEST-PC-01:

```
ping 192.168.10.11
ping 192.168.10.12
ping dc-server-02.dclab.local
```

All should **succeed**. This is the critical half of the test. An ACL that blocks everything is not a working ACL, it is an outage — and proving that only the intended traffic was blocked is what separates a validated change from a lucky one.

### Test 5: other VLANs unaffected

From **NOC-PC-01** (VLAN 30, DHCP):

```
ping 192.168.20.2
ping 192.168.20.3
ping core-sw-01.dclab.local
```

All should **succeed**. Operations retains full management access. This confirms the ACL is scoped to VLAN 40 only, and it is the baseline NET-005 will break.

### Test 6: post-ACL reachability matrix

Re-run the full matrix from Phase 4 and record the differences. Only the VLAN 40 → VLAN 20 cells should have changed from success to failure. Any other changed cell means the ACL is over-scoped.

Save both matrices — pre-ACL and post-ACL — into `network-design/reachability-baseline.md`. The pair is the evidence that the change did exactly what was intended.

## 7.5 Phase 7 screenshots

Capture only what `screenshots/README.md` lists for this phase. The project targets roughly 25–30 screenshots in total (12–15 baseline, 2 per incident), so capture the clearest single view of each capability rather than every command you run.


---

# BASELINE CAPTURE — do this before any incident

The network is now feature-complete. Everything from here is deliberate breakage, so the known-good state must be preserved first.

## Step 1: save configurations to startup-config

On all three switches:

```
enable
copy running-config startup-config
```

## Step 2: archive the running configs

On each switch run `show running-config` and paste the complete output into:

- `configurations/core-switch.txt`
- `configurations/access-switch-01.txt`
- `configurations/access-switch-02.txt`

Each file in the repository already contains the reference configuration that *should* have been applied. Append your actual captured output below the marker in each file. If the two differ, the captured output is the truth and the reference needs correcting.

## Step 3: save two copies of the .pkt

```
packet-tracer/data-center-network-baseline.pkt
packet-tracer/data-center-network.pkt
```

Use **File → Save As** twice. The baseline copy is never modified again — it is the rollback point. All incident work happens in `data-center-network.pkt`, which is restored to this state at the end.

## Step 4: baseline evidence checklist

Confirm you have captured, from real output:

- [ ] `show vlan brief` — all three switches
- [ ] `show interfaces status` — both access switches
- [ ] `show interfaces trunk` — all three switches
- [ ] `show mac address-table` — all three switches
- [ ] `show ip interface brief` — CORE-SW-01
- [ ] `show ip route` — CORE-SW-01
- [ ] `show ip dhcp pool` and `show ip dhcp binding` — CORE-SW-01
- [ ] `show access-lists` and `show ip interface Vlan40` — CORE-SW-01
- [ ] `ipconfig /all` — NOC-PC-01 and OPS-PC-01
- [ ] DNS resolution success from a VLAN 30 client
- [ ] Completed pre-ACL and post-ACL reachability matrices

Anything unchecked is evidence you will wish you had while writing the incident reports. Incident write-ups compare a broken state against a known-good state; without the known-good capture there is nothing to compare against.
