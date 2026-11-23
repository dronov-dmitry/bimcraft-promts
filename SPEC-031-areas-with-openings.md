# SPEC-031. Areas and volumes of planar elements with openings and holes

Status: **implemented** (2026-11-23). No automated tests from §8 were added for areas; warning
tests were added and not run — `a_wall_of_an_unknown_type_says_its_area_is_unknown` and
`a_roof_whose_faces_cannot_be_found_says_its_area_is_unknown`. Running tests — only on direct
request (project rule).
Date: 2026-11-23
Project: BIMCraft 031 (Rust; `crates/core/src/commands.rs` — `elements` :4444 (wall), :4579
(roof), :4627 (floor), :4643 (ceiling), :4410 (toposolid), :4697 (filled region);
`crates/core/src/model.rs` — `floor_slabs` :7353, `floor_prisms` :7365, `floor_volume` :7373,
`ceiling_slabs` :7392, `less_shafts` :7402, `roof_outline` :6733, `roof_faces` :6811,
`roof_pieces` :7045, `roof_prisms` :7000; `crates/core/src/walls.rs` — `prisms` :1043,
`host_voids_in` :1068; `crates/core/src/schedule.rs` — fields :276–360, `shared` :1140;
`crates/core/src/ifc.rs` — `wall_quantities` :1389, floor :1861, roof :2535)
Source: preparation of the material takeoff (SPEC-032). Consumption by layers multiplies an
element's area by the layer thickness, but the areas of the four categories are currently
computed differently: a wall has no area in the schedule at all, a roof gives a plan without
openings, floor and ceiling are already net. The same name `Area` means different quantities.

Related: SPEC-032 (material takeoffs), SPEC-036 (unit configuration), SPEC-037 (roof edge
without slope).
Task type: fixing quantity calculation; the `.bimc` format does not change, fields are added
additively.

> **In short.** One rule is needed for all planar elements: **`Area` = net and it is the surface
> area** (for a wall — one face; for floor, ceiling and roof — the surface including slope), while
> `Plan Area` is the projection in plan. `Openings Area` is the area of subtracted holes. A wall
> must subtract doors, windows and family cuts (including arched ones); floor and ceiling — their
> openings and shafts; roof — openings and shafts. Gross is computed only internally, to obtain
> the openings area, and is not output to the model or the schedule.

## 0. General working rules

1. Language — Russian; code and strings — English; window strings — `msg!`/`t()`.
2. Tests — only on request; CI separately. Allowed: `check`, `build`, `clippy` (without
   `--all-targets`), `fmt --check`, `i18n check`.
3. **Commits are atomic.** One commit — one logically complete change (one feature, one fix, one
   refactoring); unrelated changes are not mixed. Every commit builds and does not break existing
   behavior.
4. Commit order: unified API → `elements` fields → schedule fields → IFC/crosscheck → i18n/docs.
5. Before a commit — `git diff` and `git status`; do not capture build artifacts, temporary files
   or other people's changes.
6. After a push — wait for green CI; on failure find the cause, fix it with a separate atomic
   commit and repeat until a successful run.
7. No new dependencies; `unsafe` is not added.
8. Old projects are read as is; there are no new fields in the file, everything is computed from
   geometry.

## 1. The essence of the problem

### 1.1. What is computed now

