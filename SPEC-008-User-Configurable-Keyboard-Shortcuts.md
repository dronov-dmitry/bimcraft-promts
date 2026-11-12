# SPEC-008. Custom keyboard shortcuts: assigning own combinations to every command, several combinations per command, layout independence, import and export

Status: **implemented**
Date: 2026-11-08
Last modified: 2026-11-12
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Related documents: `ТЗ-007-ленточный-фундамент-ручки-стыки-приклеивание.md` (document layout), `ТЗ-009-освобождение-ctrl-d-и-f8.md` (clarification on `Ctrl+D`/Home and `F8`/`F11`), `BIMCraft/docs/areas/desktop.md` (the "Keyboard Shortcuts" section), `BIMCraft/README.md` (the keyboard shortcuts section)
Task type: user interface customization (app-level settings), keyboard input, settings dialog, file format, migration of existing settings.

> **Relevance.** The decisions on `Ctrl+D`, Home and `F8`/`F11` were refined in `ТЗ-009-освобождение-ctrl-d-и-f8.md` (SPEC-009), and where clauses conflict, SPEC-009 takes precedence:
> - Home is `Ctrl+U`, not `Ctrl+D` and not `Ctrl+Shift+H`;
> - `Ctrl+D` in the BIMCraft default profile is **free** (Move on it — only in the Archicad profile);
> - "Alignment guides" — `F8` only, without `F11`.
> Examples and acceptance criteria of SPEC-008 that state otherwise should be read with this correction.
>
> **Change 2026-11-12 (takes precedence over SPEC-009 regarding Home).** Home is back on `Ctrl+D`:
> - `ui.view.home` in the BIMCraft profile again has the default binding `Ctrl+D`;
> - `Ctrl+U` is removed from Home and stays free;
> - in the Archicad-like profile `Ctrl+D` may still be taken by Move (the profile replaces Home with that binding);
> - in text fields `Ctrl+D` belongs to the field; in the note editor `Ctrl+U` remains underline.
> Clauses of SPEC-008 and SPEC-009 where Home is listed on `Ctrl+U` are to be considered outdated. See Appendix H for details.
>
> **Change 2026-11-12 (sequences).** Besides two-key sequences (`WA`, `MV`), a three-key one (`BMM`) is also accepted. Input: the first two keys wait for the third only if a binding starting with them exists; otherwise the two-key binding fires.

---

## 0. General working rules

- Commits are atomic, one item per commit; messages follow the branch style (an imperative title, a body that explains *why*, not *what*).
- The language of code, identifiers, code comments and commit messages is English. Documentation in `docs/` is Russian; this document is the English translation of `ТЗ-008-горячие-клавиши.md` — keep both files in sync when either changes.
- All UI strings go only through `t()` / `tf()` / `msg!()` from `crates/ui-egui/src/lang.rs` and `bimcraft-i18n`; add new strings to `translations/ui/bimcraft-ui.pot` and to the translations.
- Settings are application settings, not project settings: they live in `settings.json`, survive New/Open and never get into `.bimcraft`.
- Do not break the existing two-letter Revit-style commands (`WA`, `DR`, `WN`, `RM`, `CL`, `DI`, `TG`, `MV`, `CO`, …): they keep working and become one kind of binding.
- Cover every behavior change with tests; for UI — `egui_kittest`, for the parser/model — ordinary unit tests.
- Do not use `subagent` or parallel agents (user rule); the work is done in the current session.

---

## 1. Context and task

### 1.1. What the user wants

Today BIMCraft commands are typed in the Revit style: two Latin letters or digits (`WA` — wall, `DR` — door, `MV` — move). The user needs:

1. **To assign their own combinations to every command in the program** — as in Revit, through the "Keyboard Shortcuts" dialog, but with real modifiers: `Ctrl`, `Shift`, `Alt`, `Cmd`.
2. **Several combinations per command.** For example, "Move" can be `Ctrl+D`, `Ctrl+Shift+M`, and the old `MV` at the same time.
3. **Combinations must work on different layouts.** `Ctrl+D` on the English layout and `Ctrl+В` on the Russian (ЙЦУКЕН) layout are the same physical key, and both must perform the same action.
4. **To export all combinations to a text file and load them back.** The file must be human-readable, suitable for editing in a text editor, and must allow settings to be moved between machines.
5. **A starting set like Archicad's:** at minimum `Ctrl+D` — "Move" (currently `Ctrl+D` opens the start page/Home, this has to be reassigned).
6. **A simple way to find out which combinations exist at all.** Today BIMCraft has hard-coded fixed combinations but no single place to review them: some are visible in the dialog, some in tooltips, some only in code/README. An export of **all active and all existing (default) combinations** into a human-readable Markdown file is needed — as a reference/cheat sheet that can be opened, printed or handed to a colleague.

### 1.2. Statement of the task

The current model "one two-letter combination → one action" must be replaced with the model "action ↔ many bindings", where a binding is either:

- a **chord**: modifiers + one key, e.g. `Ctrl+D`, `Ctrl+Shift+D`, `Alt+F3`;
- a **sequence**: two keys without modifiers, e.g. `WA`, `MV` (Revit-style compatibility).

At the same time:

- an action is addressed by a stable identifier, not by a localized label;
- a key is determined by **physical position** (`physical_key`), not by the character produced in the current layout;
- the same chord cannot be assigned to two actions;
- an action may have 0, 1 or many bindings;
- bindings are stored in `settings.json`, exported and imported in a text format;
- old settings and old two-letter combinations keep working.

### 1.3. User scenarios

- As an Archicad user, I press `Ctrl+D` and get the "Move" tool, not the start page.
- As a user with a Russian layout, I press `Ctrl+В` and get the same "Move" as `Ctrl+D` on the English layout.
- As a user, I assign both `Delete` and `Ctrl+Shift+Delete` to "Delete" and see both combinations in the tooltip.
- As a user, I export the settings to `my-keys.txt`, edit a couple of lines in a text editor and load them back; all valid lines are applied, invalid ones are shown with line numbers.
- As a user, I see the assigned combinations in ribbon tooltips, in menus and in Find (Ctrl+K).
- As a user, I open the "Shortcut reference" and/or export it to `.md` to see all commands, their active and default combinations, including the ones I did not assign, and print the cheat sheet.
- As a user, I do not want `Ctrl+C` / `Ctrl+V` to be intercepted while typing text in a field.
- As a user, I want to reset one command, a category or everything to the default set.

---

## 2. What exists now (per the code)

This section records the current state so that the implementation is targeted rather than "rewrite everything".

### 2.1. Action catalog and the current binding model

- `crates/ui-egui/src/shortcuts.rs`:
  - `ACTIONS: &[&str]` — about 110 action names that can be assigned (walls, doors, views, hiding, snaps, etc.). This is **not** the complete core command catalog: some core commands are missing, and some UI actions are not core commands.
  - `is_key(k)` — a key is considered valid only if `k.len() == 2` and both characters are ASCII alphanumeric. That is, `Ctrl+D` is currently impossible.
  - `resolve_action(desc)` — looks up an action by its full English or localized name.
  - `parse_shortcuts_file(content)` — reads:
    - a JSON object `{"WA": "Wall"}`;
    - Revit lines `<Shortcut CommandName="Wall" Shortcuts="WA#W1"/>`;
    - lines `WA - Wall`, `WA = Wall`, `WA: Wall`.
  - `find_conflict(key, action, app)` — checks whether a two-letter key is taken by another action.
  - `add_shortcuts(parsed)` — adds "key → action" pairs, skips conflicts and invalid keys.
  - `load_shortcuts_from_file(path)` — reads a file up to 64 KiB, adds valid pairs, shows a message.
  - `execute_shortcut_action(action)` — a large `match` on the action name: sets `Tool`, opens a dialog, calls `self.run(...)`.
  - `shortcuts_window(ctx)` — a modal dialog: search, Load File, Add Shortcut, Restore Defaults, a "action → keys" table, Edit/Save/Cancel.
- `crates/ui-egui/src/lib.rs`:
  - `SHORTCUTS: [(&str, &str); 54]` — built-in two-letter combinations (`WA`, `DR`, `WN`, `MM`, `DM`, `AL`, `OF`, `DW`, `ZF`, `TL`, `CS`, `PP`, `NP`, `SA`, `VV`, `EH`, `HH`, `KS`, …).
  - `shortcuts()` — merges `SHORTCUTS`, MEP fitting letters and core command aliases.
  - `BimApp::custom_shortcuts: BTreeMap<String, String>` — user bindings "two-letter key → English action name".
  - `BimApp::typed: String` — the last typed characters (at most two) for two-letter combinations.
  - `BimApp::shortcut()` — first tries `custom_shortcuts`, then the built-in `match` on `typed`.
  - `BimApp::keys()` (around lines 1780–2050) — keyboard handling.
- `crates/ui-egui/src/settings.rs`:
  - `Settings.shortcuts: BTreeMap<String, String>` — the same two-letter bindings in `settings.json`.
  - `MAX_SHORTCUTS: usize = 256`.
  - `shortcuts_from(v, base)` — reads only keys of two ASCII characters and names from `ACTIONS`.
- `crates/core/src/engine.rs`:
  - `Command { id, label, aliases, params, run, undoable, menu }` — a core command.
  - `commands()` — the whole catalog, currently **170 commands**.
  - `find(name)` — a command by `id` or alias.
- `crates/core/src/commands.rs`:
  - `ALL: &[Command]` — 170 commands, of which 35 have `menu`, 57 have aliases.
- `crates/ui-egui/src/find.rs`:
  - Find (Ctrl+K) shows actions from `shortcuts::ACTIONS` and uses only the **first** found combination for the label.
- `docs/areas/desktop.md`, lines 479–488 — the current description of the mechanism.
- `README.md`, line 407 — the current user-facing description.
- **There is no single reference and no export.** The dialog shows a table of actions, but it cannot be exported/printed; built-in combinations are listed only partially in the README and in tooltips, and the user does not see the full list of "what exists". The `.md` export must close exactly this gap.

### 2.2. Hard-coded combinations with modifiers

In `keys()` there are currently separate checks, unrelated to the dialog and not configurable:

| Combination | What it does | Where |
|---|---|---|
| `Ctrl+S` | Save | `lib.rs`, around 1826 |
| `Ctrl+Shift+S` | Save As | same |
| `Ctrl+O` | Open | around 1829 |
| `Ctrl+N` | New | around 1832 |
| `Ctrl+K` | Find | around 1835 |
| `Ctrl+,` | Settings | around 1838 |
| `Ctrl+1` | Properties | around 1841 |
| `Ctrl+P` | Export PDF | around 1844 |
| `Ctrl+D` | Home / start page | around 1851 |
| `Ctrl+Z` | Undo (or sketch step) | around 1856 |
| `Ctrl+Shift+Z`, `Ctrl+Y` | Redo | around 1859 |
| `F8`, `F11` | Alignment guides | around 1905 |
| `Delete` | Delete | around 1888 |
| `Space` | Flip / rotate component / dimension mode | around 1890–1900 |

All of them, except the purely contextual `Esc`, `Enter`, `Tab`, `Space`, must move into the action registry and become configurable.

### 2.3. How the keyboard is read now

In `keys()`:

1. If any modal dialog, popup or menu is open, handling stops.
2. Hard-coded `pressed(Modifiers, Key)` checks run.
3. If `ctx.egui_wants_keyboard_input()`, handling stops.
4. `Event::Text` is collected — **characters of the current layout**, not physical keys.
5. Characters are appended to `typed`; at length 2 `shortcut()` is called.

Consequences:

- on the Russian layout `WA` produces the text `ЦФ`, and the built-in combination does not fire;
- `Ctrl+D` cannot be stored: `is_key` accepts only two ASCII characters;
- order and modifiers are checked by hand, outside the dialog;
- two combinations cannot be assigned to one action: `custom_shortcuts` is a `Map<key, action>`, not a `Map<action, Vec<key>>`;
- when editing in the dialog, the action's old user keys are deleted (`retain(|_, v| *v != action)`), i.e. exactly one user key is supported;
- there is no export at all; import only adds and skips conflicts.

### 2.4. Why `Ctrl+D` conflicts now

`Ctrl+D` is hard-wired to `toggle_home()` (the start page). The user asks for `Ctrl+D` = "Move", as in Archicad. Therefore:

