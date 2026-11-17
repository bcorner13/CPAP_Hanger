# Project rules — CPAP_Hanger

The primary risk in this project is the single profile sketch. Every dimension in the hook outline lives in one sketch, and the shank geometry is re-solved from the hook diameters. One literal edit, or one deleted tangency, silently reshapes the whole part. The top-face Chamfer is topologically fragile: it was already producing an invalid solid on 2026-11-16 because of kinked shank joints.

> **How to use this template:** Every `[FILL: …]` marker is a required edit. Leave none. A thin CLAUDE.md (one that just restates the global rules) is the failure mode this template exists to prevent — the point is to capture *this project's* specific topology, history, and current state, so the user does not have to re-explain it each session.

---

## Hard rules (this project)

These restate the global rules in `~/.claude/CLAUDE.md` with project-specific context. Cite the actual incident or constraint that motivates each one — generic restatements are useless because the global file already has them.

1. **Everything parametric.** The worst offender is `Sketch` (the whole hanger outline). As of 2026-11-16, all 7 of its dimensional constraints plus `Pad.Length` and `Chamfer.Size` were literals. The binding work is in `macros/parameterize_hanger.FCMacro` (plan.md rev 1). Hook sizes are tuned **only** through `HookWall`, `LargeHookInnerD`, `SmallHookInnerD` and `HoseSnapInterference`.

2. **No fixing geometry by editing raw sketch coordinates.** No prior incident in this project; the rule applies preventively. The end-cap radii (R = HookWall/2) and the shank lines have **no** dimensional constraint of their own. Tangency derives them. Don't "fix" them by adding a radius or moving points.

3. **Attach sketches to datum planes, not feature faces.** No prior DAG incident; the rule applies preventively. `Sketch` is on the Body Origin's `XY_Plane` ✅.

4. **Clearance concepts stay decoupled.** No mating printed parts yet, so there are no clearance Params. If a hose/mask fit clearance is ever needed, add one Param per interface (e.g. `HoseHookClearance`). Never fold it into `SmallHookInnerD`.

---

## Assembly architecture

Single printed part, no assembly. The profile lies in the XY plane and is padded +Z by `HangerDepth`. It prints flat, chamfered face up.

- **Large hook** (bottom): centred at the origin, inner Ø `LargeHookInnerD`, opening to the right; a round tip cap at +X on the X axis.
- **Small hook** (top-left): inner Ø `SmallHookInnerD`, tip cap centre at (−`SmallTipCenterX`, `SmallTipCenterY`).
- **Shank**: two straight lines joining the large hook's outer/inner arcs to the small hook's inner/outer arcs, with endpoint **tangents** (rev 1). They come out parallel and exactly `HookWall` apart.
- **Small hook = hose clip** (`SmallHookGap` constraint). The hook wraps **> 180 deg** (about 203.5 deg at 17/1) so it closes around the CPAP flex hose. The hose's **actual entry gap** is from the tip cap to the shank's inner straight edge (line 6). It's set to `SmallHookInnerD - HoseSnapInterference` (16 mm) by a point-to-line Distance: tip-cap centre (0/3) to line 6 = gap + `HookWall`/2. The wrap angle is derived.
  - **Don't swap this for a chord or an angle.** rev 2 used a lip-to-joint chord; a chord fits two wraps, so the solver took the open ~140 deg branch and the hook didn't close. rev 3 used a wrap angle of 219.5 deg; it closed, but the real entry gap was 14.26 mm. `parameterize_hanger` prints wrap, chord and true min gap; check that the true gap equals the target.
  - When migrating, the macro first walks the solver onto the closed branch with a temporary wrap angle, then steps the gap. A point-to-line distance has a mirror solution, so don't jump big values in one go.
- Wall width is `HookWall` everywhere (both hooks; caps are HookWall/2 by tangency).

The **small hook** grips the CPAP flex hose (snap-in through the undersized throat). The **large hook** hangs on the **bed frame rail** (Bradley, 2026-11-16). `LargeHookInnerD` (40) is the knob if a different rail doesn't fit. The rail is a hang-over fit, not a clip, so it has no clearance Param. If it ever needs one, add `RailHookClearance` rather than reusing `HoseSnapInterference`.

---

## Files in this project

| File | Role | Depends on | Status |
|---|---|---|---|
| `Params.FCStd` | VarSet — all parametric variables | — | ✅ created 2026-11-16 by `create_params.FCMacro` |
| `CPAP_Hanger.FCStd` | Hanger body (doc internal name `Unnamed`, label `CPAP_Hanger`) | `Params.FCStd` (after rev 1) | ✅ fully bound, Chamfer valid, audit clean — saved 2026-11-16 |

**CPAP_Hanger.FCStd history:** before rev 1, the Chamfer (`Pad.Face10`, top face, 1 mm) recomputes "Up-to-date" but its shape fails `isValid()` (self-intersecting wires / unorientable). This was diagnosed on 2026-11-16: the 4 shank line↔arc joints are 6–17° kinks, and the chamfer fails even per-edge and at 0.5 mm. Fixed by the rev 1 tangent joints; flex-tested at large Ø30–46 (Ø30 breaks the Pad). `3mf/` and `stl/CPAP_Hanger-BodyChamfer.*` were re-exported 2026-11-16 from the valid rev 4 solid (watertight). The 5 mm-deep test print worked, and they were re-exported again at the 15 mm production depth (54 × 111.67 × 15 mm, vol 20037 mm³).

