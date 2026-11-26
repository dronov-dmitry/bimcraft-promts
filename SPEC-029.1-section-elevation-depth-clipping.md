# SPEC-029.1. Near and far clip offsets for sections and elevations

Status: **draft**, implementation required (in the BIMCraft-033 snapshot `depth` already
exists, `near` does not)
Date: 2026-11-26
Project: BIMCraft 033 (snapshot 2026-11-26); `crates/core/src/model.rs` — `Section`,
`crates/core/src/commands.rs` — `elevation`, `section`, `modify`, `elements`,
`crates/core/src/render.rs` — `section_camera`, `Clip`
Source: the first part of the split SPEC-029. The "Wall elevations" plugin (SPEC-029.2) cannot
clip the model by depth on its own: only the core does that.
Task type: core enhancement, additive; the `.bimc` format gains an optional `near` field.

> **In short.** A section/elevation already has `depth` — the far limit from the view line. It
> needs the same **near** limit `near`: the visible band starts not at the line but `near` mm
> away from it. By default `near = 0`, that is, the view line, as now. The SPEC-029.2 plugin may
> omit `near`, but if it sets one, it sees only the band `[near, depth]`, not the whole house.
> `crop` still cuts along the line and by height; that is a different dimension.

## 0. General working rules

1. Language — Russian; code, error strings and identifiers — English; window strings — `msg!`/`t()`.
2. Tests — only on request; CI separately. `check`, `build`, `clippy` (without
   `--all-targets`), `fmt --check`, `i18n check` are allowed.
3. No new dependencies; `unsafe` is not added.
4. Old projects are read as is: no `near` — that is `0` (the line), no `depth` — everything is visible.
5. Do not extend the task: no `from`, `depth_from`, negative `near`, section box,
   plan view range or depth panels. Only the two section/elevation limits.

## 1. The essence of the problem

The SPEC-029.2 plugin places an elevation 2 m from a wall, crops it by length and height, and
wants to see only that wall and a small margin behind it.

| What the core already does | What is missing |
| -------------------------- | --------------- |
| `elevation {start, end, depth, name}` | without `depth` the view sees the whole house behind the wall |
| `Section.depth` (far limit) | the near limit is hard-coded to `0` |
| `crop {view, region}` | cuts along the line and by height, but not by depth |
| `scale`, `place`, `sheet`, `text` | that is enough |
| `elements` by walls | no curtain-wall flag for `include_curtain: false` |

The far limit already exists; the plugin cannot make the near one itself: hiding categories also
removes the wall it needs, and `crop` knows nothing about depth. So both limits must live in the
core.

## 2. Goals

1. `elevation` and `section` accept `near` and `depth` — distances in mm from the view line
   forward along `looking()`.
2. `near` missing or `null` — `0`, that is, the view line; `depth` missing or `null` —
   no far limit, everything is visible.
3. `modify {id, near?, depth?}` changes both limits; `elements` reports both.
4. The limit applies to the section/elevation drawing itself: screen, `render`, sheet,
   PNG/SVG, point clouds; it does not apply to plans, 3D, IFC or schedules.
5. `crop`, `scale`, `place` and `sheet` do not change `near`/`depth` and do not cancel them.
6. `elements` reports `curtain_wall: true|false` for a wall — the minimal flag so that the plugin
   can implement `include_curtain: false`.
7. No new commands or model fields beyond `near` and the above.

## 3. Non-goals

- No `from`: it would be a second name for `near`; use `near`.
- No `depth_from`, no binding of the limits to a level or a wall.
- No negative `near` (including the space in front of the line): v1 only goes forward from the line.
- No setting the limits in the properties window beyond the existing fields.
- No per-element clipping, section box, plan view range.
- No uniqueness of view names: the plugin checks the names itself (SPEC-029.2 §6.2).
- No changes to IFC, schedules, `quantities`, `takeoff`.
- No `brief`/projection of the `elements` fields: a plugin token budget is not a reason to
  change the core.

## 4. Core contract

### 4.1. Model

`Section` (shared by section and elevation):

```rust
pub near: Option<f64>,   // new field
pub depth: Option<f64>,  // already exists
```

