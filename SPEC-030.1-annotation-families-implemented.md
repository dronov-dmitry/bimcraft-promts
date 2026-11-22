# SPEC-030.1. Annotation families: implemented in PR #600

Status: **implemented** in the `annotation-families-030` branch (PR #600), on top of PR #598
Date: 2026-11-23
Project: BIMCraft (Rust)
Related documents: `ТЗ-030.2-annotation-families-remaining.md` — what is left;
`ТЗ-030-annotation-families.md` — the original requirements (kept as history).

## 1. Scope of this document

This document collects only what already works in the current PR. The original ТЗ-030 has not been
rewritten; its unimplemented part is moved to ТЗ-030.2.

The branch is based on **PR #598** (@emilianocapasso, "Family editor: Modify tools, clipboard,
Measure and Create Similar") and extends it rather than duplicating it. There are no Move/Copy/
Rotate/Mirror/Measure of its own in the branch — the #598 tools are used. There are no conflicts
with #598. The branch is additionally merged with the current `main` so that the PR is mergeable.

## 2. Model and file

- `Category::Annotation` (in the file `"annotation"`, in the UI — **Generic Annotation**);
  `Family::is_annotation()`, `Family::space()`, `Family::scale_with_view` (defaults to `true`).
- `Symbol::fill` — `rings` and the fill type name; `Symbol::style` — the line/arc style.
- `Label::text_type`, `Label::valign`, `Label::wrap` — the text type, vertical alignment and the
  label's wrapping width.
- `Element::Annotation`: `view`, `family`, `type`, `at`, `angle`, `mirrored`, `params`;
  `annotation_plan()` returns the view, so the common Move/Copy/Delete work the same as for `text`.
- New module `crates/core/src/family/annotation.rs`: `PaperMark`, `marks`, `marks_indexed`,
  `bounds`, `check`. Parameters and formulas are read by the same traversal as `marks_metered`, but
  without a level or forms.
- `family` accepts `category: "annotation"`, `scale_with_view` and symbols with `fill`;
  `family {options}` reads and changes `scale_with_view`.
- Old `.bimc`/`.bimf` files open; the new fields have a `default`. The file format version was not
  bumped.
- Model limits: up to **64 symbols** per family (`MAX_SYMBOLS`), up to **64 points** in a line
  (`MAX_PROFILE`).

## 3. 2D content and rendering

- Lines, arcs, dots, text, labels and fills. Drawing order: masking fills, then fills, lines/arcs/
  dots, then text and labels.
- `style` — a project line style; without it — `Thin Lines` with a warning.
- `text_type` — a project text type; without it — the previous BIMCraft stroke font. `valign` and
  `wrap` work for labels.
- A dot is drawn as a **circle**, not a square; the default radius is 1 mm.
- A circle is an arc from 0–360°; a rectangle is a closed line.
- A `Masking` fill hides the model beneath it, but not other annotations.
- Rendering on sheets, plans, ceiling plans, sections and elevations; `render {family}` draws an
  annotation family on its own, without a project.
- `scale_with_view` behaves as follows:

| `scale_with_view` | Family unit | 1×1 square on a 1:100 plan | On 1:100 paper | On a sheet |
|---|---|---|---|---|
| `true` | 1 mm on paper | 100×100 mm of model | 1×1 mm | 1×1 mm |
| `false` | 1 mm of model | 1×1 mm of model | 0.01×0.01 mm | 1×1 mm |

- The boundary of a Filled/Masking Region is invisible in the project and on the sheet; in the
  family editor a dashed hint outline is shown. It does not make it into the family, the sheet or
  the project.
- Selection: clicking an annotation selects the instance; the highlight is the same as for the
  other annotations.

## 4. Family editor

- `File > New > Family > Generic Annotation` opens a paper view; an annotation has no `{3D}`,
  ceiling or elevation tabs.
- The plan frame is `Width × Height`, by default **100 × 50 mm**. The family content lives in
  paper millimetres; for this a Plan view is used, there is no separate `Which::Paper`.
- The category is called **Generic Annotation** everywhere: New Family, Project Browser, document
  title, Properties.
- **Scale with View** — in Family Category and Parameters / Options, with a hint about 1:100.
- Annotate:
  - **Line** — clicks on points; Enter or a double click finishes.
  - **Arc** — three clicks: start, end, a point on the arc; the centre and radius are computed. An
    erroneous third click on the line of the first two is rejected with a message.
  - **Text** — a click places text; the content, height, type and alignment are edited in
    Properties.
  - **Label** — a click places a label; by default it reads the family's first parameter.
  - **Dot** — a click places a dot/circle (default radius 1 mm).
  - **Filled Region** — clicks along the contour; the first corner again or Enter closes it.
  - **Masking Region** — the same with the `Masking` type.
- The active tool is highlighted on the ribbon; the first click after pressing the button is not
  lost.
- Properties of the selected symbol: fill type, line style, text type, `valign`, `wrap`, label
  content, coordinates, radius/height.
- Selection and nodes:
  - a symbol is selected by a click and dragged as a whole;
  - a line shows all of its points;
  - a Filled and Masking Region shows all corners of each contour;
  - an arc shows its start, middle and end; the end changes the arc, the middle changes the bulge,
    and the arc is rebuilt through the two remaining points.
- A label with a non-existent `{Parameter}` does not remove the view: a `{Parameter}` placeholder is
  drawn, and in Properties under the Text field there is a red message listing the existing
  parameters.
- Family snaps are configured separately: **Manage > Snaps** — object snaps, angles, Alignment
  Guides, tolerance and snapping. Additionally **Snap to coordinates** and **Step** (by default
  **1 mm**, the toggle is off). Without a found snap the point follows the cursor.
  **Guides from grids** applies only to structural grid lines, not to the coordinate grid.
- Undo: one drawing/editing operation — one step.

## 5. Placing and editing instances

- `annotation` places an instance on a sheet or a flat view: exactly one of `sheet`/`view`, `at`,
  `angle`, `mirrored`, `params`. `view` accepts a level plan, plan, ceiling, section, elevation;
  `{3D}` and schedules are rejected.
- `params` are validated against the family's parameters; an unknown name is a rejection with a
  list of parameters.
- `modify` changes `at`, `angle`, `mirrored`, `type`, `params`; `null` restores the type's value.
- `move`, `rotate`, `mirror`, `delete` and copy/paste support `Element::Annotation`: `transform`
  handles `at`/`angle`/`mirrored`, the clipboard assigns the annotation to `Sort::Annotation`, and
  deleting a sheet/view deletes its annotations.
- Placement from the Project Browser: the family is under **Families > Generic Annotation**, the
  types are leaves; selecting a type enables `Tool::Component`, and a click on a plan or a sheet
  places the annotation. In `{3D}` — a rejection with a hint to open a sheet or a flat view.
- After a family is loaded into the project it is immediately selected and expanded in the Project
  Browser: the Families tab, the tree open down to the family, the row highlighted and scrolled
  into view. The same — when loading into a host family.
- `elements`/`query_elements` return the category `annotation`, `view`, `sheet` or `view_name`,
  `family`, `type`, `at`, `angle`, `mirrored`, `params`, `size`, `size_model`,
  `scale_with_view`.

## 6. Relationship with PR #598

- Move, Copy, Rotate, Mirror, Scale, Offset, Trim, Split, Measure, Create Similar and the clipboard
  come from #598; there are no copies of them in the branch.
- Annotation symbols have been added to the #598 selection: `select_box::Scene::pieces` reads
  `annotation::marks_indexed`, so lines, arcs, dots, labels and **fills** take part in clicking,
  window selection, highlighting and Move/Copy.
- Move/Copy of a symbol moves the line, arc, text and dot; the branch adds moving of all points of
  all fill rings, so that Filled/Masking Region move and copy together with the symbol.
- The Rotate/Mirror/Scale behaviour for symbols remains exactly as #598 defines it; there are no
  separate annotation variants.

## 7. Differences from the original ТЗ-030

1. There are no separate Rectangle and Circle buttons: a rectangle is a closed Line, a circle is an
   arc from 0–360°.
2. Arc is built with three clicks (start, end, a point on the arc), not "start — end — angle".
3. Text and Dot are click tools, not creation at the zero point.
4. Dot is a circle, not a square; the default radius is 1 mm.
5. An erroneous `{Parameter}` produces a placeholder and a red message instead of removing the view.
6. Family snaps are configured separately, with `Snap to coordinates` and `Step`; the default step
   is 1 mm.
7. After loading, the Project Browser expands and scrolls to the family automatically.
8. The default frame is 100 × 50 mm; the family content is in paper millimetres.
9. Modify/Measure come from #598; annotation symbols are integrated into its selection and fill
   movement.
10. Offset, Trim/Extend and Split Element for annotation symbols follow #598 behaviour; there are no
    separate annotation implementations.

## 8. Code map

- Model: `crates/core/src/family.rs` (`Category::Annotation`, `scale_with_view`, `Symbol`,
  `Label`), `crates/core/src/model.rs` (`Annotation`, `Element::Annotation`,
  `annotation_plan`).
- Content/rendering: `crates/core/src/family/annotation.rs`, `crates/core/src/render.rs`,
  `crates/core/src/sheet.rs`, `crates/core/src/graphics.rs`.
- Commands/file: `crates/core/src/commands.rs` (`family`, `annotation`, `modify`, `elements`),
  `crates/core/src/clipboard.rs`.
- Editor: `crates/ui-egui/src/editor/views.rs` (`SymbolKind`, `SymbolDraw`, `SymbolNode`,
  `symbol_in_view`, `paint_symbol`), `crates/ui-egui/src/editor.rs` (`SymbolText`, `Part`,
  `symbol_properties`), `crates/ui-egui/src/editor/document.rs` (Annotate, Manage > Snaps),
  `crates/ui-egui/src/editor/options.rs` (`Scale with View`).
- Selection/move: `crates/ui-egui/src/editor/select_box.rs`, `.../drag_move.rs`.
- Placement: `crates/ui-egui/src/browser.rs` (`Act::Place`, Generic Annotation),
  `crates/ui-egui/src/canvas.rs`, `crates/ui-egui/src/paper.rs`, `crates/ui-egui/src/orbit.rs`
  (rejection in 3D), `crates/ui-egui/src/properties.rs`.
- Documentation: `docs/areas/annotation-families.md` (English).

## 9. Checks

- `cargo fmt --all` — clean.
- `cargo check --workspace` and `cargo check --workspace --all-targets` — clean.
- `cargo clippy --workspace -- -D warnings` — clean.
- `cargo i18n check` — clean after `cargo i18n update`.
- Tests were not run locally (project rule). CI for PR #600 runs Linux, Windows, macOS,
  IFC; at the time this document was written the run was still in progress. The first run revealed
  a test literal `Label` without the new fields — fixed.

## 10. Checklist of what is implemented

- [x] `Category::Annotation`, `Family::is_annotation`, `Family::space`,
      `Family::scale_with_view`; `family` creates and reads an annotation family.
- [x] `Symbol::fill`, `Symbol::style`, `Label::text_type/valign/wrap`.
- [x] `Element::Annotation`, `annotation_plan`, `annotation`, `modify`, move/rotate/mirror/
      delete, copy/paste, `elements`/`query_elements`.
- [x] `family/annotation.rs`: `PaperMark`, `marks`, `marks_indexed`, `bounds`, `check`.
- [x] Rendering of sheets, plans, ceilings, sections and elevations; `render {family}`.
- [x] `scale_with_view` in the UI and in rendering; table of modes.
- [x] Editor: paper frame, Annotate tools, highlighting, first click.
- [x] Line, Arc (3 clicks), Text, Label, Dot, Filled Region, Masking Region.
- [x] Nodes of the line, both region kinds and the arc; dragging the symbol.
- [x] `{Parameter}` error: placeholder and a list of parameters.
- [x] Family snaps, `Snap to coordinates` + `Step` 1 mm.
- [x] Dashed fill outline in the editor only.
- [x] Placement from the Project Browser on a plan and a sheet; rejection in `{3D}`.
- [x] Expanding and highlighting the family in the Project Browser after loading.
- [x] Integration with the #598 selection; movement of fill rings.
- [x] `docs/areas/annotation-families.md` in English.
