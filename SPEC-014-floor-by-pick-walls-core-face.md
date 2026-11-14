# SPEC-014. Floor by Pick Walls: finish face and wall core face

Status: **planned** (not started)
Date: 2026-11-13
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: [IPedrax/BIMCraft#434 "Nice to have"](https://github.com/IPedrax/BIMCraft/discussions/434), moved from issue [#301](https://github.com/IPedrax/BIMCraft/issues/301) (AwayZz20, 2026-11-07): "When I use the floor tool, I choose pick wall I can't put the line aligned to the core face, its just in the center or draw manually outside or inside".
Related documents: `ТЗ-010-правка-размерной-цепочки.md` (a sample of sketch and dialog edits), `ТЗ-013-типы-осевых-линий.md` (a sample of the spec structure), `BIMCraft/docs/areas/*` (description of floors and walls).
Task type: new functionality (wall geometry, sketch, Options Bar, commands/MCP, tests), without changing the file format.

> **In short.** Today Pick Walls in the floor sketch always places a line along the **wall's location line** (`sketch.rs:649–658`), and `floor {}` without a boundary and the "Walls' Outline" button build the boundary from centre lines (`commands.rs:3192`, `sketch.rs:853–867`). As a result, the floor either runs into the wall by half its thickness, or has to be drawn manually "by eye" — it is impossible to hit the core face. In Revit this is solved on the Options Bar: **Offset** and **Extend into wall (to core)**; the line binds to the face on the cursor's side, and a selected line can be flipped to the other core face with the flip arrows. The task is to reproduce this behaviour: Pick Walls with face selection (finish/core) and an offset, the same options for "Walls' Outline" and `floor`/`edit_boundary` in MCP, a preview, and tests.

## 0. General working rules

1. Communication language — Russian; code, comments, code strings, console messages — English. Documentation in the repository (`docs/`, `README.md`, `CLAUDE.md`) — English; this file — English (translated from `ТЗ-014-пол-по-грани-ядра-стены.md`).
2. Tests (`cargo test`, `cargo nextest`, doctests and any runs of test code) are **not run** in this task (rule `~/.config/opencode/rust-rules.md`): the check is `cargo check`, `cargo build`, `cargo clippy` (without `--all-targets`) and CI in the PR.
3. Commits are atomic: one commit — one logically complete change; every commit builds and does not break existing behaviour.
4. Identifiers (names of fields, commands, types, files) are not translated. New user-facing strings — via `msg!`/`ui_msg!`/`t()`, the `translations/` catalogs are updated (`cargo i18n update`).
5. `Model::check` is the only place for model validation; sketch checks remain in `core/sketch.rs::solve` (the floor's `check` validates the resulting boundary, not how it was obtained). New errors are worded so that it is clear what to fix.
6. Do not copy Autodesk data: neither fonts, nor templates, nor type names from `.rvt`/`.rfa`. We reproduce behaviour from public documentation and compute the geometry ourselves.

## 1. Problem statement

### 1.1. What exists now (per the code)

**Floor sketch and Pick Walls**

- The Floor tool opens the sketch immediately (`lib.rs:2955–2963`): `start_sketch(Of::Floor(level), Vec::new())`. There is no separate "create a floor with one click" in the UI anymore; a click inside the walls is a preview of the future boundary (`canvas.rs:4783–4790`), and the floor itself is created on Finish.
- Pick Walls is `Tool::SketchPick`, a button in the Draw panel (`draw.rs:318–323`). A click on a wall (`sketch.rs:649–658`) adds **one** line:
  - straight wall → `SketchLine::new(w.start, w.end)`;
  - arc wall → `SketchLine::arc(w.start, w.end, axis.at(axis.length() / 2.0, 0.0))`.
  That is, the wall's **location line** is taken (`axis.at(..., 0.0)`), without regard to `Wall.location`, thickness or layers.
- Status bar hint: "Click a wall: a boundary line along it" (`sketch.rs:966`); nothing about the face/core.
- "Walls' Outline" (`sketch.rs:841–867`) takes `bimcraft_core::floor_outline(&model, level)` and puts the edges of the resulting loops into the sketch. `floor_outline` (`commands.rs:3192–3216`) builds them from `m.wall_lines(level)` (`model.rs:6841`), i.e. from the walls' centre lines, filtered by level and Room Bounding.
- `floor {}` without `boundary` uses the same `floor_outline` (`commands.rs:9808–9818`); `edit_boundary` without `sketch` returns the current boundary (`commands.rs:11973`), and with `sketch` accepts ready-made lines.
- Options Bar in sketch mode: `lib.rs:5290–5298` calls `sketch_options` and returns; `sketch_options` (`sketch.rs:896–946`) today shows only the line style of the **filled region** (`sk.styled`); for floor/ceiling/roof/shaft it is empty. The "Offset" and "Extend into wall (to core)" fields exist nowhere: `grep -rn "to core\|Extend into" crates` is empty.

**What already exists in wall geometry and can be reused**

- `Wall.location` (`model.rs:339–381`) defines where the location line lies relative to the thickness: `centre`, `core-centre`, `exterior-finish`, `interior-finish`, `exterior-core`, `interior-core` (`Location`, parsed in the wall commands).
- `walls::depth(thickness, layers, loc)` (`walls.rs:153–171`) — the offset of the location line from the **exterior face** (0 — exterior finish, `thickness` — interior).
- `walls::boundaries(m, w)` (`walls.rs:175–188`) — the offsets of all layer boundaries "to the left of the location line", from the exterior face to the interior one; the core is from the first to the last `Function::Structure` layer.
- `walls::faces(m, w)` (`walls.rs:191–195`) — `(ext, int)`: the exterior and interior finish faces.
- `walls::centre(m, w)` (`walls.rs:198–201`) — the centre line of the physical wall (not always equal to the location line).
- `walls::layer_geometry` (`walls.rs:513–534`) — the same lines, but **trimmed at the ends** against adjoining walls (`on_cap`, `end_geometry`), taking the core "butt" into account. This is exactly what is needed for a face line: without trimming, neighbouring lines either will not meet or will intersect, and `sketch::solve` will refuse to build the boundary.
- `walls::outline` (`walls.rs:424–449`) — the wall outline (ends + both faces); `Model::face_lines` (`model.rs:6885`) — wall outline edges for ceilings.
- `WallType { thickness, layers }` (`model.rs:2735–2739`), `Layer { function, thickness }` — the core is determined by `Function::Structure`.

**Conclusion.** The problem is not a missing parameter, but that Pick Walls has no notion of a "face": it always returns the location line. Faces and the core are already computed in the model for drawing walls and rooms — they need to be brought into the sketch.

### 1.2. Revit scenarios to reproduce

According to Autodesk documentation ("Create a Floor", "Place a Door" and "About Flip Controls"; verified 2026-11-13):

1. **Floor by walls.** Floor → sketch → Pick Walls is active by default. Hovering over a wall shows a preview of the boundary; a click places the line on **the face nearest to the cursor** (interior or exterior). The Options Bar has **Offset** (mm) and the **Extend into wall (to core)** checkbox.
2. **Core face.** Checkbox on → the line runs along the **core face** (the edge of the structural layer) on the side the cursor points to; off → along the **finish face**. Offset is measured from the selected face and shifts the line outward (positive value — bigger floor, negative — smaller).
3. **Flip to the other core face.** Having selected a line placed by Pick Walls, in Revit you can press the flip arrows and switch it to the opposite face of the same core (documented in Autodesk Community answers about "Extend into wall (to core)"). This matters when the automation picked the wrong side.
4. **The same Pick Walls in Edit Boundary.** When editing an existing floor, the same tools and the same options are available; a new line immediately changes the boundary, and Finish applies it.
5. **Walls' Outline / "floor by walls".** A boundary built from walls follows the face in Revit (the core when the checkbox is on), not the location line; therefore the floor area is the room area plus/minus Offset, rather than "half the wall".
6. **Roof by walls.** The roof's Pick Walls also has `Extend into wall (to core)` and `Overhang` (in the line's properties); in BIMCraft the roof already computes the overhang from faces (`roof_footprint`, `commands.rs:12367–12402`), but takes the location line for the boundary from `floor_outline`.

### 1.3. Why it matters

- **Areas and volumes are wrong.** Today the floor by default reaches the location line: in a room 4 × 6 m with 200 mm walls its area is larger than the real one by about 1.0 m² along the perimeter. This ends up in `schedule`, IFC and material quantities — a silent error, not cosmetics.
- **The floor-to-wall joint is visible.** In a section/3D view the floor sticks out of the wall by half its thickness, or, conversely, does not reach the core; the user compensates with a manual line "by eye", and it does not survive wall edits.
- **No structural layer in the calculation.** It is impossible to place a finish floor up to the finish face and a structural one up to the core: both scenarios exist in any project, and a Revit user expects them by default.
- **The mechanism already exists.** Layers, the core, end trimming and the cursor-side face are computed for walls, rooms and the roof. The work is to connect what exists, not to build new geometry.

## 2. Terms

| Term | Meaning |
|---|---|
| Pick Walls | Sketch tool (`Tool::SketchPick`) that places a boundary line along a wall |
| Location line | The line on which the wall's `start`/`end` are defined; its position within the thickness is set by `Wall.location` |
| Exterior/interior face | The wall's finish faces: `ext` and `int` from `walls::faces`, "left/right" of the location line |
| Core | `Function::Structure` layers in `WallType.layers`; its faces are the edges of the first and last structural layer |
| Extend into wall (to core) | Checkbox: the Pick Walls line follows the core face instead of the finish face |
| Offset | Offset of the line from the selected face, mm; positive — outward from the wall/room |
| Walls' Outline | Button that places into the sketch the boundary formed by the level's walls |
| Picked line | A sketch line that remembers the wall and face it was taken from (in sketch memory only, not in the file) |
| Face flip | Switching a picked line to the opposite face of the same wall |
| Cursor side | Sign of `Axis::project(cursor).1` relative to `walls::centre(m, w)` |
| Trimmed face line | A face line trimmed at the ends against adjoining walls (`walls::layer_geometry`) |

## 3. Functional requirements

### 3.1. F1. Face and core geometry in `walls.rs`

- Introduce a public face enum suitable both for the sketch and for the commands:

  ```rust
  pub enum Face {
      Centre,
      ExteriorFinish,
      InteriorFinish,
      ExteriorCore,
      InteriorCore,
  }
  ```

  Names in JSON commands — `centre`, `exterior-finish`, `interior-finish`, `exterior-core`, `interior-core` (the same as the `Wall.location` vocabulary, so as not to introduce a second language).
- `pub fn core_faces(m: &Model, w: &Wall) -> Option<(f64, f64)>` — offsets of the exterior and interior core faces to the left of the location line:
  - find the first and last `Function::Structure` by `WallType.layers`;
  - if there are no structural layers, return `None` (the caller falls back to `faces` and reports "the wall has no core");
  - compute the offsets via `depth(...)`: exterior core face = `ext - c0`, interior = `ext - c1`, where `(c0, c1)` are the core boundaries from the exterior face (the formula is already used in `depth`, `walls.rs:153–171`).
- `pub fn face_offset(m: &Model, w: &Wall, face: Face) -> Option<f64>` — offset of the selected face to the left of the location line: centre — `walls::centre`, finish — `faces`, core — `core_faces`.
- `pub fn face_line(m: &Model, id: Id, face: Face, offset: f64, index: Option<&Index>) -> Option<Curve>` — the face line, **trimmed at the ends**, shifted by `offset` mm outward:
  - take `layer_geometry` (or its generalization) and the required line from `LayerGeometry.lines`;
  - add `offset` to the face offset **before** trimming, to preserve the joins (do not move an already trimmed polyline);
  - return `Curve::polyline(points)` for arc walls and `Curve::line(a, b)` for straight ones; if there are fewer than two points — `None`.
- Do not change the meaning of `layer_geometry`'s internals: it already trims lines at the ends and accounts for the core butt (`walls.rs:475–534`). It is acceptable to extract the shared computation into a private function with an offset parameter so as not to duplicate code.
- Checks and errors: `face_line` returns `None` for a non-existent wall; the caller writes to the status "This wall has no core: its faces are the core" and **does not** reject Pick Walls (it places along the finish face, as Revit does when there is no structural layer).

### 3.2. F2. Pick Walls with a face and an offset

- Tool state (in `BimApp`, next to `floor_structural`/`ceiling_height`, reset in `set_tool`, `lib.rs:2943–2961`):
  - `pick_core: bool` — "Extend into wall (to core)";
  - `pick_offset: f64` — "Offset", mm (entered as text, like the other Options Bar fields, with project units);
  - defaults: for `Of::Floor` and the floor's `Of::Edit` — `pick_core = true`, `pick_offset = 0.0`; for `Of::Roof` — `pick_core = false` (as in Revit for a roof), `pick_offset = roof_overhang`? **No**: the roof Offset is a separate `Overhang`, do not mix it in this task; `pick_offset = 0.0`.
  - the state lives until the end of the sketch; a new sketch starts from the defaults. It is not written to the file.
- `Tool::SketchPick` branch (`sketch.rs:649–658`):
  1. `let side = Axis::of(w).project(raw_or_cursor).1` (access to the cursor point is required; `sketch_click` already receives `p`/`raw` — use `raw` or `p` consistently, as in `opening_params`, `canvas.rs:2147–2162`);
  2. face: `Face::Exterior*` if `side > walls::centre(m, w)`, otherwise `Face::Interior*`; when equal (cursor on the centre line) — the side of the previous pick of this wall, otherwise the interior one;
  3. `core`/`finish` — by `pick_core`;
  4. outward shift: `+pick_offset` for the exterior face, `-pick_offset` for the interior face; for `Face::Centre` — `+pick_offset` to the left;
  5. the line — `walls::face_line(m, id, face, signed_offset, None)`; add `sk.add(line)` and record the pick (F3).
- One wall — one line per click; a repeated click on the same wall from the other side adds a second line (as in Revit), it does not replace the first.
- Preview under the cursor: while `Tool::SketchPick` is active and the cursor is over a wall, draw the future line in `pal().accent` and show a caption next to it, "Core — interior, Offset 0" (or "Finish"); over empty space — nothing. Implement in `sketch_preview` (`sketch.rs:712+`) or in `canvas.rs`, where tool previews already live.
- The status bar hint (`sketch.rs:966`) becomes dynamic: "Click a wall: a boundary line on its core face (Extend into wall). Space…" — without mentioning keys that do not exist; text via `msg!`.

### 3.3. F3. Picked lines and face flipping

- `Sketch` (`sketch.rs:55–73`) gets a field that is **not serialized**:

  ```rust
  pub picks: BTreeMap<usize, Pick>,
  pub struct Pick { pub wall: Id, pub face: Face, pub offset: f64 }
  ```

  The key is the line index in `Sketch.lines`; the index is stable as long as the line is not deleted/rebuilt. All operations that change a line's geometry not through `Pick` (handle drag, Trim, Fillet, Offset, insert/delete) remove the entry for that line: the line becomes ordinary, as in Revit after a manual edit.
- Recomputing picked lines: the function `refit_picks(&mut self, m: &Model)` calls `walls::face_line` again and replaces `curve`, preserving `id`/`guid`/`style`/`invisible`/`slope`. Call it:
  - when entering the sketch and after the sketch's `Undo`/`Redo` (the history stores only `SketchLine`, so picks must survive undo; if the line comes back — the entry is restored by index if the index matches; otherwise the pick is lost — acceptable and described in the tests);
  - after `pick_core`/`pick_offset` changes on the Options Bar, if picked lines are selected;
  - **not** on every frame: only on events.
- Selecting a picked line in Modify: show its fields in the Properties palette, as Revit shows the properties of a boundary line:
  - "Extend into wall (to core)" — checkbox;
  - "Offset" — number, mm;
  - "Wall" — wall name/type read-only (do not show the identifier as a GUID; id and type are enough).
  Changing any field recomputes the line and records one change in the sketch history (a single undo).
- Flip: double-click on a picked line in Modify (and the "Flip" item in the palette's context menu) switches `ExteriorFinish ↔ InteriorFinish` or `ExteriorCore ↔ InteriorCore`, preserving `offset`; the line is recomputed. This is exactly the Revit behaviour "clicking the flip arrows moves the line to the other core face".
- If the wall is deleted/changed such that `face_line` returned `None`, the pick is dropped, the line stays as it was; the status shows a warning once: "Wall of a boundary line is gone: the line stays as drawn".

### 3.4. F4. Sketch Options Bar

- In `sketch_options` (`sketch.rs:896–946`) add a branch for `Of::Floor(..)` and `Of::Edit(id)`, where `id` is a floor (checked against the model), as well as `Of::Roof(..)` for symmetry:
  - `Extend into wall (to core)` checkbox — `ui.checkbox(&mut self.pick_core, t("Extend into wall (to core)"))`, available only while `Tool::SketchPick` is active or picked lines are selected; otherwise visible but inactive (as Revit shows the Options Bar only in sketch mode — ours is always in sketch mode, so simply disabled);
  - an `Offset:` field with `TextEdit` and a units hint, like `floor_slope`/`ceiling_height` (`lib.rs:5333–5350`); values accept project units; an empty field = 0;
  - `Space`/`Enter` must not be intercepted: the fields behave like the other Options Bar text fields.
- Changing the checkbox/field is applied:
  - to subsequent picks — immediately;
  - to already selected picked lines — per the F3 rules (one undo per field change, not per frame).
- Labels and hints — via `msg!`/`t()`; new strings go into `translations/ui` (`cargo i18n update`). Texts:
  - "Extend into wall (to core)";
  - "Offset";
  - "Pick Walls: a boundary line on the wall's face under the pointer";
  - "This wall has no core: its faces are the core".
- Accessibility: the checkbox and the field have `labelled_by`/`WidgetInfo::labeled`, like the other fields (`lib.rs:5333–5345`), so that tests and screen readers find them by label.

### 3.5. F5. "Walls' Outline" and the default boundary

- `floor_outline` (`commands.rs:3192`) gets a variant with faces:

  ```rust
  pub fn floor_outline_faces(
      m: &Model,
      level: impl Key,
      face: Face,
      offset: f64,
  ) -> Option<(Vec<P2>, Vec<Vec<P2>>)>
  ```

  - for each wall of the level that belongs to `bounding_walls`, take `walls::face_line` with the selected face and offset;
  - assemble segments from the lines and pass them through `geom::faces(..., TOL)`, as today;
  - choose the largest piece as the outer boundary, the rest as inner loops (courtyards), by the same rules as `floor_outline` (`commands.rs:3194–3215`);
  - the face side for each wall: the face turned **inward into the boundary** that this wall limits. Determine it by the loop orientation: outer loop CCW → the inner side is to the left of the traversal direction, courtyard CW → to the right; if a wall falls into both (which should not happen for a simple boundary), take the side facing the room (`bounding_walls` + `rooms`).
  - document that with mixed thicknesses each wall takes its own face (unlike `roof_footprint`, which today takes the thickest wall, `commands.rs:12377–12385`; that is a known simplification, do not repeat it here).
- `floor {}`:
  - without `boundary` and without `outline` — **previous behaviour** (centre), so that old scripts and tests do not change geometry;
  - with `outline: "centre" | "exterior-finish" | "interior-finish" | "exterior-core" | "interior-core"` — the boundary along that face;
  - `outline_offset` — mm, default 0; the sign as on the Options Bar (positive — outward).
  - `outline` without `boundary`; if both are given — error "give either `boundary` or `outline`, not both".
- `edit_boundary`:
  - without `sketch` — returns the boundary as is; does it add `picks` to the response? No: picks are not stored in the model, so there is nothing to return. The response does not change.
  - with `sketch` — accepts ready-made lines, as today; `outline`/`outline_offset` do not apply (the lines are already given), an error if an attempt is made to combine them.
- "Walls' Outline" in the sketch (`sketch.rs:853–867`) calls `floor_outline_faces` with the current `pick_core`/`pick_offset`; the button's Tip changes: "Walls' Outline: the outline the level's walls enclose, along their faces". With `pick_core = true` by default this is the core face — as a Revit user expects.
- The floor preview on click (`canvas.rs:4783–4790`) uses the same variant, so that the shown line matches the created one.
- Roof: `roof_footprint` can be switched to `floor_outline_faces` with the `Face` from `pick_core` in a separate commit. This is included in the scope of this task only if `Of::Roof` got a checkbox (see F4); otherwise — in §9.

### 3.6. F6. Commands and MCP

- `floor` (`commands.rs:345`, description; `commands.rs:9803`, implementation) gets parameters:

  | Parameter | Type | Default | Meaning |
  |---|---|---|---|
  | `outline` | `"centre" \| "exterior-finish" \| "interior-finish" \| "exterior-core" \| "interior-core"` | none | The face along which the boundary is built when `boundary` is not given |
  | `outline_offset` | number, mm | 0 | Offset of the boundary from the face, outward |

- `edit_boundary`: no changes; in the command description state that Pick Walls picks are UI sketch state, not command state (MCP sets the boundary by coordinates).
- Errors:
  - unknown face → list the allowed ones;
  - `exterior-core`/`interior-core` on a wall without structural layers → error with the wall/type name, not "no walls enclose an area";
  - `outline_offset` without `outline` → error "`outline_offset` needs `outline`".
- The `floor` response is extended with the `outline` (which face was applied) and `outline_offset` fields, so that the agent sees what happened.
- `sketch {lines}` and `floor {sketch}` do not change: the lines come as coordinates.
- MCP scenario (verification): a 200 mm wall, layers 50 finish / 100 structure / 50 finish, location centre; `floor {"outline": "interior-core"}` gives a boundary 50 mm inward from the interior finish face (that is, 50 mm from the location line toward the centre), and `interior-finish` — 100 mm; the areas differ by the perimeter × 50 mm.

### 3.7. F7. Consistency with rooms, roof and IFC

- Rooms (`rooms.rs`) compute area by wall faces; a floor along the core face may be smaller/larger than the room — that is normal, as in Revit; do not link them in `check`.
- Do not change `roof_footprint` in this task, except for the F4/F5 case (the checkbox for the roof). If the roof is not included — leave its behaviour and note the difference in §9.
- IFC: the floor is exported by its own boundary (`ifc.rs`), there are no new fields; nothing changes on export.
- `Insert from File`: floor elements are transferred as is; sketch picks are not transferred (they are not in the file).

## 4. Model, format, migration

- The `.bimcraft` format **does not change**: `Floor.boundary`/`openings` store ready-made coordinates; `Sketch` and picks are UI state, not model state.
- `Model::check` does not change: it validates a simple boundary and holes; the face/core does not affect the resulting boundary.
- Old files open unchanged: the floor geometry is already recorded.
- There is no migration. If future associativity is needed (the floor follows the wall after Finish) — a separate spec; here it is deliberately not done (§9).

## 5. Rendering and UX

- The sketch line stays the `SKETCH` colour (`sketch.rs:78–79`); a picked line additionally gets a small lock icon at its midpoint and a tooltip "Wall: interior core, Offset 0" (only in Modify, to avoid noise while drawing). The icon — vector, in screen pixels, not scaled with zoom.
- Pick Walls preview: a line in `pal().accent`, a caption near the cursor with the face and offset; when `face_line` returns `None` — a dashed line along the finish face and a warning in the status.
- The cursor over a wall in SketchPick — `PointingHand` (like wall picks), off the wall — normal.
- No modal dialogs: everything on the Options Bar and in the Properties palette, as in Revit.
- Sketch errors (`sketch::solve`) are shown as today (`wrong`/status); a pick must not mask the "lines intersect" error — if the offset made the lines intersect, the user sees the same error and can reduce Offset.

## 6. Tests (describe; do not run in this task)

Unit tests `crates/core/src/walls.rs` (or `walls/tests`):

1. `depth`/`core_faces` for a 200 mm wall with 50/100/50 layers at `Location::Centre`: exterior core face 50, interior -50; for `Location::ExteriorFinish`: 150 and 50; for `InteriorCore`: 0 and -100.
2. A wall without structural layers: `core_faces` → `None`, `face_line` with `Core` falls back to the finish (or returns `None` — fix the decision in the code and the test).
3. `face_line` on a straight wall with a join: the end of the line lies on the neighbouring wall's face, not on the centre line (compare with `walls::outline`).
4. `face_line` on an arc wall: the radius differs by the thickness/offset, the ends coincide with `axis.at(0, off)`/`axis.at(len, off)`.
5. `face_line` with `offset ±50` shifts the line outward/inward and preserves the trimming.

Integration tests `crates/core/tests` (via `Session::execute`):

6. `floor {"outline": "interior-core"}` on a three-layer wall gives an area smaller than `centre` and larger than `interior-finish`; check the specific corner coordinates.
7. `interior-core` on a wall without a core — error with the wall name.
8. `outline_offset` without `outline` — error.
9. `floor {}` without parameters gives the previous boundary (regression).

UI tests `crates/ui-egui/src/tests/sketch.rs`:

10. Floor → Pick Walls, checkbox on: a click on the interior side of the wall places the line on the interior core face; off — on the interior finish face; check the coordinates against the model, not pixels.
11. A click on the exterior side gives the exterior face; a repeated click on the same wall from the other side adds a second line.
12. Offset 100 shifts the line outward; a negative one — inward.
13. Select a picked line in Modify → the palette shows Extend into wall and Offset; changing them recomputes the line, Ctrl+Z reverts a single edit.
14. Double-click/"Flip" moves the interior core face to the exterior one, Offset is preserved.
15. "Walls' Outline" with the checkbox on gives a boundary along the core; the floor preview on click matches the boundary after Finish.
16. Finish creates a floor with the boundary along the selected face; the area in the response/`query_elements` matches the expected one.
17. Dragging a picked line's handle drops the pick (after that Offset no longer moves the line).

Visual check: a `render` of the plan and section — the floor line matches the core face, with no gap and no protrusion beyond the wall.

## 7. Scope boundaries

- **Do not** make it associative after Finish: the floor boundary is not rebuilt when the wall moves. In Revit the Pick Walls line stays tied; for us that would require storing picks in the model and a format migration — a separate spec.
- **Do not** implement `Plate Offset From Base`, `Overhang` as a roof line property, slope-defining lines and other Revit boundary-line properties.
- **Do not** touch ceilings and shafts: their Pick Walls stays along the centre line (the ceiling already reaches the faces via `face_lines`; the shaft is a rectangle).
- **Do not** change `roof_footprint` to faces unless the roof checkbox is included in F4; in any case, do not touch the overhang calculation.
- **Do not** implement "floor by room" (Room Bounding) and do not link the floor to the room.
- **Do not** add `Extend into wall` to IFC export: in IFC the floor boundary is a boundary, not a reference to a wall face.

## 8. Acceptance criteria

1. In the floor sketch, the Options Bar has "Extend into wall (to core)" and "Offset"; the defaults are on and 0.
2. Pick Walls places the line on the face on the cursor's side: with the checkbox on — on the core face, with it off — on the finish face; Offset shifts it while preserving the joins.
3. Arc walls and walls with different `Wall.location` give a correct line (not the centre line).
4. A selected picked line can be flipped to the other core face; Offset and Extend into wall are visible and editable in Properties, one undo per change.
5. "Walls' Outline" and the floor preview use the same options; the boundary after Finish matches the preview.
6. `floor {"outline": "..."}` and `outline_offset` work via MCP; the errors name the face/wall and what to fix.
7. Old projects and `floor {}` without parameters give the previous geometry.
8. `cargo check`, `cargo build`, `cargo clippy` (without `--all-targets`) pass; CI is green; tests are added but were not run locally (project rule).

## 9. Open questions and decisions made

1. **Sign of Offset.** Decided: positive — outward from the wall (bigger floor), negative — inward. This matches the expectation "Offset enlarges the floor" and the roof's `Overhang`; fix it in the field hint.
2. **Default for `floor {}`.** Decided: keep centre, so as not to change existing projects and tests; the UI sketch by default follows the core (Revit). If the user expects "floor by default up to the core" in MCP as well — `outline: "interior-core"` could be added as the default in the next major version; this is a deliberate decision, not an omission.
3. **Roof.** Decided: include the checkbox and `floor_outline_faces` for `Of::Roof`, but do not switch `roof_footprint` to faces in this task (it currently takes the thickest wall, `commands.rs:12377`); otherwise the overhang and the existing tests will break. A separate commit/spec.
4. **Associativity.** Decided: out of scope; picks live only in the sketch. This difference from Revit must be explicitly stated in the documentation (`docs/areas`), so as not to promise too much.
5. **Flip with a single arrow or a double-click.** Decided: double-click + the "Flip" item in the palette; separate arrows on the line — in SPEC-016 (they are about doors/windows, not sketch lines).
6. **Wall without a core.** Decided: not an error; a line along the finish face + a warning in the status. An error only for `floor {"outline": "*core"}` via MCP, where the user explicitly asked for the core.

## 10. Work plan by commits

1. `core: add face and core geometry for walls` — `Face`, `core_faces`, `face_offset`, `face_line`, unit tests.
2. `core: floor outline along wall faces` — `floor_outline_faces`, `floor {outline, outline_offset}`, errors, integration tests.
3. `ui: pick walls on a face with offset` — `pick_core`/`pick_offset` state, `SketchPick`, Options Bar, status, preview, UI tests.
4. `ui: keep picked boundary lines tied to their wall face` — `Pick`, `refit_picks`, Properties palette, Flip, clearing on manual edit, UI tests.
5. `ui: walls outline and floor preview follow the face options` — `walls_outline`, `canvas.rs` preview, tests.
6. `docs: describe floor faces and the offset sign` — `docs/areas` + `README` (English), update of the i18n catalogs.

Every commit builds and does not break the previous behaviour; tests are written in the same commits but are not run locally (CI).