- `Ctrl+D` by default must perform `Move`;
- Home must move to another combination (suggested `Ctrl+Shift+H`) or be left without a combination;
- migration of old settings will not suffer: old user keys were only two-letter ones, so `Ctrl+D` could not be among them.

---

## 3. Terms

| Term | Meaning |
|---|---|
| **Action** | Any user command that can be assigned keys: a tool, a core command, opening a dialog, toggling a palette, a view. Has a stable `id`. |
| **Command** | An entry from `bimcraft_core::engine::commands()` with an `id` field; a special case of an action. |
| **Binding** | One keyboard combination assigned to an action. |
| **Chord** | Modifiers + one key: `Ctrl+D`, `Ctrl+Shift+D`, `Alt+F3`, `F2`. |
| **Sequence** | Two keys without modifiers: `WA`, `MV`. Revit-style compatibility. |
| **Physical key** | A position on the keyboard (`egui::Key` from `physical_key`) independent of the layout. `Ctrl+D` and `Ctrl+В` are one physical key `KeyD`. |
| **Logical key** | A key taking the layout into account; for non-Latin layouts egui falls back to the physical key. |
| **Layout** | The active OS layout (US, RU/ЙЦУКЕН, UA, DE/QWERTZ, FR/AZERTY…). |
| **Profile** | A whole set of bindings: a default one (BIMCraft/Revit/Archicad) or a user one. |
| **Conflict** | The same chord/sequence assigned to two different actions. |
| **Reserved key** | A key the application does not give to bindings (Esc, Enter, Tab, Backspace, arrows, Space) or one intercepted by the OS (Alt+Tab, Alt+F4, Cmd+Tab…). |
| **Canonical form** | A single notation of a binding in the file and in memory: the Latin name of the physical key, modifiers in the order `Ctrl+Shift+Alt+Meta+Key`. |

---

## 4. Functional requirements

### 4.1. Completeness of the action catalog

1. **All user commands must be in the registry.** The registry is the single source of truth for the dialog, Find, tooltips and keyboard handling. It includes:
   - all 170 commands from `engine::commands()`;
   - all actions from the current `shortcuts::ACTIONS`;
   - UI actions that are not core commands: Save As, Export PDF, Settings, Home, Properties, Project Browser, 3D View, Floor Plan, Section Box, Thin Lines, Dark Mode, Close Inactive, Load Family, Favorites, Materials, Project Parameters, Snap overrides, Alignment Guides, Zoom to Fit, Hide/Unhide, Temporary Hide/Isolate, Visual Style, etc.
2. **No command may be lost.** Even if a command cannot be assigned (it requires mandatory parameters and has no dialog), it is shown in the dialog as a greyed-out row with the reason. This requirement is checked by a test: every `Command.id` from `engine::commands()` either has an entry in the registry or is explicitly listed in the exclusion list with a reason.
3. **Stable identifier.** For core commands — `cmd.<id>` (e.g. `cmd.wall`, `cmd.move`). For UI actions — `ui.<area>.<action>` (e.g. `ui.file.save_as`, `ui.palette.properties`, `ui.view.3d`). For legacy actions that are not in the core — `legacy.<English name>`. The ID does not change with localization and does not depend on the label.
   If a UI action is a thin wrapper over a core command (`Open`, `New`, `Save`, `Undo`, `Redo`), use `cmd.<id>` instead of creating a second `ui.*`. `ui.*` is only for what is not in the core or for distinct behavior: Save As, Export PDF, Settings, Home, palettes, views.
4. **Action classification:**
   - `Tool` — selects the active tool (`Wall`, `Door`, `Move`, `Rotate`, …);
   - `Dialog` — opens an existing dialog (`Settings`, `Visibility/Graphics`, `Filters`, `Phasing`, …);
   - `Immediate` — runs at once without mandatory parameters (`Zoom to Fit`, `Thin Lines`, `Undo`, `Redo`, `Save`, …);
   - `Input` — requires mandatory data (a file path, points, parameters); assignable only if the command has an existing UI parameter collector, otherwise marked "requires parameters";
   - `Internal` — not a user command (autosave, recover, cross-check export, etc.); shown but not assignable.
5. **One ID — one executor.** A test checks that IDs are unique and that every assignable action has a working executor (`execute_action` returns `true`).
6. **Duplicate names are allowed, duplicate IDs are not.** For example, `column` and `structural_column` share the alias `CL`; they are different actions with different IDs, and the contextual choice of `CL` remains a separate rule.

### 4.2. Bindings: several per action

1. An action may have **0, 1 or many** bindings. The upper limit is 32 bindings per action (protection against junk); the overall limit is 4096 bindings.
2. The bindings of one action are shown as a list in the order they were added: defaults first, then user ones.
3. The user can:
   - add a binding;
   - remove one binding;
   - replace a binding;
   - remove all bindings of an action (the action stays without keys);
   - reset the action to its default bindings;
   - reset a category/all actions to the selected profile.
4. Default bindings can be overridden and deleted. If the user deleted a default binding, it does not come back by itself.
5. Priority: a user binding and a default binding are equal when searching; a conflict between them is not allowed (see 4.3).
6. The same binding **cannot** belong to two actions. On an attempted assignment a conflict is shown: "`Ctrl+D` is already assigned to 'Move'. Replace?" Options: "Replace" (remove it from the previous action) or "Cancel".
7. If a conflict is found during file import, the line is skipped and the report states the line number, the action and the conflicting action. The first valid line in the file wins.
8. If an action is unavailable in the current context (nothing selected, another view open), the key is still "consumed" and shows the status "Unavailable: <reason>"; it must not unexpectedly run another command.

### 4.3. Chord model and exact matching

1. A chord consists of:
   - the main modifier `Ctrl` (on macOS — `Cmd`; in the file `Ctrl` means "primary", see 4.4);
   - `Shift`;
   - `Alt` / `Option`;
   - `Meta` / `Cmd` — the literal Command on macOS (`Modifiers::mac_cmd`); on Windows/Linux the Win/Super key is usually intercepted by the OS and does not reach egui, so `Meta` does not work there (the UI warns);
   - one key.
2. **Modifier matching is exact.** `Ctrl+D` and `Ctrl+Shift+D` are different bindings and can coexist. `egui::InputState::consume_key` cannot be used as is: it applies `Modifiers::matches_logically`, which ignores extra `Shift`/`Alt` and may run `Ctrl+D` while `Ctrl+Shift+D` is pressed. Compare `Modifiers::matches_exact` (taking `command`/`mac_cmd` into account) and consume only the exact event.
3. **Modifier order in canonical form:** `Ctrl`, `Shift`, `Alt`, `Meta`, then the key. Example: `Ctrl+Shift+Alt+Meta+D`. The displayed order may be platform-specific (`Cmd+Shift+D` on macOS).
4. **A modifier key by itself** (Ctrl, Shift, Alt, Cmd) is not a binding.
5. **Key repeat is ignored:** holding `Ctrl+D` must not launch "Move" repeatedly. Use only events with `pressed: true, repeat: false`.
6. **CapsLock and NumLock** do not affect the physical key. Numpad keys are considered separate if egui distinguishes them; otherwise the generic `Num0..Num9` are used.
7. **A single key without modifiers** is allowed only for keys that do not participate in sequences: `F1`–`F24`, `Delete`, `Insert`, `Home`, `End`, `PageUp`, `PageDown`, `PrintScreen`, etc. Single **letters and digits** without modifiers are forbidden: otherwise `D` would fire immediately and make the sequences `DR`, `DW`, `DI` impossible. The UI explains the reason for the refusal.
8. **AltGr.** On Windows AltGr arrives as `Ctrl+Alt`; chords with `Ctrl+Alt` are allowed, but the UI warns that on some layouts AltGr may be intercepted by the OS/driver. If the same frame contains an `Event::Text` generated by AltGr and the focus is in a text field — the text field wins.

### 4.4. Layout independence

This is the key requirement.

1. **The source of truth is the physical key.** In `egui 0.36` the `Event::Key` event contains:
   - `key: Key` — the logical key; for non-Latin layouts egui falls back to the physical key;
   - `physical_key: Option<Key>` — the physical position, independent of the layout;
   - `modifiers: Modifiers`;
   - `repeat: bool`.
   For bindings use `physical_key.unwrap_or(key)`. This makes `Ctrl+D` == `Ctrl+В` on ЙЦУКЕН, `Ctrl+D` == `Ctrl+D` on QWERTZ (the US D position), and gives identical behavior when switching layouts while running.
2. **Never use `Event::Text` for chords and sequences.** `Event::Text` depends on the layout and produces `ЦФ` instead of `WA`, `в` instead of `d`. Text is needed only for input into text fields and for typing lengths/numbers in tools.
3. **Canonical storage uses Latin names of physical keys.** In memory and on export, `Ctrl+D` is `physical_key = Key::D`, regardless of what is printed on the key. On ЙЦУКЕН the same key is labeled `В`.
4. **Import accepts localized letters.** `Ctrl+В` in a file is recognized as `Ctrl+D` via the ЙЦУКЕН table (Appendix C). `Ctrl+С` (Cyrillic `С`) → `Ctrl+C`, `Ctrl+М` → `Ctrl+V`, `Ctrl+А` → `Ctrl+F`, `Ctrl+Ф` → `Ctrl+A`, etc. Similar tables can be added for Ukrainian and other layouts; the minimum per this spec is Russian.
5. **Display.** The UI shows the canonical form `Ctrl+D`; if the UI language is Russian (or showing localized labels is enabled), show `Ctrl+D (Ctrl+В)` in the dialog and tooltip. In compact places (menus, Find) — the canonical form only.
6. **Switching the layout while running** does not change bindings and does not require a restart.
7. **Limitation for non-Latin and non-ЙЦУКЕН layouts.** The physical position is the US keyboard position. On AZERTY/QWERTZ a user who thinks in terms of the letter printed on their key may see a mismatch with the label. This is a deliberate decision: the user's requirement is "the same button on different layouts". The tooltip should say "physical key D (on ЙЦУКЕН — В)".
8. **A test is mandatory:** an event with `physical_key = Some(Key::D), modifiers = Ctrl`, regardless of `text = "в"` / `text = "d"` / no text, must perform the same action.

### 4.5. Sequences (Revit-style compatibility)

1. A sequence is two keys without `Ctrl`/`Alt`/`Meta` (Shift is allowed but does not change the key). Example: `W`, then `A`.
2. Sequences are read from physical keys, not from `Event::Text`. This fixes `WA` on the Russian layout.
3. Sequence buffer:
   - holds at most 2 keys;
   - is cleared by `Esc`, on a mouse click, a tool change, opening a modal dialog/menu, and after a binding fires;
   - is cleared by an inactivity timeout (2 seconds recommended) so that an old letter does not "stick" to a new command;
   - does not intercept digits/period when the tool is waiting for a length (`typed_length`).
4. A sequence can be assigned to any action on equal terms with a chord. In the dialog it is displayed as `WA` (without `+`).
5. One sequence cannot be assigned to two actions; a conflict is handled the same way as for a chord.
6. The old two-letter bindings from `SHORTCUTS`, command aliases and `custom_shortcuts` migrate to sequences without behavior loss.

### 4.6. Context, priority and key consumption

Keyboard handling order (top to bottom, the first level that fires wins):

1. **OS/window.** Combinations intercepted by the OS (Alt+Tab, Alt+F4, Cmd+Tab, Cmd+Space, Win+L, etc.) do not reach the application; the application cannot reassign them.
2. **Modal dialog.** While a modal dialog is open, only its local keys (`Esc`, `Enter`, navigation) and the capture mode in the keyboard shortcuts dialog work. Global bindings do not fire.
3. **Text input.** If `ctx.egui_wants_keyboard_input()` is true (input field, search, name editor, numeric field), all configurable bindings are disabled. The exception is "global" actions that do not steal text:
   - `Save`, `Save As`, `Open`, `New`, `Find`, `Settings` — allowed;
   - `Undo`/`Redo` in a text field stay with the field (undoing typed text), not with the model;
   - `Ctrl+C`/`Ctrl+V`/`Ctrl+X`/`Ctrl+A` always belong to the text field.