| Category                    | `elements`                                                                                    | Schedule                                                                                                      | Openings                                            |
| --------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Wall                        | `length`, `height`, `thickness`; no area (`commands.rs:4444`)                                 | no own `Area` (`schedule.rs:315`); in the multi-category `Area` = `Length` × `Height` (`schedule.rs:1151`)     | not subtracted                                      |
| Floor                       | `area_m2` = slabs minus their own `openings` and shafts (`commands.rs:4627`, `model.rs:7353`, `7402`) | `Area` is taken from `area_m2` → net plan                                                                     | subtracted                                          |
| Ceiling                     | `area_m2` = slabs minus `openings` and shafts (`commands.rs:4643`, `model.rs:7392`)           | `Area` → net plan                                                                                             | subtracted                                          |
| Roof                        | `area_m2` = outline in plan minus courtyards (`commands.rs:4588`)                             | only `Plan Area` (`schedule.rs:350`)                                                                          | roof `openings` are not subtracted; slope is not taken into account |
| Filled region (`FilledRegion`) | `area_m2` from `sketch::solve`                                                              | `Area`                                                                                                        | holes are subtracted (`geom.rs:545`)                |
| Toposolid                   | `area_m2` = outline (`commands.rs:4413`)                                                      | `Area`                                                                                                        | no `openings` in the model                          |
| Mass, shaft, property line  | outline/plan                                                                                  | `Area`                                                                                                        | no holes                                            |

### 1.2. Discrepancies between consumers

- **Wall.** IFC computes `NetSideArea` differently: for a wall without adjacencies, door and
  window rectangles are subtracted (`ifc.rs:1395`); for an adjacent one — the volumes of prisms
  with family cuts (`ifc.rs:1405`). An arched window in the first case is subtracted as a
  rectangle, in the second — as an arch.
- **Floor.** `elements.area_m2` subtracts shafts, while IFC `NetArea` subtracts only its own
  `openings` (`ifc.rs:1862`). The same slab yields different numbers.
- **Roof.** Net by slope is computed only in IFC (`ifc.rs:2535–2561`); it is absent in the model
  and in the schedule.
- **Multi-category schedule.** `Area` for a wall is gross, for floor and ceiling — net plan, for
  a roof — plan. One field name means different things.
- **Sloped floor.** `area_m2` is a plan; there is no surface area either in `elements` or in the
  schedule, although material consumption needs exactly that.

### 1.3. Why this blocks the takeoff

The material takeoff (SPEC-032) multiplies an element's area by the layer thickness. It needs
exactly the net **surface** area: material consumption must not include openings, and for a
sloped floor it must not be computed by projection. Currently a wall has no net at all, a roof
has none, a sloped floor has only the plan. Without a common rule the takeoff would have to
repeat four different computations and would again diverge from `elements` and IFC.

## 2. Goals

1. One rule for walls, floors, ceilings and roofs: `Area` = net surface area,
   `Openings Area` = the area of subtracted holes, `Plan Area` = the projection in plan.
2. One API in `Model`, used by `elements`, schedules, IFC, crosscheck and the takeoff.
3. Net takes into account **any** holes: doors and windows, family cuts (including arched ones),
   components that cut a face, each slab's and roof's own `openings`, shafts.
4. Slope is taken into account for the roof and for a sloped floor; `Plan Area` remains a
   separate quantity.
5. Fields in `elements` and schedules: `Area` (net surface), `Openings Area`, `Volume` (net);
   for floor and roof additionally `Plan Area`.
6. Identical numbers in `elements`, schedule and IFC for the same geometry.
7. An explicit warning wherever the area is undefined (no type, zero thickness, a roof without
   splitting into slopes).
8. Show the same fields in Properties; all dimensional parameters — in Dimensions, units — in
   values, names — without units.
9. A computed `Opening Area` for openings/families: from the void, otherwise `Width × Height`.
10. Roof thickness in `elements`, the schedule and Properties (from the type).

## 3. Non-goals

- Gross area and gross volume as model fields: not needed in v1. Gross is computed only
  internally to obtain `Openings Area`.
- Geometry, the 3D view and boolean operations do not change.
- Room areas (`RoomArea`) are not touched: that is a separate computation.
- Sheet and board cutting, waste — SPEC-032.
- Measurement rules per estimating standards (opening deduction threshold, reveals, sills) — a
  separate rules layer, not in v1.
- The area of an opening itself as an element (width × height) remains nominal; actual family
  cuts are taken into account only in the wall area.
- No new fields are added to `.bimc`.

## 4. Key decision

