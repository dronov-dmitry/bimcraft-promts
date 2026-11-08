# SPEC-007. Strip footing (and all runs) as a wall: end grips, mitred corner joints, gluing to the bottom of a wall

Status: **implemented; section 12 — follow-up work on the corner node when moving**
Date: 2026-11-06 (follow-up — 2026-11-08)
Project: BIMCraft (working copy `BIMCraft-plugins-clean`, branch `feature/wasm-plugins`, current project format 17)
Related specs: `SPEC-006-turning-point-behind-anchor.md` (document formatting)
Task type: behaviour of linear elements ("runs": strip footing, beam, pipe, cable tray, etc.) — editing ends with grips, mitred corner joints modelled on walls, the "strip under the bottom of a wall" link, and a general join toggle for walls and runs.

Screenshot of the joint defect (from the task):
`https://private-user-images.githubusercontent.com/9892891/696054534-fbc7c1d4-d1e6-46ab-8cfa-9b2329b33626.png`

---

## 0. General working rules

- Commits are atomic, one item per commit (1, 2, 3, 4, "Smaller"), messages follow the branch style
  (imperative-mood subject, body explains *why*, not *what*).
- Code and commit-message language — English; documentation in `docs/` — Russian (as currently in
  `docs/areas/plugins.md`).
- UI strings — only via `t()` / `tf()` from `crates/ui-egui/src/lang.rs`, with `msgctxt` where

## 1. Context and task

In BIMCraft the strip footing is created almost like a wall (`strip_footing` tool, a chain of points, as for a wall), but it behaves differently. Three things are required:

1. **End grips** — a strip (and all runs) must have the same plan-view grips that walls currently have: select the element in Modify, see squares at the ends, drag one — the end follows the cursor with snaps.
2. **Mitred corner joint** — where a strip meets another strip (a corner, a T-junction, a joint with a wall), its end currently stays flat, perpendicular to its own axis, and the strip does not "close" (screenshot above). The joint must be formed the way a wall joint is formed: mitre in the corner, cut along the face of the meeting element.
3. **Gluing to the bottom of a wall** — if the centre line of the strip coincides with the centre line of a wall and the bottom of the wall touches the top of the footing, the strip "glues" to the bottom of the wall and behaves as in Revit: it moves with the wall in plan and travels vertically with the bottom of the wall (when the wall's `base offset` changes, the strip's `offset` is recalculated).

Scope decisions confirmed by the user:

- **Vertical gluing — full:** the top of the strip always stays at the bottom of the wall; raise/lower the wall (its `base offset`, level) and the strip travels vertically with it.
- **Coverage — all runs at once:** end grips and mitred joints are done not only for the strip footing but for all linear elements (`RUN_CATEGORIES` — 10 categories: `beam`, `stripfooting`, `pipe`, `duct`, `conduit`, `cabletray`, `wire`, `downconductor`, `groundconductor`, `cable`).
- **Joining is on by default** for all elements (walls too), but it can be turned off/on per element: right click → **"Disallow Join" / "Allow Join"** (analogous to Revit). A disallowed element joins with nothing: its ends stay square, it does not stretch behind a joint.

---

## 2. Why it happens this way now (causes in the code)

### 2.1. Runs have no grips

Plan-view grips are implemented **only for walls** and are hard-wired to the `Wall` type:

- `crates/ui-egui/src/canvas.rs`, `gripped()` — returns the selected element's id as "grippable" only if `model.wall(id)` (single selection + `Modify` tool). For runs this function always yields `None`, so grips are not even drawn.
- `grip_target()` takes a `(&Wall, bool)` pair and builds the drag target from a wall end; `grip_drag()` reads `m.wall(id)` and on release sends a `move` command with the `end` parameter (`"start"`/`"end"`); `grip_overlay()` draws the wall outline preview via `walls::faces`; `wall_angles()` — angle entry for a wall.
- In the core, the `move` command outright refuses everything except a single wall: `crates/core/src/commands.rs`, `move_elements()` — if `end` is given and `ids` is not exactly one wall, the error is "`end` moves an end of one wall; give `ids` one wall". So even if the UI sent `end` for a run, the command would reject it.
- `relocate()` handles `end` only in the `Element::Wall` branch; runs, when moved, are transformed as a whole (both ends at once, the `run_arm!` branch in `transformed`).

