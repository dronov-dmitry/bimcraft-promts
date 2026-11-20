# SPEC-008.1. Keyboard shortcut presets: built-in "Revit-like"/"Archicad-like" layouts and saving your own named set in the config

Status: **proposal**
Date: 2026-11-20
Project: BIMCraft (Rust, `crates/ui-egui`, egui 0.36)
Related documents: `ТЗ-008-горячие-клавиши.md` (base spec for the action registry and bindings), `docs/areas/shortcuts.md` (current user documentation), PR #403 (registry and dialog implementation), `crates/ui-egui/src/presets.rs` (existing pattern of named presets for palettes)
Task type: user interface customization (app-level settings), settings dialog, `settings.json` format, migration of existing settings.

> **Relevance.** The request from the PR #403 comment was written before starting sets appeared in the branch. As of 2026-11-20, `shortcuts.rs` already has `Profile::{BimCraft, Revit, Archicad, Empty, Custom}`, and the dialog has profile selection and an "Apply Profile" button. Therefore the "Revit like / Archicad like" item is **already done** and is not redone in this spec. What remains is the second half of the request: **save your own set under a name in the config and come back to it**. In addition, applying a profile currently silently overwrites the set the user was editing — this must be fixed along the way.

---

## 0. General working rules

- Commits are atomic, one item per commit; messages follow the branch style (an imperative-mood subject, a body that explains *why* rather than *what*).
- The language of code, identifiers, code comments, and commit messages is English. This file and the user documentation in `docs/` are Russian (or English if the repository requires English documentation).
- All UI strings go only through `t()` / `tf()` / `msg!()` from `crates/ui-egui/src/lang.rs`; new strings must be added to `translations/ui/bimcraft-ui.pot` and to the translations.
- Key presets are application settings, not project settings: they live in `settings.json`, survive New/Open, and never end up in `.bimcraft`.
- Do not break the existing starting sets or the current `version: 2` format: presets are added as **optional fields** that older builds ignore.
- Every behavior change must be covered by tests; for the UI — `egui_kittest`, for settings and the model — regular unit tests.
- Do not use `subagent` or parallel agents (user rule); work happens in the current session.

---

## 1. Context and task

### 1.1. Original request

`dansity`'s comment in PR #403 (`#issuecomment-7041237508`):

> Hi @dronov-dmitry, a quicky: can we add presets to shortcuts? "Revit like" "Archicad like" and maybe saving your custom into a preset that is stored in config.

So the user asks for three things, of which the first is already done:

1. a ready-made "Revit-like" set — **exists** (`Profile::Revit`);
2. a ready-made "Archicad-like" set — **exists** (`Profile::Archicad`, plus `ARCHICAD_CHORDS` with `Ctrl+D` = Move);
3. saving your own set as a named preset in the config — **missing**.

### 1.2. What exists today (per the `BIMCraft-keys` code)

- `crates/ui-egui/src/shortcuts.rs`:
  - `Profile` — the starting set: `BimCraft`, `Revit`, `Archicad`, `Empty`, `Custom`; `Profile::CHOICES` holds the four selectable sets (without `Custom`).
  - `Shortcuts::defaults(profile)` assembles a set: action bindings from the registry, then `DEFAULT_CHORDS`, then (for Archicad) `ARCHICAD_CHORDS`.
  - `Config { version, profile, bindings }` — what lives in `settings.json`; `bindings` is the **full effective set**, not a diff from the starting profile.
  - The `ShortcutsDialog`:
    - a `profile_pick: Profile` field;
    - the "Apply Profile" button does `pending = Shortcuts::defaults(profile_pick)` — a **full replacement of the editable set with no confirmation and no way back**;
    - "Reset All" restores the starting set of the selected profile;
    - "Apply Changes" writes `pending` to `settings.json`.
