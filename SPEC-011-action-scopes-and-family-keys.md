# SPEC-011. Action scopes and Family Editor keys on the shared registry

Status: **planned** (not started)
Date: 2026-11-13
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: discussion in [IPedrax/BIMCraft#403 "Add configurable keyboard shortcuts with multiple bindings"](https://github.com/IPedrax/BIMCraft/pull/403): emilianocapasso's comment of 2026-11-12 (how the family-editor keys from #311 fit into the registry, two rules, the suggestion of `Category::Family` or a scope) and dronov-dmitry's reply (option 2: #403 stays project-scope, family keys move in a follow-up; `Category::Family` rejected; the full list of required changes)
Related documents: `SPEC-008-User-Configurable-Keyboard-Shortcuts.md`, `ТЗ-008-горячие-клавиши.md`, `ТЗ-009-освобождение-ctrl-d-и-f8.md`, `BIMCraft/docs/areas/shortcuts.md`, `BIMCraft/docs/areas/families.md`
Task type: new functionality (action registry, input, Find, settings format), moving existing behaviour onto the shared pipeline, tests.

> **In brief.** PR #403 built an action registry with a global "one chord → one action" mapping and a physical input pipeline, but left it project-scope: the Family Editor tab has its own input (`family_typed` on `Event::Text` with hardcoded `RP`/`TR`/`DI`/`AL` and `family_keys` with hardcoded `Ctrl+S`/`Ctrl+Shift+S`/`Ctrl+K`), the shared `keys()` pipeline is disabled on the family tab, and Find shows a separate allow-list `family_find_actions`. The family-editor keys (WP/WW, FA, AN, PL, FG/CG, DD, AA, ZX from follow-up #311 and RP/TR/DI/AL from the current code) need to move onto the shared registry — for that the registry gains a **scope** `ActionScope { Project, Family, Both }`: bindings are keyed by `(Scope, Binding)`, conflicts are checked only within a single scope, `execute_action` and Find become scope-aware, the v2 settings format gains an optional `"scope"` field, and family input moves onto the same physical pipeline as #403.

## 0. General working rules

1. Communication language — Russian; code, comments, code strings, console messages — English.
2. Tests (`cargo test` and any other runs of test code) are **not run** in this task (rule from `~/.config/opencode/rust-rules.md`): verification is `cargo build`, `cargo check`, `cargo clippy` (without `--all-targets`) and CI in the PR. At the end of the task — `bash scripts/debug.sh --non-interactive`.
3. Commits are atomic: one commit — one logically complete change; every commit builds and does not break existing behaviour.
4. Documentation and code comments — in English; this file — in Russian.
5. Action ids, field and type names, commands, and file names are not translated.

## 1. The essence of the problem

### 1.1. What exists now (per the code)

The registry from PR #403 (`crates/ui-egui/src/shortcuts.rs`):

- `ActionDef` (~189) has the fields `id`, `label`, `category`, `kind`, `defaults`, `menu`, `legacy` — **there is no scope field**;
- `Category` (~96) is a group in the dialog and in the Markdown help (File, Edit, Architecture, …): belonging to a category does **not** say in which document the action works;
- bindings are indexed globally: `Shortcuts::by_chord`, `by_sequence`, `by_sequence3` (~1222), the "one chord — one action" conflict is checked globally (`check_assignable` ~1089, `owner` in `parse_shortcuts` ~1644);
- `execute_action` (~2140) and `execute_shortcut_action` (~2251) do not know in which document (project or family) the action is executed;
- the v2 settings format (`Config` ~1146) — `{"version": 2, "profile": …, "bindings": [{"action": …, "keys": […]}]}`, there is no `"scope"` field.

Input on the family tab (`crates/ui-egui/src/lib.rs`, `crates/ui-egui/src/editor/`):

- `keys()` (`lib.rs` ~2186) — the physical pipeline from #403 (reading `Event::Key { physical_key }`, chords, two-/three-key sequences) — **returns early on the family tab**: `if self.keys_taken(ctx) || self.active_family().is_some() { return }` (`lib.rs` ~2197);
- `family_keys` (`editor/document.rs` ~1517) handles input separately and **hardcoded**: `Ctrl+S` → `family_save(false)`, `Ctrl+Shift+S` → `family_save(true)`, `Ctrl+K` → open Find, plus Escape/Delete/`Ctrl+Z` for sketch mode;
- `family_typed` (`editor/temp_hide.rs` ~243) reads **`Event::Text`** (letters as text, not physical keys), accumulates a two-letter buffer and **hardcodes** `RP` (Reference Plane, only `editing_2d()`), `TR`, `DI`/`AL`; HH/HC/HI/IC/HR (Temporary Hide/Isolate) already go through `action_for_text` — that is, through the registry's global sequences;
- `family_keys` is called before/instead of the shared `keys()` (`lib.rs` ~1871), and the processing order is set by hand.

Find (`crates/ui-egui/src/find.rs`):

- `find_matches` (~62) on the family tab goes entirely to `family_find_matches` (~111);
- `family_find_matches` is built from the allow-list of strings `family_find_actions` (`editor/document.rs` ~1437): `Delete`, `Aligned Dimension`/`Align` (only `editing_2d()` without sketch), Temporary Hide/Isolate (5 actions), `3D View`, `Zoom to Fit` (only `Sheet::Family(_, None)`), `Properties`, `Project Browser`, `Keyboard Shortcuts`; **bindings (keys) are not shown next to them** (the second element of the tuple is always `None`);
- `family_find_action` (~1459) executes the selected action through a separate manual path, not through `execute_action`.

### 1.2. Scenarios and expected behaviour

1. **Family editor keys work from the registry.** The user opens a family, types `WP` (or `WW`) — Work Plane opens, `FA` — Load Family, `AN` — Angular Dimension, `PL` — Pick Lines, `FG`/`CG` — Finish/Cancel Edit Mode; `RP`/`TR`/`DI`/`AL` do the same as today, but are read by physical keys and edited in the Keyboard Shortcuts dialog like any others.
2. **The same key in different scopes — different commands.** `PL` on the project tab may mean one thing, and in a family another; they do not conflict, because they never work in the same document.
3. **A shared key — the same command.** If a chord exists both in the project and in the family, it is the same action (the rule from emilianocapasso's comment).
4. **Family-only commands stay silent on the project tab** and are not shown in Find on the project tab (the second rule from the comment).
5. **Find is scope-aware:** on the project tab — actions of scopes `Project`/`Both`, on the family tab — `Family`/`Both`; the shown actions display their current keys (including user ones).
6. **Physical input in the family:** `Ctrl+D` works as `Ctrl+В` on the Russian layout, modifiers are compared exactly, key repeat is ignored — as in the project after #403.
7. **Settings migrate without loss:** the existing v2 config without `"scope"` is read as before; family actions in the file are marked with a scope; old builds that do not know `ui.family.*` ignore such records rather than failing the import.

### 1.3. Why this matters

- Family editor keys are today written by two inconsistent mechanisms (`Event::Text` + hardcode and the registry's global sequences): they cannot be configured through Keyboard Shortcuts, they do not read physical keys, and they diverge from the rest of the application.
- Without a scope, the registry cannot represent actions that exist only in one document type: either collisions (a family `PL` versus a project `PL`) or dead entries in the dialog.
- The rules from the #403 discussion (a shared key = one command; family-only stays silent in the project and is not visible in Find) must be fixed in the code, not in an agreement.

## 2. Terms

| Term | Meaning |
|---|---|
| Scope | The area of action: project, family, or both. A property of behaviour (`ActionScope`), not of presentation |
| `ActionScope` | The new enum `{ Project, Family, Both }` on `ActionDef` |
| Action | A registry entry `ActionDef` with an id of the form `cmd.*`, `ui.*`, `legacy.*` |
| Binding | A chord (`Ctrl+D`), a two- or three-key sequence (`WA`, `BMM`) |
| Scope key | The pair `(Scope, Binding)` by which bindings are indexed after the change |
| Physical pipeline | Reading input through `Event::Key { physical_key }` (as in `keys()` after #403) |
| Family tab | An open family document: `BimApp::active_family().is_some()` / `Sheet::Family` |
| Find | The `Ctrl+K` window, a list of commands/views by substring (`find.rs`) |
| v2 format | The text/JSON format of `settings.json` with `version = 2` from SPEC-008 |
| Follow-up #311 | emilianocapasso's unmerged branch with the family-key table (WP/WW, FA, AN, PL, FG/CG, DD, AA, ZX) |

## 3. Functional requirements

### 3.1. F1. `ActionScope` on `ActionDef`

- In `shortcuts.rs` add `pub enum ActionScope { Project, Family, Both }`.
- `ActionDef` gains the field `scope: ActionScope`.
- Existing actions get `Project` or `Both`; `Both` — those that work in both documents and whose ids are already shared: `Delete`, `Align`, Temporary Hide/Isolate (`HH`/`HC`/`HI`/`IC`/`HR`), `Properties`, `Project Browser`, `Keyboard Shortcuts`, Find, Save/Save As, 3D View, Zoom to Fit and the like (the exact list — by fact: everything listed in `family_find_actions` that works on the family tab today stays `Both`).
- Do not mix `Category` and `ActionScope`: the category is responsible for **where the action is shown** in the dialog and help; the scope — for **where it works**. A single family action may live in the Architecture, Annotate, or Palettes category.

### 3.2. F2. Bindings are keyed by `(Scope, Binding)`

- The indexes `by_chord`/`by_sequence`/`by_sequence3` (and, mirroring them, `binding_owner`) are keyed by the pair `(Scope, Binding)` or split into scope sets — implementation is a choice, but the behaviour is:
  - the same chord may be assigned to different actions in `Project` and in `Family`;
  - within a single scope the chord is still unique (error/warning as today);
  - the `Both` scope participates in both sets: a conflict of a `Both` action is checked against both `Project` and `Family` in their sets.
- Conflict checking (`check_assignable`, duplicates in `parse_shortcuts`, "first owner wins" in the dialog) becomes scope-aware with exactly the same logic.

### 3.3. F3. Family editor keys — registry entries

- New actions with ids of the form `ui.family.*` (the suggestion from the #403 discussion), for example: `ui.family.work_plane`, `ui.family.load_family`, `ui.family.angular_dimension`, `ui.family.pick_lines`, `ui.family.finish_edit_mode`, `ui.family.cancel_edit_mode`, plus actions for `DD`, `AA`, `ZX` (the exact composition and labels — per the follow-up #311 table; see §10, open question 1).
- The hardcode `RP`/`TR`/`DI`/`AL` from `family_typed` moves into the registry: the default sequences are the same, the availability conditions (for example, `RP` only in the family plan/elevation — `editing_2d()`) move into the action's execution condition, not into text parsing.
- Default sequences of family actions: `WP`/`WW`, `FA`, `AN`, `PL`, `FG`/`CG`, `DD`, `AA`, `ZX`, `RP`, `TR`, `DI`, `AL` — as in Revit and in the current code.
- All these actions are `ActionScope::Family`; they are visible in the Keyboard Shortcuts dialog (with a scope marker or filter — see F8).

### 3.4. F4. Scope-aware `execute_action`

- `execute_action` / `execute_shortcut_action` receive the current scope (based on `active_family()`: `Family`, otherwise `Project`) and:
  - execute actions of `Both` and of the current scope;
  - **silently skip** actions of a foreign scope (family-only on the project tab does nothing; project-only on the family tab — likewise, unless the decision in §10, open question 2, is different).
- A chord/sequence not found in the current scope set does not "fall through" into a foreign set.

### 3.5. F5. Scope-aware Find

- `find_matches` on the project tab shows actions of `Project`/`Both` (+ views, plugins, etc., as now), on the family tab — `Family`/`Both` (+ family views, as now).
- The allow-list `family_find_actions` and the manual path `family_find_action` are replaced by filtering the registry by scope (preserving current conditions like `editing_2d()` and `Sheet::Family(_, None)` — they move into action availability).
- Every shown action in Find displays its current keys (`hints`), including user ones — as the project branch of `find_matches` already does.
- The rule "family-only is not visible in the project Find" is satisfied automatically by the scope filter.

### 3.6. F6. Physical family input instead of `Event::Text`

- `keys()` on the family tab no longer returns early: chords and sequences are read by the same physical code as in the project (`Event::Key { physical_key }`, exact modifier comparison, no key repeat).
- `family_typed` (`Event::Text`) is removed; two-letter/three-letter family input goes through the same sequence buffer as in the project, but in the `Family` scope set.
- `family_keys` loses the hardcode `Ctrl+S`/`Ctrl+Shift+S`/`Ctrl+K`: these are registry actions (Save, Save As, Find — of the `Both` scope), executed by the shared `execute_action`; Escape/Delete/`Ctrl+Z` in sketch mode remain document-specific logic (outside the registry) — see §9.
- Russian layout: a physical `В` in place of `D` yields `Ctrl+D`, as in the project after #403.

### 3.7. F7. Optional `"scope"` in the v2 format

- The `bindings` entry in JSON (`Config::to_json`/`from_json`, `serialize_shortcuts`, `parse_shortcuts`/`parse_json_file`) gains an optional field `"scope"`: `"project"` | `"family"` | `"both"` (default — the value from the registry, absence of the field = behaviour as now).
- Old builds that do not know `ui.family.*`, when importing a file with family records, **ignore** these records (do not treat the file as erroneous and do not roll back the import entirely); for this, an unknown `ui.family.*` id in the import is a warning, not an error (see §10, open question 3).
- Migration: existing `settings.json` without `"scope"` are read without changes and without warnings.

### 3.8. F8. Keyboard Shortcuts dialog

- Family actions are shown in the dialog (categories do not change) with a visible scope — a column, an icon, or a "Project / Family / All" filter; implementation is a choice, the criterion is that the user distinguishes a family action from a project one.
- Editing family chords in the dialog goes through scope-aware conflict checking (F2): a change in one scope does not touch the other.
- The Markdown help (Export → Markdown help) lists actions taking the scope into account.

### 3.9. F9. Change map

| What | File | Where/what now |
|---|---|---|
| `ActionScope`, `scope` field on `ActionDef` | `crates/ui-egui/src/shortcuts.rs` | `ActionDef` ~189, `Category` ~96 |
| Scope indexes of bindings, scope-aware `check_assignable`/`binding_owner` | `crates/ui-egui/src/shortcuts.rs` | `Shortcuts` ~1222, `check_assignable` ~1089 |
| Scope-aware `execute_action` | `crates/ui-egui/src/shortcuts.rs` | `execute_action` ~2140, `execute_shortcut_action` ~2251 |
| `"scope"` in v2: writing/reading/serialization | `crates/ui-egui/src/shortcuts.rs` | `Config` ~1146, `to_json`/`from_json` ~1158/1171, `serialize_shortcuts` ~1780, `parse_shortcuts` ~1633 |
| Registry of family actions `ui.family.*` and their defaults | `crates/ui-egui/src/shortcuts.rs` | `build_registry`/`SHORTCUTS` (`lib.rs` ~383) |
| `keys()` works on the family tab | `crates/ui-egui/src/lib.rs` | early `return` ~2197 |
| Removal of the `Ctrl+S`/`Ctrl+Shift+S`/`Ctrl+K` hardcode | `crates/ui-egui/src/editor/document.rs` | `family_keys` ~1517 |
| Removal of `family_typed` (`Event::Text`) and the `RP`/`TR`/`DI`/`AL` hardcode | `crates/ui-egui/src/editor/temp_hide.rs` | `family_typed` ~243 |
| Find by scope instead of the allow-list | `crates/ui-egui/src/find.rs` | `find_matches` ~62, `family_find_matches` ~111 |
| Replacement of `family_find_actions`/`family_find_action` | `crates/ui-egui/src/editor/document.rs` | ~1437, ~1459 |
| Dialog: scope, scope-aware conflicts, help | `crates/ui-egui/src/shortcuts.rs` | `ShortcutsDialog` ~2013 |

## 4. Pitfalls

1. **`Category` ≠ scope.** The attempt to express scope via a category (`Category::Family`) was rejected in the discussion: a category is presentation (dialog, help), a scope is behaviour (execution, Find, conflicts). A family action may live in any category.
2. **Global uniqueness is baked into three places:** the `Shortcuts` indexes, `check_assignable`, and the `owner` map in `parse_shortcuts` ("already assigned to …"), plus "first owner" in the dialog. All three must become scope-aware at the same time, otherwise import and the dialog will start arguing with the executors.
3. **`Both` actions create an intersection of sets:** a conflict of a `Both` chord must be checked in both sets, otherwise the user can reassign `Delete` in the family and silently break the project (or vice versa).
4. **Input processing order:** today `family_keys` is called separately (`lib.rs` ~1871) and lives by its own rules (`ed.modal()`, `keys_taken`, menus, `text_edit_focused`). After enabling `keys()` on the family tab, the priorities must be preserved: modal windows and input fields take keys before the registry, the sketch handling of Escape/Delete/`Ctrl+Z` — after (and stays outside the registry).
5. **`Event::Text` vs physical keys:** `family_typed` today accepts input from the IME and any layouts "as text". The physical pipeline must correctly handle non-Latin letters (the #403 example: `Ctrl+В` = `Ctrl+D`), otherwise family sequences on the Russian layout will break.
6. **Availability conditions move from text parsing into execution:** `RP` works only under `editing_2d()`, `Aligned Dimension`/`Align` — in plan/elevation without sketch, `Zoom to Fit` — on `Sheet::Family(_, None)`. If these conditions are left in parsing, the scope will not help; if removed without moving them, the action becomes available where it should not be.
7. **Settings migration:** v2 files without `"scope"` and legacy v1 objects must be read without warnings; old builds reading the new file — ignore `ui.family.*` rather than fail on "Unknown action".
8. **The #403 tests and the family tests** rely on the current behaviour: `align.rs` (`AL`), `family_temporary_hide.rs` (HH/HC/HI/IC/HR), `family_tabs.rs` (Find on the family tab), `family_document.rs`. The migration must preserve them (with expected edits only where the behaviour intentionally changes).
9. **`action_for_text` is also used in tests** (`lib.rs` ~2354, `#[cfg(test)] shortcut`): after the move to scope sets, this path must remain functional for the family-action tests.

## 5. Tests

> Tests in this task are **not run locally** (project rule); they are written and checked by CI in the PR. Below is what must be covered.

### 5.1. Unit tests (`crates/ui-egui/src/shortcuts.rs`)

- Indexing: one chord in `Project` and another in `Family` coexist; in one scope — a conflict (error/warning as now).
- A `Both` action conflicts in both the `Project` and the `Family` sets.
- `execute_action`/`execute_shortcut_action`: a family-only action on the project tab is not executed (returns `false`/does nothing); on the family tab it is executed; a project-only one — mirrored; `Both` — in both.
- v2: a record with `"scope": "family"` is written and read; without the field — read as before; a legacy file without scope does not change behaviour.
- Import: an unknown `ui.family.*` — a warning, not an error (if the §10.3 decision — and vice versa, the test fixes the adopted decision).

### 5.2. UI tests (`crates/ui-egui/src/tests/…`, `egui_kittest`)

- Family: typing `WP`/`RP`/`FG` with physical keys launches the corresponding action; on the Russian layout/via `physical_key` — as well.
- Project: a family-only chord (for example, the family default `PL`, if it is not taken by the project) does nothing.
- Find: on the project tab there are no family-only actions; on the family tab there are `ui.family.*` and `Both`, and all of them show their keys.
- Dialog: a family action is visible, its chord can be changed, the conflict is checked within the scope.
- Save/Save As/Find on the family tab work through the registry (`Ctrl+S`, `Ctrl+Shift+S`, `Ctrl+K`).
- Existing: `family_temporary_hide.rs`, `family_tabs.rs`, `align.rs`, `family_document.rs`, the #403 `shortcuts` tests — without unplanned changes to expectations.

### 5.3. Quality control

- `cargo check`, `cargo build`, `cargo clippy -p bimcraft-ui-egui` (without `--all-targets`) — clean.
- `bash scripts/debug.sh --non-interactive` — the debug binaries build.
- CI in the PR — green.

## 6. Documentation and translations

- `BIMCraft/docs/areas/shortcuts.md`: scope, family actions, the "shared key — one command" rule, the format of the `"scope"` field.
- `BIMCraft/docs/areas/families.md`: family editor keys listed as registry actions, edited in Keyboard Shortcuts.
- `SPEC-008-User-Configurable-Keyboard-Shortcuts.md`: a section about scope and the v2 `"scope"` field.
- `README.md` (the Shortcuts section): one sentence about scopes.
- New strings (`msg!`/`t()`) — into the translation templates; update the existing `.po` catalogs.

## 7. Acceptance criteria

1. `ActionScope` exists, every registry action has a scope, no field is "dead" (everything is read by execution, Find, or the dialog).
2. The keys WP/WW, FA, AN, PL, FG/CG, DD, AA, ZX, RP, TR, DI, AL are executed on the family tab through the registry; `family_typed` and the hardcode `RP`/`TR`/`DI`/`AL`/`Ctrl+S`/`Ctrl+Shift+S`/`Ctrl+K` are removed.
3. The physical pipeline also works on the family tab: exact modifiers, repeat ignored, `Ctrl+В` = `Ctrl+D`.
4. On the project tab family-only actions are not executed and not visible in Find; on the family tab project-only ones — likewise.
5. One chord may mean different actions in `Project` and `Family`; conflicts are checked only within a scope (and for `Both` — in both).
6. Find is scope-aware and shows the actions' keys.
7. `"scope"` is read and written in v2; old files are read without changes; old builds ignore `ui.family.*`.
8. The existing family and shortcut tests are green in CI; the new tests from §5 are present.
9. Documentation and translations are updated (§6).

## 8. How to verify manually

1. Open a family → type `RP` in the plan: the Reference Plane tool launches; the same with `WP`, `FG`/`CG` in sketch mode.
2. In the same family `Ctrl+S` saves, `Ctrl+Shift+S` — Save As, `Ctrl+K` opens Find; all three — through the Keyboard Shortcuts dialog (check that the action is highlighted).
3. In the project type a family-only sequence — nothing happens; in the project Find there are no family-only actions.
4. Switch the layout to Russian, press `Ctrl+В` in the project — the action on `D` fires; in the family — the same for family sequences.
5. In the Keyboard Shortcuts dialog assign, in the Project scope, a chord taken in Family: no conflict; in the same scope — there is a conflict.
6. Save the settings, open `settings.json`: family actions have `"scope"`; delete the field — the file is still read.
7. `Ctrl+K` in the family: the shown actions display their keys.

## 9. Out of scope

- Scope in PR #403 itself (per the discussion: #403 stays the project-scope foundation; the scope field is not added there as a "dead" one).
- Import/export of Revit shortcuts (file limit, `<ShortcutItem>`, several elements per line, `Paths`, XML entities) — **a separate SPEC-012**.
- `Category::Family` and any expression of behaviour through a category.
- Changing the composition or behaviour of the #403 project keys.
- Scope for MCP/CLI commands (core `cmd.*`): if needed — a separate task.
- Moving the sketch-mode Escape/Delete/`Ctrl+Z` into the registry (document-specific logic; may be a separate task).

## 10. Adopted decisions and open questions

**Adopted (the #403 discussion):**

1. Option 2: #403 — project scope; family keys — a follow-up in one focused PR; Revit import — a separate small PR (SPEC-012).
2. `Category::Family` rejected: "Category is presentation; scope is behaviour".
3. `ActionScope { Project, Family, Both }` on `ActionDef`; bindings by `(Scope, Binding)`; conflicts within a scope; scope-aware `execute_action` and Find; optional `"scope"` in v2; family input through the physical pipeline instead of `Event::Text` and hardcode.
4. Shared ids (`Delete`, `Align`, Temporary Hide, `Properties`, `Keyboard Shortcuts`) stay shared (`Both`); family-only commands become `ui.family.*`.
5. Action ids in #403 stay stable so that the family table moves over without renames.

**Open questions:**

1. **The full composition of the family table** (what exactly `DD`, `AA`, `ZX` mean, the exact labels and defaults of all actions) — from emilianocapasso's follow-up (#311); his branch is not in this clone; clarify with the author or port from his branch when work starts.
2. **What a project-only action does on the family tab** (apart from family-only in the project, the rule is not fixed in the comment): silently not execute (the default option in F4) — to be confirmed.
3. **An unknown `ui.family.*` during import in old builds:** currently `resolve_action` gives the "Unknown action" error, while the requirement is "old builds ignore". Whether the import should be changed so that unknown `ui.family.*` are a warning (and do not break the import of files with family records for old builds) — to be confirmed with the discussion author.
4. **Showing the scope in the dialog:** a column, an icon, or a filter — to be chosen during implementation (the criterion is F8).

## 11. Work plan (atomic commits)

1. **`ActionScope` and scope indexes of bindings.** Enum + field on `ActionDef` (all existing actions get `Project`/`Both`), scope keys of the indexes, scope-aware `check_assignable`/conflicts/`binding_owner`. No UI and input: project behaviour does not change, unit-level tests.
2. **Scope-aware `execute_action` and `"scope"` in v2.** The current scope in the executor, skipping foreign actions, writing/reading `"scope"` in `Config`, serialization/parsing, migration without warnings.
3. **Family actions in the registry.** `ui.family.*` with defaults (WP/WW, FA, AN, PL, FG/CG, DD, AA, ZX, RP, TR, DI, AL), moving availability conditions into execution.
4. **Physical family input.** Enabling `keys()` on the family tab, removing `family_typed` and the `RP`/`TR`/`DI`/`AL` and `Ctrl+S`/`Ctrl+Shift+S`/`Ctrl+K` hardcode from `family_keys`, preserving priorities (modals, input fields, sketch).
5. **Scope-aware Find.** Filtering the registry by scope instead of `family_find_actions`/`family_find_action`, showing keys.
6. **Dialog and help.** Showing the scope, scope-aware editing, Markdown help.
7. **Documentation and translations** (§6) + UI tests (§5.2).

Every commit builds (`cargo check`) and does not break existing behaviour; tests are written together with the commit they belong to and are run by CI.

## 12. Scenario traceability

| # | Scenario | Requirement | Test |
|---|---|---|---|
| 1 | `WP`/`RP`/`FG` in a family work and are edited in the dialog | F3, F6 | 5.2 (family input), 5.1 (defaults) |
| 2 | The project `PL` and the family `PL` do not conflict | F2 | 5.1 (indexing) |
| 3 | A shared key = one command in both documents | F1 (`Both`), F4 | 5.1 (`execute`) |
| 4 | A family-only chord in the project stays silent and is not visible in Find | F4, F5 | 5.1, 5.2 (Find) |
| 5 | `Ctrl+В` = `Ctrl+D` in the family | F6 | 5.2 (physical input) |
| 6 | `Ctrl+S`/`Ctrl+Shift+S`/`Ctrl+K` in the family through the registry | F6 | 5.2 |
| 7 | Find in the family shows `ui.family.*`/`Both` with keys | F5 | 5.2 |
| 8 | A settings file with `"scope"` is written/read; an old one is read | F7 | 5.1 (v2) |
| 9 | An old build ignores `ui.family.*` during import | F7 | 5.1 (import) |
| 10 | A family action cannot be assigned into a foreign scope with a conflict | F2, F8 | 5.1, 5.2 (dialog) |
