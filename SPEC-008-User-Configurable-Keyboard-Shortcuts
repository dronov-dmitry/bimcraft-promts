# SPEC-008. User-Configurable Keyboard Shortcuts: Assign Custom Combinations to All Commands, Multiple Combinations per Command, Layout Independence, Import and Export

**Status:** Ready for implementation  
**Date:** 2026-11-08  
**Project:** BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)  
**Related documents:** `TS-007-ribbon-foundation-handles-joints-snapping.md` (document formatting), `TS-009-free-ctrl-d-and-f8.md` (clarification regarding `Ctrl+D`/Home and `F8`/`F11`), `BIMCraft/docs/areas/desktop.md` (Keyboard Shortcuts section), `BIMCraft/README.md` (keyboard shortcuts section)  
**Task type:** User-configurable interface settings (app-level settings), keyboard input, settings dialog, file format, migration of existing settings.

> **Current status.** Decisions concerning `Ctrl+D`, Home, and `F8`/`F11` are clarified in `TS-009-free-ctrl-d-and-f8.md`. TS-009 takes precedence wherever the documents conflict:
>
> - Home is `Ctrl+U`, not `Ctrl+D` or `Ctrl+Shift+H`;
> - `Ctrl+D` is **free** in the default BIMCraft profile (Move may use it only in the Archicad profile);
> - **Alignment Guides** use `F8` only, without `F11`.
>   Examples and acceptance criteria in SPEC-008 that specify otherwise must be read with these corrections.

---

## 0. General Working Rules

- Commits must be atomic, one item per commit; messages must follow the branch's style (an imperative title, with the body explaining *why*, not *what*).
- The language of code, identifiers, code comments, and commit messages is English. Documentation in `docs/` and this file are in Russian.
- All UI strings must go through `t()` / `tf()` / `msg!()` from `crates/ui-egui/src/lang.rs` and `bimcraft-i18n`; add new strings to `translations/ui/bimcraft-ui.pot` and the translations.
- Settings are application settings, not project settings: they live in `settings.json`, survive New/Open, and are not stored in `.bimcraft`.
- Do not break existing two-letter Revit-style commands (`WA`, `DR`, `WN`, `RM`, `CL`, `DI`, `TG`, `MV`, `CO`, …): they remain functional and become one type of binding.
- Cover every behavior change with tests; use `egui_kittest` for UI tests and ordinary unit tests for the parser/model.
- Do not use `subagent` or parallel agents (user requirement); all work must be performed in the current session.

---

## 1. Context and Task

### 1.1. What the User Wants

BIMCraft commands are currently entered using the Revit-style approach: two Latin letters or digits (`WA` — Wall, `DR` — Door, `MV` — Move). The user needs to:

1. **Assign custom combinations to all application commands** — as in Revit, through a "Keyboard Shortcuts" dialog, but with real modifiers: `Ctrl`, `Shift`, `Alt`, `Cmd`.
2. **Assign multiple combinations to one command.** For example, Move may have `Ctrl+D`, `Ctrl+Shift+M`, and the legacy `MV`.
3. **Make combinations work across keyboard layouts.** `Ctrl+D` on an English layout and `Ctrl+В` on a Russian (ЙЦУКЕН) layout refer to the same physical key and must perform the same action.
4. **Export all combinations to a text file and import them back.** The file must be human-readable, editable in a text editor, and suitable for transferring settings between machines.
5. **Provide a default set similar to Archicad:** at minimum, `Ctrl+D` should mean Move (currently `Ctrl+D` opens the Start page/Home; this must be reassigned).
6. **Provide an easy way to discover all available combinations.** BIMCraft currently has hardcoded fixed combinations but no single place where users can review them: some appear in the dialog, some in tooltips, and some only in code/README. The application needs to export **all active and all existing (default) combinations** to a human-readable Markdown file — a reference/cheat sheet that users can open, print, or share with colleagues.

### 1.2. Task Definition

Replace the current model of "one two-letter combination → one action" with an "action ↔ multiple bindings" model, where a binding is either:

- a **chord**: modifiers plus one key, e.g. `Ctrl+D`, `Ctrl+Shift+D`, `Alt+F3`;
- a **sequence**: two keys without modifiers, e.g. `WA`, `MV` (Revit-style compatibility).

Requirements:

- An action is addressed by a stable identifier, not a localized label.
- A key is identified by its **physical position** (`physical_key`), not by the character produced by the current keyboard layout.
- The same chord cannot be assigned to two actions.
- An action may have 0, 1, or many bindings.
- Bindings are stored in `settings.json` and can be imported/exported in a text format.
- Existing settings and legacy two-letter combinations continue to work.

### 1.3. User Scenarios

- As an Archicad user, I press `Ctrl+D` and get the Move tool, not the Start page.
- As a user with a Russian keyboard layout, I press `Ctrl+В` and get the same Move action as `Ctrl+D` on an English layout.
- As a user, I assign both `Delete` and `Ctrl+Shift+Delete` to Erase and see both combinations in its tooltip.
- As a user, I export settings to `my-keys.txt`, edit a few lines in a text editor, and import the file again; all valid lines are applied, while invalid ones are reported with line numbers.
- As a user, I see assigned combinations in ribbon tooltips, menus, and Find (Ctrl+K).
- As a user, I open "Keyboard Shortcuts Help" and/or export it to `.md` to see all commands, their active and default combinations, including combinations I have not assigned, and print a cheat sheet.
- As a user, I do not want `Ctrl+C` / `Ctrl+V` intercepted while entering text in a field.
- As a user, I want to reset one command, a category, or everything to the default set.

---

## 2. Current State (Based on the Code)

This section records the current implementation so that the work can be targeted rather than becoming a complete rewrite.

### 2.1. Action Catalog and Current Binding Model

- `crates/ui-egui/src/shortcuts.rs`:
  - `ACTIONS: &[&str]` — approximately 110 action names that can be assigned (walls, doors, views, hide, snaps, etc.). This is **not** the complete core command catalog: some core commands are missing, and some UI actions are not core commands.
  - `is_key(k)` — a key is considered valid only if `k.len() == 2` and both characters are ASCII alphanumeric. Therefore, `Ctrl+D` is currently impossible.
  - `resolve_action(desc)` — finds an action by its full English or localized name.
  - `parse_shortcuts_file(content)` — reads:
    - JSON object `{"WA": "Wall"}`;
    - Revit lines `<Shortcut CommandName="Wall" Shortcuts="WA#W1"/>`;
    - lines in the forms `WA - Wall`, `WA = Wall`, `WA: Wall`.
  - `find_conflict(key, action, app)` — checks whether a two-letter key is already assigned to another action.
  - `add_shortcuts(parsed)` — adds key/action pairs; skips conflicts and invalid keys.
  - `load_shortcuts_from_file(path)` — reads files up to 64 KiB, adds valid pairs, and displays a message.
  - `execute_shortcut_action(action)` — a large `match` on the action name: sets `Tool`, opens a dialog, or calls `self.run(...)`.
  - `shortcuts_window(ctx)` — modal dialog with search, Load File, Add Shortcut, Restore Defaults, and an action → keys table with Edit/Save/Cancel.
- `crates/ui-egui/src/lib.rs`:
  - `SHORTCUTS: [(&str, &str); 54]` — built-in two-letter combinations (`WA`, `DR`, `WN`, `MM`, `DM`, `AL`, `OF`, `DW`, `ZF`, `TL`, `CS`, `PP`, `NP`, `SA`, `VV`, `EH`, `HH`, `KS`, …).
  - `shortcuts()` — combines `SHORTCUTS`, MEP fitting letters, and core command aliases.
  - `BimApp::custom_shortcuts: BTreeMap<String, String>` — user bindings of "two-letter key → English action name".
  - `BimApp::typed: String` — the most recently typed characters (up to two) for two-letter combinations.
  - `BimApp::shortcut()` — checks `custom_shortcuts` first, then a built-in `match` on `typed`.
  - `BimApp::keys()` (approximately lines 1780–2050) — keyboard event handling.
- `crates/ui-egui/src/settings.rs`:
  - `Settings.shortcuts: BTreeMap<String, String>` — the same two-letter bindings in `settings.json`.
  - `MAX_SHORTCUTS: usize = 256`.
  - `shortcuts_from(v, base)` — reads only two-character ASCII keys and action names from `ACTIONS`.
- `crates/core/src/engine.rs`:
  - `Command { id, label, aliases, params, run, undoable, menu }` — a core command.
  - `commands()` — the full catalog, currently **170 commands**.
  - `find(name)` — finds a command by `id` or alias.
- `crates/core/src/commands.rs`:
  - `ALL: &[Command]` — 170 commands, 35 of which have `menu`, and 57 of which have aliases.
- `crates/ui-egui/src/find.rs`:
  - Find (Ctrl+K) displays actions from `shortcuts::ACTIONS` and uses only the **first** matching binding in the label.
- `docs/areas/desktop.md`, lines 479–488 — current description of the mechanism.
- `README.md`, line 407 — current user-facing description.
- **There is no unified help or export.** The dialog displays an action table, but it cannot be exported or printed; built-in combinations are only partially listed in the README and tooltips, and users cannot see the complete list of available commands. Markdown export is intended to close this specific gap.

### 2.2. Hardcoded Modifier Shortcuts

`keys()` currently contains separate checks that are not connected to the dialog and cannot be configured:

| Shortcut                 | Action                                   | Location              |
| ------------------------ | ---------------------------------------- | --------------------- |
| `Ctrl+S`                 | Save                                     | `lib.rs`, around 1826 |
| `Ctrl+Shift+S`           | Save As                                  | same                  |
| `Ctrl+O`                 | Open                                     | around 1829           |
| `Ctrl+N`                 | New                                      | around 1832           |
| `Ctrl+K`                 | Find                                     | around 1835           |
| `Ctrl+,`                 | Settings                                 | around 1838           |
| `Ctrl+1`                 | Properties                               | around 1841           |
| `Ctrl+P`                 | Export PDF                               | around 1844           |
| `Ctrl+D`                 | Home / Start page                        | around 1851           |
| `Ctrl+Z`                 | Undo (or sketch step)                    | around 1856           |
| `Ctrl+Shift+Z`, `Ctrl+Y` | Redo                                     | around 1859           |
| `F8`, `F11`              | Alignment guides                         | around 1905           |
| `Delete`                 | Delete                                   | around 1888           |
| `Space`                  | Flip / rotate component / dimension mode | around 1890–1900      |

All of these, except strictly contextual keys such as `Esc`, `Enter`, `Tab`, and `Space`, must move into the action registry and become configurable.

### 2.3. Current Keyboard Reading Flow

In `keys()`:

1. If any modal dialog, popup, or menu is open, processing stops.
2. Hardcoded `pressed(Modifiers, Key)` checks are evaluated.
3. If `ctx.egui_wants_keyboard_input()` is true, processing stops.
4. `Event::Text` events are collected — these are **characters from the current layout**, not physical keys.
5. Characters are appended to `typed`; when its length reaches 2, `shortcut()` is called.

Consequences:

- On a Russian layout, `WA` produces the text `ЦФ`, so the built-in combination does not work.
- `Ctrl+D` cannot be stored: `is_key` accepts only two ASCII characters.
- Key order and modifiers are checked manually, outside the dialog.
- Multiple bindings cannot be assigned to one action: `custom_shortcuts` is `Map<key, action>`, not `Map<action, Vec<key>>`.
- When editing an action in the dialog, the old user keys for that action are removed (`retain(|_, v| *v != action)`), so exactly one custom key is supported.
- There is no export; import only adds entries and skips conflicts.

### 2.4. Why `Ctrl+D` Currently Conflicts

`Ctrl+D` is hardcoded to `toggle_home()` (the Start page). The user wants `Ctrl+D` = Move, as in Archicad. Therefore:

- `Ctrl+D` should execute Move by default;
- Home must be moved to another combination (the proposed option is `Ctrl+Shift+H`) or left without a shortcut;
- migration of old settings is safe: legacy user keys were two-letter combinations only, so `Ctrl+D` could not have been stored in them.

---

## 3. Terminology

