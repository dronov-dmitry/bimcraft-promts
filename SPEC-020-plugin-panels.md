# SPEC-020. Plugin panels: declarative panels and input forms (ABI 2)

Status: **draft**, implementation required
Date: 2026-11-18
Project: BIMCraft (Rust, `crates/plugin`, `crates/plugin-sdk`, `crates/ui-egui`, egui 0.36, wasmi 2.0)
Source: user request of 2026-11-18: "can plugins create their own panels, like in Revit?"
Related spec: `SPEC-019-panel-visibility-presets.md` (panel visibility and presets).
PR order: **second**, after SPEC-019.
Task type: extending the ABI to version 2 and the window; the model, the project file, and the commands
do not change, ABI 1 keeps working.

> **In short.** A plugin already declares a tab and ribbon panels, but a button runs a function without
> parameters, and the only way a plugin can show something to a person is a text or Markdown report in a
> dialog. We need a plugin to declare a dockable panel with fields, buttons, text, and a table, and the
> host to draw it itself, and the input form to be built from the already existing `params` schema. The
> interface is still drawn by the host: the ABI stays command-based, the sandbox and the limits are not
> extended, and ABI 1 plugins work without a rebuild. Reactivity on model events is not part of this
> iteration.

## 0. General working rules

1. The language of communication is Russian; code, comments, strings in code, compiler and log messages —
   English. Captions, texts, and panel names of a plugin are its data and are not translated; host user
   strings go through `msg!`/`t()`, and the catalogs are updated (`cargo i18n update`).
2. Tests (`cargo test`, `cargo nextest`, doctest) are **not run** in the task without the user's direct
   request (rule `~/.config/opencode/rust-rules.md`). Allowed: `cargo check`, `cargo build`,
   `cargo clippy` (without `--all-targets`), `cargo fmt --check`, `cargo i18n check`; tests — on CI and
   by the user.
3. Commits are atomic: the form from `params` separately, the manifest and panel model separately, the
   window and docking separately, panel responses separately, documentation and translations separately.
   Each commit builds, and ABI 1 plugins keep working.
4. ABI 1 does not change: old manifests are read as before, `panels` — only with `abi: 2`; the guest
   gets no new imports.
5. The sandbox and limits are not extended: the guest has no network, files, clock, or random numbers;
   the `files` permission is still granted only by the user.
6. No new dependencies; `unsafe` remains only in `crates/plugin-sdk` (the guest ABI boundary), and panics
   are forbidden by lints.

## 1. The essence of the problem: what exists today

A plugin is a third-party `.wasm` in a sandbox that talks to the application only through commands
(`Session::execute`).

What already works:

- **The manifest declares the ribbon.** Each function has `tab`, `panel`, `label`, `order`
  (`crates/plugin/src/manifest.rs:57-77`). The application assembles tabs and panels from them
  (`crates/plugin/src/lib.rs:466-505`), and buttons are sorted by `order`.
- **Tabs and panels are grouped by name, not by plugin.** Two plugins declaring
  `tab: "Areas"`, `panel: "Rooms"` will land in the same tab and the same panel (`lib.rs:482-491`).
  So a panel's identity is the pair (tab name, panel name).
- **A button runs a function without parameters** and opens the `report` response in a dialog
  (`crates/ui-egui/src/plugins.rs:241-257`, `337-378`).
- **The only exception** is a mandatory `path`/`file` in `params`: then a system file picker opens
  (`plugins.rs:266-315`).
- **The plugin does not draw the interface** — this is the fixed ABI 1 boundary
  (`docs/areas/plugins.md:310-316`).
- **A parameter schema already exists** in the manifest, but it is used only for the agent catalog and
  for finding the mandatory `path`.

Two pains follow from this:

1. A button without parameters is useless for almost any real operation ("make a summary by level" —
   but which level?).
2. A panel is just a strip of buttons. There is no way to show a list, a table, a total, or a state.

## 2. Goals

1. Give a plugin an **input form**: the application draws fields from `functions[].params` itself and
   passes the values into the call. Without changing the ABI (this is a behavior extension, not a format
   change).
