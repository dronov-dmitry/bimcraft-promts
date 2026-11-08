# SPEC-009. Freeing `Ctrl+D` for the User and Removing the Duplicate `F11` Binding

**Status:** **implemented** (except for items dependent on TS-008: assigning modifier-key combinations, profile migration, and the one-time notification)  
**Date:** 2026-11-08  
**Project:** BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)  
**Related documents:** `TS-008-hotkeys.md` (custom shortcuts, action registry, settings migration), `BIMCraft/docs/areas/desktop.md` (Start page section), `BIMCraft/README.md` (Start page / Shortcuts section)  
**Task type:** Changes to start-page and hardcoded shortcuts, UX refinement, settings migration.

> **Implementation (2026-11-08).** Implemented according to option 5.1 because TS-008 has not yet been implemented. Branch
> `feature/free-ctrl-d-and-f8`, PR [#308](https://github.com/IPedrax/BIMCraft/pull/308): Home is now bound to `Ctrl+U`, `Ctrl+D`
> is free, alignment guides are available only on `F8`, and `F11` is free; tooltips, tests, README, and `docs/areas/shortcuts.md` have been updated.
> Assigning `Ctrl+D`/`F11` to actions, profile migration, and the one-time notification have been deferred until TS-008: without a modifier-key
> binding registry, there is nothing to assign, and the notification would promise functionality that is not yet available.

> **Priority.** SPEC-009 refines the decisions in TS-008 regarding `Ctrl+D`, Home, and `F8`/`F11`. If any items in TS-008 conflict with this document, SPEC-009 takes precedence:
>
> - Home is bound to `Ctrl+U`, not `Ctrl+D` or `Ctrl+Shift+H`;
> - `Ctrl+D` is **unassigned** by default and is not assigned to either Home or Move;
> - **Alignment Guides** are bound to **`F8` only** by default, with no `F11`.
>   The "Archicad-like" profile may assign `Ctrl+D` to Move, but this is a profile choice, not BIMCraft's default behavior.

---

## 0. General Working Rules

- Commits must be atomic, one item per commit; commit messages must follow the branch's style (an imperative title, with the body explaining *why*).
- The language for code, identifiers, code comments, and commit messages is English. Documentation in `docs/` and this file are in Russian.
- UI strings must use `t()` / `tf()` / `msg!()` only; new strings must be added to `translations/ui/bimcraft-ui.pot` and the translations.
- Do not break existing user-defined shortcuts: if the user explicitly assigned `Ctrl+D` or `F11`, preserve that choice.
- Cover every behavior change with a test; use `egui_kittest` for UI tests.
- Do not use `subagent` or parallel agents (user requirement); all work must be performed in the current session.

---

## 1. Context and Task

### 1.1. Current Behavior

The following shortcuts are hardcoded in `crates/ui-egui/src/lib.rs`:

| Shortcut                    | Action                                                                                    | Location                              |
| --------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------- |
| `Ctrl+D` (`Cmd+D` on macOS) | `toggle_home()` — opens the Start page over the project or returns to the project from it | around line 1851                      |
| `F8`                        | `toggle_alignment_guides()` — toggles alignment guides                                    | around line 1902                      |
| `F11`                       | Same action, `toggle_alignment_guides()`                                                  | same location, via the same condition |

There are also two-letter sequences, `AG` and `OG`, which call `toggle_alignment_guides()`. These are legacy Revit-style aliases, not function keys. Their status is not changed separately by this specification (see 4.3.6).

### 1.2. Why `Ctrl+D` Must Be Freed

Currently, `Ctrl+D` is assigned to the **Start page**. This is a valuable, convenient one-handed shortcut:

- The left `Ctrl` key is pressed with the little finger and `D` with the index or middle finger of the same hand, without stretching or involving the other hand.
- Users are accustomed to assigning such shortcuts to frequently used work commands rather than rarely used toggles.
- In Archicad, `Ctrl+D` is assigned to **Drag** (move/copy), one of the most frequently used modeling operations.
- Users migrating from Archicad expect to be able to assign `Ctrl+D` to **Move**.

At the same time, **opening the Start page is a function that practically does not require a keyboard shortcut**:

- Users spend much more time working directly in a project than opening the Start page.
- The Start page is mainly needed at startup, when selecting another project, or when returning to the recent-files list.
- A Home button on the Quick Access Bar, a Back arrow, and a menu item are sufficient.
- Keeping the most convenient shortcut occupied by a rarely used function is a waste.

Therefore, the proposal is:

1. Move Home from `Ctrl+D` to `Ctrl+U`.
2. Free `Ctrl+D`: it must **not be assigned to anything** by default.
3. Allow users to assign it to whichever action they prefer — especially **Move** for users migrating from Archicad.

`Ctrl+U` is also a left-hand shortcut and is readily accessible, making it more than sufficient for the rarely used Home action. Home remains visible through the Home button, the Back arrow, and the `Ctrl+U` tooltip.

### 1.3. Why Having Both `F8` and `F11` Is a Problem

Currently, `F8` and `F11` perform the same action:

```rust
if pressed(Modifiers::NONE, Key::F8) || pressed(Modifiers::NONE, Key::F11) {
    self.toggle_alignment_guides();
}
```

Problems:

1. **Duplication.** One function is assigned to two keys; users do not know which one is the "real" shortcut, and documentation and tests can become inconsistent.
2. **`F11` is a poor choice.** On many systems and in many windows, `F11` means full-screen mode or is intercepted by the window manager. An application that captures `F11` creates a conflict.
3. **An unnecessarily occupied key.** If `F11` is not needed for alignment guides, it is better to free it so users can assign it to something else (or leave it to the operating system).
4. **`F8` is already established.** It is documented in BIMCraft and covered by a test; it should remain the only function key for this action.

Decision: **keep only `F8`; remove `F11` from the application.** `F11` becomes free and is not assigned by default.

### 1.4. User Scenarios

- As an Archicad user, I open Settings and assign `Ctrl+D` to **Move**. This is now possible because `Ctrl+D` is no longer occupied by Home.
- As a user, I press `Ctrl+U` and go to the Start page; pressing it again returns me to the project.
- As a user with a Russian keyboard layout, I press `Ctrl+Г` (the physical `U` key) and get the same Home behavior as with `Ctrl+U` on an English layout.
- As a user, I press `F8` to toggle alignment guides; `F11` no longer does anything unless I assign it myself.
- As a user, I see `Ctrl+U` in the Home tooltip, and the help documentation says that `Ctrl+D` and `F11` are free.
- As a user who has already manually assigned `Ctrl+D` or `F11`, I do not lose my settings after an update.

---

## 2. Current Implementation (Based on the Code)

### 2.1. `Ctrl+D` and Home

- `crates/ui-egui/src/lib.rs`, around line 1851:

  ```rust
  if pressed(Modifiers::COMMAND, Key::D) {
      self.toggle_home();
  }
  ```

  The check is placed **after** `ctx.egui_wants_keyboard_input()`, so `Ctrl+D` does not toggle Home while a text field is active.

- `crates/ui-egui/src/start.rs`:

  - line 47: comment `/// Whether it was brought up over a project, which Back (and Ctrl+D, as in Revit) goes back to.`
  - line 149: comment `/// Home and Ctrl+D: the start page over the project, or, from it, back to the project...`
  - line 110: the Back button tooltip is built as `tf("Back to the project ({}D)", ...)`, which also hardcodes `D`.

- `crates/ui-egui/src/tests/start.rs`:

  - test `ctrl_d_and_the_back_arrow_go_between_the_start_page_and_the_project` checks `Ctrl+D`;
  - a separate check verifies that `Ctrl+D` in a text field does not leave the project.

- `docs/areas/desktop.md`, line 12 — description of Home and `Ctrl+D`.

- `README.md`, line 417 — description of the Start page and `Ctrl+D`.

- `TS-008` — proposes `Ctrl+D` for Move and `Ctrl+Shift+H` for Home. SPEC-009 supersedes that decision.

### 2.2. `F8` and `F11`

- `crates/ui-egui/src/lib.rs`, around line 1902:

  ```rust
  if pressed(Modifiers::NONE, Key::F8) || pressed(Modifiers::NONE, Key::F11) {
      self.toggle_alignment_guides();
  }
  ```

- In `crates/ui-egui/src/tests.rs`, the test `shortcut_toggles_alignment_guides` checks only `F8`, followed by `AG`/`OG`. There is no test for `F11`, so the duplicate binding is not even covered by tests.

- `AG` and `OG` are two-letter sequences in `BimApp::shortcut()`; they also call `toggle_alignment_guides()`.

### 2.3. `Ctrl+U` Conflict with the Notes Editor

`Ctrl+U` is **already used** in `crates/ui-egui/src/notes.rs`:

- When editing a text note with Override Formatting enabled, `Ctrl+U` underlines the selected text.
- Around lines 337–341: `Key::B | Key::I | Key::U if command`.
- Tooltip: `Underline (Ctrl+U)`.

This does not prevent moving Home to `Ctrl+U`, but it requires an explicit priority rule:

- **While editing a note**, `Ctrl+U` remains the underline shortcut (text context takes priority over global Home).
- **Outside note editing**, `Ctrl+U` opens/closes the Start page.
- In an ordinary text field, `Ctrl+U` must not open Home (the existing `egui_wants_keyboard_input` check applies).

If the product later decides that `Ctrl+U` must **always** open Home, the underline shortcut will have to be moved separately (for example, to `Ctrl+Shift+U`). This is not required by SPEC-009; context-based priority is sufficient.

---

## 3. Terminology

| Term                     | Meaning                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| **Home / Start page**    | The screen for selecting a project, viewing recent files, and creating a new project (`start.rs`, `BimApp::home`, `toggle_home`). |
| **Occupied shortcut**    | A shortcut assigned to an action by default and active without user configuration.                                                |
| **Free shortcut**        | A shortcut that is not assigned to any action in the default profile; the user can assign it.                                     |
| **Default profile**      | The default set of bindings (BIMCraft/Revit/Archicad/Empty) defined in TS-008.                                                    |
| **User-defined binding** | A binding explicitly added or changed by the user; it takes precedence over the default binding.                                  |
| **Function key**         | `F1`–`F24`; SPEC-009 concerns `F8` and `F11`.                                                                                       |
| **Legacy sequence**      | A two-letter/digit Revit-style sequence (`AG`, `OG`, `WA`), not a function key.                                                   |

---

## 4. Functional Requirements

### 4.1. Freeing `Ctrl+D`

1. Remove the hardcoded `Ctrl+D → toggle_home()` check from `BimApp::keys()`.
2. In the default BIMCraft profile, **do not assign** `Ctrl+D` to Home, Move, or any other action. After the update, `Ctrl+D` must be free.
3. In the shortcuts dialog, `Ctrl+D` must be available for assignment to any action; if the user assigns it to **Move**, it must work as a normal binding (after TS-008 is implemented).
4. If TS-008 has not yet been implemented, `Ctrl+D` simply stops doing anything; assignment support will arrive with TS-008. This is an acceptable intermediate state, but both specifications must be aligned for release.
5. The "Archicad-like" profile **may** assign `Ctrl+D` to Move — that is the purpose of this profile. The default BIMCraft profile must not do so.
6. If the user has already explicitly assigned `Ctrl+D` to an action (after TS-008), preserve that binding. Only the **default** Home binding is removed.
7. `Ctrl+D` must not be reserved: it must not be included in the list of hard-reserved keys (Esc, Enter, Tab, Backspace, arrow keys, Space).

### 4.2. Moving Home to `Ctrl+U`

1. Replace the `Ctrl+D` check with `Ctrl+U`:

   ```rust
   if pressed(Modifiers::COMMAND, Key::U) {
       self.toggle_home();
   }
   ```

2. The behavior of `toggle_home()` does not change:

   - if the Start page is not visible, show it over the project (`home.over = true`);
   - if it is visible and `home.over` is true, return to the project;
   - if no project is open, remain on the Start page (there is nowhere to return to).

3. `Ctrl+U` must work based on the **physical** `U` key: on a Russian keyboard layout, this is `Ctrl+Г`. This follows from TS-008; even if TS-008 has not yet been implemented, `egui::Key::U` is already generated by the physical key, so the behavior is the same.

4. In text fields, `Ctrl+U` must not open Home: the existing `ctx.egui_wants_keyboard_input()` check applies.

5. While editing a note, `Ctrl+U` remains the underline shortcut (see 2.3 and 4.4). This is an intentional context-specific exception and must be reflected in the help and tooltip.

6. If TS-008 is implemented, the `ui.view.home` action receives the default `Ctrl+U` binding; `Ctrl+D` and `Ctrl+Shift+H` are removed from the default Home bindings.

7. The Back button tooltip on the Start page (`start.rs`, around line 110) must display `Ctrl+U` (or `⌘U` on macOS), not `Ctrl+D`.

8. Update the comments in `start.rs` (lines 47 and 149) to refer to `Ctrl+U`.

### 4.3. `F8` Only for Alignment Guides

1. Remove `Key::F11` from the condition:

   ```rust
   if pressed(Modifiers::NONE, Key::F8) {
       self.toggle_alignment_guides();
   }
   ```

2. `F8` remains the only **function key** for this action and its default binding.

3. `F11` must no longer call `toggle_alignment_guides()` and must not be assigned to anything by default. It is free for the user or the operating system.

4. If TS-008 is implemented, the `ui.view.alignment_guides` action in the default BIMCraft profile must retain only `F8`; remove `F11` from the default bindings.

5. If the user has explicitly assigned `F11` to an action, preserve that binding. Only the default `F11` binding for alignment guides is removed.

6. `AG`/`OG` are legacy sequences, not function keys. They are not removed by SPEC-009, to avoid disrupting established habits. If the product wants exactly one binding, that can be decided separately, or the user can remove them in the dialog. They may remain in the default BIMCraft profile for compatibility; the "F8 only" criterion applies to function keys.

7. If TS-008 has not yet been implemented, `AG`/`OG` continue to work as before; `F11` stops working.

8. `F11` must not be consumed by the application: if the OS or window manager uses it for full-screen mode, that behavior must remain available.

### 4.4. Context and Priority

The order remains the same as in TS-008, with the following clarifications:

1. **Modal dialogs** — receive keys first; `Ctrl+U`/`F8` are not global shortcuts inside a dialog.
2. **Text field** — `Ctrl+U` belongs to the field (or is ignored); Home is not opened.
3. **Note editing** — `Ctrl+U` = Underline (Override Formatting); Home is not opened. `F8` must not toggle alignment guides in the note editor if the editor captures keyboard input.
4. **Sketch** — `Ctrl+U` does not conflict with `Ctrl+Z`; sketch behavior does not change.
5. **Regular canvas/project** — `Ctrl+U` = Home, `Ctrl+D` = free (or user-defined binding), `F8` = alignment guides, `F11` = free.
6. **Open menu/popup** — keys belong to the menu; after it closes, they belong to the application again.

### 4.5. Migration and Compatibility

#### 4.5.1. If TS-008 Has Not Yet Been Implemented

Only the hardcoded checks change:

- `Ctrl+D` → `Ctrl+U` for Home;
- `F8 || F11` → `F8` only.

User settings are not migrated: legacy `custom_shortcuts` contain only two-letter sequences, so `Ctrl+D`/`F11` could not have been stored there.

#### 4.5.2. If TS-008 Has Already Been Implemented

The default profile and/or `settings.json` must be migrated:

| State                                   | Before                                           | After                                                                       |
| --------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------- |
| Home, default binding                   | `Ctrl+D` or `Ctrl+Shift+H` (according to TS-008) | `Ctrl+U`                                                                    |
| `Ctrl+D`, default Move binding (TS-008) | `Ctrl+D`                                         | Remove from the default BIMCraft profile; keep only in the Archicad profile |
| `Ctrl+D`, user-defined binding          | Whatever the user assigned                       | Preserve                                                                    |
| Alignment guides, default bindings      | `F8`, `F11`                                      | `F8`                                                                        |
| `F11`, user-defined binding             | Whatever the user assigned                       | Preserve                                                                    |
| `AG`/`OG`, default bindings             | `AG`, `OG`                                       | No change under SPEC-009                                                      |

Rules:

1. Replace **default** bindings according to the table; **user-defined** bindings must be preserved.
2. If `settings.json` does not distinguish between "default" and "user-defined", introduce such a marker in TS-008 or determine the type by comparing the binding with the profile: if it matches the old default, treat it as a default binding; if it differs, treat it as user-defined.
3. If, after migration, `Ctrl+D` is assigned to both Home and Move (for example, because the user assigned it manually), show a conflict in the dialog and ask the user to choose which action owns it. Do not silently delete the user-defined binding.
4. If `settings.json` is corrupted or its version is unknown, do not modify it automatically. Show its status and retain the current behavior until the user takes action.

#### 4.5.3. One-Time Notification

After the update, show the following notification once (not on every launch):

> "`Ctrl+D` is now free. The Start page shortcut is `Ctrl+U`. You can assign `Ctrl+D` to **Move** in Manage > Shortcuts."

Do not show the notification if:

- the user has already opened the shortcuts dialog in this version;
- the user has explicitly assigned `Ctrl+D` themselves;
- this is the first launch with no project (the Start page is already open).

### 4.6. UI, Tooltips, and Documentation

1. **Quick Access Bar, Home button.** Tooltip: `Home (Ctrl+U)`; on macOS: `Home (⌘U)`. If TS-008 is implemented, the label must be generated from the registry rather than hardcoded.
2. **Back button on the Start page.** Tooltip: `Back to the project (Ctrl+U)` / `(⌘U)`.
3. **Shortcuts dialog.** `ui.view.home` displays `Ctrl+U`; `Ctrl+D` displays "unassigned"; `ui.view.alignment_guides` displays `F8`; `F11` displays "unassigned".
4. **Markdown help (TS-008, 4.8.4).** The help must show:
   - Home: active/default binding `Ctrl+U`;
   - `Ctrl+D`: "free" (or the user-defined binding, if one exists);
   - alignment guides: `F8`;
   - `F11`: "free";
   - note: "`Ctrl+U` in the note editor means Underline."
5. **README.md**, line 417: replace `Ctrl+D` with `Ctrl+U` and explain that `Ctrl+D` is free for user-defined bindings.
6. **docs/areas/desktop.md**, line 12: replace `Ctrl+D` with `Ctrl+U`.
7. **docs/areas/shortcuts.md** (from TS-008): add a section titled "What Changed in SPEC-009": Home → `Ctrl+U`, `Ctrl+D` is free, `F11` is free, and `F8` is the only function key for alignment guides.
8. All new strings must use `t()`/`tf()` and be added to translations.

### 4.7. Scope Boundaries

1. Do not change the behavior of `toggle_home()` or `toggle_alignment_guides()` — change only their shortcuts.
2. Do not change `Ctrl+U` in the note editor (Underline); preserve context-based priority.
3. Do not change `AG`/`OG` unless the product makes a separate decision; these are legacy sequences.
4. Do not change other uses of `F8`/`F11` elsewhere, if any (a code search is mandatory).
5. Do not touch `Ctrl+D` in text fields: it remains the field's behavior.
6. Do not delete user-defined bindings if they were explicitly configured.

---

## 5. Technical Plan

### 5.1. If TS-008 Has Not Yet Been Implemented

Minimal changes:

1. `crates/ui-egui/src/lib.rs`:
   - around line 1851: `Key::D` → `Key::U`;
   - around line 1902: remove `|| pressed(Modifiers::NONE, Key::F11)`.
2. `crates/ui-egui/src/start.rs`:
   - line 47: `Ctrl+D` → `Ctrl+U`;
   - line 149: `Home and Ctrl+D` → `Home and Ctrl+U`;
   - line 110: `tf("Back to the project ({}D)", ...)` → `tf("Back to the project ({}U)", ...)`.
3. `crates/ui-egui/src/tests/start.rs`:
   - rename/rewrite test `ctrl_d_and_the_back_arrow_go_between_the_start_page_and_the_project` to `ctrl_u_and_the_back_arrow_go_between_the_start_page_and_the_project`;
   - replace all `Key::D` occurrences with `Key::U`;
   - add a check that `Ctrl+D` no longer opens Home;
   - retain the check that `Ctrl+U` does not open Home in a text field.
4. `crates/ui-egui/src/tests.rs`:
   - keep the `F8` test;
   - add a check that `F11` does not toggle `alignment_guides`;
   - retain existing `AG`/`OG` behavior unless those sequences are removed.
5. Update the text in `README.md`, `docs/areas/desktop.md`, and `docs/areas/shortcuts.md`.
6. `translations/ui/*` — only if new strings are added; existing `Home`/`Back` strings do not change, only the shortcut substitution changes.

### 5.2. If TS-008 Has Already Been Implemented

Additional changes:

1. `crates/ui-egui/src/shortcuts.rs` / registry:
   - `ui.view.home`: default binding `Ctrl+U` (not `Ctrl+D` or `Ctrl+Shift+H`);
   - `ui.view.alignment_guides`: only `F8` as the default binding (no `F11`);
   - `Ctrl+D` must be absent from the default BIMCraft profile;
   - the Archicad profile: `cmd.move` may have `Ctrl+D`.
2. `crates/ui-egui/src/settings.rs`:
   - migrate the default profile according to table 4.5.2;
   - preserve user-defined `Ctrl+D`/`F11` bindings;
   - show a dialog if there is a conflict; do not lose data.
3. `crates/ui-egui/src/start.rs` and `lib.rs`:
   - if tooltips are already generated from the registry, do not hardcode anything; otherwise update the fallback strings.
4. Tests:
   - `Ctrl+U` = Home;
   - `Ctrl+Г` = Home (physical `U`);
   - `Ctrl+D` = free, but can be assigned to Move and works;
   - `F8` = alignment guides;
   - `F11` = free, but can be assigned and works;
   - migration of the old profile: Home `Ctrl+D` → `Ctrl+U`, `Ctrl+D` removed from the default BIMCraft Move binding, user-defined bindings preserved;
   - `.md` help displays the new values.

### 5.3. Files to Be Changed

| File                                | Required change                                                  |
| ----------------------------------- | ---------------------------------------------------------------- |
| `crates/ui-egui/src/lib.rs`         | `Ctrl+D` → `Ctrl+U`; `F8 \|\| F11` → `F8` only; Home tooltip.    |
| `crates/ui-egui/src/start.rs`       | Update comments and Back tooltip to `Ctrl+U`.                    |
| `crates/ui-egui/src/tests/start.rs` | Home tests use `Ctrl+U`; add negative test for `Ctrl+D`.         |
| `crates/ui-egui/src/tests.rs`       | `F8` test; negative test for `F11`; handle `AG`/`OG` as decided. |
| `crates/ui-egui/src/shortcuts.rs`   | If TS-008 is implemented: default Home/alignment-guide bindings. |
| `crates/ui-egui/src/settings.rs`    | If TS-008 is implemented: migration.                             |
| `README.md`                         | Line 417: `Ctrl+U`, `Ctrl+D` is free.                            |
| `docs/areas/desktop.md`             | Line 12: `Ctrl+U`.                                               |
| `docs/areas/shortcuts.md`           | "SPEC-009 Changes" section.                                        |
| `translations/ui/*`                 | New notification/tooltips strings, if required.                  |
| `TS-008-hotkeys.md`                 | Add a reference to SPEC-009 and a note about its priority.         |

### 5.4. What Does Not Change

- `toggle_home()` and `toggle_alignment_guides()`.
- `Ctrl+U` in notes (Underline).
- `AG`/`OG` (unless a separate decision is made).
- Other global bindings (`Ctrl+S`, `Ctrl+O`, `Ctrl+Z`, `Ctrl+K`, …).
- Project format and settings, except for the `shortcuts` field.

---

## 6. Potential Pitfalls

1. **`Ctrl+U` is already used in notes.** In `notes.rs`, `Ctrl+U` means Underline. Do not simply add a global check before `note_keys`: while a note is being edited, text editing takes priority. A test is mandatory.
2. **`Ctrl+U` in text fields.** On some platforms, `Ctrl+U` in a field means "delete line" (readline) or triggers another action. The existing `egui_wants_keyboard_input()` check must remain before the global Home check.
3. **Russian keyboard layout.** `Ctrl+U` is the physical `U` key, labeled `Г` on the Russian ЙЦУКЕН layout. The tooltip/help should display `Ctrl+U (Ctrl+Г)`; otherwise, users may think the shortcut does not work.
4. **macOS.** `Ctrl` in the file/document means the primary modifier, i.e. `Cmd`; Home is `Cmd+U`. In notes, `Cmd+U` means Underline; text-editing priority is preserved.
5. **`F11` and full-screen mode.** After the binding is removed, `F11` may be passed to the window manager/OS. This is expected; the application must not consume it.
6. **Other uses of `F11`.** Before making the change, check that `F11` is not used elsewhere (search the codebase). Currently, it appears only in the `F8 || F11` condition.
7. **`F8` in text fields.** `F8` is currently checked after `egui_wants_keyboard_input()`, so it does not trigger in a field. Preserve this behavior.
8. **`Ctrl+D` in text fields.** After freeing `Ctrl+D`, it must not unexpectedly start opening Home or executing Move in a field; the field has priority.
9. **User-defined bindings.** Do not "free" `Ctrl+D`/`F11` by deleting shortcuts manually assigned by the user. Only default bindings are removed.
10. **Conflict after migration.** If the user manually assigned Home to `Ctrl+U` and we add the default Home binding to `Ctrl+U`, a duplicate may occur. Do not create a second owner; update the existing binding instead.
11. **Archicad profile.** In this profile, assigning `Ctrl+D` to Move is expected. Do not globally enforce that "`Ctrl+D` is free" across all profiles; this criterion applies to the default BIMCraft profile.
12. **Notification.** Show it once; do not display it on every launch. Do not show it if the user has already configured `Ctrl+D` themselves.
13. **Documentation.** There are several references to `Ctrl+D` in the README and docs. Update all of them to avoid confusing users.
14. **TS-008.** If TS-008 implementation is already in progress, do not leave `Ctrl+D` = Move and `Ctrl+Shift+H` = Home in the default profile: this directly contradicts SPEC-009.
15. **AG/OG.** If the product decides to keep them, the help must honestly state that the action has `F8` and legacy sequences. If it decides to remove them, update tests and documentation. Do not leave a silent inconsistency.

---

## 7. Tests

### 7.1. Unit/UI Tests

1. `Ctrl+U` opens the Start page over the project.
2. Pressing `Ctrl+U` again returns to the project.
3. Pressing `Ctrl+U` on the Start page when no project is open does not break anything.
4. `Ctrl+Г` (physical `U` on the Russian layout) does the same as `Ctrl+U`.
5. `Ctrl+D` does **not** open the Start page (after it is freed).
6. `Ctrl+D` in a text field does not open Home or execute Move.
7. `Ctrl+U` in a text field does not open Home.
8. `Ctrl+U` while editing a note with Override Formatting underlines the text and does not open Home.
9. `Ctrl+U` without Override Formatting in a note displays the tooltip/does not underline, but also does not open Home.
10. `F8` toggles `alignment_guides`.
11. `F11` does **not** toggle `alignment_guides`.
12. `F11` in a text field does nothing.
13. If TS-008 is implemented: `Ctrl+D` can be assigned to Move and works.
14. If TS-008 is implemented: `F11` can be assigned to an arbitrary action and works.
15. If TS-008 is implemented: the default BIMCraft profile has no `Ctrl+D` binding, and Home = `Ctrl+U`.
16. If TS-008 is implemented: in the Archicad profile, `Ctrl+D` = Move.
17. Migration: old Home `Ctrl+D` → `Ctrl+U`.
18. Migration: user-defined `Ctrl+D` is preserved.
19. Migration: default `F11` is removed, user-defined `F11` is preserved.
20. The `.md` help displays `Ctrl+U` for Home, `F8` for alignment guides, and `Ctrl+D`/`F11` as free.

### 7.2. Manual Verification

- Open a project and press `Ctrl+U` — the Start page appears; press it again — the project reappears.
- Press `Ctrl+D` — nothing happens (in the default BIMCraft profile).
- On the Russian layout, press `Ctrl+Г` — the same as `Ctrl+U`.
- Select a note, enable Override Formatting, and press `Ctrl+U` — the text is underlined; Home does not open.
- Press `F8` — alignment guides toggle on/off.
- Press `F11` — nothing happens; if the OS enables full-screen mode, that is handled by the OS.
- Assign `Ctrl+D` to Move (if TS-008 is implemented) — Move works.
- Assign `F11` to any action — it works.
- Check the Home/Back tooltips, README, docs, and `.md` help.

### 7.3. Quality Assurance Commands

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test -p bimcraft-ui-egui
cargo test -p bimcraft-core
cargo test --workspace
```

---

## 8. Acceptance Criteria

1. `Ctrl+U` opens/closes the Start page in the same way that `Ctrl+D` did previously.
2. `Ctrl+D` does **nothing** by default in the BIMCraft profile and is available for assignment.
3. `Ctrl+D` can be assigned to **Move** (after TS-008) and works.
4. On the Russian keyboard layout, `Ctrl+Г` works as `Ctrl+U`.
5. `Ctrl+U` in a text field does not open Home.
6. `Ctrl+U` in the note editor remains Underline and does not open Home.
7. `F8` toggles alignment guides.
8. `F11` no longer toggles alignment guides and is free by default.
9. `F11` can be assigned to another action (after TS-008), and it works.
10. User-defined `Ctrl+D`/`F11` bindings are not lost during migration.
11. The default Home binding is migrated from `Ctrl+D` (or `Ctrl+Shift+H` from TS-008) to `Ctrl+U`.
12. Home/Back tooltips, README, docs, and `.md` help are updated.
13. The one-time notification is displayed and is not repeated.
14. `cargo fmt`, `clippy`, and `test --workspace` pass.
15. No regressions occur in the Start page, notes, alignment guides, or other shortcuts.

---

## 9. Documentation and Translations

1. `README.md`:
   - line 417: `Ctrl+D` → `Ctrl+U`;
   - add: "`Ctrl+D` is free for user-defined bindings; in the Archicad profile, it may be assigned to Move."
2. `docs/areas/desktop.md`:
   - line 12: `Ctrl+D` → `Ctrl+U`;
   - clarify that Home is a rarely used function and has therefore been moved to `Ctrl+U`.
3. `docs/areas/shortcuts.md`:
   - add a new section, "SPEC-009 Changes";
   - include a "before → after" table;
   - add a note about `Ctrl+U` in notes;
   - add a note about `F11` and the OS.
4. `TS-008-hotkeys.md`:
   - add a note at the beginning stating that decisions about `Ctrl+D`/Home and `F8`/`F11` are refined by SPEC-009;
   - in sections 4.12, 5.5, 10, and appendices B/E, SPEC-009 takes precedence.
5. `translations/ui/*`: add new notification and tooltip strings; existing `Home`/`Back` strings do not need to be translated again.
6. CHANGELOG: note that `Ctrl+D` has been freed, Home has moved to `Ctrl+U`, and `F11` no longer duplicates `F8`.

---

## 10. Out of Scope

- Moving the note editor's underline shortcut from `Ctrl+U` to another combination (if required, create a separate specification).
- Removing `AG`/`OG` (legacy sequences) — requires a separate decision.
- Changing other function keys.
- New macros/sequences.
- Automatically detecting the keyboard layout for shortcut labels (use the physical key and the ЙЦУКЕН mapping table).
- Changing the behavior of the Start page or alignment guides.

---

## Appendix A. Before → After

| Action                     | Before              | After                                                      |
| -------------------------- | ------------------- | ---------------------------------------------------------- |
| Home / Start page          | `Ctrl+D`            | `Ctrl+U`                                                   |
| `Ctrl+D`                   | Home                | Free (BIMCraft default); Move only in the Archicad profile |
| Home (old TS-008 proposal) | `Ctrl+Shift+H`      | `Ctrl+U`                                                   |
| Alignment guides           | `F8`, `F11`         | `F8`                                                       |
| `F11`                      | Alignment guides    | Free                                                       |
| `Ctrl+U` in notes          | Underline           | Underline (unchanged)                                      |
| `Ctrl+Г` on ЙЦУКЕН         | —                   | Home (physical `U`)                                        |
| `Ctrl+D` on ЙЦУКЕН         | Home (physical `D`) | Free                                                       |

---

## Appendix B. Implementation Checklist

- [x] Removed the hardcoded `Ctrl+D → toggle_home()` check.
- [x] Added the `Ctrl+U → toggle_home()` check.
- [x] `Ctrl+U` does not trigger in a text field.
- [x] `Ctrl+U` in notes remains Underline.
- [x] `Ctrl+D` is free by default.
- [ ] `Ctrl+D` can be assigned to Move (TS-008). — Not implemented: modifier-key assignment will be available with TS-008.
- [x] Removed `F11` from the condition.
- [x] `F8` remains the only function key for alignment guides.
- [x] `F11` is free by default.
- [ ] `F11` can be assigned to another action (TS-008). — Not implemented: modifier-key assignment will be available with TS-008.
- [x] Updated `start.rs`, `tests/start.rs`, and `tests.rs`.
- [x] Updated README, docs, and `.md` help.
- [ ] Updated default profiles and migration (if TS-008 is implemented). — Not applicable: TS-008 has not been implemented, so there is nothing to migrate.
- [ ] Displayed the one-time notification. — Deferred until TS-008: without a binding registry, the notification would promise that `Ctrl+D` can be assigned, although that functionality is not available yet.
- [x] Tests from section 7 pass.
- [x] `cargo fmt`, `clippy`, and `test --workspace` pass.
- [x] No regressions in the Start page, notes, or alignment guides.
