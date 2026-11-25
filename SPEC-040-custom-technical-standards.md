# SPEC-040. Custom technical standards: the built-in preset is read-only, your own copy is editable

Status: **draft**, implementation required
Date: 2026-11-25
Project: BIMCraft 032 (Rust; `crates/core/src/standards.rs` — `TechnicalStandard`, `SheetMargins`,
`DimensionStyle`, `line_weights`, `object_styles`, `sheet_margins`; `crates/core/src/model.rs` —
`standard` :4652, `standard()` :4765, `apply_standard` :4793, `line_weights()` :4819,
`object_style`, `Model::check`; `crates/core/src/commands.rs` — `standard` :3108/:23762,
`line_weights` :3009/:24703, `object_styles` :3035/:24737, `element_json` :5149;
`crates/core/src/sheet.rs` :195, `crates/core/src/model.rs` :11477 — sheet fields from the standard;
`crates/core/src/library.rs` — `Kind`; `crates/core/src/lenient.rs`;
`crates/ui-egui/src/standards.rs` — the Technical Standards dialog; `crates/ui-egui/src/lineweights.rs`,
`crates/ui-egui/src/objectstyles.rs`; `crates/ui-egui/src/properties.rs` — `edit_type` :1802;
`crates/core/src/elevation.rs` and `crates/ui-egui/src/leveltypes.rs` — SPEC-039;
`docs/areas/views-and-graphics.md`, `docs/areas/desktop.md`; tests `crates/core/src/tests.rs`,
`crates/ui-egui/src/tests/standards.rs`)
Source: the decisions of 2026-11-25 on discussion #616 and the code review. Today `TechnicalStandard` is
a closed enum of five presets; its properties are not editable, a standard of one's own based on an
existing one cannot be created, and some properties (projection, dimension style) are only shown in
the window.
Related: SPEC-036 (units and signs), SPEC-039 (spot elevation and level label types), discussion #616.
Task type: technical standards and the edit lock; `.bimc` is extended additively, `FORMAT` does not
change.

> **In short.** The built-in technical standard (NBR, ISO, ANSI, EU, JIS) becomes a **read-only
> preset**: its pens, object styles, sheet margins/titleblock and head formats cannot be edited in the
> project. If the user wants to change something, they make **their own copy** based on any built-in
> or own standard (`Duplicate…`), and inside the copy absolutely everything is editable. Own standards
> are project library records (GUID), carried with the project; the copy is created from the current
> state of the project, so the drawing does not change. Project units are not locked. Applying a
> standard copies its values into the project according to the `adapt_*` flags;
> `adapt_annotation_types` is on by default. Projection and dimension style are descriptive in this
> version — applying them is a separate stage.

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
7. **`FORMAT` is not bumped.** New fields and maps are additive (`serde(default)`), old files are
   read, new keys in an old version end up in `left_out` (`lenient.rs`) rather than a refusal.
8. `.md` files in the project follow the project language.

## 1. The essence of the problem

Today `TechnicalStandard` (`standards.rs:26`) is a closed enum. All its data is hard-coded in a
`match`; the `standard` command accepts only `nbr|iso|ansi|eu|jis`; the Technical Standards window
only shows tabs and applies a preset.

| Property | Applied? | Where | Editable? |
|---|---|---|---|
| Line weights | yes, by copy | `apply_standard` → `Model.line_weights` | yes, `line_weights` |
| Object styles | yes, by copy | `apply_standard` → `Model.object_styles` | yes, `object_styles` |
| Units | per flag | `apply_standard` → `Model.units` | yes, `units` |
| Sheet margins, titleblock, folding | yes, at drawing time | `sheet.rs:195`, `model.rs:11477` | **no** |
| Projection method | **no** | the command response and the window only (`commands.rs:23788`) | no |
| Dimension style (tick, leaders, text) | **no** | the window only; dimensions live in `Model.dimension_types` | no |
| Level marker | **no** | text in the window only | no (becomes real in SPEC-039) |

Consequences:

1. **The preset cannot be corrected.** The preset numbers, by the command's own admission
   (`commands.rs:3114–3115`), "came with the contribution that added them and have not been verified
   against the standards". If a firm knows that its values differ, it cannot change them within the
   standard.
2. **The label can lie.** A project can be called "NBR" while its pens and styles are different: they
   live in the project separately and are not tied to the preset in any way.
3. **An own standard cannot be expressed.** An office standard cannot be saved, cannot be spread
   across projects, cannot be handed to a colleague.
4. **Half the window is decoration.** Projection and dimension style look like standard properties
   but affect nothing.
5. **It is unclear what can be changed.** The window looks like a form, but everything in it is
   read-only; Line Weights and Object Styles, on the contrary, are editable and silently diverge from
   the standard.

## 2. Goals

1. The built-in standard is **read-only**: its pens, object styles, margins/titleblock and head
   formats cannot be changed in the project.
2. **An own copy** based on any built-in or own standard; inside it everything is editable: name,
   description, region, references, projection, default units, margins/titleblock, pens, styles,
   dimension style, `LevelType`/`SpotElevationType` formats.
3. **Duplicate** creates a copy from the current state of the project (pens, styles, types), so after
   switching to the own copy the drawing does not change.
4. Own standards are project library records (`Kind::Standard`, GUID); they are carried with the
   project, the template and Insert from File.
5. The standard owns the conventions it applies: pens, object styles, margins/titleblock, head formats
   (SPEC-039).
6. The lock is in the core, not only in the window: CLI, scripts and MCP get the same refusals as the
   GUI.
7. Project units are **not locked**: the standard provides `default_units`, the project chooses for
   itself.
8. Applying a standard copies values into the project according to the `adapt_line_weights`,
   `adapt_object_styles`, `adapt_units`, `adapt_annotation_types` flags; `adapt_all` enables all of
   them; `adapt_annotation_types` is on by default.
9. A project without a standard (`standard: null`) works as today: pens and styles are project-level
   and editable, sheet margins are ISO by default.
10. Migration does not change the drawing: an old project opens as is, and a divergence is shown, not
    silently corrected.
11. Projection and dimension style stay descriptive in this version; applying them is a separate
    stage.
12. Tests, i18n, documentation; old files and the API are read.

## 3. Non-goals

- Do not lock model elements (walls, doors, floors): that is not a property of the standard.
- Do not lock project units.
- Do not apply projection to the placement of views on sheets in this version (stage 2).
- Do not turn `DimensionStyle` into a generator of `Model.dimension_types` in this version (stage 2).
- Do not recompute or verify the numbers of the built-in presets against the standards: that is
  separate work with sources; for now they stay as they are.
- Do not add `TechnicalStandard::Gb` to the code: GB practice is assembled as an own standard based
  on ISO (SPEC-039 §13.3); a built-in preset — later, if needed.
- Do not make a global library of the user's standards (`settings.json`): in this version a standard
  lives in the project. File export/import is the next step (§13, question 6).
- Do not change building codes (`codes.rs`) — that is another subsystem.
- Do not change the elevation label format: its types and format are SPEC-039; here only ownership
  and the lock.

## 4. Key decision

**A standard is a complete definition (`StandardDefinition`). A built-in one is served from the code
read-only; an own one is stored in the project and editable. Applying materializes values into the
project according to flags; the lock forbids edits under a built-in one.**

```rust
pub struct StandardDefinition {
    pub name: String,
    pub base: Option<StandardBase>,       // what it is based on, display only
    pub region: String,
    pub governing_body: String,
    pub references: Vec<String>,
    pub description: String,
    pub projection: ProjectionMethod,     // descriptive in this version
    pub default_units: Units,
    pub sheets: BTreeMap<String, SheetMargins>, // A0…A4 and custom
    pub line_weights: LineWeights,
    pub object_styles: BTreeMap<String, ObjectStyle>,
    pub dimension_style: DimensionStyle,  // descriptive in this version
    pub level_heads: BTreeMap<String, LevelType>,        // SPEC-039
    pub spot_elevations: BTreeMap<String, SpotElevationType>, // SPEC-039
}
```