2. Give a plugin **its own palette panel** — a dockable window with a declared set of fields, buttons,
   text, and a table, like the dockable pane of a Revit add-in.
3. Preserve all ABI 1 properties: sandbox, limits, no UI code in the guest, no `unsafe`, one call — one
   undo step.

## 3. Non-goals

- The plugin does **not** draw arbitrary interface (no immediate-mode `ui.*` calls from WASM).
- No HTML/webview/JS panels.
- No network and no loading panels from the internet.
- No calls to functions of other plugins from a panel.
- No "after every model command" reactivity in this iteration (see §9, stage C).
- No storing panel values in the project file (the rule "no plugin code or data in the project").
- No custom panel icons (the icon is shared, `Icon::Code`).

## 4. Key decision and its rationale

**A panel is described declaratively (a widget tree in the manifest + a JSON response), and the host
draws it itself.**

Why not "the plugin calls egui itself":

| Reason | Consequence |
|---|---|
| A guest call is limited to 5 s and 20M fuel and **terminates** (`docs/areas/plugins.md:105-121`) | immediate-mode UI requires a call per frame and state between frames — the ABI cannot do that |
| Would have to export dozens of imports (`ui.label`, `ui.text_edit`, painter, layout, popup) | A huge ABI surface that cannot be changed without breaking plugins |
| egui 0.36 changes between versions | Any BIMCraft update would break panels |
| Theme, DPI, translations, accessibility, width fitting, docking | All of this already exists in the host; it cannot be duplicated in every plugin |

A declarative panel preserves the rule "the plugin does not draw the interface" and still delivers 90% of
the value of a dockable pane: fields, buttons, a table, refresh on demand.

## 5. Stage A. A form from `params` (without ABI 2)

### 5.1. Behavior

Clicking a plugin button (in the ribbon, in the Tools palette, in Find) with non-empty `params` opens a
small dialog with fields; the "Run" button runs the function with the entered values, "Cancel" does
nothing.

If `params` is empty or absent — the behavior does not change: a one-click run.

### 5.2. Supported subset of JSON Schema

| In the schema | Widget |
|---|---|
| `"type": "boolean"` | checkbox |
| `"type": "integer"` | numeric field, integer |
| `"type": "number"` | numeric field |
| `"type": "string"` | text field |
| `"enum": [...]` (with `type: string` or `integer`) | dropdown list |
| `"type": "array"`, `"items": {"type": "string"}` | list of strings (no UI in v1 — rejected, see 5.4) |
| `"default"` | initial value |
| `"minimum"`, `"maximum"` | numeric field bounds |
| `"title"` | field caption |
| `"description"` | hover hint |
| `"required": [...]` | mandatory fields; an empty mandatory one — an error at the field |

BIMCraft extensions (the `x-bimcraft` namespace, to avoid conflicting with the standard):

```json
{
  "type": "object",
  "properties": {
    "level":   {"type": "string", "x-bimcraft": {"kind": "level"}},
    "width":   {"type": "number", "x-bimcraft": {"kind": "length"}},
    "wall":    {"type": "string", "x-bimcraft": {"kind": "library:type:wall"}},
    "ids":     {"type": "array",  "x-bimcraft": {"kind": "selection"}},
    "path":    {"type": "string", "x-bimcraft": {"kind": "path"}}
  },
  "required": ["level"]
}
```

- `level` — a dropdown list of project levels.
- `length` — input in project units; a number in mm is passed into the call (`Model::units`, as in
  PyAddins).
- `library:<kind>` — a list of library names of the required kind.
- `selection` — the identifiers of the selection; a read-only list.
- `path` — a path chosen by the system dialog; requires the `files` permission (as now).

### 5.3. Implementation

- A new module `crates/ui-egui/src/plugin_form.rs` (or an extension of `plugins.rs`):
  `PluginForm { fields: Vec<PluginField>, values: Map<String, Value>, error: Option<String> }`,
  schema parsing — `PluginForm::from_schema(&Function) -> Result<PluginForm, String>`.
