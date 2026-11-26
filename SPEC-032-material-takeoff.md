# SPEC-032. Material takeoff: consumption by layers and materials

Status: **draft**, implementation required
Date: 2026-11-23
Project: BIMCraft 031 (Rust; `crates/core/src/schedule.rs` — `fields` :725, `Material` :191,
`standard` :163; `crates/core/src/library.rs`; `crates/core/src/model.rs` — `areas`/`volumes`
from SPEC-031; wall/floor/roof types with layers)
Source: task 2026-11-20: quantities of lumber, OSB, insulation and membranes had to be counted
by hand and moved into CAD tables — schedules give the area of an element, but not consumption
by layers.
Related: SPEC-031 (net areas with openings), SPEC-033 (panel framing), SPEC-035 (tables on
sheets), SPEC-029 (depth-based clipping).
Task type: a new kind of schedule and a calculation by layers; the model and the file are
extended additively.

> **In short.** Schedules in BIMCraft count elements: the area of a wall, the length of a beam,
> the area of an opening. But construction documents need **material takeoffs**: how much
> OSB-3 12 mm, how much mineral wool 195 mm, how much 50×195 timber, how much membrane — taking
> into account the layers of a type, openings, waste and density (mass). The data for this
> already exists in types (layers: material + thickness), and net areas with openings are
> provided by SPEC-031. What is missing is the calculation and the view.

## 0. General working rules

1. Language — Russian; code and strings — English; window strings — `msg!`/`t()`.
2. Tests — only on request; CI separately. `check`, `build`, `clippy` (without
   `--all-targets`), `fmt --check`, `i18n check` are allowed.
3. **Commits are atomic.** One commit — one logically complete change (one feature, one
   fix, one refactoring); unrelated changes must not be mixed. Each commit builds and does not
   break existing behaviour.
4. Commit order: model and calculation → view and render → command → waste/density → export →
   i18n/docs.
5. Before a commit — `git diff` and `git status`; do not capture build artifacts, temporary
   files or other people's changes.
6. After a push — wait for a green CI; if it fails, find the cause, fix it with a separate
   atomic commit and repeat until the run succeeds.
7. No new dependencies; `unsafe` is not added.
8. Old projects are read as is; new fields have `default`.

## 1. The essence of the problem

| Capability | Command 031 | Limitation |
|---|---|---|
| Element specification | `schedule {category: "walls"}` | one row per element, not per material |
| Element material | `Material` field (`schedule.rs:191`) | one material per element (layer of the structure) |
| Type layers | `type {layers: [...]}` | stored but not counted |
| Areas | `elements`/`schedule` | net, with openings (SPEC-031) |
| Openings | — | already accounted for in the area, must not be subtracted again |
| Mass | — | no density in the material |
| Waste | — | no waste percentage |

How it was counted by hand: 84.76 m² of panels × layers, 9.3 m³ of timber by sections,
44.2 m³ of mineral wool, 429 m² of OSB, 350 m² of membranes; all of it outside BIMCraft, with
a risk of error.

## 2. Goals

1. `takeoff {category|categories, by: material|layer|type, waste: %, level?}` — a consumption
   schedule: material, thickness, area, volume, mass, waste, total.
2. Account for **type layers** (walls/floors/roofs/ceilings): material, thickness, function.
3. Take the net area from SPEC-031; openings, shafts and slope (roofs and sloped floors) must
   not be subtracted a second time.
4. Material density — in the library (`density`), with reasonable default values
   (timber 500, OSB 650, mineral wool 35, gypsum board 900, membrane 1 kg/m²).
5. Grouping and totals; the schedule view is placed on a sheet and printed; XLSX/CSV export.
6. Refresh from the model (`refresh_takeoff`) and integration with SPEC-035 (tables) for
   arbitrary rows.

## 3. Non-goals

- No sheet/board cutting optimisation (separate task).
- No prices and estimates (quantities only).
- No per-section waste accounting in v1 (a single %).
- No export to a supplier format.
- No own calculation of areas and openings: that is SPEC-031.

## 4. Key decision

**The takeoff is a schedule of a new kind that counts not elements but material layers.**

| Option | Consequence |
|---|---|
| Manual tables | out of sync, errors, not refreshed |
| Formulas in a schedule per element | does not see layers |
| **Takeoff by layers** | data already in types, refreshed, printed |

Model: `Takeoff { categories, by, waste, level, rows }` as a view; calculation at render time
and via `refresh_takeoff`; rows: `{material, thickness, function, area_m2, volume_m3, mass_kg,
waste, total}`. The element area is `Model::areas(e).net` (SPEC-031).

## 5. Stage A. Calculation by layers

- For each element of the category (walls, floors, roofs, ceilings) — the `net` surface area
  from SPEC-031: a wall — one face; a floor, ceiling and roof — the surface (for a sloped floor
  — along the slope). `Plan Area` is a projection and is not used for the takeoff.