- `crates/ui-egui/src/settings.rs`:
  - `Settings.shortcuts: crate::shortcuts::Config` — a single set, with no history and no named copies;
  - `MAX_FILE = 1 MiB`, `MAX_SHORTCUTS = 4096`, `MAX_ACTIONS = 512`;
  - `presets: Vec<Preset>` — an **already existing** mechanism of named presets for palettes and layout (`presets.rs`: `MOST = 32`, name up to 64 characters, unique names, `to_json`/`from_json`).
- `crates/ui-egui/src/shortcuts.rs`:

  ```rust
  pub struct Config {
      pub version: u32,
      pub profile: Profile,
      pub bindings: Vec<(String, Vec<String>)>,
  }
  ```

  `Config::from_json` reads only `version`, `profile`, `bindings`; the remaining keys of the object are ignored. This is exactly the extension point that does not break older builds.

### 1.3. What the gap is

1. **"Custom" is a label, not a preset.** `Profile::Custom` means "the set differs from the starting one", but it stores neither a name nor a copy of the set. The user cannot return to a successful configuration after experimenting.
2. **Applying a profile is destructive.** Pressing "Apply Profile" replaces `pending` entirely; if the user changes their mind, the previous set can only be brought back by cancelling the whole dialog, and after "Apply Changes" — not at all.
3. **There is no way to hand your set to a colleague** as a named preset without exporting the entire bindings file.
4. **There is no support for multiple working layouts.** The same user may want an "office Archicad set", a "home minimum", and a "drafting set" — right now switching between them is impossible without losing them.

### 1.4. What is not in this spec

- New starting sets beyond the existing four.
- Changing the semantics of `Profile` as the "origin" of a set for resetting a single action.
- Family/Project scopes (`SPEC-011`) — key presets stay in the project scope until scopes exist.

### 1.5. User scenarios

- As a user, I set up my keys, press "Save Current as Preset…", enter the name "My Archicad", and later return to that set with one button.
- As a user, I switch between "Revit-like", "Archicad-like", and my own preset without losing any of them.
- As a user, I rename the preset "My Archicad" to "Work" — the bindings do not change.
- As a user, I delete an old preset; my current set does not change.
- As a user, I restart the program and see my presets in the same places.
- As a user, I accidentally pressed "Apply" on someone else's preset — the dialog asked for confirmation first, and "Cancel" restored everything as it was.
- As a user, I do not want the built-in "Revit-like"/"Archicad-like" ones to be accidentally deleted or renamed.

---

## 2. Terms

| Term | Meaning |
|---|---|
| **Starting set** | A built-in binding set defined by code: `BimCraft`, `Revit-like`, `Archicad-like`, `Empty`. Corresponds to `shortcuts::Profile`. Cannot be edited or deleted. |
| **Preset** | A named copy of a full binding set, saved by the user in `settings.json`. Can be applied, renamed, deleted. |
| **Current set** | The effective bindings that are active right now and live in `Settings.shortcuts.bindings`. |
| **Preset base (`base`)** | The starting set the preset originated from. Affects "Reset Action" after applying the preset and the "Based on…" label. |
| **Active preset** | The preset the current set came from, if the user has not changed bindings since applying it. An optional label. |
| **Built-in preset** | A preset available only for applying, not for editing: the four starting sets. |
| **User preset** | A preset from `settings.json` created by the user. |

> **Terminology caution.** The code already has three different `Profile`s: `shortcuts::Profile`, `style::Profile` (window appearance), and `presets::Preset` (palette layout). Name the new type `ShortcutPreset` to avoid confusion with them; in the UI — "Presets" / "Пресеты".

---

## 3. Functional requirements

### 3.1. Data model

1. The current set stays in `bindings` and is still stored **in full**. This guarantees that an old build unaware of presets will open `settings.json` and get exactly the bindings that were there.
2. Presets are stored in the same `shortcuts` object, in the optional `presets` array.
3. Each preset is a **full snapshot** of the set, not a diff:
   - the snapshot reproduces exactly what the user saved, even after a program update;
   - the absence of an action in the snapshot means "the action has no bindings", not "take it from the starting set";
   - the order of bindings within an action is preserved.
