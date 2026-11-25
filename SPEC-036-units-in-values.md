# SPEC-036. Units and signs in parameter values

Status: **draft**, implementation required
Date: 2026-11-23 (updated 2026-11-25: the sign setting is moved here from SPEC-039)
Project: BIMCraft 031 (Rust; `crates/ui-egui/src/properties.rs` — `shown`, `parse`, `Kind`, Properties
strings; `crates/ui-egui/src/editor/types_dialog.rs` — Family Types; `crates/ui-egui/src/editor.rs` —
component parameters; `crates/core/src/units.rs` — `Units`, `length_to` :110, `decimals` :98;
`crates/core/src/dims.rs` — `rounded` :293; `crates/core/src/render.rs` — `height` :6813, level
label :3005; `crates/core/src/model.rs` — `length` :4770, `decimals_shown` :4786;
`crates/core/src/commands.rs` — `units` :3053, :24000)
Source: in Properties units are shown differently: for `Area` — "12.34 m²", for `Volume` — a number,
while the unit sits in the field name "Volume (m³)"; lengths have no units at all. There is no
common setting. The sign setting is moved here as well (it was in SPEC-039, from discussion #616):
today `+` and `±` are hard-coded in `render::height` (`render.rs:6813`), the level label has no sign,
and UI fields show values without a sign. One project setting must define the sign of positives and
the sign of zero for all values at once.
Related: SPEC-031 (area and volume fields), SPEC-032 (schedules), SPEC-039 (elevation label format:
types, units and label precision; it takes signs from here).
Task type: value display and settings; project units do not change; `.bimc` is extended additively
(project sign setting), `FORMAT` does not change.

> **In short.** One setting is needed: "show units in values" (on by default). When it is on, the
> unit sits to the right of the number in a parameter value (`12.34 m²`), and parameter names have no
> units (`Volume`, not `Volume (m³)`). When it is off, only the number is shown, as lengths do
> today. The setting is common to Properties, family parameters and Family Types.
>
> The second, project setting defines signs: `+` for positives and `±` for zero. By default it is
> off, and every kind of value shows signs as today: spot elevation — `+3.600` and `±0.000`, the
> level label and UI fields — without a sign. When on — one sign for the whole project (`3.600`
> without plus, `±0.000` at zero — the practice of GB/T 50001, DIN, GOST). The setting is stored in
> the project (`.bimc`), because it is printed on the drawing; "show units" stays a user setting in
> `settings.json`.

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
7. **`FORMAT` is not bumped.** `Model.signs` is an additive field (`serde(default)`); old files are
   read, a new key in an old version ends up in `left_out` (`lenient.rs`) rather than a refusal.

## 1. The essence of the problem

| Where | Now | Problem |
|---|---|---|
| Properties, `Area` | `AreaShown`: "12.34 m²" | the unit is there, but only for area |
| Properties, `Volume` | `Shown`: "12.34", name "Volume (m³)" | the unit is in the name, the value has none |
| Properties, lengths | `LengthShown`: "3000" | no unit; the name has none either |
| Properties, angles | `Number`, name "Slope (°)" | the unit is in the name |
| Component parameters | `Holds::Number`: a number | family Area/Volume have no units |
| Family Types | a value without a unit, the name may contain it | inconsistent with Properties |
| Sign of positives | `render::height` (`render.rs:6813`): `+3000` | `+` is hard-coded; UI fields have no sign |
| Zero | `render::height`: `±0` | `±` is hard-coded; the level label has `0`, fields have `0` |
| Level label | `render.rs:3005`: `m.length(...)` | unsigned and cannot take the project sign |

Common cause: `shown` formats every `Kind` in its own way, and the sign is not a setting but a code
branch.

Causes:

- `shown` formats every `Kind` in its own way: `AreaShown` adds `m²` itself, `LengthShown` does not,
  `Shown` returns the number as is.
- `Units` has `length`, `with_unit`, `area`, but no common "value + unit" for volume, angle, mass and
  ratio.
- There is no setting that turns units on or off.
- The sign and zero are hard-coded in `height`, so neither annotations nor fields share one rule;
  project practice (GB/T 50001) requires `±0.000` and `3.600` without `+`.

## 2. Goals

1. One UI setting: **"Show units in values"** (on by default).
2. When the setting is on, all dimensional values show the unit to the right of the number: length,
   area, volume, angle, mass, ratio (kg/m³), possibly scale.
3. When it is off — only the number, in project units, without the symbol.
4. Parameter names never contain units: `Volume`, `Slope`, `Rotation`, `Pattern Angle`, `Azimuth`,
   `Elevation`, `Formwork Area`, `Rebar Weight`, `Rebar Ratio`.
5. Units come from the project (`Model::units`): metric — `mm`, `cm`, `m`, `m²`, `m³`; imperial —
   `in`, `ft`, `ft²`, `ft³`.
6. Editable fields accept a value with and without a unit: `3000`, `3000 mm`, `3.00 m`, `12.5 m²`,
   `30°`.
7. The same in Properties, component parameters and Family Types. Schedule headers — where possible
   too: the unit is either in the header or in the value, not in both places.
8. One project sign setting: **Positive** (`Always` / `Never`) and **Zero** (`Signed` / `Plain`).
9. By default the setting is not set (`signs: null`): signs stay as today for each kind of value —
   spot elevation `+3.600`/`±0.000`, the level and UI fields without a sign.
10. A defined setting applies to all values and labels that read the common formatter: spot
    elevation, the level label, UI fields (if they show a sign).
11. Rounding — before the sign: `-0.0004` at three decimals gives `±0.000`, not `-0.000`.

## 3. Non-goals

- Project units do not change; value conversion does not change.
- `show_units` stays a user setting in `settings.json`. Signs are the opposite: a project setting in
  `.bimc`, because the printed drawing depends on it and two users must see the same.
- Annotation types (label units and precision, symbol, `show_name`) are SPEC-039; here only signs.
- Localization of unit names (`m²` as the Russian `м²`, `ft²` as the Russian `фут²`) — as strings get
  translated; the symbols `m²`, `ft²`, `°` are common.
- Table layout and column widths are not redone, except for room for the unit.
- Do not change the rule that a project file does not store user settings: `signs` is an exception,
  because it is a drawing format, not a convenience.

## 4. Key decision

**One formatter `Units::show(value, kind)` returns a string with or without the unit, and one setting
`show_units` decides whether to add it.**

| Quantity | Value | Metric unit | Imperial |
|---|---|---|---|
| Length | mm | `mm` / `cm` / `m` | `in` / `ft` / `ft-in` |
| Area | mm² | `m²` | `ft²` |
| Volume | mm³ | `m³` | `ft³` |
| Angle | degrees | `°` | `°` |
| Mass | kg | `kg` | `lb` (if it appears) |
| Ratio | kg/m³ | `kg/m³` | `lb/ft³` |

- `Units::length_with_unit`, `area`, `volume`, `angle`, `weight`, `ratio` — existing or new methods;
  `show_units` decides whether to call the variant with the unit.
- The UI does not build strings by hand: `shown(kind, value, units, show_units)` is the single point.
- `parse(kind, text, units)` strips the unit if present and parses the number; otherwise — as now.
- The setting is stored in user settings (`ui-egui`), not in the project; default `on`.

### 4.1. Signs

**The sign is part of the same formatter: `Signs { positive, zero }`, project setting
`Model.signs`.**

| `positive` | `zero` | zero | `3.6` | `-0.45` |
|---|---|---|---|---|
| `always` | `signed` | `±0.000` | `+3.600` | `-0.450` |
| `never` | `signed` | `±0.000` | `3.600` | `-0.450` |
| `always` | `plain` | `0.000` | `+3.600` | `-0.450` |
| `never` | `plain` | `0.000` | `3.600` | `-0.450` |

(example in metres with three decimals; precision is set by the value or the label type, see
SPEC-039)

- `Model.signs: Option<Signs>`; `None` (default) — each kind of value shows signs as today: spot
  elevation `always`/`signed`, level label `never`/`plain`, UI fields unsigned.
- `Some(signs)` — one rule for the whole project: `{"positive": "never", "zero": "signed"}` gives
  GB/T 50001 (`±0.000`, `3.600`, `-0.450`).
- Rounding — before the sign: `-0.0004` m at three decimals — `±0.000`; `-0.0005` — `-0.001`.
- The formatting function is one: `dims::signed(units, decimals, signs, mm)` on top of
  `dims::rounded` (`dims.rs:293`). Do not introduce a second formatter; SPEC-039 calls this same one.
- The setting is changed by the `units {signs: {positive, zero}}` command and reset by `signs: null`;
  it is read in Settings > Units.

## 5. Stage A. Core: units and formatting

- `Units`: add `volume`, `angle`, `weight`, `ratio` (and `length_with_unit` as the common form of
  `with_unit`).
- One quantity type for the UI (`Kind::Length`, `Area`, `Volume`, `Angle`, `Weight`, `Ratio`) instead
  of the current "Number for everything".
- New types `PositiveSign`, `ZeroSign`, `Signs` and `dims::signed(...)`; `Signs` serializes as
  `{"positive": …, "zero": …}`. `Signs::default()` — `Always`/`Signed` (the spot elevation behavior);
  the level label at `signs: null` takes `Never`/`Plain` — this is the annotation's built-in behavior,
  not a different value of the setting.
- `Model.signs: Option<Signs>` (`serde(default, skip_serializing_if)`); `Model::check` ensures the
  values come from the enum.
- The `units` command (`commands.rs:3053`, `:24000`) accepts `signs` and returns it; without the
  field — it does not change anything; `signs: null` — resets.
- No "(m³)" strings in names: all callers remove them.

## 6. Stage B. Settings

- In Settings (section Units) — the **"Show units in values"** toggle, on by default.
- The setting is read by Properties, by component parameters and by Family Types.
- Toggling redraws open panels immediately, without reloading the project.
- The setting value is not written to `.bimc`; every user has their own.
- Next to it — a **"Signs"** block: the **"Same signs throughout the project"** checkbox (off by
  default) and, under it, **Positive** (`Always` / `Never`) and **Zero** (`±0` / `0`). An unchecked
  box writes `signs: null`.
- The checkbox and the two lists are project-level: they are written to `.bimc` and affect the
  drawing. This is the only exception to "UI settings are not in the project" in this SPEC, and it is
  deliberate.
- Changing `signs` redraws open views and panels immediately; Undo — one step, as with any command.

## 7. Stage C. Properties and family parameters

- All dimensional `Kind`s are formatted through `Units::show`: lengths, areas, volumes, angles,
  masses, ratios.
- Editable fields show the unit if the setting is on; `parse` strips it.
- Field names — without units.
- Component parameters (`Holds::Length/Area/Volume/Angle`) are formatted the same way; for area and
  volume it is taken into account that family values are in mm²/mm³ while they are shown in m²/m³.
- Family Types: the value of the selected type — with or without the unit according to the setting;
  the parameter name — without.
- Sign in a field: if `show_units` is on and `signs` is set, a positive value is shown with a sign
  (`+3.600` or `3.600`), zero — `±0.000` or `0.000`; without `signs` the field stays unsigned, as
  today.

## 8. Stage D. Schedules and sheets

- In a schedule the unit is either in the column header (`Volume (m³)`) or in every cell — but not in
  both places.
- When the setting is on — in the cell (`12.34 m³`), the header is `Volume`; when it is off — in the
  header, the cell without the unit.
- CSV/XLSX export: numbers stay numbers, the unit is in the header or in the text, but does not break
  the sum.
- The level schedule (`Elevation` as `Kind::Length`) stays in project units and precision and does
  **not** take the label format from SPEC-039. The sign — per `Model.signs`, like other values.

## 9. Test plans

1. Wall Properties: `Length`, `Thickness`, `Area`, `Openings Area`, `Volume` — with units.
2. Toggling the setting: the same fields without units.
3. `Volume` of a wall — in the Dimensions group, the name is `Volume`.
4. Angles (`Slope`, `Rotation`, `Pattern Angle`) — the value `30°`, the name without `(°)`.
5. A length field accepts `3000`, `3000 mm`, `3.00 m`; it keeps the same value.
6. An area field accepts `12.5 m²`; a volume field — `1.2 m³`.
7. Family Types: Area/Volume/length — with the unit, the name without.
8. Schedule: no double unit (header and cell).
9. Imperial units: `ft²`, `ft³`, `°`; lengths — `ft`/`ft-in`.
10. Saving/opening a project does not depend on the `show_units` setting.
11. `signs: null`: spot elevation — `+3000`/`±0` (mm), the level label — `3000`/`0`, fields —
    without a sign.
12. `signs {positive: never, zero: signed}`, `m`/3: `±0.000`, `3.600`, `-0.450` for spot and level.
13. `signs {positive: always, zero: plain}`: `+0.000`, `+3.600`.
14. Rounding: `-0.0004` m / 3 → `±0.000`; `-0.0005` → `-0.001`.
15. The `units {signs: …}` command writes and returns the setting; `signs: null` resets it; save/open
    keeps it, while `show_units` is not kept.
16. Undo: a `signs` change is cancelled in one step and returns the previous labels.

## 10. i18n and documentation

- Strings: "Show units in values", "Units".
- Sign strings: "Signs", "Same signs throughout the project", "Positive", "Zero", "Always", "Never",
  "±0", "0".
- `docs/areas/desktop.md` — where the setting is and what it changes; separately — that signs are
  project-level.
- `docs/areas/quantities.md` — units of areas and volumes.
- `docs/areas/views-and-graphics.md` — how signs affect elevation and level labels (with a reference
  to SPEC-039).

## 11. Acceptance criteria

- [ ] The "Show units in values" setting exists and is on by default.
- [ ] All dimensional values show the unit on the right.
- [ ] Parameter names have no units.
- [ ] Editable fields accept a value with and without a unit.
- [ ] Properties, component parameters and Family Types are consistent.
- [ ] Schedules have no double unit.
- [ ] The project sign setting exists, defaults to `null`, and is stored in `.bimc`.
- [ ] `Always`/`Never` and `+0`/`0` produce all four combinations; spot elevation and the level label
      use it.
- [ ] Rounding before the sign: `-0.0004` → `±0.000`.
- [ ] `show_units` stays a user setting and is not written to the project.
- [ ] Tests §9, i18n, documentation.

## 12. Risks and questions

| Risk | Answer |
|---|---|
| The user is used to a number without a unit | the setting can be turned off; by decision it is on by default |
| A unit in a field gets in the way of typing | `parse` strips it; input without a unit works too |
| Imperial units | the symbols `in`, `ft`, `ft²`, `ft³`, `ft-in`; the angle is always `°` |
| Schedules and sums | a unit in a cell is text; the number stays a number for the sum |
| Old names with units in a project | parameter names are checked on load and are not changed automatically; new names have no units |
| Two annotations today with different signs | `signs: null` preserves both behaviors; the common setting is turned on deliberately |
| The user expects a sign in fields | fields read the formatter; with `signs: null` they stay unsigned |
| The project setting changes the drawing for everyone | this is the goal: printing is deterministic; the value is visible in Settings > Units |

1. Is a separate setting needed for schedules, or one common to all places?
2. Should the unit be shown for dimensionless values (`Count`, `Ratio`)?
3. Should unit names be localized (`m²` as the Russian `м²`) or should the symbols stay?
4. Are signs needed in UI fields (Properties) or only on the drawing? By default — only where `signs`
   is set; fields without `signs` stay unsigned.
5. Is a one-button "GB/DIN" sign preset needed? For now there are two lists; the preset comes with
   `TechnicalStandard::Gb` (see SPEC-039 §13).

## 13. Definition of Done

- [ ] A single unit formatter.
- [ ] The setting and its persistence.
- [ ] Properties/component parameters/Family Types.
- [ ] Schedules without double units.
- [ ] `Signs`, `Model.signs`, the `units {signs}` command and the UI block; the default preserves the
      current appearance.
- [ ] Tests §9 (including 11–16), i18n, documentation.
