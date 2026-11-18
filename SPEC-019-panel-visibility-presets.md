# SPEC-019. Panel visibility and a preset's own palette set

Status: **spec draft**, for discussion
Date: 2026-11-18
Project: BIMCraft (Rust, `crates/ui-egui`, egui 0.36)
Source: user request of 2026-11-18: "panels cannot be enabled or disabled; they are not in Settings;
each preset should keep its own set of palettes; the interface profile should be stored in a preset;
a preset needs a button that resets it to the standard view".
Related documents: `ТЗ-019-panel-visibility-presets.md` (the Russian original of this spec),
`ТЗ-020-plugin-panels.md` / `SPEC-020-plugin-panels.md` (plugin panels, the next PR),
`PANELS-PR-SEQUENCE.md` (why this spec is done first).
Task type: window UI and the `settings.json` format; the model, the project file, commands and the ABI
do not change.

> **In short.** A ribbon panel cannot be hidden or brought back today: Settings toggles five palettes
> only, Edit Tools toggles individual tools, and a panel stays in place even when all of its buttons are
> unticked. A preset formally stores the palette layout, but reading it completes the layout and opens
> palettes the preset never had, the Rev profile rewrites the layout every frame, and the interface
> profile is not part of a preset at all. This spec hides and shows panels from Edit Tools, stores that
> per preset together with palettes, tools, scale and profile, gives every preset its own palette set,
> and adds a per-preset reset to the standard view that does not touch the profile.

## 0. General working rules

1. Communication language — Russian; code, comments, strings in code, console messages and commit
   messages — English. Documentation — English, this file — English as well, because it is the public
   version of the Russian original. User-facing strings go through the existing `msg!`/`t()` helpers and
   the `translations/` catalogs are updated (`cargo i18n update`).
2. Tests (`cargo test`, `cargo nextest`, doctests, any execution of test code) are **not run** in this
   task (rule `~/.config/opencode/rust-rules.md`). Allowed: `cargo check`, `cargo build`, `cargo clippy`
   without `--all-targets`, `cargo fmt --check`, `cargo i18n check`; tests run on CI and on the user's
   machine.
3. Commits are atomic: the layout-reading policy, panel hiding, presets and profile, documentation and
   translations each get their own commit. Every commit builds and does not change the behaviour of an
   old `settings.json` beyond what §8 states.
4. The `settings.json` format only grows: old files are read, new keys are optional, unknown or
   malformed values are dropped instead of breaking the window.
5. Settings are the window's, never the project's: none of this is written into a `.bimc` file.
6. No new dependencies; no new `unsafe` — the only existing one stays in `crates/plugin-sdk` (the
   guest ABI boundary); panics are denied by the lints.

## 1. The essence of the problem

### 1.1. Panels cannot be shown or hidden

Today:

- **Settings > User Interface** (`crates/ui-egui/src/settings.rs:317-352`, `palette_switches` —
  `settings.rs:550-567`) has checkboxes only for the five **palettes**: Properties/Inspector,
  Project Browser/Navigator, Tools, Assistant, Script Automation. There are no ribbon panels there at all.
- **Edit Tools** removes individual **tools**: clicking a button ticks/untick its checkbox,
  and the name is written to `tools_hidden` (`crates/ui-egui/src/lib.rs:1445-1470`, `2413-2419`;
  `crates/ui-egui/src/presets.rs:44-50`).
- **A whole panel** is hidden only by the Sketch/Studio/Team "project scale", and only by a hard-coded
  list of names (`crates/ui-egui/src/presets.rs:58-69`). In `ribbon::panel` the only check is
  `scale_hides(title)` (`crates/ui-egui/src/ribbon.rs:265-272`). You can untick every tool of a panel,
  but the panel itself stays in place with a single caption.

Bottom line: there is no way to say "I don't need this panel", neither in Settings nor in Edit Tools.

### 1.2. Presets and palettes

Formally a preset already stores the palette layout (`Preset { docking, hidden, scale, compact }`,
`presets.rs:74-108`; JSON key `palettes`). But in practice "its own set" does not work out, for three
reasons:

