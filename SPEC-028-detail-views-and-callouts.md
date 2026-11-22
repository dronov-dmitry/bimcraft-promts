# SPEC-028. Detail views and callouts: 1:10 and 1:5 details

Status: **implemented** (2026-11-22), the functionality is done; PR #589
Date: 2026-11-22
Project: BIMCraft 028 (Rust; `crates/core/src/commands.rs` — `elevation` :1457, `crop` :1538,
`detail` :1593; `crates/core/src/render.rs`; `crates/core/src/sheet.rs`; `crates/core/src/dims.rs`)
Source: task of 2026-11-20 "construction documentation section of a wooden house": 15 details had to be drawn in CADCraft
because BIMCraft has no callout views.
Related: SPEC-029 (panel elevations), SPEC-030 (schedules on sheets), SPEC-032 (framing).
Task type: new view and annotation; the `.bimc` format is extended, old projects are read as is.

> **In short.** BIMCraft has sections, elevations, `crop`, `detail_line`, `filled_region`, `text`,
> `repeating_detail` — but no **callout view**, as in Revit: you cannot select a fragment on a plan
> or section and get a 1:10/1:5 view linked to it, which is cropped to that fragment and
> marked on the parent view. Details are drawn in CAD, and the "detail ↔ model" link is lost.
> A detail view and a callout marker must be added without touching the existing views.

## 0. General working rules

1. Communication language — Russian; code, comments, strings — English. Window strings — via `msg!`/`t()`.
2. Tests (`cargo test`, `nextest`, doctest) — only at the user's direct request; CI — on it.
   Allowed are `cargo check`, `cargo build`, `cargo clippy` (without `--all-targets`), `cargo fmt --check`,
   `cargo i18n check`.
3. Commits are atomic: view model and file → `detail_view` → marker and link → annotation → sheets and
   printing → i18n/documentation.
4. No new dependencies; no `unsafe` is added.
5. Old projects open without migration: a detail view is a new view type.

## 1. The problem: what exists today

| Capability | Command (028) | Limitation |
|---|---|---|
| Section along a line | `section {start, end, depth, name}` | a view of the whole section, not a fragment |
| Elevation / wall face | `elevation {on, facing, at, …}` (`commands.rs:1457`) | a side of the building or a wall, not a detail |
| Crop | `crop {view, region}` (`:1538`) | manual, with no link and no marker |
| Level of detail | `detail {view, fine}` (`:1593`) | does not create a fragment |
| 2D | `detail_line`, `filled_region`, `repeating_detail`, `text` | present, but in view coordinates |
| Sheet | `sheet`, `place` (`sheet.rs`) | places a view, does not create a detail |

Workaround today: `section` → `crop` to the detail → `scale 1:5` → draw `detail_line` on top. This works,
but there is no marker on the parent, no link (the detail does not follow the model), the crop is manual,
the 2D is not tied to the geometry, and details are not grouped in the browser.

## 2. Goals

1. `detail_view {parent, region, scale, name}` — a detail view cropped to a fragment of the parent.
2. A callout marker on the parent (a circle with a number and a label), linked to the detail in both directions.
3. Detail annotation (lines, fills, text, dimensions) **in model coordinates**, not sheet coordinates.
4. Placing the detail on a sheet and printing it without losing the link.
5. Compatibility and no regressions in the existing views.

## 3. Non-goals

- Not a 2D editor: annotation over the model only.
- No automatic detailing (the detail does not draw itself).
- No nested details (a detail of a detail) and no 3D sections in v1.
- No export of a detail as a separate DWG in v1.

## 4. Key decision

**A detail is a view with a parent and a rectangular fragment, not a separate drawing.**

| Option | Consequence |
|---|---|
| A sheet-detail with 2D | the link to the model is lost, as in CAD |
| Manual crop + scale | no marker and no link, manual work for every detail |
| **A detail view with a parent** | marker, link, automatic recalculation, grouping |

Model: `View { kind: Detail { parent: ViewId, region: Rect }, scale, name }`. Rendering uses the same
pipeline as a section/plan, with clipping to `region`; the annotation is in model coordinates.

## 5. Stage A. View model and file

- `ViewKind::Detail { parent, region }`; `region` in the parent's coordinates (mm).
- Serialization with `default`; old files are readable.
- `elements` returns `kind: "detail"`, `parent`, `region`, `scale`.
- Project browser: the detail under its parent (`Plan 1 > Details > D-03`).

## 6. Stage B. The `detail_view` command

