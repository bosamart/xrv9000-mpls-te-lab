# MPLS-TE Lab — Handoff for Next Chat

**Last updated:** 2026-06-25 · **Owner:** Bo Sam Ath (ATH)
**Read this first, then the lab [`README.md`](README.md) and [`notes/`](notes/).**

---

## TL;DR status

The **xrv9000-mpls-te-lab is complete and verified** — 9 phases (plus a 7b sub-lab),
all with real `show`-output verification logs. Fully documented (README + SVG diagram +
beginner guide + concept docs) and on GitHub. Audited **twice**; configs are technically
correct and internally consistent.

- **Local:** `C:\Users\ATH\Claude\Projects\Network-Engineering\Labs\MPLS-TE`
- **GitHub:** https://github.com/bosamart/xrv9000-mpls-te-lab
- **Platform:** Cisco IOS-XRv9000 24.3.1 on EVE-NG, single IS-IS **L2 area 49.0001**,
  the shared "diamond" topology (R1/R4 = PE, R2/R3 = P, CE1/CE2 = customer).

---

## Phase status (all ✅ verified)

| Phase | Topic | Log |
|-------|-------|-----|
| 1 | IS-IS baseline | notes/phase1-isis-verify.md |
| 2 | LDP | notes/phase2-ldp-verify.md |
| 3 | RSVP-TE explicit-path tunnel (scenic R1→R2→R3→R4) | notes/phase3-rsvp-te-verify.md |
| 4 | CSPF + autoroute | notes/phase4-cspf-autoroute-verify.md |
| 5 | FRR (auto-tunnel backup, node protection at R2) | notes/phase5-frr-verify.md |
| 6 | L3VPN over MPLS-TE (CUST-A) | notes/phase6-l3vpn-verify.md |
| 7 | Affinity coloring (GOLD north / BRONZE south) | notes/phase7-affinity-verify.md |
| 7b | explicit+affinity+`verbatim`, default-affinity trap | notes/phase7b-explicit-affinity-verify.md |
| 8 | Auto-bandwidth (tunnel-te1) | notes/phase8-autobw-verify.md |
| 9 | DS-TE priority + preemption | notes/phase9-preemption-verify.md |

**Base config vs experiments:** `configs/R1-R4.txt` + `CE1/CE2.txt` carry the *base*
build (Phases 1–8 features). **Phase 7b tunnels (te3/te4) and Phase 9 tunnels (te5/te6)
are throwaway experiments run from the README — NOT in the base configs.**

---

## OPEN ITEMS (do these / be aware)

1. **Git push may be pending.** Last session hit a stale `.git/index.lock`
   (`fatal: Unable to create '.git/index.lock'`). Fix on ATH's PowerShell:
   close VS Code (its git extension can hold the lock), then
   `Remove-Item .git\index.lock`, then `git add -A` / `git commit -m "..."` /
   `git push --force`. Confirm the audit + doc-refresh commits actually landed on GitHub.

2. **The LIVE EVE router has drifted from the repo** (from the Phase 7b/8/9 experiments):
   - R2 lost `mpls traffic-eng interface GigabitEthernet0/0/0/1` (+ its `auto-tunnel
     backup`) on the cross-link — the **repo `R2.txt` is correct**, the running box isn't.
   - Leftover teaching tunnels **te3, te4, te5, te6** may still exist on R1.
   - `auto-bw` is active on tunnel-te1 (resizes to 30M floor under low traffic).
   - `rsvp interface Gi0/0/0/3 bandwidth` may have been changed during Phase 9.
   - **To restore a clean state matching the repo:** re-paste `R2.txt`, then run the
     Phase 7b + Phase 9 cleanup blocks from the README (`no interface tunnel-teN`,
     restore `rsvp ... bandwidth 1000000`).

3. **Git workflow reminder:** run git in ATH's **PowerShell 5.x** (no `&&` chaining),
   NOT the sandbox. Don't `git clone`/rename in the sandbox — recreate files with the
   Write tool. The `.unl` is **gitignored** (kept local, not published).

---

## Key gotchas / lessons (so you don't relearn them the hard way)

