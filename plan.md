# Plan — CPAP_Hanger parameterization (rev 1, 2026-11-16)

**Status: APPROVED 2026-11-16 (tangent shank yes). Diameters remeasured by Bradley: large 40, small 17.**

## Current state (observed via MCP, FreeCAD 1.1.4 GUI, 2026-11-16)

`CPAP_Hanger.FCStd`: `Body` → `Sketch` (on XY_Plane, DoF 0) → `Pad` (15 mm) → `Chamfer` (1 mm, `Pad.Face10` = top face). Every dimension is a literal; nothing is bound.

| Sketch geo | Role | Now |
|---|---|---|
| 0 | small hook tip cap | R3.5 (derived from tangency) at (−16, 70) |
| 1 / 2 | small hook outer / inner arc | R19 / R12 → inside Ø **24** |
| 3 | large hook tip cap | R3.5 (derived), centre on X axis |
| 4 / 5 | large hook outer / inner arc | R30 / R23 → inside Ø **46**, centre = origin |
| 6 / 7 | shank lines | endpoint-**Coincident** (not tangent) to the arcs |

**Chamfer is already broken.** Pad solid is valid. Chamfer solid fails `isValid()` (`check()`: self-intersecting wires, unorientable shape). In-memory tests (no doc change): chamfering the top face edges fails at 1.0, 0.8 and 0.5 mm, and the line edges fail even individually. The four line↔arc joints are kinks of 6–17° (tangent angles 169.8°, 5.7°, 163.6°, 7.2°). Near-tangent kinks are a known OCC chamfer failure mode. *Inference:* making those joints truly tangent fixes the chamfer. This is to be verified in step 3, not assumed.

## PARAMETERS (new `Params.FCStd` → `VarSet`)

| Name | Type | Default | Drives |
|---|---|---|---|
| `HookWall` | Length | 7 mm | outer R − inner R, both hooks (caps follow at Wall/2 via tangency) |
| `LargeHookInnerD` | Length | **40 mm** | large inner R = D/2; outer R = D/2 + HookWall |
| `SmallHookInnerD` | Length | **17 mm** | small inner R = D/2; outer R = D/2 + HookWall |
| `SmallTipCenterX` | Length | 16 mm | small tip cap centre, left of Y axis |
| `SmallTipCenterY` | Length | 70 mm | small tip cap centre height |
| `ShankTopY` | Length | 53.28 mm | height where the shank meets the small hook's outer arc |
| `HangerDepth` | Length | 15 mm | Pad.Length |
| `ChamferSize` | Length | 1 mm | Chamfer.Size |

The four arc-sweep Angle constraints (211.5°/208.8°/189.4°/178.4°) are **removed, not parameterized**. Tangency (below) determines them, which is the actual design intent: the hook follows the diameters.

## FEATURE TREE (unchanged order)

1. `Sketch` on XY_Plane (origin datum, not a feature face ✅)
2. `Pad` — Length = `HangerDepth`
3. `Chamfer` — top face, Size = `ChamferSize` (kept, per Bradley)

## CONSTRAINT STRATEGY — `macros/parameterize_hanger.FCMacro`

1. **Tangent shank:** replace the 4 line↔arc endpoint Coincidents with endpoint Tangents and delete the 4 sweep Angles. DoF-neutral (−12 / +12 equations). This visibly smooths the 4 shank joints slightly.
2. Name and bind: `LargeOuterR`, `LargeInnerR`, `SmallOuterR`, `SmallInnerR`, `SmallTipY`, `ShankTopY`, `SmallTipX`.
3. Bind `Pad.Length` and `Chamfer.Size`.

Constraints are located by signature, not index. The macro aborts if the topology differs. It is idempotent and does not save.

## EXECUTION (after approval)

1. Run `create_params.FCMacro` → creates and saves `Params.FCStd` (a save is required before cross-doc expressions resolve).
2. Run `parameterize_hanger.FCMacro`. The Params already default to the *new* diameters (40 / 17), so this one step produces the target geometry.
3. Verify: the sketch has DoF 0, FullyConstrained, no redundant/conflicting/malformed constraints. Inner Ø reads 40 / 17 and wall reads 7. **Chamfer `isValid()` is True** and `check()` is clean. Take a screenshot for Bradley's eyeball check of the shank shape.
4. Flex test (rule 10): change `LargeHookInnerD` to 46, recompute, and confirm no `?EdgeN`/`?FaceN` and a valid Chamfer. Restore to 40 and recompute.
5. Run `python3 scripts/audit_parametric.py` → must be clean.
6. **Save only on Bradley's OK.**

**Fallback if tangency doesn't fix the Chamfer:** undo, report the evidence and propose options (e.g. chamfer a specific edge set, or a Fillet on the top edges with a flex test). No silent workaround.

## VALIDATION

- `TEST ENVIRONMENT VERIFIED` banner quoted from every FreeCAD-side check
- Sketch: `DoF == 0`, `FullyConstrained`, empty Redundant/Conflicting/Malformed (not `solve()==0` — rule 9)
- Measured inner radii from `sk.Geometry` = 20 / 8.5; outer = 27 / 15.5
- Body solid valid; a single solid
- Audit script clean
- Then: STL → `stl/`, 3MF → `3mf/` (after the save is approved)

## REV 2 (2026-11-16, Bradley): small-hook throat

- The small-hook throat must be `SmallHookInnerD - 1 mm` so the flex hose snaps in.
- New Param `HoseSnapInterference` = 1 mm. New constraint `SmallThroat`, a Distance from the lip (2/2) to the joint (6/1).
- It replaces `ShankTopY` (DoF-neutral swap). The `ShankTopY` Param was deleted as dead.
- Result: throat 16.000 (true min, verified with distToShape), ID 40/17, wall/shank 7, DoF 0. The Chamfer is valid.
- Flex test: SmallHookInnerD 20 -> throat 19, HoseSnapInterference 1.5 -> 15.5. Both stayed valid.

## REV 3 + 4 (2026-11-16, Bradley): the small hook must GRAB the hose

- **rev 3:** rev 2's chord constraint solved to the open branch (140.5 deg wrap), so the hook didn't close. A wrap Angle of 219.5 deg closed it, but the real entry gap measured 14.26 mm.
- **rev 4 (Bradley chose "actual entry gap = 16"):** `SmallHookGap` is a point-to-line Distance from the tip-cap centre to the shank inner edge = `SmallHookInnerD - HoseSnapInterference + HookWall/2`. Wrap is derived: 203.56 deg.
- **Verified:** true min gap 16.000, ID 40/17, wall/shank 7, DoF 0, Chamfer valid.
- **Flex test:** SmallHookInnerD 20/15 gives a gap of 19/14. HoseSnapInterference 2/0.5 gives a gap of 15/16.5 and a wrap of 213.6/196.6. LargeHookInnerD 46 leaves the gap unchanged. Everything stayed valid and was restored.
- `HangerDepth` went to 5 mm for a quick test print, which worked; Bradley set it back to 15 mm in the GUI (2026-11-16).

## OPEN QUESTIONS

- Approve **tangent shank** (needed to fix the Chamfer)? If no, the Chamfer stays invalid, and I would parameterize the 4 sweep angles instead.
- The large hook gets much smaller (Ø46 → Ø40), so the shank re-solves. If the new shape looks off in the screenshot, `SmallTipCenterX/Y` are the knobs (`ShankTopY` was removed in rev 2).
- ~~Git remote~~ — done 2026-11-16: public repo https://github.com/bcorner13/CPAP_Hanger (`main` + `develop`).
- ~~What does the large hook hang on?~~ The bed frame rail (Bradley, 2026-11-16).