4. **Open menu/popup.** Navigation and `Esc` belong to the menu. After the menu closes, bindings work again.
5. **Tool input.** If the active tool is waiting for a number/point/length, digits, `.` and `,` go to `typed_length`; `Enter` completes the input. Chords with modifiers may still work if they do not conflict with the input.
6. **Chords.** Exact match by `physical_key` + `Modifiers::matches_exact`.
7. **Sequences.** If no chord fired and there are no modifiers (except Shift), the key goes to the sequence buffer.
8. **Contextual tool keys.** `Esc`, `Enter`, `Tab`, `Backspace`, arrows, `Space` — reserved (see 4.10) and always handled by the tool; an action cannot be assigned to them.
9. **Sketch mode.** While a sketch is active, `Ctrl+Z` undoes the last sketch line rather than a model step (as now in `keys()`); `Ctrl+Shift+Z`/`Ctrl+Y` in a sketch must not undo the model. This contextual rule takes precedence over the global Undo/Redo binding.

Additionally:

- One key press is consumed exactly once. After a binding fires, the event is removed from `InputState.events` (or marked consumed) so that other handlers do not run the same command.
- A chord clears the sequence buffer.
- If a binding is assigned but the action is unavailable, the key is consumed and the status is shown; there is no fallback to a sequence/another command.
- `Ctrl+Shift+...` must not fire as `Ctrl+...`: the check order must not mask a more specific chord. It is better to look up an exact match in a hash table than to iterate from "weak" to "strong".

### 4.7. The "Keyboard Shortcuts" dialog

The existing `shortcuts_window` is replaced/extended. Opening: Manage > Shortcuts, the Shortcuts button, Find, by default `KS` (sequence) and/or `Ctrl+Shift+K` (if it is decided to keep a chord; it may be left unassigned).

#### 4.7.1. Overall structure

- A modal window, about 720–780 px wide, up to 80% of the window height.
- Top panel:
  - a search field (by localized name, English name, ID, binding);
  - a filter: "All", "Assigned", "Without keys", "Changed", "Conflicts";
  - a category filter (Architecture, Structure, Views, Manage, File, Palettes, Snaps, MEP, Annotations, All);
  - a profile choice: "BIMCraft", "Revit-like", "Archicad-like", "Empty";
  - buttons "Import…", "Export…", "Reference (Markdown)…", "Reset…".
- The "Reference (Markdown)…" button sits next to export and immediately opens saving of the Markdown cheat sheet (see 4.8.4); this is the main "simple way to get acquainted" with all combinations.
- Table:
  - the "Command" column — the localized name + the ID in a small font;
  - the "Category" column;
  - the "Combinations" column — chips/buttons, each with `×`; a `+` button to add;
  - the "Source" column — "default", "custom", "changed";
  - the "Actions" column — "Reset" / "Remove all".
- Bottom panel: a counter "Assigned X of Y", "Conflicts: N", buttons "Cancel", "Apply".
- Changes in the dialog **accumulate** and are applied on "Apply"; "Cancel"/`Esc` rolls back. After "Apply" the settings are saved to `settings.json` immediately (through the common `keep_settings` mechanism).
- If there are unsaved changes, closing the window/`Esc` asks for confirmation.

#### 4.7.2. Adding and editing a binding

- The `+` button next to an action opens a capture field: "Press a combination…".
- In the capture field:
  - pressing `Ctrl+D` shows `Ctrl+D`; on the Russian layout additionally `(Ctrl+В)`;
  - pressing the key labeled `А` on ЙЦУКЕН is recorded as `Ctrl+F` and shows `(Ctrl+А)`: this is the physical key `F`, not the Latin `A`;
  - pressing two keys without modifiers (`W`, `A`) shows `WA` and creates a sequence;
  - `Esc` cancels the capture;
  - `Backspace` clears the capture;
  - `Tab`/`Enter` are not considered a binding;
  - pressing only a modifier is ignored with a hint;
  - a conflict is shown immediately, before saving, with the name of the owning action and a "Replace" button;
  - a reserved key shows the reason why it cannot be assigned.
- After a successful addition the binding appears as a chip; the next one can be added.
- Editing an existing chip: a click opens the same capture with the current binding; "Save" replaces, "Cancel" keeps.
- Deleting a chip: `×` or the context menu "Delete binding".

#### 4.7.3. Search, filters and sorting

- Search is case-insensitive; it looks through:
  - the localized name;
  - the English name;
  - the ID (`cmd.move`, `ui.file.save_as`);
  - the binding string (`Ctrl+D`, `WA`, `В`).
- An empty search shows everything.
- Default sorting: category, then localized name. Sorting by name/ID/number of bindings is also possible.
- Rows without bindings are visually dimmed but not hidden unless the filter is "Without keys".
- Conflicting rows are highlighted with the error color; clicking a conflict jumps to the second participant.

#### 4.7.4. Reset

- "Reset action" — restore the default bindings of the selected action.
- "Reset category" — restore the default bindings of all actions in the category.
- "Reset all" — restore the default profile (BIMCraft by default), with confirmation.
- "Clear all" — leave all actions without keys (for those who want to start from scratch), with confirmation.
- Reset does not touch other application settings.

#### 4.7.5. Applying a profile

- Choosing a profile shows a preview: how many bindings will be added/changed/removed.
- Applying a profile replaces all current bindings with the selected set (after confirmation).
- Profiles: `BIMCraft` (default), `Revit-like`, `Archicad-like`, `Empty`.
- The user profile is saved as "Custom" after any change.

### 4.8. Import and export

#### 4.8.1. Export

1. The "Export…" button opens the system save dialog.
2. Formats:
   - the main one — text `.txt` / `.bks` (BIMCraft Keyboard Shortcuts), UTF-8, suitable for re-import;
   - additionally — `.json` (the same model, convenient for scripts, suitable for re-import);
   - **`.md` — a Markdown reference/cheat sheet** for reading, printing and handing to a colleague (see 4.8.4); this is **not** a backup format and not an import format;
   - optionally — Revit XML (sequences only; chords cannot be represented, warn on export).
3. Export modes:
   - **the whole profile** (all actions, including default and empty ones) — by default;
   - **only user changes** (actions whose bindings differ from the default profile).
4. Export is deterministic: the same state → the same file (no random order, no timestamp in the content; the application version — in a comment, if needed).
5. Writing is atomic: first a temporary file, then `rename`. A write error must not corrupt the existing file.
6. Default name: `bimcraft-shortcuts.txt`; on repeated export — the last chosen path (stored in settings, optionally).

#### 4.8.2. Import

1. The "Import…" button opens the system file picker.
2. Supported formats:
   - the new text format `.txt` / `.bks` (section 5);
   - JSON (`{"version": 2, "bindings": [...]}` or the old `{"WA": "Wall"}`);
   - Revit XML lines `<Shortcut CommandName="…" Shortcuts="WA#W1"/>`;
   - legacy lines `WA - Wall`, `WA = Wall`, `WA: Wall`.
   - **Markdown `.md` is not supported**: it is a reference for reading (4.8.4). If `.md` is chosen, show a message and do not change settings.
3. Import modes:
   - **Replace all** — the current bindings are replaced by the file contents; actions absent from the file are left without keys. This is the default mode for a file with `version = 2`.
   - **Add/merge** — valid bindings from the file are added to the current ones; conflicts are skipped with a report.
4. Before applying, a preview is shown:
   - how many bindings will be added, replaced, skipped;
   - a list of errors with line numbers and reasons;
   - a list of conflicts;
   - a warning if the file is empty or contains no valid bindings.
5. Applying is atomic: first a full parse and validation, then replacement in memory, then saving `settings.json`. If saving failed — roll back in memory and show an error.
6. Before replacement, save a backup of the current profile next to the settings (`shortcuts.bak`) or in the settings folder with a date.
7. Limits: a file of at most 256 KiB, at most 10,000 lines, at most 4096 bindings. Exceeding — refusal with a message, without partial application.
8. Errors do not "break" the import as a whole: valid lines are applied, invalid ones are listed. The exception is a file version/structure error: then nothing is applied.
9. Import does not execute any code from the file: data only.

#### 4.8.3. The "export → import" check

- Export → import in "Replace all" mode must produce a semantically identical set of bindings (the order within an action is preserved; the order of actions may be canonicalized).
- Re-exporting after import produces the same text (determinism).
- `Ctrl+В` on import is canonicalized to `Ctrl+D`; on export `Ctrl+D` is written. This is not a loss: both records mean one physical key.

#### 4.8.4. Exporting the reference to Markdown

This is a separate, **human-readable** export: it does not replace `.bks`/`.json` and is not intended for re-import.

1. **Purpose.** To give the user a simple way to see all combinations that exist in the program: active, default (built-in), custom, as well as commands without keys. The resulting `.md` can be opened in any Markdown viewer, printed, put into a repository or handed to a colleague.
2. **Entry point.** The "Reference (Markdown)…" button in the keyboard shortcuts dialog (next to "Export…"), as well as the "Markdown (reference)" item in the general export dialog.
3. **What goes into the reference:**
   - **all registry actions** — all 170 core commands and all UI actions, including those with no binding at all;
   - **active bindings** (what will actually fire now);
   - **default bindings** (the built-in values, even if the user changed or deleted them) — in a separate column or section;
   - the localized action name, the English name and the stable ID;
   - the category and/or menu path (`Architecture > Wall`, `Manage > Link Model`);
   - the source: "default", "custom", "changed", "deleted";
   - for actions without keys — an explicit "not assigned" mark;
   - conflicts (if any) and reserved/system combinations;
   - the profile, the format/application version, the generation date and the platform;
   - for Russian — localized hints of the form `Ctrl+D (Ctrl+В)`.
4. **File structure:**
   - the heading `# BIMCraft — keyboard shortcuts`;
   - a brief "how to read" instruction and a table of contents;
   - sections by category (File, Edit, Architecture, Structure, Views, Manage, Palettes, Snaps, MEP, Annotations, Other);
   - within a section — a table:
     `| Command | ID | Active | Default | Source | Note |`;
   - a separate section "Commands without keys" — so that they are visible and can be assigned;
   - a separate section "Reserved and system combinations";
   - a separate section "Conflicts" — only if there are any, naming both actions.
5. **Export options:**
   - include commands without keys — by default **yes**;
   - include default bindings — by default **yes**;
   - only active bindings / only user changes — optionally;
   - include bindings of other default profiles (Revit-like, Archicad-like, Empty) — optionally, as a separate section "Other profiles";
   - the language of labels — the UI language by default, English can be chosen;
   - `Ctrl+D (Ctrl+В)` hints — yes by default for the Russian UI.
6. **Determinism.** The same state → the same `.md` (sorting by category, then by localized name/ID; no random order). The generation date may be in the header but must not affect profile comparison; for diffs, provide an option "without date".
7. **Valid Markdown.** Tables with `|` escaped in names, no raw HTML, proper headings and anchors. The file must open correctly in GitHub, VS Code, Obsidian and an ordinary viewer.
8. **Writing** — UTF-8, atomic (temporary file + `rename`), default name `bimcraft-shortcuts.md`.
9. **Importing `.md` is not supported.** If the user chooses `.md` in the import dialog, show the message: "This is a reference for reading. To import, use `.bks`, `.txt` or `.json`". This protects against an attempt to "load the reference" and lose the settings.
10. **Single source.** The reference is generated from the same action registry and the same `Shortcuts` state as the dialog, so it cannot diverge from it. A test checks that every registry action got into the `.md`.
11. **Not required, but a useful bonus:** a "Preview" button in the dialog showing the same Markdown in a built-in scrollable window (without saving). This covers the "just take a look" scenario without creating a file.

### 4.9. Settings storage and migration

1. `Settings.shortcuts` changes its type from `BTreeMap<String, String>` to a structure:
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
2. The settings file stores the **full effective set** of bindings (not just the differences from the defaults). This makes behavior predictable when the program is updated: default bindings do not "creep back".
3. `shortcuts.version`:
   - `2` — the new format;
   - missing, or an object without `version` — the old format `{"WA": "Wall"}`.
4. Migrating the old format:
   - each `key → action` pair turns into a sequence (`WA` → `WA`);
   - the action name is resolved to an ID via the registry (English/localized name);
   - unknown actions are skipped and logged (without breaking the load);
   - after the first successful load the settings are rewritten in version 2.