4. Diffs relative to `base` are **not used**: they cause unexpected "creeping in" of new starting keys after an update and complicate applying. The size of a full snapshot is acceptable (see 3.11).
5. Required preset fields: `name` (non-empty, ≤ 64 characters, unique case-insensitively among all presets, including built-in ones) and `bindings`. The `base` field is optional and defaults to `custom`.
6. A preset does not store localized action names — only stable IDs (`cmd.*`, `ui.*`, `legacy.*`).

### 3.2. Built-in presets

1. The preset list shows four built-in ones: `BIMCraft`, `Revit-like`, `Archicad-like`, `Empty`. They are generated from `Shortcuts::defaults(profile)` and are **not duplicated** in `settings.json`.
2. Built-in presets can only be applied. "Rename" and "Delete" are unavailable for them, with the tooltip "Built-in presets cannot be renamed or deleted".
3. Built-in presets are localized through `t()`; user names are never translated.
4. If the user saves their preset with a name matching a built-in one (case-insensitively), it is rejected with a message that the name is taken.

### 3.3. Saving the current set as a preset

1. The presets block has a "Save Current as Preset…" button.
2. The dialog asks only for a name. It shows the label "Based on: <current profile>" (informational only; the base is recorded automatically).
3. The name is trimmed; an empty name, a name longer than 64 characters, and a name consisting only of spaces are rejected with a message, and no preset is created.
4. If the name is already taken by a user preset — confirmation "A preset named “{}” already exists. Replace it?". If accepted, the old copy is replaced; if declined, nothing changes.
5. Saving captures the **current editable state** (`pending`), not the last applied one. That is, unsaved edits in the dialog also go into the preset; this is stated explicitly in the button tooltip.
6. Saving a preset does not change the current set and does not require "Apply Changes".
7. The preset is saved to `settings.json` immediately (through the existing `keep_settings`) so that it survives closing the dialog and an application crash.
8. There are no more than 32 presets (like `presets::MOST`); when attempting to create the 33rd — the message "That is as many presets as are kept: delete one first".

### 3.4. Applying a preset

1. The presets block has a preset selector and an "Apply Preset" button.
2. Applying replaces `pending` entirely; `pending.profile` becomes the preset's `base` (the starting set itself for a built-in one, the saved base for a user one).
3. If `pending` has edits not yet applied with "Apply Changes", or already applied but differing from the selected preset, the confirmation "Replace the set being edited with “{}”?" is shown. "Replace" replaces, "Cancel" leaves it as it was.
4. Applying a preset is a change to `pending`, not an immediate write: the final save is performed by "Apply Changes". "Cancel"/`Esc` rolls back everything, including the applied preset.
5. After applying, the preset is marked active. Any subsequent binding edit clears the "active" mark (the set now differs from the preset).
6. If the preset contains an action that does not exist in the current build (a file from a newer version), such an action is skipped with a warning on apply; there must be no conflicts or crashes.
7. If the preset contains conflicting bindings (a corrupted or hand-edited file), only the first valid ones are applied; the conflict is shown in a warning, and the current set is not partially corrupted: the application either applies the valid part or, on a structural error, refuses to apply entirely.

### 3.5. Renaming, deleting, duplicating

1. "Rename Preset…" is available only for user presets: the new name goes through the same checks as on creation; if it matches another preset — rejection.
2. Renaming does not change bindings and does not clear the "active" mark if the preset was active.
3. "Delete Preset" is available only for user presets; deletion requires confirmation "Delete the preset “{}”?".
4. Deleting the active preset does not change the current set; the "active" mark is simply cleared.
5. "Duplicate" (P2, optional): creates a copy of a user or built-in preset with the name "<name> copy"; if the name is taken — "<name> copy 2", and so on.

### 3.6. Behavior at startup