- Field drawing — following the PyAddins form (`crates/ui-egui/src/pyaddins.rs:631-720`): caption,
  accessibility row via `labelled_by`, hint via `on_hover_text`, `ComboBox` combo boxes.
- Validation before the call — `PluginForm::resolve()`; the error message names the field.
- Values live in window memory (`BimApp::plugin_forms: BTreeMap<String, PluginForm>` by command id),
  kept between calls until the window closes; not written to `settings.json`.
- The call — the same `finish_plugin_function` (`plugins.rs:317-332`), one undo step.

### 5.4. Schema errors

A schema the host does not understand (`type: object` inside a property, `array` without an extension,
`oneOf`, `$ref`, etc.) **does not break the plugin**: such a field is shown as a text field with the
message "Unsupported field: <json>", and if the field is mandatory, the call is rejected with a clear
error. This matters: `params` is free-form JSON, and forbidding it entirely would be a regression.

## 6. Stage B. A declarative panel (ABI 2)

### 6.1. Manifest

```json
{
  "id": "rooms",
  "version": "0.2.0",
  "name": "Rooms",
  "description": "Room areas and totals.",
  "abi": 2,
  "module": "rooms.wasm",
  "permissions": [],
  "panels": [
    {
      "id": "areas",
      "title": "Area Summary",
      "order": 10,
      "dock": "right",
      "on_open": "summary",
      "refresh": "summary",
      "widgets": [
        {"type": "label", "text": "Totals by level"},
        {"type": "level", "id": "level", "label": "Level"},
        {"type": "number", "id": "min_area", "label": "Min area", "unit": "area", "min": 0},
        {"type": "checkbox", "id": "outer", "label": "Outer walls only"},
        {"type": "select", "id": "mode", "label": "Mode",
         "options": [{"value": "sum", "label": "Sum"}, {"value": "list", "label": "List"}]},
        {"type": "library", "id": "wall_type", "label": "Wall type", "kind": "type:wall"},
        {"type": "selection", "id": "ids", "label": "Elements"},
        {"type": "button", "id": "run", "label": "Summarise", "function": "summary"},
        {"type": "table", "id": "rows",
         "columns": [
           {"key": "level", "label": "Level", "width": 120},
           {"key": "area",  "label": "Area",  "width": 90, "align": "right"}
         ]},
        {"type": "text", "id": "note", "readonly": true}
      ]
    }
  ],
  "functions": [
    {"id": "summary", "label": "Area Summary", "tab": "Areas", "panel": "Rooms", "order": 1,
     "params": {"type": "object", "properties": {}, "required": []}}
  ]
}
```

Manifest rules:

- `abi` — `1` or `2`. A plugin with `abi: 1` **cannot** declare `panels` (rejection with a message).
  A plugin with `abi: 2` may declare `functions` without panels — then it works as before.
- `panels[].id` — `[a-z0-9_.-]`, 1–64 characters, unique within the plugin; this is the stable palette
  name (in settings and presets it is called `plugin:<plugin id>:<panel id>`, see SPEC-019 §4).
- `panels[].title` — up to 200 characters, plugin data, **not translated**.
- `panels[].dock` — `left` / `right` / `bottom` / `float`; default `right`.
- `panels[].on_open` — the name of a function called once when the panel is first shown; optional.
- `panels[].refresh` — the name of a function called by the Refresh button and after every press of a
  panel button; optional (if absent, Refresh is not drawn).
- `panels[].widgets` — up to 512 widgets, nesting depth (for `row`/`group`) up to 8.
- `function` of a button must reference a function of **the same** plugin; otherwise a rejection on load.
- Functions called from a panel may or may not be declared in the ribbon (in the ribbon — only those
  that have a `label` and should be on the ribbon; see 6.6).

### 6.2. Widgets