| Term                         | Meaning                                                                                                                                                                                      |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Action**                   | Any user-facing command that can be assigned keys: a tool, core command, dialog opening, palette toggle, or view. Has a stable `id`.                                                         |
| **Command**                  | An entry from `bimcraft_core::engine::commands()` with an `id` field; a specific type of action.                                                                                             |
| **Binding**                  | One keyboard combination assigned to an action.                                                                                                                                              |
| **Chord**                    | Modifiers plus one key: `Ctrl+D`, `Ctrl+Shift+D`, `Alt+F3`, `F2`.                                                                                                                            |
| **Sequence**                 | Two keys without modifiers: `WA`, `MV`. Revit-style compatibility.                                                                                                                           |
| **Physical key**             | Keyboard position (`egui::Key` from `physical_key`), independent of layout. `Ctrl+D` and `Ctrl+В` are the same physical `KeyD`.                                                              |
| **Logical key**              | A key interpreted according to the active layout; for non-Latin layouts, egui falls back to the physical key.                                                                                |
| **Keyboard layout**          | The active OS layout (US, RU/ЙЦУКЕН, UA, DE/QWERTZ, FR/AZERTY…).                                                                                                                             |
| **Profile**                  | A complete set of bindings: a built-in profile (BIMCraft/Revit/Archicad) or a user profile.                                                                                                  |
| **Conflict**                 | The same chord/sequence is assigned to two different actions.                                                                                                                                |
| **Reserved key**             | A key the application does not make available for bindings (`Esc`, `Enter`, `Tab`, `Backspace`, arrow keys, `Space`), or a key intercepted by the OS (`Alt+Tab`, `Alt+F4`, `Cmd+Tab`, etc.). |
| **Canonical representation** | A single representation of a binding in files and memory: Latin physical-key name, modifiers ordered as `Ctrl+Shift+Alt+Meta+Key`.                                                           |

---

## 4. Functional Requirements

### 4.1. Completeness of the Action Catalog

1. **All user-facing commands must be in the registry.** The registry is the single source of truth for the dialog, Find, tooltips, and keyboard processing. It includes:

   - all 170 commands from `engine::commands()`;
   - all actions from the current `shortcuts::ACTIONS`;
   - UI actions that are not core commands: Save As, Export PDF, Settings, Home, Properties, Project Browser, 3D View, Floor Plan, Section Box, Thin Lines, Dark Mode, Close Inactive, Load Family, Favorites, Materials, Project Parameters, Snap overrides, Alignment Guides, Zoom to Fit, Hide/Unhide, Temporary Hide/Isolate, Visual Style, etc.

2. **No command may be lost.** Even if a command cannot be assigned (it requires mandatory parameters and has no dialog), it must appear in the dialog as a greyed-out row with the reason. Test this requirement: every `Command.id` from `engine::commands()` must either have a registry entry or be explicitly listed in an exclusion list with a reason.

3. **Stable identifier.** For core commands use `cmd.<id>` (e.g. `cmd.wall`, `cmd.move`). For UI actions use `ui.<area>.<action>` (e.g. `ui.file.save_as`, `ui.palette.properties`, `ui.view.3d`). For legacy actions that are not in the core, use `legacy.<English name>`. An ID must not change when localization changes and must not depend on the label.

   If a UI action is a thin wrapper around a core command (`Open`, `New`, `Save`, `Undo`, `Redo`), use `cmd.<id>` rather than creating a second `ui.*` action. Use `ui.*` only for functionality that is not in the core or for distinct behavior: Save As, Export PDF, Settings, Home, palettes, and views.

4. **Action classification:**

   - `Tool` — selects the active tool (`Wall`, `Door`, `Move`, `Rotate`, …);
   - `Dialog` — opens an existing dialog (`Settings`, `Visibility/Graphics`, `Filters`, `Phasing`, …);
   - `Immediate` — executes immediately without mandatory parameters (`Zoom to Fit`, `Thin Lines`, `Undo`, `Redo`, `Save`, …);
   - `Input` — requires mandatory data (file path, points, parameters); assignable only if the command has an existing UI parameter collector; otherwise mark it as "requires parameters";
   - `Internal` — not a user-facing command (autosave, recover, cross-check export, etc.); displayed but not assignable.

5. **One ID — one executor.** A test verifies ID uniqueness and that every assignable action has a working executor (`execute_action` returns `true`).

6. Duplicate labels are allowed; duplicate IDs are not. For example, `column` and `structural_column` share the alias `CL`; they are different actions with different IDs, and contextual `CL` selection remains a separate rule.

### 4.2. Bindings: Multiple per Action

1. An action may have **0, 1, or many** bindings. The limit is 32 bindings per action (protection against garbage); the global limit is 4096 bindings.
2. Bindings for one action are displayed in insertion order: built-in bindings first, then user-defined bindings.
3. The user can:
   - add a binding;
   - delete one binding;
   - replace a binding;
   - delete all bindings for an action (the action remains without keys);
   - reset an action to its built-in bindings;
   - reset a category/all actions to the selected profile.
4. Built-in bindings can be overridden and deleted. If the user deletes a built-in binding, it must not reappear automatically.
5. Priority: user-defined and built-in bindings have equal priority during lookup; conflicts between them are not allowed (see 4.3).
6. The same binding **cannot** belong to two actions. When assignment is attempted, show a conflict: "`Ctrl+D` is already assigned to 'Move'. Replace it?" Options: "Replace" (remove it from the previous action) or "Cancel".
7. If a conflict is found while importing a file, skip that line and report its line number, action, and conflicting action. The first valid line in the file wins.
8. If an action is unavailable in the current context (nothing selected, a different view is open), the key is still **consumed** and the status "Unavailable: <reason>" is displayed; it must not unexpectedly execute another command.


### 4.3. Chord model and exact matching

1. A chord consists of:
   - the primary modifier `Ctrl` (on macOS — `Cmd`; in the file `Ctrl` means "primary", see 4.4);
   - `Shift`;
   - `Alt` / `Option`;
   - `Meta` / `Cmd` — literal Command on macOS (`Modifiers::mac_cmd`); on Windows/Linux the Win/Super key is usually intercepted by the OS and does not reach egui, so `Meta` does not work there (the UI warns);
   - one key.
2. **Modifier matching is exact.** `Ctrl+D` and `Ctrl+Shift+D` are different bindings and can coexist. You cannot use `egui::InputState::consume_key` as is: it applies `Modifiers::matches_logically`, which ignores extra `Shift`/`Alt` and may fire `Ctrl+D` while `Ctrl+Shift+D` is held. You must compare with `Modifiers::matches_exact` (taking `command`/`mac_cmd` into account) and consume only the exact event.
3. **Order of modifiers in canonical form:** `Ctrl`, `Shift`, `Alt`, `Meta`, then the key. Example: `Ctrl+Shift+Alt+Meta+D`. The displayed order may be platform-specific (`Cmd+Shift+D` on macOS).
4. **A modifier key by itself** (Ctrl, Shift, Alt, Cmd) is not a binding.
5. **Key repeat is ignored:** holding `Ctrl+D` must not trigger "Move" multiple times. Use only events with `pressed: true, repeat: false`.
6. **CapsLock and NumLock** do not affect the physical key. Numpad keys are considered separate if egui distinguishes them; otherwise the common `Num0..Num9` are used.
7. **A single key without modifiers** is allowed only for keys that do not participate in sequences: `F1`–`F24`, `Delete`, `Insert`, `Home`, `End`, `PageUp`, `PageDown`, `PrintScreen`, etc. Single **letters and digits** without modifiers are forbidden: otherwise `D` would fire immediately and make the sequences `DR`, `DW`, `DI` impossible. The UI explains the reason for rejection.
8. **AltGr.** On Windows AltGr arrives as `Ctrl+Alt`; chords with `Ctrl+Alt` are allowed, but the UI warns that on some layouts AltGr may be intercepted by the OS/driver. If in the same frame there is an `Event::Text` generated by AltGr and focus is in a text field — the text field has priority.

### 4.4. Layout independence

This is a key requirement.

1. **The source of truth is the physical key.** In `egui 0.36` the `Event::Key` event contains:
   - `key: Key` — the logical key; for non-Latin layouts egui falls back to the physical one;
   - `physical_key: Option<Key>` — the physical position, independent of layout;
   - `modifiers: Modifiers`;
   - `repeat: bool`.
     For bindings use `physical_key.unwrap_or(key)`. This gives `Ctrl+D` == `Ctrl+В` on ЙЦУКЕН, `Ctrl+D` == `Ctrl+D` on QWERTZ (US D position), and identical behavior when switching layout during work.
2. **Never use `Event::Text` for chords and sequences.** `Event::Text` depends on layout and gives `ЦФ` instead of `WA`, `в` instead of `d`. Text is needed only for input into text fields and for typing length/numbers in tools.
3. **Canonical storage — Latin names of physical keys.** In memory and on export, `Ctrl+D` is `physical_key = Key::D`, regardless of what is printed on the key. On ЙЦУКЕН the same key is labeled `В`.
4. **Import accepts localized letters.** `Ctrl+В` in a file is recognized as `Ctrl+D` via the ЙЦУКЕН table (Appendix C). `Ctrl+С` (Cyrillic `С`) → `Ctrl+C`, `Ctrl+М` → `Ctrl+V`, `Ctrl+А` → `Ctrl+F`, `Ctrl+Ф` → `Ctrl+A`, etc. Similar tables can be added for Ukrainian and other layouts; the minimum per the ТЗ is Russian.
5. **Display.** In the interface show the canonical form `Ctrl+D`; if the interface language is Russian (or showing localized labels is enabled), show `Ctrl+D (Ctrl+В)` in the dialog and hint. In compact places (menu, Find) — canonical form only.
6. **Changing layout during work** does not change bindings and does not require a restart.
7. **Limitation for non-Latin and non-ЙЦУКЕН layouts.** The physical position is the US keyboard position. On AZERTY/QWERTZ a user who thinks about the letter on their key may see a mismatch with the label. This is a deliberate decision: the user requirement is "the same button on different layouts". In the hint write "physical key D (on ЙЦУКЕН — В)".
8. **A test is mandatory:** an event with `physical_key = Some(Key::D), modifiers = Ctrl`, regardless of `text = "в"` / `text = "d"` / no text, must perform the same action.

### 4.5. Sequences (Revit-style compatibility)

1. A sequence is two keys without `Ctrl`/`Alt`/`Meta` (Shift is allowed but does not change the key). Example: `W`, then `A`.
2. Sequences are read from physical keys, not from `Event::Text`. This fixes `WA` on the Russian layout.
3. The sequence buffer:
   - holds no more than 2 keys;
   - is reset by `Esc`, on mouse click, tool change, opening a modal dialog/menu, after a binding fires;
   - is reset by an inactivity timeout (recommended 2 seconds) so that an old letter does not "stick" to a new command;
   - does not intercept digits/period if the tool is waiting for length input (`typed_length`).
4. A sequence can be assigned to any action on par with a chord. In the dialog it is displayed as `WA` (without `+`).
5. One sequence cannot be assigned to two actions; a conflict is handled the same way as for a chord.
6. Old two-letter bindings from `SHORTCUTS`, command aliases and `custom_shortcuts` migrate into sequences without loss of behavior.

### 4.6. Context, priority and key consumption

Keyboard processing order (top to bottom, the first level that fires wins):

1. **OS/window.** Combinations intercepted by the OS (Alt+Tab, Alt+F4, Cmd+Tab, Cmd+Space, Win+L, etc.) do not reach the application; the application cannot rebind them.
2. **Modal dialog.** While a modal dialog is open, only its local keys work (`Esc`, `Enter`, navigation) and the capture mode in the hotkeys dialog. Global bindings do not fire.
3. **Text input.** If `ctx.egui_wants_keyboard_input()` is true (input field, search, name editor, numeric field), all configurable bindings are disabled. The exception is "global" actions that do not steal text:
   - `Save`, `Save As`, `Open`, `New`, `Find`, `Settings` — allowed;
   - `Undo`/`Redo` in a text field remain with the field (undo of text typing), not with the model;
   - `Ctrl+C`/`Ctrl+V`/`Ctrl+X`/`Ctrl+A` always belong to the text field.
4. **Open menu/popup.** Navigation and `Esc` belong to the menu. After the menu closes, bindings work again.
5. **Tool input.** If the active tool is waiting for a number/point/length, digits, `.` and `,` go to `typed_length`; `Enter` completes the input. Chords with modifiers can still work if they do not conflict with input.
6. **Chords.** Exact match by `physical_key` + `Modifiers::matches_exact`.
7. **Sequences.** If the chord did not fire and there are no modifiers (except Shift), the key goes into the sequence buffer.
8. **Tool context keys.** `Esc`, `Enter`, `Tab`, `Backspace`, arrows, `Space` — are reserved (see 4.10) and are always handled by the tool; an action cannot be assigned to them.
9. **Sketch mode.** While a sketch is active, `Ctrl+Z` undoes the last sketch line, not a model step (as now in `keys()`); `Ctrl+Shift+Z`/`Ctrl+Y` in a sketch must not undo the model. This context rule has priority over the general Undo/Redo binding.

Additionally:

- One press is consumed exactly once. After a binding fires, the event is removed from `InputState.events` (or marked as consumed) so that other handlers do not perform the same command.
- A chord resets the sequence buffer.
- If a binding is assigned but the action is unavailable, the key is consumed and a status is shown; there is no fallback to a sequence/another command.
- `Ctrl+Shift+...` must not fire as `Ctrl+...`: the check order must not mask a more specific chord. Better to look up an exact match in a hash table rather than iterate from "weak" to "strong".

### 4.7. "Keyboard Shortcuts" dialog