- **Default tunnel affinity is `0x0/0xffff` = "uncolored links only."** Once links are
  colored (Phase 7), any tunnel without its own affinity goes down with `No path
  (affinity)`. Fix: `affinity 0x0 mask 0x0`. (tunnel-te1 already has this in the repo.)
- **A link needs THREE independent things for TE:** IS-IS adjacency, `rsvp interface`,
  AND `mpls traffic-eng interface`. Any one missing = a different failure. **Read the
  error type:** `PCALC Error` = CSPF/topology (link advertised? affinity? bw?);
  `Signalled Error` = RSVP along the path.
- **Auto-tunnel backup needs `ipv4 unnumbered mpls traffic-eng Loopback0`** or every
  bypass sits down with "No IP source address is configured." (On R2/R3/R4 in the repo.)
- **Node protection only works at R2** (R3's next hop is the tail R4 — no NNHOP to bypass to).
- **FRR test:** shut the ON-PATH link the LSP uses (`R2 Gi0/0/0/1`, the cross-link), not
  an off-path one.
- **IOS-XR: `!` is a COMMENT, not a submode exit.** Use `exit` / `root`. (Bit us on
  `interface tunnel-teN`, which is global config, not under `mpls traffic-eng`.)
- **CE2↔R4 wiring must be `R4 Gi0/0/0/1 ↔ CE2 Gi0/0/0/0`** in EVE (a mismatch here broke
  Phase 6 L3VPN once). XRv shows `Up/Up` even on an unconnected NIC — don't trust it, ping.
- **Tooling:** the Edit tool has occasionally injected NUL bytes into `.md` files (makes
  git see them as binary) — a quick `tr -d '\000'` / python strip cleans it. The sandbox
  mount can **lag** file-tool writes, so verify with the Read tool, not bash, after edits.

---

## Repo contents

```
README.md                     9-phase walkthrough, SVG topology, TL;DR
HANDOFF.md                    this file
.gitignore                    ignores *.unl, OS/editor junk, push-* scripts
configs/  R1-R4, CE1, CE2     cumulative base configs (Phases 1-8 features)
diagrams/topology.svg         full-detail diamond w/ GOLD/BRONZE coloring
docs/  BEGINNER-GUIDE.md       concepts + guided build (start here if new)
       CONCEPTS.md             the "why" per phase + interview Q&A
       PARAMETERS.md           every knob explained, tagged by scope
       TE-Config-Guide.md      copy-paste command cheat-sheet (all phases)
       RSVP-TE-vs-SR-TE.md     full comparison to the SR-MPLS lab
       INTER-AREA-TE-and-PCE.md  concept deep-dive (not labbed - needs multi-area)
notes/  phase1..9 + 7b logs    real show-output verification per phase + notes/README.md tracker
xrv9000-mpls-te-lab.unl       EVE topology (LOCAL only, gitignored)
```

---

## Possible next directions (ATH's call)

- **Advanced RSVP-TE still un-labbed:** path protection + SRLG (disjoint secondary),
  BFD-for-TE (sub-second detection), soft preemption. **Inter-area TE** needs an IGP
  redesign (split diamond into L1 areas + L2 backbone) — that's a *separate* lab; the
  concept is already written up in `docs/INTER-AREA-TE-and-PCE.md`.
- **Other labs in the project:** `BGP-Advanced` and `IGP-Deep-Dive` are built but
  **not yet tested in EVE-NG**. Then the **Automation** track (Python/Netmiko/NAPALM).
- Per the roadmap, suggested order rides the IGP: IGP-Deep-Dive → BGP-Advanced → SR/TE.

---

## Working with ATH (from project CLAUDE.md)

- Goal: move from 8 yrs of template/copy-paste to understanding **why** — teach the "why",
  quiz/checkpoint, be honest about the roadmap. Starting at **Cellcard** (L2 IP ops, ~6 Jul 2026).
- Style: **concise, direct, kind.** Prose over heavy bullets/bold.
- **Conserve tokens:** confirm BEFORE usage-heavy actions (sub-agents, big web research,
  many/large files, long automated workflows). Prefer the cheapest approach that works.
- One chat per lab; the file system is the only shared state between chats.
```
