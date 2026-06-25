# Phase 9 — DS-TE Priority + Preemption: Verification Log

**Date:** 2026-06-25
**Objective:** prove the one case where bandwidth makes a tunnel go **down** — a
higher-priority tunnel preempting a lower-priority one off a link that's out of
reservable bandwidth.

**Result:** ✅ **PASS** — `tunnel-te6` (setup priority 0) preempted `tunnel-te5`
(hold priority 7) on the R1→R3 link; te5 went down with an admission/bandwidth PathErr,
and the 600M reservation moved from the priority-7 row to the priority-0 row.

---

## Priority model (the concept)

Each tunnel has **setup** and **hold** priority, `0` (best) to `7` (worst), default `7 7`.
- **Setup** = how aggressively it preempts others when establishing.
- **Hold** = how well it resists being preempted once up.
- **Rule:** a new tunnel preempts an existing one when
  `new setup-priority < existing hold-priority` (lower number wins).

Demo: two tunnels forced over the same path (R1→R3→R4), each wanting 600M on a 1G link —
only one fits. `te5` = `priority 7 7`, `te6` = `priority 0 0`.

---

## Bonus lesson — the default-affinity trap struck the NEW tunnels too

First attempt: both tunnels went **down** before any bandwidth fight, with:
```
Bandwidth: 70000  Priority: 7 7  Affinity: 0x0/0xffff
Last PCALC Error: No path to destination (node unreachable)
```
`te5`/`te6` inherited the **default affinity `0x0/0xffff` = "uncolored links only,"** and
the south path is BRONZE-colored (Phase 7). Same trap as `tunnel-te1` in Phase 7b — every
*new* tunnel hits it in a colored network. **Fix:** `affinity 0x0 mask 0x0` on both, then
bumped bandwidth to 600M to force real contention on the 1G link.

---

## Step A — low-priority te5 up alone

```
show mpls traffic-eng tunnels 5 brief
   tunnel-te5  4.4.4.4  up  up

show mpls traffic-eng link-management bandwidth-allocation   (Gi0/0/0/3, the R1->R3 link)
   Max Reservable BW: 1000000 (reserved: 60%)
   KEEP PRIORITY   BW LOCKED
        7            600000        <-- te5 holds 600M at priority 7
```

## Step B — high-priority te6 released → preemption

```
show mpls traffic-eng tunnels brief
   tunnel-te1  up
   tunnel-te2  up
   tunnel-te5  DOWN      <-- evicted
   tunnel-te6  up        <-- took the bandwidth

show mpls traffic-eng tunnels 5 detail
   Last Signalled Error: PathErr(1,2)-(admission (1), bandwidth unavailable (2)) at 1.1.1.1
   Last PCALC Error: No path to destination, 4.4.4.4 (bw)
   Soft Preemption: Disabled          <-- HARD preemption (torn immediately)

show mpls traffic-eng link-management bandwidth-allocation   (Gi0/0/0/3)
   KEEP PRIORITY   BW LOCKED
        0            600000        <-- te6 now holds 600M at priority 0
```

The 600M reservation moved **priority 7 → priority 0** — the link's bandwidth changed
hands from te5 to te6.

---

## Analysis — answering "does too much traffic drop packets or drop the tunnel?"

- **A reservation is admission-control bookkeeping, not a data-plane policer.** Sending
  more than the reserved rate drops nothing and keeps the tunnel up (you need a separate
  QoS policer/shaper to enforce a rate).
- **The only way bandwidth takes a tunnel *down* is preemption:** when the link's
  reservation books are full and a higher-priority tunnel needs the room, it tears down a
  lower-priority one. `te6 setup 0 < te5 hold 7` → te5 preempted. ✔
- **The preemption signature** is the RSVP `PathErr` with `admission (1) / bandwidth
  unavailable (2)`, followed by the preempted tunnel failing to re-establish: `No path
  (bw)`. With no fallback path, te5 stayed down. ✔
- **Hard vs soft:** default is **hard** preemption (immediate tear). `soft-preemption`
  (configurable, network-wide + per-tunnel) lets the victim reroute gracefully first.

> In production you give critical services (voice, signalling) low priority numbers so
> they can preempt best-effort tunnels under contention — and you give best-effort tunnels
> a dynamic fallback so being preempted means *reroute*, not outage.

**Conclusion:** priority + preemption is the mechanism by which bandwidth contention
resolves — and the only place a tunnel goes down over bandwidth. Phase 9 complete.

---

## Cleanup (te5/te6 are teaching tunnels; restore the link)

```
configure
no interface tunnel-te5
no interface tunnel-te6
no explicit-path name SOUTH-R1-TO-R4
rsvp interface GigabitEthernet0/0/0/3
 bandwidth 1000000        ! ensure full reservable restored
root
commit
```