The existing `shortcuts_window` is replaced/extended. Opening: Manage > Shortcuts, the Shortcuts button, Find, by default `KS` (sequence) and/or `Ctrl+Shift+K` (if it is decided to keep a chord; it may be left unassigned).

#### 4.7.1. General structure

- Modal window, width ~720–780 px, height up to 80% of the window.
- Top panel:
  - search field (by localized name, English name, ID, binding);
  - filter: "All", "Assigned", "Without keys", "Modified", "Conflicts";
  - filter by category (Architecture, Structures, Views, Manage, File, Palettes, Snaps, MEP, Annotations, All);
  - profile selection: "BIMCraft", "Revit-like", "Archicad-like", "Empty";
  - buttons "Import…", "Export…", "Help (Markdown)…", "Reset…".
- The "Help (Markdown)…" button stands next to export and immediately opens saving of the Markdown cheat sheet (see 4.8.4); this is the main "simple way to get acquainted" with all combinations.
- Table:
  - "Command" column — localized name + ID in small font;
  - "Category" column;
  - "Combinations" column — chips/buttons, each with `×`; a `+` button to add;
  - "Source" column — "default", "user", "modified";
  - "Actions" column — "Reset" / "Delete all".
- Bottom panel: counter "Assigned X of Y", "Conflicts: N", buttons "Cancel", "Apply".
- Changes in the dialog **accumulate** and are applied on "Apply"; "Cancel"/`Esc` rolls back. After "Apply" the settings are saved immediately to `settings.json` (via the common `keep_settings` mechanism).
- If there are unsaved changes, closing the window/`Esc` asks for confirmation.

#### 4.7.2. Adding and editing a binding

- The `+` button next to an action opens a capture field: "Press a combination…".
- In the capture field:
  - pressing `Ctrl+D` shows `Ctrl+D`; on the Russian layout additionally `(Ctrl+В)`;
  - pressing a key labeled `А` on ЙЦУКЕН will be recorded as `Ctrl+F` and show `(Ctrl+А)`: this is the physical key `F`, not the Latin `A`;
  - pressing two keys without modifiers (`W`, `A`) shows `WA` and creates a sequence;
  - `Esc` cancels capture;
  - `Backspace` clears capture;
  - `Tab`/`Enter` are not considered a binding;
  - pressing only a modifier is ignored with a hint;
  - a conflict is shown immediately, before saving, with the name of the owning action and a "Replace" button;
  - a reserved key shows the reason why it cannot be assigned.
- After a successful addition the binding appears as a chip; the next one can be added.
- Editing an existing chip: a click opens the same capture with the current binding; "Save" replaces, "Cancel" keeps.
- Deleting a chip: `×` or the context menu "Delete binding".

#### 4.7.3. Search, filters and sorting

- Search is case-insensitive; it searches by:
  - localized name;
  - English name;
  - ID (`cmd.move`, `ui.file.save_as`);
  - binding string (`Ctrl+D`, `WA`, `В`).
- An empty search shows everything.
- Default sorting: category, then localized name. You can sort by name/ID/number of bindings.
- Rows without bindings are visually dimmed but not hidden, unless the filter is "Without keys".
- Conflicting rows are highlighted with the error color; clicking a conflict leads to the second participant.

#### 4.7.4. Reset

- "Reset action" — restore the start bindings of the selected action.
- "Reset category" — restore the start bindings of all actions in the category.
- "Reset all" — restore the start profile (BIMCraft by default), with confirmation.
- "Clear all" — leave all actions without keys (for those who want to start from scratch), with confirmation.
- Reset does not touch other application settings.

#### 4.7.5. Applying a profile

- Selecting a profile shows a preview: how many bindings will be added/changed/removed.
- Applying a profile replaces all current bindings with the selected set (after confirmation).
- Profiles: `BIMCraft` (default), `Revit-like`, `Archicad-like`, `Empty`.
- A custom profile is saved as "Custom" after any change.

### 4.8. Import and export

#### 4.8.1. Export

1. The "Export…" button opens the system save dialog.
2. Formats:
   - main — text `.txt` / `.bks` (BIMCraft Keyboard Shortcuts), UTF-8, suitable for re-import;
   - additionally — `.json` (the same model, convenient for scripts, suitable for re-import);
   - **`.md` — Markdown help/cheat sheet** for reading, printing and passing to a colleague (see 4.8.4); this is **not** a backup format and not an import format;
   - optionally — Revit XML (sequences only; chords are not representable, warn on export).
3. Export modes:
   - **whole profile** (all actions, including start and empty) — by default;
   - **only user changes** (actions whose bindings differ from the start profile).
4. Export is deterministic: the same state → the same file (no random order, no timestamp in the content; the application version — in a comment, if needed).
5. Writing is atomic: first a temporary file, then `rename`. A write error must not corrupt the existing file.
6. Default name: `bimcraft-shortcuts.txt`; on repeated export — the last chosen path (stored in settings, optionally).

#### 4.8.2. Import

1. The "Import…" button opens the system file selection dialog.
2. Supported formats:
   - the new text format `.txt` / `.bks` (section 5);
   - JSON (`{"version": 2, "bindings": [...]}` or the old `{"WA": "Wall"}`);
   - Revit XML strings `<Shortcut CommandName="…" Shortcuts="WA#W1"/>`;
   - legacy strings `WA - Wall`, `WA = Wall`, `WA: Wall`.
   - **Markdown `.md` is not supported**: it is a help for reading (4.8.4). If `.md` is selected, show a message and do not change settings.
3. Import modes:
   - **Replace all** — current bindings are replaced by the file contents; actions absent from the file remain without keys. This is the default mode for a file with `version = 2`.
   - **Add/merge** — valid bindings from the file are added to the current ones; conflicts are skipped with a report.
4. Before applying, a preview is shown:
   - how many bindings will be added, replaced, skipped;
   - a list of errors with line numbers and reasons;
   - a list of conflicts;
   - a warning if the file is empty or contains no valid bindings.
5. Application is atomic: first full parsing and validation, then replacement in memory, then saving `settings.json`. If saving fails — roll back in memory and show an error.
6. Before replacement, save a backup of the current profile next to the settings (`shortcuts.bak`) or in the settings folder with a date.
7. Limits: file no more than 256 KiB, no more than 10,000 lines, no more than 4096 bindings. Exceeding — refusal with a message, without partial application.
8. Errors do not "break" the import entirely: valid lines are applied, invalid ones are listed. The exception is a file version/structure error: then nothing is applied.
9. Import does not execute any code from the file: data only.

#### 4.8.3. "Export → import" check

- Export → import in "Replace all" mode must give a semantically identical set of bindings (the order within an action is preserved, the order of actions may be canonicalized).
- A repeated export after import gives the same text (determinism).
- `Ctrl+В` on import is canonicalized to `Ctrl+D`; on export it is written as `Ctrl+D`. This is not a loss: both records mean one physical key.

#### 4.8.4. Exporting help to Markdown

This is a separate, **human-readable** export: it does not replace `.bks`/`.json` and is not intended for re-import.

1. **Purpose.** Give the user a simple way to see all combinations that exist in the program: effective, start (built-in), user, as well as commands without keys. The resulting `.md` can be opened in any Markdown viewer, printed, put in a repository or passed to a colleague.
2. **Entry point.** The "Help (Markdown)…" button in the hotkeys dialog (next to "Export…"), as well as the "Markdown (help)" item in the general export dialog.
3. **What goes into the help:**
   - **all registry actions** — all 170 core commands and all interface actions, including those with no bindings at all;
   - **effective bindings** (what will actually fire now);
   - **start bindings** (built-in values, even if the user changed or deleted them) — in a separate column or section;
   - the localized action name, the English name and the stable ID;
   - category and/or menu path (`Architecture > Wall`, `Manage > Link Model`);
   - source: "default", "user", "modified", "deleted";
   - for actions without keys — an explicit "not assigned" mark;
   - conflicts (if any) and reserved/system combinations;
   - profile, format/application version, generation date and platform;
   - for Russian — localized hints like `Ctrl+D (Ctrl+В)`.
4. **File structure:**
   - heading `# BIMCraft — keyboard shortcuts`;
   - a short "how to read" instruction and a table of contents;
   - sections by category (File, Edit, Architecture, Structures, Views, Manage, Palettes, Snaps, MEP, Annotations, Other);
   - within a section — a table:
     `| Command | ID | Effective | Start | Source | Note |`;
   - a separate section "Commands without keys" — so they are visible and can be assigned;
   - a separate section "Reserved and system combinations";
   - a separate section "Conflicts" — only if they exist, indicating both actions.
5. **Export options:**
   - include commands without keys — **yes** by default;
   - include start bindings — **yes** by default;
   - only effective bindings / only user changes — optionally;
   - include bindings of other start profiles (Revit-like, Archicad-like, Empty) — optionally, as a separate section "Other profiles";
   - label language — interface language by default, English can be chosen;
   - `Ctrl+D (Ctrl+В)` hints — yes by default for the Russian interface.
6. **Determinism.** The same state → the same `.md` (sorting by category, then by localized name/ID; no random order). The generation date may be in the header but must not affect profile comparison; for diffs, provide an option "without date".
7. **Valid Markdown.** Tables with `|` escaped in names, no raw HTML, with proper headings and anchors. The file must open correctly in GitHub, VS Code, Obsidian and a regular viewer.
8. **Writing** — UTF-8, atomic (temporary file + `rename`), default name `bimcraft-shortcuts.md`.
9. **Import of `.md` is not supported.** If the user selects `.md` in the import dialog, show a message: "This is a help for reading. For import use `.bks`, `.txt` or `.json`". This protects against an attempt to "load the help" and lose settings.
10. **Single source.** The help is generated from the same action registry and the same `Shortcuts` state as the dialog, so it cannot diverge from it. A test verifies that every registry action is included in the `.md`.
11. **Not mandatory, but a useful bonus:** a "Preview" button in the dialog showing the same Markdown in a built-in scrollable window (without saving). This covers the "just look" scenario without creating a file.

### 4.9. Settings storage and migration

1. `Settings.shortcuts` changes type from `BTreeMap<String, String>` to a structure:

   ```json
   "shortcuts": {
     "version": 2,
     "profile": "bimcraft",
     "bindings": [
       {"action": "cmd.move", "keys": ["Ctrl+D", "Ctrl+Shift+M"]},
       {"action": "cmd.wall", "keys": ["WA", "Ctrl+Shift+E"]}
     ]
   }
   ```
2. The settings file stores the **full effective set** of bindings (not only differences from the start ones). This makes behavior predictable when the program is updated: start bindings do not "creep back in".
3. `shortcuts.version`:
   - `2` — the new format;
   - absent or an object without `version` — the old format `{"WA": "Wall"}`.
4. Migration of the old format:
   - each `key → action` pair turns into a sequence (`WA` → `WA`);
   - the action name is resolved to an ID via the registry (English/localized name);
   - unknown actions are skipped with a log entry (without breaking loading);
   - after the first successful load the settings are rewritten in version 2.
5. Increase `MAX_SHORTCUTS` to 4096 bindings; `MAX_ACTIONS` — 512. Increase `settings.rs::MAX_FILE` from 64 KiB to 1 MiB (or 512 KiB): 4096 bindings in JSON do not fit in 64 KiB, otherwise the settings will be silently unreadable.
6. A corrupted/too large `settings.json` must not erase settings: keep the last successful copy in memory, show the status, do not rewrite the file until the user acts.
7. New/Open do not touch bindings: these are application settings.
8. `use_settings` loads bindings into `BimApp::shortcuts`; `settings_now`/`keep_settings` save them.
9. Backward compatibility: if a new version of the program reads an old `settings.json`, it migrates; if an old version reads the new one — it must not crash (an unknown object is ignored, start bindings remain). This is a desirable but non-blocking requirement.

### 4.10. Reserved keys and conflicts with the OS

1. **Always reserved by the application** (cannot be assigned):
   - `Esc` — cancel/close;
   - `Enter` — complete input/construction;
   - `Tab` — cyclic selection/focus;
   - `Backspace` — delete the last character/step;
   - arrows — navigation/movement;
   - `Space` — tool context action (flip, rotate, size mode).
