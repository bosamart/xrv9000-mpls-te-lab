# Phase 8 — Auto-Bandwidth: Verification Log

**Date:** 2026-06-25
**Objective:** let `tunnel-te1` size its own RSVP reservation from measured traffic
instead of a hardcoded number, within a min/max range.

**Result:** ✅ **PASS** — after one application period, auto-bw resized the reservation
from the static 100000 kbps down to the 30000 kbps floor (measured traffic was light),
re-signalling make-before-break (LSP ID 5 → 6).

---

## Config (R1, on tunnel-te1)

```
interface tunnel-te1
 signalled-bandwidth 100000          ! starting value (auto-bw takes over from here)
 auto-bw
  application 5                      ! resize every 5 min (minimum; default 1440 = 24h)
  bw-limit min 30000 max 500000      ! never reserve below 30M or above 500M
  adjustment-threshold 5             ! only resize if measured rate differs > 5%
```

> Lab note: `application 5` (the minimum) is set so the resize is observable in minutes.
> Production typically uses hours (or the 24h default) to avoid churn.

---

## Before — period running

```
RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels auto-bw brief
   Tunnel       LSP  Last appl  Requested  Signalled  Highest  Application
   Name          ID   BW(kbps)   BW(kbps)   BW(kbps)  BW(kbps)   Time Left
   tunnel-te1     5          -     100000     100000        0     2m 31s
```
- `Requested/Signalled 100000` = starting reservation (the old static value).
- `Highest 0` = peak rate sampled so far this period (≈ no traffic).
- `Last appl -` = hasn't resized yet.

## After — period elapsed

```
RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels auto-bw brief
   Tunnel       LSP  Last appl  Requested  Signalled  Highest  Application
   Name          ID   BW(kbps)   BW(kbps)   BW(kbps)  BW(kbps)   Time Left
   tunnel-te1     6      30000      30000      30000        0     4m 39s

RP/0/RP0/CPU0:R1#show mpls traffic-eng tunnels 1 detail | include Bandwidth
    Bandwidth Requested: 30000 kbps  CT0
    Bandwidth:   100000 kbps (CT0) Priority: 7 7      <-- configured base (unchanged)
      Bandwidth Min/Max: 30000-500000 kbps
```
- `Signalled/Requested/Last appl = 30000` — resized to the **min floor**.
- `LSP ID 5 → 6` — new LSP signalled (make-before-break), old one torn after cutover.
- Timer reset to a fresh ~5 min.

---

## Analysis

- **Auto-bw right-sized the reservation.** Measured peak (`Highest BW`) was ~0, so the
  100000 guess was dropped to the 30000 **min** — it reserves what it needs, never less
  than the floor. ✔
- **No outage on resize** — the LSP ID incrementing (5→6) shows make-before-break: the
  new-bandwidth LSP came up before the old one went away. ✔
- **Config base vs live value** — `Bandwidth: 100000` (the configured `signalled-bandwidth`)
  stays as the starting point; `Bandwidth Requested: 30000` is what auto-bw actually
  asks the network to reserve now. Auto-bw owns the live value within min/max. ✔

**Why it dropped instead of climbing:** a virtual lab can't push >30 Mbps, so the
measured peak never approached the floor. In production carrying real load, the same
mechanism grows the reservation toward `max` during busy hours and shrinks it back when
traffic eases — automatically.

### Key fields to read (ignore the rest)
`Requested/Signalled BW` (live reservation) · `Highest BW` (measured peak this period) ·
`Last appl BW` (value set at last resize) · `Application Time Left` (countdown) ·
`Bandwidth Min/Max` (the clamps).

**Conclusion:** Auto-bandwidth resizes the tunnel reservation from real measurement,
make-before-break, within configured bounds. Phase 8 complete.

---

## Optional cleanup / revert to static
```
interface tunnel-te1
 no auto-bw          ! reservation returns to the static signalled-bandwidth 100000
commit
```
