# SPEC-034. F2 — renaming an item in the Project Browser

Status: **implemented** (PR #664, branch `feat/project-browser-rename-f2`); the project is kept below, actual refinements — §15
Date: 2026-11-22; implemented 2026-11-28
Project: BIMCraft 028 (Rust; `crates/ui-egui/src/browser.rs` — `Act` :90, `Node` :127,
`browser` :332, `browser_node` :433, `views_node` :569, `schedules_node` :673,
`families_node` :732; `crates/ui-egui/src/lib.rs` — `Sheet` :151, `SCHEDULES` :441,
`view_title` :2244, `keys` :2647; `crates/core/src/model.rs` — `Schedule` :1727,
`PlanView` :1756, `DetailView` :1783, `View3D` :1831, `Section` :2499, `Sheet` :2569;
`crates/core/src/commands.rs` — `sheet` :1472, `schedule` :2998, `rename_type` :2715,
`family` :17100, `modify` :18982; `crates/core/src/schedule.rs` — `VIEWS` :127,
`standard` :163)
Source: task 2026-11-22: a view, schedule, sheet or family can be renamed only
with a command or through Properties. The Project Browser tree has no F2 as in Revit: an error in the name
does not show what exactly is wrong, and after a refusal one has to start typing from scratch.
Related: SPEC-008 (keyboard shortcuts), SPEC-026 (file name in the window title), SPEC-030 (tables and legends),
SPEC-003 (families through MCP).
Task type: UI behaviour and one core command; the `.bimc` format is extended additively
(`Schedule.standard`).

> **In short.** The Project Browser is a tree, but its rows only open views and place
> components. F2 does nothing: a view, schedule, sheet and family are renamed only by
> a command (`modify`, `schedule`, `family`, `rename_type`) or through Properties, and built-in
> schedules and `{3D}` are not items at all until they are modified. F2 is needed on the selected
> row: the name is edited **in place**, Enter saves, Esc cancels, an error stays in the row.
>
> **Implemented 2026-11-28.** A single click on a row now only selects it, a view is opened by
> a double click. F2 is intercepted only when a row is selected; without a selection it goes to Keyboard
> Shortcuts (by default `F2 → Floor Plan`). The level's own floor and ceiling plan has become a real
> `PlanView` with its own name, so F2 changes the view name, not the level name. Details and differences — §15.

## 0. General working rules

1. Language — Russian; code and strings — English; window strings — via `msg!`/`t()`/`tf()`.
2. Tests — only on request; CI separately. Allowed: `check`, `build`, `clippy` (without
   `--all-targets`), `fmt --check`, `i18n check`.
3. Commits: the `rename` command → targets in the tree (`Ren`) → F2 and the inline editor → families
   and types → i18n/docs.
4. No new dependencies; `unsafe` is not added.
5. Old projects are read as is: the new `Schedule.standard` has a `default`;
   renaming changes `name`, and the flag only marks the category's standard schedule.

## 1. The problem

| Capability | 028 command | Limitation |
|---|---|---|
| View | `modify {id, name}`, `duplicate_view {name}` | not available from the browser; errors are text in the corner |
| Schedule | `schedule {name, …}` | the name is set on creation; a built-in one cannot be renamed |
| Sheet | `sheet {number, name}` | number and name only on creation; the browser shows rows "A101 — Name" |
| Family | `family {name, rename}` | there is no item in the browser's menu |
| Type | `rename_type {type, name}` | there is no item in the browser's menu |
| Built-in schedule | `schedule::standard` | until modified, there is no item |
| `{3D}` | `default_3d` | until modified, there is no item |
| Undo | each command has its own | there is no single "F2 → Enter → Undo" gesture |

What happens now: the user selects a view, presses F2 — nothing. They go to Properties or
type a command, make a mistake in the name, get a message in the status bar and lose what was typed.
For a sheet one has to remember that F2 should change only the name, and the number — separately. For the
built-in schedule and `{3D}` there is no renaming at all until the view is "materialized".

## 2. Goals

1. F2 on the selected Project Browser row opens in-place name editing.
2. It works in all tabs (`All`, `Views`, `Schedules`, `Sheets`, `Families`) and with
   an active search.
3. Enter saves, Esc cancels, loss of focus saves; an empty or taken name on Enter —
   an error in the row, the editor stays open.
4. Renaming is a single Undo step; labels in the tree, tab titles, view labels on
   sheets and in Properties update by themselves.
5. One core command `rename` for views, schedules, sheets, levels, families and types;
   for families and types it delegates to the existing `family {rename}` and `rename_type`.
6. The built-in schedule and the virtual `{3D}` are materialized on the first real
   rename.
7. A "Rename" item with an `F2` hint in the row's context menu — so the feature is
   discoverable without knowing the key.
8. Tests, i18n and documentation.

## 3. Non-goals

- No renaming of tree groups, built-in categories ("Walls", "Doors"), tabs and
  search: these are not items.
- No editing of a sheet's **number** by F2: F2 changes the name, the number — through Properties or
  the `sheet`/`modify` command.
- No slow double click, no batch renaming and no renaming of several
  selected rows with one command (v1: one row).
- No renaming of linked models (`Linked Models`) — a separate stage if
  needed.
- No renaming of model items from the canvas: only the Project Browser.
- No mandatory migration of old files: the new `Schedule.standard` field is read with
  a `default`, and everything else — the existing `name` fields.

## 4. Key decision

**F2 is an inline editor in a tree row, not a modal dialog; the target is stored as
a stable `Ren`, not as a pointer to a node that is rebuilt every frame.**

| Option | Consequence |
|---|---|
| Modal "Rename" dialog | an extra click, the tree context is lost, unlike Revit |
| A separate panel/Properties | F2 does nothing in place, two different paths |
| Editing only by command | no feedback, errors in the window corner |
| **`TextEdit` in the row + `browser_sel`/`browser_edit`** | expected, fast, one Undo, the error next to it |

The following is added to `browser.rs`:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub(crate) enum Ren {
    Level(Id),
    View(Id),                 // PlanView, Section/Elevation, DetailView, View3D, Schedule, PanelSchedule
    Default3D,
    Sheet(Id),
    BuiltinSchedule(&'static str),
    Family(String),
    Type { category: Option<&'static str>, family: Option<String>, name: String },
}
```

- `Node` gets `rename: Option<Ren>`; it is filled in where the row is built.
- `BimApp` gets `browser_sel: Option<Ren>` (the last selected row) and
  `browser_edit: Option<BrowserEdit>` (`target`, `text`, `was`, `error`, `focus`, `select_all`).
- `Ren` is UI state only, it is never written to the file.
- `Schedule` gets an additive `standard: bool` (`default: false`): this is the standard
  schedule of its category. Without it, a renamed built-in schedule would lose its
  identity, and the tree would show both the virtual row and the new one. The field is serialized as
  `#[serde(default, skip_serializing_if = "std::ops::Not::not")] pub standard: bool`,
  so old files are read without migration.
- The row key for egui is stable (`"level:123"`, `"view:456"`, `"family:Name"`,
  `"type:door:900 x 2100"`), so that rebuilding the tree does not drop the focus.

## 5. Stage A. The `rename` command

`modify {id, name}` can already change the name of a view, schedule, sheet and level. The new command is a
thin wrapper with a unified reply, checks and materialization of virtual rows.

```json
{"id": 812, "rename": "Plan 1"}
{"view": "{3D}", "rename": "Overview"}
{"schedule": "room", "rename": "Rooms"}
{"sheet": "A101", "rename": "Floor Plans"}
{"family": "Door 900", "rename": "Door 900 x 2100"}
{"category": "door", "type": "900 x 2100", "rename": "900 x 2200"}
```

- `rename` — the new name; exactly one selector: `id`, `view`, `schedule`, `sheet`, `family`,
  or `type` with `category` or `family`.
- `id` — a view, schedule, sheet or level (`view_name(id)`/`sheet(id)`).
- `view` — a name or id; `{3D}`/`3D` — a special case.
- `schedule` — a category from `schedule::VIEWS` (built-in) or a project name/id.
- `sheet` — a number, id or name (`sheet_numbered`).
- `family` — a family; `family` + `type` or `category` + `type` — a type.
- The reply is normalized: `{kind, id?, from, to, created?}`; `created: true` — the built-in
  schedule or `{3D}` was materialized. The replies of `family`/`rename_type` are brought to it.
- Registration — `edit("rename", "Rename", &[], …)`: without a two-letter
  alias, F2 is a separate fixed key.

Checks:

1. `rename` is trimmed; an empty name, spaces only, and a name longer than `MAX_TEXT` (:65) — rejected.
2. The name did not change — success without an Undo step.
3. Views, schedules, panel schedules, 3D views, sections, elevations and callouts:
   the name is unique among views (`Model::check`, :10102/:10135/:10184/:10209); the names `{3D}` and
   `3D` are not given to other views: the commands understand them as the 3D view.
4. Level: the name is unique among levels (:9482); a floor plan cannot be named like
   a level (:10118). Renaming a level changes both its floor plan and ceiling plan.
5. Sheet: only `Sheet.name` changes; `Sheet.number` stays. An empty name — rejected; a name
   equal to another sheet's number — rejected, so that `sheet_numbered` (:7572) does not become
   ambiguous. Equal sheet names are allowed by the model.
6. A built-in schedule cannot be given the name of a built-in schedule of **another** category:
   `schedule::named` (:152) would otherwise start finding it as a foreign one. `standard: true` is allowed
   only for a category from `VIEWS`; a second standard schedule of the same category is not
   let through by `Model::check`. While a category has a `standard`, its built-in name is taken:
   a second schedule with it is not created.
7. Family/type: the rules of the existing commands (`family`, `rename_type`); `rename` calls
   them and brings the reply to the common form.
8. The final judge is `Model::check()`; `Session::execute` rolls the model back on error
   (`engine.rs` :653–668), so a refusal does not leave a half-renamed project.

Materialization:

- `schedule` points to a built-in one that does not exist yet: `schedule::standard` (:163) gives
  a template, the command inserts `Element::Schedule(Schedule { name: new, standard: true, ..sc })`,
  `created: true`. `schedule::standard` looks up the standard schedule by `of` and the flag, and
  old files without the flag — by the built-in name, as now; `schedule::named` by the
  built-in name also first takes the category's `standard` schedule. Otherwise, after
  "Door Schedule" → "Doors" the tree would show both rows.
- `view: "{3D}"` without an item: `View3D { name: new, ..View3D::default_view() }` is inserted,
  `created: true`; the UI opens the new id in the 3D tab.
- Existing items are renamed through `modify {id, name}`; one rename —
  one Undo step.
- The standard schedule's identity: `standard: true`, and for old projects — the built-in
  name. `rename` sets the flag if it renames the built-in; `schedule`, creating
  a project schedule instead of the built-in, does too.

## 6. Stage B. The browser: selection, F2 and the editor

### 6.1. Row selection

- A click on any row (a leaf or a family header) remembers `browser_sel` from
  `Node.rename`; a click on a row without `Ren` (a group, category, link) clears `browser_sel`.
  A single click only selects the row; a view or sheet is opened by a double click (§15.2).
- Right now `browser_node` takes `&self` and defers the action into `Act`; the selected `Ren`
  is returned next to `Act` (or `browser_node` gets `&mut self`), so that `browser()`
  records `browser_sel` without a second pass over the tree.
- A click on the empty space under the tree clears `browser_sel`; the palette catches a background click
  after the rows and does not count a row click as one.
- Before F2 and drawing, `browser_sel` is checked against the model; a target that has disappeared (Undo,
  deletion, project close) clears the selection.
- `browser_sel` is a separate state, not `browser_picked`: `browser_picked` collects views and
  sheets for printing and knows nothing about families and types.
- The selected row is marked with a 2 px accent stripe on the left; the active view/type keeps
  its usual fill. This matters for a family: a click on the header both collapses the group
  and selects it, and F2 must be understandable.
- `browser_rect` is remembered when the palette is drawn; `browser_focused` is set when clicking
  inside the browser and cleared by clicking outside it or by hiding the palette.
- If the browser is hidden, `browser_sel` is empty or a dialog is open, F2 renames nothing;
  the status bar shows a short hint. Multi-selection for printing does not interfere: F2 edits
  the row with the accent stripe.

### 6.2. F2

The handling is in `BimApp::keys`, before shortcut parsing (otherwise `F2 → Floor Plan` swallows the key),
but after `ctx.egui_wants_keyboard_input()`. The implemented check:

```rust
let picked = self.browser_sel.as_ref().is_some_and(|ren| self.ren_exists(ren));
if self.browser_visible()
    && self.browser_focused
    && picked
    && !ctx.text_edit_focused()
    && pressed(Modifiers::NONE, Key::F2)
{
    self.begin_browser_rename(None);
    return;
}
```

- `keys_taken` already excludes dialogs, Find and open menus.
- If the focus is in the browser search or another `TextEdit`, F2 is not intercepted.
- **Without a selected row F2 is not consumed** and the Keyboard Shortcuts binding fires; this is
  a difference from the original project (§15.2).
- A second F2 with the editor open selects the whole name, as in Explorer, instead of opening
  a second editor.
- F2 is not part of `shortcuts::ACTIONS` or the two-letter Ctrl shortcuts: it is a fixed
  function key, like F8.

### 6.3. The inline editor

- Instead of the row label, a `TextEdit::singleline` is drawn with the same indent and font; the width —
  the available row. For a sheet the field contains only the name, without "A101 — ".
- When opened, the text is selected entirely; the focus is requested once through a stable id.
  The id is constant, like the search's (`browser-find`): the palette may be drawn twice with
  dock-float, but there must be only one editor.
- Enter — save; Esc — cancel; loss of focus — save if the name is accepted, otherwise
  cancel. If the name is empty or taken, Enter leaves the editor open and shows
  the reason; loss of focus cancels the edit and writes the reason to the status bar (the user
  is not locked in the field).
- Enter on an error does not close the editor; the text stays, the cursor is at the end, so as
  to fix it rather than lose what was typed.
- While the editor is open, a click on another row first finishes the edit, then performs
  the row's usual action.
- Search does not interfere: after a successful rename the tree is simply re-filtered; if
  the new row no longer matches the query, the editor closes.
- If the target has disappeared (Undo, external change, project close), the editor is cancelled without
  a command.
- Success: the editor closes, `browser_sel` points to the new target (for materialized ones —
  to the new id), the status bar is cleared.
- Error: `self.exec(...)` (not `run`), so that the error text stays with the editor rather than only
  in the window corner.

### 6.4. Context menu

- Every renameable row has "Rename" with `shortcut_text("F2")` as the first menu item.
- The item carries its own `Ren`, so it works on the row that was just right-clicked
  even if `browser_sel` points to another.
- `Act::Rename(Ren)` is handled in `browser()` by calling `begin_browser_rename(ren)`.

## 7. Stage C. Views, schedules and sheets

| Tree row | `Ren` | What changes |
|---|---|---|
| Floor Plans → the level's own plan | `Level(level)` until the first rename | a primary `PlanView` is created, then `View(id)`; the level name does not change (§15) |
| Floor Plans → duplicate/dependent | `View(id)` | the `PlanView` name |
| Ceiling Plans → the own plan | `CeilingLevel(level)` until the first rename | a primary `PlanView` with `ceiling: true`, then `CeilingView(id)` |
| Ceiling Plans → duplicate/dependent | `CeilingView(id)` | the name of the same `PlanView` |
| 3D Views → `{3D}` without an item | `Default3D` | a `View3D` is created with the new name |
| 3D Views → camera | `View(id)` | the `View3D` name |
| Elevations / Sections | `View(id)` | the `Section`/`Elevation` name |
| Callouts | `View(id)` | the `DetailView` name |
| Schedules → built-in | `BuiltinSchedule(cat)` | materialization; one row, under the new name |
| Schedules → own | `View(id)` | the `Schedule` name |
| Panel Schedules | `View(id)` | the `PanelSchedule` name |
| Sheets → "A101 — Name" | `Sheet(id)` | `name` only; `number` does not change |

- A view tab's title, a view label on a sheet and the automatic title block fields
  (`SHEET_NAME`) read the model, so they update by themselves.
- For a built-in schedule, F2 with an unchanged name does not create an item: the text is
  compared first, then the command is called.
- An open `Sheet::Schedule(cat)` tab keeps showing the same
  schedule after the rename: `schedule::standard` finds it by the `standard` flag, not by the old name.
- The seven `SCHEDULES` rows remain one per category: `BimApp::schedules` (:4314) does not
  list the standard (and, for old files, built-in-named) schedule a second
  time.
- Renaming a level does not rename its duplicate plans: they have their own names; if
  the new level name matches the name of such a plan, `Model::check` will refuse.
- For a sheet F2 does not touch the number: this is stated explicitly in the documentation and the hint.

## 8. Stage D. Families and types

- A family row (`Families` → family) gives `Ren::Family`; F2 renames the key in
  `model.families`, the instances, the library records and nested references — `rename`
  delegates this to `family {name, rename}`.
- A type row (system or project) gives `Ren::Type`; F2 calls `rename`, which
  for a type calls `rename_type` with `category` or `family`; instances and the library follow
  the type.
- Built-in categories ("Walls", "Doors", "Curtain Panels") and groupings like "Generic
  Panel" are not renamed: they have no `Ren`.
- After success the UI updates its state:
  - family — the `family` part in `self.component` and in the selected type;
  - type — `self.types[category]` and `self.component`, if it pointed to the old type.
  Otherwise Component would keep placing a non-existent name.
- If a family is open in the Family Editor, F2 in the Project Browser renames the project's
  family; the open document is not renamed and not closed.

## 9. Stage E. Edge cases

- An empty name, spaces only, a length > `MAX_TEXT` (:65) — rejected with a message.
- A name equal to the name of another view/level/family/type — rejected; the editor stays
  open.
- The name `{3D}` or `3D` for a non-3D view — rejected: the commands understand these names as the 3D view.
- A built-in schedule named like another built-in one — rejected (see §5, item 6).
- A renamed built-in schedule remains the only row of its category:
  no virtual copy appears, the open tab is not reset.
- A sheet name equal to another sheet's number — rejected; equal sheet names are allowed.
- A case-only change ("Level 1" → "level 1") is allowed if the model does not see an exact
  duplicate: `Model::check` compares names exactly.
- F2 on a group, category, link, empty space — no editor, with a hint.
- With a multi-selection, F2 edits `browser_sel` — the row with the accent stripe (the last
  clicked); the other selected rows do not change.
- In guest Co-Work and in read-only the command may refuse; the editor shows the error and does not
  change the project.
- Undo after a rename restores the old name; if the editor was open, it is
  cancelled.
- Save/open: the names already live in existing fields, there is no migration.

## 10. Test plan

1. UI: "Level 1" → F2 → "Ground" → Enter: the level name and both rows (floor/ceiling) are the new one;
   one Undo step; Undo returns "Level 1".
2. UI: duplicate plan → F2 → new name; the tab title and the view label on the sheet
   updated.
3. UI: sheet "A101 — Plans" → F2: the field contains only "Plans"; the new name is saved, the number A101;
   the `SHEET_NAME` title block field updated; an empty name and a name equal to another sheet's number
   are rejected.
4. UI: built-in "Door Schedule" → F2 → "Doors" → Enter: a `Schedule` is created, the tree
   shows one "Doors" row (not "Door Schedule" and "Doors" side by side), the open tab
   shows the same table; Undo deletes the created item and returns the row; F2 with the same
   name creates nothing.
5. UI: built-in "Door Schedule" → F2 → "Room Schedule": refused, the editor is open, the project
   is not changed.
6. UI: `{3D}` without an item → F2 → "Model" → Enter: a `View3D` is created, the 3D tab shows
   it; Undo returns `{3D}`.
7. UI: family → F2 → new name: the family key, the instances, the Type Selector; Undo.
8. UI: type → F2 → new name: the instances and `self.types`/`self.component`; Undo.
9. UI: Esc cancels; loss of focus saves; a duplicate leaves the editor; F2 without a selection,
   on a group and with the focus in search — no edit; a second F2 selects everything.
10. UI: right click → "Rename" (F2) on an unselected row renames exactly it.
11. Core: `rename` for a view, level, sheet, schedule, built-in schedule, `{3D}`,
    family and type; replies `{kind, from, to, created?}`; no-op; Undo; rollback on error.
12. Regression: an old project without `Schedule.standard` opens; the tree without F2 is
    unchanged; the flag is written only on the standard schedule.

Core tests — `crates/core/tests/rename.rs`; UI — `crates/ui-egui/src/tests/browser.rs`
(a row click, `h.key_press(Key::F2)`, typing into `TextEdit`, Enter/Esc, Undo).

## 11. i18n and documentation

- Strings: "Rename", "Rename (F2)", "A name can't be empty.", "A name is at most {} characters.",
  "There's already a view named `{}`.", "There's already a level named `{}`.",
  "There's already a family named `{}`.", "There's already a type named `{}`.",
  "This item can't be renamed.", "Select an item in the Project Browser to rename.",
  "`{3D}` and `3D` name the 3D view.", "A sheet's number changes in Properties."
- `docs/areas/views-and-graphics.md` — F2 in the Project Browser, views, sheets, 3D.
- `docs/areas/schedules.md` — built-in schedules and their materialization.
- `docs/areas/families.md` — renaming a family and a type.
- `docs/areas/desktop.md` — F2 among the function keys (next to F8).
- In the Keyboard Shortcuts dialog F2 is not configurable: only the two-letter Ctrl keys are there.

## 12. Acceptance criteria

- [x] F2 on the selected row opens in-place editing; Enter, Esc and loss of focus
      work as described.
- [x] Views, schedules, sheets, levels, families and types are renamed.
- [x] The built-in schedule and the virtual `{3D}` are materialized on modification; the built-in one
      remains one row of the category.
- [x] Errors (empty, taken, reserved) are visible in the row, the project is not changed.
- [x] One Undo step per rename; tree labels, tab titles and title block fields update.
- [x] A "Rename" item with F2 in the context menu.
- [x] Tests §10, i18n, documentation; old projects are read, `.bimc` is extended additively
      (`Schedule.standard`, `PlanView.primary`, `PlanView.ceiling`).

## 13. Risks and questions

| Risk | Answer |
|---|---|
| The tree is rebuilt every frame | the target is `Ren`, the editor id is a stable key, not a node address |
| F2 fires when the focus is on the canvas | `browser_focused` + `browser_rect` + `egui_wants_keyboard_input` |
| A click on a family collapses the group | the selection and the accent stripe are visible; F2 works after the click |
| Materializing the built-in schedule is unexpected | only on a real name change, one Undo |
| The built-in schedule is duplicated after renaming | `Schedule.standard` + `schedule::standard` by `of`/flag; `BimApp::schedules` excludes standard ones |
| The family was renamed, Component places the old one | update `component`/`types` after success |
| The model's error is too technical | pre-checks for common cases, `Model::check` is the last resort |
| The sheet was renamed, the number "got lost" | F2 changes only the name; the number — Properties, this is in the docs |

1. Should F2 on a sheet change the number too (an "A101 / Name" dialog), or is the name enough?
2. Are links (`Linked Models`) needed in this same task or as a separate stage?
3. Is a slow double click needed as a second way, as in Revit?
4. Compare duplicates case-insensitively (as Revit does) or keep the exact comparison
   of `Model::check`?
5. Is F2 needed for the selected canvas item (renaming a type) or only the browser?

## 14. Definition of Done

- [x] The `rename` command with selectors and materialization, one Undo.
- [x] `Schedule.standard` and looking up the standard schedule by category and flag.
- [x] `Ren`, `browser_sel`, `browser_edit`, F2 in `keys`, the selection accent stripe.
- [x] The inline editor: Enter/Esc/loss of focus, errors, a second F2.
- [x] A "Rename" item in the context menu.
- [x] Families and types through the existing commands, updating `component`/`types`.
- [x] Tests §10, i18n, documentation; regression of old projects and of the format.

## 15. Implementation (2026-11-28)

### 15.1. What was done

- Core: one `rename` command for a view, level, the level's own plan, schedule, sheet,
  family and type. Families and types are delegated to `family {rename}` and `rename_type`; the reply
  is normalized to `{kind, id?, from, to, created?}`.
- `Schedule.standard` (`#[serde(default, skip_serializing_if = "Not::not")]`): a built-in
  schedule that has become an item remains a schedule of its category after renaming.
  `schedule::standard_id` looks it up by category and flag, `schedule::named` by the built-in name
  first takes the standard schedule; `BimApp::schedules` does not list the standard one a second time.
- The built-in schedule and the virtual `{3D}` are materialized on the first real name
  change (`created: true`); with an unchanged name no item is created.
- The level's own floor/ceiling plan has become a real `PlanView` (`primary: true`, `ceiling`),
  dependent on the level. It has its own name; `Level.name` does not change on F2.
- Project Browser: a single click selects a row, a double click opens a view or sheet. F2 (or
  "Rename" in the row menu) edits the name in the row: Enter saves, Esc cancels, loss of focus
  saves an accepted name and cancels a refused one; the error stays in the row. The selected row is marked
  with an accent stripe.
- F2 in the focused browser is intercepted before shortcuts only when an existing
  row is selected. Without a selection F2 goes to Keyboard Shortcuts, so the `F2 → Floor Plan` binding works.
- The Floor Plan shortcut itself was fixed: from a 3D view, sheet or schedule it opens the current/already
  open plan, otherwise the first level, instead of `Sheet::Plan(0)` with the message "This project has no
  levels".
- Documentation (`docs/areas/desktop.md`, `views-and-graphics.md`, `schedules.md`,
  `families.md`), i18n (`cargo i18n update`, `cargo i18n check`) and tests were updated.

### 15.2. Differences from the original project

1. **A single click does not open a view.** Otherwise there would be no room for F2 on a view: a click only
   selects the row (and `browser_picked` for printing), a double click opens. This applies to
   all views, sheets and schedules; family types are still placed with a single click.
2. **F2 is not always intercepted by the browser.** Only if the browser has a selected row.
   Without a selection the key remains a shortcut. This was done after testing on a live window: F2 was
   bound to Floor Plan and did not reach renaming.
3. **The level's own plan is a `PlanView`, not the `Level` itself.**
   - On the first rename, `PlanView { primary: true, ceiling, parent: Some(level) }` is created;
     `parent` pointing to the level is needed so that the plan shows the level's annotations, view range and graphics.
   - `Model::primary_plan(level, ceiling)`, `Model::ceiling_view_name(level)`;
     `Model::view_name(level)` returns the primary floor plan's name if it exists.
   - `Model::plan_views(level)` no longer lists the primary plan as a duplicate.
   - `Model::views()` does not list a level that has a primary floor plan: the view has become
     a `PlanView`; `Model::view_named` by the level name still finds that plan.
   - `Model::check`: one primary floor and one primary ceiling per level; a primary plan may
     share its name with its own level, but not with a foreign one; two primary plans of one level with the same
     name are still rejected by the general view-name uniqueness check.
   - `annotation_in` follows the parent chain, so a view dependent on the primary plan also
     sees the level's annotations.
   - Open `Sheet::Plan(level)` / `Sheet::Ceiling(level)` tabs are replaced by the new `PlanView`'s tabs
     on materialization; `browser_picked`, `browser_anchor`, `sheet` and
     `front` are also switched to it. Properties for such a tab edits the view name.
4. **New `Ren` variants.** `Ren::CeilingLevel(id)` and `Ren::CeilingView(id)` separate the
   Floor Plans and Ceiling Plans rows. Without this, one `Ren` opened two `TextEdit`s with one id: egui
   drew the red "First use of widget" and the name was edited in two rows at once. The editor id
   now depends on the row and the drawing place and is stored in `BrowserEdit`.
5. **The `.bimc` format is extended additively** not only with `Schedule.standard`, but also with
   `PlanView.primary` / `PlanView.ceiling` (`serde(default)`, not written when `false`). Old
   files are read: before the first rename, a level's plan rows are virtual and named after the
   level; F2 with a new name materializes the view.
6. **The sheet number** is still changed only through Properties/`modify`; F2 changes only the name,
   and this is stated as a hint in the sheet row's menu.
7. **A double click** is used to open a view, not to rename; a slow double
   click as a second renaming method was not made.

### 15.3. Selectors of the `rename` command

```json
{"id": 812, "rename": "Plan 1"}
{"view": "{3D}", "rename": "Overview"}
{"level": 812, "plan": "floor", "rename": "Ground Floor"}
{"level": "Level 1", "plan": "ceiling", "rename": "RCP"}
{"schedule": "door", "rename": "Doors"}
{"sheet": "A101", "rename": "Floor Plans"}
{"family": "Door 900", "rename": "Door 900 x 2100"}
{"family": "Door 900", "type": "900 x 2100", "rename": "900 x 2200"}
{"category": "door", "type": "900 x 2100", "rename": "900 x 2200"}
```

- `id` on a `Level` still renames the level itself; `level` + `plan` — its own
  plan as a view.
- `plan` — `floor` (the default) or `ceiling`.
- `id`/`view` on an existing `PlanView` changes only the view name.
- Reply: `{kind, id?, from, to, created?}`; `created` only `true`.

### 15.4. Tests

Added:

- `crates/core/tests/rename.rs`: renaming a level, a sheet (the number is kept, a foreign number
  is rejected), a built-in schedule (one row, Undo), the prohibition of a foreign built-in name,
  `{3D}`, a family/type, refusal on a taken name, and the level's own plan as a `PlanView`
  (the view name is independent of the level, floor and ceiling separately, Undo step by step).
- `crates/ui-egui/src/tests/browser.rs`: F2 on a plan, Esc and a duplicate name, a built-in
  schedule as one row, renaming through the context menu, F2 without a selection as Floor Plan.

Test helpers that used to open a view with a single click were updated: `browse`,
`browse_entry`, `topmost`, as well as the expectation in `tests/printsets.rs`.

### 15.5. Checks

- `cargo fmt --check`
- `cargo check -p bimcraft-core -p bimcraft-ui-egui`
- `cargo clippy -p bimcraft-core -p bimcraft-ui-egui -- -D warnings`
- `cargo i18n check`
- a manual CLI run: renaming a level, a sheet, a built-in schedule and `{3D}`.

Tests were not run locally (per the project rule); they are run by CI in PR #664.

### 15.6. What was not included

- F2 with the Family Editor active: the keys belong to the family document, the Project Browser
  shows its tree, not the project tree; renaming the project's family from there was not
  added.
- `Linked Models` are not renamed (this remains a separate stage).
- The sheet number does not change on F2.
- Renaming several selected rows with one command was not made: F2 edits the row with the
  accent stripe.

### 15.7. Additions found while polishing

- `BimApp::view_name(Sheet::Ceiling(key))`: for a `Level` it takes `Model::ceiling_view_name`, for a
  `PlanView` — its own name. Without this the ceiling plan tab showed the floor plan's name.
- `BimApp::unplaced_views` names the floor plan by `Model::view_name(level)` (the primary plan's name,
  if it exists); `Model::plan_views` no longer returns the primary, so the view is not duplicated in
  the Place View list.
- `duplicate_view`: the primary plan can be duplicated; `Duplicate as Dependent` is allowed because
  `Model::check` considers the primary plan a valid parent. `annotation_in` follows the parent
  chain, so a view dependent on the primary plan sees the level's annotations.
- Properties: `Sheet::Plan(level)` substitutes the primary plan's id if it exists; the primary plan
  itself is shown with `PLAN_VIEW` rows, not `PLAN_DEPENDENT`, so View Name, Associated Level and
  View Range remain available.
- `Model::free_view_name` reserves level names too; `Model::view_named` by a level name
  returns its primary floor plan if the view has already been materialized.
- The actual UI strings: `New name`, `A name can't be empty.`, `A name is at most {}
  characters.`, `Select an item in the Project Browser to rename.`, `This item can't be renamed.`,
  `A sheet's number changes in Properties.`; `Rename` and the `F2` hint in the row menu.
- CI: the first PR #664 run failed on the outdated `translations/project/bimcraft-project.pot`
  (the shared `target` cache hid this locally) and on a test string with `{3D}` unescaped in
  a `format!` message. The catalogs were rebuilt forcibly, the message was fixed to `{{3D}}`.