**`Area` = net surface area, `Plan Area` = projection, `Openings Area` = subtracted
holes. Gross is an internal quantity.**

| Category | `Area` measure                  | `Net` (field `Area`)                                                            | `Plan Area`            | `Openings` (field `Openings Area`) |
| -------- | ------------------------------- | ------------------------------------------------------------------------------- | ---------------------- | ---------------------------------- |
| Wall     | one face, m²                    | outline minus doors, windows and family/component cuts, trimmed by top/bottom     | none                   | `internal Gross − Net`             |
| Floor    | surface including slope, m²     | outline minus `openings` and shafts, along the slope                             | net projection         | `internal Gross − Net`             |
| Ceiling  | surface, m²                     | outline minus `openings` and shafts (a ceiling is flat)                          | not output: flat       | `internal Gross − Net`             |
| Roof     | slope surface, m²               | slopes minus `openings` and shafts                                               | net projection         | `internal Gross − Net`             |

- `Area` in the schedule and `area_m2` in `elements` = **net surface** for all four
  categories.
- In code the model computes in millimetres: the API `Model::quantities`/`areas`/`plan_area`/
  `volume` returns **mm² and mm³**. In `elements` and schedules the same quantities are shown in
  **m² and m³** (`m2`, rounded to 0.01 m² and 0.001 m³).
- `Plan Area` — for floor and roof: the outline projection minus holes and shafts. For a flat
  floor it coincides with `Area`; for a sloped one it is smaller.
- For a wall with zero/unknown thickness `Area` is undefined: `null` and a warning, not `0`.
- Regulatory rules (`GESN`/`FER`) with opening deduction thresholds are not modelled: ours is a
  geometric net, while measurement rules are separate, if ever needed.

## 5. Stage A. Unified API

In `crates/core/src/model.rs`:

```rust
#[derive(Clone, Copy, Debug, PartialEq)]
pub struct Areas {
    pub net: f64,       // mm², net surface area
    pub openings: f64,  // mm², subtracted holes (by surface)
}

impl Model {
    pub fn areas(&self, id: impl Key) -> Option<Areas>;
    /// Net projection in plan, mm²: for floor and roof; for wall and ceiling — None.
    pub fn plan_area(&self, id: impl Key) -> Option<f64>;
    /// Net volume, mm³.
    pub fn volume(&self, id: impl Key) -> Option<f64>;
}
```

- Dispatch by `Element`: `Wall`, `Floor`, `Ceiling`, `Roof`; for the rest `None`.
- Gross is not part of the public result: each branch computes it locally and subtracts it to
  obtain `openings`.
- The computation is lazy, without a cache in v1: the area is needed during rendering and
  commands, not in a hot loop. If profiling shows a problem — a cache in `derived` as a separate
  step.
- Rounding — at the output boundary (`m2`, `hundredth`); internally — millimetres.

### 5.1. Wall

```rust
let volume = |openings| walls::prisms(m, id, openings)
    .iter().map(prism_volume).sum::<f64>(); // mm³
let (gross_v, net_v) = (volume(false), volume(true));
let t = m.thickness(w); // mm
if t <= 0.0 { return None; }
Areas { net: net_v / t, openings: (gross_v - net_v) / t }
```

- **Decision: net = `volume / thickness`.** Prisms are an extrusion of the wall cross-section,
  so volume/thickness gives exactly the area of one face: for a straight wall, an arc wall, a
  wall cut by a roof, and with family cuts. Walking the faces would require separately finding
  the "long" edges and distinguishing them from ends and cuts — more code and more errors.
- `walls::prisms(m, id, true)` already cuts out doors, windows, family cuts and face components
  (`walls.rs:1039–1071`), and also trims the top/bottom by adjacencies.
- Gross — the same prisms without openings: `prisms(m, id, false)`; it is not output to the
  result.
- Net volume = `net_v`.
- Method limitation: the thickness must be constant. A wall with `t == 0` (type not found) —
  `None`, warning `wall_area_unknown`.