Result: runs have no end-editing mechanism at all — neither in the UI nor in the core. The only thing possible is to move/rotate/scale the whole element.

### 2.2. Runs have no mitred joints

Walls join because their **outline is always rebuilt taking neighbours into account**:

- `crates/core/src/walls.rs`: `arms()` looks for neighbouring walls at a wall's end (candidates via `lookup.walls_near_at` by wall thickness, with a height-overlap check — `stacked()`), `cap()` trims an end that butts against another wall's face, `outline_in()`/`outline()` assemble the final outline with mitres in corners; `centre_line()` extends the centre lines to the joint.
- This single outline is consumed everywhere: plan (`render.rs`, `cut_outlines()` → `walls::outline_in`, fill union `cut_unions()` and seams `cut_edges()`, bottom view `seen_below()`), 3D (`view3d.rs` — `walls::prisms_in`), IFC (`ifc.rs` — "Its outline swept up, mitred where it joins others as the plan's is"), measurements and `Model::outline`.

Runs have **nothing of the kind**:

- The only outline source is `runs.rs`, `Geom::footprint()`: a rectangle across the axis from `start` to `end` with ends strictly perpendicular to its own axis. No neighbour search exists for runs at all: the `derived::Lookup` index indexes only walls (`walls_near_at`, `walls_near`, etc.).
- This rectangle is consumed by: plan (`render.rs`, `stands()` → the same `footprint`; through it — `cut_outlines`, `seen_below`, extent, highlight and selection), 3D (`view3d.rs` — a `Solid::Prism` from `g.prism()`), IFC (`ifc.rs` — a section swept along the axis: `body_along`), sections.
- `relocate()` also knows nothing about run joints: moving a neighbour does not stretch or trim the strip.

Result (screenshot): at the point of contact the strip's end stays square and perpendicular; the walls above/next to it close with a mitre at the same place, but the footing does not. Two square ends in a corner produce a visible step/overlap instead of a closed corner.

### 2.3. No gluing to the bottom of a wall

- A wall has `attach_top` / `attach_base` fields (`crates/core/src/model.rs`, the `Wall` struct), but this is the **wall itself** attaching to a roof/floor: `Model::check` accepts only `Element::Roof | Element::Floor` there (not runs), and the semantics are the opposite — the wall follows the element, not the element follows the wall.
- `Run` (`model.rs`) has **not a single reference field to another element**: only `level`, points, `offset`, `justify`, `system`, `params`. `visit_refs` for runs returns only `r.level`, so `Lookup::referrers` will never show a strip as a "reference" to a wall — neither `relocate` nor cascade deletion knows about such a link.
- When a wall is moved, `relocate` moves only doors/windows (by `o.host`); the strip stays put.
- When a wall's `base offset` changes, its z-extent changes, while the strip's height comes from `run.level + offset` (the strip defaults to `justify = top` — `runs::default_justify`) — and is not corrected by anything.
- The condition "axes coincide + bottom of wall touches top of strip" does not exist anywhere in the code — it simply isn't there.

Result: the footing is completely independent of the wall. The "gluing" mechanism must be built from scratch.

### 2.4. What current tests confirm

- `crates/core/src/structure_tests.rs`, `a_strip_footing_is_a_run_under_a_wall` — pins down the current strip arrangement (category `stripfooting`, type `Strip 600 x 300`, `justify = top`, top of section at the level, axis at −150 mm).
- `crates/core/src/tests.rs`, `moving_one_end_of_a_wall_stretches_it_and_the_walls_joined_there`, `moving_a_wall_drags_the_walls_joined_to_it_and_the_rooms_follow` — pin down wall joint behaviour (ends stretch, a wall abutting the span slides to the intersection).
- There are no analogous tests for "run joints" and "strip behind a wall" — the behaviour was not tested because it does not exist.

---

## 3. Terms

