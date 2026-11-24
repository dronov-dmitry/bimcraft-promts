# SPEC-019.2. Panel menu on right-click of a panel header

Status: **draft**, implementation required
Date: 2026-11-24
Project: BIMCraft (`main`, worktree `BIMCraft-019.2`, commit `1bf407e`; Rust, `crates/ui-egui`,
egui 0.36.2)
Source: user request of 2026-11-24: "when I right-click the header of any panel, a drop-down list of
all possible panels should open; if a panel is enabled, it has a check mark on the left, if not —
no check mark; clicking a panel shows or hides it".
Related specs: `ТЗ-019-panel-visibility-presets.md` (hiding panels, `panels_hidden`, keys
`tab:<tab>/<panel>`; already implemented), `ТЗ-020-plugin-panels.md` (plugin panels).
Task type: window UI only; the model, `.bimc`, commands, ABI and the `settings.json` format do not
change.

> **In short.** Right-clicking a panel name — in the Rev ribbon, in the BIMCraft Tools palette, on a
> folded panel and in Edit Tools — opens a menu with check marks. It contains the panels of the
> current tab (a check mark = the panel is visible), and below that **Other Tabs** with the panels of
> the remaining tabs, including plugin panels. Clicking a row hides the panel or brings it back; the
> menu stays open so several panels can be toggled in one go. There are no new settings or keys: the
> menu reads and writes the same `panels_hidden` as Edit Tools — only faster and from anywhere a
> panel is visible.

