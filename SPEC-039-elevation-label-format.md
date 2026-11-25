# SPEC-039. Elevation label format: own units, precision and marker

Status: **draft**, implementation required
Date: 2026-11-25
Project: BIMCraft 032 (Rust; `crates/core/src/render.rs` — `height` :6813, `spot_mark` :6771, calls
:1616, :3316, :4412, level label :3005, `level_names` :2324; `crates/core/src/spots.rs` —
`SpotCoordinateType` :97, `Symbol` :44, `types` :365, `default_type` :378; `crates/core/src/model.rs` —
`Level` :262, `Spot` :1801, `spot_reads` :8967, `level_shown` :9027, type maps :4691, type
checking :9575, `type_names` :6661, `default_type` :6696; `crates/core/src/library.rs` — `Kind` :61,
`:897`; `crates/core/src/commands.rs` — `spot` :14937, `spot_coordinate` :15036,
`spot_coordinate_type` :15076, `level` :360, `modify` :20662, `element_json` :5149;
`crates/ui-egui/src/properties.rs` — `edit_type` :1802; `crates/ui-egui/src/spots.rs` —
`SpotTypeDialog` :25; `crates/ui-egui/src/gridtypes.rs` — the sample type dialog;
`docs/areas/views-and-graphics.md`)
Source: discussion IPedrax/BIMCraft#616 "Locale-aware elevation labels: ±0.000, three decimals
(Chinese GB/T 50001)" (linjiahljh, 24.11.2026) and a code review: the elevation label ignores the
project precision, elevations and levels have no types of their own, and label units cannot differ
from plan units. The sign setting (`+` for positives, `±` for zero) is moved to SPEC-036 as a common
project setting and is not duplicated here.
Related: SPEC-036 (units and signs in values), SPEC-040 (user standards: under a built-in standard
these types come from the standard and are locked), discussion #617 (annotation families; does not
overlap).
Task type: annotation format; `.bimc` is extended additively, `FORMAT` does not change.

> **In short.** Elevation labels (spot elevation) and level labels (in sections and elevations) get
> types of their own — `SpotElevationType` and `LevelType` — with their own units, precision, marker
> and text style. `render::height` stops ignoring the project `decimals`: this is a separate bug
> because of which, with `units m, decimals 3`, an elevation prints `+3.60` instead of `+3.600`. The
> sign itself is not a property of the type but the common project setting of SPEC-036:
> `units {signs: {positive: never, zero: signed}}` together with a "metres, three decimals" type
> gives the Chinese practice GB/T 50001: `±0.000`, `3.600`, `-0.450`. Defaults reproduce today's
> drawing; schedules and Properties stay in project units per SPEC-036. `"symbol"` is marker
> geometry, not part of the number format. If the project follows a built-in technical standard,
> spot elevation and level types come from it and are not edited: editing them is possible only in
> one's own standard (SPEC-040).

## 0. General working rules

1. Language — Russian; code and strings — English; window strings — `msg!`/`t()`.
2. Tests — only on request; CI separately. Allowed: `check`, `build`, `clippy` (without
   `--all-targets`), `fmt --check`, `i18n check`.
3. **Commits are atomic.** One commit — one logically complete change; unrelated changes are not
   mixed. Every commit builds and does not break existing behavior.
4. Before a commit — `git diff` and `git status`; do not capture build artifacts, temporary files or
   other people's changes.
5. After a push — wait for green CI; on failure find the cause, fix it with a separate atomic commit
   and repeat until a successful run.
6. No new dependencies; `unsafe` is not added.
7. **`FORMAT` is not bumped.** New fields are additive (`serde(default)`), old files are read, new
   keys in an old version end up in `left_out` (`lenient.rs`) rather than a refusal.
8. `.md` files in the project follow the project language.

## 1. The essence of the problem