1. At startup the application reads `bindings` and applies them as usual; presets do not affect how keys work.
2. There is **no** separate "apply a preset at startup" setting: the current set is already saved in full, and re-applying a preset adds nothing but the risk of unexpectedly overwriting edits. This is a deliberate decision; if needed, it can be added in a separate spec.
3. If the `presets` array is missing or corrupted, the application starts with the current set, shows only built-in presets, and writes a warning to the log; `bindings` are left untouched.

### 3.7. Compatibility and migration

1. The schema stays `version: 2`. Do **not** bump the version: the current `Config::from_json` returns the base set when `version != 2`, which means an older build would silently lose the user's bindings. The optional `presets` and `active_preset` fields are ignored by older builds.
2. A `settings.json` without `presets` is read as today: the current set plus the four built-in presets.
3. The old format `{"WA": "Wall"}` migrates as today; after migration `presets` is empty and `profile` = `custom`.
4. On reading:
   - a non-array `presets` is ignored;
   - a preset without a name, with an empty or too-long name is skipped;
   - a duplicate name (case-insensitive) is skipped, the first one is kept;
   - `base` with an unknown code → `custom`;
   - broken bindings inside a preset are skipped one by one, the preset is kept;
   - more than 32 presets — the extras are skipped.
5. Writing remains atomic (the existing `settings.rs` mechanism).
6. Round-trip: `to_json` → `from_json` yields the same presets in the same order.

### 3.8. Interaction with import and export

1. Export to `.bks`/`.txt` and `.json` still exports **one current set**; presets do not go into the file. It is a backup of the set, not of the whole application.
2. "Replace everything" import replaces the current set; presets are not deleted and do not change.
3. "Add to the current set" import adds bindings; presets do not change.
4. P2: in the import preview, a "Save as Preset…" button — save the imported set under a name without applying it to the current one.
5. P2: a separate export of all presets to JSON (`bimcraft-shortcut-presets.json`) for moving to another machine; importing such a file adds presets, and name conflicts are resolved with a suffix.
6. Markdown help: show the active preset (if any) and its base in the header; the table still shows only the effective bindings.

### 3.9. Interface

1. The existing "Profile + Apply Profile" block is replaced by a "Keyboard presets" block ("Пресеты клавиш") so as not to be confused with the layout presets on the title chip:
   - a dropdown with two sections: "Built-in" (the four starting sets) and "My presets" (user ones);
   - an "Apply Preset" button;
   - a "Save Current as Preset…" button;
   - "Rename Preset…" and "Delete Preset" — enabled only for user presets;
   - "Duplicate" — P2.
2. Next to the active preset an "active" label is shown; if the current set differs from the active preset — "modified".
3. The name saving dialog:
   - one field, autofocus, `Enter` confirms, `Esc` cancels;
   - shows "Based on: …";
   - errors (empty, too long, taken) are shown below the field, the dialog does not close.
4. Confirmations (replacing an existing preset, renaming to a taken name, deleting, replacing the set being edited) — built-in confirmation groups in the style of the existing `confirm_reset_all`, without system modal windows.
5. All new strings go through `t()`; user names are displayed as is.
6. The block layout must not "jump" when switching presets: a fixed-width list, buttons in one row, wrapping when there is not enough space.
7. Accessibility: buttons and the dropdown have meaningful accessible names (as already done for binding chips) so that `egui_kittest` can find them by label.

#### 3.9.1. Where keyboard presets are shown

The only place is the **Keyboard Shortcuts dialog** (Manage > Shortcuts, `KS`, `Ctrl+Shift+K`), a block in place of the current "Profile + Apply Profile".

Why not elsewhere:

| Place | What is there now | Why keyboard presets do not go there |
|---|---|---|
| Title bar chip (`presets.rs::scale_chip`, `lib.rs:4473`) | Project scale and layout presets: palettes, hidden tools, compact | A single chip would start changing two unrelated things: the window layout and the keyboard. Besides, it is already labeled "Presets" — the user would not understand which presets are meant. |
| Settings > User Interface (`settings.rs::presets_section`) | The appearance `Profile` (BIMCraft/Rev) and layout presets | `Profile` here is appearance, not keys. A third preset list would create a desync with the shortcuts dialog. |
| Status bar / title | Project name, view | The place is too narrow for a list; the active preset label is already shown in the dialog itself. |