- Check: for a simple rectangular wall `net` coincides with `length × height − openings`.

### 5.2. Floor and ceiling

- `Net` (surface):
  - floor — via `floor_prisms`: for each piece `less_shafts` gives the plan, multiplied by the
    slope coefficient `(1 + top.g·top.g).sqrt()`; summed over the pieces;
  - a flat ceiling — coefficient 1.
- `Plan Area` — the sum of `less_shafts` areas without the slope coefficient (as now).
- Gross — the same slabs with temporarily cleared `openings` (a clone of the element, as IFC
  does for the roof), without subtracting shafts; needed only for `openings`.
- `Openings = gross surface − net surface`.
- Net volume: floor — `floor_volume`; ceiling — area × thickness.
- Sloped floor: `Area` — the surface, `Plan Area` — the projection; for a flat floor they
  coincide.
- Floor with shape editing: the coefficient is taken by each piece's plane (`top` from
  `floor_prisms`).

### 5.3. Roof

- `Net` — as in IFC (`ifc.rs:2550–2558`): `roof_prisms` → `less_shafts` → area ×
  `(1 + g·g).sqrt()` (slope).
- `Plan Area` — the net projection: outline minus courtyards, `openings` and shafts.
- Gross — `roof_faces` on the roof slope with cleared `openings`, ring area × slope; courtyards
  remain holes (as in `ifc.rs:2538–2548`); needed only for `openings`.
- `Openings = gross surface − net surface`.
- If `roof_faces` returned an error (for example, a gable edge without an edge in the outline) —
  `None` and a warning, as now in IFC.
- A roof with `extrusion` is computed the same way (`roof_outline`, `roof_prisms`).

### 5.4. Other planar elements

- `FilledRegion`: the area is already net (`geom::Profile::area` = outer outline minus holes,
  `geom.rs:545`). Not part of the API; regression — the number does not change.
- `Toposolid`: there are no `openings` in the model; `area_m2` = outline, `volume_m3` — volume.
  Does not change.
- `Shaft`, `PropertyLine`: no holes. Does not change.
- `Mass`, curtain panels, mullions: not planar elements with holes. Does not change.

## 6. Stage B. Fields and commands

### 6.1. `elements`

For `Wall`, `Floor`, `Ceiling`, `Roof` add keys:

| Key                | Meaning                                                          |
| ------------------ | ---------------------------------------------------------------- |
| `area_m2`          | net surface (existing key; appears for a wall for the first time) |
| `openings_area_m2` | hole area                                                        |
| `volume_m3`        | net volume                                                       |
| `plan_area_m2`     | net projection: for floor and roof                               |

- Rounding: m² to 0.01, m³ to 0.001.
- For a wall `length`, `height`, `thickness` remain; a wall has no `plan_area_m2`.
- For a floor `area_m2` becomes the surface (for a flat floor — as before), `plan_area_m2` — the
  former projection.
- For a roof `area_m2` becomes the net surface, `plan_area_m2` — the net projection.
- There are no gross keys (`gross_area_m2`, `gross_volume_m3`).

### 6.2. Schedules

- `wall`: after `THICKNESS` add `Area` (net), `Openings Area`, `Volume` (net) to the own fields.
  The default values (the first 6) do not change, so that old wall schedules look as before.
- `floor`: `Area` — net surface; add `Plan Area`, `Openings Area`; `Volume` already exists.
- `ceiling`: `Area` — net surface (flat, coincides with the plan); add `Openings Area`,
  `Volume`.
- `roof`: add `Area` (net surface), `Openings Area`, `Volume`; `Plan Area` remains and becomes
  the net plan.
- Multi-category: `Area` = net surface for all four; add `Openings Area` to `SHARED`; for a roof
  `Area` → the own `Area` (surface), not `Plan Area` (`schedule.rs:1142`).