5. Increase `MAX_SHORTCUTS` to 4096 bindings; `MAX_ACTIONS` — 512. Increase `settings.rs::MAX_FILE` from 64 KiB to 1 MiB (or 512 KiB): 4096 bindings in JSON do not fit into 64 KiB, otherwise the settings will be silently unreadable.
6. A corrupt/oversized `settings.json` must not wipe the settings: keep the last successful copy in memory, show a status, and do not overwrite the file until the user acts.
7. New/Open do not touch bindings: these are application settings.
8. `use_settings` loads bindings into `BimApp::shortcuts`; `settings_now`/`keep_settings` save them.
9. Backward compatibility: if a new version of the program reads an old `settings.json`, it migrates; if an old version reads the new one, it must not crash (an unknown object is ignored, default bindings remain). This is a desirable but non-blocking requirement.

### 4.10. Reserved keys and OS conflicts

1. **Always reserved by the application** (cannot be assigned):
   - `Esc` — cancel/close;
   - `Enter` — finish input/construction;
   - `Tab` — cyclic selection/focus;
   - `Backspace` — delete the last character/step;
   - arrows — navigation/movement;
   - `Space` — the tool's contextual action (flip, rotate, dimension mode).
2. **Assignable, but with a warning:**
   - `Delete` (by default — the Delete command);
   - `F1`–`F12` (some may be taken by the OS/window);
   - `Ctrl+Alt+…` (AltGr);
   - `Ctrl+Shift+Esc`, `Alt+F4`, `Alt+Tab`, `Ctrl+Esc`;
   - on macOS: `Cmd+Tab`, `Cmd+Space`, `Cmd+Q`, `Cmd+H`, `Cmd+M`, `Cmd+``;
   - on Windows: `Win+L`, `Win+D`, `Win+Tab`, `Ctrl+Alt+Del` (the last one does not arrive at all).
3. UI on assignment:
   - a hard-reserved key is not accepted, and the reason is explained;
   - an OS-conflicting one is accepted, but a yellow warning is shown: "may not work: intercepted by the system".
4. On import such bindings are:
   - for hard-reserved ones — skipped with an error;
   - for OS-conflicting ones — accepted with a warning.
5. `Ctrl+D` is not reserved; it is reassigned to Move.

### 4.11. Displaying bindings in the interface

1. **Ribbon and tool palette tooltips.** For each action with bindings the tooltip has the form:
   `Move (Ctrl+D, Ctrl+Shift+M, MV)`.
   If there are no bindings — only the name and description, without empty parentheses.
2. **Menus.** To the right of a menu item show the first (or all, if they fit) binding via egui's standard `shortcut_text`. Example: `Save   Ctrl+S`.
3. **Find (Ctrl+K).** In the action row show up to two bindings and `+N` if there are more. Search by binding also works.
4. **Context menus.** If a context menu item corresponds to an action, show its binding.
5. **Status bar.** After a binding fires, if the action is unavailable, show the reason. If it is available — show nothing extra.
6. **Dialog.** Binding chips, conflicts, source.
7. **Localization.** Action names — via `t()`. Keys — canonical; for the Russian layout add `(Ctrl+В)` in the dialog/tooltip.
8. **No hard-coded strings** of the form `"Modify (MD)"`: all labels with keys are generated from the registry. Existing strings in `lib.rs` (e.g. `"Modify (MD): select elements"`) are replaced with a helper.

### 4.12. Default profiles and `Ctrl+D`

> **SPEC-009 (takes precedence):** in the BIMCraft default profile `Ctrl+D` is **free**, Home is `Ctrl+U`. The clauses below about `Ctrl+D` = Move and Home = `Ctrl+Shift+H` apply only to the Archicad-like profile or to the draft of SPEC-008.

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
2. **Revit-like:** the current `SHORTCUTS` + aliases; `Ctrl+D` may be left as Home or unassigned (the profile is deliberately different from BIMCraft). The user can choose it in the dialog.
3. **Archicad-like:** at minimum `Ctrl+D` = Move, `Ctrl+R` = Rotate, `Ctrl+M` = Mirror Pick Axis, `Ctrl+Shift+M` = Mirror Draw Axis, `Ctrl+Shift+D` = Copy, `Ctrl+Shift+O` = Offset, `Ctrl+T` = Trim, `Ctrl+0` = Zoom to Fit, `F2`/`F3` = Floor Plan/3D View. The rest — as desired; the profile does not have to match Archicad byte for byte.
4. **Empty:** no bindings at all; useful for a clean start.
5. All profiles are data, not code: they can be stored in a built-in resource (`include_str!`) and updated without changing the logic.
6. Conflicts within a default profile are forbidden; a test checks that no chord/sequence occurs twice.
7. On the first launch after an update, show a one-time notification: "`Ctrl+D` is now 'Move'; the start page is `Ctrl+Shift+H`. You can change this in Manage > Shortcuts". Do not show the notification if the user has already opened the new dialog or changed bindings.

### 4.13. Performance and reliability

1. Looking up a binding on a key press is O(1): a hash table `KeyChord → ActionId` and `[Key; 2] → ActionId`.
2. Handling must not allocate on every key press (except rare cases); reuse buffers.
3. `settings.json` is written atomically (already implemented) and no more than once per frame.
4. Importing 4096 bindings must take < 50 ms and not block the UI for more than one frame (a background thread with progress if necessary; for 4096 this is not required).
5. An error in one binding must not break application startup: the registry and settings load into a consistent state.

---

## 5. Text file format

### 5.1. General properties

- Encoding: UTF-8; a BOM is allowed and ignored.
- Line endings: LF or CRLF; the parser normalizes them.
- Extension: `.txt` or `.bks` (the same content).
- Comments: a line starting with `#` after optional spaces. Inline comments are not supported, so that `#` can be used as a key in the future.
- Blank lines are ignored.
- The case of action names and modifiers does not matter; the canonical form is lowercase IDs and `Ctrl+...`.
- Maximum size: 256 KiB; at most 10,000 lines; at most 4096 bindings.

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

- `action` — a stable ID (`cmd.move`, `ui.file.save_as`) or the English/localized action name. The ID takes precedence.
- One line — one binding. Several bindings for an action are given by several lines with the same `action`. A comma-separated list is **not supported**: `Ctrl+,` is the Settings key, and a comma cannot be a separator.
- Legacy lines `KEY - Name` (`WA - Wall`) are read by a separate parser branch only for compatibility; in the new format the action is always to the left of `=`.
- Spaces around `=`, `:` and `+` are ignored.
- If a line contains `+`, it is a chord.
- If there is no `+`:
  - two single-letter names without a space (`WA`) or two tokens separated by a space (`W A`) — a sequence;
  - one key name (`F2`, `Delete`) — a chord without modifiers; single letters and digits are forbidden (see 4.3.7).
- The `+` key is written as `Plus`; `Ctrl+Plus` is a chord with the "plus" key. The notation `Ctrl++` is not recommended and may be rejected as ambiguous.
- `Ctrl` in the file means **primary**: `Ctrl` on Windows/Linux, `Cmd` on macOS. `Cmd`/`Meta` — the literal Command on macOS; on Windows/Linux Win/Super is usually unavailable to the application. For portable files use `Ctrl`; import `Cmd`/`Meta` on Windows/Linux with a warning and treat them as primary so that the file does not lose bindings.
- `Primary` — a synonym for `Ctrl`.
- `Option` — a synonym for `Alt`; on export on macOS write `Alt` for portability, accept `Option`.

### 5.3. Key names

Canonical names (synonyms from `egui::Key::from_name` are also accepted):

- letters: `A`…`Z`;
- digits: `0`…`9` (the main row), `Num0`…`Num9` (Numpad);
- functions: `F1`…`F24`;
- navigation: `Up`, `Down`, `Left`, `Right`, `ArrowUp`, `ArrowDown`, `ArrowLeft`, `ArrowRight`, `Home`, `End`, `PageUp`, `PageDown`, `Insert`, `Delete`, `Backspace`, `Enter`, `Return`, `Tab`, `Escape`, `Esc`, `Space`;
- punctuation: `Comma`, `Period`, `Slash`, `Semicolon`, `Quote`, `Backquote`, `Minus`, `Equals`, `OpenBracket`, `CloseBracket`, `Backslash`, `Plus`, `Colon`, `Pipe`, `Questionmark`, `Exclamationmark`;
- Numpad: `NumpadAdd`, `NumpadSubtract`, `NumpadMultiply`, `NumpadDivide`, `NumpadEnter`, `NumpadDecimal`;
- other: `PrintScreen`, `Pause`, `Menu`, `BrowserBack` (if egui provides it).

Symbolic abbreviations for punctuation (`Ctrl+,`, `Ctrl+.`, `Ctrl+/`, `Ctrl+;`, `Ctrl+'`, `Ctrl+\``, `Ctrl+-`, `Ctrl+=`, `Ctrl+[`, `Ctrl+]`, `Ctrl+\`) are accepted. On export, for readability, write the symbols themselves (`Ctrl+,`, `Ctrl+.`) rather than `Comma`/`Period`; the exception is the "plus" key: write `Ctrl+Plus`, because `Ctrl++` is ambiguous. This matters for the familiar `Ctrl+,` (Settings).

### 5.4. Localized letters and the layout

- `Ctrl+В` (Cyrillic) is accepted as `Ctrl+D`; the table is in Appendix C.
- `Ctrl+С` → `Ctrl+C`, `Ctrl+М` → `Ctrl+V`, `Ctrl+А` → `Ctrl+F`, `Ctrl+Ф` → `Ctrl+A`, etc. per the table.
- If a character is in the table, it is converted to the physical key; if not, the line is treated as an error with a number.
- Latin characters are always treated as the physical position on the US keyboard, not as "the letter on the current layout".
- `Ctrl+Shift+В` → `Ctrl+Shift+D`.
- `WA` and `WА` (the second `А` Cyrillic) are the same sequence; on export `WA`.

### 5.5. Example file

> **SPEC-009 (takes precedence):** the example below is outdated regarding Home and `Ctrl+D`: in BIMCraft Home is `Ctrl+U`, and `Ctrl+D` is not assigned in the default profile.

```text
# BIMCraft keyboard shortcuts
# version: 2
# Format: <action> = <binding>
# One line — one binding; comments start with #.
version = 2
profile = bimcraft

# --- Move ---
cmd.move = Ctrl+D
cmd.move = Ctrl+Shift+M
cmd.move = MV

# --- Wall ---
cmd.wall = WA
cmd.wall = Ctrl+Shift+E

# --- Door ---
cmd.door = DR
cmd.door = Ctrl+В

# --- Save ---
cmd.save = Ctrl+S
ui.file.save_as = Ctrl+Shift+S

# --- Properties ---
ui.palette.properties = Ctrl+1

# --- Start page (moved off Ctrl+D) ---
ui.view.home = Ctrl+Shift+H
```

A note on `cmd.door = Ctrl+В`: on the Russian layout this is the physical key `D`; on loading the record is canonicalized to `Ctrl+D`.

### 5.6. Errors and warnings

The parser returns:

```rust
struct ParsedProfile {
    version: u32,
    bindings: Vec<(ActionId, Binding)>,
    warnings: Vec<Warning>,   // line, reason, action
    errors: Vec<Error>,       // line, reason
}
```

Examples of errors:

- `unknown action: "Wallz"` (line 12);
- `unknown key: "Ctrl+Ъ"` (line 15);
- `a sequence must consist of two keys: "W"` (line 18);
- `conflict: "Ctrl+D" is already assigned to "cmd.move"` (line 21);
- `reserved key: "Esc"` (line 24);
- `version 3 is not supported` (line 2).

An error in one line does not prevent the other valid lines from being applied (except a version/structure error). In the import preview show the first 50 errors and "N more".

### 5.7. JSON format

```json
{
  "version": 2,
  "profile": "custom",
  "bindings": [
    {"action": "cmd.move", "keys": ["Ctrl+D", "Ctrl+Shift+M", "MV"]},
    {"action": "cmd.wall", "keys": ["WA", "Ctrl+Shift+E"]}
  ]
}
```

- `keys` — an array of strings in the same syntax as the text format.
- The old JSON `{"WA": "Wall"}` is read as legacy.
- JSON is exported with 2-space indentation, keys in a stable order (actions by ID).

### 5.8. Revit XML compatibility

Input lines of the form:

```xml
<Shortcut CommandName="Wall" CommandId="ID_OBJECTS_WALL" Shortcuts="WA#W1" />
```

are read as the sequences `WA` and `W1` for the action found by `CommandName`/`CommandId` (localized/English name). Chords cannot be represented in Revit XML. Export to Revit XML is optional; if it is done, sequences only, with a warning about skipped chords.

---

## 6. Architecture and technical plan

### 6.1. New types

```rust
// crates/ui-egui/src/shortcuts.rs (or a shortcuts/ module as it grows)

use std::sync::Arc;

/// Action ID. UI strings are static; core command IDs (`cmd.<id>`) are built
/// once when the registry is constructed and interned into `Arc<str>`.
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
    pub execute: fn(&mut BimApp, &Context), // Context is needed by Open/dialogs/file actions
}