| `type` | Mandatory fields | Optional | Purpose |
|---|---|---|---|
| `label` | `text` | — | a line of text |
| `row` | `widgets` | — | horizontal group |
| `group` | `widgets`, `title` | `collapsed` | collapsible group |
| `text` | `id`, `label` | `readonly`, `multiline`, `unit` | string input |
| `number` | `id`, `label` | `unit`, `min`, `max`, `decimals` | number |
| `checkbox` | `id`, `label` | — | boolean |
| `select` | `id`, `label`, `options` | — | choice from `{value,label}` |
| `level` | `id`, `label` | — | project level |
| `library` | `id`, `label`, `kind` | — | name from the library |
| `selection` | `id`, `label` | — | current selection, read-only |
| `button` | `id`, `label`, `function` | `enabled_when` | function call |
| `table` | `id`, `columns` | — | table from the response |
| `text_panel` | `id` | `height` | multiline text from the response |
| `separator` | — | — | line |

- `unit`: `length` | `area` | `volume` | `angle` | `none` (values are converted via project units).
- All widget `id`s are unique within a panel.
- Widget values are the call's `params`: `{id: value}`. Read-only widgets are not sent.
- `enabled_when` — a simple condition (`{"widget": "mode", "equals": "list"}`), otherwise the button is
  always active.

### 6.3. Panel response

A function's response is the usual envelope `{"ok": true, "result": {...}}`, where `result` may contain
`report` (as now — a dialog will open) and/or `panel`:

```json
{
  "panel": {
    "values": {"level": "Level 1"},
    "text": "3 rooms, 42.5 m²",
    "table": {"rows": [["Level 1", "42.5"], ["Level 2", "18.0"]]},
    "badge": "5",
    "error": null
  }
}
```

- All fields are optional: only what was sent is updated.
- `values` — widget values by `id`; an unknown `id` is ignored with a single line in the log.
- `table.rows` — up to 2000 rows, up to 32 cells per row, a cell is a string/number/bool/null; a cell
  longer than 1024 characters is truncated.
- `error` — a string shown as a red line at the bottom of the panel; it does not roll back what has
  already been done.
- An invalid response (not an object, unknown types, exceeding limits) — one line in the plugin log and
  in the status bar; the panel keeps the last successful content. The call itself is still considered
  successful, just as with `report` today (`plugins.rs:317-332`).

### 6.4. Lifecycle and calls

| When | What happens |
|---|---|
| The panel is shown for the first time | a call to `on_open` with the current values (if declared) |
| A panel button is pressed | a call to `function` with its panel's parameters |
| Refresh | a call to `refresh` with the current values |
| A widget changes | **no call** (values accumulate locally) |
| Every frame | **no call** — otherwise the per-call fuel limit is meaningless |
| The panel is closed | nothing; the values stay in window memory |

The call goes through the existing path (`Plugin` → `Session::transaction` → `wasmi`), which means:

- one call = one undo step labeled `<Plugin name>: <function label>`;
- an error/trap/limit rolls back the transaction;
- commands of other plugins are still unavailable;
- the `files` permission is checked by the host before the call (`docs/areas/plugins.md:145-168`).

Since the call is synchronous and may take up to 5 s (the limit), during the call the panel shows
"Running…", disables its buttons, and changes the cursor. Moving the call to a thread is a separate task,
not part of this spec.

### 6.5. Integration into the window

- A panel is a `Palette` in the existing docking system (`crates/ui-egui/src/dock.rs`): it is stacked,
  dragged by the header, floats, and closes like Properties.
- `Palette` is a `Copy` enum; a `Palette::Plugin(usize)` variant is added with a registry
  `BimApp::plugin_palettes: Vec<PluginPalette>` (following `Tab::Plugin(usize)`,
  `crates/ui-egui/src/lib.rs:368-387`). `Palette::known()` (introduced in SPEC-019) is extended with the
  plugin registry; `Palette::named` gets `known` as a parameter. Serialization stays by names:
  `plugin:<plugin id>:<panel id>` (the same name as in the layout; see SPEC-019 §4).
- The registry order is the plugin read order, then `panels[].order`, then `id`; the index is stable
  within a session but is **not** saved: only names go into the settings file.
- A plugin panel cannot be embedded into an existing ribbon tab; its button functions are duplicated in
  the plugin tab's panel if desired (as now, §1).