- `near: None` — the same as `Some(0.0)`: the band starts at the view line.
- `near: Some(n)` — the visible band starts `n` mm from the line in the `looking()` direction.
- `depth: None` — no far limit, visible to infinity.
- `depth: Some(d)` — the band ends `d` mm from the line in the `looking()` direction.
- `n` — a finite number, `0 <= n <= MAX_COORD`; `n = 0` is allowed.
- `d` — a finite number, `0 < d <= MAX_COORD`.
- If both are set: `n < d - TOL`; otherwise the band is empty or inverted — an error.
- If `depth` is missing, `near` may be any `>= 0`: visible from `near` to infinity.
- `crop: Option<[f64; 2]>` — still only bottom and top; not mixed with `near`/`depth`.
- Both limits are a distance in mm, not an absolute level.

### 4.2. Commands

| Command | Requirement |
| ------- | ----------- |
| `elevation {start, end, near?, depth?, name?}` | `near`/`depth` — the limits of the band |
| `elevation {on\|reference, at, near?, depth?, name?}` | the same for an elevation along a line/element |
| `section {start, end, near?, depth?, name?}` | the same for a section |
| `modify {id, near?, depth?}` | changes the limits; `null` removes (near → 0, depth → no limit) |
| `elements {category:"elevation"\|"section"}` | reports `near` (0 if missing) and `depth` (null if missing) |
| `render` of a view, sheet, viewport | drawn taking both limits into account |
| `crop`, `scale`, `place`, `sheet` | do not change `near`/`depth` |

`near` and `depth` are not parameters of `crop` and are not set via `view_range`.

### 4.3. Clipping semantics

1. The `looking()` direction — perpendicular to the `start → end` line, to its left.
2. The near plane is parallel to the view line and stands `near` mm from it in the `looking()`
   direction; the far plane stands at `depth` mm.
3. Everything that intersects the band `[near, depth]` is visible; elements before `near` are not
   drawn, elements beyond `depth` are not drawn.
4. With `near = 0` the view starts at the line, as before; with `near > 0` the first `near` mm of
   the band are cut off. If `near > 0`, the section plane at the line is not visible either:
   that is the deliberate price of the band; for an ordinary section/elevation `near = 0`.
5. `near = depth` or `near > depth` — an error; a band of zero/negative thickness is not created.
6. `depth = null` — no far limit; `near > 0` then hides only the beginning.
7. `crop` still limits the view along the line and by height; `crop` and the limits work together.
8. `crop` may move the line along itself (as now); the `near`/`depth` distances from the line do
   not change, because they are measured from the line, not from the origin.
9. Scale, placement on the sheet, underlay and annotation do not change the limits; the
   annotation itself does not have to be clipped if it refers to the visible part.
10. Plans, the 3D view, IFC, schedules and `quantities` do not read the `near`/`depth` of a
    section/elevation.

### 4.4. Validation

- `near`: finite, `0 <= near <= MAX_COORD`.
- `depth`: finite, `0 < depth <= MAX_COORD`.
- If both are set: `near < depth - TOL`; otherwise an error with both values.
- A violation is a command error; the model and history do not change; `Model::check` checks the
  same for a file.
- Old files without `near` are read as `0`; without `depth` — with no far limit.
- `near`/`depth` set in a project with other units are still in mm in JSON and in commands.
- Format: `near` is a new field. If a build without it would silently ignore `near` and show
  extra geometry, raise `FORMAT` and add a `Model::migrate` step; the new build reads old files
  with `near = 0`.

### 4.5. Minimal `elements` data

Besides `near`/`depth` on a section/elevation, the plugin needs one flag on a wall:

```json
{"id": 100123, "category": "wall", "type": "Timber panel 200", "curtain_wall": false}
```

- `curtain_wall` — `true` if the wall type is a curtain wall (`types.wall[type].curtain.is_some()`),
  otherwise `false`.
- The field is output for every wall; it is not written to the file and `modify` does not accept it.
- The name is chosen so as not to collide with the existing `curtain` field (curtain-wall edits).
- The rest of the fields the plugin already gets: `id`, `guid`, `type`, `level`, `start`, `end`,
  `arc`, `height`, `base_offset`, `top_level`, `top_offset`, `attach_top`, `attach_base`,
  `params`, `length`, `thickness`.