- `fields`, `field_text`, tags and formulas receive the new fields automatically.

### 6.3. Compatibility and visible change

- The file does not change: fields are computed, not stored.
- Old schedules are read; the new fields are available but not selected by default.
- **Visible change in numbers** (record in the release notes):
  - multi-category `Area` for a wall: was gross `L×H`, became net;
  - `Area` for a roof: was `Plan Area`, became the net surface;
  - `Area` for a sloped floor: was the plan, became the surface; for a flat floor it did not
    change;
  - `Plan Area` for a roof now minus `openings` and shafts (previously only courtyards).
- The decision is taken: `Area` = net surface. Gross fields are not added.

### 6.4. Summary table of changes

"Appears" — the field/key does not exist now. "Changes" — the name exists, but its meaning or
number changes. "Disappears" — none at all: existing fields are preserved, gross fields are not
added.

**`elements` keys:**

| Category | Appears | Changes | Disappears |
|---|---|---|---|
| Wall | `area_m2`, `openings_area_m2`, `volume_m3` | — | — |
| Floor | `plan_area_m2`, `openings_area_m2` | `area_m2`: plan → surface (the number is the same for a flat floor) | — |
| Ceiling | `openings_area_m2`, `volume_m3` | — | — |
| Roof | `plan_area_m2`, `openings_area_m2`, `volume_m3` | `area_m2`: plan → surface | — |

**Schedule fields:**

| Category | Appears | Changes | Does not change | Disappears |
|---|---|---|---|---|
| Wall | `Area`, `Openings Area`, `Volume` | — | `Length`, `Height`, `Thickness` | — |
| Floor | `Plan Area`, `Openings Area` | `Area`: plan → surface (the number is the same for a flat floor) | `Thickness`, `Volume` | — |
| Ceiling | `Openings Area`, `Volume` | — | `Area`, `Thickness` | — |
| Roof | `Area`, `Openings Area`, `Volume` | `Plan Area`: gross plan → net plan (minus `openings` and shafts) | `Slope` | — |
| Multi-category | `Openings Area` | `Area`: wall gross → net; floor and roof plan → surface; `Volume`: wall, ceiling and roof empty/0 → net | `Length`, `Height`, `Thickness` | — |

Notes:

- "Does not change" lists only existing fields that participate in the area/volume computation
  and remain the same in meaning and number; the other fields of the category are not listed
  here.
- `Area` for a wall in its own schedule is a new field; in the multi-category one it existed and
  changes meaning from `L×H` to net.
- `Plan Area` for a roof already exists, so it is not listed under "Appears"; for a floor it is
  new.
- `Plan Area` is not added to the multi-category one: it is present in the floor and roof
  schedules.
- A ceiling has no `Plan Area`: it is flat, `Area` already equals the projection.
- `Volume` for a floor already existed; for a wall, ceiling and roof it appears. Therefore in the
  multi-category schedule `Volume` for these three categories changes from empty/0 to net.

### 6.5. Properties, groups and units

In addition to `elements` and schedules, the fields are output in Properties:

| Category | Dimensions | Quantities |
|---|---|---|
| Wall | Length, Thickness, Area, Openings Area, Radius | Volume |
| Floor | Area, Plan Area, Openings Area, Thickness | — |
| Ceiling | Area, Openings Area, Thickness | Volume |
| Roof | Slope, Thickness, Max Ridge Height, Area, Plan Area, Openings Area | Volume |

- All dimensional parameters (Length, Area, Volume, Thickness, Formwork Area) — in the
  **Dimensions** group; **Quantities** keeps only weight and rebar ratio.
- Units are shown to the right of the value: `13150 mm`, `33.87 m²`, `6.774 m³`, `30°`; parameter
  names — without units (`Volume`, not `Volume (m³)`).
- Editable length and angle fields accept a value with and without a unit (`3000`, `3000 mm`,
  `30°`).
- This is the default behavior. The full setting "show units in values" and reconciling Family
  Types/schedules — a separate SPEC-036; in this task units are always on.