2. **Assignable, but with a warning:**
   - `Delete` (by default — the Delete command);
   - `F1`–`F12` (some may be occupied by the OS/window);
   - `Ctrl+Alt+…` (AltGr);
   - `Ctrl+Shift+Esc`, `Alt+F4`, `Alt+Tab`, `Ctrl+Esc`;
   - on macOS: `Cmd+Tab`, `Cmd+Space`, `Cmd+Q`, `Cmd+H`, `Cmd+M`, `Cmd+``;
   - on Windows: `Win+L`, `Win+D`, `Win+Tab`, `Ctrl+Alt+Del` (the latter does not reach at all).
3. UI on assignment:
   - a hard-reserved key is not accepted and the reason is explained;
   - an OS-conflicting one is accepted, but a yellow warning is shown: "may not work: intercepted by the system".
4. On import such bindings:
   - for hard-reserved — skipped with an error;
   - for OS-conflicting — accepted with a warning.
5. `Ctrl+D` is not reserved; it is reassigned to Move.

### 4.11. Displaying bindings in the interface

1. **Ribbon and tool palette hints.** For each action with bindings the hint has the form:
   `Move (Ctrl+D, Ctrl+Shift+M, MV)`.
   If there are no bindings — only the name and description, without empty parentheses.
2. **Menu.** To the right of the menu item show the first (or all, if they fit) binding via egui's standard `shortcut_text`. Example: `Save   Ctrl+S`.
3. **Find (Ctrl+K).** In the action row show up to two bindings and `+N` if there are more. Search by binding also works.
4. **Context menus.** If a context menu item corresponds to an action, show its binding.
5. **Status bar.** After a binding fires, if the action is unavailable, show the reason. If available — show nothing extra.
6. **Dialog.** Binding chips, conflicts, source.
7. **Localization.** Action names — via `t()`. Keys — canonical; for the Russian layout add `(Ctrl+В)` in the dialog/hint.
8. **No hard-coded strings** like `"Modify (MD)"`: all labels with keys are generated from the registry. Replace the existing strings in `lib.rs` (e.g. `"Modify (MD): select elements"`) with a helper.

### 4.12. Start profiles and `Ctrl+D`

> **ТЗ-009 (priority):** in the BIMCraft start profile `Ctrl+D` is **free**, Home — `Ctrl+U`. The items below about `Ctrl+D` = Move and Home = `Ctrl+Shift+H` apply only to the Archicad-like profile or to the ТЗ-008 draft.

1. **BIMCraft (default):**
   - keeps all current two-letter Revit combinations (`WA`, `DR`, `WN`, `RM`, `CL`, `DI`, `TG`, `MV`, `CO`, `RO`, `MM`, `DM`, `AL`, `OF`, `DW`, `ZF`, `TL`, `CS`, `PP`, `NP`, `SA`, `VV`, `EH`, `HH`, `KS`, …);
   - adds chords for frequently used actions (table in Appendix B);
   - `Ctrl+D` = **Move**;
   - Home = `Ctrl+Shift+H` (instead of `Ctrl+D`);
   - `Ctrl+Shift+D` = Copy (so as not to take `Ctrl+C` from text fields);
   - `Ctrl+R` = Rotate, `Ctrl+M` = Mirror Pick Axis, `Ctrl+Shift+M` = Mirror Draw Axis, `Ctrl+Shift+O` = Offset, `Ctrl+T` = Trim/Extend to Corner, `Ctrl+Shift+T` = Split, `Ctrl+0` = Zoom to Fit, `F3` = 3D View, `F2` = Floor Plan;
   - `Delete` = Delete;
   - `Ctrl+Z` = Undo, `Ctrl+Shift+Z` and `Ctrl+Y` = Redo;
   - `Ctrl+S` = Save, `Ctrl+Shift+S` = Save As, `Ctrl+O` = Open, `Ctrl+N` = New, `Ctrl+K` = Find, `Ctrl+,` = Settings, `Ctrl+1` = Properties, `Ctrl+P` = Export PDF.
2. **Revit-like:** the current `SHORTCUTS` + aliases; `Ctrl+D` may be left as Home or not assigned (the profile explicitly differs from BIMCraft). The user can choose it in the dialog.
3. **Archicad-like:** at minimum `Ctrl+D` = Move, `Ctrl+R` = Rotate, `Ctrl+M` = Mirror Pick Axis, `Ctrl+Shift+M` = Mirror Draw Axis, `Ctrl+Shift+D` = Copy, `Ctrl+Shift+O` = Offset, `Ctrl+T` = Trim, `Ctrl+0` = Zoom to Fit, `F2`/`F3` = Floor Plan/3D View. The rest — as desired; the profile does not have to be byte-for-byte like Archicad.
4. **Empty:** no bindings; useful for a clean start.
5. All profiles are data, not code: they can be stored in a built-in resource (`include_str!`) and updated without changing the logic.
6. Conflicts within the start profile are forbidden; a test verifies that no chord/sequence occurs twice.
7. On the first launch after the update, show a one-time notification: "`Ctrl+D` is now 'Move'; the start page is `Ctrl+Shift+H`. You can change this in Manage > Shortcuts". Do not show the notification if the user has already opened the new dialog or changed bindings.

### 4.13. Performance and reliability

1. Binding lookup on a keypress is O(1): hash table `KeyChord → ActionId` and `[Key; 2] → ActionId`.
2. Processing must not allocate on every keypress (except rare cases); reuse buffers.
3. `settings.json` is written atomically (already implemented) and no more than once per frame.
4. Importing 4096 bindings should take < 50 ms and not block the UI for more than one frame (if necessary — in a background thread with progress; for 4096 this is not required).
5. An error in one binding must not break application startup: the registry and settings are loaded into a consistent state.

---

## 5. Text file format

### 5.1. General properties

- Encoding: UTF-8; BOM is allowed and ignored.
- Line endings: LF or CRLF; the parser normalizes.
- Extension: `.txt` or `.bks` (same content).
- Comments: a line starting with `#` after optional spaces. Inline comments are not supported, so that `#` can be used as a key in the future.
- Empty lines are ignored.
- Case of action names and modifiers does not matter; the canonical form is lowercase IDs and `Ctrl+...`.
- Maximum size: 256 KiB; maximum 10,000 lines; maximum 4096 bindings.

### 5.2. Grammar

```
file        := { line }
line        := comment | blank | directive | binding
directive   := "version" "=" integer
             | "profile" "=" name
binding     := action ("=" | ":") binding-spec
binding-spec:= chord | sequence
chord       := { modifier "+" } key
sequence    := key key | legacy-two-letter
legacy-two-letter := letter letter
modifier    := "Ctrl" | "Control" | "Primary"
             | "Shift"
             | "Alt" | "Option"
             | "Meta" | "Cmd" | "Command" | "Super" | "Win"
key         := letter | digit | named-key | punctuation
```

- `action` — a stable ID (`cmd.move`, `ui.file.save_as`) or the English/localized action name. ID takes priority.
- One line — one binding. Multiple bindings for an action are given by multiple lines with the same `action`. A comma-separated list is **not supported**: `Ctrl+,` is the Settings key, and the comma cannot be a separator.
- Legacy lines `KEY - Name` (`WA - Wall`) are read by a separate parser branch for compatibility only; in the new format the action is always to the left of `=`.
- Spaces around `=`, `:` and `+` are ignored.
- If the line contains `+` — it is a chord.
- If there is no `+`:
  - two single-letter names without a space (`WA`) or two tokens separated by a space (`W A`) — a sequence;
  - one key name (`F2`, `Delete`) — a chord without modifiers; single letters and digits are forbidden (see 4.3.7).
- The `+` key is written as `Plus`; `Ctrl+Plus` — a chord with the "plus" key. The notation `Ctrl++` is not recommended and may be rejected as ambiguous.
- `Ctrl` in the file means **primary**: `Ctrl` on Windows/Linux, `Cmd` on macOS. `Cmd`/`Meta` — literal Command on macOS; on Windows/Linux Win/Super is usually unavailable to the application. For portable files use `Ctrl`; import `Cmd`/`Meta` on Windows/Linux with a warning and treat as primary, so that the file does not lose bindings.
- `Primary` — a synonym for `Ctrl`.
- `Option` — a synonym for `Alt`; on export on macOS write `Alt` for portability, accept `Option`.

### 5.3. Key names

Canonical names (synonyms from `egui::Key::from_name` are also accepted):

- letters: `A`…`Z`;
- digits: `0`…`9` (main row), `Num0`…`Num9` (Numpad);
- functions: `F1`…`F24`;
- navigation: `Up`, `Down`, `Left`, `Right`, `ArrowUp`, `ArrowDown`, `ArrowLeft`, `ArrowRight`, `Home`, `End`, `PageUp`, `PageDown`, `Insert`, `Delete`, `Backspace`, `Enter`, `Return`, `Tab`, `Escape`, `Esc`, `Space`;
- punctuation: `Comma`, `Period`, `Slash`, `Semicolon`, `Quote`, `Backquote`, `Minus`, `Equals`, `OpenBracket`, `CloseBracket`, `Backslash`, `Plus`, `Colon`, `Pipe`, `Questionmark`, `Exclamationmark`;
- Numpad: `NumpadAdd`, `NumpadSubtract`, `NumpadMultiply`, `NumpadDivide`, `NumpadEnter`, `NumpadDecimal`;
- other: `PrintScreen`, `Pause`, `Menu`, `BrowserBack` (if egui provides it).

Symbolic abbreviations for punctuation (`Ctrl+,`, `Ctrl+.`, `Ctrl+/`, `Ctrl+;`, `Ctrl+'`, `Ctrl+\``, `Ctrl+-`, `Ctrl+=`, `Ctrl+[`, `Ctrl+]`, `Ctrl+\`) are accepted. On export, for readability, write the symbols themselves (`Ctrl+,`, `Ctrl+.`), not `Comma`/`Period`; the exception is the "plus" key: write `Ctrl+Plus`, because `Ctrl++` is ambiguous. This matters for the familiar `Ctrl+,` (Settings).

### 5.4. Localized letters and layout

- `Ctrl+В` (Cyrillic) is accepted as `Ctrl+D`; the table is in Appendix C.
- `Ctrl+С` → `Ctrl+C`, `Ctrl+М` → `Ctrl+V`, `Ctrl+А` → `Ctrl+F`, `Ctrl+Ф` → `Ctrl+A`, etc. per the table.
- If the symbol is in the table, it is converted to the physical key; if not — the line is considered an error with a line number.
- Latin characters are always treated as the physical US keyboard position, not as "the letter on the current layout

# 6. Architecture and Technical Plan

### 6.1. New Types

```rust
// crates/ui-egui/src/shortcuts.rs (or shortcuts/ module as it grows)

use std::sync::Arc;

/// Action ID. UI strings are static; core command IDs (`cmd.<id>`) are collected
/// once during registry construction and interned as `Arc<str>`.
pub type ActionId = Arc<str>;

#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub enum Binding {
    Chord(KeyChord),
    Sequence([Key; 2]),
}

#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct KeyChord {
    pub key: Key,          // physical key, egui::Key
    pub ctrl: bool,        // primary: Ctrl on Win/Linux, Cmd on macOS
    pub shift: bool,
    pub alt: bool,
    pub meta: bool,        // literal Cmd/Super/Win
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum ActionKind {
    Tool,                  // sets Tool::…
    Dialog,                // opens an existing dialog
    Immediate,             // runs at once
    Input,                 // needs parameters; only bindable with a UI
    Internal,              // not user-bindable
}

pub struct ActionDef {
    pub id: ActionId,
    pub label: &'static str,          // English; t() localizes
    pub category: Category,
    pub kind: ActionKind,
    pub defaults: &'static [Binding], // or &'static [&'static str] parsed at startup
    pub synonyms: &'static [&'static str],
    pub enabled: fn(&BimApp) -> bool,
    pub execute: fn(&mut BimApp, &Context), // Context needed for Open/dialogs/file actions
}

pub struct Shortcuts {
    by_chord: HashMap<KeyChord, ActionId>,
    by_sequence: HashMap<[Key; 2], ActionId>,
    by_action: BTreeMap<ActionId, Vec<Binding>>,
    dirty: bool,
}
```

Notes:
- `Binding` must be `Hash + Eq` for hash tables.
- `defaults` can be stored as strings and parsed once at startup (easier to update), but the hot path should use the already parsed `Binding`.
- `ActionDef` is better collected in a `static`/`OnceLock` registry to avoid allocations every frame.
- `enabled` and `execute` are functions, not closures, so the registry can be `'static`.

### 6.2. Action Registry

1. Base list: a static array `UI_ACTIONS` (all UI actions, including current `ACTIONS`).
2. Core commands are added from `engine::commands()`:
   - ID: `cmd.<command.id>`;
   - label: `command.label`;
   - category: by `command.menu` or an explicit table;
   - kind: by an explicit `COMMAND_KINDS` table (Tool/Dialog/Immediate/Input/Internal);
   - aliases: `command.aliases` (become default sequences);
   - menu: `command.menu`.
3. Duplicate IDs are forbidden; `debug_assert!` and a test during construction.
4. Keep `shortcuts::ACTIONS` as a compatible list of names (for old files and Find), but make `registry::actions()` the source of truth.
5. Function `resolve_action(desc)`:
   - first searches by ID;
   - then by English label;
   - then by localized label (as it does now);
   - then by legacy names;
   - returns `ActionId`.
