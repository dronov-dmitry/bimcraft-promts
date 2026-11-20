# SPEC-026. The project file name in the window title bar

Status: **task**, implementation and a CI test run are required
Date: 2026-11-20
Project: BIMCraft (Rust, `BIMCraft-025`, `eframe`/`egui` 0.36.2)
Source: the user's request of 2026-11-20: "prepare the file SPEC-026-***.md in which
you describe the task by analogy with the neighbouring SPEC files, stating that the file
name must appear in the program's title bar before the place where the Project Name
parameter is set, and it must also be separated by a dot the way Project Name is
separated from the view name"; clarification of 2026-11-20: "redo SPEC-026, the file name
must be without an extension."
Related PRs: none, this is a new task; title bar behaviour has not changed since the
window title began to be composed from the project name and the view.
Task type: UI change. The model, the project format, commands and the API do not change;
only the string the window shows in its own title bar and in the system window title
changes.

> **In short.** The window's title bar is currently composed from the project name and the
> view: `Project · Floor Plan: Level 1` (`crates/ui-egui/src/lib.rs:4729-4738`), and the
> system window title is `Project · Floor Plan: Level 1 · BIMCraft`
> (`crates/ui-egui/src/lib.rs:2308-2318`). The name of the file that holds the project is
> not there. It must be placed **first**, before the Project Name, and separated by the same
> ` · ` separator (U+00B7 with spaces) that separates the Project Name from the view name:
> `House · Project · Floor Plan: Level 1`. The name comes from `Session::path`
> (`crates/core/src/engine.rs:169-172`) and is shown **without an extension**: the file
> `House.bimcraft` appears in the title bar as `House`. A project that has not been saved
> yet has no path, so the title bar stays as it is.

## 0. General rules of work

1. Communication language is Russian. Code, comments, interface strings and log messages
   are English. The document (`.md`) is Russian.