- Panel header: `title`, the `Icon::Code` icon, tooltip "Plugin panel <plugin name>".
- `Manage > Plugins` shows, for each plugin, the list of declared panels and the reason for a rejection if
  a panel is declared incorrectly.

### 6.6. Relationship with the ribbon

A function can be:

- only in a panel (`"ui": "panel"` in the function) — no button on the tab;
- both in a panel and on a tab (`"ui": "both"`, the default) — as now;
- only on a tab (`"ui": "ribbon"`) — ABI 1 behavior.

The `ui` field is optional, the default value is `both`. This is needed so that a panel can have service
functions (refresh, on_open) that have nothing to do on the ribbon.

### 6.7. Accessibility and appearance

- Each field has a caption linked via `labelled_by` (as in PyAddins and Edit Tools).
- Traversal order is the widget declaration order.
- The panel uses the host's colors, fonts, and spacing (`pal()`, `style::semibold`), in both profiles and
  in the dark theme. The plugin does not set its own colors.
- The minimum panel width is 240 pt (like Tools), the table scrolls horizontally, and long text in a cell
  does not wrap (like Markdown report tables, `docs/areas/plugins.md:236-240`).

## 7. ABI and the host

- `crates/plugin/src/manifest.rs`: `ABI` becomes the maximum version (2), `SUPPORTED: [1, 2]` is added;
  the check is `SUPPORTED.contains(&self.abi)` with the message "this BIMCraft speaks ABI 1 and 2". The
  `Panel` and `Widget` models are strict (`deny_unknown_fields`), like the whole manifest.
- `crates/plugin/src/runtime.rs`: the imports do not change (`bimcraft.execute`, `bimcraft.log`,
  `bimcraft.info`). `info` answers `"abi": 2`; the old fields remain, `markdown_reports` is not touched.
- The sandbox limits do not change (`docs/areas/plugins.md:105-121`).
- A function response containing `panel` is validated by the host before drawing; the validation is a
  separate function with unit tests, so that a "broken" response never crashes the window.

## 8. Security

- No new imports: a panel does not extend permissions.
- A panel button cannot call a function of another plugin (checked when the manifest loads).
- Links in `label`/`text` are not made clickable in v1 (text only); if they are needed — the same rules
  as for Markdown reports (`docs/areas/plugins.md:242-245`).
- Widget values are user data; they are not written to the log.
- The panel does not read the disk; a path is chosen only by the system dialog and only with the `files`
  permission.

## 9. Stages and estimate

| Stage | Content | Estimate |
|---|---|---|
| A | The form from `params` in a dialog, validation, tests | 2–3 days |
| B1 | ABI 2 manifest, panel model, validation, rejection with clear messages | 2 days |
| B2 | `Palette::Plugin`, registry, docking, header, closing, saving to `settings.json` | 2–3 days |
| B3 | Widget rendering, table, `panel` response, Refresh, "Running…" | 3–4 days |
| B4 | `Manage > Plugins`, Find, documentation, a sample plugin | 1–2 days |
| C | Reactivity on events (ABI 3) — after change notifications appear in ROADMAP | separate |

## 10. Test plan

Unit tests (`crates/plugin`):

- `a_plugin_of_abi_one_cannot_declare_panels`;
- `a_panel_naming_an_unknown_function_is_refused`;
- `an_unknown_widget_type_is_refused_and_names_the_widget`;
- `two_widgets_with_one_id_are_refused`;
- `a_manifest_of_abi_two_without_panels_loads_as_before`;
- `the_answer_of_a_panel_is_checked_before_the_window_sees_it` — for all variants of a broken `panel`.

Window tests (`crates/ui-egui/src/tests/`, `egui_kittest`, like `tests/presets.rs`):

- `a_plugin_panel_opens_with_its_widgets_and_a_click_passes_their_values` — a fixture plugin reads the
  parameters and writes a note; we check that the values arrived;
- `a_panels_table_shows_the_rows_and_its_badge_and_error_line`;
- `a_button_that_needs_files_asks_for_the_permission_first` (as now for `path`);
- `a_panel_of_a_disabled_plugin_closes_and_its_state_is_kept` (overlaps with SPEC-019);
- `a_second_click_does_not_duplicate_the_first_calls_undo_step`.