pub struct Shortcuts {
    by_chord: HashMap<KeyChord, ActionId>,
    by_sequence: HashMap<[Key; 2], ActionId>,
    by_action: BTreeMap<ActionId, Vec<Binding>>,
    dirty: bool,
}
```

Notes:

- `Binding` must be `Hash + Eq` for the hash tables.
- `defaults` can be stored as strings and parsed once at startup (easier to update), but use already parsed `Binding`s on the hot path.
- `ActionDef` is best assembled into a `static`/`OnceLock` registry so that nothing is allocated every frame.
- `enabled` and `execute` are functions, not closures, so that the registry can be `'static`.

### 6.2. Action registry

1. The base list is the static array `UI_ACTIONS` (all UI actions, including the current `ACTIONS`).
2. Core commands are added from `engine::commands()`:
   - ID: `cmd.<command.id>`;
   - label: `command.label`;
   - category: by `command.menu` or an explicit table;
   - kind: by the explicit table `COMMAND_KINDS` (Tool/Dialog/Immediate/Input/Internal);
   - aliases: `command.aliases` (they become default sequences);
   - menu: `command.menu`.
3. Duplicate IDs are forbidden; at build time — `debug_assert!` and a test.
4. Keep `shortcuts::ACTIONS` as a compatible list of names (for old files and Find), but make `registry::actions()` the source of truth.
5. The `resolve_action(desc)` function:
   - first looks up by ID;
   - then by English label;
   - then by localized label (as now);
   - then by legacy names;
   - returns an `ActionId`.
6. The `execute_action(app, ctx, id) -> bool` function replaces `execute_shortcut_action`. Inside — a `match` on the ID or a call to `def.execute(app, ctx)`. For core commands with ready parameters — `app.run(command.id, params)`; for tools — `app.set_tool(...)`; for dialogs — opening the corresponding field. For commands with mandatory parameters (`open`, `insert`, `point_cloud`, `schedule`, `render`, `save` with a path) the executor opens the existing dialog/file picker (`guard(Pending::Open, ctx)` etc.) rather than calling `run(id, {})`; if there is no such UI, the action is marked `Input`/`Internal` and is not assignable.
7. Keep the old `execute_shortcut_action(action: &str)` as a thin wrapper during migration: `resolve_action(action).and_then(execute_action)`. Remove it after all call sites are converted.

### 6.3. Input handling in `lib.rs`

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

- Do not use `consume_key` for chords: it ignores extra `Shift`/`Alt` (`matches_logically`). Use `matches_exact` and remove/mark the specific event.
- Take `physical_key` into account; fall back to `key` (for web/some platforms).
- Ignore `repeat == true` for commands.
- Handle `typed_length` before sequences.
- After a chord fires, do not let the event reach `typed`.
- Leave `Escape`/`Enter`/`Tab`/`Space`/`Backspace`/arrows in the existing contextual handling if they are not assigned (they are reserved).
- Remove the hard-coded `pressed(Modifiers::COMMAND, Key::D)` and the rest from the list in 2.2; their role now belongs to the registry and the default profile.
- `Ctrl+D` no longer calls `toggle_home()`; Home is `Ctrl+Shift+H`.
- `Delete` may be left both in the contextual handling and in the registry; to avoid double deletion — the registry only (or the contextual handling must check that no binding fired). It is recommended to move `Delete` to the registry and remove the hard-coded check.
- Move `F8`/`F11` to the registry (`ui.view.alignment_guides`), keep them as defaults.

### 6.4. Settings

`crates/ui-egui/src/settings.rs`:

- `Settings.shortcuts: ShortcutsConfig` (a serializable structure).
- `shortcuts_from(v, base)`:
  - if `v` is an object with `version: 2` and a `bindings` array — parse it;
  - if `v` is the old `key → action` object — migrate it;
  - if `v` is missing/not an object — keep `base`.
- `MAX_SHORTCUTS` → 4096; add `MAX_ACTIONS = 512`.
- `use_settings`: `self.shortcuts = Shortcuts::from_config(kept.shortcuts)`.
- `settings_now`: `shortcuts: self.shortcuts.to_config()`.
- `keep_settings` already compares `Settings` and writes atomically; it is enough for `ShortcutsConfig` to implement `PartialEq`.
- On migration keep backward compatibility: the old key `"WA"` → `Binding::Sequence([Key::W, Key::A])`; the action `"Wall"` → `cmd.wall`.
- If an old file has a key that is not two letters (which should not happen) — skip it.

### 6.5. Parser and serializer

- `parse_shortcuts(text: &str, registry: &Registry) -> ParsedProfile`.
- `serialize_shortcuts(profile: &Shortcuts, registry: &Registry, mode: ExportMode) -> String`.
- `parse_binding(s: &str) -> Result<Binding, BindingError>`.
- `display_binding(b: &Binding) -> String` — the canonical form.
- `display_binding_localized(b, lang) -> String` — with `(Ctrl+В)` for Russian.
- `resolve_key(name: &str) -> Option<Key>`:
  - `Key::from_name` (egui);
  - a synonym table (`Esc`, `Return`, `Option`, `Cmd`, `Primary`, punctuation);
  - a Cyrillic table (Appendix C).
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
- The dialog works with `pending`, not directly with `BimApp::shortcuts`; "Apply" copies `pending` into the application and saves.
- "Cancel"/`Esc` restores `original`.
- For capture use the same `Event::Key`, `physical_key` and `matches_exact`.
- Capture must not fire as a global binding: while `capture.is_some()`, the application's binding handling is disabled.
- Show conflicts through `find_conflict` on `pending`.
- Import/export — through `rfd::AsyncFileDialog`, as now; filters `.txt`, `.bks`, `.json`, `.xml`.

### 6.7. Find, ribbon, menus

- `find.rs`:
  - `find_matches` takes actions from the registry;
  - for each action — `shortcuts::display_for(action.id)`, up to two bindings + `+N`;
  - search by the binding string.
- `lib.rs`:
  - the helper `fn shortcut_tip(action: ActionId, text: &str) -> String` appends ` (…)`;
  - the helper `fn shortcut_text(action: ActionId) -> Option<String>`;
  - replace hard-coded labels in `big_button`/`small_button` and the menus.
- `ribbon.rs` does not store commands; change it only if a helper for strings is needed.
- `find.rs` and `shortcuts.rs` must not duplicate the action list.

### 6.8. Files that change

| File | What to do |
|---|---|
| `crates/ui-egui/src/shortcuts.rs` | New types `Binding`, `KeyChord`, `ActionDef`, `Shortcuts`; registry; parser/serializer; Markdown reference generator; dialog; capture; conflicts. As it grows — split into `shortcuts/{mod,chord,registry,file,markdown,dialog}.rs`. |
| `crates/ui-egui/src/lib.rs` | Replace the `custom_shortcuts`/`typed` fields with `shortcuts: Shortcuts`; rewrite `keys`, `shortcut()`, `shortcuts()`; remove the hard-coded `pressed`; `Ctrl+D` → Move; Home → `Ctrl+Shift+H`; tooltip helpers. |
| `crates/ui-egui/src/settings.rs` | New `shortcuts` schema, migration, limits, `use_settings`, `settings_now`. |
| `crates/ui-egui/src/find.rs` | Show several bindings, search by binding. |
| `crates/core/src/engine.rs` | Unchanged if `Command` is enough; if necessary add `shortcut_kind`/`category` or keep the table in the UI. |
| `crates/core/src/commands.rs` | Unchanged; check that all 170 commands are classified in the UI registry. |
| `crates/ui-egui/src/lang.rs`, `translations/ui/*` | New strings, translations. |
| `assets/shortcuts/*.bks` (new resource) | Default profiles BIMCraft/Revit/Archicad/Empty; `include_str!`. |
| `docs/areas/desktop.md` | Rewrite the Keyboard Shortcuts section. |
| `docs/areas/shortcuts.md` (new) | User documentation and the file format. |
| `README.md` | Update line 407 and the command/shortcut table. |
| `crates/ui-egui/tests/*` | New tests: `shortcuts_kept.rs`, `shortcuts_input.rs`, `shortcuts_file.rs`, `shortcuts_dialog.rs`, `shortcuts_md.rs`. |

### 6.9. What does not change

- Core commands, their `id`, `run`, `params` and the project format.
- `settings.json` keeps storing the other settings; only the `shortcuts` field changes.
- Autosave, undo/redo, saving the project.
- Tool behavior, except for the source of key presses.
- Localization of action names.

---

## 7. Pitfalls (must be taken into account)

1. **`consume_key` ignores extra Shift/Alt.** In egui 0.36 `InputState::consume_key` uses `Modifiers::matches_logically`: `Ctrl+D` fires with `Ctrl+Shift+D` if `Ctrl+Shift+D` was not checked earlier. For exact bindings use `matches_exact` and your own event consumption. Otherwise `Ctrl+D` and `Ctrl+Shift+D` cannot coexist.
2. **`Event::Text` is the layout, not the key.** Never build chords/sequences from text. `Event::Key` only.
3. **`physical_key` may be `None`.** On web and some platforms use the `key` fallback. For non-Latin layouts egui itself substitutes the physical key into `key`, but it is not worth relying on that alone.
4. **The Russian layout and `Ctrl+В`.** Verify that `Event::Key { key: Key::D (fallback), physical_key: Some(Key::D), modifiers: Ctrl }` with `text: "в"` gives Move. A test is mandatory.
5. **AltGr.** On Windows this is `Ctrl+Alt`; the user may assign `Ctrl+Alt+E`, and the system will generate the character `€`. If the focus is in a text field — text wins. On the canvas — warn.
6. **macOS `Cmd` vs `Ctrl`.** `Modifiers::COMMAND` on macOS is Cmd, on Windows/Linux it is Ctrl. The file's `Ctrl` must mean primary, not literal Control. Should literal Control on macOS use `Control`? That is ambiguous. Decision: `Ctrl`/`Primary` = primary; `Control` is a synonym for primary for simplicity; literal Control on macOS is not supported in v1 (document it). `Cmd`/`Meta` — literal Command.
7. **Text fields.** `Ctrl+C`/`Ctrl+V`/`Ctrl+X`/`Ctrl+A`/`Ctrl+Z` must stay with the field. Do not intercept them even if the user assigned them to an action. The exception is Save/Open/New/Find/Settings, which do not steal text.
8. **IME and dead keys.** While an IME composition is in progress, do not handle bindings; `ctx.egui_wants_keyboard_input()` and `Event::Ime` are the indicators. Dead keys (accents) must not create a binding; use `Event::Key` only.
9. **Shift and letters.** `Shift+D` is `D` with `shift=true`; do not confuse it with `Ctrl+D`. For sequences Shift must not change the key, but must not turn the sequence into a chord either.
10. **CapsLock.** Does not affect `Key`; do not use text to determine it.
11. **Key repeat.** Holding `Ctrl+D` must not launch Move many times. Ignore `repeat`.
12. **Open menus and popups.** Keep the current early `return` on `Popup::is_any_open`; otherwise menus will start running commands.
13. **Double execution.** `Delete` is currently handled both in the contextual keys and can be assigned. After moving it to the registry, remove the hard-coded check or add an "already done" flag.
14. **The `Ctrl+D` conflict with Home.** Home must be reassigned; otherwise the user scenario does not work. Old settings do not contain `Ctrl+D`, so migration is safe.
15. **Duplicate aliases (`CL`).** `column` and `structural_column` share one `CL`; do not pretend they are one command. In the registry they are two IDs. In the hash table `CL` belongs to `cmd.column`; the `cmd.column` executor, when `CL` is pressed, looks at the active tab and, with `Tab::Structure`/`Context(Structure)`, chooses `cmd.structural_column`, as now. A user binding of `CL` to another action disables this contextual rule for `CL` (the user has explicitly overridden it). There must not be two owners of one `CL` in the table.
16. **Commands with mandatory parameters.** `wall {points}`, `door {at}`, `schedule`, `render`, `open {path}` and the like. You cannot simply call `run(id, {})` — it will fail. For interactive tools use `Tool`; for the rest — an existing dialog or the "requires parameters" mark.
17. **Localization and IDs.** Do not store localized names as keys in settings; IDs only. Names are for import/search.
18. **Deterministic export.** Do not write a `HashMap` to a file; use `BTreeMap`/sorting. Otherwise diffs and round-trips will be unstable.
19. **Size and corrupt files.** Limits and atomic writing; do not block the UI on a large file.
20. **Compatibility with old settings.** An old `settings.json` must not lose bindings; a new file must be read without panicking.
21. **Tests on different OSes.** Reference strings with `Ctrl` on macOS may be displayed as `Cmd`; in tests compare the canonical model, not the platform label.
22. **Performance.** Do not iterate 4096 bindings on every key press; use hash tables.
23. **Security.** The file is data only; no `eval`, plugins, paths, OS commands.
24. **User profiles.** Do not overwrite the user profile when choosing a default one without confirmation.
25. **Reset.** "Restore Defaults" must not delete user bindings of other profiles if they are stored; in v1 a single active set is stored — clarify this in the UI.
26. **Sketch.** Do not lose the contextual behavior of `Ctrl+Z` (undoing a sketch line) when moving Undo to the registry; check `self.sketch.is_some()` before the global binding.
27. **Note editing.** While `note_edit` is active and the focus is not in a field, `note_keys` handles the keys; do not intercept them with global bindings.
28. **`typed_length`.** Digits, `.` and `,` while entering a length must go to the length, not to sequences; do not break the existing logic.
29. **`Tab`.** The current code separately cuts `Tab` out of the events for cyclic selection; preserve this behavior and do not treat `Tab` as a binding.