```json
{"parent": "План 1", "region": [[3000,2000],[3800,2600]], "scale": 5,
 "name": "У-03. Панель — фундамент", "number": "У-03"}
```

- `parent` — a plan, section or elevation (name or id).
- `region` — [[x1,y1],[x2,y2]] inside the view, no more than 1/4 of its area.
- `scale` — 1…50 (default 5).
- Creates a view labelled `<number> <name>`; returns `{id, name, scale, parent}`.
- `modify {id, region}` moves the fragment; the marker and the crop follow.
- Deleting the detail deletes the marker; deleting the parent deletes the details (with a warning).

## 7. Stage C. Callout marker

- A circle Ø 8 mm on paper, a number, a label, a line to the centre of `region`.
- A view annotation, not a model element; it does not appear in schedules.
- Double-click opens the detail; in the detail there is a back-link to the parent.
- Marker type (`detail_marker_type`): radius, font, arrow; default Round 8 mm.

## 8. Stage D. Detail annotation

- `detail_line`, `filled_region`, `text`, `dimension`, `repeating_detail` accept a detail in `view`.
- Coordinates are the parent's model coordinates; rendering multiplies them by the scale.
- Dimensions in a detail attach to model faces; text is scaled by the view.
- When `region` changes, the annotation stays in the model; anything outside the edge produces a warning.

## 9. Stage E. Sheets and printing

- `place {sheet, view: "D-03", scale: 5}`; the label reads "D-03 · 1:5".
- PDF/SVG — like an ordinary view; a mismatched scale produces a warning.

## 10. Test plan

1. Creating on a plan and on a section; `elements` — `kind: detail`.
2. Detail pixels = parent fragment × scale.
3. The marker appears/disappears together with the detail.
4. `modify {region}`: the detail and the marker follow, the annotation stays in the model.
5. Save/open: the link and the region are preserved (pixels match).
6. A sheet with the detail in PDF, the label and the scale correct.
7. Regression: plans, sections, elevations and sheets are unchanged.

## 11. i18n and documentation

- "Detail View", "Callout", "Parent View", "Region", "Detail Marker".
- `docs/areas/views-and-graphics.md` — the section on views; `looking-at-results.md` — how to read details.

## 12. Acceptance criteria

- [x] `detail_view` creates a 1:5 view cropped to the fragment.
- [x] A marker with a number and a link in both directions.
- [x] Annotation in model coordinates that survives a change of region.
- [x] A sheet and PDF/SVG with the label.
- [x] Old projects open unchanged.
- [x] Tests from §10, i18n, documentation. The tests were added in `detail_tests.rs`; they were not run locally
      (project rule), verification — CI in PR #589.

## 13. Risks and open questions

| Risk | Answer |
|---|---|
| The detail "drifts" when the parent changes | annotation in model coordinates, the region is fixed |
| Many details → a heavy file | only the region and the annotation are stored |
| Expectation of a "real" 2D editor | §3: annotation over the model, not CAD |
| The marker on different sheets | in the view's paper millimetres, like dimensions |

1. Should the detail number be automatic (`D-01`…) or manual?
2. Are details needed on elevations?
3. Is export of a detail to DWG needed?

## 14. Definition of Done

- [x] A detail view in the model, the file and `elements`.
- [x] `detail_view`, `modify {region}`.
- [x] The marker and the back-link.
- [x] Annotation in model coordinates, cropping; there is no separate warning about annotation
      outside the edge — it simply does not get into the crop.
- [x] Sheets and printing.
- [x] Tests, i18n, documentation.

## 15. What was actually implemented (2026-11-22)

### 15.1. Model and file

Instead of the planned `ViewKind::Detail`, a separate element
`Element::DetailView(DetailView)` was introduced with the `detail` category (serde tag `"detail"`):

- `name` — the view name (`View Name`);
- `parent` — the GUID of the parent plan, section or elevation;
- `region` — `[P2; 2]` in the parent's coordinates;
- `scale` — 1 : n, stored in the view itself;
- `number` — the mark number (`D-03`, `D-01`);
- `mark` — optional position of the mark in the parent's coordinates;
- `graphics` — visibility, overrides, template.

Old projects are read without migration: a missing `mark` is `None`. `elements` now returns
`kind: "detail"`, `parent`, `parent_id`, `parent_name`, `number`, `region`, `scale`, `mark`.

### 15.2. Command and behaviour

- `detail_view` (UI label — **Callout**, alias `DV`): the parent is a plan, section or
  elevation; the region is no more than 1/4 of the parent's area; the scale is 1…50 (default 5); the number
  is generated (`D-01`, `D-02`, …) if not set; the name is `<number> <name>`.