> **Extension (2026-11-24, at the customer's request).** The same menu opens on right-click of the
> **dock header of any palette** (both on a stack tab and on the header of a floating palette) and
> lists the window's **palettes** with check marks. A check mark = the palette is open; a click
> hides or shows it through `Docking::show` — the same `closed` list in the `palettes` section of
> `settings.json` as in View > User Interface (no new keys). The menu stays open, and is closed by
> Escape, by a click outside, by a second right-click and together with its own palette. The list is
> `Palette::known()` without Tools (that is the tools surface, not a palette); the names are the
> palette header names (`t(p.label())`: in BIMCraft these are Inspector and Navigator). So SPEC-019.2
> now covers both the ribbon/Tools-palette panels and the window palettes.

## 0. General working rules

1. The language of communication is Russian; code, comments, strings in code and log messages are
   English. User-facing window strings go through the existing `msg!`/`t()`, and the catalogs are
   updated (`cargo i18n update`).
2. Tests (`cargo test`, `cargo nextest`, doctests) are **not run** in this task without a direct
   request from the user (rule `~/.config/opencode/rust-rules.md`). Allowed: `cargo check`,
   `cargo build`, `cargo clippy` (without `--all-targets`), `cargo fmt --check`, `cargo i18n check`;
   tests run on CI and on the user's side.
3. Commits are atomic: the shared panel list (`panel_groups`) separately, the menu itself and its
   wiring to the four header kinds separately, tests, i18n and documentation separately. Each commit
   builds and does not change window behavior beyond what is described here.
4. The `settings.json` format **does not change**: there are no new keys, and the `panels_hidden`
   keys are the same as in SPEC-019. Old files are read as before.
5. These are **window** settings, not project settings: nothing is written to `.bimc`.
6. There are no new dependencies; no new `unsafe` appears; panics are forbidden by the lints.

## 1. The problem

### 1.1. A panel can only be hidden through Edit Tools

Today panel visibility lives in `panels_hidden` (`settings.rs:111`) and is edited in exactly one
way — with a checkbox in Edit Tools:

- in the ribbon, a panel title is the tool-list button (`ribbon.rs:487-505`), and in Edit Tools it
  is a checkbox (`ribbon.rs:466-474`);
- in the BIMCraft Tools palette, a panel name is a painted label with no mouse response
  (`ribbon.rs:417-428`, `Sense::hover()` on line 425);
- on a folded panel, the name sits on the button itself (`ribbon.rs:452-463`, `folded_button` :592).

To remove one unwanted panel, the user has to turn on Edit Tools, find the tab (in the ribbon it may
be scrolled), find the panel among its tools, and only then clear the checkbox. Bringing the panel
back is the same journey. That is disproportionate to the frequent operation "this panel is in my
way right now".

### 1.2. Right-click currently does nothing

- In the ribbon, the title has `Sense::click`, but there is no right-button handler: the menu does
  not open.
- In the Tools palette, the label does not accept clicks at all: `Sense::hover()`.
- A hidden panel is not drawn at all (`ribbon.rs:410-413`), so it cannot be brought back from the
  panel itself; what remains is Edit Tools, the button on an empty tab (`presets.rs:497-503`) and
  Find for the tools.

### 1.3. What the user expects

A model familiar from other programs: right-click a panel header → a list of panels with check marks
→ click a row to enable or disable the panel. This does not replace Edit Tools (which also has tools,
presets and the profile); it provides a shortcut exactly for panel visibility.

## 2. Goals

1. Open the panel menu by right-clicking the header of **any** panel — in the ribbon and in the Tools
   palette, in normal and folded form, and in Edit Tools.
2. Show all window panels in the menu with a checkbox on the left: ticked — the panel is visible,
   empty — it is hidden.
3. On clicking a row, hide or show the panel **immediately**; the state survives a restart.
4. Keep the menu open after a click so several panels can be enabled or disabled in one go.
5. Give access to all tabs (including plugins) without stretching the menu across the whole screen.
6. Introduce no new settings, keys or formats: work on top of `panels_hidden` from SPEC-019.

## 3. Non-goals

- No reordering or changing the order of panels — visibility only.
- No hiding of individual tools: that is still Edit Tools (`tools_hidden`).
- No separate "Panels" page in Settings and no new ribbon button.
- No new hotkeys; the keyboard path is Edit Tools (see R7).
- No writing the change into the active preset bypassing the existing "Done / Save / Update" rule
  (see R8).
- No per-view, per-sheet or per-project panel visibility.
- No command, MCP tool or `.bimc` field.

## 3.1. What it looks like to the user

1. The Phasing panel on the Manage tab is in my way: I right-click its name and clear the check mark —
   the panel disappears. To bring it back: right-click the name of any Manage panel and tick it again.
2. I hid all panels of the View tab. The tab says so and offers a button; besides that, from the
   Architecture ribbon I open **Other Tabs → View** and bring back the panel I need.
3. I work in Sketch: the Sheet Composition panel is already hidden by the scale. In the menu it has
   no check mark and is unavailable, with a tooltip explaining why. I switch the scale to Studio —
   the panel comes back by itself.
4. Same in the BIMCraft profile: right-click the panel name in the Tools palette.
5. In Edit Tools, right-clicking a title opens the same menu; a left click on the checkbox works as
   before and does not "jiggle" the title itself.
6. I hid the panel whose name was in the header of an open menu: the menu closed together with it
   (otherwise it would have nothing to hang from). A neighboring panel or Edit Tools brings it back.

## 4. Terms and identifiers

**Panel header** — the strip with the panel name. It looks different in different places, but the
menu opens from any of the four:

| Where | How it is drawn | Where in code |
|---|---|---|
| Rev ribbon, normal panel | name under the tools, itself a tool-list button | `ribbon.rs:487-505` |
| Rev ribbon, folded panel | name on a "tile" button with an arrow | `ribbon.rs:452-463`, `folded_button` :592 |
| Edit Tools (both profiles) | checkbox with the name above the tools | `ribbon.rs:466-474` |
| BIMCraft Tools palette | painted name above the tool rows | `ribbon.rs:417-428` |

**Palette dock header** — the top strip of a palette (Properties/Inspector, Project Browser/Navigator,
Tools, Assistant, Script Automation), or a tab of a palette stack, or the header of a floating
palette: `dock.rs`, `stack_ui`. It has no panel key; the palette menu lists palettes, not ribbon
panels (§5.1).

**Current tab** — the tab whose panel was pressed: `CURRENT_TAB` in `ribbon.rs:69-71`, set by
`ribbon::set_tab` (`lib.rs:4897`, `4959`; `editor/document.rs:507`).

Panel keys are from SPEC-019, unchanged:

| What | Format | Example |
|---|---|---|
| Built-in panel | `tab:<tab>/<panel>` | `tab:View/Phasing` |
| Contextual tab | `tab:Modify/<panel>` | `tab:Modify/Properties` |
| Plugin tab panel | `tab:plugin:<name>/<panel>` | `tab:plugin:Areas/Rooms` |
| Family document panel | `tab:family:<DocTab>/<panel>` | `tab:family:Create/Forms` |

Panel states:

- **visible** — the key is not in `panels_hidden`, and the scale does not hide it;
- **hidden by the user** — the key is in `panels_hidden`;
- **hidden by the scale** — `ribbon::scale_hides(name)` (`ribbon.rs:147`); nothing is written to
  `panels_hidden` in that case, and the menu does not show it as enabled.

## 5. Functional requirements

### R1. Trigger

- Right-click (secondary mouse button) on a panel header opens the menu at the pointer position.
- Works for all four header kinds from §4 and in both profiles.
- Works with the ribbon folded (`ribbon_peek`), because the panels are drawn by the same
  `ribbon::panel`.
- Right-clicking the tools **inside** a panel does not open the panel menu: buttons and rows have
  their own context menus, if any.
- The left button does not change behavior: in the ribbon it still opens the panel's tool list, in
  Edit Tools it toggles the checkbox, in the Tools palette it does nothing.
- Pointer-only menu; the keyboard path stays as before — Edit Tools and the preset editor.

### R2. Menu contents

1. **Panels of the current tab** — the first group. The group heading is the tab name (translated),
   below it the checkboxes of its panels in declaration order (`PANELS`, `lib.rs:1737-1752`),
   including hidden ones.
2. A separator.
3. **Other Tabs** — a submenu; in it every other window tab, each with its own panels as checkboxes.
4. There is nothing mandatory at the bottom. "Show Every Panel", "Edit Tools…", "Settings…" are
   **not** part of the v1 menu (see §14).
5. Menu width is about 240 px; height is limited so the menu fits in the window. With 14 tabs a flat
   list of all panels would be taller than the screen, so the panels of other tabs are put into a
   submenu rather than stacked in one column.

Tab order in **Other Tabs**: as in `PANELS` (Architecture, Structure, …, Modify), then plugin tabs in
load order, then family tabs from the keys already seen. The current tab is not in the submenu.

Names: built-in tabs and panels and family ones go through `t()`; plugin tab and panel names are as
in the manifest, untranslated (the same rule as in SPEC-019 §10 and `presets.rs:475-478`).

### R3. Check marks and toggling

- Each row has a checkbox on the left: ticked — the panel is visible now, empty — hidden by the user
  or by the scale.
- A click on a row (or on its checkbox) sets or clears the check mark and **immediately** hides or
  shows the panel. The change goes through the existing queue `ribbon::toggle_panel(&key)`
  (`ribbon.rs:189`) and is applied after the frame in `lib.rs:2617-2621`, as with the Edit Tools
  checkboxes. There is no separate write path.
- The panel disappears or appears on the next frame; the ribbon width fitting is recomputed by
  itself (SPEC-019 R1).
- The state reaches `settings.json` the same way as an edit in Edit Tools; a restart preserves it.
- The menu **stays open** after a click: several check marks can be cleared in a row. It is closed by
  Escape, by a click outside the menu, and by a second right-click on the header.
- There is no undo: this is a window setting, not a model step. Ctrl+Z in the model is unaffected.
- If the check mark is cleared on the very panel whose header opened the menu, the panel disappears
  and the menu closes at that moment (R6). It can be brought back from a neighboring panel, from Edit
  Tools, or with the empty-tab button.
- A second click on an already ticked row hides the panel; a second click on an empty one shows it.
  There is no latching.

### R4. All tabs

- The menu lists every panel the window knows: built-in ones from `PANELS`, plugin tab panels from
  `plugin_tabs`, family document panels the window has encountered (`panel_keys`).
- A hidden panel of another tab is shown empty; clicking it writes its key into `panels_hidden`, and
  this is visible when the tab is open.
- If all panels of a tab are hidden, its body still shows the notification with a button
  (`presets.rs:496-503`); the menu does not change that behavior.
- Panels of a disabled or removed plugin do not appear in the menu (there is nothing to draw them
  with), but their keys remain in `panels_hidden` and in presets — as in SPEC-019 R9.

### R5. Project scale

- A panel that the scale hides (`ribbon::scale_hides`, `ribbon.rs:147`) is shown in the menu
  without a check mark and as a **disabled** row with the tooltip "The project scale leaves this
  panel out".
- Clicking such a row does nothing; it is not written to `panels_hidden`.
- If the user hid a panel manually and then the scale hid it as well — when the scale is reverted,
  the panel stays hidden: the manual decision wins (just as in SPEC-019 R1).
- The menu never changes the scale; for that there is a chip in the window title bar.

### R6. Menu lifecycle

- The menu closes on Escape, on a click outside it, and on a second right-click on the same header.
- The menu stays open while check marks are toggled (this needs
  `PopupCloseBehavior::CloseOnClickOutside`; by default the egui menu closes after the first click
  inside it).
- If the panel whose header opened the menu gets hidden, the menu is closed **forcibly** in the same
  frame. This is not cosmetics: a popup whose header is no longer drawn would remain "open" in
  memory and would pop up again when the panel is brought back. Therefore:
  - after toggling its own key, the menu is closed explicitly (`Popup::close_id`);
  - in the early-return branch of a hidden panel (`ribbon.rs:410-413`), the menu of its key is
    closed — this also covers the case where the panel was hidden by a preset or another mechanism.
- The menu and the panel's tool list (left click) are **two different popups with different ids**.
  Both hang on the same `Response`, and `Popup::menu(r)`/`Popup::context_menu(r)` share one id by
  default: then the open tool list and the panel menu would blend into one window. The menu id is
  built from the panel key, for example `Id::new(("ribbon-panel-menu", key))`.

### R7. Appearance and accessibility

- The group heading is the tab name in bold, and the rows are indented, as in the preset editor
  (`presets.rs:480-490`).
- A panel row is a checkbox with the `Checkbox` role and the panel name; a disabled row keeps the
  name and the tooltip.
- Colors come from `pal()`; it works in the light and dark theme, in both profiles and at both
  interface scales.
- The menu opens at the pointer and is clamped to the window edges, like other egui context menus.
- The panel name in the Tools palette becomes interactive (`Sense::click` instead of `Sense::hover`
  at `ribbon.rs:425`) and gets an accessible name for screen readers and tests; a left click on it
  still does nothing. In the ribbon the header already has an accessible name (`"{name} ▾"`,
  `ribbon.rs:501-502`).
- The menu is a pointer shortcut, not the only path: a keyboard user still edits panels in Edit
  Tools, where every panel has a checkbox.

### R8. What the menu does not do

- It does not touch `tools_hidden`: panel visibility and the visibility of its tools are independent
  (SPEC-019 R2).
- It does not write to the active preset. The change is a window change, like dragging a palette; the
  preset learns it by the existing rules: **Done** in Edit Tools, **Update from Current Layout**, or
  **Save Preset**. If no preset is selected, the state stays only in the window, as it does now.
- It does not add an undo step and does not change the model, `.bimc`, commands or MCP.
- It does not change the `settings.json` or preset format.
- It does not hide panels per view or per project.

### R9. Nothing breaks

- The tool list on a left click in the ribbon, its width, tooltips and position — as now.
- Ribbon width fitting, folded panels, moving tools into the list — as now.
- Edit Tools: both checkboxes (panel and tool) work; a right click does not toggle the checkbox.
- Empty tab, Find, presets, palette layout, plugin tabs and family tabs — unchanged.
- Context menus of tools and rows inside a panel are not intercepted.

### R10. A single panel list

The list of tabs and panels for the menu and for the preset editor must be the same one, not a
second copy of the enumeration:

```rust
// lib.rs: every tab the window can show and its panels, in the order they are drawn.
pub(crate) fn panel_groups(&self) -> Vec<(String, Vec<String>)>

// The flat keys, as the preset editor lists them; built from panel_groups, not from PANELS again.
pub(crate) fn known_panel_keys(&self) -> BTreeSet<String>
```

- `panel_groups` collects: built-in tabs from `PANELS` (declaration order), plugin tabs from
  `plugin_tabs`, then the missing tabs and panels from `panel_keys` (family documents and everything
  the window has drawn).
- `known_panel_keys` is the flat union of `panel_groups`; `presets::panel_tree` and the menu get the
  same key set.
- `ribbon::panel_rules` (`ribbon.rs:179`, called from `lib.rs:2447`) receives this list along with
  the hidden keys: `panel_rules(hidden, groups)`. The groups are recomputed per frame; caching them
  is unnecessary and harmful — the list depends on the loaded plugins and open family documents.
- No second key space appears: still the same `tab:<tab>/<panel>`.

### 5.1. Palette menu (extension)

The same model, but for window palettes rather than ribbon panels.

- **P1. Trigger.** Right-click on a palette dock header (`dock.rs::stack_ui`, `back`) opens the menu
  at the pointer; on a palette stack tab — too, with its palette as "own"; on a floating palette —
  on its header. A left click does not change behavior: the header is still dragged, the tab is
  selected, the close and Dock buttons work.
- **P2. Contents.** A flat list of the window palettes — `Palette::known()` in its order, **except
  Tools**: Properties, Project Browser, Assistant, Script Automation (and plugin palettes, when they
  appear). Tools is not a palette but the tools surface (in Rev the ribbon replaces it, in BIMCraft
  the left column), so it is not part of the lists of palettes that can be enabled/disabled: neither
  here, nor in Settings (where the list is shared with the **Palettes** menu in the window title
  bar), nor in View > User Interface, nor in the preset editor — just as the Rev ribbon is not there
  either. Those lists take `Palette::switchable()` = `Palette::known()` without Tools; `known()`
  remains the layout list, and Tools is still stored in the `palettes` section of `settings.json` and
  in presets. A layout where Tools is closed (an old file, a preset) is restored by **Reset Palette
  Layout**. Names are the palette header names: `t(p.label())`, that is, Inspector and Navigator in
  BIMCraft. There is no Other Tabs submenu and no group headings: there are few palettes, and the
  menu does not exceed the screen.
- **P3. Check marks.** A check mark is set if `docking.shown(p)`; a click writes visibility through
  `show_palette`/`Docking::show` — the same path as View > User Interface, Settings and presets. The
  palette disappears or appears on the next frame; the menu stays open for several toggles. If the
  palette whose header opened the menu is hidden, the menu is closed explicitly (`Popup::close_id`)
  and does not pop up again; it is also closed by Escape, by a click outside, and by a second
  right-click on the same header (`PopupCloseBehavior::CloseOnClickOutside`, its own id
  `("dock-palette-menu", palette.name())`).
- **P4. Persistence.** There are no new keys: it is the same `palettes` (the `closed` list) in
  `settings.json` that is written on any palette move. The change reaches the active preset only
  through **Done** in Edit Tools, **Update from Current Layout**, or **Save Preset** — like dragging
  a palette. Undo, the model, `.bimc`, commands and MCP are unaffected.
- **P5. Profiles and scale.** The menu works in both profiles and for floating palettes. The project
  scale does not hide palettes (`Scale::hides` is about panels and tools), so there are no rows
  unavailable due to scale here, and there are no disabled rows in the menu at all. Tools is not in
  the list in either Rev or BIMCraft: in Rev its place is taken by the ribbon (`drawn_stack` draws it
  neither docked nor floating), in BIMCraft it replaces the ribbon with the left column. There is no
  Tools check mark in the other palette lists either (Settings, the **Palettes** menu, View > User
  Interface, the preset editor) — just as the Rev ribbon is not there. The list still remains in
  `settings.json`: a Tools closed in an old file or preset is restored by **Reset Palette Layout**,
  and a preset stores it as part of the layout.
- **P6. Nothing breaks.** Dragging by the header, stack tabs, the "×" and Dock buttons, palette
  alignment, floating windows, presets and settings — unchanged; the right button does not start a
  drag.

## 6. Data model

Nothing new. The menu reads and writes existing fields:

```json
{
  "panels_hidden": ["tab:View/Phasing", "tab:plugin:Areas/Rooms"],
  "presets": [
    {
      "name": "Documentation",
      "panels_hidden": ["tab:Architecture/Wall"]
    }
  ]
}
```

- The top level is the window state; the menu changes only that.
- `presets[].panels_hidden` changes only through Done/Save/Update (R8).
- Limits and robustness against garbage are from SPEC-019 §6 (`MOST_PANELS` = 512, `PANEL_KEY` =
  160, `panels_from_json`).

## 7. Integration points (worktree `BIMCraft-019.2`, `1bf407e`)

| File | What to change |
|---|---|
| `crates/ui-egui/src/ribbon.rs` | new thread-local with the group list; second parameter of `panel_rules` (:179); `panel_menu()` (menu rendering), `tab_label()` (tab name translation), `panel_menu_id(key)`; wiring the menu to the four headers: Edit Tools checkbox (:470), ribbon row-button (:495-504), folded button (:592), palette label (:417-428); closing the menu in the hidden-panel branch (:410-413) and after hiding its own key |
| `crates/ui-egui/src/lib.rs` | `panel_groups()` and `known_panel_keys()` through it (:4936-4952); the call `ribbon::panel_rules(self.panels_hidden.clone(), self.panel_groups())` (:2447); the `take_toggled_panels` queue (:2617-2621) is unchanged |
| `crates/ui-egui/src/presets.rs` | `panel_tree` (:461-494) takes `tab_label`/`known_panel_keys` from the shared place; preset editor behavior is unchanged |
| `crates/ui-egui/src/settings.rs` | no changes for panels: `panels_hidden` is already read and written (:111, :185, :238); for palettes, `palette_switches` takes `Palette::switchable()` (without Tools) |
| `crates/ui-egui/src/tests/presets.rs` | new window tests (R9 above); existing panel tests stay green |
| `crates/ui-egui/src/dock.rs` | (extension §5.1) `Palette::switchable()` — palettes without Tools for all lists; `palette_menu()` and `palette_menu_popup()`; wiring to `back` in `stack_ui` and to the stack tabs; `palette_menu_id(palette)`; closing when its own palette is hidden |
| `crates/ui-egui/src/lib.rs`, `crates/ui-egui/src/presets.rs` | (extension) View > User Interface and the preset editor list palettes through `Palette::switchable()`; Tools remains only in the layout and `settings.json` |
| `crates/ui-egui/src/tests/dock.rs` | palette menu tests (P1–P4) |
| `docs/areas/desktop.md` | the paragraph about Edit Tools and `panels_hidden` is extended with the right click and the menu; the palette menu is there too |
| i18n catalogs | two new strings for panels (§10); for palettes there are no new strings — "Palettes" already exists |

Rendering skeleton (names and signatures are a guide, not verbatim code):

```rust
fn panel_menu(ui: &mut Ui, own: &str) {
    let (hidden, groups) = menu_rules();
    let current = current_tab();
    for name in panels_of(&groups, &current) {
        row(ui, &current, name, own, &hidden); // checkbox; toggle_panel; close if own
    }
    ui.separator();
    ui.menu_button(t("Other Tabs"), |ui| {
        for (tab, panels) in groups.iter().filter(|(tab, _)| *tab != current) {
            ui.menu_button(tab_label(tab), |ui| {
                for name in panels {
                    row(ui, tab, name, own, &hidden);
                }
            });
        }
    });
}
```

Wiring (for each of the four headers):

```rust
let id = Id::new(("ribbon-panel-menu", key.as_str()));
Popup::context_menu(&response)
    .id(id)
    .close_behavior(PopupCloseBehavior::CloseOnClickOutside)
    .show(|ui| panel_menu(ui, &key));
```

## 8. Compatibility

- `settings.json` and presets do not change: old files are read, new ones are written with the same
  keys.
- Rolling back to a previous version is safe: the menu is only a way to edit the existing field.
- Plugin panels are picked up automatically: their keys are already in `known_panel_keys` and in
  `panel_keys`, so no separate work under SPEC-020 is required.
- Keys of panels of disabled plugins and of renamed panels are not deleted and do not interfere with
  the menu.
- The menu does not appear in the project, in export, in PNG/PDF/IFC or in the command line.

## 9. Test plan

Unit tests (`lib.rs`/`ribbon.rs`):

- `panel_groups_list_every_tab_once_in_ribbon_order` — built-in tabs follow the `PANELS` order, each
  panel has exactly one key, and a plugin tab comes after the built-in ones;
- `known_panel_keys_is_the_flat_panel_groups` — the flat set matches the union of the groups; there is
  no second source;
- `a_panel_the_scale_hides_is_listed_but_not_ticked` — `scale_hides(name)` yields an empty and
  disabled row, and `panels_hidden` does not change.

Window tests (`egui_kittest`, modeled on `tests/presets.rs`):

- `right_clicking_a_panel_name_lists_its_tabs_panels_with_their_boxes` — View, right-click
  "Graphics ▾": the menu contains Graphics, Create, Sheet Composition, Windows, all with check marks;
- `clearing_a_box_in_the_menu_takes_a_panel_out_and_ticking_it_brings_it_back` — clear the check mark
  of a neighboring panel, check `panels_hidden` and the disappearance; bring it back with the same
  menu;
- `the_menu_stays_open_for_several_panels_and_closes_when_its_own_panel_goes` — two panels in one
  go; then hide the own one — the menu is closed; a tab with a single panel shows the notification;
- `every_tabs_panels_are_under_other_tabs` — from View open Other Tabs → Manage → Phasing and check
  `tab:Manage/Phasing` in `panels_hidden`;
- `a_panel_the_scale_leaves_out_is_listed_unticked_and_cannot_be_ticked` — Sketch, Sheet Composition:
  the row is empty, disabled, with a tooltip; a click writes nothing;
- `the_menu_opens_on_a_folded_panel_and_in_the_tools_palette` — narrow ribbon (Rev) and BIMCraft
  profile (Tools palette);
- `the_menu_leaves_the_edit_tools_box_alone` — in Edit Tools, right-clicking the "Build" checkbox
  opens the menu and does not toggle the check mark; then toggling from the menu yields the same
  `panels_hidden`;
- `a_menu_toggle_reaches_the_settings_file_and_waits_for_done_to_reach_the_preset` — the settings
  file gets the key immediately; the preset only after Done/Update;
- `a_menu_never_reopens_by_itself_when_its_panel_returns` — hide the own panel, bring it back from
  Edit Tools: the menu does not pop up;
- after SPEC-020 (if the test plugin already has plugin panels):
  `a_plugins_panels_are_listed_under_its_tab`;
- regression: the existing `a_panel_taken_out_in_edit_tools_…`, `a_hidden_panel_is_still_listed_…`,
  `a_tab_whose_panels_are_all_hidden_…` stay green; a left click in the ribbon opens the tool list,
  not the panel menu.

Window tests for palettes (`tests/dock.rs`, §5.1):

- `right_clicking_a_palettes_header_lists_every_palette_with_its_box` — Rev, right-click the
  Properties header: the menu contains all of `Palette::known()` except Tools, with correct check
  marks, including closed ones;
- `a_palettes_box_in_its_menu_closes_and_opens_it_and_the_menu_stays_open` — clear the check mark of
  Project Browser, check `docking.shown`, open it with the same menu;
- `the_palettes_menu_closes_with_its_own_palette_and_does_not_reopen` — hide Properties from its own
  menu: the menu is closed; bring the palette back — the menu does not pop up;
- `the_palettes_menu_opens_on_the_inspectors_header_in_bimcraft` — BIMCraft profile: right-click the
  Inspector header, the Navigator check mark hides Project Browser, Tools is not in the menu;
- `the_switchable_palettes_are_the_known_ones_without_tools` — `Palette::switchable()` = `known()`
  without Tools;
- `the_settings_palette_list_leaves_the_tools_palette_out` — there is no Tools row in Settings;
- `the_app_bars_palettes_menu_brings_a_closed_palette_back_and_resets_the_layout` — in the
  **Palettes** menu there is no Tools, a closed Project Browser comes back with a check mark, and a
  closed Tools is restored by Reset;
- regression: dragging by the header, stack tabs, "×", Dock, floating palettes and the existing
  `tests/dock.rs` tests — green.

Build and checks: `cargo check --workspace`, `cargo clippy` (without test profiles),
`cargo fmt --check`, `cargo i18n check`. Tests are run on a direct request.

## 10. i18n

| String | Where |
|---|---|
| `Other Tabs` | submenu of the panels of the remaining tabs |
| `The project scale leaves this panel out` | tooltip of a disabled row |

Panel and tab names are taken from existing ones: built-in — `t(name)`, plugin — as in the manifest.
The menu heading is the current tab name; there is no separate "Panels" string.

For palettes there are no new strings: the "Palettes" heading already exists in the catalogs, and the
palette names are `t(p.label())` (Inspector/Navigator in BIMCraft), existing.

## 11. Documentation

- `docs/areas/desktop.md`: in the paragraph about Edit Tools and `panels_hidden`, add that a right
  click on a panel name opens a menu with check marks, that it works in both profiles, on a folded
  panel and in Edit Tools, and that the panels of other tabs are under **Other Tabs**; the change is
  a window change, not a preset one.
- Same place: a panel hidden by the scale is visible in the menu without a check mark and is
  disabled.
- Same place (extension §5.1): a right click on a palette dock header, its tab or a floating palette
  header opens the menu of all palettes except Tools (`Palette::known`), with check marks; the change
  is a window layout change, the same `closed` in `settings.json`, and reaches the preset through
  Done/Save/Update.
- `README.md` (optional): one sentence in the interface description.
- No release notes are required: the behavior of old files does not change.

## 12. Acceptance criteria

- [ ] A right click on a panel header opens the menu in the ribbon, on a folded panel, in Edit Tools
      and in the Tools palette; in both profiles.
- [ ] The menu contains the panels of the current tab with check marks and **Other Tabs** with the
      panels of the remaining tabs.
- [ ] The check mark matches visibility; a click hides and shows a panel; several panels can be
      toggled without closing the menu.
- [ ] Hiding the panel whose header opened the menu closes the menu; it does not pop up afterwards.
- [ ] A panel hidden by the scale is shown without a check mark and is disabled, with a tooltip.
- [ ] The state reaches `settings.json` immediately; the active preset only through Done/Save/Update.
- [ ] The left click, the tool list, Edit Tools, Find, ribbon fitting and the empty tab are unchanged.
- [ ] The menu and the tool list do not blend (different popup ids).
- [ ] There are no new `settings.json` keys; old files are read.
- [ ] A right click on a palette dock header (both on a stack tab and on a floating palette) opens the
      list of all palettes except Tools, with check marks; a click hides and shows a palette, and
      several can be toggled.
- [ ] There is no Tools row in any palette list: neither in the palette menu, nor in Settings, nor in
      the **Palettes** menu, nor in View > User Interface, nor in the preset editor — just as the Rev
      ribbon is not there either; a layout with a closed Tools is restored by Reset Palette Layout.
- [ ] Hiding the palette whose header opened the menu closes the menu; it does not pop up afterwards.
- [ ] Palette visibility goes the same way as View > User Interface: immediately into
      `settings.json`, into the preset through Done/Save/Update; dragging, tabs, "×" and Dock are
      unchanged.
- [ ] The strings are translated and `cargo i18n check` is green; the §9 tests are green (at the
      customer's request).

## 13. Stages

| Stage | Content | Estimate |
|---|---|---|
| 1 | `panel_groups()`, `known_panel_keys()` through it, second parameter of `panel_rules`, `tab_label` | 0.5 day |
| 2 | `panel_menu()`: current tab, **Other Tabs**, check marks, scale, closing its own popup | 1 day |
| 3 | Wiring to the four headers, popup id, `CloseOnClickOutside`, interactive palette label | 0.5–1 day |
| 4 | Tests, i18n, documentation, checks | 0.5–1 day |

Total — about 2.5–3.5 working days, with no changes to the model or formats.

## 14. Risks and open questions

| Risk | Mitigation |
|---|---|
| A menu of all window panels is taller than the screen | the panels of the current tab are in the menu, the rest under **Other Tabs**; the height is limited by the window |
| The menu and the tool list share a popup on one `Response` | the menu has its own id from the panel key; only one popup is open at a time |
| The menu closes after the first click | `PopupCloseBehavior::CloseOnClickOutside` |
| A hidden panel leaves the popup "open" in memory | close the menu in the hidden-panel branch and after hiding its own key |
| The right click intercepts the tool menu | wiring only to the panel header, not to the whole panel |
| The label in the Tools palette does not accept clicks | `Sense::click` instead of `Sense::hover`; a left click does not change behavior |
| The user expects the menu to change the preset immediately | the menu and the documentation say: the window changes immediately, the preset through Done/Save/Update |
| A panel is hidden by the scale and the user looks for its check mark | the row is visible, without a check mark, with a tooltip; the scale is changed with the chip |
| Hiding a panel of another tab goes unnoticed | this is an intentional action; the result is visible when the tab is opened, and the key is the same as in Edit Tools |
| A right click on the dock header interferes with dragging the palette | the menu opens on the secondary button; dragging stays on the primary one |
| The own palette is closed but its popup would remain in memory | when the own palette is hidden, the menu is closed with `Popup::close_id`; without a drawn header the popup is not maintained and disappears |

Open questions:

1. Keep **Other Tabs** as a submenu, or, if the customer wants one flat list, show all panels in one
   column with scrolling (the recommendation is a submenu: otherwise the menu is taller than the
   screen).
2. Whether a **Show Every Panel** item is needed at the bottom (clears `panels_hidden` entirely) —
   currently this capability exists in Settings and on the empty tab.
3. Whether an **Edit Tools…** item is needed to go to tool and preset editing — or whether the chip in
   the window title bar is enough.
4. Whether a keyboard invocation of the menu (Shift+F10) on a panel header is needed; in v1 the
   keyboard path is Edit Tools.
5. Whether to show the number of hidden panels in a tab row under **Other Tabs**; in v1 — no.

Decided for implementation: the menu changes only the window and `panels_hidden`, writes through the
existing `toggle_panel` queue, does not touch the preset until Done/Save/Update, and adds no new
`settings.json` keys. For palettes — the same thing through `Docking::show` and the existing
`palettes.closed` list.

## 15. Definition of Done

- [ ] `panel_groups` is the single source of the list; `known_panel_keys` and the preset editor are
      built on it.
- [ ] The menu on all four headers, both profiles, folded ribbon, Tools palette, Edit Tools.
- [ ] Check marks, **Other Tabs**, scale-disabled rows, closing its own popup.
- [ ] The palette menu on the dock header, stack tab and floating palette header; check marks of
      `Palette::switchable()` (all palettes without Tools), closing its own popup, the same
      `settings.json` and preset rule. Tools is absent from the other palette lists as well.
- [ ] The right click breaks nothing: left click, tool menu, Edit Tools, Find, ribbon, dock menu,
      palette dragging and floating windows.
- [ ] i18n, documentation, §9 tests; `cargo check`, `clippy`, `fmt --check`, `i18n check` green.