Consequences:

1. The block header is **"Keyboard presets"** / "Пресеты клавиш", not "Presets", so as not to match the layout presets on the title chip.
2. Built-in presets (`BIMCraft`, `Revit-like`, `Archicad-like`, `Empty`) and user presets — in one dropdown with "Built-in" and "My presets" sections.
3. Quick switching without opening the dialog is **not in this iteration**; if needed, it is added as a Manage > Shortcuts > Preset submenu (see open questions).
4. The "active"/"modified" label lives next to the dropdown, in the same block.

### 3.10. Localization

New strings (English source, translations go into the catalogs):

- "Keyboard presets" (block header; not "Presets", to avoid confusion with layout presets)
- "Built-in"
- "My presets"
- "Apply Preset"
- "Save Current as Preset…"
- "Rename Preset…"
- "Delete Preset"
- "Duplicate Preset"
- "Preset name"
- "Based on: {}"
- "Built-in presets cannot be renamed or deleted"
- "A preset needs a name, of at most 64 letters"
- "A preset named “{}” already exists. Replace it?"
- "Delete the preset “{}”?"
- "Replace the set being edited with “{}”?"
- "That is as many presets as are kept: delete one first"
- "Preset “{}” applied to the pending set."
- "Preset “{}” saved."
- "active", "modified"

### 3.11. Limits and performance

| Parameter | Value | Rationale |
|---|---|---|
| Presets total | 32 | Same as `presets::MOST`, familiar to the user. |
| Name length | 64 characters | Same as `presets::NAME`. |
| Bindings in a preset | 4096 | `MAX_BINDINGS`. |
| Bindings across all presets | 8,192 | Estimate: ~300 actions × ~1.2 bindings ≈ 360 bindings per snapshot; 32 snapshots ≈ 11,500 do not fit the budget, while 22 typical ones do. The limit guarantees that `settings.json` together with the current set stays below 1 MiB. |
| `settings.json` size | unchanged, 1 MiB | `settings::MAX_FILE`. |
| Preset apply time | < 20 ms | A single pass over the array and building hash tables. |

If the overall limit is exceeded, saving a new preset is rejected with a message; existing presets and the current set are unaffected.

---

## 4. `settings.json` format

```json
{
  "shortcuts": {
    "version": 2,
    "profile": "custom",
    "bindings": [
      {"action": "cmd.move", "keys": ["Ctrl+Shift+M", "MV"]},
      {"action": "cmd.wall", "keys": ["WA"]},
      {"action": "ui.view.home", "keys": ["Ctrl+D"]}
    ],
    "active_preset": "Work keys",
    "presets": [
      {
        "name": "Work keys",
        "base": "archicad",
        "bindings": [
          {"action": "cmd.move", "keys": ["Ctrl+D", "MV"]},
          {"action": "cmd.wall", "keys": ["WA", "Ctrl+Shift+E"]}
        ]
      },
      {
        "name": "Drafting minimum",
        "base": "custom",
        "bindings": [
          {"action": "cmd.wall", "keys": ["WA"]}
        ]
      }
    ]
  }
}
```

Rules:

- `presets` and `active_preset` are optional; their absence is a valid state.
- `active_preset` is only a label for the interface; if the name is not found, the label is ignored.
- Preset order = order in the array (do not sort), so the user sees a familiar list.
- `base` is a `shortcuts::Profile` code (`bimcraft`, `revit`, `archicad`, `empty`, `custom`); an unknown code → `custom`.
- Older builds read `version`, `profile`, `bindings` and do not notice the new fields.

---

## 5. Architecture and change points

### 5.1. New types (`crates/ui-egui/src/shortcuts.rs`)

