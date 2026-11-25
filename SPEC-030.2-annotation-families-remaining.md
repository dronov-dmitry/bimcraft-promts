# SPEC-030.2. Annotation families: what is not implemented yet

Status: **draft**, implementation required
Date: 2026-11-23; updated 2026-11-25
Project: BIMCraft (Rust; `crates/core/src/commands.rs` — `annotation`, `modify`, `elements`,
`reload`; `crates/core/src/model.rs` — `Annotation`, `DetailView`; `crates/core/src/render.rs`,
`crates/core/src/sheet.rs`, `crates/core/src/family/annotation.rs`;
`crates/ui-egui/src/properties.rs`, `crates/ui-egui/src/browser.rs`,
`crates/ui-egui/src/context.rs`, `crates/ui-egui/src/canvas.rs`)
Source: review of PR #600 after implementation — what remained open; previously part of the
original SPEC-030.
Related: SPEC-030.1 (what already works), SPEC-030.3 (follow-ups from discussion #617), SPEC-030
(the original requirements), PR #600, PR #598.
Task type: follow-up work on an existing category and the UI; the format does not change, except
for new fields with `default`.

> **In short.** The base family already works (SPEC-030.1). What remains is to finish what makes
> it unusable in an office: placement in a detail view, drag and keys during placement, instance
> Properties, the Annotations group under a view, sheet fields in labels, nested families,
> `Reload`, limits and cache, tests. A separate Detail Item category and parameters on detail
> groups are not done (SPEC-030.3 §3).

## 0. General working rules

1. Language — Russian; code and strings — English; window strings — `msg!`/`t()`.
2. Tests are written per §11; they run only on request, CI separately. Allowed: `check`, `build`,
   `clippy` (without `--all-targets`), `fmt --check`, `i18n check`.
3. **Commits are atomic.** One commit — one logically complete change; do not mix unrelated
   changes. Every commit builds and does not break existing behaviour.
4. Before committing — `git diff` and `git status`; do not capture build artifacts, temporary
   files or other people's changes.
5. After pushing — wait for green CI; on failure find the cause, fix it in a separate atomic
   commit and repeat until the run passes.
6. **Thin UI**: new placement, copying and selection logic lives in the core; the window only
   shows and calls commands. If geometry is needed both by the editor and by rendering, it lives
   in `family/annotation.rs`, not in `ui-egui`.
7. **Do not break anything existing**: the common `move`/`rotate`/`mirror`/`delete`, detail groups,
   `title_block`, component families and their tests stay as they are.
8. The format version is not raised while the addition is readable by old files; new fields —
   `serde(default)`.

## 1. The problem

| Area | Now | Consequence |
|---|---|---|
| Detail view | `annotation` rejects `DetailView`, although the error text lists it | an annotation cannot be placed on a detail from a callout |
| Placement | clicking a plan/ceiling/section/elevation/sheet and the preview work; drag and keys do not | a symbol cannot be dragged from the browser, rotated or cancelled on the fly |
| Instance Properties | only the family and type are visible | parameters, `at`, `angle`, `mirrored` cannot be edited from the window |
| Context menu | `Annotation` does not know it | no Select All Instances, Edit Family, Delete |
| Browser | annotation families only under Families > Generic Annotation | no list of instances under a sheet/view |
| Sheet fields | a label reads only family parameters | a revision stamp and a title frame cannot be assembled |
| Nesting | a family symbol cannot be another family | no composite office symbols |
| `Reload` | existing instances are not updated | editing a family does not reach the project |
| Limits | `MAX_SYMBOLS` = 64, `PaperMark` is computed every frame | a frame/stamp does not fit, the sheet is slow |
| Tests | there is no `#[test]` for annotations | regressions are not caught |

## 2. Goals

1. `annotation` accepts a detail view; `at` — along the section line and upwards, for a plan
   fragment — the parent's `x, y`; the annotation travels with the fragment.
2. Drag a family/type from the Project Browser onto a sheet and a flat view.
3. `Space`/angle, `Tab`/mirror, `Esc` during placement; snapping to the sheet, the stamp and the
   family's lines.
4. Instance Properties: family, type, `at`, `angle`, `mirrored`, parameters, `size`,
   `scale_with_view` (read-only); an edit redraws the view immediately.
5. Edit Type / Family Types and the instance context menu.
6. An **Annotations** group under a sheet/view with a counter and a double click.
7. Copying between views and sheets; angle and mirroring on a section/elevation via `modify`.
8. Sheet fields in labels with family parameter precedence.
9. A nested annotation family inheriting `scale_with_view`.
10. `Reload`/`Load into Project` updates instances while keeping parameters and overrides.
11. PDF/SVG/PNG and `publish` include annotations; IFC/GLB do not (checks).
12. `MAX_SYMBOLS` up to 1000, 10 000 instances per sheet, a `PaperMark` cache.
13. The §11 test plan is closed.

## 3. Non-goals

- A separate **Detail Item** category (SPEC-030.3 §3).
- Parameters on detail group types (SPEC-030.3 §3).
- Individual instance scale: scale is a property of the view.
- A full tabular editor, arrays and row repeaters.
- Raster images and logos inside a family.
- Replacing `title_block` with an annotation family.
- Annotations in 3D, IFC and GLB.
- Collaborative editing and cloud family libraries.

## 4. Key decision

**Everything is built on top of the existing `Annotation` category; there are no new document
kinds and no second parameter engine.**

- A detail view is the same placement target as a plan and a section: `Element::DetailView` is
  added to the `annotation` check, to `annotation_in` and to rendering; `at` is computed in the
  parent's coordinates, as for `text`.
- Sheet fields are not stored in the instance: the label is evaluated at draw time from
  `title_values`, `Sheet.fields` and the project. A family parameter takes precedence over a field
  of the same name.
- Nesting — a family symbol with a reference to another annotation family; the `scale_with_view`
  mode is inherited from the parent, cycles are rejected.
- Limits are raised as a separate step after the cache: `PaperMark` is cached by (family, type,
  parameter values), so that 1000 symbols and 10 000 instances are not computed every frame.
- Priority of work: first the detail view and sheet fields (without them there are no office
  stamps), then Properties/browser, then nesting, `Reload`, limits and tests.

The items motivated in detail by discussion #617 (detail view, sheet fields, nesting, limits,
`family_from_group`) are written out in SPEC-030.3. The work is not duplicated: it is implemented
once, the status is tracked here.