| Where | Now | Problem |
|---|---|---|
| Spot elevation label | `height(units, z)` (`render.rs:6813`) | the project `decimals` is not applied |
| Label calls | `spot_mark(..., m.units)` (`render.rs:1616`, `:3316`, `:4412`) | only units are passed: with `m, decimals 3` the result is `+3.60`, not `+3.600` |
| Level label in a section/elevation | `m.length(m.level_shown(l))` (`render.rs:3005`) | project units and precision, but no own unit; with plans in mm an elevation in metres cannot be shown |
| Level label width | `level_names` computes `m.length(...)` (`render.rs:2324`) | the layout will break if the label format is changed in one place |
| Spot elevation type | no type; `Spot.type_name` is documented as "spot coordinate's type" (`model.rs:1815`) | there is nothing to set the format with; the command response does not contain the label text |
| Level type | no type | the level label can be neither configured nor reused |
| Sign and zero | `height()`: `+3000`, `±0`; the level has no sign | solved by the common project setting in SPEC-036, not in types |
| Technical standards | `TechnicalStandard` (NBR/ISO/ANSI/EU/JIS) defines lines, sheets, styles | the elevation format is not configurable; GB/T 50001 is missing |
| Type lock | no types, nothing to lock | will appear with types; the lock rule is SPEC-040 |

Causes:

1. `height()` takes `Units`, not the project precision, so `decimals` never reaches the label.
2. The label format is not a property of the annotation but a code branch: it can be neither saved
   nor overridden.
3. Units and precision exist only project-wide. But elevations in Chinese practice are metres with
   three decimals, while plans are millimetres; one project setting cannot express that.
4. Spot elevation and level have no type, although spot coordinate, text, dimensions and grids do.

## 2. Goals

1. New project types: `SpotElevationType` and `LevelType`, in the library (`Kind`), like
   `SpotCoordinateType`.
2. A common `ElevationFormat`: `units: Option<Units>` (None — project units), `decimals: Option<u8>`
   (None — project precision; `Some(n)` — exactly `n` decimals).
3. `LevelType` additionally: `show_name` (default `true`), `symbol` (`none|triangle|circle`, default
   `none`), `symbol_size`; text style, as in `SpotCoordinateType`.