---

## Params variables (summary)

`Params.FCStd` → `VarSet`, 8 variables (defined in `macros/create_params.FCMacro`):
- **Hook:** `HookWall` (7), `LargeHookInnerD` (40), `SmallHookInnerD` (17), `HoseSnapInterference` (1)
- **Layout:** `SmallTipCenterX` (16), `SmallTipCenterY` (70)
- (`ShankTopY` was removed in rev 2; `SmallHookGap` now holds that DoF.)
- **Body:** `HangerDepth` (15; set to 5 for the successful test print, then back to 15 by Bradley 2026-11-16), `ChamferSize` (1)

The macro never overwrites existing values, so tune in the VarSet.

---

## How to verify your change didn't break parametric

After any FreeCAD edit, before considering the task done:

```bash
python3 scripts/audit_parametric.py
```

This script flags:
- Sketches with 0 constraints
- Sketches with dimensional constraints lacking expression bindings
- Sketches attached to feature faces (DAG risk)
- Params variables used nowhere (dead Params)

The script is authoritative. If it reports violations, fix them via a `macros/*.FCMacro` change before saving or committing — never by direct coordinate edits or FCStd XML surgery.

No exemptions; the audit must be clean. (`scripts/audit_parametric.py` is the fully fixed k2-RFID-Case copy, with correct constraint/Pad enums and Hole support.) A clean audit does not prove the Chamfer is valid. Also check `Chamfer.Shape.isValid()`.

---

## Memory files (deeper context)

`~/.claude/projects/[encoded-path]/memory/MEMORY.md` indexes the persistent memories for this project. If you're unsure *why* a rule exists, read those files first — they trace each rule to a concrete past incident.

No project-scoped memories yet — this is a fresh project.

---

## Workflow notes

**Invariant (apply to every FreeCAD project — do not edit):**

- **Inspect/edit FreeCAD models via the MCP bridge — never with shell tools.** Do **not** `unzip`/`grep`/`cat`/`sed`/`strings`/etc. a `.FCStd`. Use the FreeCAD Robust MCP server: `get_connection_status` first, then `open_document`, `list_objects`, `inspect_object`, `execute_python`, and macros. This is **enforced** by a PreToolUse hook in `.claude/settings.json` (copied from the root `settings.template.json` during bootstrap) — raw shell access to `.FCStd` is blocked. Only fall back to read-only `unzip` if the MCP bridge is genuinely unreachable, and ask the user first.
- **MCP server auto-starts with FreeCAD.** If an `mcp__freecad__*` call fails, the right interpretation is "FreeCAD isn't running" — ask whether to launch it. Do **not** silently fall back to `unzip` + XML parsing, and do **not** retry the same MCP call.
- **Write changes as `macros/*.FCMacro` files**, not direct XML edits. Reasons: reviewable, re-runnable, idempotent-friendly, uses FreeCAD's own serialization.
- **Cross-document expressions**: use the canonical form `<<Params>>#VarSet.VarName`. The shorter `<<Params>>.VarName` form sometimes fails with "Params not found."
- **Run `python3 scripts/audit_parametric.py` before committing.** If it reports violations, fix via macro, not by editing FCStd XML.
- **Stop means stop; only Bradley's laptop counts.** If the laptop's FreeCAD or MCP bridge is unreachable, the plan isn't approved, or required files are missing, the task is **blocked**: report it with the evidence and end the turn. Do not install, download, or run any other FreeCAD, MCP server, VM, sandbox, or headless instance, or take any other route to the result not described here, including routes this file doesn't mention. Every validation run (FreeCAD checks, sweeps, audit) calls `require_laptop()` from `scripts/test_env_guard.py` first and quotes its banner. A result without a `TEST ENVIRONMENT VERIFIED` banner is **invalid**, not partial evidence, and must never be reported as passing. Never edit, stub, or bypass the guard, and never run `--enroll`; only Bradley enrolls the machine.

**Project-specific:**

- The hanger document's internal name is `Unnamed` (label `CPAP_Hanger`). Macros find documents by file path, not name.
- The project-scoped `.claude/settings.local.json` enables the `freecad` MCP server for this folder.
- The PreToolUse guard also blocks Bash heredocs that mention `.FCStd` together with `cat` (bit `.gitignore` creation at bootstrap). Use the Write tool for such files.

---

## Print profile

**Test print at 5 mm depth succeeded (2026-11-16): the hose clip works.** Slicer setup from Bradley's PrusaSlicer screenshot (2026-11-16):

| Setting | Value |
|---|---|
| Slicer | PrusaSlicer (Expert mode) |
| Printer | Prusa CORE One L, HF0.4 nozzle |
| Filament | Prusament PETG @COREONE HF0.4 — Matt Blue |
| Print profile | 0.15mm BALANCED |
| Infill | 15% |
| Supports | For support enforcers only (none painted, so effectively none) |
| Brim | Off |
| Orientation | Flat, profile on the bed, Z = `HangerDepth` |
| File | `3mf/CPAP_Hanger-BodyChamfer.3mf`: 54 × 111.67 × 15 mm, vol 20037.31 mm³, 2588 facets, 1 shell, "No errors detected" |

The screenshot shows the 15 mm production export loaded with these settings. The successful test print was the 5 mm version; this assumes it used the same profile.