2. Tests (`cargo test`, `cargo nextest`, doctests) are **not run** in this task unless the
   user explicitly asks for it (the rule in `~/.config/opencode/rust-rules.md`). Allowed are
   `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
   `cargo clippy -p bimcraft-ui-egui` (without `--all-targets`) and `cargo i18n check`.
3. No new dependencies. No `unsafe` is added, lints are not weakened.
4. Model behaviour, the `.bimcraft` format, commands and the API do not change. Only the
   title bar text changes: the project file name without an extension is added to it.
5. There are no new translatable strings: the file name is project data, not an interface
   string. `translations/` is not touched.
6. Commits are atomic: title composition separately, tests separately.

## 1. Current behaviour

Reproduction:

1. Open a saved project, for example `House.bimcraft` with Project Name `Project`, and the
   `Floor Plan: Level 1` tab.
2. Look at the window's title bar (which is also the system window title if "Use the
   system's title bar" is enabled in Settings > User Interface).
3. You see:

```
Project · Floor Plan: Level 1 · BIMCraft
```

The file name is missing: two different files with the same Project Name are
indistinguishable in the title bar, while the user's task requires seeing which file is
open.

Expected behaviour:

```
House · Project · Floor Plan: Level 1 · BIMCraft
```

- the file name is **first**, before the Project Name;
- the separator is the same ` · ` (space, U+00B7, space) as between the Project Name and
  the view name;
- the file name is **without an extension**: `House.bimcraft` shows `House`, and
  `house.v2.bimcraft` shows `house.v2`;
- the Project Name is still taken from Project Information and does not change;
- for a project without a file (new, unsaved, untitled recovery) the title bar looks
  exactly as it does today;
- the same is visible in the window's own title bar (when the system does not draw it),
  only without `BIMCraft`: `House · Project · Floor Plan: Level 1`.

## 2. How the title bars are built now

### 2.1. System window title

`crates/ui-egui/src/lib.rs:2308-2318`:

```rust
let view = self.view_title(self.sheet);
let title = match self.family_title() {
    Some(family) => format!("{family} · BIMCraft"),
    None => {
        format!("{}{} · {view} · {}BIMCraft", self.session.model.name, if self.dirty() { "*" } else { "" }, self.cowork_title())
    }
};
if title != self.title {
    ctx.send_viewport_cmd(ViewportCommand::Title(title.clone()));
    self.title = title;
}
```

This title is seen both by the window manager (taskbar, window switcher) and by the user
when the system title bar is selected in the settings. The string is cached in
`BimApp::title` (`lib.rs:1172`, `lib.rs:1983`) and is sent only when it changes.

### 2.2. The window's own title bar

When the window draws its title bar itself (`BimApp::own_bar()`, `titlebar.rs:40-43`), the
text is drawn in `quick_access` (`crates/ui-egui/src/lib.rs:4717-4738`):

```rust
let mut title = egui::text::LayoutJob::simple_singleline(
    self.family_title().unwrap_or_else(|| format!("{} · {view}", self.session.model.name)),
    FontId::proportional(12.5),
    pal().app_bar_muted,
);
title.wrap = egui::text::TextWrapping::truncate_at_width((right - left).max(0.0));
let galley = ui.painter().layout_job(title);
let width = galley.size().x;
let x = ui.max_rect().center().x.clamp(left + width / 2.0, (right - width / 2.0).max(left + width / 2.0));
ui.painter().galley(pos2(x - width / 2.0, row.center().y - galley.size().y / 2.0), galley, pal().app_bar_muted);
```

The text is centred between the buttons on the left and the palette/window buttons on the
right; when there is not enough room it is cut off by width via `truncate_at_width`.

### 2.3. Where the file name comes from

`crates/core/src/engine.rs:169-172`:

```rust
pub struct Session {
    pub model: Arc<Model>,
    /// Where the model was opened from or last saved to.
    pub path: Option<PathBuf>,
```

- `open` sets the path of the opened file, `save`/Save As sets the path it was saved to;
- `recover` deliberately sets the path of the **project**, not of the temporary backup
  (`crates/core/src/commands.rs:5459-5466`); an untitled recovery keeps the path as `None`
  (`autosave_tests.rs:48-49`), so there is no file name and the title bar stays as before;
- exporting a copy does not change the path.

So `Session::path` is the right source: it is exactly the file that is open, and it changes
on Open and Save As without any additional events. The title bar receives not the whole file
name (`House.bimcraft`) but its stem without the last extension — `House`
(`Path::file_stem`).

### 2.4. What the title bar does not show now either

- A family opened in the Family Editor shows its name and view
  (`family_title()`, `crates/ui-egui/src/editor/document.rs:328-332`): it has no "project"
  file, and this behaviour does not change.
- The start page draws only `BIMCraft`
  (`crates/ui-egui/src/lib.rs:2449-2462`): there is no project on it.

## 3. Goals

1. For a saved project the file name (without an extension) is **first** in the title bar,
   before the Project Name, in both title bars: the window's own one and the system one.
2. The separator between the file name and the Project Name is exactly ` · ` (space,
   U+00B7, space), as between the Project Name and the view name.
3. Only the file name without an extension is shown; the full path and the extension are
   not shown.
4. The Project Name is still taken from Project Information (`model.name`) and is not
   replaced by the file name.
5. For a project without a path the title bar stays exactly as it is today.
6. The behaviour of family documents and of the start page does not change.
7. The window's own title bar keeps its font, colour, centring and width truncation.
8. The file name updates immediately after Open and Save As, without a restart.

## 4. Non-goals

- Changing the Project Name, its storage in the model, or the Project Information dialog.
- Showing the full path, directory, network location, or a tooltip with the path.
- Showing the extension (`.bimcraft`, `.ifc`) or the file format in the title bar.
- Renaming the file, changing the Open/Save/Save As dialogs or the recent files list.
- Moving the unsaved-changes asterisk (`*`) from the Project Name to the file name.
- Showing the file name on the start page or in the title bars of family documents.
- Changing the view tab titles, the Project Browser, or the OS taskbar beyond the window
  text.
- New translatable strings, edits in `translations/`.
- Changes to the model, the file format, commands, IFC and the API.

## 5. Functional requirements

### F1. A single source of the file name

Add to `impl BimApp` next to `view_title`/`dirty`:

```rust
/// The project file's name without its extension, as the title bars show it: `House`.
/// A project never saved has none.
fn project_file_stem(&self) -> Option<String> {
    let name = self.session.path.as_deref()?.file_stem()?;
    Some(name.to_string_lossy().into_owned())
}
```

Requirements:

- only the name stem is taken (`file_stem()`), without directories and without the last
  extension: `House.bimcraft` → `House`;
- dots inside the name are preserved: `house.v2.bimcraft` → `house.v2`; only the last
  extension is dropped, not everything after the first dot;
- a path ending in a separator or at a root yields `None` (`Path::file_stem` has no name
  for it) — the title bar then stays as before;
- a file named simply `.bimcraft` yields the stem `.bimcraft` under Rust's rules: this is a
  rare case, and there is no crash;
- a non-UTF-8 name does not panic: `to_string_lossy` replaces the lost bytes with `�`;
- the function caches nothing: the path is read every frame, so Open and Save As are picked
  up immediately.

### F2. System window title

Move the string assembly into a separate method so it can be covered by a test:

```rust
/// The whole window, as the system's title bar and the task switcher say it.
pub(crate) fn window_title(&self, view: &str) -> String {
    let file = self.project_file_stem().map(|f| format!("{f} · ")).unwrap_or_default();
    let star = if self.dirty() { "*" } else { "" };
    format!("{file}{}{star} · {view} · {}BIMCraft", self.session.model.name, self.cowork_title())
}
```

In `show()` (`lib.rs:2308-2318`) the project branch is replaced with
`self.window_title(&view)`; the family branch (`Some(family) => format!("{family} · BIMCraft")`)
stays as it is.

Requirements:

- order: `file · Project Name · view · [Co-Work · ]BIMCraft`;
- the `*` asterisk stays right after the Project Name, as today;
- `cowork_title()` (`crates/ui-egui/src/cowork.rs:474-480`) stays where it is and already
  ends with ` · `;
- the `self.title` cache and the `ViewportCommand::Title` send do not change: the string
  comparison itself will notice the new file name after Save As.

### F3. The window's own title bar

The text for `quick_access` is assembled by the same rule, but taking into account the width
that has already been computed (`right - left`). A separate method:

```rust
/// What the window's own title bar says: the file, the project and the view, or the project
/// and the view when the file's name would take the room of either.
pub(crate) fn bar_title(&self, view: &str, room: f32, ctx: &Context) -> String {
    let with_file = match self.project_file_stem() {
        Some(file) => format!("{file} · {} · {view}", self.session.model.name),
        None => return format!("{} · {view}", self.session.model.name),
    };
    let font = FontId::proportional(12.5);
    let width = ctx.fonts_mut(|f| f.layout_no_wrap(with_file.clone(), font, Color32::WHITE)).size().x;
    if width <= room { with_file } else { format!("{} · {view}", self.session.model.name) }
}
```

In `quick_access`:

```rust
let mut title = egui::text::LayoutJob::simple_singleline(
    self.family_title().unwrap_or_else(|| self.bar_title(view, (right - left).max(0.0), ui.ctx())),
    FontId::proportional(12.5),
    pal().app_bar_muted,
);
```

Requirements:

- the order and separators match F2, but without `BIMCraft`;
- the font `FontId::proportional(12.5)`, the colour `pal().app_bar_muted`, the centring and
  `truncate_at_width` do not change;
- **part priority:** if the full string with the file name does not fit into `room`, the
  file name is not shown at all, and the title bar becomes exactly today's
  (`Project Name · view`). The project and the view must not disappear because of a long
  file name;
- if even today's string does not fit, the previous `truncate_at_width` truncation at the
  end applies — that does not change;
- the family branch (`family_title()`) still has priority and gets no file name.

### F4. Exact format and examples

The separator is everywhere the same literal `" · "` (space, U+00B7, space). No extension
gets into the title bar: `House.bimcraft` → `House`, `house.v2.bimcraft` → `house.v2`. A
plain dot `.` is not used as a separator.

| Project state | System title bar | Own title bar |
|---|---|---|
| `House.bimcraft`, Project Name `Project`, view `Floor Plan: Level 1` | `House · Project · Floor Plan: Level 1 · BIMCraft` | `House · Project · Floor Plan: Level 1` |
| The same file with unsaved changes | `House · Project* · Floor Plan: Level 1 · BIMCraft` | `House · Project · Floor Plan: Level 1` |
| `house.v2.bimcraft`, Project Name `Project` | `house.v2 · Project · Floor Plan: Level 1 · BIMCraft` | `house.v2 · Project · Floor Plan: Level 1` |
| New project without a path | `Project · Floor Plan: Level 1 · BIMCraft` | `Project · Floor Plan: Level 1` |
| In Co-Work (host) | `House · Project · Floor Plan: Level 1 · Co-Work · BIMCraft` | `House · Project · Floor Plan: Level 1` |
| A family in the Family Editor | `Door · 3D View · BIMCraft` (as today) | `Door · 3D View` (as today) |

### F5. A project without a file

`session.path == None` — a new project, an untitled recovery, an import: the title bars look
exactly as today, without a leading ` · ` and without an empty slot. The `None` check must
happen before formatting, not after (no `"None"` and no double separators).

### F6. Live update

- Open: the path appears → both title bars change in the same frame in which the model was
  replaced.
- Save As: the path changes → the file name changes without a restart; the `self.title` cache
  sends the new window title.
- Save to the same file: nothing changes, there is no redundant `Title` send.
- Switching to another view tab: only the view part changes, the file name and the Project
  Name stay.

### F7. Families and the start page

- `family_title()` has priority in both title bars; the project file name is not shown in a
  family document.
- `start_title_bar` stays with a single `BIMCraft`.

### F8. Localization

There are no new strings; `cargo i18n check` stays green. The file name is not translated and
not escaped: it is data, not an interface message.

## 6. Key decision and alternatives

### 6.1. Recommended solution: the name stem from `Session::path` in the UI layer

The file path is a property of the session, not of the model: it is not saved into
`.bimcraft` and must not get there. Therefore the file name is read in the UI from
`self.session.path` (`engine.rs:169-172`) and inserted into the two existing title bar
strings. Both places use one helper, `project_file_stem()`, so the format cannot diverge.
`Path::file_stem()` drops only the last extension, so dots inside the name
(`house.v2.bimcraft` → `house.v2`) are preserved, while the technical format suffix leaves
the title bar together with the extension.

For the window's own title bar a width check is added: the file name is the newest and the
most optional part, so when there is not enough room it yields first, while the Project Name
and the view stay readable as today.

### 6.2. Rejected options

| Option | Why not |
|---|---|
| Show the file name with its extension (`file_name`) | Directly contradicts the clarified request: the name must be without an extension |
| Store the file name in `model.name` | Changes the saved data and the Project Name; the path does not belong to the model |
| Show the full path | Long, exposes directories and network names; the user asked for the file name |
| Drop everything after the first dot | Would break `house.v2.bimcraft` → `house`; the correct behaviour is to remove only the last extension |
| Put the name after the Project Name | Contradicts the request: the name must be **before** the Project Name |
| Separate with a plain dot `"."` | The request says "the way Project Name is separated from the view name", and that is ` · `; a plain dot would blend with the dots inside the name |
| Hide the Project Name when it equals the file name | Changes the meaning of the title bar and the Project Information behaviour; duplication is acceptable |
| Move `*` to the file name | A separate behaviour change; in this task the asterisk stays after the Project Name |
| Cache the file name in `BimApp` | Would need hooks on Open/Save As; reading `Path::file_stem()` once per frame is cheaper and cannot desynchronize |
| Always truncate the tail of a long string | With a long file name the view would disappear from the title bar; F3 removes the file name first |
| Draw the file name as a separate `Label` over the title bar | Breaks centring and window dragging; the text is drawn by one `LayoutJob`, as now |

## 7. Edit points

| File | What to change |
|---|---|
| `crates/ui-egui/src/lib.rs:2308-2318` | system window title: `self.window_title(&view)` |
| `crates/ui-egui/src/lib.rs:4717-4738` | own title bar: `self.bar_title(view, room, ui.ctx())` |
| `crates/ui-egui/src/lib.rs` (next to `view_title`, `dirty`) | new `project_file_stem`, `window_title`, `bar_title` |
| `crates/ui-egui/src/tests/titlebar.rs` | tests T1–T7 (`own()` and `asked()` already exist there) |

`Session` (`crates/core/src/engine.rs`), `recover` (`commands.rs:5459-5466`),
`family_title` (`editor/document.rs:328-332`), the `quick_access` layout, `titlebar.rs`
and `translations/` do not change.

## 8. Test plan (to be added; the run is done by CI/the user)

Tests go into `crates/ui-egui/src/tests/titlebar.rs`, in the existing `egui_kittest` style.
The path is set the same way as in `tests/assistant.rs:768`:
`h.state_mut().session.path = Some(dir.path().join("Sample House.bimcraft"));` —
this does not make the project dirty.

### T1. System title bar with a file

1. `own(BOX)`, set `session.path` to `…/Sample House.bimcraft`.
2. `h.state().window_title("Floor Plan: Level 1")` equals
   `Sample House · Project · Floor Plan: Level 1 · BIMCraft`.
3. After `asked(&mut h)` the commands include `ViewportCommand::Title` with the same string.

### T2. A project without a file does not change

1. `own(BOX)`, `session.path == None`.
2. `window_title(...)` equals today's
   `Project · Floor Plan: Level 1 · BIMCraft`, without a leading separator.
3. `bar_title(..., 2000.0, ctx)` equals `Project · Floor Plan: Level 1`.

### T3. Own title bar with a file

1. As in T1.
2. `bar_title("Floor Plan: Level 1", 2000.0, h.ctx())` equals
   `Sample House · Project · Floor Plan: Level 1`.
3. The separators are exactly `" · "`: check `split(" · ")` for three parts.
4. The string does not contain the `.bimcraft` extension.

### T4. The asterisk stays after the Project Name

1. Set the path, run any command that changes the model (`h.state_mut().run("wall", …)`).
2. `window_title(...)` equals `Sample House · Project* · Floor Plan: Level 1 · BIMCraft`.
3. `bar_title(...)` contains no asterisk, as today.

### T5. A long file name yields room

1. Set a path with a very long name stem (for example 120 characters) and `.bimcraft`.
2. `bar_title(view, 200.0, ctx)` equals `Project · Floor Plan: Level 1` — the file name is
   removed entirely.
3. With `room = 2000.0` the string with the file name is returned.
4. `window_title(...)` does not depend on `room` and always contains the file name.

### T6. Save As changes the name in the title bar

1. Set the path `…/First.bimcraft`, remember `window_title`.
2. Change `session.path` to `…/Second.bimcraft`.
3. `window_title` contains `Second · ` and does not contain `First`;
   `h.step()` sends the new `ViewportCommand::Title`.

### T7. Family and start page

1. Open a family document in the Family Editor.
2. Both title bars remain `family_title()` strings, without the project file name.
3. On the start page (`home.shown`) the own title bar is still only
   `BIMCraft`.

### T8. Hygiene

`cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
`cargo clippy -p bimcraft-ui-egui`, `cargo i18n check` pass.

## 9. Acceptance criteria

- [ ] A saved project shows `file · Project Name · view` in the system title bar.
- [ ] The own title bar shows the same without `BIMCraft`.
- [ ] The file name is first, before the Project Name.
- [ ] The separator is exactly ` · ` (U+00B7 with spaces), as between the Project Name and
      the view.
- [ ] The file name is **without an extension** and without directories: `House.bimcraft` →
      `House`.
- [ ] Dots inside the name are preserved: `house.v2.bimcraft` → `house.v2`.
- [ ] The Project Name is still taken from Project Information.
- [ ] The `*` asterisk stays right after the Project Name.
- [ ] A project without a path looks exactly as today (no leading ` · `).
- [ ] A long file name is removed first; the Project Name and the view remain.
- [ ] Save As and Open update the file name without a restart.
- [ ] Family documents and the start page are unchanged.
- [ ] There are no new strings in `translations/`, `cargo i18n check` is green.
- [ ] `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
      `cargo clippy -p bimcraft-ui-egui` pass.
- [ ] Tests were not run locally unless the user asked for it; this is reflected in the PR.

## 10. Risks and open questions

1. **Duplication.** If the file is called `Project.bimcraft` and the Project Name is
   `Project`, the title bar shows `Project · Project · …`. This is expected: the file name
   and the Project Name are different entities, and the request requires both.
2. **Very long names.** In a narrow window the file name is removed entirely (F3) rather than
   truncated. This is a deliberate choice: keep the Project Name and the view without adding
   new ellipsis logic. In a wide window everything is visible.
3. **Dots in the name.** `file_stem()` is used, that is, only the last extension is dropped:
   `house.v2.bimcraft` → `house.v2`. A file named simply `.bimcraft` yields the stem
   `.bimcraft` under Rust's rules — a rare case, with no crash.
4. **Non-UTF-8 names.** `to_string_lossy` may show `�`; there is no panic and no file loss.
5. **Privacy.** Only the file name is shown, the directory is not exposed — this matters for
   screenshots and screen sharing.
6. **Untitled recovery.** `recover` without `project` leaves `path = None`
   (`autosave_tests.rs:48-49`), so the title bar stays as before — which is correct: there
   is no file yet.
7. **Network and removable drives.** The name comes from the path, not from the file system,
   so an unavailable drive does not affect the title bar.
8. **Meaning of the asterisk.** It stays with the Project Name, although it would arguably
   look more logical on the file name; moving it is a separate task, so as not to change
   behaviour together with this edit.
9. **Consistency of the two title bars.** Both are assembled from the same helpers; a
   divergence is possible only in `BIMCraft`, Co-Work and the asterisk, as today.

## 11. Order of work and DoD

1. Add `project_file_stem()`, `window_title()`, `bar_title()` to `impl BimApp`.
2. Switch `show()` to `window_title(&view)`.
3. Switch `quick_access` to `bar_title(view, room, ctx)`, keeping `LayoutJob`, the font, the
   colour and the centring.
4. Add tests T1–T8 to `crates/ui-egui/src/tests/titlebar.rs`.
5. Run `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
   `cargo clippy -p bimcraft-ui-egui`, `cargo i18n check`.
6. Do not run tests without an explicit request; state in the PR that CI will run them.
7. Split into commits:
   - `ui: the project's file name in the window title`;
   - `ui: tests for the file name in the title bar`.
8. Definition of Done: the acceptance criteria are met, the model, the file format and the
   commands are unchanged, there are no new dependencies and no `unsafe`.

## 12. What not to do

- Do not change the Project Name or the Project Information dialog.
- Do not show the full path and do not add a tooltip with the directory.
- Do not add the file extension to the title bar.
- Do not change the separator to `.` (a plain dot) or `-` — only ` · `.
- Do not move `*` from the Project Name to the file name.
- Do not add the file name to the start page or to family documents.
- Do not touch the `.bimcraft` format, the model, commands, IFC or the API.
- Do not add strings to `translations/` and do not translate the file name.
- Do not cache the file name in a way that would make Save As require a restart.