| Term                          | Meaning                                                                                                                                                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Run (linear element)          | A straight element between two points: `Element::Beam`, `StripFooting`, `Pipe`, `Duct`, `Conduit`, `CableTray`, `Wire`, `DownConductor`, `GroundConductor`, `Cable` (macro `run_arm!`, `RUN_CATEGORIES` — 10 categories). |
| Mitre                         | A joint of two elements at an angle: each end is cut by the plane where their faces intersect, the corner "closes" without a step — like a wall corner.                                                                   |
| Cap (trim)                    | The end of an element butting against another's face is cut along that face — like the end of a wall standing on another wall's span (`walls::cap`).                                                                      |
| Join                          | The behaviour "the element joins with neighbours": its outline is mitred/trimmed, and when a neighbour moves it stretches/slides behind the joint. On by default.                                                         |
| Disallow Join                 | Turning joining off for a specific element (Revit *Disallow Join*): its outline stays rectangular, and it takes part in no joint on either side.                                                                          |
| Grip                          | A square at the end of a selected element in plan (the `Modify` tool) which moves that end when dragged.                                                                                                                  |
| Glue                          | The condition "the strip's axis coincides with the wall's axis and the top of the strip's section touches the bottom of the wall": when it holds, the strip follows the wall in plan and vertically.                      |
| Bottom of wall / top of strip | `base_z(wall)` (level + `base_offset`) and the top of the strip's section per its `justify`, type and `offset` (the strip defaults to `justify = top`, so `offset` is the top height).                                    |

---

## 4. Functional requirements

### 4.1. End grips for runs (like walls)

1. In plan, with the **Modify** tool, when **one** element is selected that is a wall **or** a run of any category → grips are drawn at its ends (`start`, `end`) — as for a wall now (hit tolerance ~8 px, single selection only).
2. Dragging a grip behaves as for a wall: the target point is computed from the untouched end with the same snaps and guides (`grip_target`); **Ctrl** — rotate around the untouched end keeping the length; during the drag a preview of the element, a ruler with length and angle, and the snap marker are shown.
3. On release the same `move` command is sent with the `end` parameter (`start`/`end`), only for one run instead of one wall: the core accepts `end` for **one wall or one run** (currently only walls).
4. If a neighbour joins at this end (4.2, joining enabled for both) — the neighbour behaves as walls behind a joint: an end at an end travels along with it, an end on the span slides to the intersection.
5. Dragging the end of a **glued** strip (4.4) counts as a manual change of the strip and removes the gluing (otherwise the next wall change would overwrite what the user manually corrected).
6. All actions go through commands, so undo/redo work as usual.
7. Grips for run ends in 3D view are not required (walls have none in 3D either — there the grips are on levels, the section box and floor points).

### 4.2. Mitred joints for runs (like walls)

1. **Joints between runs** (same category, axes/ends touching, height intervals overlapping — analogous to `stacked` for walls):
   - **corner** (two axes intersect, ends near the node) → **mitre**, like a wall corner: the strip "closes", no square end remains in the corner (the screenshot from the task);
   - **T-junction** (one element's end butts against another's span) → the end is cut along the meeting element's face (analogous to `walls::cap`);
   - **butt joint, collinear** → closing without gap and without overlap.
2. **Joint of a strip with a wall**: the strip's end butting against a wall (the strip's axis perpendicular or obliquely approaching the wall face) is cut along the wall face so that the strip closes into the wall — with no leftover perpendicular end. The wall itself is **not changed** (the strip lies below the wall, its plan outline is the bottom view). This is the special case of the screenshot; it is implemented only for `stripfooting`, joints "beam/pipe ↔ wall" are out of scope (see sec. 12).
3. Different section widths/heights in a joined pair are allowed — the mitre is built from the actual faces (as for walls of different thickness).
4. The joint must be identical in **all** display views:
   - plan: fill and seam are united as for walls (outlines go into `cut_unions`/`cut_edges`), the bottom view and highlight/selection take the same outline;
   - selection, highlight, snaps: end/corner points are taken from the new outline so a click does not land "in mid-air" of the mitre; the run's dashed centre line stays;
   - 3D: the element body is cut by the same joint plane as the plan outline (plan and 3D must not diverge);
   - sections/elevations: the same;
   - IFC: for horizontal joined runs the body is built from the outline with the joint — as already done for walls ("mitred where it joins others as the plan's is").