CLI/MCP: regression — the `plugin:<id>:<fn>` commands work without a window and without panels;
`list_commands` does not change; in read-only mode a panel button that changes the model is rejected just
like a command.

Build: `cargo check --workspace`, `cargo clippy` (without test profiles), `cargo i18n check`.

## 11. i18n

New interface strings (English — the key, via `t(...)`/`msg!(...)`):

- `"Run"`, `"Cancel"`, `"Refresh"`, `"Running…"`;
- `"Unsupported field"`, `"This field is required"`, `"A number is expected"`, `"Out of range"`;
- `"The plugin's panel couldn't be shown"`, `"Panel"`, `"Plugin panel"`;
- `"Tick the fields and run the plugin's command"`.

Panel captions and texts are plugin data, **not translated** (like `label`, `tab`, `panel` today).

After the changes: `cargo i18n update` (after `rustfmt`), then `cargo i18n check`.

## 12. Documentation

- `docs/areas/plugins.md`: new sections "Parameter form", "Panels (ABI 2)", "Response schema",
  "Boundaries" (what a panel cannot do), update §"Version 1 boundaries" → "Version 2".
- `crates/plugin-sdk/src/lib.rs`: a doc comment about panels and an example.
- `crates/plugin/tests/sample-plugin`: add a panel declaration and its function — it also serves as the
  test fixture.
- `README.md`: a line in "What works today" about plugin panels (one sentence).

## 13. Acceptance criteria

1. An ABI 1 plugin loads and works unchanged; an ABI 2 plugin without `panels` — too.
2. A button with `params` shows a form; the values go into the call; a validation error names the field.
3. A panel opens as a dockable window, is dragged, floats, and closes; its position and open state are
   saved in `settings.json` and survive a restart.
4. Pressing a button = one call = one undo step; an error rolls everything back.
5. A broken panel response does not crash the window: a line in the log, the panel shows the previous
   content.
6. No new imports for the guest; still no `unsafe` in the workspace.
7. `Manage > Plugins` shows the panels and the reasons for rejection.
8. The §10 tests are green; `cargo i18n check` is green.

## 14. Risks and open questions

| Risk | Mitigation |
|---|---|
| A synchronous call of up to 5 s "freezes" the window | "Running…", disabling the panel; moving to a thread is a separate task |
| A panel with 100 fields does not fit in the window | limits (512 widgets), scrolling, minimum width |
| A plugin declared a panel, but the function always fails | an error in the panel, `error`, a line in the log; the panel remains |
| Panel names change between plugin versions | a key based on the panel `id`, not `title`; SPEC-019 requires preserving unknown keys |
| Reactivity is expected right away | explicitly move it to stage C and explain the dependency on events |

Questions for discussion:

1. Are nested groups (`group`) needed in v1 — or are `row` and a flat list enough?
2. Is a `file`/`folder` widget needed with path selection right in the panel (now a path is only via the
   `files` permission and the system dialog)?
3. Does a panel button need the ability to send the result to a report (`report`) instead of `table`?

## 15. Execution order: this PR is the second, after SPEC-019

### 15.1. What will already be in `main` by the time of the rebase

- `panels_hidden`, panel keys `tab:…`, the current tab in `ribbon::panel`, the panel checkbox in Edit
  Tools, "Show Every Panel", the empty tab.
- `Docking::from_json(v, policy, known)` and `Policy::{Live, Preset}`.
- `Palette::known()` — a single point of built-in palette enumeration (instead of `Palette::ALL`).
- `Preset { panels_hidden, profile }`, `apply_preset` with the profile first, "Reset to Standard View".
- The profile rule at draw time: in Rev the Tools palette is not drawn and is not written to the file.
- Plugin ribbon panels **already** hide and are part of presets — their keys
  `tab:plugin:<tab name>/<panel>` do not require ABI 2.

### 15.2. Rebase checklist (follow in order)