- In code the API computes in mm²/mm³; Properties shows m²/m³ (see §5).

### 6.6. Opening Area of openings and families

Separately from host areas: the wall subtraction of a void does not change. For openings and
families a computed `Opening Area` was added:

- for a family with a host-cutting void — the area of the cut profile (extrusion volume /
  length);
- for a door/window without a void — `Width × Height`;
- for a door/window of a **project type** — `Width × Height` in the element's `values`;
- `Opening Area` is visible in `values`, in Family Types (a read-only row under Dimensions), in
  the Properties of a door/window/component and in door/window/component schedules;
- it is not stored in the file: it is computed from the type/instance;
- `Width`/`Height` — the hosted family contract (`Model::opening_dims`): renaming them breaks
  the opening size itself, not just `Opening Area`;
- for a family with a void the area is taken from geometry, so renaming the parameters does not
  affect it.

### 6.7. Roof thickness

- `RoofType.thickness` already existed in the type but was not output for the instance. Added:
  - roof `elements` → `thickness`;
  - roof schedule → the `Thickness` field;
  - roof Properties → Dimensions → Thickness.
- The thickness is set in the roof type (Type Properties); for the instance it is shown
  read-only.

### 6.8. Warnings

`Model::warnings` (visible via `inspect`/`inspect_model`) says where the area is undefined:

- a wall with an unknown type or zero thickness: `wall {id}: ... area and volume are unknown`;
- a roof whose slopes cannot be found: `roof {id}: its faces can't be found (...), so its area and
  volume are unknown`.

The area is not substituted with zero: `quantities` returns `None`, the fields are absent.

## 7. Stage C. Reconciling consumers

- **IFC** (`ifc.rs`):
  - `wall_quantities` — always via the `Model` computation, without a separate branch
    "rectangles versus prisms"; `NetSideArea` subtracts family cuts.
  - floor — `NetArea` subtracts both its own `openings` and shafts; for a sloped floor IFC writes
    volumes, there is no surface area in the standard set.
  - roof — `NetArea` = `Model::areas().net`, `ProjectedArea` remains the gross projection per the
    schema.
  - `Gross*` in IFC remain as the schema requires (`Qto_*BaseQuantities`), but there are no new
    gross fields in the model.
- **crosscheck** (`crosscheck.rs:153`): add wall `area_m2` (net) to `length_m` and `openings`;
  the floor already matches by plan.
- **tags/schedules** — via the fields.
- **SPEC-032** — takes `Model::areas(...).net` and does not subtract openings or slope again.

## 8. Test plans

1. Wall 4000×3000, door 900×2100: `net = 10.11`, `openings = 1.89`.
2. Wall with a window 1200×1500 at a 900 sill: `openings = 1.80`.
3. An arched window with a family cut: `openings` equals the arch area, not the rectangle.
4. A wall cut by a roof (adjacency on top): `net = net_volume / thickness`, the height is not
   uniform.
5. Flat floor 8000×6000 with a 1000×1000 hole and a 500×500 shaft: `Area = 46.75`,
   `Plan Area = 46.75`, `openings = 1.25`.
6. Sloped floor 8000×6000 at 5°: `Plan Area = 46.75`, `Area = 46.75 / cos 5°`.
7. A ceiling with the same holes: `Area` coincides with the flat floor.
8. Roof at 7° with an opening and a courtyard: `Area = (plan − courtyard − opening) / cos 7°`;
   `Plan Area = plan − courtyard − opening`.
9. IFC: the wall's `NetSideArea` and the flat floor's/roof's `NetArea` equal `Area`; for a sloped
   floor the volumes are checked.
10. `FilledRegion` with a hole: the area did not change.
11. Save/open; old schedules open; `elements` regression.
12. Properties: Length/Thickness/Area/Openings Area/Volume for a wall; Area/Plan Area/Openings
    Area for floor and roof; Volume — in Dimensions, not in Quantities.