5. If joining is disallowed for **either** element of a pair (4.3) — both ends in this pair stay rectangular, as now.
6. Joints do not affect centre lines or the element's geometric data (`start`/`end` do not change) — the mitre changes only the display and outline computation, as for walls.
7. Existing wall behaviour does not change: their joints, stretching behind a joint and caps work as before, unless joining is disallowed.

### 4.3. Turning joining off/on (right click)

1. By default joining is **on** for all walls and all runs — the default behaviour must not change (except that runs will now start joining at all).
2. In the context menu (RMB on the selection), if the selection contains walls or runs, a menu item appears:
   - **"Disallow Join"** — if at least one selected element is currently joined;
   - **"Allow Join"** — if at least one selected element has joining disallowed.
     (Two items can be in one group, active per state — as in Revit.)
3. Choosing the item executes a command (everything in the project is commands): `modify` with the join flag on the selected elements. The state is stored in the element and saved in the file.
4. Effect of "Disallow Join":
   - the element's outline is built rectangular (as now), mitres/caps are built neither for it nor for its joint neighbours (the neighbour is not cut towards it);
   - when a neighbour moves/stretches, this element is not stretched behind the joint, and when it moves itself it does not drag neighbours;
   - end grips (4.1) still work — just the end moves, without joined neighbours.
5. The flag applies to the whole element (both ends). Per-end separation, as in Revit (*Disallow Join* on a specific end), is out of scope (sec. 12).
6. On walls the item acts the same as on runs: a wall with joining disallowed is not mitred with neighbours and is not stretched behind a joint (currently walls always join — they could not be turned off).

### 4.4. Gluing a strip to the bottom of a wall (as in Revit)

**Gluing condition** (checked geometrically, for `strip_footing`):

- the strip's centre line coincides with the wall's centre line (direction and position — within the model `TOL`) **and** their projections intersect (the strip really lies under the wall, not beside it);
- the top of the strip's section (per its `justify` and type) touches the bottom of the wall: `bottom of wall` = wall level + `base offset`, within `TOL`.

Requirements:

1. **Automatic gluing.** As soon as the geometry satisfies the condition (a strip was drawn under an existing wall; a wall was moved/lowered onto the strip; the strip was raised to the bottom of the wall) — the strip is considered glued. No separate user action is required.
2. **Following in plan.** Moving, rotating or mirroring a glued wall carries the strip by **the same transformation** (as doors/windows travel with a wall): axis coincidence and touching are preserved. Dragging a wall end (stretching along its own axis) does not move the strip as a whole — after the command the condition is re-checked (if the wall axis stayed in place — the strip is glued, it may just be shorter than the wall).
3. **Following vertically (full).** When the `base offset` or level of a glued wall changes, the strip's `offset` is recalculated so that **the top of the strip's section is exactly at the bottom of the wall**: raise the wall — the strip goes up, lower it — down. This user action is confirmed: "fully behind the bottom of the wall".
4. **Self-restoration.** The condition is pure geometry, checked after every command that changes geometry: axes diverged — gluing removed; the wall was put back — the strip glues again (and follows again).
5. **Removal of gluing** happens if:
   - the axes diverged (the wall was moved/rotated so it no longer coincides with the strip, or the strip was moved manually);
   - the strip was changed manually (points, `offset`, type) — a manual change has priority and breaks the link;
   - the wall was deleted (see item 7).
6. **Only the strip glues** (`strip_footing`): beams, pipes and other runs do not glue to the bottom of a wall.
7. **Deleting a wall** does not delete the strip: the gluing simply ceases to exist (the geometric condition disappears with the wall). Cascade deletion of the strip together with the wall **must not** happen.
8. **Non-glued** strips and old files behave as now — the requirement changes behaviour only when the condition holds.
9. Limitation: a strip with a set `end_offset` (sloped, different end heights) is not detected for gluing — its top along the axis is not flat (acceptable simplification; see sec. 6).

---

## 5. Technical plan (by location, no code)

### 5.1. General idea