4. `SpotElevationType` additionally: `symbol` (`none|triangle` — marker shape), `symbol_size`
   (default: today's filled triangle 5×8 px) and text style.
5. `height()` and the level label use one formatter; the project `decimals` is taken into account.
6. Chinese practice: a type with `units m`, `decimals 3` plus the project sign setting from SPEC-036
   (`positive never`, `zero signed`) → `±0.000`, `3.600`, `-0.450`.
7. The sign of positives and zero comes from `Model.signs` (SPEC-036) or, if it is not set, from the
   annotation's built-in behavior; SPEC-039 types have no sign fields.
8. Commands `spot_elevation_type` and `level_type`; `spot`, `level` and `modify` accept `type`;
   responses give the ready label text.
9. GUI: Type Properties for both types, Edit Type from Properties, Type Selector.
10. Old projects are read without migration and drawn as before (for projects where `decimals` is
    not set).
11. IFC, schedules, Properties and export do not change.
12. Types stay ordinary library records; the "under a built-in standard" lock and storing formats
    inside a standard are added by SPEC-040, without changing the type format from this SPEC.

## 3. Non-goals

- Do not set signs in annotation types: that is the project setting of SPEC-036. A type is
  responsible for units, precision, marker and typeface, not for `+`/`±`.
- Do not change the value format in Properties, Family Types and schedules — that is SPEC-036. The
  level schedule (`Elevation` as `Kind::Length`) stays in project units and precision.
- Do not add `TechnicalStandard::Gb` in this SPEC: the preset requires sources for lines, sheets and
  fonts (see §13, question 3). The format is configurable by type already.
- Do not change the default spot elevation marker geometry or its position relative to the point.
- Do not change level labels in 3D: only the name is drawn there (`view3d::level_marks`).
- Do not touch IFC: elevation labels are not exchanged.
- Do not do "up to N decimals" with trailing zeros trimmed: `decimals` means exactly N decimals.
- Do not move `symbol` into the number format (the mistake of #616).
- Do not change `Spot.origin`/`Level.elevation_base`: where the elevation is read from is a property
  of the instance, not of the type.
- Do not add leaders and shoulders to spot elevation: there are none today, and this is not a task
  about the format.
- Do not introduce the standard lock and user standards here: that is SPEC-040. SPEC-039 only leaves
  types as ordinary records that SPEC-040 will be able to lock.

### 3.1. Relation to SPEC-036 and SPEC-040

| Question | SPEC-036 | SPEC-039 | SPEC-040 |
|---|---|---|---|
| What is configured | units in UI values and signs (`+`, `±`) | label units and precision, marker, level name | the whole technical standard, including label formats |
| Where it is stored | `show_units` — `settings.json`; `signs` — project | project types in `.bimc` | built-in — in code; own — in the project, a library record |
| What changes in the file | `Model.signs`, additively | new type maps, additively | `Model.standards` and a reference to the active one, additively |
| Formatter | `Units::show`, `dims::signed` | `ElevationFormat::text` calls `dims::signed` | does not format; stores type values |
| Schedules | the unit in the header or the cell | do not take the label format | do not take it |
| Lock | — | a single write path, so that SPEC-040 can lock | under a built-in standard types are read-only |

Common primitives — `Units::length_to` (`units.rs:110`), `dims::rounded` (`dims.rs:293`) and
`dims::signed` (SPEC-036 §4.1). A second number formatter is not introduced: `ElevationFormat` only
chooses units, precision and the marker, while the sign and rounding are done by the common layer.

## 4. Key decision

**`ElevationFormat` — a field of the two new annotation types; one formatter for the whole project;
"symbol" stays marker geometry; the sign comes from SPEC-036.**

```rust
// crates/core/src/elevation.rs
pub struct ElevationFormat {
    pub units: Option<Units>,       // None — project units
    pub decimals: Option<u8>,       // None — project precision
}

impl ElevationFormat {
    /// Elevation `mm` as text: units and precision from the type, sign from `Model.signs` (SPEC-036).
    pub fn text(&self, mm: f64, project: Units, project_decimals: Option<u8>, signs: Option<Signs>) -> String;
}
```

How the text is composed (example `m`, three decimals):

| Project | `3.6 m` | zero | `-0.45 m` |
|---|---|---|---|
| `signs: null`, spot elevation | `+3.600` | `±0.000` | `-0.450` |
| `signs: null`, level label | `3.600` | `0.000` | `-0.450` |
| `signs {positive: never, zero: signed}` | `3.600` | `±0.000` | `-0.450` |

The full table of the four combinations and the rounding rule — SPEC-036 §4.1.

Rules:

- `decimals: Some(n)` — exactly `n` decimals with trailing zeros (`3.600`, `0.000`). `None` — project
  precision (`Model::decimals`), and if there is none — as the units show (`Units::length`).
- `decimals` is allowed only for `mm`, `cm`, `m` and `ft`; for `in`/`ft-in` — an error
  (`Model::check`), as with the `units` command. Values — 0..=6 and within the range of
  `Units::decimals()` for the chosen units.
- A level with `show_name: false` prints only the elevation; `show_name: true` — the name above the
  line, the elevation below the line, as now.
- Marker: `SpotElevationType.symbol` — `none|triangle`; `LevelType.symbol` —
  `none|triangle|circle`. The level `triangle` — an open equilateral triangle of size `symbol_size`,
  apex down, on the level line, the number above the line (GB/T 50001; check the shape against the
  standard's figure during implementation).

Defaults (reproduce today's drawing in a project without `decimals` and without `signs`):

| Type | units | decimals | symbol | show_name |
|---|---|---|---|---|
| `Spot Elevation` | None | None | `triangle` (filled) | — |
| `Level Head` | None | None | `none` | `true` |

One deliberate change: in a project where `decimals` is set, spot elevation is now printed with the
same precision as the rest of the drawing. This fixes the bug from §1; it is described in §9 and
checked by a test.

### 4.1. How this relates to #616

| #616 | Here | Why |
|---|---|---|
| `decimals: 3` | `decimals: 3` | "exactly three decimals", not "up to three" |
| `zero: "±"` | `signs.zero` in SPEC-036 | the zero sign is a common project setting, not a type field |
| `plus: false` | `signs.positive` in SPEC-036 | does not overlap with zero, no contradictory combinations |
| `unit: "m"` | `units: Option<Units>` | own label units; project units by default |
| `symbol: "triangle"` | the type's `symbol` | geometry, not the number format |
| "on annotation type" | `spot_elevation_type`, `level_type` | the model has annotation types, there is no common "annotation type" |
| `*_type_create` / `*_set` | the `spot_elevation_type`, `level_type` commands | like `spot_coordinate_type`: create/list/modify/rename/delete |
| "regenerate `docs/commands.md`" | not needed | there is no such file in the repository; command documentation is the params doc in `ALL` |
| "file-format default" | `serde(default)` | the format version does not change: fields are additive |
| "IFC output unchanged" | IFC is not involved | labels are drawing only |

## 5. Stage A. Model and library

- New module `crates/core/src/elevation.rs`:
  - `ElevationFormat`, `LevelSymbol`;
  - `SpotElevationType`, `LevelType` with the format fields, text style (as in `SpotCoordinateType`:
    `font`, `text_size`, `bold`, `italic`, `underline`, `width_factor`, `text_color`), `symbol`,
    `symbol_size`; `LevelType` additionally has `show_name`;
  - `ElevationFormat::text` (calls `dims::signed` from SPEC-036), `SpotElevationType::check(name)`,
    `LevelType::check(name)`, `spot_types()` and `level_types()` (one default each), `MAX_TYPES` (as
    in `spots::MAX_TYPES`);
  - unit tests for the format.
- `Model` (`model.rs:4691`):
  - `#[serde(default = "crate::elevation::spot_types")] pub spot_elevation_types: BTreeMap<String, SpotElevationType>`;
  - `#[serde(default = "crate::elevation::level_types")] pub level_types: BTreeMap<String, LevelType>`;
  - initialization in `Model::new` (`:4876`);
  - `type_names` (`:6661`): `"spot"` → `spot_elevation_types`, `"level"` → `level_types`;
  - `default_type` (`:6696`): `"spot"` → the `spot_types` default, `"level"` → the `level_types`
    default;
  - `Model::spot_elevation_type(&Spot)`, `Model::level_type(&Level)` — name resolution with the
    default;
  - `Model::elevation_text(&Spot)` and `Model::level_elevation_text(&Level)` — the single formatting
    point for rendering, commands and responses; the sign is taken from `m.signs` (SPEC-036), and
    with `None` — the built-in behavior: spot `always/signed`, level `never/plain`.
- `Spot.type_name` (`model.rs:1816`): the documentation — the spot elevation type when
  `coordinate == false`; the behavior keeps the same field.
- `Level` (`model.rs:262`): `#[serde(default, rename = "type", skip_serializing_if = "Option::is_none")] pub type_name: Option<String>`;
  `None` — the default type. Add the field to `Level::new`.
- `Model::check`:
  - both maps are no larger than `MAX_TYPES`; every type passes `check`;
  - for `Element::Spot`: `coordinate == true` → the name from `spot_coordinate_types`; otherwise the
    name from `spot_elevation_types`; a name that does not exist is an error with a hint;
  - for `Element::Level`: `type_name` is either `None` or exists in `level_types`;
  - `decimals`: 0..=6 and within the range of `Units::decimals()` for the chosen units; for
    `in`/`ft-in` — only `None`;
  - `symbol_size` 0..=50 mm, `text_size` > 0 and ≤ 100, `width_factor` 0.1..=10, the font is not
    empty (the same limits as in `SpotCoordinateType::check`, `spots.rs:269`).
- `library.rs`:
  - `Kind::SpotElevationType`, `Kind::LevelType` (`:61`), name/string pairs (`:114`, `:156`, `:198`);
  - `add(...)` for both maps (`:897`);
  - `copy_definition` for Insert from File — as with other types.
- `lenient.rs` needs no changes: maps are read by type, unknown keys are dropped by the common walk.
- The standard membership field is **not added** in this SPEC: it is introduced by SPEC-040.
  SPEC-039 types stay ordinary library records; a standard stores its formats as copies inside its own
  definition and, on Apply, materializes them into these maps.

## 6. Stage B. Formatter and rendering

- `render::height` (`:6813`) is removed; `spot_mark` (`:6771`) takes the ready text and
  `symbol`/`symbol_size`.
- Calls:
  - plan (`:1616`), section/elevation (`:3316`), sheet (`:4412`) — through `m.elevation_text(sp)`
    and the type `m.spot_elevation_type(sp)`;
  - "`?`" for an unknown elevation is preserved and does not go through the format.
- Level label (`:3005`): text — `m.level_elevation_text(l)`, name visibility — `show_name`, marker —
  `symbol`/`symbol_size`; the width in `level_names` (`:2324`) is computed from the same text.
- Sign and rounding — `dims::signed` (SPEC-036 §4.1): the label does not build them itself, so that
  the rule is one per project.
- Marker drawing:
  - spot: `symbol: triangle` — today's filled triangle; `none` — without it, only text on the line;
  - level: `triangle` — an open triangle on the line (apex down), `circle` — a circle of radius
    `symbol_size / 2` at the end of the line, `none` — as now.
- `±` already exists in `font.rs:104`, `+`/`-` are drawn by our font; do not add separate glyphs.
- 3D (`view3d::level_marks`) does not change.

## 7. Stage C. Commands

- `spot_elevation_type` — following `spot_coordinate_type` (`commands.rs:15076`):
  - without `name` — a list `{types, fonts}`;
  - `{name, like?, …fields}` — create/modify, missing fields from `like` or the defaults;
  - `{name, rename}` and `{name, delete}`; deleting a type that something is placed with — an error
    with a list of elements (as with spot coordinate, `:15085`);
  - unknown keys are rejected (`only`), values are checked before writing.
- `level_type` — the same, plus `show_name` and `symbol`.
- `spot` (`:14937`): `type?` — the name of a `SpotElevationType`; without it, the default type.
  Response: `{"id", "elevation", "text", "type", "host"}`, where `text` is the ready label.
- `level` (`:360`): `type?`; the response and `elements` give `"type"` and `"elevation_text"`.
- `modify` (`:20662`) already changes the fields that `elements` shows; it is enough that `type` is
  in the level JSON and in the spot elevation JSON. Add a `modify {type}` test for both.
- Signs are changed not by the type but by `units {signs: …}` (SPEC-036 §5); type commands do not
  accept `signs`.
- `elements` (`element_json`): spot elevation gets `"text"`, `"type"`; level — `"type"`,
  `"elevation_text"` (`elevation_shown` in hundredths of a millimetre stays a number and does not
  change).
- Read-only: modifying commands refuse automatically; the type list is available.
- In `ALL` — params docs for agents; the menu — as with `spot_coordinate_type`.
- `hostile_params_never_panic_or_break_the_model`: add payloads with the real keys of the new
  commands and type fields.
- Lock: if a built-in standard is active (SPEC-040), `spot_elevation_type` and `level_type` refuse
  create/modify/rename/delete with a hint to make one's own copy of the standard; the list (`types`)
  stays available. The check is not implemented here — the commands must have a single write path
  where SPEC-040 will insert it.

## 8. Stage D. Interface

- `crates/ui-egui/src/spots.rs`: `SpotTypeDialog` (`:25`) currently covers spot coordinate only.
  - Add `SpotElevationTypeDialog` (or generalize the existing one by category) and the field
    `spot_elevation_type_dialog` in `BimApp` (`lib.rs:975`, `:2041`, `:2890`).
- New `crates/ui-egui/src/leveltypes.rs` — `LevelTypeDialog` following `gridtypes.rs`
  (`GridTypeDialog`): `new`, `params`, `level_type_for`, the window; registration in the same place
  as `grid_type_window` (`lib.rs:7865`).
- `properties.rs::edit_type` (`:1802`): branches `"spot"` → the spot elevation dialog, `"level"` →
  the level dialog, next to `"spotcoordinate"` (`:1816`) and `"grid"` (`:1820`).
- Dialog fields:
  - **Units** — `Project` (default), `mm`, `cm`, `m`, `ft`, `ft-in`;
  - **Decimals** — `As the units show` / project precision / 0…5 (unavailable for `in`/`ft-in`);
  - **Show level name** — level only;
  - **Symbol** — `None` / `Triangle` / `Circle` (spot — only `None`/`Triangle`), **Symbol size**;
  - text style — as in the neighbouring type dialogs.
- There are no sign (`+`/`±`) fields in type dialogs: they are in Settings > Units per SPEC-036.
- At the bottom — a preview of three values (`0`, `+3.6`, `-0.45`) taking the current `signs` into
  account: the user sees `±0.000 / 3.600 / -0.450` before applying.
- The Type Selector in Properties shows spot elevation and level types (after `type_names`);
  choosing another type does not rewrite already placed elements.
- One Undo step per type creation/edit; the selection is not reset.
- Under a built-in standard, type dialogs will be read-only and lead to Duplicate — that is SPEC-040.

## 9. Compatibility

- `FORMAT` does not change; both maps are additive. An old file without them gets the defaults on
  reading.
- In a project without `decimals` and without `signs` the drawing matches today's: the spot label
  `±0`, `+3000`, `-600`, the level label `0`, `3000`, `-600`.
- In a project with `decimals` the spot label changes to the project precision — this is the fix from
  §1; it is documented in `docs/areas/views-and-graphics.md` and mentioned in the release notes.
- In a project with `signs` set (SPEC-036) the labels of both annotations switch to the common sign —
  that is the point of the setting; with `signs: null` the behavior is as before.
- IFC, schedules, Properties, PDF/PNG export and sheets use the same rendering, so they get the
  format automatically; the level schedule does not (SPEC-036).
- Old `elements`/`spot`/`level` responses are extended with fields; existing keys are not renamed.
- `Spot.type_name` continues to serve both spot coordinate and spot elevation; `Model::check` does
  not let the categories be confused.
- `.bimt` (templates) carry types together with the project, like other library records.

## 10. Test plans

1. Formatter, `m`/3, `signs: null`: spot `±0.000`, `+3.600`, `-0.450`; level `0.000`, `3.600`,
   `-0.450`.
2. Formatter, `m`/3, `signs {positive: never, zero: signed}`: both annotations — `±0.000`, `3.600`,
   `-0.450`.
3. Defaults in mm and without `signs`: spot — `±0`, `+3000`, `-600`; level — `0`, `3000`, `-600`;
   the width in `level_names` matches the drawn text.
4. A project `m`/`decimals 3`: spot without a type prints `+3.600` (the bug fix), level — `3.600`.
5. Section rendering: a spot with the GB type — the text `±0.000`; a level with `show_name: false` —
   only the elevation; the `triangle` marker produces triangle primitives; `none` produces none.
6. `spot {"type": …}` returns `text`; `modify {"type": …}` changes the label; `level` and `elements`
   return `type` and `elevation_text`.
7. `spot_elevation_type`/`level_type`: creation via `like`, renaming, deleting a used type — an error
   with a list; an unknown key — an error; `decimals` for `ft-in` — an error; `MAX_TYPES`.
8. `Model::check`: a spot with `coordinate: true` and an elevation type name — an error; a level with
   a non-existent type — an error; `decimals` outside the units' range — an error; a `signs` field
   unknown to the type — an error.
9. An old file without the type maps opens, `left_out` is empty, the drawing matches the reference.
10. UI: Edit Type on a spot elevation opens the elevation dialog, on a level — the level dialog;
    changing `decimals` redraws the section immediately; the preview matches the drawing; the dialogs
    have no sign fields.
11. `i18n check`; `hostile_params_never_panic_or_break_the_model` with the new payloads.
12. Lock (after SPEC-040): under a built-in standard editing a type is refused; under one's own it is
    allowed. Here it is enough that types are ordinary records and the commands have a single write
    path.

Rounding and signs as such are checked in the SPEC-036 §9 tests (11–16); here — that labels take the
ready result.

Existing tests that need fixing: `render.rs:8195`
(`spot_elevations_read_signed_heights_in_the_project_units`), `tests.rs:3661`
(`spot_elevations_read_the_height_of_what_they_stand_on`), `coordinates_tests.rs:525`.

> The tests in this SPEC are described but are not run without a separate request; CI — separately.

## 11. i18n and documentation

- New strings (English msgids, translations — per the project rules):
  - `Spot Elevation Type Properties`
  - `Level Type Properties`
  - `Units`, `Decimals`, `As the units show`, `Project`
  - `Show level name`
  - `Symbol`, `None`, `Triangle`, `Circle`, `Symbol size`
  - `Preview`
- Sign strings (`Positive`, `Zero`, `Always`, `Never`, `±0`, `0`) — in SPEC-036.
- Lock strings (`Built-in standard`, `Read-only`, `Duplicate…`) — in SPEC-040.
- Update `translations/ui/bimcraft-ui.pot` and all `.po` files; `i18n check` must be green.
- `docs/areas/views-and-graphics.md`:
  - spot elevation and the level label: where units, precision, sign and zero come from;
  - the GB/T 50001 example: the type `units m`, `decimals 3` + `units {signs: {positive: never, zero: signed}}`
    and what results;
  - that schedules and Properties stay per SPEC-036; that under a built-in standard the types come
    from the standard (SPEC-040).
- `docs/areas/desktop.md`: Type Properties of elevations and levels, where Edit Type is; where signs
  are (Settings > Units, SPEC-036).
- `crates/core/CLAUDE.md`: the `elevation.rs` module line in the map.
- `crates/ui-egui/CLAUDE.md`: the `leveltypes.rs` module line in the map.
- Do not create `docs/commands.md`: it does not exist, the command params doc lives in `ALL`
  (`commands.rs`).

## 12. Acceptance criteria

- [ ] `SpotElevationType` and `LevelType` exist, are stored in the project and in the library, and are
      read and written without migration.
- [ ] The `units`/`decimals` format works for elevation and level labels; `3.600`/`±0.000` are
      configured by the type together with the signs from SPEC-036.
- [ ] `height()` no longer ignores `decimals`; the spot label and the level label use one formatter
      and the common `dims::signed`.
- [ ] Defaults give today's drawing in a project without `decimals` and without `signs`.
- [ ] Types have no sign fields; the sign is changed only by `units {signs}` (SPEC-036).
- [ ] `spot`, `level`, `modify`, `elements` accept/return `type`; responses contain the label text.
- [ ] `triangle`/`circle`/`none` markers are drawn; the level label width is computed from the actual
      text.
- [ ] GUI: Type Properties, Edit Type, Type Selector, preview.
- [ ] Schedules and Properties are unchanged (SPEC-036), IFC is unchanged.
- [ ] Types contain no lock/standard fields: SPEC-040 adds them without changing these types.
- [ ] Tests §10, i18n, documentation.
- [ ] `check`, `build`, `clippy`, `fmt --check`, `i18n check` — green (per the project rules).

## 13. Risks and questions

| Risk | Answer |
|---|---|
| Two practices in one project | different types in one's own standard or without a standard; under a built-in one — one head from the standard (SPEC-040) |
| An old project with `decimals` changes the spot label | this is a bug fix; described in §9 and in the documentation |
| `decimals` for feet-inches | forbidden by validation: there are sixteenths there, as in `Units::length` |
| Level label width | `level_names` uses the same formatter, otherwise the text overlaps |
| A level without a name | `show_name: false`; the name stays in the browser and in properties |
| The shape of the GB triangle | check it against the GB/T 50001 figure during implementation; test by primitives |
| Deleting a type that something is placed with | an error with a list, as with spot coordinate |
| `Spot.type_name` for two categories | `coordinate` decides which map; `Model::check` catches a mismatch |
| One sign per project | with `signs: null` today's different signs are preserved; the common one is turned on deliberately |
| Under a built-in standard types are locked | editing — only in one's own standard (SPEC-040); the list and type selection stay available |

1. **`level_type` — decided: a separate type.** The decision of 2026-11-25: a level has a type, like
   Revit's Level Head and like the other annotations (Grid, Spot Coordinate). The type is reused,
   duplicated, carried by a template and gives one Undo step per edit. A project setting instead of a
   type was rejected: it does not let two heads live in one file. With SPEC-040 several heads are
   available in one's own standard or without a standard; under a built-in standard there is one
   head, from the standard, and it is not editable.
2. **Where does the type end and the project begin?** Units and precision — the type (it is part of
   the level head); the sign — the project (SPEC-036). If another sign is needed for a separate type,
   that is a separate task: a `signs` override field in the type.
3. **The GB/T 50001 preset.** Add a sixth `TechnicalStandard` in this SPEC or separately? Here — no:
   the preset requires sources for lines, sheets, projection and fonts. With SPEC-040 the GB head can
   be made as one's own standard based on ISO.
4. **The level schedule.** Keep it in project units (SPEC-036) or show the label format? By default —
   keep it: a schedule is a project table, not a drawing.
5. **3D level labels.** Keep only the name or apply `show_name`? By default — keep as is, so as not to
   change 3D without a request.
6. **`origin`/`elevation_base` in the type?** No: where the elevation is read from is a property of
   the instance (`Spot.origin`, `Level.elevation_base`), as today.
7. **The circle marker.** Is it needed in the first version? It is kept for practices where a level
   is marked with a circle; if it is not needed, remove the variant before implementation — the
   fields do not break.

## 14. Definition of Done

- [ ] `elevation.rs`: the format, both types, checks, tests.
- [ ] `Model`, `library.rs`, `Model::check`, `type_names`, `default_type`, `elements`, `modify`.
- [ ] `render.rs`: `height` removed, spot and level go through one formatter and `dims::signed`;
      markers and width.
- [ ] The `spot_elevation_type`, `level_type` commands; `spot`/`level`/`modify` with `type`; responses
      with the text.
- [ ] GUI: dialogs for both types, Edit Type, Type Selector, preview (without sign fields).
- [ ] The lock and storing formats in a standard are not implemented here; the place for them is left
      (SPEC-040).
- [ ] i18n, `docs/areas/views-and-graphics.md`, `docs/areas/desktop.md`, module maps.
- [ ] Tests §10 (including the old file and the `decimals` fix).
- [ ] `check`, `build`, `clippy`, `fmt --check`, `i18n check` — green.