6. Function `execute_action(app, ctx, id) -> bool` replaces `execute_shortcut_action`. Inside: `match` on ID or call `def.execute(app, ctx)`. For core commands with ready parameters: `app.run(command.id, params)`; for tools: `app.set_tool(...)`; for dialogs: open the corresponding field. For commands with mandatory parameters (`open`, `insert`, `point_cloud`, `schedule`, `render`, `save` with a path), the executor opens an existing dialog/file picker (`guard(Pending::Open, ctx)`, etc.) rather than calling `run(id, {})`; if no such UI exists, the action is marked `Input`/`Internal` and cannot be bound.
7. Keep the old `execute_shortcut_action(action: &str)` as a thin wrapper during migration: `resolve_action(action).and_then(execute_action)`. After migrating all calls, remove it.

### 6.3. Input Handling in `lib.rs`

Replace the `keys()` block (around 1780–2050) with a pipeline:

```text
if modal_open { return }
if text_input_focused {
    handle_global_allowlist(); // Save/Save As/Open/New/Find/Settings
    return;
}
if menu_or_popup_open { return }

for event in key_press_events(!repeat) {
    let key = event.physical_key.unwrap_or(event.key);
    let chord = KeyChord::from(key, event.modifiers);
    if let Some(action) = shortcuts.chord(&chord) {
        consume_exact_event(event);
        shortcuts.clear_sequence();
        self.execute_action(ctx, action);
        continue;
    }
    if no_command_modifiers(event.modifiers) {
        shortcuts.push_sequence(key);
        if let Some(action) = shortcuts.take_sequence_if_matched() {
            self.execute_action(ctx, action);
        }
    }
}

// Contextual tool keys: Esc, Enter, Tab, Space, Backspace, arrows, typed length.
```