- **Grips** — generalise the existing wall mechanism to "wall or run": drop the type restriction in the UI, in the core — allow `end` for one run and teach `relocate` run ends.
- **Joints** — build for runs what walls already have: an element outline assembled taking neighbours into account (`outline_in` modelled on `walls::outline_in`), and stretching behind a joint in `relocate`. One outline is consumed by plan, 3D, sections, IFC and snaps — as for walls.
- **Gluing** — **without storing new state**: it is a geometric condition (sec. 4.4) that the core checks before and after commands that change wall geometry, and by which it recalculates/carries the strip. While the condition holds — the strip follows; it stops holding — it unglues itself; it holds again — it glues itself again.
  - The alternative (storing an explicit "strip → wall" reference in the model) was rejected: it would require a format bump just for the reference itself, reference cleanup on deletion (otherwise the cascade in `delete_indexed` would delete the strip with the wall), handling dangling references in `Model::check`, and the condition "several walls on one axis under one strip" would still have to be solved geometrically. The geometric approach describes exactly what the user formulated ("when the axes coincide and the bottom touches the top — it glues"), requires no migration and creates no dangling references.
  - The `.bimcraft` format changes anyway — but only because of the join flag (item 5.2), not because of gluing.

### 5.2. Data and file format

- `Wall` (`crates/core/src/model.rs`): a new field `join: bool` — joining on by default; serialised to file only when `false` (so old and new files do not grow by default).
- `Run` (`model.rs`): the same field `join: bool` (default `true`).
- Both fields are `#[serde(default)]`, so format-17 and earlier files are read unchanged ("file is read leniently", `lenient.rs` unchanged).
- **Bump `FORMAT` 17 → 18** with an explanation in the doc comment above `FORMAT`: the old version, reading a new file, would silently lose `join: false` (the element joins again) — the same class of change that caused formats 12→13 and 13→14 to be raised. A step in `Model::migrate` is most likely not needed (old file data is not converted, new fields take defaults) — but this must be verified with a format-17 file reading test.
- Gluing (sec. 4.4) is **not stored** — see 5.1; there are no new fields for it.
- `Model::check`:
  - `join` is validated automatically as a bool during serialisation/deserialisation;
  - there are no new references — nothing to check;
  - the existing strip checks (`runs::check`, category/type/`justify`) remain.

### 5.3. Geometry of run joints

- **Run neighbour index.** In `derived::Lookup` (or a separate index modelled on `walls::Index`) — searching for runs near an end/axis: same category, height interval overlap (analogous to `stacked`), end/axis proximity within tolerance. Do not do O(n²) over all runs — model it on the wall tile search (`walls_near_at`, `BoxGrid`).
- **Outline with a joint** — a new function modelled on `walls::outline_in`/`arms`/`cap` (in `runs.rs` or a new module):
  - for a "corner" pair — intersection of faces (mitre);
  - for a "T" pair — cut along the meeting face;
  - for a "butt" pair — closing;
  - for a "strip ↔ wall" pair — cut the strip's end along the wall face;
  - if joining is disallowed for either of the pair, or there are no joints — the previous rectangular outline (`Geom::footprint`) is returned.
- The centre line (`start`–`end`), the element data and its section do not change — the mitre changes only the derived outline (as for walls).

### 5.4. Commands: grips, joints, following a wall

- `move` + `end` (`commands.rs`, `move_elements`): allow `end` for one run (currently only one wall); the error remains for "several elements" and "not a wall/run".
- `relocate` (`commands.rs`) — the main place:
  - **run ends**: the `end` branch for a run — the second end is fixed; runs joined at this end behave as walls behind a joint (an end at an end travels with it, an end on the span slides to the intersection), respecting `join` flags;
  - **walls with joining disallowed**: the existing logic "ends drag neighbours" skips elements with `join = false` (and does not cut them in `outline_in`);
  - **gluing, plan**: for each moved wall, before the move find strips glued by the condition (axis coincidence + touching, sec. 4.4) and apply the same transformation to them as to the wall; on an `end`-drag of a wall do not move the strip as a whole (the condition is re-checked after the command);
  - **gluing, vertical**: when the geometry/heights of a wall change (`modify` `base_offset`/`level` etc.) — for strips glued **before** the change, recalculate `offset` so that the top of the strip's section is at the new bottom of the wall (respecting `justify`);
  - repeat the same logic in all paths where a wall changes: `move`, `rotate`, `mirror`, `modify`, dragging a whole selection. `constraints.rs` moves elements through the common path `commands::move_by` — make sure it goes through the same branches.