13. Units: values with units (`mm`, `m²`, `m³`, `°`), names without `(m³)`/`(°)`; input `3000 mm`
    and `30°` is accepted.
14. A family's `Opening Area` with a void equals the cut area; for a door/window without a void —
    `Width × Height`; for a project type — likewise.
15. `Opening Area` is visible in Family Types, Properties and schedules; renaming `Width`/`Height`
    breaks the opening size — this is expected.
16. Roof: `thickness` in `elements`, `Thickness` in the schedule and Properties.
17. `Model::warnings`: a wall without a type/thickness and a roof without slopes give a warning;
    the areas remain `None`, not `0`.

## 9. i18n and documentation

- Strings: "Openings Area", "Plan Area", "Opening Area", "Defines Slope" (the last one —
  SPEC-037).
- `docs/areas/quantities.md` — definitions of net surface, projection and openings by category;
  a "Changed in this version" section about the change in the meaning of `Area`.
- `docs/areas/schedules.md` — which fields come from where; `Area` — always the net surface.
- `docs/areas/families.md` — a family's `Opening Area` and the `Width`/`Height` contract.
- `SPEC-036-units-in-values.md` — the general unit setting; `SPEC-037-roof-edge-slope-toggle.md` —
  a roof edge without slope (not part of this task).

## 10. Acceptance criteria

- [ ] `Model::areas`/`plan_area`/`volume` for walls, floors, ceilings, roofs.
- [ ] A wall subtracts doors, windows, family and component cuts; arched ones — as they are.
- [ ] Floor and ceiling: net surface and opening area; for a flat floor the net coincides with
      the current one.
- [ ] Sloped floor: `Area` — the surface, `Plan Area` — the projection.
- [ ] Roof: net surface by slope, openings and shafts, `Plan Area` — the net projection.
- [ ] Fields in `elements` and schedules; `Area` = net surface.
- [ ] IFC and crosscheck use the same API; the numbers match wherever IFC writes an area.
- [ ] Warnings for zero thickness/unknown wall type and an unsplit roof.
- [ ] Properties: area/volume fields, all dimensions in Dimensions, units in values, names
      without units.
- [ ] `Opening Area` for openings/families: void, otherwise `Width × Height`, including project
      types.
- [ ] Roof thickness in `elements`, the schedule and Properties.
- [ ] Tests §8 (no automated tests added yet; the warning tests were added, not run), i18n,
      documentation.

## 11. Risks and decisions

| Risk                                                 | Answer                                                       |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| Change in the meaning of `Area` for wall, roof and sloped floor | decided: `Area` = net surface; record in the release notes |
| Wall without a type/thickness                        | `None` + warning, not `0`                                    |
| An opening above the wall or overlapping another     | trimming by the top and merging — as in `prisms`; test      |
| A roof whose slopes cannot be found                  | `None` + warning, as in IFC                                  |
| Sloped floor                                         | `Area` — the surface, `Plan Area` — the projection; both are output |
| Performance on large models                          | lazy computation; a cache only based on profiling results     |
| Estimating rules for opening deduction               | not in v1: geometric net, rules — as a separate layer        |

There are no open questions: the wall method is `volume / thickness`; the surface area of a
sloped floor is part of the task.

## 12. Definition of Done

- [ ] Unified API and definitions (net surface, projection, openings).
- [ ] `elements` and schedule fields (only net, openings and projection).
- [ ] IFC/crosscheck reconciled.
- [ ] Properties: area/volume fields, Dimensions/Quantities groups, units in values.
- [ ] `Opening Area` for openings/families; roof thickness.
- [ ] Warnings for a wall without a type/thickness and a roof without slopes.
- [ ] Release notes about the change in `Area` (`docs/areas/quantities.md`, "Changed in this version").
- [ ] Automated tests §8: not added; the warning tests were added, not run (project rule).
- [ ] i18n and documentation.