- Area not determined (no type, zero thickness, roof without slopes) — a row
  "area not determined" and a warning, not `0`.
- Type layers: `volume = area × thickness`; sheet materials (OSB, gypsum board, membrane) —
  `area`.
- `openings` are already subtracted from `net` in SPEC-031; in `takeoff` they must not be
  subtracted again.
- The slope of a roof and of a sloped floor is already accounted for in the surface area of
  SPEC-031; the `1/cos α` factor must not be applied again.
- Rounding: m² to 0.01, m³ to 0.001, kg to 1.
- Layers without a material — warning `layer_without_material`, the row is not invented.

## 6. Stage B. Materials and density

- `material {name, density}` — a new field (kg/m³) in the library; default values.
- Mass = volume × density; for sheet materials — area × mass of 1 m² (if specified).
- Materials without density — a warning, mass is empty (do not invent).
- Compatibility: old projects get densities by material name.

| Material | Default density |
|---|---|
| Timber / lumber | 500 kg/m³ |
| OSB-3 | 650 kg/m³ |
| Mineral wool | 35 kg/m³ |
| Gypsum board | 900 kg/m³ |
| Membrane | 1 kg/m² (sheet) |

## 7. Stage C. View and command

```json
{"name": "Material takeoff", "categories": ["walls","floors","roofs"],
 "by": "material", "waste": 10}
```

- `by`: `material` (aggregated), `layer` (by layers), `type` (by construction types).
- `waste` — % for waste (default 0; for OSB 10, timber 5, membranes 10 — recommendations).
- `level` — restrict to a level.
- Rows and totals; sorting by material/volume.
- `refresh_takeoff {id}` — recalculation; `place` onto a sheet.
- There is no `deduct_openings` flag: the area is always net (SPEC-031).

## 8. Stage D. Special cases

- **Framing** (SPEC-033): if studs/joists are elements, they appear as "timber" with a section
  and a length; without them — a calculation by the `timber_framing` layout
  (area × coefficient).
- **Membranes**: 10–15 % overlap — a coefficient in the material type or in `takeoff`.
- **Fasteners**: from parameters/elements (SPEC-033) or from a table (SPEC-035), do not invent.
- **Terrace**: decking and beams — in a separate group.
- **Area not determined**: a row with a warning, the quantity is empty; do not substitute zero.

## 9. Stage E. Export and sheets

- XLSX/CSV — as with schedules (`path`), with totals.
- On a sheet — as a view (table), with a caption and scale; compatible with SPEC-035.
- PDF/SVG.

## 10. Test plans

1. A wall of 84.76 m² (net, SPEC-031) with layers: OSB 12 ×2, mineral wool 195, gypsum board
   12.5 — areas and volumes match the manual calculation (reference — R-004).
2. Openings: `takeoff` takes the net from SPEC-031 and does not subtract openings a second
   time; changing a door and `refresh_takeoff` update the rows.
3. A 7°/9° roof and a sloped floor: the area is already along the slope from SPEC-031;
   `1/cos α` is not applied again. For a floor, `Area` (the surface) is used, not `Plan Area`.
4. Mass: timber 9.3 m³ × 500 = 4,650 kg; mineral wool 44.2 × 35 = 1,547 kg.
5. Waste 10 %: total = base × 1.1.
6. `by: material` / `layer` / `type` — different groupings, totals match.
7. `refresh_takeoff` after a wall change updates the rows.
8. A layer without a material and a material without density — warnings, mass is empty.
9. Save/open; schedules regression.

## 11. i18n and documentation

- "Material Takeoff", "Layer", "Waste", "Density", "Mass".
- `docs/areas/quantities.md` — takeoffs; `schedules.md` — how it differs from schedules.

## 12. Acceptance criteria

- [ ] `takeoff` counts by type layers from the net areas of SPEC-031.
- [ ] Density and mass; warnings when density is missing.
- [ ] Groupings `material/layer/type`, waste.
- [ ] View on a sheet and XLSX/CSV export.
- [ ] `refresh_takeoff`, save/open.
- [ ] Reference R-004 matches within rounding.
- [ ] Tests §10, i18n, documentation.

## 13. Risks and questions

| Risk | Answer |
|---|---|
| Net area not determined | warning, quantity empty (SPEC-031) |
| No layers defined for a type | warning, a "no layers" row |
| Materials without density | mass empty, not 0 |
| Framing without elements | calculation by `timber_framing` until SPEC-033 |
| Double counting of openings/slope | forbidden: only `net` from SPEC-031 |

1. Is per-material waste accounting needed?
2. Are overlap coefficients needed in the material type?
3. Is a link between the takeoff and an estimate (prices) needed in v2?

## 14. Definition of Done

- [ ] Calculation by layers from net areas.
- [ ] Densities and mass.
- [ ] View, groupings, waste, `refresh_takeoff`.
- [ ] Export and sheet.
- [ ] Tests (reference R-004), i18n, documentation.
