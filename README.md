# CPAP_Hanger

Parametric two-ended hanging hook for CPAP hose / mask, designed in FreeCAD 1.1.

| Param (Params.FCStd → VarSet) | Default |
|---|---|
| `HookWall` | 7 mm |
| `LargeHookInnerD` | 40 mm |
| `SmallHookInnerD` | 17 mm |
| `HoseSnapInterference` | 1 mm (small-hook hose entry gap = Ø minus this; the hook wraps >180° to clip the hose) |
| `HangerDepth` | 15 mm |
| `ChamferSize` | 1 mm |

To tune, open `Params.FCStd`, edit the VarSet, and recompute `CPAP_Hanger.FCStd`.

Layout: FCStd sources at the root · `stl/` and `3mf/` print deliverables · `macros/` model changes · `scripts/` audit + env guard. See `CLAUDE.md` and `plan.md`.
