# SPEC-013. Grid line types: `GridType`, editable line pattern and text type

Status: **planned** (not started)
Date: 2026-11-13
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: [IPedrax/BIMCraft#373 "Request for Editable Axis Lines (Standardization Errors)"](https://github.com/IPedrax/BIMCraft/issues/373) — the description by Tri4c3 dated 2026-11-10 (a grid axis must be a dash-dot line per ISO 128-23, not a dashed one; the axis has no Edit Type — the line type and text type cannot be changed; ISOCPEUR/DIN fonts) and the analysis by harijsvucens of the same day (`render::grid` with a hardcoded `[14, 4]`, `Grid` — only a name and two points, therefore there is nothing for Edit Type to open), plus [PR #379](https://github.com/IPedrax/BIMCraft/pull/379) — the first item, the dash, already merged into `main`.
Related documents: `README.md` (command table, Annotation and Datums sections), `BIMCraft/docs/areas/views-and-graphics.md`, `BIMCraft/docs/areas/ifc-exchange.md`, `ТЗ-010-правка-размерной-цепочки.md` (a template for types, the Edit Type dialog and migration), `ТЗ-012-импорт-клавиш-revit.md`.
Task type: new functionality (element type, library record, command, dialog, format migration), replacing hardcoded drawing with type data, tests.

> **In short.** PR #379 fixed only the appearance: `render::grid` (`crates/core/src/render.rs:5108`) draws a grid axis as a dash-dot line per ISO 128-23, but the pattern `[14, 3, 2, 3]` is still hardcoded in pixels, the colour and pen are constants (`GRID`, `Pen::PROJECTION`), the label is our single-stroke lettering at 10 px, the bubble is the constant `BUBBLE = 11`. `Grid` (`model.rs:260`) has neither a type nor a reference to one: `{name, start, end}`. That is why the Type Selector is empty, Edit Type is unavailable (`Model::type_names`/`default_type` do not know `"grid"`, `properties.rs:949` returns `None`), and there is no way to change the line pattern or the label font. The task: introduce `GridType` (line pattern + pen + colour + label text parameters) and let the user **choose the grid line type themselves**: from the starting presets of European and American standards (ISO 128 / EN ISO 7519, DIN 1356, ASME Y14.2), from any project pattern, or from their own type. Carry it through the model, the library, commands, rendering, IFC/Insert and the Type Properties dialog, the way it is already done for `DimensionType` (`dims.rs:85`, `dimtypes.rs`).

## 0. General working rules

1. Communication language — Russian; code, comments, lines in code, console messages — English. Documentation in the repository (`docs/`, `README.md`, `CLAUDE.md`) — English, this file — Russian.
2. Tests (`cargo test` and any other runs of test code) are **not run** for this task (rule `~/.config/opencode/rust-rules.md`): verification is `cargo check`, `cargo build`, `cargo clippy` (without `--all-targets`) and CI in the PR.
3. Commits are atomic: one commit — one logically complete change; every commit builds and does not break existing behaviour.
4. Identifiers (field, command, type and file names) are not translated. New user-facing strings go through `msg!`/`ui_msg!`/`t()`, the `translations/` catalogues are updated (`cargo i18n update`).
5. The `Model::check` path is the single place of validation; new rules go there too, with a message saying what to fix.
6. Do not copy Autodesk data: neither fonts, nor templates, nor type names from `.rvt`/`.rfa`. Everything in the task is ours or from public standards (ISO 128-23 is a description of the dash, not a file).

## 1. The essence of the problem

### 1.1. What exists now (by code)

Rendering (`crates/core/src/render.rs`):

- `fn grid(d, start, end, name)` — `render.rs:5108`. The pattern is hardcoded: `d.stroke(vec![a, b], Pen::PROJECTION, GRID, Some(vec![14.0, 3.0, 2.0, 3.0]), false)` (`render.rs:5112`; after PR #379). A comment right there explains that the 2 px dot works around a 1 px loss in the egui retained canvas (`retained.rs`).
- Colour `const GRID: Rgb = Rgb(60, 100, 150)` (`render.rs:51`), bubble radius `const BUBBLE: f32 = 11.0` (`render.rs:58`), label `d.text(name, c, 10.0, GRID)` (`render.rs:5123`), that is our `font::` lettering at 10 px height, not project text.
- The pattern is specified in **screen pixels** and scales neither with zoom nor with the view scale: the line is drawn without `d.grow`, whereas the bubble grows (`d.grow(from, (x, y))`).
- Two call sites: plan (`render.rs:3320`, clipped to the crop) and section/elevation (`render.rs:2498`, `along`, `lo - GRID_PAST`, `hi + GRID_PAST`). Both pass only `&g.name` — the function has neither the model nor the element.

Element (`crates/core/src/model.rs`):

- `pub struct Grid { name, start, end }` (`model.rs:260`), `#[serde(deny_unknown_fields)]`. Category — `"grid"` (`model.rs:2503`), `Element::Grid(Grid)` (`model.rs:2248`).
- `Model::check` for a grid axis validates only the name (non-empty, unique) and the line (`model.rs:7140`).
- `Element::Grid` does not participate in `lib_refs_mut` — it has no library reference (`library.rs:1470–1493`).

Types in the project:

- `Model::type_names(category)` (`model.rs:4505`) and `Model::default_type(category)` (`model.rs:4530`) know `region`, `text`, `dimension`, wall/floor/door/… — **not `grid`**. Therefore `properties.rs::type_subject` (`properties.rs:949`) returns `None` for a grid axis → the Edit Type button is unavailable, and the Type Selector (`properties.rs:1416`) is empty.
- The templates to follow: `DimensionType` (`dims.rs:85`) with the text fields `font`, `text_size`, `bold`, `italic`, `underline`, `width_factor`, `color`, `line_pen`; the `Dimension.type_name` reference with `#[serde(rename = "type")]` (`model.rs:1151`); the Dimension Type Properties dialog (`ui-egui/src/dimtypes.rs`); `TextType` (`text.rs:176`) and the Text Type Properties dialog (`ui-egui/src/notes.rs:462`).

What is already in place and reused:

- `lines.rs`: `Segment` (`Dash`, `Space`, `Dot`, `DotLength`, `Offset`, `Wave`, `Zigzag`, `Text` — lengths in mm on paper), `LinePattern` (`lines.rs:41`) with `check` validation and conversion to pixels `dash(px_per_mm)` (`lines.rs:115`), `LineStyle` (`lines.rs:148`), starting patterns `patterns(language)` (`lines.rs:174`). This is exactly the mechanism the grid axis is missing.
- `graphics.rs`: `"grid"` is in the annotation list (`graphics.rs:105`), `category_of` (`graphics.rs:184`), `Drawing::dress` (`graphics.rs:1279`) applies view overrides after drawing — colour, weight and `Dash` (`Solid | Dashed | Dotted`, `graphics.rs:285`).

**Can a grid axis be assigned a line type today? No.** The closest thing is a category override in Visibility/Graphics: `graphics {view, categories: {grid: {projection: {dash: "solid" | "dashed" | "dotted", color, weight}}}}`. It works (in `Look::with`, `graphics.rs:1138–1219`, the `grid` category is spread into `paint` per axis; there are no tests for it), but it is not a line type:

- **for the whole view and all axes at once** — a pattern cannot be chosen for a single axis;
- **three coarse dashes** baked into pixels: `Dash::pattern` (`graphics.rs:293`) yields `[10, 5]` and `[2, 4]`, without dots, without mm, without scaling;
- **it does not touch the label and the bubble** (`Part::Text` is skipped, `graphics.rs:1298`);
- **it is not stored as a type**: it can be neither reused, nor carried to another axis or another view, nor seen in Properties;
- **it does not survive** a view change: the setting belongs to one view, while the axis "lives" in all of them.

Project patterns (`line_patterns`, `lines.rs:41`) are library records, but only detail lines refer to them (`Element::DetailLine.style`, `lib_refs_mut` `library.rs:1479`) and the boundaries of filled regions (`FilledRegion.line_style`, `library.rs:1484`); a grid axis does not reference a pattern. Neither `grid` (parameters `start`, `end`, `name?` — `commands.rs:213`) nor `modify` accept a `type` or a pattern; the palette has no type row for a grid axis (`properties.rs:228`).
- `text.rs`: `TextType` (`text.rs:176`), `text::layout(&t, text, styles)` (`text.rs:589`), `render::lettering` (`render.rs:4929`), `NoteFrame` (`render.rs:4873`) — the label can be drawn with the project font, including a system one.
- IFC: `grids()` (`ifc.rs:923`) exports the geometry of the axes (`IfcGrid`/`IfcGridAxis`/`IfcAnnotation`), it writes no styles.

### 1.2. Scenarios and expected behaviour

1. **Edit Type on a grid axis opens Grid Type Properties** — there the line type (pattern), weight, colour, font, size, style and width of the label in the bubble are changed; OK/Apply/Duplicate/Rename work as they do for dimensions.
2. **The Type Selector on a grid axis is not empty**: it lists the project's grid types, the selected axis shows its type; choosing one switches the type of the selected axes and is remembered for the Grid tool.
3. **The line type is chosen by the user themselves** — from the starting presets by standard (ISO 128 / EN ISO 7519, DIN 1356, ASME Y14.2, office variants — table in F2), from the project patterns (Dash, Long Dash, Dot, Dash Dot, Hidden, Centre… and the ones they created) or from their own type; the dash, gap and dot are drawn in mm on paper and scale with the view and zoom instead of "freezing" in pixels. A preset is an ordinary type: it can be changed, duplicated and put on any axis, not "one line switched on for all".
4. **A new label font** — any font available in the system (including ISOCPEUR or DIN, if the user installed them), plus IBM Plex and our single-stroke lettering; the size is in mm on paper.
5. **Plan and section draw the axis by its type in the same way** (both `render` branches go through a single `grid()`).
6. **View overrides are stronger than the type**: `graphics {view, categories: {grid: {projection: {dash: "dotted", color: …}}}}` still changes the line as it does today — the type provides the "default".
7. **Old projects open without loss**: a format 19 file is read, each grid axis gets the default type; a saved project looks like the previous style (dash, colour, bubble) up to small scaling differences (§10, item 1).
8. **Commands and MCP**: `grid {start, end, name?, type?}`, `grid_type {name?, like?, …}`; `modify {ids, type}` switches the type; `inspect`/`query_elements` show `type` on a grid axis.
9. **Insert from File** brings the grid type together with the axis if the project does not have it; **IFC** keeps exporting the axis geometry unchanged.
10. **Fonts are not bundled**: ISOCPEUR/DIN is a matter of licensing and installing in the OS; the project stays clean-room. In this spec that is the answer to the third item of the issue, it requires no code.

### 1.3. Why this matters

- The grid axis is the only annotation kind that today has **no type at all**: both the line and the label are hardcoded. Because of that, standardising a drawing (ISO/EN, DIN, ASME, an office standard) is impossible without editing the sources.
- A pattern in pixels is not just "it cannot be changed": when the view scale changes and when printing, the dash does not match the paper, whereas all other lines (`LinePattern`, mm on paper) do. That is a silent drawing error, not cosmetics.
- A label in our lettering does not allow setting the font required by a company standard — the reporter says this directly.
- The type mechanism already exists and is well-tested (dimensions, text, regions, line styles): the work is to wire up what exists, not to build it anew.

## 2. Terms

| Term | Meaning |
|---|---|
| `GridType` | A new grid axis type: line pattern, pen, colour and label parameters. The counterpart of `DimensionType` for axes |
| `Grid` | The grid axis element: name, start/end and (new) the type name |
| `LinePattern` | A line pattern (`lines.rs`): `Dash`/`Space`/`Dot`/…, lengths in mm on paper |
| `Segment::Dot` | A dot in a pattern; drawn as a segment of ~0.01 mm, so on screen it needs a lower bound in pixels |
| `LineStyle` | Pen + colour + pattern (`lines.rs:148`); the grid axis does **not** use it, see §10, decision 2 |
| `TextType` | A text type (`text.rs:176`): font, size, style, width factor, colour, frame |
| Type Selector | The type drop-down list in the Properties palette (`properties.rs:1416`) |
| Edit Type | The button in the palette (`properties.rs:1092`) → the Type Properties dialog |
| Type Properties | The type dialog (`typeprops::head/table`, the template is `dimtypes.rs`) |
| Library record | `library.rs::Entry` with an id (counting down from 999999) and a GUID; needed for Insert from File, renaming and `library` |
| V/G | Visibility/Graphics of a view; `graphics.rs::dress` is applied **after** drawing |
| `paper` | The annotation scale of a view: `px_per_mm = sheet::DENSITY * d.paper`, `DENSITY = 5.0` px per mm of paper |
| Annotation scale | `Drawing::k`/`d.grow`: 1 layout px becomes `k` px on screen |
| Migration | `Model::migrate` (`model.rs:3822`) + `FORMAT` (`model.rs:175`, currently 19) |

## 3. Functional requirements

### 3.1. F1. `GridType`: structure and validation

- A new module `crates/core/src/grids.rs` (following `dims.rs`): `pub struct GridType` with fields mirroring the `DimensionType` vocabulary, so that the dialog and the JSON feel familiar:

| Field | Type | Default | Meaning |
|---|---|---|---|
| `line_pattern` | `Option<String>` | for the starting type — the name of the ISO pattern (see F2); `None` — solid | The name of a `LinePattern` in the project |
| `line_pen` | `Pen` | `Pen::PROJECTION` (=4) | Line weight in annotation weights |
| `color` | `[u8; 3]` | `[60, 100, 150]` | Colour of the line and the label (as `render::GRID`) |
| `font` | `String` | `text::ARIAL` | Label font (system, IBM Plex or our lettering) |
| `text_size` | `f64` | `2.5` | Type size, mm on paper |
| `bold`, `italic`, `underline` | `bool` | `false` | Style |
| `width_factor` | `f64` | `1.0` | Letter width, as in `TextType` |

- `#[serde(deny_unknown_fields)]`, all fields with `#[serde(default …)]`; `GridType::default()` — without `line_pattern` (solid) and with the Arial font, so that the default value does not depend on the project language.
- `GridType::check(&self, name, line_patterns: &BTreeMap<String, LinePattern>) -> Result<(), String>`:
  - `line_pattern`, if set, must exist (the error lists the available ones, like `LineStyle::check`, `lines.rs:158`);
  - `line_pen.is_valid()`;
  - `font` — non-empty, without control characters, no longer than the `TextType::check` limit (`text.rs:237` and further; the limits are taken from there instead of being duplicated as numbers);
  - `text_size` and `width_factor` — the same bounds as in `TextType::check`;
  - the message starts with `grid type \`{name}\`:`.
- The constructor `GridType::text_type(&self) -> TextType` assembles a `TextType` from the fields (font, size, bold, italic, underline, width_factor, color), so that it can be handed to `text::layout` — without a second source of truth.
- Do not add new public functions duplicating `text.rs`: font checks — `text::known`/`text::families`, as in `dimension_type`.

### 3.2. F2. The user chooses the grid line type: a starting set by standard

The goal is not "change the axis line in general" but to let the user **choose the grid line type**: both from a ready-made set by standard and freely (any project pattern, any pen, colour and font), and by creating their own types. The presets below are starting data, not a closed list.

- In `lines::patterns(language)` (`lines.rs:174`) add patterns for the presets (mm on paper, names via `msg!`/`language.word`):

| Pattern | Segments |
|---|---|
| `Grid - ISO Dash Dot` | `[Dash(12.7), Space(3.2), Dot, Space(3.2)]` |
| `Grid - DIN Dash Dot` | `[Dash(20.0), Space(4.0), Dot, Space(4.0)]` |
| `Grid - ASME Chain` | `[Dash(20.0), Space(3.0), Dash(5.0), Space(3.0)]` |
| `Grid - Architectural Chain` | `[Dash(15.0), Space(2.0), Dash(2.0), Space(2.0), Dash(2.0), Space(2.0)]` |
| `Grid - Compact Dash Dot` | `[Dash(6.0), Space(1.5), Dot, Space(1.5)]` |

- In `grids.rs`: `pub fn types(language: Language) -> BTreeMap<String, GridType>` — the starting grid axis types:

| Type | Pattern | Pen (mm of paper) | Text | Source |
|---|---|---|---|---|
| `Grid - ISO 128` | `Grid - ISO Dash Dot` | 4 (0.18) | 2.5 mm | ISO 128-2, type 04 "chain"; ISO 128-23 (axes on construction drawings); ISO 7519 / EN ISO 7519 (grid and circles); BS 8888 follows ISO 128 |
| `Grid - DIN 1356` | `Grid - DIN Dash Dot` | 7 (0.25) | 2.5 mm | DIN 1356-1; DIN 15-1 / ISO 128 (Strichpunktlinie) |
| `Grid - ASME Y14.2` | `Grid - ASME Chain` | 9 (0.30) | 3.5 mm | ASME Y14.2 (center line: long — short dash) |
| `Grid - Architectural` | `Grid - Architectural Chain` | 4 (0.18) | 2.5 mm | office practice, **not** a standard |
| `Grid - Continuous` | solid | 4 (0.18) | 2.5 mm | coordination and zone grids (NCS practice) |
| `Grid - Compact` | `Grid - Compact Dash Dot` | 4 (0.18) | 2.0 mm | Revit/AutoCAD templates for dense plans, **not** a standard |

- `default_type(types) -> Option<&String>` — `Grid - ISO 128`, otherwise the first alphabetically (like `text::default_type`, `text.rs:276`). The colour of all is `[60, 100, 150]` (the former `render::GRID`), the font is Arial.
- These are ordinary `grid_types` records: `grid_type {name, rename, delete}` and Edit Type change them like any types; they can be duplicated; one in use cannot be deleted. The list is not closed — the user adds their own type on any project `LinePattern` (or solid) with any pen, colour, font and size (§3.1, §3.8).
- Caveats about the standards (important for honesty):
  - dash lengths in ISO are given in multiples of the line thickness (and depend on the line group), in DIN — on the group and the sheet format, in ASME — the long and the short dash are of fixed length; here one variant of each is pinned, automatic recalculation by thickness and conformance checking are out of scope (§9);
  - pen ↔ mm of paper — from `pen.rs` (`WIDTHS`): 4 → 0.18, 7 → 0.25, 9 → 0.30;
  - bubble: ISO 7519 — Ø 8–10 mm; in ASME the size is per the company standard; formula F4 with a 2.5–3.5 mm text gives Ø ≈ 5.5–7.4 mm, the exact size requires a "Bubble Size" parameter (§10, question 3);
  - `italic` gives a slant of ≈ 11.3° (`SLANT = 0.2`, `text.rs:37`); ISO 3098 and ASME Y14.2 use upright letters, therefore the presets are not italic; a slant per a company standard is a field in the type, out of scope;
  - the length values are our reading of the standards from public descriptions; before release verify them against the text of the standard (the task does not assume buying or copying standards).
- All patterns fit within the `Segment` limits (`MIN_SEGMENT = 0.05`, `MAX_SEGMENT = 100`, `MAX_SEGMENTS = 32`, `lines.rs:51–55`).
- `Model::new`/`Model::resolve` receives `grid_types: crate::grids::types(language)` next to `text_types`/`dimension_types` (`model.rs:3242`).
- `Model::migrate`, when reading a file where `grid_types` is empty, fills it with the starting set (an old file has no such field — the serde default fires, but an empty map in a file must not leave projects without types).

### 3.3. F3. `Grid` gets a type

- `Grid` (`model.rs:260`) gets `#[serde(rename = "type", default)] pub type_name: String`.
- `Model::check` for a grid axis (`model.rs:7140`): if `type_name` is empty — an error "the axis has no type" (the command and the migration always set it), if the type is not in `grid_types` — an error with the list of available ones, as for dimensions (`model.rs:7384`).
- `Model::type_names` (`model.rs:4505`) and `Model::default_type` (`model.rs:4530`) get the branch `"grid" => names(&self.grid_types)` / `crate::grids::default_type(&self.grid_types)`.
  This is the single switch that turns on both the Type Selector and Edit Type; without it the task does not work, even if everything else is done.
- `next_grid_name` (`commands.rs:2974`) does not change: the axis name is not tied to the type.
- `modify {ids, type}` (`commands.rs:14441`) works automatically (it edits the element JSON), but only after the F3 validation; the error must name the axis and the missing type.

### 3.4. F4. Drawing by type

- Signature: `fn grid(d: &mut Drawing, m: &Model, g: &Grid)` (instead of `start, end, name`); both call sites (`render.rs:2498`, `render.rs:3320`) pass `m` and `&g`. The type is taken as `m.grid_types.get(&g.type_name)`, in its absence — the project's default type, otherwise `GridType::default()` (only a guard against a corrupt file: `check` would not let it through).
- Line: `d.stroke(pts, t.line_pen, Rgb(t.color[0], t.color[1], t.color[2]), pattern_dash, false)`, where `pattern_dash` is `m.line_patterns.get(t.line_pattern).and_then(|p| p.dash(px_per_mm))`, `px_per_mm = sheet::DENSITY as f32 * d.paper` — exactly as for detail lines (`Drawing::styled`, `render.rs:320`). Solid — `None`.
- **Dot**: `LinePattern::dash` converts `Segment::Dot` into 0.01 mm → a fraction of a pixel, and the dot disappears (even more so in the retained canvas, `retained.rs`). On the axis path, pattern lengths after conversion to px are raised from below: every segment — at least 2 px (for a dash and a gap this is still scalable, for a dot it is the same workaround as in today's comment `render.rs:5110`). The rule is formalised as a single function in `render.rs` (for example `grid_dash(pattern, px_per_mm)`) and covered by a test.
- Label: `text::layout(&t.text_type(), &g.name, &[])`, then `lettering(d, &layout, frame, Rgb(t.color[0], t.color[1], t.color[2]))` with `frame = NoteFrame { at: px(centre of the bubble), k: sheet::DENSITY as f32 * d.paper, angle: 0 }`, shifted by `(-width/2, -height/2)` — the way `note()` does it (`render.rs:4929`). Part of the primitives — `Part::Text` (V/G does not change the weight and dash of the label, but does change the colour — as now).
- Bubble: the radius in mm of paper `r = max(2.2, 0.6·text_size + max(layout.width, layout.height)/2)` (for 2.5 mm and the mark "1" ≈ 2.75 mm; today 11 px at `DENSITY = 5` — 2.2 mm), the centre — at `r` beyond the end of the line along it; the ring and the `PAPER` fill — as now, the pen/colour — from the type. Everything drawn in pixels is still wrapped in `d.grow(from, about)` so that the bubble scales with the view.
- Both ends of the axis, the crop and "bubbles beyond the crop" (`render.rs:3317`) — with no change in behaviour.
- The colour and weight of the line after drawing may be overridden by V/G (`dress`, `graphics.rs:1279`) — that is the required priority; `grid()` itself contains no view checks.

### 3.5. F5. Commands

- `grid` (`commands.rs:213`, `fn grid` `commands.rs:7932`) gets `type?`:
  `edit("grid", ui_msg!("Grid"), &["GR"], "{start: [x,y], end: [x,y], name?, type?} …", grid)`; without `type` — `Model::default_type("grid")`; an unknown type — an error with the list (through `default_type`/`grid_types`).
- A new command `grid_type` — following `dimension_type` (`commands.rs:10051`) and the `ALL` record next to `"dimension_type"` (`commands.rs:977`):
  - `grid_type` without `name` — a listing: `{"types": {…}, "fonts": text::families()}`;
  - `grid_type {name, like?}` — create/modify: the values are read the way a file reads a type (`deny_unknown_fields`), `font` is validated by `text::known`, `color` — `[r,g,b]` or `#rrggbb` (the `colour` helper, as in `dimension_type`);
  - `rename?` — rename (a taken name — an error) through `rename_entry(m, &LibKind::GridType, …)` (`commands.rs:17171`);
  - `delete: true` — delete if no grid axis uses the type (otherwise the error names the axis), as in `dimension_type`;
  - the answer — `{"name", "type"}`; all changes — one command, one undo step.
- `inspect`/`query_elements`/`elements_json` show `type` on a grid axis automatically (a serde field); do not write a separate branch, but verify it with a test.

### 3.6. F6. Library, Insert from File, renaming

- `library.rs`: `Kind::GridType` — in `enum Kind` (~28), in `label()` ("grid type"), in `parse` (`"grid-type"`), in `Display` (the same `"grid-type"`); `library_names()` — `add(Kind::GridType, &mut self.grid_types.keys())` next to `DimensionType` (`library.rs:821`).
- `lib_refs_mut`: `Element::Grid(g) => f(&Kind::GridType, &mut g.type_name)` next to dimensions (`library.rs:1492`).
- `insert.rs`: `Kind::GridType => one(&mut to.grid_types, Some(&from.grid_types), name)` in `copy_definition` (`insert.rs:221`, next to `Kind::DimensionType` ~253) and accounting in `renames`/`lib_refs` — otherwise a grid axis inserted from another project gets a type that the project does not have, and `check` rejects it.
- Purge: `PURGE_KINDS` (`library.rs:631`) and `remove_entry` (`library.rs:645`) do **not** touch grid axis types — just like dimension/text types; this must be left as is and confirmed by a test that `purge` does not carry away grid axis types.
- IFC: export (`ifc.rs:923`) does not change — we do not write axis styles to IFC (§9). Import (`import.rs`) does not create axes — not touched.

### 3.7. F7. File format and migration

- `FORMAT` 19 → 20 (`model.rs:175`).
- `Model::migrate`: a new `fn typed_grids(&mut self)` next to `typed_dimensions` (`model.rs:3932`): every grid axis with an empty `type_name` gets the default type; if `grid_types` is empty — the starting set. A comment line about the 19 → 20 transition is mandatory (the style of the neighbouring steps).
- A format 20 project opened by a build 19 must be **rejected** as "saved by a newer version" — this is what justifies the `FORMAT` bump: a `Grid` with `deny_unknown_fields` would otherwise produce "unknown field `type`".
- A format 19 file opens: `lenient.rs` drops the unknown part, `typed_grids` sets the type, `render` draws as before (adjusted for pattern scaling).

### 3.8. F8. UI: Type Selector, Edit Type, Grid Type Properties

- `type_subject`/`edit_type` (`properties.rs:949`, `1020`): the branch `if category == "grid" { self.grid_type_dialog = Some(crate::gridtypes::GridTypeDialog::new(&self.session.model, name, elements)); return; }` — next to `"text"`/`"dimension"`.
- A new file `crates/ui-egui/src/gridtypes.rs` following `dimtypes.rs`:
  - `GridTypeDialog { name, t: GridType, elements: Vec<Id>, naming: Naming, error, find }`;
  - `params(name)` — the JSON for the `grid_type` command;
  - `Modal::new(...).width(typeprops::WIDTH)`, `typeprops::head(ui, "grid-type", &tf("System Family: {}", &[t("Grid")]), …)`, `typeprops::table` with the groups:
    - **Graphics**: Line Pattern (the drop-down list of project patterns + `Solid`), Line Weight (`DragValue` 0..=MAX_PEN), Color (as in the text type: `tb.color`);
    - **Text**: Text Font (the `text::families()` list with the `find` filter, as in `notes.rs:516–530`), Text Size (`DragValue` 0.5..=50 mm, although `check` allows up to 100 — like `TextType`), Bold/Italic/Underline, Width Factor;
  - OK/Cancel/Apply/Duplicate/Rename — the same parsing of `Head`/`Naming` as in `notes.rs:555–591`; changes go only through `self.exec("grid_type", …)`, never by editing the model;
  - duplicate and rename must not touch the axes until OK is pressed (as in `TextTypeDialog`).
- `BimApp`: the field `grid_type_dialog: Option<gridtypes::GridTypeDialog>` next to `text_type_dialog` (`lib.rs:707`), initialisation (`lib.rs:1468`), participation in the list of open modals blocking input (`lib.rs:2145`), calling the window where `text_type_window`/`dimension_type_window` are called (`lib.rs:5663` looks for `type_properties`; add `self.grid_type_window(ctx)` next to it).
- Palette: supplement the rows for a grid axis (`properties.rs:228`) with `("Other", "type", msg!("Type"), Shown)` or show the type only through the Type Selector — decision §10, question 4; on top, the Type Selector will show the types itself.
- `browser.rs`: do **not** add grid axes to `SYSTEM_FAMILIES` (`browser.rs:65`) — like text and dimensions, this is an annotation, not a family; confirm that the Type Selector works nevertheless (F3).
- The Grid tool: the type chosen in the Type Selector is remembered in `self.types["grid"]` and passed to `grid {type}` — as for the other tools (`type_selector`, `properties.rs:1416`).

### 3.9. F9. Change map

| What | File | Where/what now |
|---|---|---|
| `GridType`, `types()`, `default_type()`, `check` | `crates/core/src/grids.rs` (new) | template: `dims.rs:85` |
| Starting axis pattern | `crates/core/src/lines.rs` | `patterns()` ~174 |
| The `Model::grid_types` field, the seed in `resolve` | `crates/core/src/model.rs` | next to `text_types`/`dimension_types` 3116–3119, 3243–3244 |
| `Grid.type_name` | `crates/core/src/model.rs` | `Grid` 260 |
| `type_names`/`default_type` for `"grid"` | `crates/core/src/model.rs` | 4505, 4530 |
| Type validation on a grid axis | `crates/core/src/model.rs` | `Model::check`, `Element::Grid` 7140; template 7384 |
| Validation of the types themselves | `crates/core/src/model.rs` | the loop over types 6660–6680 |
| `FORMAT` 20, `typed_grids()` | `crates/core/src/model.rs` | 175, 3822, template `typed_dimensions` 3932 |
| `Kind::GridType` (label/parse/Display) | `crates/core/src/library.rs` | 28–95, 103–127, 149–162 |
| `library_names`, `lib_refs_mut` | `crates/core/src/library.rs` | 821, 1492 |
| Insert from File | `crates/core/src/insert.rs` | `copy_definition` 221 |
| `grid` + `type`, `grid_type`, `ALL` | `crates/core/src/commands.rs` | 213 / 7932; 977 / 10051 |
| Drawing by type | `crates/core/src/render.rs` | `grid()` 5108, calls 2498 and 3320, `BUBBLE` 58, `GRID` 51, `styled` 320 (the px_per_mm template) |
| The Grid Type Properties dialog | `crates/ui-egui/src/gridtypes.rs` (new) | template `dimtypes.rs` |
| The dialog field and call | `crates/ui-egui/src/lib.rs` | 707, 1468, 2145, 5663 |
| Edit Type for `"grid"` | `crates/ui-egui/src/properties.rs` | `type_subject` 949, `edit_type` 1020 |
| Palette rows (if we decide to show the type) | `crates/ui-egui/src/properties.rs` | 228 |
| Type Selector (no changes, but verify) | `crates/ui-egui/src/properties.rs` | 1416 |
| README, docs, translations | `README.md`, `docs/areas/*`, `translations/*` | 227, 248, 601; `cargo i18n update` |

## 4. Pitfalls

1. **Pixels versus millimetres.** Today the axis pattern is in screen pixels and does not change with zoom or a view scale change. Moving to `LinePattern` (mm on paper) is right, but visually noticeable: the dash becomes longer, and on a sheet and in a PNG it is counted from paper. Convert only through `px_per_mm = sheet::DENSITY * d.paper` (as `Drawing::styled`, `render.rs:320`), not through `d.k` and not through `px`.
2. **A dot of 0.01 mm disappears.** `LinePattern::dash` (`lines.rs:115`) converts `Segment::Dot` into 0.01 mm; at `DENSITY = 5` that is 0.05 px. In the egui retained canvas single-pixel dashes are lost when the view moves (the comment at `render.rs:5110`). A minimum of 2 px per segment on the axis path is mandatory, otherwise the dash-dot turns into "long dash, gap, nothing".
3. **`BUBBLE`/`10.0`/`GRID` — three constants that are easy to forget.** The colour must come from the type into the line **and** into the label; otherwise V/G recolours the line while the label stays blue. The bubble radius comes from the text size, otherwise a 2.5 mm label does not fit into 11 px.
4. **`d.grow` and the label.** The bubble and the label grow by the annotation multiplier, while the line does not (the line is scaled by the pattern in mm). If the label is drawn through `lettering` without a `NoteFrame` with `k = DENSITY * d.paper`, it ends up microscopic or huge on a sheet.
5. **`Part::Text`.** `dress` (`graphics.rs:1298`) for `Part::Text` skips `weight` and `dash` but applies `color`. This must be preserved: V/G must not hatch the label, but must recolour it together with the line.
6. **Two switches of Type Selector/Edit Type.** `type_names` without `default_type` gives an empty list or an empty current value; `default_type` without `type_names` — an unavailable button. Check both scenarios (axis selected / the Grid tool active).
7. **`deny_unknown_fields` and old builds.** Without a `FORMAT` bump, a file with `"type"` on a grid axis will not be read by version 19 as an "unknown field" rather than as a "newer version". The bump + `typed_grids` are a mandatory pair.
8. **Lettering and diagnostics.** Today the axis label goes through `font::` and `Drawing::text_turned`, which reports to `notes` about a glyph replaced with `?` (`render.rs:476–488`). `text::layout` uses the system fallback and stays silent. Two tests in `crates/core/tests/drawing_lettering.rs` (`unsupported_grid_letters_are_reported_once_and_survive_native_reopen`, `engineering_marks_are_visible_and_distinct_in_exported_grid_labels`) check exactly this behaviour — their expectations change deliberately (§10, decision 5). If keeping the diagnostics is mandatory — keep the `font::` path and do not allow choosing a system font, which contradicts the issue.
9. **Library ids/GUIDs.** Three places in `library.rs` (`label`, `parse`, `Display`) plus `library_names` and `lib_refs_mut`, plus `insert.rs` — omitting any of them gives either "unknown kind" on read, or a lost type on insert, or a dangling reference in `check`. A round-trip test `Kind::GridType ↔ "grid-type"` is needed.
10. **`modify` and JSON.** `modify {ids, type}` goes through the element's `serde_json`: `type: null` or `""` will produce a `check` error whose message must name the axis, not "invalid type".
11. **Palette and multi-selection.** For several axes with different types `shared_type` returns `None`: the Type Selector must show empty, and Edit Type must open the type of the first axis (as it does for dimensions today); it must not crash.
12. **Sections.** In a section `grid()` receives already clipped `lo/hi` and `along`; the type is looked up by the element, not by pixels, so the crop and `GRID_PAST` (`render.rs:1388`) are preserved unchanged.
13. **Duplicate names.** Renaming a type must not change the axis names (the axis name is a separate field) and must not touch the axes until the dialog is confirmed.
14. **IFC and Insert.** Import does not create axes, export does not write styles — do not extend the scope; but `insert` must bring the type, otherwise an axis from another project will not pass `check`.

## 5. Tests

> Tests for this task are **not run locally** (project rule); they are written and checked by CI in the PR. Below is what must be covered.

### 5.1. Core (`crates/core`, through `Session::execute`)

- `grid_type` without `name` lists the types and fonts and changes nothing (`history` is empty).
- All starting presets exist in a new project (`Grid - ISO 128`, `Grid - DIN 1356`, `Grid - ASME Y14.2`, `Grid - Architectural`, `Grid - Continuous`, `Grid - Compact`), pass `check`, are listed by `grid_type`; `default_type("grid")` — `Grid - ISO 128`.
- A custom type: `line_pattern` adds a pattern, `grid_type` — a type on it, `grid {type}` puts it on an axis; a preset can be renamed, modified and deleted (if not in use).
- A new axis gets the default type; `grid {type}` sets the named one; an unknown one — an error with the list.
- `grid_type {name, like}` creates a copy type; editing `line_pattern`, `line_pen`, `color`, `font`, `text_size` changes only the type; `rename` carries the axis references along; `delete` of a type in use — an error with the axis name; `delete` of a free one — success.
- `check`: a type with a non-existent pattern, pen 17, an empty font, `text_size` out of bounds — errors with a clear text; an axis with a non-existent type — an error with the list.
- `modify {ids, type}` switches the type; `inspect` contains `type` on a grid axis.
- Saving and reopening: the type and the references to it are preserved; `Kind::GridType` is read/written (`"grid-type"`).
- Migration: a format 19 file (without `grid_types`, without `type` on a grid axis) opens, the axis gets the default type; the rejection of a format 20 file is covered by the existing test `mep_tests.rs:1094` (`FORMAT + 1`).
- `insert`: a source project with an axis of its own type — after insertion the type exists, `check` passes; on a name collision the type is not overwritten (as with the other records).
- `purge` does not carry away grid axis types and does not break axes.
- `hostile_params_never_panic_or_break_the_model` passes with the new command (it is in `ALL`, the set is walked automatically).

### 5.2. Drawing (`crates/core/tests/`, the output is PNG pixels)

- An axis with a `Dash Dot` pattern differs from a solid one and from `Dash` (image comparison, not numbers).
- The presets are distinguishable: the pictures of `Grid - ISO 128`, `Grid - DIN 1356`, `Grid - ASME Y14.2`, `Grid - Architectural`, `Grid - Continuous`, `Grid - Compact` do not match pairwise.
- Changing `line_pen`/`color` of the type changes the line **and** the label; changing `font`/`text_size`/`width_factor` changes the bubble.
- V/G on top of the type: `graphics {categories: {grid: {projection: {dash: "dotted", color: …}}}}` overrides the pattern and colour of the type; hiding the `grid` category still removes the axis (`graphics.rs` tests 2225).
- The pattern scales: an axis in a 1:50 and a 1:200 plan gives a different number of dashes over the same model length (now — the same).
- A section/elevation draws the axis by its type in the same way as a plan (the same `grid()`).
- Crop: the bubble stays beyond the crop edge, the line is clipped (the behaviour of `render.rs:3317`).
- `drawing_lettering.rs`: the label tests (`1′`, `1″`, `1…`) remain meaningful; the test about `?` and `notes` is rewritten for the new lettering path (an expected change, §10, decision 5).

### 5.3. IFC

- `crates/core/tests/ifc_grids.rs` passes unchanged: the axis geometry, `IfcGridAxis`, GUIDs — as they were.

### 5.4. UI (`crates/ui-egui/src/tests/`, `egui_kittest`)

- A selected axis: Edit Type is active and opens Grid Type Properties (now the button is unavailable — the test fails before the change).
- The dialog: changing the pattern/font/size is applied on Apply and rolled back on Cancel; Duplicate creates a type, Rename carries the axes along; a duplicate with a taken name shows an error in the dialog.
- Type Selector: the type list for an axis is not empty, choosing one switches the type (through `modify`), the Grid tool places a new axis of that type.
- Deleting a type that an axis uses from the dialog — an error, the dialog does not close.
- Multi-selection of axes with different types: the palette does not crash, Edit Type opens the type of the first one.

### 5.5. Quality control

- `cargo check`, `cargo build`, `cargo clippy -p bimcraft-core -p bimcraft-ui-egui` (without `--all-targets`) — clean.
- `bash scripts/debug.sh --non-interactive` — the debug build passes.
- CI in the PR — green.

## 6. Documentation and translations

- `README.md`:
  - command table: the `grid` row (`README.md:601`) gets `type?`, and `grid_type` appears next to it;
  - the Annotation section (`README.md:248`): one or two paragraphs about grid axis types (the pattern in mm on paper, the label font, Edit Type) — following the paragraph about dimension types;
  - Datums (`README.md:227`): "grids that name themselves…" is supplemented with "of a grid type, edited with Edit Type".
- `docs/areas/views-and-graphics.md`: a new subsection "Grid types" (fields, V/G priority, mm on paper, migration) and a table of the starting types: ISO 128 / EN ISO 7519, DIN 1356, ASME Y14.2, office variants (with a note that the office ones are not a standard).
- `docs/areas/ifc-exchange.md`: one line — axis styles are not exported to IFC (out of scope).
- `CLAUDE.md` (root — the module map): a line about `grids.rs` in the `crates/core` list; a paragraph in `crates/core/CLAUDE.md` if needed.
- Translations: new strings (`msg!("Grid - ISO 128")`, `msg!("Grid - DIN 1356")`, `msg!("Grid - ASME Y14.2")`, `msg!("Grid - Architectural")`, `msg!("Grid - Continuous")`, `msg!("Grid - Compact")`, pattern names and dialog group captions) — `cargo i18n update`, the `.po` catalogues in `translations/ui` and `translations/project`.

## 7. Acceptance criteria

1. `GridType` exists, is validated in `Model::check`, is serialised with `deny_unknown_fields` and full defaults.
2. `Grid` stores the type name (`"type"`), the type is validated, old files get a type during migration; `FORMAT` = 20.
3. `type_names("grid")`/`default_type("grid")` work: the Type Selector on an axis is not empty, Edit Type is active.
4. A new project has the starting types by standard (ISO 128 / EN ISO 7519, DIN 1356, ASME Y14.2) and the office variants; they pass `check` and are distinguishable in a drawing; the user can choose any of them, modify, duplicate or delete an unused one.
5. The user can create their own pattern and their own grid axis type and put it on any axis — the choice is not limited to the presets.
6. Edit Type opens Grid Type Properties; the pattern, weight, colour, font, size, style and width factor are changed there; OK/Apply/Cancel/Duplicate/Rename work through commands, not by editing the model.
7. The axis pattern is a `LinePattern` in mm on paper; the dash scales with the view and with printing; the dot does not disappear (minimum 2 px).
8. The axis label is drawn with the font and size of the type (a system font, IBM Plex or our lettering) and is centred in the bubble; the bubble radius is derived from the text.
9. Plan and section draw the axis by its type in the same way.
10. V/G (`dash`, `color`, `weight`, hiding the category) is still stronger than the type.
11. `grid {type}`, `grid_type` and `modify {ids, type}` work through the CLI/MCP; `inspect` shows the axis type.
12. Insert from File brings the grid type; `purge` does not touch grid axis types; IFC has not changed.
13. The tests of §5 are added; the existing tests, apart from the deliberately changed ones (§4.8), are green in CI.
14. README, `docs/areas/views-and-graphics.md`, `docs/areas/ifc-exchange.md`, the module map and the translations are updated.

## 8. How to verify manually

1. Open a project, draw an axis (GR), select it: the Type Selector shows the type and lists the starting types (`Grid - ISO 128`, `Grid - DIN 1356`, `Grid - ASME Y14.2`, `Grid - Architectural`, `Grid - Continuous`, `Grid - Compact`), Edit Type is active; choosing DIN or ASME immediately changes the axis dash.
2. Edit Type → change Line Pattern to "Dash Dot", the weight to 6, the colour to red: the line and the label are red, the dash is dash-dot; Apply, then Cancel — the changes are rolled back step by step via undo.
3. Duplicate the type, rename it to "Оси ISO", put it on the second axis through the Type Selector: the axes have different types, visible in the palette.
4. Install ISOCPEUR in the system, choose it in Text Font, size 3.5 mm: the label in the bubble changes its style, the bubble grows.
5. Reduce the view scale from 1:100 to 1:200: the dash on paper stays the same (12 mm), while on screen it shortens together with the view (now it does not change at all), the bubbles get smaller.
6. Open a section with axes: it is drawn by the same type.
7. In V/G override the dash of the `grid` category to `dotted`, the colour to green: the view overrides the type; hide the category — the axes disappear.
8. Save, open in a text form: the axis has `"type"`, the project has `grid_types`; open the file again — everything is in place.
9. Insert an axis from another project (Insert from File): the type arrives, the axis is editable.
10. Export IFC and open it in IfcOpenShell: the axes are in place, as before the task.

## 9. Out of scope

- Bundling the ISOCPEUR/DIN fonts and any others: clean-room; the font is taken from the system (`text::system`). This is the answer to the third item of the issue, it requires no code.
- Exporting the axis style to IFC (`IfcStyledItem`, weights, dashes): a separate task; today IFC carries geometry.
- Exporting axes to DXF: the project has no axis export to DXF.
- Revit "Center Segment" / "End Segment" (a continuous axis without a bubble, an axis with a bubble on one end only, a break in the middle): separately, if requested.
- Overriding the axis type at the view level (V/G sets only colour/weight/coarse dash — we leave it as is).
- Axes in the 3D view and in schedules.
- Moving all annotation (levels, tags, callouts) to `TextType` — axes only.
- A master type "per ISO/DIN/ASME standard" with automatic font selection by locale and dash length recalculation by line thickness.
- Changing the existing `LineStyle`/`LinePattern` (the task only reads patterns).

## 10. Decisions taken and open questions

**Taken:**

1. `GridType` is a separate library record (`Kind::GridType`), like `DimensionType`; not `Kind::Type("grid")`, because `Model::types` is a map by category with a different type structure, while `Kind::Type` is used for model types (wall/floor/…), which `insert` and `purge` treat differently.
2. The type fields mirror the `DimensionType` vocabulary (`line_pattern`, `line_pen`, `color`, `font`, `text_size`, `bold`, `italic`, `underline`, `width_factor`) instead of referencing `LineStyle`/`TextType`: a grid axis has no "line style" in the Revit sense, and a reference to `TextType` would drag the renaming and deletion of a text type into the related grid axis types. The reference to `LinePattern` is by name — because patterns are already library records and are renamed together.
3. The line type is a pattern in mm on paper, not pixels; a dot in the pattern is raised to 2 px in rendering (the retained-canvas workaround, as now), without changing `retained.rs` in this task.
4. V/G is stronger than the type: `dress` is applied after `grid()` and overrides the colour, weight and dash (`Solid`/`Dashed`/`Dotted`).
5. The label is drawn through `text::layout` + `lettering` (the project font, including a system one), not `font::`; the consequence is that grid axes lose the `?` diagnostics in `notes` and two tests in `drawing_lettering.rs` change. This is a deliberate trade-off: without it the font from the issue (ISOCPEUR/DIN) is unreachable.
6. Grid axis types do not participate in Purge Unused and do not appear in the Families of the Project Browser tree — like dimension and text types; they are visible in the Type Selector and in `library`.
7. The starting set is the presets ISO 128 / EN ISO 7519, DIN 1356 and ASME Y14.2 plus office variants (`Grid - Architectural`, `Grid - Continuous`, `Grid - Compact`); the default type is `Grid - ISO 128`; Russian standards (GOST) are not added by the customer's decision; old projects get the set through migration.
8. There are no axis styles in IFC (out of scope), `insert` brings the types.

**Open questions:**

1. **Dash lengths.** The presets pin one variant per standard: ISO 12.7/3.2/dot/3.2, DIN 20/4/dot/4, ASME 20/3/5/3. In the standards the lengths are given in multiples of the line thickness and by groups, so this is our reading — agree on it with the issue author and correct the F2 table if needed. For ISO the dash is noticeably longer than today's (14 px ≈ 2.8 mm of paper) — this is an expected change in appearance.
2. **The 2 px dot** — keep the workaround or fix `retained.rs` (the 1 px loss when the view moves). Proposal: the workaround now, a separate PR for the canvas.
3. **Label and bubble size.** The starting 2.5 mm give a cap height of ≈1.75 mm against today's 2.0 mm (10 px at `DENSITY = 5`), that is the label is slightly smaller; if exact continuity is needed — 2.8 mm. And the bubble radius — derive it from the text (proposed in F4) or make it a type parameter ("Bubble Size", as in some BIM systems)?
4. **Whether to show the `Type` row in the palette** for a grid axis (the rows at `properties.rs:228` today), or whether the Type Selector and Edit Type suffice, as for dimensions.
5. **The relation to `TextType`**: if a shared text type is needed for axes and notes (renaming/deletion with propagation), that changes decision 2 — then `grid_type` gets `text_type?: string` instead of the inline fields. Decide before starting work.
6. **A continuous axis without bubbles** (Revit "Center Segment") — is it needed in the first PR.

## 11. Work plan (atomic commits)

1. **`grids.rs`: `GridType` and the starting set.** Structure, defaults, `check`, `types(language)`/`default_type`, `text_type()`; the starting preset patterns in `lines::patterns`; unit tests of the module. The module is not wired to anything yet, the project behaviour does not change.
2. **Model and library.** `Model::grid_types` (+ the seed in `resolve`), `Grid.type_name`, `type_names`/`default_type` for `"grid"`, validation in `check`, `FORMAT` 20 + `typed_grids()`, `Kind::GridType` in all five places of `library.rs`, `insert.rs`. Tests: read/write, migration, insert, purge.
3. **Commands.** `grid {type}`, the new `grid_type` (+ the `ALL` record with documentation), `modify {ids, type}` — with a test. The Type Selector and Edit Type do not work yet: the UI is untouched.
4. **Rendering.** `render::grid(d, m, g)` by type: the pattern in mm through `px_per_mm`, a minimum of 2 px for the dot, the label through `text::layout`/`lettering`, the bubble from the text; both call sites. Update `drawing_lettering.rs` and add new PNG tests (pattern, scale, V/G, crop, section).
5. **UI: dialog and palette.** `gridtypes.rs`, the `grid_type_dialog` field, calling the window, the `edit_type` branch for `"grid"`, the palette rows (if §10 question 4 is decided so), remembering the type in the tool.
6. **UI tests** (`egui_kittest`): Edit Type, the dialog, the Type Selector, deleting a type in use, multi-selection.
7. **Documentation and translations** (§6): README, `docs/areas/*`, the module map, `cargo i18n update`.

Every commit builds (`cargo check`) and does not break existing behaviour; the tests are written together with the commit they belong to and are run by CI.

## 12. Scenario traceability

| # | Scenario | Requirement | Test |
|---|---|---|---|
| 1 | Edit Type on an axis opens Grid Type Properties | F3, F8 | 5.4 (dialog), 5.1 (`type_names`) |
| 2 | The Type Selector on an axis is not empty and switches the type | F3, F8 | 5.1 (`modify`), 5.4 (selector) |
| 3 | The line pattern is chosen from the project patterns | F1, F4 | 5.1 (`check`), 5.2 (PNG) |
| 4 | The label font is a system one (ISOCPEUR/DIN) | F4 | 5.2 (PNG by font), 5.4 (dialog) |
| 5 | The dash scales with the view and with printing | F4 | 5.2 (1:50 against 1:200) |
| 6 | The dot of the dash-dot does not disappear | F4, §4.2 | 5.2 (pattern PNG) |
| 7 | Plan and section draw the axis in the same way | F4 | 5.2 (section) |
| 8 | V/G is stronger than the type | F4, §10, decision 4 | 5.2 (V/G), 5.4 |
| 9 | An old project opens, the axis gets a type | F7 | 5.1 (migration) |
| 10 | `grid {type}`, `grid_type`, `modify` through the CLI/MCP | F5 | 5.1 (commands) |
| 11 | Insert from File brings the type | F6 | 5.1 (insert) |
| 12 | IFC has not changed | F6 | 5.3 (`ifc_grids.rs`) |
| 13 | The bubble matches the type size | F4 | 5.2 (PNG), 5.4 |
| 14 | Multi-selection of axes with different types does not break the palette | F8, §4.11 | 5.4 |
| 15 | The user chooses an ISO/DIN/ASME preset and puts it on an axis | F1, F2 | 5.1 (presets), 5.2 (PNG) |
| 16 | A custom grid line: pattern + type + axis | F1, F2, F5 | 5.1 (custom type) |