- `modify {region}` changes the fragment; `modify {mark}` sets the position of the mark; `scale` changes
  the Callout's scale.
- `move` moves the Callout together with `region` and `mark`; rotating and mirroring a Callout
  is not allowed — it is only moved.
- Deleting a Callout removes its boundary and mark; deleting the parent deletes the Callout and everything
  drawn in it.

### 15.3. The Callout boundary and mark

- On the parent a **rectangular boundary** of `region` is drawn (`render::detail_boundary`).
- The mark is a circle Ø 8 mm on paper with a label; by default it sits at the north-east corner, and its
  position can be dragged (`DetailView::mark`).
- **The leader runs from the mark to the boundary of the crop and stops on it**, not at the centre
  (`render::detail_boundary_point`).
- The mark shows the view name without an automatic number prefix: `D-01 Corner` → `Corner`;
  after the `View Name` is renamed, the mark immediately shows the new name. A long name is
  shrunk to fit inside the circle.
- The boundary is edited with handles (4 sides) **both on the parent view and on the Callout view
  itself**. Dragging a side changes `region`, so the boundary and the Callout's area
  move in sync in both directions.
- Double-clicking the mark or the row in the browser opens the Callout.

### 15.4. Annotation

- `detail_line`, `filled_region`, `text`, `dimension`, `repeating_detail` accept a Callout in
  `view`.
- For a plan Callout, the coordinates are the parent's model coordinates.
- For a section/elevation Callout, `section_geometry` clips the parent's line to `region`, and
  `section_finish` shifts the annotation to the start of the region; `drawing_plane` takes the parent's
  line. The annotation therefore stays in the parent's coordinates and survives a change of
  `region`.
- Dimensions in a section/elevation Callout attach to levels, grids perpendicular to them, and the parent's
  detail lines.

### 15.5. Sheets and printing

- `place` places a Callout like an ordinary view; the label is the name and the scale (`1 : 10`).
- `sheet::framing_in` uses `render::detail_extent`, so the frame equals
  `region / scale`.
- The viewport follows the Callout's scale, as with other views; placing it with a different scale
  produces a warning.
- PNG/PDF of a sheet and a direct `render {view: "…"}` (`detail plan` / `detail section`) work.

### 15.6. Interface

- Manual creation: **View → Create → Callout** (the only button; the button was removed from Annotate → Detail
  at the user's request).
- Options Bar: `Scale 1:`, `Name`, `Number`.
- Browser: a **Callouts** group, inside it subgroups by parent view.
- The scale is managed like in the other views: the bottom **Scale** bar and
  **Properties → Graphics → View Scale** (a drop-down list) plus `Scale Value 1:`.
- The Properties of the selected Callout show only its own parameters (`View Name`, `Number`,
  `Parent View`, `Region`, the Callout's own Graphics) rather than the parent view's properties.
- Renaming `View Name` updates the mark.

### 15.7. i18n, documentation, tests

- Strings: **Callout**, **Callouts**, **Parent View**, **Region**, **Detail Marker**;
  the catalogues were updated with `cargo i18n update`, `cargo i18n check` — green.
- `docs/areas/views-and-graphics.md` — the section "Callout: детальные виды и выноски";
  `docs/areas/looking-at-results.md` — how to verify via the CLI.
- `crates/core/src/detail_tests.rs` gained tests: creating a plan Callout, the boundary and
  the annotation when the region changes, a section Callout. The tests were not run locally due to the
  project rule; CI — in PR #589.

### 15.8. Differences from the original plan

| SPEC-028 plan | Actually |
|---|---|
| `ViewKind::Detail { parent, region }` | `Element::DetailView` (category `detail`) |
| the marker draws a line to the centre of `region` | the leader stops at the boundary; at the user's request |
| the marker is annotation only and does not move | the mark is stored in `DetailView.mark` and can be dragged |
| the `detail_marker_type` marker type | a fixed round marker Ø 8 mm; there is no separate type |
| "Detail View" in the UI | the user-facing name is **Callout** everywhere, as in Revit |
| `scale` only in `Graphics` | the scale is stored in `DetailView.scale`, but is managed by the bottom bar and Properties like other views |
| a warning about annotation outside the edge | not implemented; the annotation is simply cropped |

### 15.9. Answers to the questions in §13

1. The number is issued automatically (`D-01`…), but it can be set manually (`number`).
2. A Callout works on plans, sections and elevations.
3. Exporting a separate Callout to DWG was not done in this iteration.