1. **`Palette::Plugin(usize)` + registry.** `BimApp::plugin_palettes: Vec<PluginPalette>`; populated
   when plugins are installed (next to `plugin_tabs`, `plugins.rs`). The index is the plugin order, then
   `panels[].order`, then `id`; it is not written to the file.
2. **`Palette::known()` is extended with the registry.** The signature from SPEC-019 does not change:
   `known()` must return the built-in and connected plugin palettes. If the registry is needed in
   `named`/`from_json` — pass it the same way as `known` today.
3. **Palette names** — `plugin:<plugin id>:<panel id>` in `to_json`/`closed`; `Policy::Preset` still
   skips unknown names, `Policy::Live` does not open what does not exist.
4. **Do not touch plugin ribbon panels.** Their keys stay `tab:plugin:<tab>/<panel>`;
   `panels[].id` is the name of a **palette**, not of a ribbon panel. Do not mix the two namespaces.
5. **"Reset to Standard View"** closes plugin palettes by itself, because they are not in
   `Docking::default()`/`revit()`. Nothing needs to be added to the reset.
6. **Do not change `Policy`, `profile`, the reset, or `palette_switches`.** If new settings behavior is
   needed for plugin panels — a separate PR.
7. **i18n catalogs.** After the rebase — `cargo i18n update` (`rustfmt` first); do not resolve conflicts
   in `.po` manually: regenerate.
8. **Tests.** Own ones — in `tests/plugin_panels.rs` and as unit tests in `crates/plugin`; do not rewrite
   `tests/presets.rs` and `tests/dock_policy.rs`, only add to them if necessary.

### 15.3. The second half of R9 from SPEC-019 is closed here

- Plugin palettes appear in Settings > User Interface (`palette_switches`), in the "Palettes" section of
  the preset editor, and in `closed` — automatically, via `Palette::known()`.
- They behave like built-in ones: dragged, floated, closed, included in a preset, not included in
  `panels_hidden`.
- The palette state of a disabled plugin is preserved in `settings.json` and in presets; when the plugin
  returns — it is restored.

### 15.4. Definition of Done for this PR

- [ ] ABI 1 works unchanged; ABI 2 with `panels` and without — loads.
- [ ] The palette panel opens, docks, and is saved in `settings.json` and in presets.
- [ ] The form from `params` works for ribbon and panel buttons.
- [ ] Plugin palettes are visible in the toggles and in the preset editor (R9 fully closed).
- [ ] "Reset to Standard View" closes plugin palettes too.
- [ ] A broken panel response does not crash the window; the limits are respected; there are no new
      imports.
- [ ] The §10 tests are green; `cargo i18n check` is green; the documentation is updated.

## 16. Whether it can be done in parallel

| What | Can it be parallel with SPEC-019 | Why |
|---|---|---|
| Stage A (the form from `params`) | **Yes** | does not touch `dock.rs`, `presets.rs`, `settings.rs`; only `plugins.rs` and the new `plugin_form.rs` |
| Stage B1 (ABI 2 manifest, panel model, validation) | **Yes, with a caveat** | `crates/plugin` barely overlaps with SPEC-019; there are no conflicts in the UI |
| Stages B2–B4 (`Palette::Plugin`, registry, docking, panel responses) | **No** | direct conflicts in `dock.rs` (`Palette`, `named`, `to_json`/`from_json`), `settings.rs` (`palette_switches`), `presets.rs` (the editor), `lib.rs` (`palette_body`) |

If parallel work is really needed:

- agree on the contract **before** starting: `Palette::known()`, `Docking::from_json(v, policy, known)`,
  the names `plugin:<id>:<panel id>`, the `tab:…` keys — all of this is already described in SPEC-019 §4
  and §15.2;
- keep an integration branch and rebase it more often than once a day;
- budget **+1–2 days** for resolving conflicts and re-checking layout saving (the most likely source of
  regression — `closed` and `Policy::Preset`).

**Conclusion:** two PRs sequentially — **SPEC-019, then SPEC-020**. If parallelism is needed, the only
safe parallel work is stage A (and, with a caveat, B1) from this spec.
