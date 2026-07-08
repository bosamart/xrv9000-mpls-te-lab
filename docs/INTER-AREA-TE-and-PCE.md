# Inter-Area TE and PCE — Concept Deep-Dive

A reference companion to the lab. This lab is a **single IS-IS Level-2 area**, so it can't
demonstrate inter-area TE directly — but this is exactly the problem you hit in a real
operator (access / aggregation / core are usually separate areas for scale). This doc
explains *why* it's hard across areas and the two ways the industry solves it: **loose-hop
expansion** (classic RSVP-TE) and **PCE** (the modern, SR-friendly way).

---

## 1. Why it's a problem at all

TE works by flooding link attributes — available bandwidth, affinity colors, TE metric —
into the IGP using TE extensions (IS-IS TLVs / OSPF opaque LSAs). The catch:

> **That flooding is scoped to one area/level. A tunnel headend only ever sees the TE map
> of its *own* area.**

So when the headend runs CSPF to compute a constrained path, it can only compute *within
its own area*. The links in the next area simply aren't in its database — it can't see
their bandwidth, colors, or even their existence as TE links. Therefore:

> **A headend cannot compute an end-to-end CSPF path that crosses an area boundary.**

In this lab you never felt this, because every router is in one Level-2 area and the
headend (R1) sees the whole diamond. Add a second area and the headend goes blind past the
border.

**Why have multiple areas at all?** Scale and stability. Flooding every link-state change
to every router doesn't scale to thousands of nodes, and a flap in the access shouldn't
ripple into the core. So operators split the network into areas (IS-IS L1 areas + an L2
backbone, or OSPF areas) joined by **ABRs** (Area Border Routers). The price of that
scaling is exactly the TE-visibility problem above.

---

## 2. Solution A — Loose hops + per-area expansion (classic RSVP-TE)

Since the headend can't see the far area, it doesn't try to. Instead it specifies the
**ABRs as *loose* hops** and lets each area compute its own segment.

- A **strict** hop = "the very next hop must be exactly this, directly connected."
- A **loose** hop = "head generally toward this address; whoever is local works out the
  strict path to it."

```
explicit-path name TO-FAR-PE
 index 10 next-address loose ipv4 unicast <ABR1-loopback>
 index 20 next-address loose ipv4 unicast <ABR2-loopback>
 index 30 next-address loose ipv4 unicast <tail-PE-loopback>
```

**How it signals (ERO expansion):**
1. The headend computes a strict path *to ABR1* inside its own area, and sends the RSVP
   PATH with the remaining loose hops still in the ERO (Explicit Route Object).
2. PATH reaches ABR1. ABR1 now sits in the next area, runs CSPF there to ABR2 (the next
   loose hop), and **rewrites the ERO** with strict hops for that segment.
3. Repeat at each ABR until the tail PE. The result is one **contiguous** end-to-end LSP,
   computed piece by piece, each area filling in its own part.

This is RFC 5151 "contiguous LSP." Two cousins exist:
- **LSP stitching** (RFC 5150): build a separate TE LSP per area and *stitch* them into one
  forwarding path at the borders.
- **LSP hierarchy / nesting** (RFC 4206): carry the per-area LSPs *inside* a backbone LSP
  (a forwarding-adjacency tunnel). Common in big cores.

**The limitations (why this is the "classic, awkward" way):**
- **No global optimization.** Each area optimizes locally, so the end-to-end path can be
  worse than a globally-computed one. The headend can't compare alternatives it can't see.
- **FRR across a boundary is hard.** Node-protecting an ABR needs the next-next hop — which
  lives in another area the PLR can't see. (You met this limitation in miniature in Phase 5:
  R3 couldn't node-protect because its next hop was the tail. Area boundaries make it worse.)
- **Bandwidth admission is per-area** — there's no single view confirming the whole path
  has capacity before signalling; you find out hop-by-hop.
- **You must know the ABRs** (configure them as loose hops, or rely on auto-discovery).

It works, and it's deployed — but it's clearly straining. That's what PCE fixes.

---

## 3. Solution B — PCE (the modern way)

A **PCE (Path Computation Element)** is a server that *does* have the full multi-area
(even multi-domain) TE picture, computes the end-to-end path centrally, and hands it to the
router. The pieces:

- **PCC (Path Computation Client)** — the headend router. It asks the PCE for a path.
- **PCEP** — the protocol between PCC and PCE (request/reply, and more).
- **BGP-LS (BGP Link-State)** — how the PCE *learns* the topology. Routers export their
  IGP + TE database into BGP-LS; the PCE peers with BGP-LS and assembles a **single TED
  (Traffic Engineering Database) spanning all areas/domains.** This is the key: BGP-LS gives
  the PCE the cross-area visibility no single router has.

**Flavors:**
- **Stateless PCE** — computes a path on request and forgets it. Simple, but can't
  re-optimize or coordinate between LSPs.
- **Stateful PCE** — keeps live state of *every* LSP. It can re-optimize globally, enforce
  cross-LSP constraints (e.g. two LSPs must be **disjoint**), and react to topology changes.
- **PCE-initiated (PCInitiate)** — the PCE doesn't just answer; it *creates* and programs
  LSPs into routers. This is central control / SDN for the transport network.

**Why this is the direction of travel:** a stateful PCE with BGP-LS sees everything, so it
delivers globally-optimal, disjoint, bandwidth-aware paths *across* area and domain
boundaries — exactly what loose-hop expansion can't. And it pairs naturally with **SR-TE**,
where the headend just pushes a label stack (no RSVP state in the core), so the PCE only has
to compute and the routers only have to forward. SR + stateful PCE is how modern "anywhere
to anywhere" transport is built.

---

## 4. How this maps to a real operator

Mobile/SP networks are layered: **access** (many small nodes) → **aggregation** → **core**.
Each layer is often its own IGP area or even its own domain, for scale and blast-radius
control. A service LSP (say, mobile backhaul from a cell-site router to a core gateway) has
to cross all of that.

- The **classic** answer was inter-area RSVP-TE with loose hops, or **Seamless MPLS** —
  end-to-end label transport built with **BGP Labeled Unicast (BGP-LU)** to carry loopback
  reachability across domains, stitched hierarchically.
- The **current** answer is **SR (SR-MPLS or SRv6) + a stateful PCE fed by BGP-LS** — central
  computation, stateless core, end-to-end SR policies across every boundary. This is why your
  SR-MPLS lab matters: it's the foundation of where transport networks actually are now.

So inter-area TE is both a real thing you'll meet *and* a great illustration of why the
industry moved from RSVP-TE to SR — the same "less core state, central smarts" story you
already felt comparing the two labs.

---

## 5. Quick glossary

| Term | Meaning |
|------|---------|
| **ABR** | Area Border Router — sits between two IGP areas; the expansion point for loose hops |
| **Loose hop** | "Head toward here; local routers compute the strict path" — vs a strict (directly-connected) hop |
| **ERO** | Explicit Route Object — the list of hops in the RSVP PATH message; expanded at each ABR |
| **Contiguous / stitched / nested LSP** | The three inter-area LSP models (RFC 5151 / 5150 / 4206) |
| **PCE / PCC / PCEP** | Path Computation Element (server) / Client (router) / the protocol between them |
| **BGP-LS** | BGP Link-State — exports IGP+TE topology into BGP so a PCE can build a cross-area TED |
| **Stateful PCE** | A PCE that tracks all LSPs and can re-optimize / enforce global constraints |
| **TED** | Traffic Engineering Database — the TE topology a router (per-area) or PCE (multi-area) computes over |
| **Seamless MPLS** | End-to-end transport across domains using BGP-LU + hierarchy |

---

## 6. References

- RFC 4726 — Framework for inter-area / inter-AS MPLS-TE
- RFC 5151 — Inter-area MPLS-TE (contiguous LSP, loose-hop expansion)
- RFC 5150 — LSP stitching · RFC 4206 — LSP hierarchy (forwarding adjacencies)
- RFC 5440 — PCEP · RFC 8231 — Stateful PCE · RFC 8281 — PCE-initiated LSPs
- RFC 7752 — BGP-LS (North-Bound Distribution of Link-State and TE info)

*To actually lab this: split the diamond into IS-IS L1 areas + an L2 backbone with R2/R3 as
ABRs — that's a separate lab, since it changes the shared topology. This doc is the "why" to
read first.*