```rust
/// A named snapshot of a full binding set, kept in settings.json.
#[derive(Clone, Debug, PartialEq)]
pub struct ShortcutPreset {
    pub name: String,
    pub base: Profile,
    /// Action id to canonical binding strings, as in Config::bindings.
    pub bindings: Vec<(String, Vec<String>)>,
}

impl ShortcutPreset {
    /// Snapshots `s` under `name`, with `base` as its origin.
    pub fn from_shortcuts(name: &str, base: Profile, s: &Shortcuts) -> ShortcutPreset;
    /// Rebuilds a Shortcuts set; unknown actions and bad bindings are skipped with warnings.
    pub fn to_shortcuts(&self) -> (Shortcuts, Vec<String>);
    pub fn to_json(&self) -> Value;
    pub fn from_json(v: &Value) -> Option<ShortcutPreset>;
}
```

```rust
pub struct Config {
    pub version: u32,
    pub profile: Profile,
    pub bindings: Vec<(String, Vec<String>)>,
    /// The preset the current set came from, if the user has not edited it since.
    pub active_preset: Option<String>,
    /// The user's named snapshots; built-in presets are generated, not stored.
    pub presets: Vec<ShortcutPreset>,
}
```

New functions:

- `pub const MAX_PRESETS: usize = 32;`
- `pub const MAX_PRESET_BINDINGS_TOTAL: usize = 16_384;`
- `pub fn builtin_presets() -> [Profile; 4]` — the existing `Profile::CHOICES`;
- `pub fn all_preset_names(config: &Config) -> Vec<(String, bool)>` — the name and a "built-in" flag;
- `Shortcuts::save_preset(&mut self, name, base)` / `rename_preset` / `delete_preset` / `apply_preset` — either at the `Config` level or at the `BimApp` level; choose one point and do not duplicate.

### 5.2. Dialog

`ShortcutsDialog` gains fields:

```rust
pub preset_pick: PresetChoice,          // BuiltIn(Profile) | User(String)
pub preset_name: String,                // input field on save/rename
pub preset_action: Option<PresetAction>, // SaveAs | Rename | Delete | Duplicate
pub confirm_preset_apply: Option<PresetChoice>,
pub confirm_preset_delete: Option<String>,
pub confirm_preset_overwrite: Option<String>,
pub preset_note: Option<String>,        // error or confirmation below the field
pub active_preset: Option<String>,
```

`PresetChoice` is rendered in the dropdown in sections; `pending` remains the single source of the editable set.

### 5.3. Settings

`crates/ui-egui/src/settings.rs`:

- `Settings.shortcuts` is already `Config`; add reading/writing of `presets` and `active_preset` to `Config::to_json`/`from_json`.
- Limits are enforced in `from_json` (not only in the UI) so that a corrupted file does not bloat memory.
- Do not change `MAX_FILE`; check the overall binding budget on serialization.

### 5.4. Files that change

| File | What to do |
|---|---|
| `crates/ui-egui/src/shortcuts.rs` | `ShortcutPreset`, `Config` fields, reading/writing, limits, dialog presets block, confirmations. |
| `crates/ui-egui/src/settings.rs` | Pass `presets`/`active_preset` through `shortcuts_from` (the type is already `Config`, minimal edits). |
| `crates/ui-egui/src/lang.rs`, `translations/ui/*` | New strings from 3.10. |
| `crates/ui-egui/tests/*` | `shortcuts_presets.rs` (model + settings), `shortcuts_dialog.rs` (kittest). |
| `docs/areas/shortcuts.md` | Section "Profiles" → "Starting sets and presets"; describe saving, applying, limits. |
| `README.md` | The keyboard shortcuts line: mention presets. |

### 5.5. What does not change

- The action registry, IDs, executors.
- The `.bimcraft`/`.bimc` project format.
- The current starting sets and their contents.
- The `.bks`/`.txt` format and the import/export rules of the current set.
- The behavior of `Ctrl+C/X/V`, sequences, and physical keys.

---