Implementation requirements:
- Do not use `consume_key` for chords: it ignores extra `Shift`/`Alt` (`matches_logically`). Use `matches_exact` and consume/mark the specific event.
- Account for `physical_key`; fallback to `key` (for web/some platforms).
- Ignore `repeat == true` for commands.
- Process `typed_length` before sequences.
- After a chord triggers, do not let the event reach `typed`.
- Keep `Escape`/`Enter`/`Tab`/`Space`/`Backspace`/arrows in existing contextual handling if they are not bound (they are reserved).
- Remove hardcoded `pressed(Modifiers::COMMAND, Key::D)` and others from list 2.2; their role is now handled by the registry and default profile.
- `Ctrl+D` no longer calls `toggle_home()`; Home is `Ctrl+Shift+H` *(Note: overridden by TS-009, see below)*.
- `Delete` can remain in both contextual handling and the registry; to avoid double deletion, use only the registry (or contextual handling must check that the binding didn't trigger). It is recommended to move `Delete` to the registry and remove the hardcoded check.
- Move `F8`/`F11` to the registry (`ui.view.alignment_guides`), keep as defaults.

### 6.4. Settings

`crates/ui-egui/src/settings.rs`:
- `Settings.shortcuts: ShortcutsConfig` (serializable struct).
- `shortcuts_from(v, base)`:
  - if `v` is an object with `version: 2` and a `bindings` array — parse it;
  - if `v` is an old `key → action` object — migrate it;
  - if `v` is missing/not an object — keep `base`.
- `MAX_SHORTCUTS` → 4096; add `MAX_ACTIONS = 512`.
- `use_settings`: `self.shortcuts = Shortcuts::from_config(kept.shortcuts)`.
- `settings_now`: `shortcuts: self.shortcuts.to_config()`.
- `keep_settings` already compares `Settings` and writes atomically; it is sufficient for `ShortcutsConfig` to implement `PartialEq`.
- During migration, maintain backward compatibility: old key `"WA"` → `Binding::Sequence([Key::W, Key::A])`; action `"Wall"` → `cmd.wall`.
- If a key in the old file is not two letters (which shouldn't happen) — skip it.

### 6.5. Parser and Serializer

- `parse_shortcuts(text: &str, registry: &Registry) -> ParsedProfile`.
- `serialize_shortcuts(profile: &Shortcuts, registry: &Registry, mode: ExportMode) -> String`.
- `parse_binding(s: &str) -> Result<Binding, BindingError>`.
- `display_binding(b: &Binding) -> String` — canonical form.
- `display_binding_localized(b, lang) -> String` — with `(Ctrl+В)` for Russian.
- `resolve_key(name: &str) -> Option<Key>`:
  - `Key::from_name` (egui);
  - synonym table (`Esc`, `Return`, `Option`, `Cmd`, `Primary`, punctuation);
  - Cyrillic table (Appendix C).
- `canonicalize(b: &Binding) -> Binding` — normalize modifiers and keys.
- The parser must not panic on any input; all errors go to `errors`.

### 6.6. Dialog

`crates/ui-egui/src/shortcuts.rs`:
- `ShortcutsDialog`:
  ```rust
  pub struct ShortcutsDialog {
      pub search: String,
      pub filter: Filter,
      pub category: Option<Category>,
      pub profile: ProfileChoice,
      pub pending: Shortcuts,
      pub original: Shortcuts,
      pub capture: Option<Capture>,
      pub import: Option<ImportPreview>,
      pub export: Option<ExportOptions>,
      pub error: Option<String>,
      pub message: Option<String>,
      pub dirty: bool,
  }
  ```
- `Capture { action: ActionId, current: Option<Binding>, keys: Vec<Key>, ... }`.
- The dialog works with `pending`, not directly with `BimApp::shortcuts`; "Apply" copies `pending` to the app and saves.
- "Cancel"/`Esc` restores `original`.
- For capture, use the same `Event::Key`, `physical_key`, and `matches_exact`.
- Capture must not trigger as a global binding: while `capture.is_some()`, app binding processing is disabled.
- Show conflicts via `find_conflict` on `pending`.
- Import/export via `rfd::AsyncFileDialog`, as currently done; filters `.txt`, `.bks`, `.json`, `.xml`.

### 6.7. Find, Ribbon, Menu

- `find.rs`:
  - `find_matches` gets actions from the registry;
  - for each action: `shortcuts::display_for(action.id)`, up to two bindings + `+N`;
  - search by binding string.
- `lib.rs`:
  - helper `fn shortcut_tip(action: ActionId, text: &str) -> String` adds ` (…)`;
  - helper `fn shortcut_text(action: ActionId) -> Option<String>`;
  - replace hardcoded labels in `big_button`/`small_button` and menus.
- `ribbon.rs` does not store commands; change only if a helper for strings is needed.
- `find.rs` and `shortcuts.rs` must not duplicate the action list.

### 6.8. Files to Change

| File | What to do |
| --- | --- |
| `crates/ui-egui/src/shortcuts.rs` | New types `Binding`, `KeyChord`, `ActionDef`, `Shortcuts`; registry; parser/serializer; Markdown help generator; dialog; capture; conflicts. If it grows, split into `shortcuts/{mod,chord,registry,file,markdown,dialog}.rs`. |
| `crates/ui-egui/src/lib.rs` | Replace `custom_shortcuts`/`typed` fields with `shortcuts: Shortcuts`; rewrite `keys`, `shortcut()`, `shortcuts()`; remove hardcoded `pressed`; `Ctrl+D` → Move; Home → `Ctrl+Shift+H` *(Note: overridden by TS-009)*; tooltip helpers. |
| `crates/ui-egui/src/settings.rs` | New `shortcuts` schema, migration, limits, `use_settings`, `settings_now`. |
| `crates/ui-egui/src/find.rs` | Show multiple bindings, search by binding. |
| `crates/core/src/engine.rs` | No changes, if `Command` is sufficient; if necessary, add `shortcut_kind`/`category` or keep the table in UI. |
| `crates/core/src/commands.rs` | No changes; verify that all 170 commands are classified in the UI registry. |
| `crates/ui-egui/src/lang.rs`, `translations/ui/*` | New strings, translations. |
| `assets/shortcuts/*.bks` (new resource) | Default profiles BIMCraft/Revit/Archicad/Empty; `include_str!`. |
| `docs/areas/desktop.md` | Rewrite the Keyboard Shortcuts section. |
| `docs/areas/shortcuts.md` (new) | User documentation and file format. |
| `README.md` | Update line 407 and the command/shortcut table. |
| `crates/ui-egui/tests/*` | New tests: `shortcuts_kept.rs`, `shortcuts_input.rs`, `shortcuts_file.rs`, `shortcuts_dialog.rs`, `shortcuts_md.rs`. |

### 6.9. What Does Not Change

- Core commands, their `id`, `run`, `params`, and project format.
- `settings.json` continues to store other settings; only the `shortcuts` field changes.
- Autosave, undo/redo, project saving.
- Tool behavior, except for the source of keystrokes.
- Localization of action names.

---

## 7. Pitfalls (Must be considered)

1. **`consume_key` ignores extra Shift/Alt.** In egui 0.36, `InputState::consume_key` uses `Modifiers::matches_logically`: `Ctrl+D` will trigger on `Ctrl+Shift+D` if `Ctrl+Shift+D` isn't checked first. For exact bindings, use `matches_exact` and consume the event manually. Otherwise, `Ctrl+D` and `Ctrl+Shift+D` cannot coexist.
2. **`Event::Text` is layout, not a key.** Never build chords/sequences from text. Only `Event::Key`.
3. **`physical_key` can be `None`.** On web and some platforms, fallback to `key`. For non-Latin layouts, egui itself substitutes the physical key into `key`, but relying solely on this is not advisable.
4. **Russian layout and `Ctrl+В`.** Verify that `Event::Key { key: Key::D (fallback), physical_key: Some(Key::D), modifiers: Ctrl }` and `text: "в"` trigger Move. A test is mandatory.
5. **AltGr.** On Windows, this is `Ctrl+Alt`; a user might bind `Ctrl+Alt+E`, and the system will generate the `€` symbol. If focus is in a text field, text has priority. On the canvas, warn the user.
6. **macOS `Cmd` vs `Ctrl`.** `Modifiers::COMMAND` on macOS is Cmd, on Windows/Linux it is Ctrl. The file `Ctrl` should mean primary, not literal Control. For literal Control on macOS, use `Control`? This is ambiguous. Decision: `Ctrl`/`Primary` = primary; `Control` is a synonym for primary for simplicity; literal Control on macOS is not supported in v1 (document this). `Cmd`/`Meta` = literal Command.
7. **Text fields.** `Ctrl+C`/`Ctrl+V`/`Ctrl+X`/`Ctrl+A`/`Ctrl+Z` must remain with the field. Do not intercept them even if the user bound them to an action. Exception: Save/Open/New/Find/Settings, which do not steal text.
8. **IME and dead keys.** While IME composition is active, do not process bindings; `ctx.egui_wants_keyboard_input()` and `Event::Ime` are indicators. Dead keys (accents) must not create a binding; use only `Event::Key`.
9. **Shift and letters.** `Shift+D` is `D` with `shift=true`; do not confuse it with `Ctrl+D`. For sequences, Shift should not change the key, nor should it turn a sequence into a chord.
10. **CapsLock.** Does not affect `Key`; do not use text to determine it.
11. **Key repeat.** Holding `Ctrl+D` must not trigger Move multiple times. Ignore `repeat`.
12. **Open menus and popups.** Keep the current early `return` when `Popup::is_any_open`; otherwise, menus will start executing commands.
13. **Double execution.** `Delete` is currently handled in contextual keys and might be bound. After moving to the registry, remove the hardcoded check or add an "already executed" flag.
14. **`Ctrl+D` conflict with Home.** Must reassign Home; otherwise, the user scenario fails. Old settings do not contain `Ctrl+D`, so migration is safe. *(Note: TS-009 overrides this: Home is `Ctrl+U`, `Ctrl+D` is free).*
15. **Duplicate aliases (`CL`).** `column` and `structural_column` share `CL`; we cannot pretend they are one command. In the registry, they are two IDs. In the hash table, `CL` belongs to `cmd.column`; the executor of `cmd.column` upon pressing `CL` looks at the active tab and, if `Tab::Structure`/`Context(Structure)`, chooses `cmd.structural_column`, as it does now. A user binding `CL` to another action disables this contextual rule for `CL` (the user explicitly overridden it). There must not be two owners of the same `CL` in the table.
16. **Commands with mandatory parameters.** `wall {points}`, `door {at}`, `schedule`, `render`, `open {path}`, etc. Cannot just call `run(id, {})` — it will error. For interactive tools, use `Tool`; for others, use the existing dialog or mark as "requires parameters".
17. **Localization and ID.** Do not store localized names as keys in settings; only IDs. Names are for import/search.
18. **Deterministic export.** Do not write `HashMap` to a file; use `BTreeMap`/sorting. Otherwise, diffs and round-trips will be unstable.
19. **Size and corrupted files.** Limits and atomic writes; do not block the UI on large files.
20. **Backward compatibility with old settings.** The old `settings.json` must not lose bindings; the new file must read without panicking.
21. **Cross-OS tests.** Reference strings with `Ctrl` on macOS might display as `Cmd`; in tests, compare the canonical model, not the platform-specific label.
22. **Performance.** Do not iterate over 4096 bindings on every keystroke; use hash tables.
23. **Security.** The file is data only; no `eval`, plugins, paths, or OS commands.
24. **User profiles.** Do not overwrite a user profile when selecting a default profile without confirmation.
25. **Reset.** "Restore Defaults" must not delete user bindings of other profiles if they are stored; in v1, only one active set is stored — clarify in the UI.
26. **Sketch.** Do not lose the contextual behavior of `Ctrl+Z` (undo sketch line) when moving Undo to the registry; check `self.sketch.is_some()` before the general binding.
27. **Note editing.** While `note_edit` is active and focus is not in a field, `note_keys` processes the keys; do not intercept them with global bindings.
28. **`typed_length`.** Digits, `.`, and `,` during length input must go to length, not to sequences; do not break existing logic.
29. **`Tab`.** The current code separately strips `Tab` from events for cyclic selection; preserve this behavior and do not treat `Tab` as a binding.

---

## 8. Tests

### 8.1. Unit Tests (`crates/ui-egui`)

1. `parse_binding("Ctrl+D")` → `Chord { key: D, ctrl: true }`.
2. `parse_binding("Ctrl+В")` → same as `Ctrl+D`.
3. `parse_binding("WA")` → `Sequence([W, A])`.
4. `parse_binding("WА")` (Cyrillic А) → `Sequence([W, A])`.
5. `parse_binding("Ctrl+Shift+D")` ≠ `Ctrl+D`; both exist.
6. `parse_binding("Ctrl++")` → error/`Plus` per rule.
7. `parse_binding("Esc")` → error "reserved" (or valid key, but forbidden at assignment level — clarify; test must capture the decision).
8. `display_binding` for all `Key::ALL` keys does not panic.
9. `resolve_action` by ID, English label, localized label, legacy name.
10. Registry: all IDs are unique; all `engine::commands()` are covered or in exceptions; all `ActionDef` with `kind != Internal` have `execute`.
11. Default profiles: no conflicts; `Ctrl+D` → `cmd.move`; Home → `Ctrl+Shift+H` *(Note: overridden by TS-009)*.
12. `find_conflict` for chords, sequences, default, and user bindings.
13. `add_binding`: multiple bindings per action; conflict; replacement; deletion.
14. `parse_shortcuts` v2: comments, CRLF, BOM, duplicate actions, multiple bindings.
15. `parse_shortcuts` legacy JSON/XML/`KEY - Name`.
16. `parse_shortcuts` errors: unknown action, unknown key, conflict, version 3, file too large.
17. `serialize` → `parse` → semantic equality; re-export is identical.
18. `Settings::from_json` old `{"WA":"Wall"}` → new `Shortcuts`; new JSON → new; corrupted → `base`.
19. `MAX_SHORTCUTS`/`MAX_ACTIONS` are respected.
20. `enabled` for actions without selection/in inappropriate views.
21. Markdown help contains **every** registry action, including commands without keys; table row count matches action count.
22. Markdown help distinguishes active and default bindings; modified/deleted bindings are visible as such.
23. Markdown help is valid: tables with escaped `|`, no raw HTML, headers and sections in place.
24. Markdown help is deterministic (with "no date" option); repeated generation yields the same text.
25. Importing `.md` is rejected with a clear message, settings remain unchanged.

### 8.2. Integration Tests (`egui_kittest`)

1. `Ctrl+D` (physical `Key::D`, modifiers Ctrl) → active tool `Tool::Move`.
2. `Ctrl+В` (physical `Key::D`, `Event::Text("в")`) → `Tool::Move`.
3. `Ctrl+D` on Russian layout without text event → `Tool::Move`.
4. `Ctrl+Shift+D` → Copy, not Move; `Ctrl+D` → Move.
5. `Ctrl+D` with focus in a text field → not Move; text `d`/`в` is typed.
6. `Ctrl+C` with focus in a field → copy text, not a command.
7. `Ctrl+S` with focus in a field → Save (allowlist).
8. Modal dialog open → `Ctrl+D` not Move; `Esc` closes dialog.
9. Menu open → `Ctrl+D` not Move; `Esc` closes menu.
10. `WA` on Russian layout (physical W, A) → Wall.
11. `WA` with timeout: after a pause, the buffer is cleared (if timeout is implemented).
12. Holding `Ctrl+D` (repeat) → Move once.
13. Assigning two bindings via dialog → both work.
14. Conflict in dialog → "Replace" removes it from the previous action.
15. "Apply" saves to `settings.json`; "Cancel" does not save.
16. Import "Replace all" → old bindings disappear, new ones work.
17. Import "Add" → conflicts skipped, report shown.
18. Export → file exists, UTF-8, contains `version = 2` and all bindings.
19. Export → import → export → identical text.
20. Find shows multiple bindings; searching for `Ctrl+D` finds Move.
21. Ribbon tooltip contains `Ctrl+D` after assignment.
22. New app launch with saved `settings.json` → bindings are in place.
23. Old `settings.json` → migration, old `WA` works.
24. Action unavailable → status, no fallback.
25. Import file with `Ctrl+В` → `Ctrl+D` works.
26. "Help (Markdown)…" button saves `.md`; file opens in a Markdown viewer and contains all sections.
27. Help includes commands without keys and default bindings, even if the user changed them.
28. In Russian UI help, there are hints like `Ctrl+D (Ctrl+В)`.
29. Importing `.md` shows a message "this is help documentation" and does not touch settings.

### 8.3. Manual Verification (Matrix)

| OS | Layouts | What to check |
| --- | --- | --- |
| Linux | US, RU | `Ctrl+D`/`Ctrl+В`, `WA`, text in fields, AltGr |
| Windows | US, RU | `Ctrl+D`/`Ctrl+В`, AltGr, `Ctrl+Shift+Esc` |
| macOS | US, RU | `Cmd+D` (primary), literal `Ctrl`, `Cmd+Tab` |
| Any | live layout switching | bindings do not change |
| Any | external keyboard | Numpad, F-keys |

Manual checks:
- all ribbon buttons with a binding show it in the tooltip;
- File/Edit/Manage/View menus show bindings;
- Find (Ctrl+K) shows bindings and searches by them;
- import/export via system dialog;
- reset action/category/all;
- no double execution of `Delete`;
- no false triggers when typing name, size, text.

### 8.4. Quality Control Commands

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test -p bimcraft-ui-egui
cargo test -p bimcraft-core
cargo test --workspace
```

For UI tests — `egui_kittest` following the existing pattern (`crates/ui-egui/tests/`).

---

## 9. Documentation and Translations

1. `README.md`, line 407: rewrite the Shortcuts section — chords, multiple bindings, import/export, `Ctrl+D` = Move, layout independence.
2. `docs/areas/desktop.md`, lines 479–488: update description of `shortcuts_window`, data models, `settings.json`.
3. New `docs/areas/shortcuts.md`:
   - user instructions;
   - file format with examples;
   - Russian layout table;
   - list of reserved keys;
   - migration from the old format;
   - section "How to view all combinations": Manage > Shortcuts > "Help (Markdown)…"; what goes into the file and how to read it.
4. In README and app help — a short instruction: where to find the full list, how `.md` differs from `.bks` (help vs backup).
5. `translations/ui/bimcraft-ui.pot` and language files: all new strings.
6. `docs/video/` — optional short video/gif, if accepted in the project.
7. CHANGELOG/ROADMAP — note the change of `Ctrl+D` (Home → Move) as noticeable to the user.

---

## 10. Acceptance Criteria

> **TS-009 (Priority):** the criteria below regarding `Ctrl+D` = Move and Home = `Ctrl+Shift+H` are replaced: in the BIMCraft profile, `Ctrl+D` is free, Home is `Ctrl+U`.

1. In the hotkeys dialog, **any user-available command** can be assigned; all 170 core commands are present in the registry (assignable or with an explicit reason).
2. **Multiple** combinations can be assigned to one command; all of them work.
3. `Ctrl+D` executes "Move"; Home is moved to `Ctrl+Shift+H` *(Note: overridden by TS-009)*.
4. `Ctrl+D` on English and `Ctrl+В` on Russian layouts perform the same action.
5. Two-letter sequences `WA`, `DR`, `MV`, etc., work on the Russian layout (by physical keys).
6. `Ctrl+D` and `Ctrl+Shift+D` are different bindings; the more specific one is not masked.
7. A conflict of one chord on two actions is detected; the owner can be replaced.
8. The dialog allows adding, changing, deleting a single binding, deleting all, resetting an action/category/all.
9. Export to `.txt`/`.bks` (and `.json`) contains all bindings and the version; the file is human-readable.
10. Importing a file in "Replace" and "Add" modes works; errors are shown with line numbers; valid lines are applied.
11. Export → import → export yields a semantically identical set.
12. Bindings are saved in `settings.json` and survive a restart; the old format migrates without losing `WA`, etc.
13. Text fields are not intercepted; `Ctrl+C`/`Ctrl+V`/`Ctrl+Z` work in them as usual.
14. Modal dialogs and menus do not execute global bindings.
15. Ribbon, menu, and Find tooltips show assigned combinations (up to two + `+N`).
16. No command is executed twice; holding a key does not repeat the action.
17. All new strings are localized; the Russian UI shows `Ctrl+D (Ctrl+В)`.
18. Tests from section 8 pass; `cargo fmt`, `clippy`, `test --workspace` are green.
19. Documentation is updated; `docs/areas/shortcuts.md` describes the format and limitations.
20. No regressions: autosave, undo/redo, project saving, existing tools work as before.
21. There is a simple way to familiarize with all combinations: the "Help (Markdown)…" button (and optionally, a preview) generates a `.md` with all actions — active, default, and without keys, grouped by categories.
22. The Markdown file is valid, readable in GitHub/VS Code/Obsidian, contains conflicts and reserved keys; importing `.md` is explicitly rejected with a hint to use `.bks`/`.txt`/`.json`.

---

## 11. Phased Plan

### Stage 0. Registry without behavior change
- Extract `ACTIONS`/`SHORTCUTS`/aliases into `ActionDef`.
- Classify the 170 commands.
- `execute_action` as a wrapper over the current `execute_shortcut_action`.
- Coverage and ID uniqueness tests.
- Behavior does not change; old tests are green.

### Stage 1. Binding model and input
- `Binding`, `KeyChord`, `Shortcuts`.
- Exact match, `physical_key`, sequences.
- Remove hardcoded `pressed`; `Ctrl+D` → Move, Home → `Ctrl+Shift+H` *(Note: overridden by TS-009)*.
- Settings migration.
- Input tests, including `Ctrl+В`.

### Stage 2. Dialog
- New `ShortcutsDialog` with multiple bindings, search, filters, conflicts, capture.
- "Apply/Cancel".
- Dialog tests.

### Stage 3. Import/Export
- Parser/serializer v2, legacy, JSON, XML.
- Import preview, modes, errors.
- Atomic writes, backup.
- Markdown help generator: all actions, active + default bindings, commands without keys, grouping, conflicts.
- "Export → import → export" tests and Markdown help tests.

### Stage 4. Profiles, tooltips, documentation
- Built-in profiles BIMCraft/Revit/Archicad/Empty.
- Ribbon/menu/Find tooltips.
- "Help (Markdown)…" button and, optionally, preview.
- README, docs, translations.
- Manual matrix.

### Stage 5. Polish and Quality Control
- Performance, limits, corrupted files.
- Verification on Windows/Linux/macOS.
- Final acceptance criteria.

---

## 12. Out of Scope

- Macros (sequences longer than two keys, action recording, scripts).
- Contextual bindings for specific views/tools (one global table per app).
- Project-level bindings (only app settings).
- Cloud profile synchronization.
- OS-level key remapping.
- Mouse gestures and game controllers.
- Mobile/web versions (desktop only; `physical_key` may be missing on web).
- Export to Archicad/Revit format as a mandatory requirement (Revit XML is optional, sequences only).
- Markdown import: `.md` is for reading help only; use `.bks`/`.txt`/`.json` for import.
- Automatic detection of the active OS layout for key labels (we use the JCUKEN table and UI language setting).
- Changing core commands and their parameters.

---

## Appendix A. Full Catalog of Core Commands (170)

Below is a snapshot of `engine::commands()` as of 2026-11-08. The "Menu" column shows where the command is available in the UI; empty means the command exists in the API/CLI/MCP but may not have a button. The implementation must classify every row (Tool/Dialog/Immediate/Input/Internal) and not lose any.

| # | id | Command (label) | Aliases | Menu |
| ---:| --- | --- | --- | --- |
| 1 | `new` | New | — | — |
| 2 | `open` | Open | — | — |
| 3 | `highrise` | Synthetic Highrise | — | — |
| 4 | `tower` | Reference Tower | — | — |
| 5 | `autosave` | Autosave | — | — |
| 6 | `recover` | Recover Autosave | — | — |
| 7 | `insert` | Insert from File | — | — |
| 8 | `link` | Link Model | LK | Manage > Link Model |
| 9 | `reload` | Reload Links | — | — |
| 10 | `save` | Save | — | — |
| 11 | `undo` | Undo | — | — |
| 12 | `redo` | Redo | — | — |
| 13 | `level` | Level | LL | Architecture > Level |
| 14 | `grid` | Grid | GR | Architecture > Grid |
| 15 | `point_cloud` | Point Cloud | — | — |
| 16 | `import_cad` | Import CAD | — | — |
| 17 | `cad_layers` | CAD Layers | — | — |
| 18 | `reload_cad` | Reload CAD | — | — |
| 19 | `cad_query` | Query | — | — |
| 20 | `wall` | Wall | WA | Architecture > Wall |
| 21 | `door` | Door | DR | Architecture > Door |
| 22 | `window` | Window | WN | Architecture > Window |
| 23 | `floor` | Floor | — | Architecture > Floor |
| 24 | `ceiling` | Ceiling | — | Architecture > Ceiling |
| 25 | `sketch` | Sketch | — | — |
| 26 | `shaft` | Shaft Opening | — | Architecture > Shaft |
| 27 | `edit_boundary` | Edit Boundary | — | — |
| 28 | `edit_shape` | Modify Sub Elements | — | — |
| 29 | `add_point` | Add Point | — | — |
| 30 | `add_split_line` | Add Split Line | — | — |
| 31 | `reset_shape` | Reset Shape | — | — |
| 32 | `falls` | Falls | — | — |
| 33 | `mass` | Mass | MS | — |
| 34 | `mass_floors` | Mass Floors | MF | — |
| 35 | `wall_by_face` | Wall by Face | WB | — |
| 36 | `floor_by_face` | Floor by Face | FB | — |
| 37 | `roof_by_face` | Roof by Face | RB | — |
| 38 | `update_by_face` | Update to Face | UF | — |
| 39 | `show_mass` | Show Mass | MH | — |
| 40 | `room` | Room | RM | Architecture > Room |
| 41 | `room_separator` | Room Separator | — | — |
| 42 | `rebar` | Rebar | — | — |
| 43 | `reinforce` | Reinforce | — | — |
| 44 | `reinforcement_preset` | Reinforcement Presets | — | — |
| 45 | `voltage` | Voltage Definitions | — | — |
| 46 | `distribution_system` | Distribution Systems | — | — |
| 47 | `wire_type` | Wire Types | — | — |
| 48 | `column` | Column | CL | Architecture > Column |
| 49 | `structural_column` | Structural Column | CL | — |
| 50 | `footing` | Isolated Footing | FI | — |
| 51 | `beam` | Beam | BM | — |
| 52 | `strip_footing` | Wall Footing | FW | — |
| 53 | `pipe` | Pipe | PI | — |
| 54 | `duct` | Duct | DU | — |
| 55 | `conduit` | Conduit | CN | — |
| 56 | `cable_tray` | Cable Tray | CT | — |
| 57 | `wire` | Wire | EW | — |
| 58 | `down_conductor` | Down Conductor | DC | — |
| 59 | `ground_conductor` | Earthing Conductor | GC | — |
| 60 | `cabling` | Cabling | CB | — |
| 61 | `stair` | Stair | — | Architecture > Stair |
| 62 | `ramp` | Ramp | — | Architecture > Ramp |
| 63 | `railing` | Railing | — | Architecture > Railing |
| 64 | `roof` | Roof | — | Architecture > Roof |
| 65 | `dormer` | Dormer | — | Architecture > Dormer |
| 66 | `dimension` | Aligned Dimension | DI | — |
| 67 | `dimension_3d` | Aligned Dimension in 3D | — | — |
| 68 | `text` | Text | TX | — |
| 69 | `text_type` | Text Types | — | — |
| 70 | `dimension_constraint` | Dimension Constraints | — | — |
| 71 | `dimension_text` | Dimension Text | — | — |
| 72 | `dimension_type` | Dimension Types | — | — |
| 73 | `spot` | Spot Elevation | EL | — |
| 74 | `spot_slope` | Spot Slope | — | — |
| 75 | `tag` | Tag by Category | TG | — |
| 76 | `section` | Section | CR | — |
| 77 | `elevation` | Elevation | — | — |
| 78 | `sheet` | Sheet | — | — |
| 79 | `place` | Place View | — | — |
| 80 | `crop` | Crop View | — | — |
| 81 | `duplicate_view` | Duplicate View | DP | — |
| 82 | `scale` | View Scale | — | — |
| 83 | `detail` | Detail Level | — | — |
| 84 | `visual_style` | Visual Style | — | — |
| 85 | `type` | Edit Type | — | — |
| 86 | `material` | Materials | — | Manage > Materials |
| 87 | `favorite` | Favorites | — | Manage > Favorites |
| 88 | `paint` | Paint | PT | — |
| 89 | `parameter` | Project Parameters | — | Manage > Project Parameters |
| 90 | `family` | Load Family | — | Manage > Family Editor |
| 91 | `symbol_set` | Load Symbol Set | — | — |
| 92 | `component` | Component | CM | Architecture > Component |
| 93 | `flip` | Flip | — | — |
| 94 | `modify` | Modify | — | — |
| 95 | `match_properties` | Match Properties | MA | — |
| 96 | `align` | Align | AL | — |
| 97 | `dimension_value` | Dimension Value | — | — |
| 98 | `move` | Move | MV | — |
| 99 | `copy` | Copy | CO | — |
| 100 | `rotate` | Rotate | RO | — |
| 101 | `mirror` | Mirror | MM, DM | — |
| 102 | `corner` | Trim/Extend to Corner | TR | — |
| 103 | `offset` | Offset | OF | — |
| 104 | `trim_extend` | Trim/Extend Single Element | — | — |
| 105 | `split` | Split | SL | — |
| 106 | `join_polyline` | Join into Polyline | PJ | — |
| 107 | `fillet` | Fillet Arc | FL | — |
| 108 | `vertex` | Add/Delete Vertex | — | — |
| 109 | `convert_spline` | Convert Spline | — | — |
| 110 | `explode` | Explode Polyline | EX | — |
| 111 | `delete` | Delete | DE | — |
| 112 | `line_pattern` | Line Patterns | — | — |
| 113 | `line_style` | Line Styles | — | Manage > Line Styles |
| 114 | `fill_pattern` | Fill Patterns | — | Manage > Fill Patterns |
| 115 | `make_fill_pattern` | Make Fill Pattern | — | — |
| 116 | `linework` | Linework | LW | — |
| 117 | `import_fill_patterns` | Import Fill Patterns | — | — |
| 118 | `export_crosscheck` | Export Cross-Check | — | — |
| 119 | `compare_crosscheck` | Compare Cross-Check | — | — |
| 120 | `export_fill_patterns` | Export Fill Patterns | — | — |
| 121 | `filled_region_type` | Filled Region Types | — | — |
| 122 | `filled_region` | Filled Region | FR | — |
| 123 | `create_group` | Create Group | GP | — |
| 124 | `detail_group` | Place Detail Group | — | — |
| 125 | `repeating_detail` | Repeating Detail | — | — |
| 126 | `ungroup` | Ungroup | UG | — |
| 127 | `group_type` | Detail Groups | — | — |
| 128 | `detail_line` | Detail Line | DL | — |
| 129 | `delete_type` | Delete Type | — | — |
| 130 | `rename_type` | Rename Type | — | — |
| 131 | `purge` | Purge Unused | PU | Manage > Purge Unused |
| 132 | `line_weights` | Line Weights | — | Manage > Line Weights |
| 133 | `object_styles` | Object Styles | — | Manage > Object Styles |
| 134 | `units` | Project Units | UN | Manage > Project Units |
| 135 | `room_area` | Area and Volume Computations | — | Manage > Room Area Computation |
| 136 | `georeference` | Project Location | — | — |
| 137 | `language` | Project Language | — | Manage > Project Language |
| 138 | `standard` | Technical Standards | — | Manage > Technical Standards |
| 139 | `inspect` | Inspect | — | — |
| 140 | `render` | Render | — | — |
| 141 | `schedule` | Schedule | — | — |
| 142 | `graphics` | Visibility/Graphics | — | — |
| 143 | `section_box` | Section Box | — | — |
| 144 | `perspective` | Perspective | — | — |
| 145 | `graphic_display_options` | Graphic Display Options | GD | — |
| 146 | `camera` | Camera | — | — |
| 147 | `work_plane` | Work Plane | — | — |
| 148 | `filter` | View Filter | — | — |
| 149 | `template` | View Template | — | — |
| 150 | `phase` | Phasing | — | Manage > Phasing |
| 151 | `phase_filter` | Phase Filter | — | Manage > Phasing |
| 152 | `option` | Design Options | — | Manage > Design Options |
| 153 | `measure` | Measure | ME | — |
| 154 | `circuit` | Size a Circuit | — | — |
| 155 | `conduit_fill` | Conduit Fill | — | — |
| 156 | `drain_size` | Size a Sanitary Drain | — | — |
| 157 | `vent_size` | Size a Sanitary Vent | — | — |
| 158 | `hydrant_size` | Size Hydrants | — | — |
| 159 | `load_points` | Residential Load Points | — | — |
| 160 | `dr_check` | Check Residual-Current Protection | — | — |
| 161 | `dps_check` | Check Surge Protection | — | — |
| 162 | `demand` | Demand and Service Category | — | — |
| 163 | `water_size` | Size Cold Water Pipes | — | — |
| 164 | `reservoir_size` | Size Water Reservation | — | — |
| 165 | `storm_size` | Size a Storm Drain | — | — |
| 166 | `library` | Library | — | — |
| 167 | `select_by_id` | Select by ID | — | Manage > Select by ID |
| 168 | `ids_of` | IDs of Selection | — | Manage > IDs of Selection |
| 169 | `filter_selection` | Filter | — | — |
| 170 | `elements` | Elements | — | — |

---

## Appendix B. Default BIMCraft Profile: New Chords

> **TS-009 (Priority):** the rows with `Ctrl+D` = Move and Home = `Ctrl+Shift+H` in this table do not apply to BIMCraft: `Ctrl+D` is free, Home is `Ctrl+U`.

Below is the **normative** part of the default BIMCraft profile. All old two-letter sequences (`WA`, `DR`, `WN`, `RM`, `CL`, `DI`, `TG`, `MV`, `CO`, `RO`, `MM`, `DM`, `AL`, `OF`, `DW`, `ZF`, `TL`, `CS`, `PP`, `NP`, `SA`, `VV`, `EH`, `HH`, `KS`, `UN`, `GD`, …) are preserved as default bindings and are not listed row by row to avoid duplicating `SHORTCUTS` and command aliases.

| Action | ID | Default Chords | Note |
| --- | --- | --- | --- |
| Move | `cmd.move` | `Ctrl+D`, `MV` | `Ctrl+D` is a user requirement; `MV` is for compatibility *(Note: overridden by TS-009)* |
| Copy | `cmd.copy` | `Ctrl+Shift+D`, `CO` | We do not take `Ctrl+C`: it belongs to text fields |
| Rotate | `cmd.rotate` | `Ctrl+R`, `RO` | as in Archicad |
| Mirror (pick axis) | `cmd.mirror` | `Ctrl+M`, `MM` | on macOS `Cmd+M` might be "minimize window" — warning |
| Mirror (draw axis) | `cmd.mirror` (draw mode) | `Ctrl+Shift+M`, `DM` | separate ID if mode is split |
| Offset | `cmd.offset` | `Ctrl+Shift+O`, `OF` | |
| Trim/Extend to Corner | `cmd.corner` | `Ctrl+T`, `TR` | |
| Split | `cmd.split` | `Ctrl+Shift+T`, `SL` | |
| Align | `cmd.align` | `Ctrl+Shift+A`, `AL` | |
| Delete | `cmd.delete` | `Delete`, `DE` | `Delete` is the only contextual key moved to the registry |
| Undo | `cmd.undo` | `Ctrl+Z` | |
| Redo | `cmd.redo` | `Ctrl+Shift+Z`, `Ctrl+Y` | |
| Save | `cmd.save` | `Ctrl+S` | |
| Save As | `ui.file.save_as` | `Ctrl+Shift+S` | separate behavior: path selection |
| Open | `cmd.open` | `Ctrl+O` | |
| New | `cmd.new` | `Ctrl+N` | |
| Find | `ui.find` | `Ctrl+K` | |
| Settings | `ui.settings` | `Ctrl+,` | |
| Properties | `ui.palette.properties` | `Ctrl+1`, `PP`, `VP` | |
| Project Browser | `ui.palette.browser` | `Ctrl+2`, `NP` | `Ctrl+2` is a new default chord |
| Export PDF | `ui.file.export_pdf` | `Ctrl+P` | |
| Start Page / Home | `ui.view.home` | `Ctrl+Shift+H` | moved from `Ctrl+D` *(Note: overridden by TS-009)* |
| Shortcuts | `ui.settings.shortcuts` | `Ctrl+Shift+K`, `KS` | `Ctrl+K` is taken by Find |
| 3D View | `ui.view.3d` | `F3`, `3D` | |
| Floor Plan | `ui.view.plan` | `F2` | |
| Zoom Fit | `ui.view.zoom_fit` | `Ctrl+0`, `ZF`, `ZE`, `ZA` | |
| Thin Lines | `ui.view.thin_lines` | `TL` | |
| Alignment Guides | `ui.view.alignment_guides` | `F8`, `F11` | |
| Visibility/Graphics | `ui.view.vg` | `VV`, `VG` | |
| Hide Element | `ui.view.hide_element` | `EH` | |
| Hide Category | `ui.view.hide_category` | `VH` | |
| Temp Hide/Isolate | `ui.view.temp_hide_*` | `HH`, `HC`, `HI`, `IC`, `HR`, `RH` | |
| Unhide | `ui.view.unhide` | `EU`, `UE`, `VU` | |
| Visual Style | `ui.view.visual_style` | `WF`, `HL`, `SD` | |
| Graphic Display Options | `ui.view.display_options` | `GD` | |
| Snaps | `ui.snap.*` | `SE`, `SM`, `SI`, `SC`, `SP`, `SQ`, `ST`, `SN`, `SO`, `SS` | |
| Wall | `cmd.wall` | `WA` | chord is not assigned by default: `Ctrl+W` on macOS closes the window; user can assign their own |
| Door | `cmd.door` | `DR` | |
| Window | `cmd.window` | `WN` | |
| Room | `cmd.room` | `RM` | |
| Column | `cmd.column` / `cmd.structural_column` | `CL` | contextually, as it is now |
| Dimension | `cmd.dimension` | `DI` | |
| Tag | `cmd.tag` | `TG` | |
| Text | `cmd.text` | `TX` | |
| Section | `cmd.section` | `CR` | |
| Measure | `cmd.measure` | `ME` | |

If the implementation splits `Mirror Pick Axis` and `Mirror Draw Axis` into two actions, `Ctrl+M` and `Ctrl+Shift+M` are assigned to them; if it leaves it as one action with a `draw` parameter, both bindings lead to `cmd.mirror` with different modes. The decision is to be fixed in the registry and test.

---

## Appendix C. Russian Layout Table (JCUKEN → Physical Key)

During import and when typing `Ctrl+В`/`Ctrl+С`/…, the symbol is converted to the US layout physical key. The table applies to letters and `ё`; for other symbols, `Key::from_name` and synonyms are used.

| Physical Key (US) | Russian lowercase | Russian uppercase |
| --- | --- | --- |
| `Q` | `й` | `Й` |
| `W` | `ц` | `Ц` |
| `E` | `у` | `У` |
| `R` | `к` | `К` |
| `T` | `е` | `Е` |
| `Y` | `н` | `Н` |
| `U` | `г` | `Г` |
| `I` | `ш` | `Ш` |
| `O` | `щ` | `Щ` |
| `P` | `з` | `З` |
| `[` | `х` | `Х` |
| `]` | `ъ` | `Ъ` |
| `A` | `ф` | `Ф` |
| `S` | `ы` | `Ы` |
| `D` | `в` | `В` |
| `F` | `а` | `А` |
| `G` | `п` | `П` |
| `H` | `р` | `Р` |
| `J` | `о` | `О` |
| `K` | `л` | `Л` |
| `L` | `д` | `Д` |
| `;` | `ж` | `Ж` |
| `'` | `э` | `Э` |
| `Z` | `я` | `Я` |
| `X` | `ч` | `Ч` |
| `C` | `с` | `С` |
| `V` | `м` | `М` |
| `B` | `и` | `И` |
| `N` | `т` | `Т` |
| `M` | `ь` | `Ь` |
| `,` | `б` | `Б` |
| `.` | `ю` | `Ю` |
| `/` | `.` | `,` |
| `` ` `` | `ё` | `Ё` |

Examples:
- `Ctrl+В` → `Ctrl+D`;
- `Ctrl+С` → `Ctrl+C`;
- `Ctrl+М` → `Ctrl+V`;
- `Ctrl+А` → `Ctrl+F`;
- `Ctrl+Ф` → `Ctrl+A`;
- `Ctrl+Ж` → `Ctrl+;`;
- `Ctrl+Ю` → `Ctrl+.`.

Note: Cyrillic `А` and `Ф` are **different** physical keys (`F` and `A` respectively), just as `М` and `Ь` are (`V` and `M`). Therefore, `Ctrl+А` and `Ctrl+Ф` do not conflict, and `Ctrl+М` is `Ctrl+V`.

If a symbol is missing from the table, import reports an error with the line number; the app does not guess.

---

## Appendix D. Reserved and System Combinations

### D.1. Hard Reserved by the Application

| Key | Why |
| --- | --- |
| `Esc` | Cancel/close dialog, tool, menu |
| `Enter` | Finish length input, construction, confirm |
| `Tab` | Cyclic selection, focus |
| `Backspace` | Delete last character/step |
| `Up`/`Down`/`Left`/`Right` | Navigation, movement, modification |
| `Space` | Contextual tool action: flip, component rotation, size mode |
| Single `Ctrl`, `Shift`, `Alt`, `Meta` | Not a binding |

### D.2. Assignable, but with Warning

| Combination | Risk |
| --- | --- |
| `Delete` | May conflict with deletion in text fields; in a field, the field has priority |
| `F1`–`F12` | Some may be occupied by OS/window/driver |
| `Ctrl+Alt+…` | AltGr on Windows; may generate a symbol |
| `Ctrl+Shift+Esc` | Windows Task Manager |
| `Alt+F4`, `Alt+Tab`, `Ctrl+Esc` | Intercepted by OS |
| `Cmd+Tab`, `Cmd+Space`, `Cmd+Q`, `Cmd+H`, `Cmd+M`, `` Cmd+` `` | Intercepted by macOS |
| `Win+L`, `Win+D`, `Win+Tab`, `Win+...` | Intercepted by Windows |
| `Ctrl+Alt+Del` | Does not reach the application |

When assigning such combinations, show a yellow warning "may not work: combination is intercepted by the system". During import — accept with a warning. Hard reserved (D.1) — do not accept and explain the reason.

---

## Appendix E. Minimal `settings.json` Example After Migration

> **TS-009 (Priority):** the example below shows `Ctrl+D` on Move and `Ctrl+Shift+H` on Home; per TS-009, in BIMCraft `Ctrl+D` is free, Home is `Ctrl+U`.

```json
{
  "shortcuts": {
    "version": 2,
    "profile": "bimcraft",
    "bindings": [
      {"action": "cmd.move", "keys": ["Ctrl+D", "Ctrl+Shift+M", "MV"]},
      {"action": "cmd.wall", "keys": ["WA", "Ctrl+Shift+E"]},
      {"action": "cmd.door", "keys": ["DR"]},
      {"action": "cmd.save", "keys": ["Ctrl+S"]},
      {"action": "ui.view.home", "keys": ["Ctrl+Shift+H"]}
    ]
  }
}
```

Old fragment that must migrate without loss:

```json
{
  "shortcuts": {
    "WA": "Wall",
    "DR": "Door",
    "ZW": "Floor"
  }
}
```

After migration:

```json
{
  "shortcuts": {
    "version": 2,
    "profile": "custom",
    "bindings": [
      {"action": "cmd.wall", "keys": ["WA"]},
      {"action": "cmd.door", "keys": ["DR"]},
      {"action": "cmd.floor", "keys": ["ZW"]}
    ]
  }
}
```

Note: in the old format, `"ZW"` could be a user key for `Floor`; in the new one, it becomes the sequence `ZW`. If the `Floor` action is not found, the line is skipped with a warning, and the rest are loaded.

---

## Appendix F. Implementation Checklist

- [ ] Action registry covers all 170 commands and all UI actions.
- [ ] `Ctrl+D` = Move; Home = `Ctrl+Shift+H` *(Note: overridden by TS-009)*.
- [ ] `Ctrl+В` = `Ctrl+D`; `WA` works on the Russian layout.
- [ ] Multiple bindings per action; all work.
- [ ] Exact modifier match; `Ctrl+D` ≠ `Ctrl+Shift+D`.
- [ ] Text fields are not intercepted.
- [ ] Modal dialogs/menus do not execute global bindings.
- [ ] Dialog: add/change/delete/reset, search, filters, conflicts.
- [ ] Import/export `.txt`/`.bks`/`.json`; preview, errors, round-trip.
- [ ] Export "Help (Markdown)…": all actions, active + default bindings, commands without keys, conflicts, grouping, valid Markdown.
- [ ] Importing `.md` is rejected with the message "this is help documentation".
- [ ] `settings.json` v2 + migration of the old format.
- [ ] Ribbon/menu/Find tooltips show multiple bindings.
- [ ] Translations, README, `docs/areas/desktop.md`, new `docs/areas/shortcuts.md`.
- [ ] Tests from section 8; `fmt`, `clippy`, `test --workspace` are green.
- [ ] No regressions in autosave/undo/project saving.

---

## Appendix G. Markdown Help Example

Below is a non-normative template, an example of what the user sees after clicking "Help (Markdown)…". The composition of sections and columns can be refined, but the principle must remain: **all actions**, **active and default bindings**, **commands without keys**, grouping, conflicts, reserved keys.

````markdown
# BIMCraft — Hotkeys

Profile: BIMCraft · Version: 2 · Platform: Linux · Generated: 2026-11-08

## How to Read

- **Active** — combinations that will trigger right now.
- **Default** — built-in values; they are visible even if you changed or removed them.
- **Source** — "default", "user", "modified", "deleted".
- `Ctrl+D` and `Ctrl+В` are the same physical key; the help shows the canonical form and a hint for the Russian layout.

## Contents

- [File](#file)
- [Edit](#edit)
- [Architecture](#architecture)
- [Views](#views)
- [Commands Without Keys](#commands-without-keys)
- [Reserved and System Combinations](#reserved-and-system-combinations)
- [Conflicts](#conflicts)

## File

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| Save | `cmd.save` | `Ctrl+S` | `Ctrl+S` | default | |
| Save As | `ui.file.save_as` | `Ctrl+Shift+S` | `Ctrl+Shift+S` | default | |
| Open | `cmd.open` | `Ctrl+O` | `Ctrl+O` | default | requires path, opens dialog |
| New | `cmd.new` | `Ctrl+N` | `Ctrl+N` | default | |

## Edit

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| Move | `cmd.move` | `Ctrl+D` (Ctrl+В), `Ctrl+Shift+M`, `MV` | `MV` | user | `Ctrl+D` — as in Archicad |
| Copy | `cmd.copy` | `Ctrl+Shift+D`, `CO` | `CO` | user | We do not take `Ctrl+C` |
| Rotate | `cmd.rotate` | `Ctrl+R`, `RO` | `RO` | user | |
| Delete | `cmd.delete` | `Delete`, `DE` | `DE` | user | in a text field, `Delete` belongs to the field |

## Architecture

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| Wall | `cmd.wall` | `WA` | `WA` | default | chord is not assigned by default |
| Door | `cmd.door` | `DR` | `DR` | default | |
| Window | `cmd.window` | `WN` | `WN` | default | |
| Column | `cmd.column` | `CL` | `CL` | default | on the Structure tab, `CL` → Structural Column |

## Views

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| 3D View | `ui.view.3d` | `F3`, `3D` | `3D` | user | |
| Floor Plan | `ui.view.plan` | `F2` | — | user | |
| Zoom Fit | `ui.view.zoom_fit` | `Ctrl+0`, `ZF`, `ZE`, `ZA` | `ZF`, `ZE`, `ZA` | user | |
| Properties | `ui.palette.properties` | `Ctrl+1`, `PP`, `VP` | `PP`, `VP` | user | |

## Commands Without Keys

| Command | ID | Category | Why no key |
|---|---|---|---|
| Reference Tower | `cmd.tower` | Other | requires parameters, available via API/CLI |
| Import CAD | `cmd.import_cad` | File | requires path, opened from menu |
| Schedule | `cmd.schedule` | Annotations | requires parameters, opened from dialog |

## Reserved and System Combinations

| Combination | Status | Reason |
|---|---|---|
| `Esc` | reserved by app | cancel/close |
| `Enter` | reserved by app | finish input |
| `Tab` | reserved by app | cyclic selection/focus |
| `Alt+Tab` | intercepted by OS | window switching |
| `Cmd+Tab` | intercepted by OS (macOS) | app switching |

## Conflicts

No conflicts. If `Ctrl+D` were assigned to both "Move" and "Copy", there would be a row here with both actions and a suggestion to replace the owner.
````
