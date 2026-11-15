# SPEC-014. Floor along a wall face: Pick Walls, Walls' Outline and sketch offsets

Status: **implemented in the `BIMCraft-floor-by-pick-walls` branch** (the main features are in the code; remaining limitations are listed in §7)
Date: 2026-11-15
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: [IPedrax/BIMCraft#434 "Nice to have"](https://github.com/IPedrax/BIMCraft/discussions/434), moved from issue [#301](https://github.com/IPedrax/BIMCraft/issues/301) (AwayZz20, 2026-11-07): "When I use the floor tool, I choose pick wall I can't put the line aligned to the core face, its just in the center or draw manually outside or inside".
Related documents: `SPEC-017-tab-highlight-and-selection.md` (Tab, separate task), `SPEC-008-User-Configurable-Keyboard-Shortcuts.md`, `SPEC-012-Revit-Shortcuts-File-Import.md`, `BIMCraft/docs/areas/levels-walls-roofs.md`, `BIMCraft/docs/areas/views-and-graphics.md`.
Task type: implemented functionality: wall-face geometry, floor sketching, Pick Walls, Walls' Outline, Offset and Draw previews, nested floors, tests, documentation and translations.

> **In short.** In the base branch, Pick Walls and Walls' Outline built a boundary on the wall's location line, Offset in Draw worked only for some tools, and a boundary along a wall face could not be produced without drawing lines by hand. This branch adds wall faces and the structural core to sketch mode, Pick Walls on the face under the cursor with Offset and Flip, a configurable Walls' Outline panel, accumulating contours, correct corners and nested floors, Offset for Pick Lines and every Draw shape, previews and Space-based Offset mirroring.

## 1. What was done: change table

| What | Before | After |
|---|---|---|
| **Wall faces in the core** | A wall had only `start`/`end` (the location line), thickness and layers for drawing; there was no separate "face for a sketch". | Added `walls::Face`: `Centre`, `ExteriorFinish`, `InteriorFinish`, `ExteriorCore`, `InteriorCore`; names `centre`, `exterior-finish`, `interior-finish`, `exterior-core`, `interior-core`; `Face::ALL`, `named`, `parse`, `flipped`, `with_core`, `is_core`. |
| **Wall core and faces** | There were no public functions for a face or the core. | Added `walls::core_faces` (structural-layer faces), `walls::face_offset` (face offset) and `walls::face_line` (a face line, trimmed at wall ends, with Offset). |
| **Face line** | Sketch lines followed the location line; joins were not considered. | `face_line` trims at wall caps and joins (`layer_geometry`, cap), returns an arc for a curved wall and a line for a straight one. |
| **`floor_outline_faces`** | `floor_outline` built only a centre-line contour. | Added contours along faces and the core: `floor_outline_faces`, `floor_outline_faces_of`, `floor_outline_only`; support for a selected set of walls (`only`) and the centre-line offset direction (`centre_outward`). |
| **`floor` command** | There were no face or offset parameters. | `floor` accepts `outline` (`centre`/`exterior-finish`/`interior-finish`/`exterior-core`/`interior-core`) and `outline_offset`; the reply includes `outline` and `outline_offset`; errors name a wall without a core and an unknown face. |
| **Centre contour and Offset** | A centre-line Offset always went to the left of the axis; walls drawn in different directions ended up with a mix of inward and outward shifts. | The centre-line Offset goes **inside the loop on every wall** by default; in Walls' Outline, `Space` flips it to the outside and back. |
| **Pick Walls: face under the cursor** | A wall click always placed a line on the location line. | A click places a line on the finish or core face on the cursor's side; curved walls get an arc along the face. |
| **Pick Walls: `Extend into wall (to core)`** | There was no checkbox. | The Options Bar has `Extend into wall (to core)`; it defaults to on for floors and off for roofs; a wall without a core falls back to its finish face with a message. |
| **Pick Walls: Offset** | Offset was not applied to Pick Walls. | Offset in project units, an empty field means 0; it shifts the line from the chosen face; a negative value goes to the other side. |
| **Pick Walls: preview** | There was no preview. | While Pick Walls is active, the future line on the face is drawn under the cursor with a `Core/Finish — interior/exterior, Offset N` label, in plan and 3D. |
| **Pick Walls: `Space`** | It did nothing. | `Space` flips a Pick Walls line to the opposite wall face; it also works while the Offset field is focused, without typing a space. |
| **Picked Pick Walls lines** | The line was an ordinary sketch line; no wall link was stored. | The sketch stores `Pick { wall, face, offset }`; lines are refitted by `refit_picks`; selecting a line in Modify shows the wall, core and Offset; a double click/Flip switches the face; Ctrl+Z undoes; a manual grip edit drops the link. |
| **Walls' Outline: panel** | The button inserted a centre-line contour immediately. | The button opens a floating Options Bar panel with `Location Line`, `Wall Component`, `Outline Offset`, the number of picked walls and a `DRAW` button. |
| **Walls' Outline: contour line** | Centre line only. | `Location Line`: `Wall Centre Line`, `Interior Face`, `Exterior Face`; `Wall Component`: `Finish`, `Core`; the component is disabled for the centre line. |
| **Walls' Outline: Offset** | There was no offset. | A separate `Outline Offset:` field; it shifts from the chosen line/component; for the centre line it goes inside by default. |
| **Walls' Outline: walls** | Every wall on the level was used. | Specific walls can be picked: without Ctrl the sketch's lines are picked, with Ctrl the walls; click and box selection (left to right — inside, right to left — touching), in plan and 3D. |
| **Walls' Outline: highlighting** | There was no highlighting. | Picked walls are highlighted; the contour and preview are drawn above wall faces. |
| **Walls' Outline: DRAW** | There was no DRAW button; a contour was inserted immediately. | `DRAW` builds a contour along all or the picked walls; the panel stays open after DRAW; another DRAW adds a new contour. |
| **Walls' Outline: duplicates** | Repeated clicks could duplicate a contour. | An exact duplicate of a contour already in the sketch is not added; the status bar says duplicates are not allowed. |
| **Walls' Outline: corners** | With Offset, neighbouring lines either crossed or failed to meet; a 45° cut could appear. | Neighbouring lines are extended/trimmed to their intersection (mitre), with no diagonal cuts; the tolerance accounts for Offset. |
| **Walls' Outline: nested contours** | `solve_one` refused the sketch with "2 separate outlines". | Finish creates a separate floor for each `solve` region: the outer floor with its hole and a separate inner island floor, in one operation. |
| **Walls' Outline: leaving the mode** | Esc/OK closed the panel. | `Esc` clears only the picked walls and keeps the mode on; `Modify` does not leave the mode; a Draw tool, Finish or Cancel leaves it. |
| **Pick Lines: Offset** | Offset was measured/applied from the location line; offsetting from the face did not work. | For a wall, Pick Lines takes the face under the cursor and offsets from it; Offset 0 places the line on the face; lines and grids are offset from their own line. |
| **Pick Lines: preview** | There was no preview. | The line that would be built with the current Offset is shown under the cursor, in plan and 3D. |
| **Draw Offset: `Space`** | `Space` did not flip Offset, and could type a space into the Offset field. | `Space` flips the Offset side for every Draw element that has one; no space is typed into the field; preview and final geometry match. |
| **Offset: which shapes** | `Shape::offsets()` covered only open lines/arcs/polylines; Space did nothing for rectangles, polygons or circles. | Added `Shape::takes_offset`; Space works for Line, arcs, Polyline, Rectangle, polygons, Circle, Ellipse, Partial Ellipse and Pick Lines. |
| **Ellipse offset** | Offset was unavailable or ignored. | A full ellipse grows/shrinks by Offset; the preview shows the result; Space changes the side. |
| **Partial ellipse offset** | Offset was unavailable or ignored. | A partial ellipse is shifted sideways by Offset; the preview shows the result; Space changes the side. |
| **Line chain with Offset** | Each segment was offset separately; at a corner the lines crossed or failed to meet. | Consecutive chain segments are trimmed/extended to their common intersection; on closure the last line meets the first. |
| **Sketch draw order** | The model could be drawn over sketch lines through the retained GPU callback; a centre-line contour was hidden under a wall. | Retained drawing is disabled while a sketch is open; the model and sketch are drawn in the usual order; the contour and preview stay above walls. |
| **Picked wall highlighting** | There was none. | Walls picked for Walls' Outline get an accent outline; it is drawn under sketch lines so it does not cover the contour. |
| **3D sketch** | Pick Walls/Walls' Outline and Offset previews worked only in plan. | 3D supports wall picking by click and box, highlighting, Pick Walls, Pick Lines and Offset previews. |
| **Tests** | There were no tests for faces, Pick Walls Offset, Walls' Outline or Draw Offset. | Added/updated tests in `core/tests.rs`, `ui/tests/sketch.rs`, `ui/tests/draw.rs`, `tests.rs`, `tests/structure.rs`, `tests/workplane.rs`. |
| **Documentation and translations** | Floor/wall and Draw docs did not cover faces, DRAW/Offset or Space. | Updated `docs/areas/levels-walls-roofs.md` and `docs/areas/views-and-graphics.md`; `translations/ui` and `translations/project` refreshed with `cargo i18n update`. |

## 2. Subsystem details

### 2.1. Core: faces and contours

- `crates/core/src/walls.rs`:
  - `Face` and its methods;
  - `core_faces`, `face_offset`, `face_line` (with `face_line_in` internally);
  - trimming face lines at wall caps and joins; curved walls.
- `crates/core/src/commands.rs`:
  - `floor_outline` (previous behaviour) and `floor_outline_only` (a subset of walls);
  - `floor_outline_faces` and `floor_outline_faces_of(..., only, centre_outward)`;
  - `outline_lines` — centre lines of the selected walls;
  - `floor` with `outline`/`outline_offset`, validation and the reply;
  - `edit_boundary` explicitly rejects `outline`/`outline_offset`;
  - contour corner mitre with tolerance `thickest + 2 × |offset| + TOL`;
  - centre-line Offset inward/outward according to `inside_left`.
- `crates/core/src/lib.rs` — new public exports `floor_outline_faces_of` and `floor_outline_only`.

### 2.2. Pick Walls

- `BimApp::pick_core`, `pick_offset`, `pick_flip`.
- `picked_face` — the cursor side, falling back to a previous Pick or the interior side.
- `Sketch::picks: BTreeMap<usize, Pick>`.
- `refit_picks`, `forget_moved_picks`, `flip_pick`.
- Properties shows the wall, `Extend into wall (to core)` and `Offset`.
- A double click on a picked line switches its face.
- `Space` switches the face, including while the Offset field is focused.
- Preview in plan and 3D.

### 2.3. Walls' Outline

- State: `outline_side`, `outline_core`, `outline_offset`, `outline_flip`, `outline_pending`, `outline_walls`.
- The `Walls` panel contains only the button; settings live in the floating Options Bar.
- `Location Line`: centre/interior/exterior; `Wall Component`: finish/core; `Outline Offset`; `Walls: all/N`; `DRAW`.
- `DRAW` calls `floor_outline_faces_of` with the picked walls and accumulates contours.
- Exact duplicate contours are rejected with a message.
- Exit: `Esc` clears the walls; `Modify` keeps the mode on; Draw/Finish/Cancel leave it.
- Wall picking: without Ctrl the sketch's lines, with Ctrl the walls (click/box, in plan and 3D).
- Contour and preview are drawn above walls.

### 2.4. Draw, Pick Lines and Offset

- `Drawing.offset_flip`; `draw_options` returns Offset adjusted by the flip.
- `Shape::takes_offset` — every shape except splines and Fillet; `Shape::offsets` — open lines/arcs/polyline/partial ellipse.
- A full ellipse: `make` applies `grown`; a partial ellipse is shifted sideways.
- Pick Lines: `picked_curve_at` takes the wall face and measures Offset from it; used in plan, 3D and previews.
- Line chain: `sketch_line_chain` (first/last line indices) and a mitre on add and closure.
- Preview: lines with Offset in the shared `preview`; shapes through `shape_preview`; Pick Walls/Pick Lines in `sketch_preview`; 3D in `sketch3d`.
- `Space`: an early interception when the Offset field is focused, then the general handler; for Walls' Outline it flips the centre-line Offset outward/inward; for Pick Walls it chooses the other face.

### 2.5. Nested floor contours

- `finish_sketch` for `Of::Floor` calls `solve`; with more than one region it calls `finish_nested_floor`.
- Each region is created by `floor` with `boundary` + `openings`; the outer region is a floor with a hole and the inner one is a separate island floor.
- Creation uses `run_all` — one Undo step; all created floors are selected after Finish.
- For a sketch on a 3D work plane, level/offset are preserved.

## 3. Behaviour and rules

- **Centre-line Offset:** inside the closed contour on every wall by default; `Space` goes outside; the next `Space` goes back inside. The field value itself does not change sign.
- **Walls' Outline:** DRAW does not replace previously built contours; it adds a new one. An exact duplicate is not added.
- **Walls' Outline panel** stays open after DRAW until the mode is left.
- **Ctrl:** without Ctrl selection works on sketch lines; with Ctrl it picks walls for Walls' Outline.
- **Pick Lines:** for a wall, Offset is measured from the face under the cursor; for other elements, from their line.
- **Pick Walls:** the binding lives only in the sketch; after Finish the floor stores coordinates, not a wall reference.
- **Manual edit:** dragging a grip, Trim, Fillet, Offset or splitting a line drops the Pick binding.
- **Wall without a core:** Pick Walls falls back to the finish face with a message; the `floor` command with `*core` returns an error naming the wall.
- **Nested contours:** supported for floors; ceilings/roofs/shafts still use a single region.

## 4. Limitations and out of scope

1. **Associativity after Finish:** the floor does not follow a wall after creation; the contour is coordinates. This differs from Revit and needs a separate task.
2. **Walls' Outline is only for a new floor:** the panel is shown for `Of::Floor`; it is not available in Edit Boundary for an existing floor. The contour is edited with lines/Edit Boundary.
3. **Nested contours are floor-only:** ceilings/roofs/shafts still use `solve_one`; this task was about floors.
4. **Exact duplicate check:** only fully identical contours are compared. Partially overlapping contours can be added and are then rejected by the solver at Finish.
5. **Roof:** face/Offset contour reading was not moved to roofs; `roof_footprint` keeps its previous behaviour.
6. **Lock/icon for a binding:** there is no separate lock icon; the link is visible in Properties and through Flip.
7. **Tests:** added but not run locally (project rule); verification is by CI/the user.

## 5. Code map

| File / function | What it implements |
|---|---|
| `crates/core/src/walls.rs` | `Face`, `core_faces`, `face_offset`, `face_line`, faces and trimming |
| `crates/core/src/commands.rs` | `floor_outline_only`, `floor_outline_faces`, `floor_outline_faces_of`, `floor {outline, outline_offset}`, mitre, centre-line Offset |
| `crates/core/src/lib.rs` | Exports `floor_outline_faces_of`, `floor_outline_only` |
| `crates/ui-egui/src/sketch.rs` | `Pick`, `picks`, `refit_picks`, `flip_pick`, Pick Walls, Walls' Outline, Line chain, nested floors |
| `crates/ui-egui/src/canvas.rs` | Wall picking and box selection, previews, retained drawing disabled in a sketch, highlighting, clicks |
| `crates/ui-egui/src/draw.rs` | `offset_flip`, `takes_offset`, `picked_curve_at`, Pick Lines from a face, previews |
| `crates/ui-egui/src/sketch3d.rs` | 3D wall picking, Pick Walls/Pick Lines, Offset previews |
| `crates/ui-egui/src/properties.rs` | Properties of a picked Pick Walls line |
| `crates/ui-egui/src/lib.rs` | State, Space/Esc handling, key sets |
| `crates/ui-egui/src/tests/sketch.rs` | Tests for Pick Walls, Walls' Outline, Offset and nested contours |
| `crates/ui-egui/src/tests/draw.rs` | Test for `Space` with Draw Offset |
| `docs/areas/levels-walls-roofs.md` | Faces, Walls' Outline and nested floors |
| `docs/areas/views-and-graphics.md` | Offset, previews and Space |

## 6. Tests added/updated in the branch

- `pick_walls_puts_a_line_on_the_core_or_the_finish_face_under_the_cursor`;
- `space_flips_a_picked_wall_line_to_the_other_face`;
- `pick_walls_offset_moves_the_line_away_from_the_wall`;
- `walls_outline_follows_the_face_options_and_finish_makes_the_floor`;
- `draw_does_not_lay_a_second_copy_of_an_outline_already_in_the_sketch`;
- `walls_outline_picks_walls_with_the_arrow_and_esc_clears_them`;
- `a_picked_boundary_line_shows_its_wall_face_and_flips_and_undoes`;
- `a_grip_moved_on_a_picked_boundary_line_makes_it_an_ordinary_line`;
- `space_flips_the_draw_offset_to_the_other_side`;
- updated Floor/Walls' Outline tests in `tests.rs`, `tests/structure.rs` and `tests/workplane.rs`.

Tests were not run locally per `~/.config/opencode/rust-rules.md`; verification is by CI and the user.

## 7. Checks

- `cargo check` — passed;
- `cargo clippy -p bimcraft-core -p bimcraft-ui-egui` — passed;
- `cargo fmt --check` — passed;
- `cargo i18n check` — passed;
- `scripts/debug_floor.sh --non-interactive` — build and window launch succeeded (per the branch log).