## 5. Stage A. Detail view and placement

### 5.1. Detail view

- `annotation {view}` accepts a `DetailView` by id or name.
- `at`: in a section/elevation — along the section line from its start and upwards from the bottom
  of the fragment; for a plan fragment — the parent plan's `x, y`.
- Drawing — at the detail view's scale (`DetailView::scale`, 1:1…1:50), taking `scale_with_view`
  into account.
- The parent fragment moves or changes its region — the instance travels with it (it is stored in
  the parent's coordinates).
- Fix the error text: today it promises a detail view, but the code rejects it.

### 5.2. Drag and keys

- Drag a family/type from the Project Browser onto a sheet and a flat view; releasing places the
  instance.
- `Space` — rotate during placement; `Tab` — mirror if the family has an orientation; `Esc` —
  cancel, the tool returns, the view does not stay empty.
- The first click after choosing the tool is not lost.
- One click — one Undo step.

### 5.3. Snapping and rejections

- Snapping to the edges of the sheet and the stamp, to the ends/midpoints/centres of the family's
  lines.
- A rejection with a clear text on `{3D}`, a schedule and an empty view; a hint to open a sheet or
  a flat view.

## 6. Stage B. Properties, menu and browser

### 6.1. Instance Properties

- Family, type, `at`, `angle`, `mirrored`, instance parameters, `size`, `size_model`,
  `scale_with_view` (read-only).
- Editing a parameter redraws the view immediately; an unknown name — a message with a list.
- "Edit Type" and "Family Types" open the existing dialogs.

### 6.2. Context menu

- Delete, Select All Instances, Edit Family.
- The actions go through the existing commands (`delete`, `elements` with a filter, opening a
  family).

### 6.3. Browser

- An **Annotations** group under a sheet and under a view with an instance counter.
- A double click selects the instance and opens its view.
- The family is still visible under Families > Generic Annotation.

### 6.4. Copying and editing on the other views

- Copying between views and sheets (today — within a view).
- In a section/elevation, angle and mirroring — via Properties/`modify`, without `rotate`/`mirror`.
- A full UI check of Move/Delete on a section, elevation and sheet.
- One operation — one Undo step; `at`/`angle`/`mirrored`/`view` are visible in `elements`.

## 7. Stage C. Sheet fields and labels

### 7.1. Keys

A label can name a sheet field:

- `{PROJECT}`, `{DATE}` — project fields (`Model::title_values`);
- `{SHEET_NAME}`, `{SHEET_NUMBER}`, `{SCALE}`, `{REVISION}`, `{SHEET_INDEX}`,
  `{SHEET_COUNT}` — sheet fields;
- stamp fields: `title_block.fields` in the format `[{key, label?, scope?: sheet|project, default?}]`,
  `title_values`, `Sheet.fields`.

### 7.2. Rules

1. A family parameter takes precedence over a sheet field of the same name.
2. A `scope: project` field is substituted on any view and on a sheet.
3. A `scope: sheet` field on a view (not on a sheet) is empty and gives a warning in Properties,
   not a placeholder in the drawing.
4. An unknown field — a `{Name}` placeholder and a message with the list of available fields.
5. `{SHEET_INDEX}`/`{SHEET_COUNT}` are counted by sheet order; on a view they are empty.
6. The value is not cached in the instance: renaming a sheet is visible immediately.

### 7.3. Label text

- A check of Cyrillic, vertical alignment and behaviour under rotation/mirroring (wrapping and
  multi-line already work).
- The label stays readable when `mirrored`; when rotated it rotates together with the instance.

## 8. Stage D. Nesting, reload, constraints

### 8.1. Nested annotation family

- A symbol inside a symbol: an annotation family inserted into another annotation one.
- The nested one inherits the parent's `scale_with_view`; its own checkbox inside has no effect.
- The nested family's parameters are visible in the parent's Properties and are passed to the
  instance; the type is chosen by the `family type` parameter.
- Cycles and too deep nesting are rejected with a list.

### 8.2. Reloading a family

- `Reload`/`Load into Project` updates all instances, keeping their parameters and overrides.
- Changing `scale_with_view` after reloading applies to all types and instances.

### 8.3. Dimensions and constraints

- Dimensions, Align, `locked`/`equal`, pivots — in the family's paper millimetres.
- A dimension with a `label` drives the parameter, not the other way round; a constraint conflict
  is rejected with a list.
- If needed — a separate `DimView::Paper`; today the editor uses a Plan view.

## 9. Stage E. Printing, limits and graphics

### 9.1. Printing and export

- PDF/SVG/PNG of a sheet and a flat view include annotations; export layers — annotation, not
  model.
- `publish`/`print_set`: a sheet or view with annotations prints as usual.
- Tests: IFC/GLB/3D do not contain `Element::Annotation`. Today annotations simply do not get into
  the writers; there are no checks that they are absent there.
- An old build opening a new file gets the error "unknown element `annotation`", the file is not
  overwritten.

### 9.2. Limits and performance

- `MAX_SYMBOLS` = 64 → up to 1000, if this does not break the general family model and
  `symbol_set`.
- The instance limit per sheet — up to 10 000.
- `PaperMark` cache by (family, type, parameter values); invalidation — on a change of parameters,
  type and on reload. Today `marks()` is called from rendering on every frame.
- Measurement: a sheet with a typical stamp (dozens of lines and labels) and with a large family;
  PNG/PDF must not degrade noticeably.

### 9.3. Graphics and visibility

- Line styles and weights: Thin/Medium/Heavy and a custom style in PDF; in `false` — model, in
  `true` — annotation.
- A missing fill type — a warning and a safe default.
- Visibility/level of detail on flat views: the symbol's `visibility` applies as for components;
  on a sheet all symbols with `visible = true` are drawn.

## 10. Compatibility

- `.bimc`/`.bimf` files do not change the format version; new fields — `serde(default)`.
- Common commands and detail groups do not change behaviour.
- Annotation families loaded before `Reload` keep drawing; updating — per §8.2.
- New sheet field keys do not conflict with family parameters: the parameter has priority.

## 11. Test plans

There are no new tests in PR #600: not a single `#[test]` was added. The test plan of the original
SPEC-030 §12 needs to be closed:

1. `scale_with_view`: 1:50, 1:100, 1:200, sheet; `size`/`size_model`.
2. Section, elevation, ceiling, detail: the same sizes as on the plan; `at` along/upwards.
3. Changing the view scale after placement.
4. A label with Cyrillic, multi-line, wrapping; a missing parameter.
5. Types, formulas, instance overrides, `Reload`.
6. A dimension with `label`, `locked`, `equal`, a constraint conflict.
7. A masking fill; hatching in PDF.
8. Line styles and weights in `true`/`false`.
9. Rotation 90°, mirroring; label readability.
10. Move/Delete on a plan, in a section, on a sheet; Rotate/Mirror/Copy on a plan/sheet; Undo.
11. View crop; an instance outside the crop.
12. Save/Open `.bimc`/`.bimf`; old projects; component families.
13. IFC/GLB contain no annotations; PDF/SVG do.
14. Browser: Families > Generic Annotation, drag onto a sheet/view, double click, Properties.
15. Regression of `title_block`, `detail_group`, `text`.

> Tests in this SPEC are described but are not run without a separate request; CI is separate.

## 12. i18n and documentation

- New UI strings — via `msg!`/`t()`/`tf()` and `cargo i18n update`; do not forget the translations.
- `docs/areas/families.md` — a section about annotation families with a link to
  `docs/areas/annotation-families.md`.
- `docs/areas/views-and-graphics.md` — the Annotation category, flat views, Visibility/Graphics,
  annotation scale.
- `docs/areas/export.md` — PDF/SVG of a sheet and a view, the absence of annotations in IFC/GLB.
- `docs/areas/schedules.md` — why annotations do not get into schedules.

## 13. Acceptance criteria

- [ ] An annotation is placed in a detail view and travels with the parent fragment; on the other
      views it already works.
- [ ] Drag from the browser; rotation and mirroring during placement; `Esc`; snapping to the sheet
      and the stamp.
- [ ] Instance Properties fully edit the type, parameters, `at`, `angle`, `mirrored`; `size` and
      `scale_with_view` are read-only; the context menu works.
- [ ] A copy between views and sheets; angle and mirroring on a section/elevation; Undo verified.
- [ ] An Annotations group under a sheet/view; selecting an instance from the browser.
- [ ] Sheet fields in labels with family parameter precedence; a warning on a view.
- [ ] A nested annotation family inherits the parent's mode; cycles are rejected.
- [ ] `Reload` updates instances while keeping parameters; a change of `scale_with_view` applies.
- [ ] PDF/SVG/PNG contain annotations; IFC/GLB/3D do not; publish/print work.
- [ ] The §11 test plan is closed; there are no regressions of component families and
      `title_block`.
- [ ] Limits and performance: 1000 symbols, 10 000 instances, sheet rendering does not degrade
      noticeably.
- [ ] Documentation and i18n are updated.

## 14. Risks and questions

| Risk | Answer |
|---|---|
| `DetailView` does not know annotations in `annotation_in` | add it together with accepting the view; otherwise the instance will not be drawn |
| Sheet fields depend on `title_values` and `Sheet.fields` | evaluate at draw time, do not store in the instance |
| 1000 symbols without a cache | the `PaperMark` cache first, then the limit |
| Nesting and cycles | limit the depth and reject a cycle before building `PaperMark` |
| `Reload` loses overrides | update by parameter name, keep unknown ones and report them |
| Drag from the browser conflicts with `Tool::Component` | one placement path, like for components |

1. Is a separate Detail Item category needed within 030, or is Annotation alone enough?
2. Is a Scale with View checkbox needed at the type level rather than for the whole family?
3. Are sheet fields in labels needed in the nearest iteration, or can they be postponed?
4. Is a nested annotation family needed in the nearest iteration?
5. Are raster images/logos inside a family needed?
6. Should the `false` mode on a sheet stay 1 unit = 1 paper mm?
7. Is moving an instance between views needed, or is copying enough?

## 15. Definition of Done

- [ ] `annotation` accepts a detail view; the error text matches the behaviour.
- [ ] Drag, `Space`/`Tab`/`Esc`, snapping to the sheet and the stamp.
- [ ] Instance Properties, the context menu, the Annotations group in the browser.
- [ ] A copy between views/sheets; angle and mirroring on a section/elevation.
- [ ] Sheet fields in labels per the rules of §7.2.
- [ ] A nested family; `Reload`; dimensions in paper mm.
- [ ] PDF/SVG/PNG and `publish`; IFC/GLB are clean (tests).
- [ ] Limits 1000/10 000 and the `PaperMark` cache; measurements recorded.
- [ ] The §11 test plan, i18n and documentation.
- [ ] `check`, `build`, `clippy`, `fmt --check`, `i18n check` — green (per the project rules).

## 16. Code map

- Commands: `crates/core/src/commands.rs` (`annotation`, `modify`, `elements`, `reload`,
  `view_param`), `crates/core/src/clipboard.rs`.
- Model and checks: `crates/core/src/model.rs` (`Annotation`, `DetailView`, `Model::check`),
  `crates/core/src/family/annotation.rs` (`marks`, `bounds`, `check`).
- Rendering and sheets: `crates/core/src/render.rs` (`annotation`, `annotation_symbol`,
  `annotation_in`, `sketch`), `crates/core/src/sheet.rs` (`title_values`, `field_values`).
- UI: `crates/ui-egui/src/properties.rs`, `crates/ui-egui/src/browser.rs`,
  `crates/ui-egui/src/context.rs`, `crates/ui-egui/src/canvas.rs`,
  `crates/ui-egui/src/paper.rs`, `crates/ui-egui/src/orbit.rs`.
- Documentation: `docs/areas/annotation-families.md`, `docs/areas/families.md`,
  `docs/areas/views-and-graphics.md`, `docs/areas/export.md`, `docs/areas/schedules.md`.