---

## 8. Tests

### 8.1. Unit tests (`crates/ui-egui`)

1. `parse_binding("Ctrl+D")` → `Chord { key: D, ctrl: true }`.
2. `parse_binding("Ctrl+В")` → the same as `Ctrl+D`.
3. `parse_binding("WA")` → `Sequence([W, A])`.
4. `parse_binding("WА")` (Cyrillic А) → `Sequence([W, A])`.
5. `parse_binding("Ctrl+Shift+D")` ≠ `Ctrl+D`; both exist.
6. `parse_binding("Ctrl++")` → an error/`Plus` per the rule.
7. `parse_binding("Esc")` → the error "reserved" (or a valid key but forbidden at the assignment level — clarify; the test must record the decision).
8. `display_binding` for all `Key::ALL` keys does not panic.
9. `resolve_action` by ID, English label, localized label, legacy name.
10. Registry: all IDs are unique; all `engine::commands()` are covered or in the exceptions; all `ActionDef`s with `kind != Internal` have `execute`.
11. Default profiles: no conflicts; `Ctrl+D` → `cmd.move`; Home → `Ctrl+Shift+H`.
12. `find_conflict` for chords, sequences, defaults and user bindings.
13. `add_binding`: several bindings per action; conflict; replacement; deletion.
14. `parse_shortcuts` v2: comments, CRLF, BOM, repeated actions, several bindings.
15. `parse_shortcuts` legacy JSON/XML/`KEY - Name`.
16. `parse_shortcuts` errors: unknown action, unknown key, conflict, version 3, oversized file.
17. `serialize` → `parse` → semantic equality; re-export is identical.
18. `Settings::from_json` old `{"WA":"Wall"}` → new `Shortcuts`; new JSON → new; corrupt → `base`.
19. `MAX_SHORTCUTS`/`MAX_ACTIONS` are respected.
20. `enabled` for actions without a selection/in an unsuitable view.
21. The Markdown reference contains **every** registry action, including commands without keys; the number of table rows matches the number of actions.
22. The Markdown reference distinguishes active and default bindings; a changed/deleted binding is visible as such.
23. The Markdown reference is valid: tables with escaped `|`, no raw HTML, headings and sections in place.
24. The Markdown reference is deterministic (with the "without date" option); regenerating produces the same text.
25. Importing `.md` is rejected with a clear message, settings do not change.

### 8.2. Integration tests (`egui_kittest`)

1. `Ctrl+D` (physical `Key::D`, modifiers Ctrl) → the active tool is `Tool::Move`.
2. `Ctrl+В` (physical `Key::D`, `Event::Text("в")`) → `Tool::Move`.
3. `Ctrl+D` on the Russian layout without a text event → `Tool::Move`.
4. `Ctrl+Shift+D` → Copy, not Move; `Ctrl+D` → Move.
5. `Ctrl+D` with the focus in a text field → not Move; the text `d`/`в` is entered.
6. `Ctrl+C` with the focus in a field → text copy, not a command.
7. `Ctrl+S` with the focus in a field → Save (allowlist).
8. A modal dialog is open → `Ctrl+D` is not Move; `Esc` closes the dialog.
9. A menu is open → `Ctrl+D` is not Move; `Esc` closes the menu.
10. `WA` on the Russian layout (physical W, A) → Wall.
11. `WA` with a timeout: after a pause the buffer is cleared (if the timeout is implemented).
12. Holding `Ctrl+D` (repeat) → Move once.
13. Assigning two bindings through the dialog → both work.
14. A conflict in the dialog → "Replace" removes it from the previous action.
15. "Apply" saves to `settings.json`; "Cancel" does not save.
16. Import "Replace all" → the old bindings are gone, the new ones work.
17. Import "Add" → conflicts are skipped, the report is shown.
18. Export → the file exists, is UTF-8, contains `version = 2` and all bindings.
19. Export → import → export → identical text.
20. Find shows several bindings; searching for `Ctrl+D` finds Move.
21. The ribbon tooltip contains `Ctrl+D` after assignment.
22. A new application launch with a saved `settings.json` → the bindings are in place.
23. An old `settings.json` → migration, the old `WA` works.
24. An action is unavailable → status, no fallback.
25. Importing a file with `Ctrl+В` → `Ctrl+D` works.
26. The "Reference (Markdown)…" button saves the `.md`; the file opens in a Markdown viewer and contains all sections.
27. The reference contains commands without keys and default bindings, even if the user changed them.
28. In the reference with the Russian UI there are `Ctrl+D (Ctrl+В)` hints.
29. Importing `.md` shows the message "this is a reference" and does not touch the settings.

### 8.3. Manual check (matrix)

| OS | Layouts | What to check |
|---|---|---|
| Linux | US, RU | `Ctrl+D`/`Ctrl+В`, `WA`, text in fields, AltGr |
| Windows | US, RU | `Ctrl+D`/`Ctrl+В`, AltGr, `Ctrl+Shift+Esc` |
| macOS | US, RU | `Cmd+D` (primary), literal `Ctrl`, `Cmd+Tab` |
| Any | switching the layout on the fly | bindings do not change |
| Any | external keyboard | Numpad, F-keys |

Check manually:

- every ribbon button that has a binding shows it in the tooltip;
- the File/Edit/Manage/View menus show bindings;
- Find (Ctrl+K) shows bindings and searches by them;
- import/export through the system dialog;
- reset of an action/category/everything;
- no double execution of `Delete`;
- no false triggers while entering a name, a size, or text.

