# Packet Tracer Files

| File | Purpose | Status |
|---|---|---|
| `data-center-network-baseline.pkt` | Known-good state captured at the end of Phase 7, before any incident injection. Never modified. This is the rollback point. | Awaiting capture |
| `data-center-network.pkt` | Working file. All incident injection and resolution happens here. Restored to baseline state after incident testing completes. | Awaiting capture |

**Built with:** Cisco Packet Tracer 8.x

**Devices:** 1x Catalyst 3650-24PS (or 3560-24PS), 2x Catalyst 2960-24TT, 3x Server, 3x PC

To create these files, follow `build-guides/` phases 1 through 7 in order, then save twice with File → Save As.