- `TechnicalStandard` and its methods stay; `TechnicalStandard::definition()` assembles a
  `StandardDefinition` once (`OnceLock`) and returns it by reference. No changes to the preset data.
- Own standards: `Model.standards: BTreeMap<String, StandardDefinition>` — library records
  (`Kind::Standard`), a name in memory, a GUID in the file.
- Active one: `Model.standard: Option<StandardRef>`:
  - `StandardRef::Builtin(TechnicalStandard)` — in the file the string `"nbr"`, as today;
  - `StandardRef::Custom(String)` — in the file `{"custom": "<GUID>"}`; in memory the name, like other
    library references (`Model::lib_refs_mut`).
- **Materialization.** Applying a standard copies its values into project fields, as today:
  `Model.line_weights`, `Model.object_styles`, `level_types`, `spot_elevation_types`. Margins and the
  titleblock are not copied — they are computed from the active definition
  (`Model::standard_margins(size)`).
  - Reason: the drawing path is not redone, old files continue to open and draw.
  - A divergence of project values from the definition is not silently corrected; it is computed and
    shown (`differs`, §6.1).
- **Editing an own standard writes both ways:** the command changes the project field (the drawing
  sees it at once) and the standard definition (so that switching the standard and back preserves the
  values). Type names of a standard are its own: the copy takes the base's canonical names and reuses
  an existing type with the same name instead of creating a duplicate.
- **Lock.** `Model::standard_locked()` is true when a built-in standard is active. Under it:
  - `line_weights` (including `reset`) — refusal;
  - `object_styles` — refusal for the categories the standard defines; categories outside the
    definition stay project-level and editable (§6.2);
  - `level_type`, `spot_elevation_type` — refusal on create/modify/rename/delete; listing and type
    selection work;
  - `units` — allowed;
  - `custom_standard` — allowed (this is the way out);
  - `standard {set}` — allowed; the standard's fields are not editable, only selection and Apply.
- **Duplicate** (`custom_standard {like, from_project: true}`) copies the definition and replaces its
  pens, styles and types with the current project values: after switching, the drawing is identical.

### 4.1. Decisions taken

| Question | Decision |
|---|---|
| Lock units? | no: `default_units` is a preset hint, the project chooses for itself |
| What to lock? | only what is applied: pens, styles, margins/titleblock, head formats |
| Projection, dimension style | descriptive in this version; applying — stage 2 |
| Head formats | option B: the standard owns copies of `LevelType`/`SpotElevationType` |
| What to lock in types | the whole type (create/modify/rename/delete), otherwise one could bypass it with a new type |
| Where head values come from | from the standard definition; Apply materializes them into the project |
| `adapt_annotation_types` | **on** by default, included in `adapt_all` |
| Where own standards live | in the project, a library record with a GUID; carried with the file |
| Migration | the drawing does not change; a divergence is shown and resolved by Duplicate/Apply |

## 5. Stage A. Model and library

- `StandardDefinition` (`crates/core/src/standards.rs` or `standards/definition.rs`):
  - fields from §4; `serde(deny_unknown_fields)`;
  - `StandardBase { builtin: TechnicalStandard } | { custom: String }` — only for showing
    "Based on …", does not affect behavior;
  - `check(&self)`: the name and texts within `MAX_TEXT`, without control characters; `references` —
    up to 32 lines; `sheets` — known formats plus custom ones, margins non-negative and no larger than
    the sheet; `line_weights.check()`; category styles through `ObjectStyle::check`; `level_heads` and
    `spot_elevations` pass `check` from SPEC-039; the total size is limited.
- Built-in ones:
  - `TechnicalStandard::definition()` assembles a `StandardDefinition` from the existing methods; the
    data does not change;
  - `definition()` is cached (`OnceLock`/`LazyLock`) and returned by reference.