If `curtain_wall` already exists under another name in the target branch, use it and do not add a
second one.

## 5. Work

### A. Model and validation

- Keep `Section.depth: Option<f64>`.
- Add `Section.near: Option<f64>` with `default`; `None` = 0.
- `Model::check`: `near` finite, `0 <= near <= MAX_COORD`; `depth` as now (`size`, `> 0`);
  if both — `near < depth - TOL`.
- Check serialization: an old file without `near` — `None`; a new one with `near` — read back.
- Raise `FORMAT` if an old build, ignoring `near`, would show the wrong band.

### B. Commands

- `elevation` and `section`: accept `near`, including in the `on`/`reference` branch.
- `modify`: allow `near`, `null` → 0; `depth`, `null` → no limit.
- `elements`: report `near` and `depth` on a section/elevation.
- Do not add `from`, `depth_from` or new commands.

### C. Drawing

- `section_camera` builds the band from the model. In `Clip` the `near` field is the depth
  **toward the camera**, not the distance forward from the line; therefore:

  ```rust
  let near = v.near.unwrap_or(0.0);
  let far = v.depth.map(|d| d - near); // Clip.far is measured from Clip.near
  Clip { near: -near, far, .. }
  ```

  With `near = 0` this is the previous `Clip { near: 0.0, far: v.depth }`. With `near = 500` and
  `depth = 2150` the band is from 500 to 2150 mm from the line.
- Check that both limits reach:
  - the on-screen section/elevation drawing;
  - `render` of a view;
  - the view on a sheet (viewport);
  - point clouds (both the near and the far limit);
  - SVG/PNG/PDF output of a sheet.
- Check every place that currently reads only `depth`: the level/grid filter, point clouds, the
  section picture; they must take `near` into account too.
- Do not touch plans and 3D.

### D. `curtain_wall`

- In `element_json` for `Element::Wall`, add `curtain_wall` from the wall type.
- Do not overwrite the existing `curtain` field (curtain-wall edits).
- Add a test: a curtain wall — `true`, an ordinary one — `false`.

### E. Tests

See §6.

### F. Documentation

- `docs/areas/views-and-graphics.md`: `near`/`depth` are the near/far limits from the line;
  `crop` does not change them; `near = 0` — the line.
- `docs/areas/plugins.md`: a short example of a plugin that uses `depth` and the optional `near`.
- `docs/areas/structure-and-mep.md` or `views-and-graphics.md`: a reference to SPEC-029.2.

## 6. Test plan

1. **`depth` only.** An elevation 2000 mm from a wall, `depth = 2150`: the wall is visible; a
   second wall 3000 mm behind the line is not visible; no `near` — the band starts at the line.
2. **`near`.** `near = 500`, `depth = 2150`: an object 300 mm from the line is not visible, an
   object 700 mm away is visible; the wall at 2000 is visible.
3. **`near = 0`.** As before: visible from the line; `elements` reports `near: 0`.
4. **`near` only.** `near = 500`, `depth: null`: up to 500 is not visible, beyond that everything
   is visible.
5. **`near >= depth`.** `near = 2150`, `depth = 2150` — an error; `near = 3000`, `depth = 2150` —
   an error; the model and history do not change.
6. **Negative/NaN.** `near: -1`, `near: NaN`, `depth: -1`, `depth: NaN` — an error.
7. **`modify`.** `modify {id, near: 400, depth: 3000}` changes the band;
   `modify {id, near: null}` returns 0; `modify {id, depth: null}` removes the far limit;
   `elements` shows the new values.
8. **`crop`.** After cropping by length and height `near`/`depth` are unchanged; the visible band
   is the same.
9. **`scale`/`place`.** Scale and sheet do not change the limits; the viewport draws the same band.
10. **`render`.** A PNG/SVG of a section/elevation with `near`/`depth` contains no objects outside
    the band; without the limits it does.