### 8.4. Quality control commands

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test -p bimcraft-ui-egui
cargo test -p bimcraft-core
cargo test --workspace
```

For UI tests — `egui_kittest` following the existing example (`crates/ui-egui/tests/`).

---

## 9. Documentation and translations

1. `README.md`, line 407: rewrite the Shortcuts section — chords, several bindings, import/export, `Ctrl+D` = Move, layout independence.
2. `docs/areas/desktop.md`, lines 479–488: update the description of `shortcuts_window`, the data model, `settings.json`.
3. A new `docs/areas/shortcuts.md`:
   - a user guide;
   - the file format with examples;
   - the Russian layout table;
   - the list of reserved keys;
   - migration from the old format;
   - a "How to see all combinations" section: Manage > Shortcuts > "Reference (Markdown)…"; what goes into the file and how to read it.
4. In the README and in the application reference — a short instruction: where to find the full list, how `.md` differs from `.bks` (reference vs backup).
5. `translations/ui/bimcraft-ui.pot` and the language files: all new strings.
6. `docs/video/` — optionally a short video/GIF, if that is accepted in the project.
7. CHANGELOG/ROADMAP — note the `Ctrl+D` change (Home → Move) as user-visible.

---

## 10. Acceptance criteria

> **SPEC-009 (takes precedence):** the criteria below about `Ctrl+D` = Move and Home = `Ctrl+Shift+H` are superseded: in the BIMCraft profile `Ctrl+D` is free, Home is `Ctrl+U`.

1. In the keyboard shortcuts dialog **any command available to the user** can be assigned; all 170 core commands are present in the registry (assignable or with an explicit reason).
2. **Several** combinations can be assigned to one command; all of them work.
3. `Ctrl+D` performs "Move"; Home is moved to `Ctrl+Shift+H`.
4. `Ctrl+D` on the English layout and `Ctrl+В` on the Russian layout perform the same action.
5. The two-letter `WA`, `DR`, `MV` and others work on the Russian layout (by physical keys).
6. `Ctrl+D` and `Ctrl+Shift+D` are different bindings; the more specific one is not masked.
7. A conflict of one chord for two actions is detected; the owner can be replaced.
8. The dialog allows adding, changing and deleting one binding, deleting all, and resetting an action/category/everything.
9. Export to `.txt`/`.bks` (and `.json`) contains all bindings and the version; the file is human-readable.
10. Importing a file in "Replace" and "Add" modes works; errors are shown with line numbers; valid lines are applied.
11. Export → import → export gives a semantically identical set.
12. Bindings are saved in `settings.json` and survive a restart; the old format migrates without losing `WA` and others.
13. Text fields are not intercepted; `Ctrl+C`/`Ctrl+V`/`Ctrl+Z` work in them as usual.
14. Modal dialogs and menus do not run global bindings.
15. Ribbon tooltips, menus and Find show the assigned combinations (up to two + `+N`).
16. No command runs twice; holding a key does not repeat the action.
17. All new strings are localized; the Russian UI shows `Ctrl+D (Ctrl+В)`.
18. The tests from section 8 pass; `cargo fmt`, `clippy`, `test --workspace` are green.
19. The documentation is updated; `docs/areas/shortcuts.md` describes the format and limits.
20. No regressions: autosave, undo/redo, saving the project and the existing tools work as before.
21. There is a simple way to get acquainted with all combinations: the "Reference (Markdown)…" button (and, optionally, a preview) produces a `.md` with all actions — active, default and without keys — grouped by category.
22. The Markdown file is valid, readable in GitHub/VS Code/Obsidian, contains conflicts and reserved keys; importing `.md` is explicitly rejected with a hint to use `.bks`/`.txt`/`.json`.

---

## 11. Phased plan

### Stage 0. Registry without changing behavior

- Move `ACTIONS`/`SHORTCUTS`/aliases into `ActionDef`.
- Classify the 170 commands.
- `execute_action` as a wrapper over the current `execute_shortcut_action`.
- Coverage and ID uniqueness tests.
- Behavior does not change; the old tests are green.

### Stage 1. Binding model and input

- `Binding`, `KeyChord`, `Shortcuts`.
- Exact matching, `physical_key`, sequences.
- Remove the hard-coded `pressed`; `Ctrl+D` → Move, Home → `Ctrl+Shift+H`.
- Settings migration.
- Input tests, including `Ctrl+В`.

### Stage 2. Dialog

- A new `ShortcutsDialog` with several bindings, search, filters, conflicts, capture.
- "Apply/Cancel".
- Dialog tests.

### Stage 3. Import/export

- Parser/serializer v2, legacy, JSON, XML.
- Import preview, modes, errors.
- Atomic writing, backup.
- Markdown reference generator: all actions, active + default bindings, commands without keys, grouping, conflicts.
- "Export → import → export" tests and Markdown reference tests.

### Stage 4. Profiles, tooltips, documentation

- Built-in profiles BIMCraft/Revit/Archicad/Empty.
- Ribbon/menu/Find tooltips.
- The "Reference (Markdown)…" button and, optionally, a preview.
- README, docs, translations.
- The manual matrix.

### Stage 5. Polish and quality control

- Performance, limits, corrupt files.
- Verification on Windows/Linux/macOS.
- Final acceptance criteria.

---

## 12. Out of scope

- Macros (sequences longer than two keys, action recording, scripts).
- Contextual bindings for individual views/tools (one global table per application).
- Project-level bindings (application settings only).
- Cloud synchronization of profiles.
- Reassigning keys at the OS level.
- Mouse gestures and game controllers.
- Mobile/web versions (desktop only; `physical_key` may be absent on web).
- Export to the Archicad/Revit format as a mandatory requirement (Revit XML is optional, sequences only).
- Markdown import: `.md` is a read-only reference; use `.bks`/`.txt`/`.json` for import.
- Automatic detection of the active OS layout for key labels (the ЙЦУКЕН table and the UI language setting are used).
- Changing core commands and their parameters.

---

## Appendix A. Full core command catalog (170)

Below is a snapshot of `engine::commands()` as of 2026-11-08. The "Menu" column shows where the command is available in the interface; empty — the command exists in the API/CLI/MCP but may have no button. The implementation must classify every row (Tool/Dialog/Immediate/Input/Internal) and lose none.

| No. | id | Command (label) | Aliases | Menu |
|---:|---|---|---|---|
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

## Appendix B. BIMCraft default profile: new chords

> **SPEC-009 (takes precedence):** the rows with `Ctrl+D` = Move and Home = `Ctrl+Shift+H` in this table do not apply to BIMCraft: `Ctrl+D` is free, Home is `Ctrl+U`.

Below is the **normative** part of the BIMCraft default profile. All the old two-letter sequences (`WA`, `DR`, `WN`, `RM`, `CL`, `DI`, `TG`, `MV`, `CO`, `RO`, `MM`, `DM`, `AL`, `OF`, `DW`, `ZF`, `TL`, `CS`, `PP`, `NP`, `SA`, `VV`, `EH`, `HH`, `KS`, `UN`, `GD`, …) are kept as default bindings and are not listed line by line, so that the table does not duplicate `SHORTCUTS` and the command aliases.

| Action | ID | Default chords | Note |
|---|---|---|---|
| Move | `cmd.move` | `Ctrl+D`, `MV` | `Ctrl+D` is the user requirement; `MV` is compatibility |
| Copy | `cmd.copy` | `Ctrl+Shift+D`, `CO` | we do not take `Ctrl+C`: it belongs to text fields |
| Rotate | `cmd.rotate` | `Ctrl+R`, `RO` | as in Archicad |
| Mirror (pick axis) | `cmd.mirror` | `Ctrl+M`, `MM` | on macOS `Cmd+M` may be "minimize window" — warning |
| Mirror (draw axis) | `cmd.mirror` (draw mode) | `Ctrl+Shift+M`, `DM` | a separate ID if the mode is split |
| Offset | `cmd.offset` | `Ctrl+Shift+O`, `OF` | |
| Trim/Extend to Corner | `cmd.corner` | `Ctrl+T`, `TR` | |
| Split | `cmd.split` | `Ctrl+Shift+T`, `SL` | |
| Align | `cmd.align` | `Ctrl+Shift+A`, `AL` | |
| Delete | `cmd.delete` | `Delete`, `DE` | `Delete` is the only contextual key moved to the registry |
| Undo | `cmd.undo` | `Ctrl+Z` | |
| Redo | `cmd.redo` | `Ctrl+Shift+Z`, `Ctrl+Y` | |
| Save | `cmd.save` | `Ctrl+S` | |
| Save As | `ui.file.save_as` | `Ctrl+Shift+S` | separate behavior: choosing a path |
| Open | `cmd.open` | `Ctrl+O` | |
| New | `cmd.new` | `Ctrl+N` | |
| Find | `ui.find` | `Ctrl+K` | |
| Settings | `ui.settings` | `Ctrl+,` | |
| Properties | `ui.palette.properties` | `Ctrl+1`, `PP`, `VP` | |
| Project Browser | `ui.palette.browser` | `Ctrl+2`, `NP` | `Ctrl+2` is a new default chord |
| Export PDF | `ui.file.export_pdf` | `Ctrl+P` | |
| Start page / Home | `ui.view.home` | `Ctrl+Shift+H` | moved off `Ctrl+D` |
| Keyboard Shortcuts | `ui.settings.shortcuts` | `Ctrl+Shift+K`, `KS` | `Ctrl+K` is taken by Find |
| 3D View | `ui.view.3d` | `F3`, `3D` | |
| Floor Plan | `ui.view.plan` | `F2` | |
| Zoom to Fit | `ui.view.zoom_fit` | `Ctrl+0`, `ZF`, `ZE`, `ZA` | |
| Thin Lines | `ui.view.thin_lines` | `TL` | |
| Alignment guides | `ui.view.alignment_guides` | `F8`, `F11` | |
| Visibility/Graphics | `ui.view.vg` | `VV`, `VG` | |
| Hide element | `ui.view.hide_element` | `EH` | |
| Hide category | `ui.view.hide_category` | `VH` | |
| Temporary hide/isolate | `ui.view.temp_hide_*` | `HH`, `HC`, `HI`, `IC`, `HR`, `RH` | |
| Unhide | `ui.view.unhide` | `EU`, `UE`, `VU` | |
| 3D style | `ui.view.visual_style` | `WF`, `HL`, `SD` | |
| Graphic Display Options | `ui.view.display_options` | `GD` | |
| Snaps | `ui.snap.*` | `SE`, `SM`, `SI`, `SC`, `SP`, `SQ`, `ST`, `SN`, `SO`, `SS` | |
| Wall | `cmd.wall` | `WA` | no chord assigned by default: `Ctrl+W` closes the window on macOS; the user can assign their own |
| Door | `cmd.door` | `DR` | |
| Window | `cmd.window` | `WN` | |
| Room | `cmd.room` | `RM` | |
| Column | `cmd.column` / `cmd.structural_column` | `CL` | contextually, as now |
| Dimension | `cmd.dimension` | `DI` | |
| Tag | `cmd.tag` | `TG` | |
| Text | `cmd.text` | `TX` | |
| Section | `cmd.section` | `CR` | |
| Measure | `cmd.measure` | `ME` | |

If the implementation splits `Mirror Pick Axis` and `Mirror Draw Axis` into two actions, `Ctrl+M` and `Ctrl+Shift+M` are assigned to them; if it keeps them as one action with a `draw` parameter, both bindings lead to `cmd.mirror` with different modes. The decision must be recorded in the registry and a test.

---

## Appendix C. Russian layout table (ЙЦУКЕН → physical key)

On import and when entering `Ctrl+В`/`Ctrl+С`/…, a character is converted to a physical key of the US layout. The table applies to letters and `ё`; for other characters `Key::from_name` and the synonyms are used.

| Physical key (US) | Russian lowercase | Russian uppercase |
|---|---|---|
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

Note: the Cyrillic `А` and `Ф` are **different** physical keys (`F` and `A` respectively), as are `М` and `Ь` (`V` and `M`). That is why `Ctrl+А` and `Ctrl+Ф` do not conflict, and `Ctrl+М` is `Ctrl+V`.

If a character is not in the table, the import reports an error with the line number; the application does not guess.

---

## Appendix D. Reserved and system combinations

### D.1. Hard-reserved by the application

| Key | Why |
|---|---|
| `Esc` | Cancel/close a dialog, tool, menu |
| `Enter` | Finish entering a length, construction, confirmation |
| `Tab` | Cyclic selection, focus |
| `Backspace` | Delete the last character/step |
| `Up`/`Down`/`Left`/`Right` | Navigation, movement, modification |
| `Space` | The tool's contextual action: flip, rotate component, dimension mode |
| Single `Ctrl`, `Shift`, `Alt`, `Meta` | Not a binding |

### D.2. Assignable, but with a warning

| Combination | Risk |
|---|---|
| `Delete` | May conflict with deletion in a text field; in a field the field wins |
| `F1`–`F12` | Some may be taken by the OS/window/driver |
| `Ctrl+Alt+…` | AltGr on Windows; may generate a character |
| `Ctrl+Shift+Esc` | Windows Task Manager |
| `Alt+F4`, `Alt+Tab`, `Ctrl+Esc` | Intercepted by the OS |
| `Cmd+Tab`, `Cmd+Space`, `Cmd+Q`, `Cmd+H`, `Cmd+M`, `` Cmd+` `` | Intercepted by macOS |
| `Win+L`, `Win+D`, `Win+Tab`, `Win+...` | Intercepted by Windows |
| `Ctrl+Alt+Del` | Does not reach the application |

When assigning such combinations show a yellow warning "may not work: the combination is intercepted by the system". On import — accept with a warning. Hard-reserved ones (D.1) — do not accept, and explain the reason.

---

## Appendix E. Minimal example of `settings.json` after migration

> **SPEC-009 (takes precedence):** the example below shows `Ctrl+D` for Move and `Ctrl+Shift+H` for Home; per SPEC-009, in BIMCraft `Ctrl+D` is free and Home is `Ctrl+U`.

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

The old fragment that must migrate without loss:

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

Note: in the old format `"ZW"` could be a user key for `Floor`; in the new one it becomes the sequence `ZW`. If the `Floor` action is not found, the line is skipped with a warning and the rest are loaded.

---

## Appendix F. Implementation checklist

- [ ] The action registry covers all 170 commands and all UI actions.
- [ ] `Ctrl+D` = Move; Home = `Ctrl+Shift+H`.
- [ ] `Ctrl+В` = `Ctrl+D`; `WA` works on the Russian layout.
- [ ] Several bindings per action; all work.
- [ ] Exact modifier matching; `Ctrl+D` ≠ `Ctrl+Shift+D`.
- [ ] Text fields are not intercepted.
- [ ] Modal dialogs/menus do not run global bindings.
- [ ] Dialog: add/change/delete/reset, search, filters, conflicts.
- [ ] Import/export `.txt`/`.bks`/`.json`; preview, errors, round-trip.
- [ ] "Reference (Markdown)…" export: all actions, active + default bindings, commands without keys, conflicts, grouping, valid Markdown.
- [ ] Importing `.md` is rejected with the message "this is a reference".
- [ ] `settings.json` v2 + migration of the old format.
- [ ] Ribbon/menu/Find tooltips show several bindings.
- [ ] Translations, README, `docs/areas/desktop.md`, the new `docs/areas/shortcuts.md`.
- [ ] The tests from section 8; `fmt`, `clippy`, `test --workspace` green.
- [ ] No regressions of autosave/undo/saving the project.

---

## Appendix G. Markdown reference example

Below is not a normative template but an example of what the user sees after "Reference (Markdown)…". The set of sections and columns may be refined, but the principle must remain: **all actions**, **active and default bindings**, **commands without keys**, grouping, conflicts, reserved keys.

````markdown
# BIMCraft — keyboard shortcuts

Profile: BIMCraft · Version: 2 · Platform: Linux · Generated: 2026-11-08

## How to read

- **Active** — the combinations that will fire now.
- **Default** — the built-in values; they are visible even if you changed or deleted them.
- **Source** — "default", "custom", "changed", "deleted".
- `Ctrl+D` and `Ctrl+В` are one physical key; the reference shows the canonical form and a hint for the Russian layout.

## Contents