- Every command that changes geometry leaves the model in a consistent state: after it the gluing condition either holds (the strip follows) or does not (we do nothing). A separate "rebinding" pass preserving state is not required — checks before/after in the listed branches suffice.
- `delete` (`delete_indexed`): the cascade goes via `referrers`/`visit_refs`; since gluing is not stored as a reference, **nothing needs to be added** — delete the wall, the condition disappears, the strip is intact (requirement 4.4 item 7). Verify this with a test.

### 5.5. Outline consumers (what switches to the new outline)

| Place                                    | What to do                                                                                                                                                                                                                          |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `render.rs`, `stands()`                  | Run outline — via the function with the joint (5.3) instead of the bare `footprint`. `cut_outlines` → `cut_unions`/`cut_edges` (plan fill/seam), `seen_below` (bottom view), extent, highlight and selection go through `stands`.   |
| `render.rs`, centre line                 | The run's dashed centre line stays as is (`start`–`end`).                                                                                                                                                                           |
| `view3d.rs`                              | Rectangular run body — a prism from the new outline; a round run (tube) — cut by the same joint plane so plan and 3D do not diverge.                                                                                                |
| Sections/elevations                      | The same outline/prism as in 3D (walls already go through `prisms_in` with mitres).                                                                                                                                                 |
| `ifc.rs`                                 | A joined **horizontal** run — body from the outline with the joint, as for walls ("mitred where it joins others as the plan's is"); an unjoined one — the previous path (section along the axis). Sloped joined runs — see sec. 12. |
| `snap.rs`, `Scene::of`                   | Run end/corner snap points are taken from the new outline (runs are already among the candidates).                                                                                                                                  |
| `Model::outline`, measurements, `bounds` | Verify that extents/measurements use a consistent outline; if they diverge — bring them to a common source.                                                                                                                         |

### 5.6. Desktop (`crates/ui-egui`)

- `canvas.rs`:
  - `gripped()` — accept not only a wall but "wall or run" (single selection + `Modify`);
  - `grip_target()`, `grip_drag()` — work from the two ends of the selected element (a common shape: gripped/untouched ends), on release — the same `move` command with `end`;
  - `grip_overlay()` — preview of the element outline in the new state: for a wall — as now (`walls::faces`), for a run — the outline with the joint/rectangular (5.3), plus the ruler and snap readout as now;
  - `wall_angles()` — keep for walls; angle entry for runs is optional (up to the implementation, not a blocker).
- `context.rs` (`Action`, `context_items`, `act`): the item(s) **"Disallow Join" / "Allow Join"** in the context menu group when the selection contains walls/runs; handling — the `modify` command with the flag (5.2).
- New UI strings — via `msg!`/`ui_msg!`, then `cargo i18n update` (translations into all `translations/*` catalogues).

### 5.7. What does not change

- How runs are created (`strip_footing`, `beam`, `pipe`…), their types, sections, `justify`, `offset`, `system`, `end_offset`.
- Default wall behaviour (joining on — exactly what exists now).
- Doors/windows in walls, rooms, dimensions, geometry snaps (except that the run outline becomes a joined one).
- Reading old files (see 5.2: the bump is only because of `join`), `lenient.rs` behaviour.
- Existing tests remain green; tests asserting a rectangular strip end in scenarios where a joint is now expected are updated deliberately and with a comment.

---

## 6. Pitfalls (must be taken into account)