11. **Sheet.** A PDF of a sheet with a viewport shows the same as `render`.
12. **Point cloud.** Points before `near` and beyond `depth` are not drawn.
13. **File.** Save/open preserves `near`/`depth`; an old file without `near` opens as `near = 0`;
    a file without `depth` — with no far limit.
14. **Doors/windows/framing.** Openings and framing in the wall are visible within the band;
    elements outside the band are not visible.
15. **`curtain_wall`.** A curtain wall — `true`, an ordinary one — `false`; `modify` does not
    accept this field.
16. **Regression.** `section` and `elevation` without `near`/`depth` are drawn as before the task;
    plans and 3D are unchanged.

## 7. Acceptance criteria

- [ ] `near` and `depth` on a section/elevation are the near and far limits from the line along
      `looking()`.
- [ ] `near` missing/`null` — 0; `depth` missing/`null` — no far limit; `near = 0` is allowed.
- [ ] `near >= depth`, negative values and NaN are rejected; the model does not change.
- [ ] `elevation`, `section`, `modify`, `elements` support `near` and `depth`.
- [ ] `render`, sheet, viewport and point clouds take both limits into account; plans/3D/IFC do not.
- [ ] `crop`, `scale`, `place` do not change `near`/`depth`.
- [ ] `elements` reports `curtain_wall` for a wall.
- [ ] No new commands, `from`, `depth_from`; the format is updated per project rules.
- [ ] Tests §6, documentation §5F.

## 8. State in the BIMCraft-033 snapshot and what remains

| Requirement | State | Action |
| ----------- | ----- | ------ |
| `Section.depth: Option<f64>` | present | keep, cover with a test |
| `Section.near: Option<f64>` | **absent** | add |
| `elevation`/`section` accept `depth` | present | add `near` |
| `modify` changes `depth` | present (generic `modify`) | add `near` |
| `elements` reports `depth` | present (`Section` serialization) | add `near` |
| `Clip { near: 0, far: depth }` | present, `near` hard-coded to 0 | replace with `near` from the view |
| `render`/sheet/point clouds | take `depth` into account | check `near` |
| Validation `depth > 0` | present (`size` in `Model::check`) | add the pair `near < depth` |
| `curtain_wall` in `elements` | **absent** | add |
| Explicit limit tests | **absent** | add §6 |
| Documentation | partial | extend §5F |

Conclusion: the new capabilities in the core are `near`, validation of the pair of limits and
`curtain_wall`; the rest is tests and documentation for the already existing `depth`.

## 9. Risks and questions

| Risk | Answer |
| ---- | ------ |
| `near > 0` also removes the section plane at the line | that is the price of the band; for an ordinary section `near = 0`, the plugin defaults to 0 |
| `near` and `depth` get confused | `near` is the near limit, `depth` the far one; both go forward from the line |
| `Clip.near` in the renderer points toward the camera | in `section_camera`: `Clip.near = -near`, `Clip.far = depth - near`; otherwise the band is on the other side |
| `near` and `crop` | `crop` is along/up, `near`/`depth` are in depth; independent |
| Negative `near` in an old file | `Model::check` rejects it; there is no migration |
| An old build will ignore `near` | raise `FORMAT` if it would otherwise show the wrong band |
| `curtain_wall` will collide with `curtain` | the name is different; `curtain` remains the curtain-wall edits |
| The plugin will not have enough token budget for 34 walls | a plugin limitation: reduce `limit`; do not add `brief` to the core |

1. Is a negative `near` needed (including the space in front of the line)? In 029.1 — no; a
   separate task if it is needed.
2. Should `near = 0` be stored in the file? No, `None` and `0` are equivalent; write it only
   when `> 0`.
3. Labels in properties: `Near Clip Offset` and `Far Clip Offset`; the value is in mm.

## 10. Definition of Done

- [ ] `Section.near`, validation of the pair, format.
- [ ] `elevation`, `section`, `modify`, `elements` with `near`/`depth`.
- [ ] `section_camera` and every place that reads `depth` take `near` into account.
- [ ] `curtain_wall` in `elements` for a wall.
- [ ] Tests §6 green (on request).
- [ ] Documentation §5F.
- [ ] SPEC-029.2 accepts `near` (default 0) and requires no other core changes.
