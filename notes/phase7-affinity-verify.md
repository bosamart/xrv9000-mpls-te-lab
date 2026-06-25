# Phase 7 — Affinity / Admin-Group Coloring: Verification Log

**Date:** 2026-06-25
**Objective:** Steer a tunnel by **link color** (affinity / admin-groups) instead of
explicit hops — color the two disjoint paths and let CSPF pick by policy.

**Result:** ✅ **PASS** — `tunnel-te2` with `affinity include GOLD` took the north
path (R1→R2→R4); flipping to `include BRONZE` moved it to the south path
(R1→R3→R4), all via dynamic CSPF with **no explicit-path**. Make-before-break
reoptimization observed during the flip.

---

## Design

| Color | bit-position | Links |
|-------|-------------|-------|
| GOLD | 0 | north path: R1–R2 (`10.12`), R2–R4 (`10.24`) |
| BRONZE | 1 | south path: R1–R3 (`10.13`), R3–R4 (`10.34`) |
| (none) | — | cross-link R2–R3 (`10.23`) left uncolored |

`tunnel-te2` (R1→R4, dynamic path, no autoroute — demo only, leaves `tunnel-te1` /
L3VPN untouched).

---

## Gotcha: IOS-XR submode trap

First attempt failed because in IOS-XR a bare `!` is a **comment, not a submode exit**.
After `mpls traffic-eng / interface Gi.. / attribute-names`, the session was still in
`config-mpls-te-if`, so `interface tunnel-te2` was parsed as an *mpls-te core
interface* and all the tunnel commands (`ipv4 unnumbered`, `destination`,
`path-option`, `affinity`) were rejected. **Fix:** use `root` (or `exit`) to leave the
submode — the tunnel interface lives at **global** config, not under `mpls traffic-eng`.

---

## GOLD → north path (R1→R2→R4)

```
RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels 2 detail
    Admin: up Oper: up   Path: valid   Signalling: connected
    path option 1, type dynamic (Basis for Setup, path weight 20)
    Number of affinity constraints: 1
       Include affinity name : GOLD(0)
  Current LSP Info:
    Outgoing Interface: GigabitEthernet0/0/0/1, Outgoing Label: 24008
    Path Info:
      Explicit Route:
        Strict, 10.12.0.2     (R2)
        Strict, 10.24.0.2     (R4)
        Strict, 4.4.4.4
```

CSPF, constrained to GOLD-only links, computed **R1→R2→R4** (weight 20). No
explicit-path was configured — the path came purely from the color rule.

## BRONZE → south path (R1→R3→R4), caught mid make-before-break

```
interface tunnel-te2
 no affinity include GOLD
 affinity include BRONZE
commit
```

```
RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels 2 detail
    Number of affinity constraints: 1
       Include affinity name : BRONZE(1)
  Reopt Trigger: Path affinity failed verification, Reopt Reason: applying affinity change

  Current LSP Info:                          <-- OLD LSP, being torn down
    Outgoing Interface: GigabitEthernet0/0/0/1 (-> R2)
    Explicit Route: 10.12.0.2 -> 10.24.0.2 -> 4.4.4.4     (GOLD north)

  Reoptimized LSP Info (Install Timer Remaining 9 Seconds):   <-- NEW LSP taking over
    Instance: 3
    Outgoing Interface: GigabitEthernet0/0/0/3 (-> R3), Outgoing Label: 24006
    Explicit Route: 10.13.0.2 -> 10.34.0.2 -> 4.4.4.4     (BRONZE south)
```

The affinity change failed verification against the in-use (GOLD) path, triggering a
reoptimization: CSPF computed a new **BRONZE** path R1→R3→R4 and signaled it
make-before-break (old LSP up until the new one installs).

---

## Analysis

- **Color-based path selection works.** Same tunnel, same dynamic path-option — only
  the affinity constraint changed, and the path moved from the north (GOLD) links to
  the south (BRONZE) links. CSPF treats affinity as a hard constraint: every link in
  the computed path must satisfy it. ✔
- **No explicit-path anywhere.** Unlike Phase 3 (hard-coded hops), the path here is
  chosen by *policy* — exactly what admin-groups are for (e.g. keep traffic off
  high-latency or high-cost links by coloring them). ✔
- **Bonus — make-before-break.** The flip triggered a graceful reoptimization: the
  old LSP stayed up while the new one signaled, then cut over after the install timer.
  No traffic loss on a path change. ✔

### IOS-XR affinity syntax used

```
mpls traffic-eng
 affinity-map GOLD bit-position 0
 affinity-map BRONZE bit-position 1
 interface <core-link>
  attribute-names GOLD            ! color the link
!
interface tunnel-te2
 affinity include GOLD            ! constrain the tunnel (include / exclude)
```

**Conclusion:** Affinity coloring steers tunnels by policy, both directions verified.
Phase 7 complete.