1. **Do not break walls.** The new `join` flag defaulting to `true` must give exactly the current `walls::arms`/`cap`/`outline_in` and `relocate` behaviour. Cover with regression tests for existing joint scenarios.
2. **Gluing must not get into `visit_refs`.** It cannot be stored as a reference without cleanup on deletion — otherwise `delete_indexed` (cascade via `referrers`) would delete the strip together with the wall. The geometric approach (5.1) removes this problem; with any decision to store state — the cascade must be checked separately.
3. **`Model::check` does not mutate the model.** All gluing and joint recalculations are in commands, `check` only validates.
4. **Tolerances.** Joints and gluing — only by the model `TOL`, not by pixels and not "by eye": a strip drawn 5 mm from the wall axis neither glues nor mitres. The user works with snaps — axes coincide exactly.
5. **Round runs (pipe, cable, conduit…).** Their plan outline is a rectangle as wide as the diameter, while 3D is a tube. Mitring only the plan would make plan and 3D diverge; the body must be cut by the same plane (5.5). Pipe fittings are not made (out of scope) — the mitre is visual.
6. **Sloped runs** (up to 75°, `runs::check`) and strips with `end_offset`: joints/gluing for them are limitations (4.4 item 9, sec. 12); do not try to solve them in this spec, but do not break them either: without a joint they are drawn as now.
7. **Performance.** The run neighbour index must be tiled/spatial (modelled on `walls::Index`, `BoxGrid`), not an all-pairs loop: the project has thousands of pipes and cables.
8. **Simultaneous selections.** `relocate` receives mixed selections (walls + runs in one `move`) — glued strips must not be moved twice (the wall carried them, and they are also being moved themselves): deduplicate as is done for walls/doors.
9. **Grip dragging.** The preview during the drag must show the new outline (for a wall — as now via `walls::faces`), otherwise the user sees one thing and gets another.
10. **Snaps and selection after the mitre.** If snap/hit points remain from the rectangular outline, a click will land in the cut-off corner ("in mid-air"). All consumers — from one outline source (5.5).
11. **`constraints.rs` (locked dimensions).** It moves elements via `commands::move_by`; if that branch does not go through the new joint/gluing logic, a locked dimension and the geometry will diverge. Check the call path.
12. **Format.** Bumping `FORMAT` 17 → 18 requires: a doc comment above `FORMAT` modelled on the existing ones (what changes and why the old version must not read silently), updating tests tied to the format, and verifying that format-17 files are read by the new version.
13. **Localisation.** "Disallow Join"/"Allow Join" — new strings: `msg!` + `cargo i18n update`, otherwise the menu will show English text.
14. **Hostile parameters.** New fields (`join`) go through the common run `hostile_params_never_panic_or_break_the_model` — make sure garbage values do not break the model.
15. **Collision "two walls on one axis under one strip".** A strip may lie under a series of wall segments on one axis. The geometric condition holds for each segment: moving any of them carries the strip; after that the strip may diverge from the other segments and no longer glue to them. This is acceptable (an edge case), but it must be understood and covered by a limitation test.

---

## 7. Tests

### 7.1. Core (`crates/core`)

**Grips / `move end` for runs:**

1. `move {ids: [one run], end: "start", by: …}` — the end moved, the second end stayed; for a wall — as now.
2. `end` for two elements or for an element that is neither a wall nor a run — error (the previous message for "not a wall", a new one for "not one element").
3. A joined neighbour behind the dragged run end behaves as a wall behind a joint (analogous to `moving_one_end_of_a_wall_stretches_it_and_the_walls_joined_there`): an end at an end — travels along, an end on the span — slides to the intersection; with `join = false` on either — does not move.

**Joints:**

4. Two runs at a right angle (end to end, same category): the outline (`outline_in`) gives a mitre — no square end in the corner, sharp angles at the face intersection point; in plan the fill unites without a step.
5. T-junction: the end outline is cut along the meeting face.
6. Collinear butt joint: closing without gap/overlap.
7. A strip butting against a wall: the strip outline is cut along the wall face; the wall outline is unchanged.
8. Different section widths — mitre by the actual faces; height mismatch (intervals do not overlap) — no joint, rectangular ends.
9. `join = false` on either of the pair — both ends rectangular; neighbours are not stretched on `move`.
10. `modify {join: false}` → the joint disappeared; `modify {join: true}` → the joint returned (the outline reassembled).
11. Wall regression: existing wall joint tests (`moving_a_wall_drags_the_walls_joined_to_it…`, `walls::outline` tests, `walls/index.rs`) — green without changed expectations.
12. 3D: the body of a joined run is cut by the same plane as the plan outline (outline/body