## 6. Tests

### 6.1. Model and settings

1. `ShortcutPreset` round-trip: snapshot → JSON → snapshot yields the same names, base, and bindings in the same order.
2. `Config` round-trip with and without presets.
3. A `settings.json` without `presets` is read, presets = built-in only.
4. A non-array `presets` is ignored, `bindings` are preserved.
5. A preset without a name / with an empty name / with a name longer than 64 characters is skipped.
6. Duplicate name: the first is kept, the second is skipped.
7. More than 32 presets: the extras are skipped, the first 32 are kept.
8. Unknown `base` → `custom`.
9. An unknown action inside a preset is skipped, the remaining bindings are applied.
10. A broken binding inside a preset is skipped, the preset is kept.
11. Exceeding the overall binding budget: the new preset is not saved, the existing ones are unchanged.
12. The snapshot is independent of the current set: changing `bindings` after saving does not change the preset.

### 6.2. Applying and editing

13. Applying a user preset reproduces the set exactly.
14. Applying a built-in preset yields `Shortcuts::defaults(profile)`.
15. After applying, `pending.profile == preset.base`.
16. Editing any binding clears the "active" mark.
17. Renaming preserves bindings and activity.
18. Deleting the active preset does not change the current set and clears the mark.
19. Built-in presets cannot be renamed or deleted (a check at the model level, not only in the UI).
20. Saving with a taken name requires confirmation; declining changes nothing.

### 6.3. UI (`egui_kittest`)

21. The dialog has the "Built-in" and "My presets" sections.
22. "Save Current as Preset…" opens the name field, `Enter` saves, `Esc` cancels.
23. An empty name shows an error, the dialog does not close.
24. "Delete Preset" shows a confirmation; "Cancel" keeps the preset.
25. "Apply Preset" with unsaved edits shows a confirmation.
26. After "Apply Changes" and a restart (a new `BimApp` from the same file), the presets are in place.

### 6.4. Compatibility

27. A file written by the new build is read by the old `Config::from_json` code without losing `bindings` (a test of the function, not of the binary).
28. `.bks` export contains no presets; a "Replace" import does not delete presets.

---

## 7. Documentation

1. `docs/areas/shortcuts.md`:
   - a new "Starting sets and presets" section;
   - the difference between "starting set" / "preset";
   - how to save, apply, rename, delete;
   - what happens to the current set on applying;
   - limits (32 presets, 64 characters, 8,192 bindings in total);
   - that presets live in `settings.json`, not in the project.
2. `README.md`: one line in the keyboard shortcuts section.
3. Markdown help: the active preset and its base in the header (if P2 is implemented).
4. An example `settings.json` with presets — in the documentation, not in the repository.

---

## 8. Acceptance criteria

1. The user can save the current set under a name and see it in the preset list.
2. After restarting the application the preset is in place and is applied unchanged.
3. "Revit-like" and "Archicad-like" are available as built-in presets and work as before.
4. Applying a preset replaces the editable set, requires confirmation when there are unsaved edits, and is rolled back by the "Cancel" button.
5. Built-in presets cannot be renamed or deleted; user ones can, with confirmation.
6. The preset name is unique case-insensitively; empty and too-long names are rejected.
7. A `settings.json` without presets is read without errors; older builds do not lose the current set.
8. Presets do not end up in the project and do not affect import/export of the current set.
9. All new strings are localized; tests are green, `cargo clippy` has no warnings.

---

## 9. Out of scope

