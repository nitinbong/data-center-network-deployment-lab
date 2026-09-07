# NET-004 — Name Resolution Failure With Intact IP Connectivity

| Field | Value |
|---|---|
| **Incident ID** | NET-004 |
| **Priority** | P2 — services reachable by IP, name-based access failing network-wide |
| **Category** | DNS service failure |
| **Affected Device** | All clients relying on name resolution |
| **DNS server** | DC-SERVER-01 (SRV-A01-001), 192.168.10.11 |
| **Affected Switch** | ACCESS-SW-01 (SW-A01-001) |
| **Affected Port** | Fa0/1 |
| **Affected VLAN** | 10 (SERVERS) — service level, not network level |
| **Detected by** | Users reporting "the servers are down" |
| **Status** | Prepared — execution/evidence pending |

---

## Issue

The DNS service on DC-SERVER-01 was disabled. All IP-level connectivity remained fully functional, but no client could resolve hostnames. Users reported the servers as unreachable when in fact only name resolution had failed.

## Initial symptoms

- `ping dc-server-02.dclab.local` from NOC-PC-01 failed with a host-not-found style error
- `ping 192.168.10.12` from the same client **succeeded immediately**
- All inter-VLAN routing, DHCP leases, trunks and switch ports were healthy
- Clients still showed the correct DNS server (192.168.10.11) in `ipconfig /all`
- DC-SERVER-01 itself was reachable by IP and responding to ping

**The critical observation:** ping by IP works, ping by name fails. That single contrast isolates the fault to name resolution and eliminates the entire network path from consideration. Nothing about VLANs, trunks, routing, or cabling needs to be investigated.

## Fault injection (for reproduction)

On DC-SERVER-01 → **Services** tab → **DNS**: set **DNS Service** to **Off**.

**Alternative injection** producing a narrower fault: delete only the `dc-server-02.dclab.local` A record, leaving the service running. That version demonstrates a single missing record rather than a total service outage — every other name still resolves, which is a more subtle and arguably more realistic scenario.

## Investigation

**Step 1 — separate network from name resolution.** The first and decisive test:

```
ping 192.168.10.12              -> SUCCESS
ping dc-server-02.dclab.local   -> FAIL
```

IP connectivity is intact. The network path — access port, VLAN, trunk, routing, gateway — is provably working, because packets are reaching 192.168.10.12 and coming back. The fault is at or above the name resolution layer.

**Step 2 — verify the client knows where the DNS server is.** `ipconfig /all` on NOC-PC-01 confirmed DNS server 192.168.10.11, delivered correctly by DHCP. The client is configured properly.

**Step 3 — verify the DNS server is reachable.** `ping 192.168.10.11` succeeded. The server is up and the network path to it is fine. So the client can reach the DNS server, but is getting no usable answer from it.

At this point the fault has been narrowed to the DNS service itself, using only three commands and no switch access at all.

**Step 4 — check the service.** DC-SERVER-01 → Services → DNS showed the service set to **Off**. Fault located.

**Step 5 — verify the records survived.** The A records were still present in the configuration; only the service was disabled.

## Commands and checks used

```
ping 192.168.10.12                       (client - by IP, succeeds)
ping dc-server-02.dclab.local            (client - by name, fails)
ping 192.168.10.11                       (client - DNS server reachable)
ipconfig /all                            (client - confirms DNS server address)
nslookup dc-server-02.dclab.local        (if supported in this Packet Tracer build)
```

Inspection of DC-SERVER-01 → Services → DNS (GUI, not CLI).

> **Note on `nslookup`:** support varies between Packet Tracer versions. If the command is unavailable in your build, record that fact here rather than presenting output you did not obtain. Hostname ping is a valid substitute test.

## Evidence

**Before — ping by IP succeeding:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — ping by hostname failing:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — `ipconfig /all` showing DNS server correctly configured:**
```
[ PASTE ACTUAL OUTPUT ]
```

**Before — screenshot of DNS service showing Off state:**
```
[ REFERENCE SCREENSHOT: NET-004-before-fail.png ]
```

## Root cause

The DNS service on DC-SERVER-01 was disabled. Clients sent DNS queries to 192.168.10.11 as configured and received no response, so every name-based operation failed.

The network layer was never involved. IP routing, VLAN configuration, trunking, DHCP and switch port state were all functioning normally throughout the incident. The DNS server itself was reachable — it simply was not answering queries.

## Resolution

1. On DC-SERVER-01 → **Services** → **DNS**, set **DNS Service** to **On**
2. Verify all four A records are present:

| Name | Type | Address |
|---|---|---|
| dc-server-01.dclab.local | A | 192.168.10.11 |
| dc-server-02.dclab.local | A | 192.168.10.12 |
| dc-server-03.dclab.local | A | 192.168.10.13 |
| core-sw-01.dclab.local | A | 192.168.20.1 |

3. Re-test resolution from a client

## Validation

| Check | Command | Expected | Actual |
|---|---|---|---|
| DNS service running | Services → DNS tab | Service = On | |
| All records present | Services → DNS tab | Four A records listed | |
| Resolve server 01 | `ping dc-server-01.dclab.local` | Success | |
| Resolve server 02 | `ping dc-server-02.dclab.local` | Success | |
| Resolve server 03 | `ping dc-server-03.dclab.local` | Success | |
| Resolve core switch | `ping core-sw-01.dclab.local` | Success | |
| Resolution from VLAN 40 | `ping dc-server-02.dclab.local` from TEST-PC-01 | Success | |
| IP connectivity unchanged | `ping 192.168.10.12` | Success | |

**After — ping by hostname succeeding:**
```
[ PASTE ACTUAL OUTPUT ]
```

## Preventive action

1. **Monitor the DNS service, not just the host.** DC-SERVER-01 responded to ping throughout this incident. Host-level monitoring would have shown it green while name resolution was completely down. Service-level checks are required.
2. **Include a name resolution test in deployment validation.** `deployment/new-server-deployment-checklist.md` Section 6 requires a hostname ping, not just an IP ping.
3. **Consider a secondary DNS server.** A single DNS server is a single point of failure for every name-based operation on the network. Not implemented in this lab, but it is the correct production answer.
4. **Train the first diagnostic reflex:** when a user says "the server is down", ping it by IP before anything else. That one test splits the problem space in half.

## Lessons learned

The three-outcome table that should be internalised:

| `ping <IP>` | `ping <hostname>` | Conclusion |
|---|---|---|
| Works | Works | Healthy |
| Works | Fails | **DNS problem.** Network path is fine |
| Fails | Fails | **Network problem.** DNS is irrelevant until connectivity is restored |

This incident is the middle row. Users reported it as a server outage; it was a name resolution outage on a fully healthy network. Being able to make that distinction in under thirty seconds — and communicate it accurately — is one of the more valuable things an entry-level technician can do, because it stops an entire team from troubleshooting the wrong layer.