- `Model`:
  - `#[serde(default, skip_serializing_if = "BTreeMap::is_empty")] pub standards: BTreeMap<String, StandardDefinition>`;
  - `#[serde(default, skip_serializing_if = "Option::is_none")] pub standard: Option<StandardRef>`;
  - `StandardRef` serializes as a string for a built-in one and as `{"custom": …}` for an own one; the
    old `"standard": "nbr"` is read without migration;
  - `Model::standard_definition() -> Cow<'_, StandardDefinition>` — a built-in one from the cache, an
    own one from the map, `None` — ISO (as today's `standard()`);
  - `Model::standard_locked() -> bool` — a built-in one is active;
  - `Model::standard_margins(size) -> SheetMargins` — from the active definition; `sheet.rs:195` and
    `model.rs:11477` are switched to it;
  - `Model::standard_code() -> Option<&str>` for responses (`"nbr"` or the own name).
- `apply_standard(std, line_weights, object_styles, units, annotation_types)`:
  - `line_weights` — `std.definition().line_weights.clone()` into `Model.line_weights`;
  - `object_styles` — insert the definition's styles into `Model.object_styles` (as today);
  - `units` — as today (ANSI ↔ imperial, `decimals` reset when units change);
  - `annotation_types` — insert/update `level_heads` and `spot_elevations` in
    `Model.level_types`/`spot_elevation_types`; do not touch types of other standards.
  - `adapt_all` enables all four.
- Library (`library.rs`):
  - `Kind::Standard`, `add(...)`, `copy_definition` for Insert from File;
  - `Model::lib_refs_mut` includes `Model.standard` and the names inside `StandardDefinition`
    (`base.custom`, standard keys/types) — renaming a standard updates the references;
  - `MAX_STANDARDS` (e.g. 100), a GUID on every record.
- `Model::check`:
  - the standards map is no larger than `MAX_STANDARDS`; each passes `check`; the name does not match
    a built-in code (`nbr|iso|ansi|eu|jis`);
  - the active reference resolves; a deleted standard — an error;
  - types from `level_heads`/`spot_elevations` exist in the project for the **active** standard (Apply
    and `custom_standard` create them); for an inactive own standard they may be missing until Apply;
  - `sheets` contains A0–A4: built-in standards define them, an own one inherits them from its base and
    can edit them.
- `lenient.rs`: new keys are read by the common walk; no special changes are needed.

## 6. Stage B. Lock

### 6.1. Rule and messages

- `Model::standard_locked()` — a built-in one is active.
- The check is at the beginning of a command, before reading/writing:
  - `line_weights`, including `reset: true` — refusal;
  - `object_styles`, including `reset: true` — refusal if the command touches categories of the
    definition;
  - `level_type`, `spot_elevation_type` — refusal on any write (create/modify/rename/delete);
  - `units` — not checked;
  - `custom_standard` — not checked;
  - `standard {set}` — not checked (this is selection and Apply).
- The error text names the standard and the way out:
  ``the project follows the built-in standard `NBR`: its line weights are read-only. Make your own copy: `custom_standard {name: "NBR — office", like: "nbr", from_project: true, set: true}` ``.
- For types: ``the built-in standard `NBR` owns the level heads and spot elevations: change them in your own copy of the standard``.
- A read-only session (`Session::read_only`) is checked earlier or later — the result is the same: a
  refusal.

### 6.2. The boundaries of the lock

- Object styles: the standard defines a set of categories. Categories that are not in the definition
  (rare ones or ones added by the user) are not a property of the standard and stay editable; the
  window marks them "Project", and their edits are written only to the project, not to the
  definition.
- Line weights: the whole table is locked, because the standard defines it entirely (model and
  annotation pens).
- Margins/titleblock: there is no project override — only the standard definition; under a built-in
  one the window only reads.
- Types: under a built-in one **any** write to the `level_types` and `spot_elevation_types` maps is
  locked, including types left over from a previous own standard. They can still be selected and
  used.
- Units: always allowed.
- `pinned`/`read-only` of the model are not related to the lock.

### 6.3. Divergence

- `Model::standard_differs() -> Differs { line_weights: bool, object_styles: bool, annotation_types: bool }`:
  compares project values with the active standard's definition — the whole pen table, the
  definition's categories and the types with the standard's names; categories and types outside the
  definition are not compared.
- Opening a file does not correct a divergence. The Technical Standards window and the `standard`
  response show it; two actions are offered:
  - **Save as my standard** — `custom_standard {like: <active>, from_project: true, set: true}`;
  - **Adopt <standard>** — Apply with the needed `adapt_*` (all on by default).
- `Model::check` does not treat a divergence as an error: a hand-edited file must open.

## 7. Stage C. Commands

- `custom_standard`:
  - without `name` — `{standards: [{name, base, custom: true}, …], builtins: [...], active}`;
  - creation: `{name, like, from_project?, set?, …fields}`; `like` — a built-in code or an own name;
    missing fields — from `like`; `from_project: true` replaces pens, styles and types with the current
    project values;
  - edit: `{name, …fields}`; every field — as in `StandardDefinition`;
  - `{name, rename: "…"}`; `{name, delete}` — an active standard or one used by a type cannot be
    deleted (an error with a list);
  - `set: true` switches the project at once (the UI uses this for Duplicate).
- `standard` (`:23762`):
  - `{set: "nbr"|"iso"|…|"<own name>", adapt_all?, adapt_line_weights?, adapt_object_styles?,
    adapt_units?, adapt_annotation_types?}`;
  - without `set` — a report: `{standard, name, custom, based_on, locked, differs, projection,
    default_units, default_sheet_size, all: [...]}`; `all` includes built-in and own ones;
  - ambiguity (an own name matching a code) is forbidden by `Model::check`.
- `line_weights`, `object_styles`, `level_type`, `spot_elevation_type` — the same commands; with an
  active own standard their write also updates the definition (§4), with a built-in one — a refusal
  (§6).
- `units` — unchanged; the standard only provides `default_units` on Apply.
- Type responses (`level_type`, `spot_elevation_type`) are extended with `owned: true|false` — whether
  the type belongs to the active standard (for the lock and the window).
- Read-only: modifying commands refuse automatically.
- `hostile_params_never_panic_or_break_the_model`: payloads for `custom_standard` (name, `like`,
  fields, rename/delete, from_project) and the new fields of `standard`.

## 8. Stage D. Interface

- The Manage > Technical Standards dialog (`crates/ui-egui/src/standards.rs`):
  - selector: built-in and own standards; own ones are marked "Custom", the active one — "Active";
  - **built-in active**: tabs are read-only, the badge `Built-in — read-only`, the **Duplicate…**
    button; at the bottom — `Apply`/`OK`/`Cancel` and the `Adapt …` checkboxes;
  - **own active**: all tab fields are editable: Overview (name, region, governing body, references,
    description), Line Weights, Sheets & Margins, Object Styles, Dimensions & Annotations; the
    buttons `Save`, `Delete`, `Rename`;
  - a divergence — a banner with `Save as my standard` and `Adopt <standard>`;
  - a preview of elevation values (`±0.000`, `3.600`) on the Dimensions & Annotations tab.
- **Duplicate…**: a modal window — name (default `<standard> — copy`), base (the current standard), the
  **Start from the project's current settings** checkbox (on); it creates and switches.
- Line Weights (`lineweights.rs`) and Object Styles (`objectstyles.rs`): under a built-in one — fields
  are read-only, with a banner and the `Duplicate…` button; under an own one — editable as today and
  written to the standard.
- Type Properties (`level_type`, `spot_elevation_type`): under a built-in one — read-only, locked, with
  `Duplicate…`; under an own one — as in SPEC-039.
- Units (Settings > Units): without a lock; a hint that the standard provides the default value.
- One Undo step per creating/editing/switching a standard; the selection is not reset.
- i18n: all new strings through `msg!`/`t()`.

## 9. Compatibility and migration

- The old `"standard": "nbr"` is read as `StandardRef::Builtin`; the new keys `standards` and
  `StandardRef::Custom` are additive.
- `FORMAT` does not change. A new-kind file opened by an old version: a `standard` of an unknown shape
  and `standards` end up in `left_out`; the project opens, but without its own standard (ISO is
  shown) — documented.
- An old project where a standard is selected but pens/styles were edited: opens without changes;
  `differs` is true, the edits are locked, Duplicate or Adopt is offered.
- A project without a standard (`null`): the behavior is as before; sheet margins — ISO, as today
  (`Model::standard()` returned ISO).
- `.bimt` templates and Insert from File carry `Model.standards` and the active reference.
- IFC and schedules do not change; sheet margins come from the active definition, so with an own
  standard they can differ — that is the goal.
- `standard` in the `info` response shows the built-in code or the own name; `custom: true` is added.

## 10. Test plans

1. A built-in one is active: `line_weights` (write and `reset`) — refusal; the text contains the
   standard's name and `custom_standard`.
2. A built-in one is active: `object_styles` by a category of the definition — refusal; by a category
   outside the definition — allowed; `reset` — refusal.
3. A built-in one is active: `level_type`/`spot_elevation_type` — refusal on create/modify/rename/delete;
   the list and `owned` are available.
4. A built-in one is active: `units` — allowed; `standard {set}` — allowed.
5. An own standard: `custom_standard {like: "iso"}`; editing pens/styles/types changes the project and
   is kept in the definition; switching to a built-in one and back restores them.
6. Duplicate `from_project: true`: the project values match before and after; the sheet drawing matches
   pixel for pixel.
7. Apply: `adapt_annotation_types: true` inserts the heads from the standard; `false` leaves the
   project ones; the default is true; `adapt_all` includes it.
8. Sheet margins: with an own standard with different margins the titleblock is drawn by them; with a
   built-in one — by the preset; there is no project override.
9. Divergence: a file with a built-in standard and edited pens opens without changes,
   `differs.line_weights` is true, the edits are locked.
10. Library: renaming the active standard updates the reference and the types; deleting the active one —
    an error; deleting another — success; `MAX_STANDARDS`; `Model::check` catches a name matching a
    code.
11. Serde: the old `"standard": "nbr"`; an own standard is saved and opened; a template carries it; an
    old version leaves `left_out`.
12. UI: under a built-in one the fields are read-only and Duplicate is present; under an own one they
    are editable; the divergence banner; Type Properties and Line Weights lead to Duplicate.
13. `i18n check`; `hostile_params_never_panic_or_break_the_model` with the new payloads.

> The tests in this SPEC are described but are not run without a separate request; CI — separately.

## 11. i18n and documentation

- New strings (English msgids):
  - `Built-in — read-only`, `Custom`, `Active`, `Based on`, `Duplicate…`, `Rename`, `Delete`,
    `Save`, `Save as my standard`, `Adopt …`, `Start from the project's current settings`
  - `Custom Standard`, `Standard name`, `Region`, `Governing body`, `References`, `Description`
  - `This project differs from …`, `Project` (for a category outside the standard)
  - `The built-in standard is read-only: make a copy to change it`
- Update `translations/ui/bimcraft-ui.pot` and all `.po` files; `i18n check` is green.
- `docs/areas/views-and-graphics.md`: technical standards — the preset is read-only, own standards,
  margins/titleblock, what is applied and what is descriptive (projection, dimension style).
- `docs/areas/desktop.md`: the Technical Standards window, Duplicate, the lock in Line
  Weights/Object Styles and Type Properties; units are not locked.
- `crates/core/CLAUDE.md`: the `standards/definition.rs` module line, if the module is split; otherwise
  extend the `standards.rs` line.
- `crates/ui-egui/CLAUDE.md`: no new module if the dialog stays in `standards.rs`.
- The params docs of the `standard` and `custom_standard` commands — in `ALL` (`commands.rs`); do not
  create `docs/commands.md`.
- State honestly in the documentation: the numbers of the built-in presets have not been verified
  against the standards; an own standard allows setting exact values.

## 12. Acceptance criteria

- [ ] The built-in standard is read-only: pens, object styles (categories of the definition),
      margins/titleblock and head formats are not editable; the refusal comes from the core, also for
      MCP/CLI.
- [ ] Project units are not locked.
- [ ] `custom_standard` creates an own standard based on a built-in or own one, with all fields.
- [ ] In an own standard everything is editable, including `LevelType`/`SpotElevationType`; changes are
      kept in the definition and in the project.
- [ ] Duplicate from the current project does not change the drawing.
- [ ] `adapt_annotation_types` is on by default and included in `adapt_all`.
- [ ] Margins/titleblock come from the active definition; with an own standard they are its own.
- [ ] Own standards are library records with a GUID; renaming, deleting, carrying with a template.
- [ ] An old project opens without changing the drawing; a divergence is shown; migration is
      non-destructive.
- [ ] `FORMAT` did not change; old `"standard": "nbr"` is read.
- [ ] Projection and dimension style are explicitly marked descriptive; applying them is not promised.
- [ ] Tests §10, i18n, documentation; `check`, `build`, `clippy`, `fmt --check`, `i18n check` — green.

## 13. Risks and questions

| Risk | Answer |
|---|---|
| Project copies diverge from the definition | `differs` computes and shows it; opening does not correct; Duplicate/Adopt resolve it |
| Apply overwrites pens/styles/types | that is its purpose; the `adapt_*` flags allow not overwriting; the window warns |
| A category outside the standard definition | stays project-level and editable, marked "Project" |
| The numbers of the built-in presets are not verified | an own standard allows setting correct values; the presets are not recomputed |
| Projection and dimension style are descriptive | explicitly marked; applying — stage 2 |
| An own standard is invisible to an old version | `left_out`; the project opens, the standard is lost — documented |
| Deleting the active standard | forbidden; switch first |
| An own standard's name matches a code | forbidden by `Model::check` |
| Deleting a type that belongs to an own standard | allowed; the type is removed from the definition too |
| Two standards own one type name | Apply updates by name; the last applied one sets the values; the window shows the owner |

1. **Materialization versus computing on the fly.** The SPEC chooses copies in the project (as today),
   because the drawing path is not redone and old files are preserved. The alternative — reading
   everything from the definition at drawing time — is cleaner, but changes the drawing path and the
   migration; rejected.
2. **Should creating new types be locked under a built-in standard?** Yes: otherwise one could bypass
   the lock by creating a type with different values. Under an own standard creation is allowed.
3. **Categories outside the definition.** We do not lock them: that is not a property of the standard.
   If a complete lock of the whole style map is needed — change the rule in §6.2.
4. **`default_units` and `decimals`.** Units are not locked; when units change, `decimals` is reset, as
   today.
5. **`dimension_style`.** For now it is only a description. Turning it into a generator of
   `dimension_types` is a separate task (stage 2), otherwise the SPEC keeps growing.
6. **`.bimstd` export/import.** In this version a standard lives in the project. An interchange file is
   the next step; without it a GB standard is transferred together with a project or a template.
7. **Auto-duplicate on an edit attempt.** We do not do it: a refusal with text and a Duplicate button.
   Silently creating a standard is unexpected for the user.

## 14. Definition of Done

- [ ] `StandardDefinition`, `StandardRef`, `Model.standards`, the cache of built-in definitions.
- [ ] `apply_standard` with four flags; `adapt_annotation_types` is on by default.
- [ ] `Model::standard_locked`, `standard_differs`, `standard_margins`; `sheet.rs`/`titleblock` on them.
- [ ] The lock in `line_weights`, `object_styles`, `level_type`, `spot_elevation_type`; texts with the
      way out.
- [ ] `custom_standard` (create/edit/rename/delete/set), `standard {set}` with an own standard's name.
- [ ] Library: `Kind::Standard`, GUID, renaming, deleting, `MAX_STANDARDS`, `Model::check`.
- [ ] UI: built-in and own modes, Duplicate, the divergence banner, locks in the pen, style and Type
      Properties windows.
- [ ] Migration without changing the drawing; the old `"standard": "nbr"`; templates.
- [ ] i18n, `docs/areas/views-and-graphics.md`, `docs/areas/desktop.md`, module maps.
- [ ] Tests §10; `check`, `build`, `clippy`, `fmt --check`, `i18n check` — green.