- Cloud synchronization of presets and a shared team profile.
- Automatic preset switching by project or by active tab.
- Family/Project scopes (`SPEC-011`); after they appear, a preset will also have to store a scope, but that is a separate spec.
- Importing presets from Revit/Archicad as separate layout files (this is `ТЗ-012` about Revit XML).
- Changing the current starting sets and their contents.
- Hotkeys for the preset operations themselves (e.g. `Ctrl+Alt+1` for preset #1).

---

## 10. Risks and pitfalls

1. **Confusion between "profile" and "preset".** The code already has three `Profile`s; the new type must be named `ShortcutPreset`, and the UI texts must clearly distinguish "Built-in" and "My presets".
2. **Losing the set on applying.** The main risk: the user pressed "Apply Preset" and lost their edits. Confirmation and rollback via "Cancel" are mandatory.
3. **Bloating `settings.json`.** Full snapshots of 32 presets may approach 1 MiB. The overall binding budget and a refusal to save are mandatory; silent truncation is unacceptable.
4. **Older builds.** Do not bump `version`; otherwise the old `Config::from_json` will return the base set and the user will lose their bindings. Add fields only as optional.
5. **Duplicate names.** Compare case-insensitively and with whitespace trimming; otherwise "Work" and "work" will create two presets and confusion in `active_preset`.
6. **A preset as a "diff".** Do not do this: after a program update, new starting keys the user did not save would appear in it.
7. **Conflicts inside a preset.** A preset is created from a valid set, but the file can be edited by hand; reading must be robust and must not crash the application.
8. **Localization of names.** A preset name is user data; do not translate it and do not normalize its case when storing.
9. **Preset order.** Use `Vec`, not `HashMap`, otherwise the list will jump between runs.
10. **Relation to `Profile`.** When applying a preset, `profile = base`; otherwise "Reset Action" will start restoring the wrong keys the user expects.

---

## 11. Open questions

1. **Quick switching without the dialog.** Should a Manage > Shortcuts > Preset submenu (or an item in the window context menu) be added so the key layout can be changed in one click? Recommendation: not in this iteration; first the dialog, then a separate small PR if there is a request. The block name is already decided: "Keyboard presets", see 3.9.1.
2. **The fate of `Profile::Custom`.** Keep it as an internal origin label or show "Custom" in the list as a separate built-in preset "current without a name"? Recommendation: do not show it; `Custom` is only a `base` for presets.
3. **Exporting presets.** Is a separate JSON of all presets needed in this iteration, or is P2 enough? Recommendation: P2, so as not to extend the import/export format in the first PR.
4. **Saving a preset from an imported set.** Is a "Save as Preset…" button needed in the import preview? Recommendation: P2.
5. **The `active_preset` label.** Store it in the file or compute it by comparing the current set with the presets? Recommendation: store it, this is cheaper and survives large sets; on divergence show "modified".
6. **Default name for a new preset.** An empty field or "Preset 1"? Recommendation: empty with a hint, so the user does not save random names.

---

## Appendix A. What the block looks like in the dialog

```text
Keyboard presets
┌──────────────────────────────────────────────────────────────┐
│ [ Work keys                         ▼ ]  [ Apply Preset ]   │
│ Built-in: BIMCraft, Revit-like, Archicad-like, Empty         │
│ My presets: Work keys (active), Drafting minimum             │
│ [Save Current as Preset…] [Rename Preset…] [Delete Preset]   │
└──────────────────────────────────────────────────────────────┘
```

- For a built-in preset, "Rename Preset…" and "Delete Preset" are disabled, and the tooltip explains why.
- "active" is shown while the set matches the preset; after an edit — "modified".
- Errors and confirmations are shown below the block, in the style of the existing `confirm_reset_all`.

## Appendix B. Implementation checklist

- [ ] `ShortcutPreset` + `Config` fields (`presets`, `active_preset`).
- [ ] Reading/writing with limits and resilience to a corrupted file.
- [ ] Saving, applying, renaming, deleting, (P2) duplicating.
- [ ] Confirmations: name replacement, deletion, replacement of the set being edited.
- [ ] The "Presets" block in the dialog, Built-in / My presets sections.
- [ ] "active" / "modified" labels.
- [ ] New strings in `.pot` and translations.
- [ ] Model, settings, and `egui_kittest` tests.
- [ ] `docs/areas/shortcuts.md`, `README.md`.
- [ ] `cargo clippy` with no warnings; tests are not run without a direct user request.