- [File](#file)
- [Editing](#editing)
- [Architecture](#architecture)
- [Views](#views)
- [Commands without keys](#commands-without-keys)
- [Reserved and system combinations](#reserved-and-system-combinations)
- [Conflicts](#conflicts)

## File

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| Save | `cmd.save` | `Ctrl+S` | `Ctrl+S` | default | |
| Save As | `ui.file.save_as` | `Ctrl+Shift+S` | `Ctrl+Shift+S` | default | |
| Open | `cmd.open` | `Ctrl+O` | `Ctrl+O` | default | requires a path, opens a dialog |
| New | `cmd.new` | `Ctrl+N` | `Ctrl+N` | default | |

## Editing

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| Move | `cmd.move` | `Ctrl+D` (Ctrl+В), `Ctrl+Shift+M`, `MV` | `MV` | custom | `Ctrl+D` — as in Archicad |
| Copy | `cmd.copy` | `Ctrl+Shift+D`, `CO` | `CO` | custom | we do not take `Ctrl+C` |
| Rotate | `cmd.rotate` | `Ctrl+R`, `RO` | `RO` | custom | |
| Delete | `cmd.delete` | `Delete`, `DE` | `DE` | custom | in a text field `Delete` belongs to the field |

## Architecture

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| Wall | `cmd.wall` | `WA` | `WA` | default | no chord assigned by default |
| Door | `cmd.door` | `DR` | `DR` | default | |
| Window | `cmd.window` | `WN` | `WN` | default | |
| Column | `cmd.column` | `CL` | `CL` | default | on the Structure tab `CL` → Structural Column |

## Views

| Command | ID | Active | Default | Source | Note |
|---|---|---|---|---|---|
| 3D View | `ui.view.3d` | `F3`, `3D` | `3D` | custom | |
| Floor Plan | `ui.view.plan` | `F2` | — | custom | |
| Zoom to Fit | `ui.view.zoom_fit` | `Ctrl+0`, `ZF`, `ZE`, `ZA` | `ZF`, `ZE`, `ZA` | custom | |
| Properties | `ui.palette.properties` | `Ctrl+1`, `PP`, `VP` | `PP`, `VP` | custom | |

## Commands without keys

| Command | ID | Category | Why without keys |
|---|---|---|---|
| Reference Tower | `cmd.tower` | Other | requires parameters, available through the API/CLI |
| Import CAD | `cmd.import_cad` | File | requires a path, opened from the menu |
| Schedule | `cmd.schedule` | Annotations | requires parameters, opened from a dialog |

## Reserved and system combinations

| Combination | Status | Reason |
|---|---|---|
| `Esc` | reserved by the application | cancel/close |
| `Enter` | reserved by the application | finish input |
| `Tab` | reserved by the application | cyclic selection/focus |
| `Alt+Tab` | intercepted by the OS | window switching |
| `Cmd+Tab` | intercepted by the OS (macOS) | application switching |

## Conflicts

There are no conflicts. If `Ctrl+D` were assigned both to "Move" and to "Copy", there would be a row here with both actions and a suggestion to replace the owner.

````

---

## Appendix H. Change 2026-11-12: Home is back on `Ctrl+D`

| What | Was (SPEC-009) | Now |
|---|---|---|
| Home / start page | `Ctrl+U` | `Ctrl+D` |
| `Ctrl+U` | Home | free |
| `Ctrl+D` | free | Home in the BIMCraft profile; in the Archicad profile it stays Move |
| `F8` | guides | guides (unchanged) |
| `F11` | free | free |

Rules:

1. The default Home binding in the BIMCraft profile is `Ctrl+D`; `Ctrl+U` is removed from the default profile.
2. A user binding of `Ctrl+U` or `Ctrl+D` assigned manually after the update is preserved.
3. In a text field `Ctrl+D` does not open Home; in the note editor `Ctrl+U` remains underline.
4. The tooltip of the Back button on the start page shows `Ctrl+D` again.
5. `docs/areas/shortcuts.md`, `docs/areas/desktop.md` and `README.md` are updated.

---

## Appendix I. Full implemented functionality (2026-11-12)

Below is a summary of what is actually implemented in `crates/ui-egui` and `settings.json` after all the refinements. Table I.5 shows the state before the work and the final state.

### I.1. Action and binding model

- A single action registry: `cmd.<id>` for core commands, `ui.<area>.<action>` for window actions, `legacy.<name>` for old actions without a core command.
- An action has a category, a kind (`Tool`, `Dialog`, `Immediate`, `Input`, `Internal`), default bindings and, if needed, a menu path.
- One action can have several bindings (up to 32). The overall limit is 4096 bindings.
- A binding is a chord (`Ctrl+D`, `Ctrl+Shift+D`, `F8`) or a sequence of two/three keys (`WA`, `MV`, `BMM`). Three-key sequences are set by the file; input gives priority to a two-key binding if it exists.
- Chords and sequences are read from the physical key (`physical_key`), not from `Event::Text`; `Ctrl+D` and `Ctrl+В` are one key, `WA` works on the Russian layout.
- Modifiers are compared exactly; `Ctrl+D` and `Ctrl+Shift+D` coexist; key repeat is ignored.
- `Ctrl+D` is not hard-wired in the code: it is the default Home binding, which can be deleted, reassigned to another action or restored via Reset/Reset All. Only `Esc`, `Enter`, `Tab`, `Backspace`, `Space` and the arrows are hard-reserved.
- In the BIMCraft profile Home is `Ctrl+D`, `Ctrl+U` is free; guides are `F8` only, `F11` is free. In the Archicad profile `Ctrl+D` is taken by Move.
- The old two-letter commands (`WA`, `DR`, `WN`, `RM`, `CL`, `DI`, `TG`, `MV`, `CO`, `RO`, `MM`, `DM`, `AL`, `OF`, `DW`, `ZF`, `TL`, `CS`, `PP`, `NP`, `SA`, `VV`, `EH`, `HH`, `KS`, …) are kept as default bindings.

### I.2. Storage and migration

- `Settings.shortcuts` is a version 2 object: `profile` and a `bindings` array with `action` and `keys`.
- The old `{"WA": "Wall"}` format migrates on load; after migration the file is rewritten in version 2.
- The full effective set is stored, not just the differences from the default profile; a binding deleted by the user does not come back with an update.
- Limits: `settings.json` up to 1 MiB, `MAX_SHORTCUTS = 4096`, `MAX_ACTIONS = 512`.
- `New`/`Open` do not touch bindings: these are application settings.

### I.3. Keyboard Shortcuts dialog

- Opening: Manage > Shortcuts, `KS`, `Ctrl+Shift+K`.
- Search by name, id and any active binding.
- Filters: `All`, `Assigned`, `Without keys`, `Changed`, `Custom`, `CantChanged`.
- `Changed` shows only Source = Changed, including when an extra combination was added to a default one.
- `Custom` shows bindings created where there were no defaults.
- `CantChanged` shows only non-bindable actions that already have combinations; they are read-only.
- Actions that cannot be reassigned and have no keys do not appear in the list.
- Profile selection and `Apply Profile` are moved to a separate block, visually separated from the filters.
- The window width is 1120 px; the table has Command, Category, Key combinations, Source, Actions columns.
- The header sorts on click; a repeated click flips the direction; the active column shows an `↑`/`↓` arrow; the default sort is Category ascending. Headers are separated by vertical lines.
- Combination chip: clicking the text edits, `×` on the right deletes; the cross is balanced in padding; there is no separate pencil button; `+` adds a new combination.
- If there are no combinations, the em dash `—` is not shown and `Remove All` is hidden. `Reset` remains for bindable actions.
- There is no longer a second line with `cmd.*` under the command name; the exact id is shown in the hover tooltip.
- Rows are compact: top alignment, zero vertical padding, a thin 2 px separator.
- The list scrolls with a bar at the right edge; the content does not go under the bar.
- `Esc` closes the whole window both in the normal list and in combination input mode (an unfinished capture is cancelled).

### I.4. Import, export and reference

- Import: `.txt`, `.bks`, `.json`, Revit XML and legacy lines `WA - Wall` / `WA = Wall`.
- Import has a preview: parsed bindings, errors with line numbers, conflicts; "Replace all" and "Add" modes; `shortcuts.bak` is written before replacement.
- Import limits: 256 KiB, 10,000 lines, 4096 bindings.
- Export is a deterministic text `.txt`/`.bks`, atomic write. There is no separate JSON export button; JSON remains the import format and the `settings.json` format.
- The Markdown reference is written to `bimcraft-shortcuts.md`: all actions, per-category tables with only the **Command** and **Active** columns, Active containing all active combinations including custom ones; the table of reserved and system combinations remains separate.
- After saving `.md`/`.txt` a single clickable link to the file is shown; the file opens in the system application associated with the extension (e.g. MarkText for `.md`), not in the browser.
- Importing `.md` is rejected: Markdown is a reference, not a backup.
- Find (`Ctrl+K`) searches actions not only by name/id but also by all bindings, including those that did not fit in the tooltip.

### I.5. The "before → after" table in order of the refinements

| No. | Area | Before the work | Now |
|---:|---|---|---|
| 1 | Action catalog | a separate list of names in `shortcuts.rs`; some core commands not covered | a single `cmd.*` / `ui.*` / `legacy.*` registry; all core commands classified |
| 2 | Bindings | one two-letter binding per action | several bindings per action, up to 32 |
| 3 | Chords | only hard-coded `Ctrl+S`, `Ctrl+O`, `Ctrl+D`, etc. | configurable chords in the registry and the file |
| 4 | Sequences | two letters through `Event::Text` | two- and three-key by physical keys |
| 5 | Layout | `WA` did not work on the Russian layout | physical key; `WA` and `Ctrl+В` work |
| 6 | Modifiers | `consume_key` with loose matching | exact matching, `Ctrl+D ≠ Ctrl+Shift+D` |
| 7 | Key repeat | holding could repeat the command | repeat is ignored |
| 8 | Profiles | effectively one set | BIMCraft, Revit-like, Archicad-like, Empty and Custom |
| 9 | `Ctrl+D` | hard-wired to Home | an ordinary default Home binding; reassignable |
| 10 | Home | `Ctrl+U` after SPEC-009 | `Ctrl+D`; `Ctrl+U` free |
| 11 | Guides | `F8` and `F11` | `F8` only; `F11` free |
| 12 | `settings.json` | old `key → name` object, 64 KiB | version 2, full set, 1 MiB, migration |
| 13 | Limits | 256 bindings | 4096 bindings and 512 actions |
| 14 | Source | default / custom / changed / deleted | `CantChanged` added |
| 15 | Non-bindable actions | shown as ordinary, sometimes with `Remove All` | read-only, no `+`, `Reset`, `Remove All`; hidden when without keys |
| 16 | Filters | All, Assigned, Unassigned, Changed | Custom and CantChanged added |
| 17 | Changed | showed everything except default | strictly only Source = Changed |
| 18 | Additional combination | was not always reflected as a change | adding to the defaults yields Changed |
| 19 | Custom | no separate filter | a separate filter for bindings without a default basis |
| 20 | Dialog search | by name/id and some bindings | by name, id and all bindings |
| 21 | Find `Ctrl+K` | name/id and the first tooltip | all bindings, including `+N` |
| 22 | Dialog window | 900 px, columns with auto width | 1120 px, fixed widths, a wide Command column |
| 23 | Header | no sorting | sorting by every column, `↑`/`↓` arrows, separators; Category by default |
| 24 | Profile in the dialog | on the same row as the filters | a separate block below the filters |
| 25 | Rows | vertical centering, large padding | top alignment, compact height, a thin separator |
| 26 | Combination chip | `WA ×` deleted, the pencil edited | clicking the text edits, `×` on the right deletes, no pencil |
| 27 | Empty combinations | `—` was shown, `Remove All` always | `—` removed, `Remove All` only when combinations exist |
| 28 | Command id | a second line under the name | hidden; shown in the tooltip |
| 29 | `Esc` | in capture it closed only the capture | closes the whole window |
| 30 | Scrolling | the bar was inset from the edge, rows went under it | the bar is at the right edge, the content ends before it |
| 31 | Import | JSON/XML/legacy only, no preview | `.txt`/`.bks`/`.json`/XML, preview, errors, modes, backup |
| 32 | Export | text and JSON | text only; the JSON button removed, JSON import kept |
| 33 | Markdown | 6 columns, Starting/Source/Note, extra sections | 2 columns Command/Active; all active bindings, including custom |
| 34 | Link to the saved file | the path was duplicated as text and as a link | one link, opens in the system application |
| 35 | `.md` import | `.md` was among the import filters | `.md` is rejected as a reference |
| 36 | Documentation | was in Russian | `docs/areas/shortcuts.md` in English; new `.md` files in English |
| 37 | Checks | there was no registry coverage check | unit tests for the registry, parser, Markdown, three-key sequences, Source added |

### I.6. Limitations that remain deliberately

- Commands with mandatory parameters and without an existing UI collector (`Conduit Fill`, `Schedule`, `Render`, etc.) are not assignable: a key cannot collect parameters by itself. They are shown only if they already have a binding, in the `CantChanged` group, and read-only.
- Three-key sequences are set by the file; the dialog capture adds a sequence after two keys.
- In text fields `Ctrl+D` belongs to the field; in the note editor `Ctrl+U` remains underline.
- `.md` is a reference only; the backup is `.bks`, `.txt` or `.json`.
