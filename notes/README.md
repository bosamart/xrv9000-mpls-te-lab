# MPLS-TE Lab Notes

## Lab Goals
- Understand classic MPLS Traffic Engineering with RSVP-TE
- Compare RSVP-TE with SR-TE from the SR-MPLS lab
- Build foundation for understanding why operators moved to Segment Routing

## Topics Covered
- [x] Phase 1 — IS-IS baseline
- [x] Phase 2 — LDP label distribution
- [ ] Phase 3 — RSVP-TE basic tunnel with explicit path
- [ ] Phase 4 — CSPF path computation + autoroute
- [ ] Phase 5 — FRR link and node protection
- [ ] Phase 6 — L3VPN over MPLS-TE

## Lab Topology
> Same diamond as SR-MPLS lab: R1(PE) – R2(P) – R4(PE), R1 – R3(P) – R4, R2–R3 cross-link
> CE1 on R1, CE2 on R4

## GitHub Repo
https://github.com/bosamart/xrv9000-mpls-te-lab.git

## Related Lab
SR-MPLS lab: ../SR-MPLS/ — same topology, same services, SR transport instead of RSVP-TE

## Session Log
| Date | Phase | Topic | Outcome |
|------|-------|-------|---------|
| 2026-06-24 | Phase 1 | IS-IS baseline — all adjacencies up, loopbacks reachable, ECMP to 4.4.4.4 | ✅ Pass |
| 2026-06-24 | Phase 2 | LDP sessions up, labels distributed, traceroute shows label + PHP | ✅ Pass |

> Detailed per-phase output: [`phase1-isis-verify.md`](phase1-isis-verify.md), [`phase2-ldp-verify.md`](phase2-ldp-verify.md)

## Lessons Learned
- `show clns interface brief` is IOS, not IOS-XR — use `show isis interface brief` on XR.
- Configs are cumulative (all 6 phases), so Phase 2 forwarding already shows L3VPN
  labels (CUST-A aggregate, CE1 prefix) and `Tunnel=Yes` on interfaces. Expected.
- ImpNull (implicit-null) on directly-connected prefixes is what drives PHP.
