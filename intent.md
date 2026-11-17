# Intent — CPAP_Hanger

> Drafted by Claude 2026-11-16 from Bradley's request; Bradley owns this file — edit freely.

Goal:
Two-ended hanging hook for a CPAP hose/mask: a large lower hook and a smaller
upper hook joined by a straight shank. Rough design is done in
`CPAP_Hanger.FCStd`; this phase is tuning the hook diameters and making the
model fully parametric so it can be re-tuned after a test print.

Requirements (2026-11-16):

* Hook wall ("overall thickness of the hanging hook"): **7 mm** — radial wall width
* Small hook inside diameter: **17 mm** (remeasured; first pass was 22)
* Large hook inside diameter: **40 mm** (remeasured; first pass was 34)
* Small hook closes around (grabs) the CPAP flex hose: it wraps >180°, and the actual entry gap is 1 mm smaller than its inside diameter
* Keep the top-edge chamfer
* Pad depth 5 mm (Bradley changed it from 15 in the GUI, 2026-11-16)

Constraints:

* Must follow CAD_STANDARDS.md
* Fully parametric (all dimensions bound to Params.FCStd VarSet)
* Printable flat on the bed without supports