1. **`Docking::from_json` completes the layout.** Anything the file neither places nor closes is put
   on the left side (`dock.rs:399-455`, the comment: "a palette it neither places nor closes … docks
   on the left"). So a preset saved before a palette existed will show it open after an update — and
   differently depending on which palettes are known at read time.
2. **The checkboxes in Settings only edit the current layout.** There is no separate preset editor:
   "a preset's own set" can be achieved only through the chain "apply preset → toggle checkboxes →
   Save Preset (overwrite)", and that is not stated anywhere.
3. **The Rev profile rewrites the layout.** Every frame in the Rev profile runs
   `self.docking.show(Palette::Tools, false)` (`lib.rs:2350-2356`), and `apply_settings` forcibly
   reopens Tools when switching back to BIMCraft (`settings.rs:373-380`). Because of this a saved
   preset depends on which profile it was saved in, and the same preset yields a different palette set
   before and after switching the profile.

### 1.3. The interface profile is not part of a preset

Currently `profile` (BIMCraft/Rev, `style.rs:501-505`) is a global window setting (`settings.rs:98`),
not a property of a preset. A preset conceived as "Rev with a ribbon", when applied in the BIMCraft
profile, shows the ribbon elsewhere in the window and without the Tools palette, and vice versa.
Since the profile determines where the tools live, it must be stored in the preset together with the
layout (customer decision, see R13).

## 2. Goals

1. Teach the window to **hide and show panels** — built-in ones and plugin panels — from Edit Tools,
   not from Settings.
2. Make all of this **part of a preset**: switching a preset changes the palette set, the panel set,
   the tool set, and the scale.
3. Give every preset **its own set of enabled palettes**, independent of other presets, of the order of
   application, and of the interface profile.
4. Store the **interface profile** (BIMCraft/Rev) in the preset, so that a "workspace" is a single whole.
5. Give a preset a **reset-to-standard-view button** — without touching either the profile or the other
   presets, and without deleting the preset itself.

## 3. Non-goals

- Settings remain **window settings**, not project settings (as now: "All of it the window's, kept in
  `settings.json`, never in a project", `presets.rs:1-4`).
- No hiding of panels for individual views, sheets, or projects.
- No user rearrangement of panels within a tab (hide/show only).
- No separate "Panels" page in Settings: the only place is Edit Tools and the preset editor.

## 3.1. How it looks to the user

1. I don't need the Phasing panel: I open **Edit Tools**, untick the header of the Phasing panel,
   and press **Done**. The panel disappears from the View tab; its tools are still reachable through
   Find.
2. I want two workspaces: "Design" and "Documentation". I set up palettes and panels for
   the first one and save the preset **Design**. I switch palettes and panels for the second one and
   save **Docs**.
3. Now switching Design ↔ Docs changes the palettes, the panels, the tools, the scale, and the
   interface profile — in any order, with no "inheritance" of settings from the previous preset.
4. I installed a plugin — its panel appeared in Edit Tools by itself. I unticked it in one preset —
   in another it stayed, because each preset has its own set.
5. I disabled the plugin: the panel is gone, but its checkboxes and entries in presets are preserved.
   I enabled it again — everything is as it was.
6. I ruined a preset with experiments: I open it in the editor and press **Reset to Standard View** —
   it returns to the standard view, while the name, its place in the list, and the **profile** are
   preserved.

## 4. Terms and identifiers

It is important to distinguish two different things:

- a **ribbon panel** (`ribbon::panel`) — a strip of buttons on a tab; hidden through `panels_hidden`;
- a **palette** — a docking window (`dock::Palette`), including a plugin palette from SPEC-020; its
  visibility is already described by the `docking` layout (the `closed` list), and `panels_hidden`
  does not concern it.

**Ribbon panel key** — a string prefixed with a namespace, so there are no ambiguities:

| What | Key format | Example |
|---|---|---|
| Built-in panel | `tab:<tab>/<panel>` | `tab:View/Phasing`, `tab:Architecture/Wall` |
| Context tab | `tab:Modify/<panel>` (it is drawn by the same `modify_tab`) | `tab:Modify/Modify` |
| Plugin tab panel | `tab:plugin:<tab name>/<panel>` | `tab:plugin:Areas/Rooms` |
| Family document panel | `tab:family:<DocTab>/<panel>` | `tab:family:Create/Forms` |
| Plugin palette name (in `docking` only) | `plugin:<plugin id>:<panel id>` | `plugin:rooms:areas` |

Rules:

- `<tab>` — the English name from `TABS` (`lib.rs:1604-1619`); for a plugin, the tab name from the
  manifest. The `plugin:` prefix resolves a plugin tab name colliding with a built-in one ("Manage").
- `<panel>` — the English panel name (`ribbon::panel` receives it as `title`); for a plugin, the
  `panel` string from the manifest.
- A palette name (`plugin:…`) is not written to `panels_hidden`: palettes live in `docking` (the
  `closed` list), and that is a separate namespace — the `tab:` prefix does not intersect with it.
- The key is not translated and does not depend on the interface language.
- Key length — up to 160 characters; up to 512 keys in a set (a palette name — up to 200 characters).
- Panel names are not unique across tabs (`Phasing` exists in View and in Manage), so the tab in the
  key is mandatory.
- Tool names and panel names overlap (`Phasing`, `Design Options`, `View Templates` — both a button
  and a panel), so **the set of hidden panels is separate from `tools_hidden`**.

## 5. Functional requirements

### R1. Hiding a panel

- `ribbon::panel` does not draw a panel if its key is in `panels_hidden` — except in Edit Tools mode.
- A hidden panel does not participate in width fitting (`ribbon::tab`, `FITS`/`SEEN`): the check runs
  before the first `allocate`, so the widths are recomputed on the next frame by themselves.
- Hidden panels take no space either on the ribbon (Rev profile), or in the Tools palette (BIMCraft
  profile), or on a plugin tab.
- Tools of a hidden panel remain available through Find (Ctrl+K) — just like hidden tools today.
- Project scale takes precedence: if a panel is hidden by scale, the user setting does not show it
  (the same as for tools now, `ribbon.rs:110-125`).

### R2. Edit Tools: panel toggle

- In Edit Tools the panel header becomes a checkbox (today it is a button with a list of tools).
- An unticked checkbox hides the panel; in Edit Tools the panel is still drawn — dimmed, with an
  empty checkbox — so that it can be brought back and so that its tools remain available for
  ticking.
- The panel checkbox does not change the checkboxes of its tools (two independent settings).
- Keyboard: the panel checkbox is focused with Tab, toggled with Space; for screen readers the role is
  `Checkbox` and the label is the panel name.
- The state is saved right after **Done**, as for tools (`lib.rs:2413-2419`).

### R3. Empty tab

If all panels of a tab are hidden, the tab body shows a single hint line and a button:

> Every panel of this tab is hidden. — [Show the panels]

The button unhides all panels of that tab. The line and the button are translated.

### R4. Storage (top level)

- `BimApp::panels_hidden: BTreeSet<String>`, `Settings::panels_hidden`.
- JSON key `panels_hidden` (array of strings), next to `tools_hidden` (`settings.rs:176-181`).
- Reading — as for `tools_hidden` (`settings.rs:226-231`): if the field is missing or is not an array —
  empty; garbage/overlong/superfluous values are dropped, the set is truncated to 512.

### R5. Preset: composition

- `Preset` gets `panels_hidden: BTreeSet<String>` and `profile: Option<Profile>`; `to_json`/`from_json` —
  keys `panels_hidden` and `profile` (next to `tools_hidden`, `presets.rs:88-108`).
- `apply_preset` applies in the order: **`profile`** → `docking` → `panels_hidden` → `tools_hidden` →
  `scale` → `compact`. The profile comes first because the way the layout is drawn depends on it (the
  profile rule, R7) as do the palette captions.
- `save_preset` captures the current state in full (including panels and profile).
- An old preset without `panels_hidden` reads with an empty set; without `profile` — with `None`
  ("do not change the profile") — nothing breaks.

### R6. Palettes: each preset's own set

- `Docking::from_json` gets a policy:
  - `Policy::Live` — the current one (for the top-level `settings.json`): an unknown palette is opened,
    so that a new version does not "lose" a palette for the user;
  - `Policy::Preset` — **exactly as recorded**: whatever is neither placed nor closed is closed;
    unknown names are skipped.
- `Preset::from_json` uses `Policy::Preset`; `Settings::from_json` uses `Policy::Live`.
- `Docking::to_json` must write `closed` explicitly and in full (it does now, `dock.rs:388-397`);
  additionally, `"palettes_known": true` is added to the preset for format clarity — optional,
  the implementation decides.
- Consequence: a palette that appeared later (new in BIMCraft or from a plugin) is **closed** in an old
  preset, not open; that is exactly "its own set".
- The order of applying a preset does not affect the result: applying means replacing `docking` as a
  whole, not gradually enabling/disabling.

### R7. The profile does not rewrite the layout

- Remove `self.docking.show(Palette::Tools, false)` from the frame (`lib.rs:2350-2356`).
- Introduce a "profile rule at draw time": in the Rev profile the Tools palette is not drawn and takes
  no space, but remains in `docking` as the user's intent.
- Remove the `apply_settings` crutch (`settings.rs:373-380`) that reopens Tools.
- Requirement: switching the profile back and forth does not change `settings.json` or the presets; a
  preset saved in Rev, when applied, sets the Rev profile (R13) and shows exactly the layout it was
  saved with — with no "add-ons" and no losses.

### R8. Preset editor

The preset editor **does not change the current window**: it edits the saved set. To see it,
one must press **Use** (or select the preset in the chip on the title bar). The current layout is
edited only by Edit Tools and ordinary palette dragging — this is how the requirement "enable/disable
panels in Edit Tools, not in Settings" is preserved.

In Settings > User Interface, the Presets section (`presets.rs:188-230`):

- each preset has buttons: **Use**, **Edit**, **Delete**;
- **Edit** opens the dialog "Preset: <name>":
  - "Profile" — radio BIMCraft / Rev (R13);
  - "Project scale" — radio Sketch/Studio/Team;
  - "Palettes" — checkboxes for all known palettes (including `Palette::Plugin`);
  - "Panels" — a tree tab → panel, with checkboxes (all known panels, including plugin panels);
  - "Tools" — a counter of hidden ones and a button that opens Edit Tools for this preset (optional
    in v1: it is acceptable to keep only the counter and the text "edit Tools after Use");
  - **Update from Current Layout** — overwrite the preset with the current state;
  - **Reset to Standard View** — reset the preset's content to the standard view without touching the
    profile (R14);
  - **OK** / **Cancel**; the changes are applied to the preset, not to the window.
- The dialog's toggles (including "Profile") change only the draft: neither the window, nor its
  profile, nor the layout changes until **OK**, and after that — only on the next **Use** (or when
  selecting the preset in the chip).
- The preset currently applied is marked, but this is an optional decoration.
- A preset row in the list shows a summary: `«BIMCraft · 3 palettes · 2 panels hidden · Sketch»`.

### R9. Plugin panels and palettes

- Plugin **ribbon panels** (today — from `functions[].tab`/`.panel`, after SPEC-020 — plus declared
  panels) appear in Edit Tools and in the "Panels" section of the preset editor automatically as soon
  as the plugin is loaded.
- **Plugin palettes** (SPEC-020) appear in `dock::Palette` and behave like built-in palettes: they are
  in the layout, in `closed`, in the "Palettes" section of the preset editor, and in the
  Settings > User Interface toggles. They do not get into `panels_hidden`.
- Panel keys of a disabled/removed plugin are **not deleted** either from `panels_hidden`, or from
  `presets[].panels_hidden`, or from `docking`: bring the plugin back — and its state comes back too.
- If a plugin renamed a panel — the old name's state remains "dangling" and affects nothing:
  this is not an error, and such keys need not be deleted automatically (the plugin may return).
- An empty panel set for a plugin with no declared panels is normal, nothing is drawn.

### R10. "Show everything"

- In the Presets section, next to "Show Every Tool" there appears **Show Every Panel** (clears
  `panels_hidden`).
- In Edit Tools, a "Show Every Panel" button/line is not needed: checkboxes are enough there.

### R11. Nothing breaks

- Find, tool hotkeys, `Tab::Context`, ribbon storyboarding (`ribbon::tab`), moving tools into the panel
  list when width is short — all work as now.
- ABI 1 plugins without panels are unaffected.
- CLI and MCP are unchanged: window settings do not affect them.

### R12. Accessibility and appearance

- Panel checkboxes have the `Checkbox` role and a name label; in Edit Tools a panel with several
  tools does not "collapse" into an empty button.
- Hint: "Tick the panels this preset shows; Find still has their tools".
- Everything in both profiles and in the dark theme.

### R13. The interface profile is part of a preset

- `Preset.profile: Option<Profile>`; in JSON — `"profile": "bimcraft" | "rev"`.
- `None` (an old preset) — the profile is not changed when applying: a file from a previous version
  behaves as before.
- `Some(p)` — `apply_preset` sets `self.profile = p` first, before the layout. The profile is part of
  `Look`, which is applied at the start of the frame (`style::follow`, `lib.rs:2260`), so the change is
  visible immediately, without a restart and without recreating the window.
- `save_preset` (and **Update from Current Layout**) always writes the window's current profile — new
  presets always "remember" their look. **OK** in the editor writes the profile as in the draft: for an
  old preset it stays unset (`None`) until the user explicitly selects a profile in the dialog.
- The profile toggle in Settings > User Interface remains and still changes the **current** window;
  it updates the preset only via the Save/Update button — just like the other settings.
- The block `if before != self.profile && self.profile == Profile::BimCraft { docking.show(Tools, true) }`
  (`settings.rs:570-582`) is removed: the layout is no longer "repaired" on a profile change (see R7).
- Consequence for R7: the profile rule (in Rev the Tools palette is not drawn) is applied at draw time
  and is not written to the file. Therefore a preset stores Tools as intent: if the same preset is
  switched to the BIMCraft profile in the editor, Tools will be where the user left it.

### R14. A preset's "Reset to Standard View"

- In the preset editor — a **Reset to Standard View** button. It resets **the preset's content**, not
  the window, and does not delete the preset: the name and position in the list are preserved.
- **The profile is not changed.** The button does not touch `profile`: the preset stays a Rev preset or
  a BIMCraft preset, whichever it was; for an old preset without a profile it also stays unset. The
  profile is selected in the dialog by separate radio buttons, and the reset must not override that
  explicit choice.
- What becomes standard:
  - `docking` — the standard layout **for the preset's profile**: `Docking::default()` for BIMCraft
    (Tools on the left, Inspector above Navigator on the right) and `Docking::revit()` for Rev; if the
    profile is unset (`None`), the standard of the window's current profile is taken;
  - `panels_hidden` — empty (all panels visible);
  - `tools_hidden` — empty (all tools visible);
  - `scale` — `Scale::Team` (the default, "nothing is lost");
  - `compact` — `false` (floating layout, as in `Docking::default()`).
- Plugin palettes are **closed** on reset: the standard view is the view without them; the plugins
  themselves stay enabled, and their panels can be reopened in the "Palettes" section.
- After the reset the dialog stays open: the reset can be adjusted with checkboxes and confirmed with
  **OK**; **Cancel** cancels the reset too (the reset is a change of the preset draft, not an
  immediate write).
- The **Reset Palette Layout** button in Settings > User Interface remains for the **current** window:
  these are two different actions — "reset the window" and "reset the saved preset".
- There is no separate "reset all presets" button: a mass reset is deleting the presets you don't need.

## 6. Data model

`settings.json` (top level), new/changed keys:

```json
{
  "tools_hidden": ["Door"],
  "panels_hidden": ["tab:View/Phasing", "tab:plugin:Areas/Rooms"],
  "palettes": { "left": [...], "right": [...], "bottom": [...], "floats": [...], "closed": ["Assistant"], "home": {...} },
  "presets": [
    {
      "name": "Documentation",
      "profile": "rev",
      "palettes": { "...": "...", "closed": ["Tools", "Assistant", "Script Automation", "plugin:rooms:areas"] },
      "panels_hidden": ["tab:Architecture/Wall", "tab:View/Phasing"],
      "tools_hidden": ["Door"],
      "scale": "team",
      "layout": "floating"
    }
  ]
}
```

Limits and resilience to a "hostile" file:

| What | Limit |
|---|---|
| `panels_hidden` | 512 keys, key ≤ 160 characters |
| `presets[].panels_hidden` | the same |
| Unknown keys | preserved as is (for plugins that are not present right now) |
| Unknown palette in a preset | skipped |
| Duplicates | collapsed |

## 7. Integration points (`main` branch, `046b357`)

| File | What to change |
|---|---|
| `crates/ui-egui/src/ribbon.rs` | `CURRENT_TAB` (thread-local), `panel_key()`, the `panels_hidden` check in `panel()`, `toggle_panel()`, `take_toggled_panels()`, the header checkbox in Edit Tools |
| `crates/ui-egui/src/lib.rs` | `panels_hidden` in `BimApp`; setting the current tab in `tab_content` (`4812-4830`) and `family_ribbon`; `take_toggled_panels` after the frame (next to `2413-2419`); removing the `docking.show(Tools,false)` mutation (`2350-2356`); the panel header as a checkbox |
| `crates/ui-egui/src/presets.rs` | `Preset.panels_hidden`, `Preset.profile`, `to_json`/`from_json`, `apply_preset` (profile first), `save_preset`, `Preset::from_json` with `Policy::Preset`, the preset editor, "Reset to Standard View", "Show Every Panel", the summary |
| `crates/ui-egui/src/dock.rs` | `Policy` in `Docking::from_json`, `Palette::known()`, the `known` parameter for `named`/serialization (before SPEC-020 — built-ins only) |
| `crates/ui-egui/src/style.rs` | nothing new: `Profile` is already here (`501-505`); the preset merely stores the value |
| `crates/ui-egui/src/settings.rs` | `panels_hidden` in `Settings`, `to_json`/`from_json`, the `presets_section` section, remove the Tools reopening (`373-380`) and the "profile brought Tools back" block (`570-582`), add the profile to the preset editor |
| `crates/ui-egui/src/tests/*` | new tests (see §9), update `tests/presets.rs` |
| `docs/areas/desktop.md`, `docs/areas/plugins.md` | description of Edit Tools and presets |

## 8. Migration and backward compatibility

- An old `settings.json` without `panels_hidden` → an empty set, everything visible (as today).
- An old preset without `panels_hidden` → an empty set, everything visible.
- An old preset without `profile` → `None`: applied to the current profile, as before. Presets saved
  after the update always record the profile.
- An old preset with `"palettes"` but without new palettes → the new palettes are **closed** (a
  behavior change, intentional). This must be written in the release notes: "a preset now stores
  exactly the palette set it was saved with".
- A user who currently has one preset will not notice a difference.
- Presets saved in the Rev profile, after the update, show Tools the way they were meant to be, not
  "the way the profile decided".

## 9. Test plan

Unit tests (`crates/ui-egui/src/presets.rs`, `dock.rs`):

- `a_preset_keeps_only_the_palettes_it_was_saved_with` — a preset with Assistant closed and without
  Scripts does not open Scripts after reading;
- `a_palette_added_later_stays_closed_in_an_old_preset`;
- `the_live_layout_still_opens_a_palette_it_does_not_know` — `Policy::Live` has not changed;
- `panels_hidden_reads_back_as_written_and_a_hostile_list_reads_as_sense` — like
  `presets_read_back_as_written_…` (`presets.rs:237-254`);
- `a_preset_without_panels_reads_with_none_hidden`;
- `a_preset_profile_reads_back_and_an_old_preset_leaves_the_profile_alone`;
- `resetting_a_preset_gives_the_standard_layout_of_its_profile_and_keeps_it` — BIMCraft → `Docking::default()`,
  Rev → `Docking::revit()`, the sets are empty, scale Team, `compact` cleared, `profile` unchanged.

Window tests (`egui_kittest`, following the pattern of `crates/ui-egui/src/tests/presets.rs`):

- `a_panel_taken_out_in_edit_tools_leaves_the_tab_and_comes_back` — in both profiles;
- `a_hidden_panel_is_still_listed_in_edit_tools_with_its_tools_tickable`;
- `a_tab_whose_panels_are_all_hidden_says_so_and_the_button_brings_them_back`;
- `a_preset_switches_palettes_panels_tools_and_scale_together`;
- `switching_to_a_rev_preset_turns_the_ribbon_on_and_back_without_rewriting_the_file`;
- `a_preset_reset_to_the_standard_view_keeps_its_name_profile_and_leaves_the_window_alone`;
- `revs_profile_does_not_rewrite_the_saved_palettes_or_the_presets`;
- `a_plugin_panels_state_survives_the_plugin_being_disabled_and_enabled` (overlaps with SPEC-020);
- regression: `the_project_scale_leaves_out_…` and `edit_tools_takes_a_tool_out_…` stay green.

Build and checks: `cargo check --workspace`, `cargo clippy` (without test profiles),
`cargo i18n check`. Tests are run on direct request.

## 10. i18n

New English strings (keys, via `t(...)` / `msg!(...)`, then `cargo i18n update`):

| String | Where |
|---|---|
| `Panels` | section header in the preset editor |
| `Palettes` | same place |
| `Profile` | same place (radio BIMCraft / Rev) |
| `Reset to Standard View` | preset editor button |
| `Reset the preset to the standard view, keeping its profile` | button hint |
| `Show Every Panel` | Settings > Presets, hint |
| `Every panel of this tab is hidden.` | body of an empty tab |
| `Show the panels` | button in the same place |
| `Tick the panels this preset shows; Find still has their tools` | Edit Tools hint |
| `Preset: {}` | dialog title |
| `Update from Current Layout` | dialog button |
| `{} · {} palettes · {} panels hidden · {}` | preset summary (`tf`) |
| `This preset keeps its own palettes, panels and profile` | hint |

Plugin panel and tab names are not translated (plugin data); built-in names are already translated
via `t(title)` in `ribbon::panel`.

## 11. Documentation

- `docs/areas/desktop.md`: Edit Tools — "hides tools and panels"; presets — "their own set of
  palettes, panels, tools, scale, and profile"; switching the profile does not change the saved
  layout; a preset's "Reset to Standard View" and the window's "Reset Palette Layout" are different
  actions.
- `docs/areas/plugins.md`: plugin panels are visible in Edit Tools and in presets; the state of a
  disabled plugin is preserved.
- `README.md` (optional): one sentence in the interface description.
- Release notes: the behavior change of old presets (§8).

## 12. Acceptance criteria

1. A built-in panel and a plugin panel can be unticked and brought back from Edit Tools; the state
   survives a restart.
2. A preset stores and restores **its own** palettes, panels, tools, scale, and **profile**; two
   presets with different palette sets yield different sets after switching in any order.
3. A new palette (or a new plugin's palette) does not appear by itself in old presets.
4. Switching the Rev ↔ BIMCraft profile does not change `settings.json` and does not spoil presets.
5. Applying a preset with the Rev profile turns the ribbon on, and one with the BIMCraft profile turns
   the Tools palette on — without a window restart and without losing the layout.
6. **Reset to Standard View** returns the preset to the standard view (layout and sets — according to
   its profile), preserving the name and the profile itself; the window does not change until **Use**
   is pressed.
7. Hiding a panel does not break Find, ribbon width fitting, and context tabs.
8. An old `settings.json` reads without errors; nothing is lost.
9. All interface strings are translated through the catalogs; `cargo i18n check` is green.
10. The §9 tests are green (on the customer's direct request).

## 13. Stages

| Stage | Content | Estimate |
|---|---|---|
| 1 | `Policy` in `Docking`, a preset's own palette set, migration, tests | 1–2 days |
| 2 | `panels_hidden` (top level), keys, the check in `ribbon::panel`, `tab_content` | 1–2 days |
| 3 | The panel checkbox in Edit Tools, the empty tab, "Show Every Panel" | 1–2 days |
| 4 | `panels_hidden` and `profile` in presets, `apply_preset` (profile first), the summary | 1 day |
| 5 | The preset editor (profile, palettes, panels), "Update from Current Layout", "Reset to Standard View" | 2–3 days |
| 6 | The profile without layout mutation, cleanup of `apply_settings` and the profile block in `interface_page` | 0.5–1 day |
| 7 | i18n, documentation, final tests | 1 day |

Total — about 9–11 working days without SPEC-020. With SPEC-020, stages 2 and 3 get plugin panels
automatically, and no additional work is required apart from tests.

## 14. Risks and open questions

| Risk | Mitigation |
|---|---|
| Plugin panel names change | a key based on `id` (SPEC-020); unknown keys are not deleted |
| There will be many panels, Edit Tools turns into a long list | grouping "tab → panel", collapsed groups, scrolling |
| The user expects hiding a panel to also hide its tools from Find | an explicit hint in Edit Tools: Find still finds everything |
| Old presets will start showing fewer palettes | a release note; optionally — the "Update from Current Layout" button in the editor |
| A hidden panel that holds the only path to a tool | the tool is still available through Find and its hotkey; it is visible in Edit Tools |
| "Reset to Standard View" accidentally overwrites a preset's manual setup | the reset changes the dialog draft, not the file; **Cancel** cancels it too, **OK** confirms |
| The user does not expect a preset to switch the profile | the profile is visible in the preset row and in the editor; in old presets it is unset, so the behavior does not change |

Open questions:

1. Is a full tool list needed in the preset editor (and not just a counter) — it duplicates Edit Tools
   but gives "everything in one window".
2. Is a "reset all panels" command needed for a single tab in that tab's context menu — or is the
   button on the empty tab enough.

Decided by the customer: the interface profile is stored in the preset (R13); a preset has a
reset-to-standard-view button that **does not affect the preset's profile** (R14).

## 15. Execution order: this PR is the first

### 15.1. Why first

| Reason | Explanation |
|---|---|
| Smaller surface | no ABI, no plugin format, no guest and sandbox — only the window and `settings.json` |
| Introduces the data model SPEC-020 relies on | `panels_hidden`, `Policy` in `Docking`, panel keys, the profile in the preset, `Reset to Standard View` |
| Cheaper than doing it the other way round | SPEC-020 will add `Palette::Plugin` and a registry; after it this PR would have to be rewritten: palette enumeration (`Palette::ALL`) lives in `dock.rs`, `settings.rs`, and the preset editor — three places instead of one |
| ABI 2 is a public contract | a manifest with `panels` is better released when the UI rework is already in `main`: otherwise a UI bug found later drags a contract change along with it |
| Immediate benefit | the user gets panel hiding and presets without waiting for plugin panels |

### 15.2. What to do here "with headroom" for SPEC-020 (+0.5 day)

So that the second PR does not rewrite this one, lay down in the first:

1. **A single point of palette enumeration.** Instead of `Palette::ALL` — `Palette::known() -> Vec<Palette>`
   (or `impl Iterator<Item = Palette>`; it currently returns the five built-ins). Returning a slice is not
   allowed: SPEC-020 will add the plugin registry to it, and the signature must not change a second time.
   Use it in `dock.rs` (`to_json`, `from_json`, `named`), in `settings.rs` (`palette_switches`), and in the
   preset editor. SPEC-020 will extend it with the registry without changing the call sites.
2. **A layout-reading signature with a policy and a list of known palettes**:
   `Docking::from_json(v: &Value, policy: Policy, known: &[Palette])`. Here `known` is always
   `Palette::known()`; SPEC-020 will pass the plugin registry there.
3. **Tests — by file.** Keep the layout and presets in `tests/presets.rs` and a new
   `tests/dock_policy.rs`; plugin panel tests (SPEC-020) will go into a separate `tests/plugin_panels.rs`.
   This way the branches will not collide in the same test files.
4. **Do not add** the `Palette::Plugin(..)` variant or the registry — that is SPEC-020's job.

### 15.3. What is deliberately NOT done here

- **Half of R9 about plugin palettes** (their appearance in `palette_switches`, in the "Palettes" section
  of the preset editor, in `closed`) — there is nothing to hook up until `Palette::Plugin` exists. State
  this explicitly in the PR description: "R9: plugin ribbon panels — done; plugin palettes — in SPEC-020".
- Plugin ribbon panels (from `functions[].tab`/`.panel`) do work already: their keys
  `tab:plugin:<tab name>/<panel>` are computed without any ABI 2.

### 15.4. Definition of Done for this PR

- [ ] `panels_hidden` works in both profiles, Edit Tools toggles panels, the state is in
      `settings.json`.
- [ ] A preset stores and restores palettes, panels, tools, scale, and profile.
- [ ] `Policy::Preset` does not open palettes that were not in the preset; `Policy::Live` behaves
      as before.
- [ ] "Reset to Standard View" does not touch the profile and edits the draft, not the window.
- [ ] The profile does not rewrite the layout and does not change `settings.json`.
- [ ] `Palette::known()` and `Docking::from_json(v, policy, known)` are introduced and used everywhere.
- [ ] The §9 tests are green, `cargo i18n check` is green, the documentation is updated.
- [ ] The PR description notes the deferred part: plugin palettes — SPEC-020.

## 16. If the order has to be changed (SPEC-020 first)

Technically possible, but more expensive:

- SPEC-020 will introduce `Palette::Plugin(usize)` and a registry; then this PR would have to:
  - rewrite `Docking::from_json` once again (registry + policy), adding the policy on top of the already
    parameterized reading;
  - rewrite the palette toggles and the preset editor to a dynamic list;
  - think through the migration: users will already have plugin palettes saved in `closed`, and the very
    first version of presets must preserve them, not lose them.
- Expected overrun: **1.5–2 days** plus a regression risk in layout saving.
- If it does go that way: first release SPEC-020 with a `Policy`-compatible signature
  (`Docking::from_json(v, known)`), so that this PR only adds `Policy` as a third parameter.

**Conclusion:** sequentially, **SPEC-019 first, then SPEC-020**. In parallel — only stage A from SPEC-020
(the form from `params`), which does not touch `dock.rs` and `presets.rs`; see SPEC-020 §16.
