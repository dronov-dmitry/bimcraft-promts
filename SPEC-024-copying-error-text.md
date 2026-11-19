# SPEC-024. Copying an error message: selection, Ctrl+C, and a Copy button

Status: **draft**, implementation required
Date: 2026-11-19
Video: https://youtu.be/8cws699O6i4
Project: BIMCraft (Rust, `BIMCraft-024`, `eframe`/`egui` 0.36.2)
Source: user request of 2026-11-19 (translated from Russian): "right now, when an error
pops up you cannot copy it. You select the text, press Ctrl+C, and it says select what
to copy first. Write SPEC-024-***.md describing the task of fixing this bug: if an error
appears and you select it, it must be possible to copy it. Also make a Copy button inside
the error box so that the error can be copied to the clipboard in one click."
Related PRs:
- **#473** "Copy-Paste everything" — introduced the global interception of
  Ctrl+C/Ctrl+X/Ctrl+V (`clipboard_events`) and the string `select what to copy first`;
  this is where the regression comes from.
- **#53** "Revit import report: a Copy Report button…" — an already accepted project
  pattern of a button copying long text; we take it as a model for the Copy button.
No direct PR or issue fixing this task was found in the repository
(all PRs and issues were checked for the words copy, clipboard, error, message, status,
`select what to copy first`, as of 2026-11-19).
Task type: UI behaviour fix. The model, project format, commands and ABI do not change.

> **In short.** The error box is `BimApp::warning()` (`crates/ui-egui/src/lib.rs:7141-7162`).
> The text in it is drawn by a plain `ui.label`, that is, it is **not selectable**.
> Besides that, `BimApp::keys()` calls `clipboard_events()` **before** the UI is drawn,
> and that one, via `std::mem::take`, takes every `Event::Copy`/`Event::Cut`/`Event::Paste`
> out of `i.events`. Therefore Ctrl+C never reaches any text, and `copy_to_clipboard()`
> with an empty element selection **replaces the original error** with the message
> `select what to copy first`. Both causes must be fixed: make the box text selectable,
> stop taking Copy/Cut away from egui when the application has a text selection,
> and add a **Copy** button to the box itself that copies the whole message in one click.

## 0. General rules of work

1. Communication language is Russian. Code, comments, UI strings and log messages are
   English. The document (`.md`) is Russian.
2. Tests (`cargo test`, `cargo nextest`, doctest) are **not run** in this task without an
   explicit user request (the rule in `~/.config/opencode/rust-rules.md`). Allowed are
   `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`, `cargo clippy -p bimcraft-ui-egui`
   (without `--all-targets`) and `cargo i18n check/update`.
3. No new dependencies. No `unsafe` is added, lints are not relaxed.
4. The behaviour of the model, the project file, commands and the API does not change.
   Only the window changes: text selection in the status box and the ways to put it on
   the clipboard.
5. Commits are atomic: the status box behaviour separately, the global Ctrl+C handler
   separately, the i18n catalogs separately. Each commit builds.

## 1. Symptom

Reproduction:

1. Get any error, for example press Ctrl+C without selecting anything in the plan.
   In the bottom-right corner of the view a box appears with the title **Error** and the
   text `select what to copy first`.
2. Try to select the text inside the box with the mouse.
   No selection appears: the text is not highlighted, the cursor stays normal.
3. Press Ctrl+C.
4. Instead of copying, the box **changes its message** to `select what to copy first`
   (the original error is lost), and nothing goes to the clipboard.
   `Ctrl+V` pastes only what was in the clipboard before.

Expected behaviour:

- the error text can be selected with the mouse and the selection is visible;
- Ctrl+C copies **exactly the selected fragment** to the system clipboard;
- the error box does not disappear and does not change its text;
- the box has a **Copy** button that copies the whole message in one click;
- the same works for the **Note** box (non-errors), because it is the same widget.

## 2. Diagnosis

### 2.1. Where the error box is drawn

`crates/ui-egui/src/lib.rs:7141-7162`:

```rust
/// Why the last command failed, in a box over the view's corner until the next one works.
fn warning(&mut self, ctx: &Context) {
    if self.status.text.is_empty() {
        return;
    }
    let (title, color) = if self.status.error { (t("Error"), pal().error) } else { (t("Note"), pal().accent_ink) };
    let at = self.view_rect.right_bottom() - vec2(16.0, 16.0);
    Area::new(UiId::new("warning")).order(Order::Middle).fixed_pos(at).pivot(Align2::RIGHT_BOTTOM).show(ctx, |ui| {
        Frame::popup(ui.style()).fill(pal().surface).stroke(Stroke::new(1.0, color)).show(ui, |ui| {
            ui.set_max_width(380.0);
            ui.horizontal(|ui| {
                ui.label(RichText::new(title).strong().color(color));
                ui.with_layout(Layout::right_to_left(Align::Center), |ui| {
                    if icon_button(ui, Icon::Close, "Dismiss").clicked() {
                        self.status = Status::default();
                    }
                });
            });
            ui.label(&self.status.text);
        });
    });
}
```

The status box is `Status { text, error }` (`crates/ui-egui/src/lib.rs:353-356`).
It is shown both on the start page (`lib.rs:2342-2355`) and in the main window
(`lib.rs:2432`), so the fix must work in both places.

### 2.2. Why Ctrl+C does not copy the text

The frame order in `BimApp::show()`:

1. `self.keys(&ctx)` (`lib.rs:2303`) — key handling **before** the UI is built.
2. Inside `keys()` `self.clipboard_events(ctx)` is called (`lib.rs:2713-2716`).
3. `clipboard_events()` (`crates/ui-egui/src/clipboard.rs:237-274`) takes the events
   out of the input entirely:

```rust
let events: Vec<Event> = ctx.input_mut(|i| {
    let (ours, rest): (Vec<Event>, Vec<Event>) =
        std::mem::take(&mut i.events).into_iter().partition(|e| matches!(e, Event::Copy | Event::Cut | Event::Paste(_)));
    i.events = rest;
    ours
});
```

4. `Event::Copy` goes to `copy_to_clipboard()` (`clipboard.rs:287-291`):

```rust
pub(crate) fn copy_to_clipboard(&mut self, ctx: &Context, cut: bool) {
    if self.selection.is_empty() {
        self.status = Status { text: t("select what to copy first").into(), error: true };
        return;
    }
    ...
```

That is, with an empty element selection the original `status.text` is **overwritten**
with the copy error message. This is exactly why the user sees
`select what to copy first` instead of their own error.

5. Even if `ui.label` were made selectable, it would not help: the egui widget
   `Label` with `selectable(true)` copies the selection by reading the same `i.events`
   (`egui-0.36.2/src/text_selection/label_text_selection.rs:679-685`,
   `got_copy_event()`), and by the time of drawing the events have already been taken
   by step 3.

### 2.3. Two independent causes

| # | Cause | Where | Consequence |
|---|---|---|---|
| 1 | The status box text is a plain `Label`, `selectable` is not enabled | `lib.rs:7159` | It cannot be selected with the mouse |
| 2 | `clipboard_events()` takes `Copy`/`Cut` out of `i.events` before the UI | `clipboard.rs:238-243` | Even a selectable label will not get Ctrl+C |
| 3 | `copy_to_clipboard()` writes a new error into `status` when the selection is empty | `clipboard.rs:288-291` | The original error is lost |

All three items must be fixed. The Copy button by itself covers only
"in one click", but not the requirement "select it — copy it".

### 2.4. Why the existing Copy Report button does not fit

In #53 a separate **Copy Report** button was made for the long Revit report
(`crates/ui-egui/src/revit.rs:259-274`): the report text was not selectable, and the
button copied it in full. That is a good model for the **button**, but it does not solve:
- selecting the text in the error box;
- the interception of Ctrl+C by the global handler;
- the loss of the original error when trying to copy it.

In our spec the Copy button complements selection and Ctrl+C rather than replacing them.

### 2.5. Checking PRs and issues

A search across all PRs/issues of the `IPedrax/BIMCraft` repository (GitHub Search API,
2026-11-19) for the words `copy`, `clipboard`, `error`, `message`, `status`,
`select what to copy first`, `status box`, `error box`, `warning box`, `Ctrl+C`,
`selectable` found no ready fix for this task.

Related work found:

- **#473** (merged 2026-11-18) — added `clipboard_events()` and the string
  `select what to copy first`. This is the source of the regression.
- **#53** (merged 2026-10-23) — added **Copy Report** for the Revit import report.
  We take the button and the test pattern from there, but the error box behaviour was
  not changed there.
- **#213** (closed) — shortened long error messages, did not touch copying.

## 3. Goals

1. The text of the Error/Note box is selected with the mouse like ordinary text, and the
   selection is visible.
2. Ctrl+C with a text selection copies the **selected fragment** to the system clipboard
   and does not start copying elements.
3. Ctrl+C without a text selection works as before: it copies the selected elements and,
   with an empty selection, shows `select what to copy first`.
4. The original error is never replaced by a copying message because of an attempt to
   copy its text.
5. The Error/Note box has a **Copy** button that copies `status.text` in full in one click.
6. The button is keyboard accessible and localized through the existing `t()`.
7. The appearance of the box (bottom-right corner of the view, width up to 380 px, the
   title, Dismiss) does not change, apart from the added button.

## 4. Non-goals

- Copying the hint from the bottom status bar (`status_bar`) — that is not error text.
- Automatic saving of errors to a log/file.
- Clipboard history, multiple slots, rich text.
- Changing the wording of errors and the translation of the line
  `select what to copy first`.
- Enabling `selectable(true)` globally for all `Label`s of the application via the style:
  that would change the behaviour of all panels and is not required.
- Copying the `Error`/`Note` title together with the message.
- Changing the model, `.bimcraft`, IFC, commands and the API.

## 5. Functional requirements

### F1. Selectable text in the Error/Note box

In `warning()` replace

```rust
ui.label(&self.status.text);
```

with

```rust
ui.add(egui::Label::new(&self.status.text).selectable(true));
```

Requirements:

- works for `error: true` and `error: false`;
- selection with the mouse within the text, including across lines when wrapping;
- the text still wraps by width (`ui.set_max_width(380.0)` is preserved);
- the selection must not intercept clicks on the Dismiss/Copy buttons;
- behaviour when clicking outside the text is standard for egui: the selection is
  cleared (`label_text_selection.rs`, `clicked_something_else`), after which Ctrl+C
  copies elements again.

### F2. Ctrl+C copies the selected text

If the application has a text selection in a label
(`LabelSelectionState::has_selection()`), then `clipboard_events()` **must not** take
`Event::Copy` and `Event::Cut` out of `i.events`. The events stay in the frame and egui
itself copies the selected fragment through `ui.copy_text(...)`. The global interception
from #473 stays in place: without a text selection Ctrl+C goes the old way through
`copy_to_clipboard()`.

Requirements:

- exactly the selected substring goes to the clipboard, without the title or any service
  additions;
- `self.status` does not change: neither the text nor `error`;
- `self.selection` (the selected model elements) does not change;
- `self.copied` and `self.clip_text` (BIMCraft's internal buffer for Ctrl+V) do not change;
- the message `select what to copy first` does not appear if text was selected;
- `Event::Paste` is still taken by the global handler, so Ctrl+V does not depend on the
  text selection.

### F3. Ctrl+C without a text selection — no regressions

- There are selected elements, no text is selected → the elements are copied, as now.
- There are neither elements nor text → `select what to copy first` is shown, as now.
- Ctrl+X with selected text copies the text (egui treats Cut as Copy for a label)
  and does **not** cut the model elements.

### F4. Copy button in the status box

In the box's title row, next to Dismiss:

```rust
ui.with_layout(Layout::right_to_left(Align::Center), |ui| {
    if icon_button(ui, Icon::Close, "Dismiss").clicked() {
        self.status = Status::default();
    }
    if ui.button(t("Copy")).on_hover_text(t("Copy the message to the clipboard")).clicked() {
        ui.ctx().copy_text(self.status.text.clone());
    }
});
```

Requirements:

- the button is always visible whenever the Error/Note box is visible;
- the label is `Copy` (through `t("Copy")`, the string already exists in the catalog);
- the hover hint is `Copy the message to the clipboard`;
- one click puts the **whole** `status.text` into the system clipboard unchanged;
- the box does not close and does not change its text;
- `self.copied` and `self.clip_text` are not touched, otherwise Ctrl+V would start trying
  to paste the error text as BIMCraft content;
- the button does not overlap Dismiss and does not push the title outside the 380 px;
- the button is keyboard accessible (Tab) and activated by Enter/Space like a normal
  `egui::Button`; for a screen reader it is a `WidgetType::Button` with the label `Copy`.

### F5. State and geometry

- The box remains an `Area` with `Order::Middle`, anchored to the bottom-right corner of
  the view.
- `ui.set_max_width(380.0)` is preserved.
- Long text still wraps; the title buttons stay on screen.
- The box must not disappear when Copy is pressed.

### F6. Localization

- `Copy` already exists in `translations/ui/bimcraft-ui.pot` (`msgid "Copy"`).
- The new string `Copy the message to the clipboard` is added by the standard
  `cargo i18n update`; untranslated languages fall back to English.
- `cargo i18n check` must be green.
- In the code — English strings only.

## 6. Key decision and alternatives

### 6.1. Recommended solution: priority of an explicit text selection

The global interception of Ctrl+C/Ctrl+X/Ctrl+V from #473 is **not removed and not
moved**. Exactly one condition changes: if egui has a text selection, then
`clipboard_events()` does not take `Event::Copy`/`Event::Cut` out of `i.events` but
leaves them to the widget that selected this text. Everything else works as before.

```rust
// crates/ui-egui/src/clipboard.rs, in clipboard_events
let text_selected = ctx
    .plugin_opt::<egui::text_selection::LabelSelectionState>()
    .is_some_and(|state| state.lock().has_selection());
let events: Vec<Event> = ctx.input_mut(|i| {
    let (ours, rest): (Vec<Event>, Vec<Event>) = std::mem::take(&mut i.events)
        .into_iter()
        .partition(|e| match e {
            Event::Copy | Event::Cut => !text_selected,
            Event::Paste(_) => true,
            _ => false,
        });
    i.events = rest;
    ours
});
```

This gives the following distribution:

| State in the frame | Ctrl+C | Ctrl+X | Ctrl+V |
|---|---|---|---|
| There is a text selection in a label | The selected text goes to the clipboard; `status` does not change | The same (a label cannot cut, egui treats Cut as Copy) | Works as usual, the model is pasted |
| No text selection, elements are selected | The model elements are copied | The model elements are cut | Works as usual |
| No text selection, no elements are selected | The old `select what to copy first` | The old Cut behaviour | Works as usual |

Why this way:

- `LabelSelectionState` is a public egui plugin
  (`egui::text_selection::LabelSelectionState`); `has_selection()` returns `true` if a
  text selection is alive in this frame;
- `plugin_opt` does not panic if the plugin is not registered (then we assume there is no
  selection and the old global path works);
- `Event::Paste` is still taken and handled, so Ctrl+V does not break;
- once the text selection is cleared (a click outside the text or Esc), the global Ctrl+C
  immediately returns to copying elements;
- the check only reads egui state, it resets and changes nothing.

### 6.2. Why this does not break the global interception from #473

1. **The handler stays in place.** `clipboard_events()` is still called from `keys()`
   before the UI and still handles Copy/Cut/Paste. We do not move it relative to drawing
   and do not touch the other shortcuts.
2. **Model copying is not removed.** The `copy_to_clipboard()` branch for Copy and Cut
   stays exactly the same; it simply does not run when the user has explicitly selected
   text. Without a text selection Ctrl+C goes the same way as in #473.
3. **The explicit Copy command always copies the model.** The Copy button on the ribbon,
   the context menu item and `Action::CopyToClipboard` go through `clip_request`, not
   through keyboard events. That branch stays unconditional: even with selected text the
   explicit command copies the model elements.
4. **Ctrl+V is unaffected.** Paste is always taken, regardless of the text selection.
5. **The text selection clears itself.** egui resets it on a click outside the text or on
   Esc; from the next frame Ctrl+C copies the model again. The user does not need to
   "unlock" anything.
6. **It is the same priority text inputs already have.** In `keys()`
   (`lib.rs:2694-2703`), when a `TextEdit` is focused, there is already an early return
   based on `ctx.egui_wants_keyboard_input()`: a text field takes Ctrl+C before the model.
   We extend this rule to selected text in a label without changing the rest of the logic.
7. **Bonus.** Currently the already existing selectable labels are broken too: IDs in
   Inquiry, plugin warnings, the Markdown report. After the fix Ctrl+C will start copying
   text there as well — without separate code for each place.

Bottom line: both scenarios work, but not at the same time and not "at random": whatever
the user has explicitly selected wins. The model is copied when no text is selected; text
is copied when it is selected; an explicit Copy command from the ribbon always copies the
model.

### 6.3. Rejected options

| Option | Why not |
|---|---|
| Only a Copy button | Does not fulfil the requirement "selected the text — Ctrl+C copied it" |
| `selectable(true)` without changing `clipboard_events` | Ctrl+C is still taken before the UI, the text is not copied |
| Never take Copy/Cut | Would break copying of model elements |
| Defer model copying to the end of the frame and check whether egui copied the text | Works, but is more complex: needs a deferred `pending`, output inspection and a one-frame delay; more risk for #473 than a single `has_selection()` check |
| Draw the box as a read-only `TextEdit::multiline` | Brings focus, a cursor, key handling and a conflict with global shortcuts; heavier and riskier than a label |
| Move `clipboard_events()` after the UI | Changes the processing order of all shortcuts and the whole frame; risk of regressions across the entire application |
| Copy elements when there is a text selection | The user expects text, not elements |
| Write `status = "Copied"` after the button | Would destroy the error text the button exists to copy |

## 7. Edit points

| File | What to change |
|---|---|
| `crates/ui-egui/src/lib.rs:7141-7162` | `warning()`: `selectable(true)` and the Copy button |
| `crates/ui-egui/src/clipboard.rs:237-274` | `clipboard_events()`: do not take Copy/Cut when text is selected |
| `translations/ui/bimcraft-ui.pot` and `*.po` | the button hint string via `cargo i18n update` |
| `crates/ui-egui/src/tests/clipboard.rs` (or `tests.rs`) | new tests |

`Status` (`lib.rs:353-356`) does not need to change: no new state field is required, and
the Copy button does not keep a "pressed" state.

## 8. Test plan (add them; running is up to CI/the user)

Tests are written in the existing `egui_kittest` style (`tests/clipboard.rs`,
`tests/revit.rs`). To check the system clipboard use
`h.output().platform_output.commands` and the `egui::OutputCommand::CopyText(text)` variant.

### T1. The error text is selectable and Ctrl+C copies the selection

1. Open the application, select nothing.
2. Send `Event::Copy` and run a frame — a box appears with
   `select what to copy first`.
3. Find the box label via `query_by_label(&status.text)`, drag the mouse
   (`drag_screen`, `PointerButton::Primary`) from the left edge of the first line to the
   right edge of the last one.
4. Send `Event::Copy`, run a frame.
5. Check:
   - `platform_output.commands` contains `CopyText` equal to the selected text;
   - `status.text` and `status.error` have not changed;
   - the element `selection` is empty and has not changed;
   - the status does not contain the message `select what to copy first`.

### T2. The Copy button copies the whole message

1. Get an error (for example as in T1).
2. Remember `status.text`.
3. Find the **Copy** button specifically inside the status box (by its position in the
   bottom-right corner of the view, so as not to confuse it with the Copy button on the
   ribbon/palette).
4. Press it (if necessary `click_accesskit`, as in the #53 test).
5. Check:
   - `CopyText` equals the full `status.text`, character for character;
   - `status.text` has not changed and the box has not closed;
   - `clip_text` and `copied` have not changed.

### T3. Both scenarios in turn: text, then elements again

1. Select a wall, get an error, select the error text.
2. Press Ctrl+C → the text went to the clipboard, the element count and `selection` have
   not changed.
3. Click an empty spot of the plan outside the text — egui clears the text selection.
4. Press Ctrl+C.
5. Check that the system clipboard holds the BIMCraft text
   (`bimcraft_core::clipboard::is_clipboard`) and `status.error` is `false`.
   This proves the global model copying from #473 is not broken:
   it returns immediately after the text selection is cleared.

### T4. Ctrl+C without elements and without text gives the old message

1. Select nothing, do not select text.
2. Press Ctrl+C.
3. Check the exact text `select what to copy first` and `error == true`.

### T5. Ctrl+X with selected text does not cut the model

1. Select a wall, raise an error, select the error text.
2. Send `Event::Cut`.
3. Check that the text was copied, the wall count has not changed, `selection` has not
   changed.

### T6. The Note box behaves the same

Repeat T1/T2 for `status.error == false` (title `Note`).

### T7. i18n

`cargo i18n check` passes; the `.pot` contains `Copy the message to the clipboard`;
`Copy` is reused from the existing catalog.

### T8. The explicit Copy command from the ribbon copies the model even with selected text

1. Select a wall, get an error, select the error text.
2. Press the **Copy** button on the ribbon (not the button in the error box).
3. Check that the clipboard holds the BIMCraft text (`is_clipboard`), not the error text,
   and that `selection` has not changed.
4. Make sure the error box is still on screen.

This locks in the agreement: Ctrl+C follows an explicit text selection, while an explicit
Copy command always copies the model.

### T9. The already existing selectable labels also get Ctrl+C

Take any existing `selectable(true)` label, for example an ID in Inquiry
(`inquiry.rs:173-175`), select it with the mouse and press Ctrl+C. Check that the selected
text was copied and that `select what to copy first` was not shown.
This is not a separate feature but a consequence of the common `has_selection()` check;
the test guards against the return of the global interception.

## 9. Acceptance criteria

- [ ] The error is displayed as before (bottom-right corner, Error title, Dismiss).
- [ ] The error text is selected with the mouse, and the selection is visible.
- [ ] Ctrl+C copies the selected fragment and does **not** write `select what to copy first`.
- [ ] After Ctrl+C the original error stays on screen unchanged.
- [ ] The box has a **Copy** button; one click copies the whole message.
- [ ] The Copy button does not close the box and does not change its text.
- [ ] Ctrl+C without a text selection copies elements as before.
- [ ] After a click outside the selected text Ctrl+C copies elements again (verified by
      test T3: both scenarios work in turn).
- [ ] The explicit Copy command from the ribbon/menu copies elements even with selected
      text.
- [ ] Ctrl+X and Ctrl+V have not changed their behaviour from #473.
- [ ] The existing selectable labels (Inquiry, plugins, Markdown) also copy text with
      Ctrl+C.
- [ ] Ctrl+C without elements and without text gives the old `select what to copy first`.
- [ ] Ctrl+V after copying the error text does not try to paste that text as BIMCraft
      content (the internal `clip_text` is not overwritten).
- [ ] The Note box behaves the same as the Error box.
- [ ] `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
      `cargo clippy -p bimcraft-ui-egui`, `cargo i18n check` pass.
- [ ] Tests were not run locally unless the user asked; this is reflected in the PR.

## 10. Risks and open questions

1. **Text selection is global.** `LabelSelectionState` is shared across the application:
   if text is selected elsewhere (a plugin, a report, Markdown), Ctrl+C will copy the text
   rather than the elements. This is the expected behaviour: an explicit text selection is
   more important than a background element selection; a click outside the text clears the
   selection.
2. **A one-frame stale state.** `has_selection()` is read from the previous frame's state.
   If the label disappeared together with the selection, a one-frame skip of Ctrl+C is
   possible. There is no practical impact: the box lives until the next successful command.
3. **Dependence on the egui API.** `ctx.plugin_opt::<LabelSelectionState>()` is public but
   tied to the API version. The version is pinned to 0.36.2; re-check the signature when
   updating egui. The fallback is a read-only `TextEdit`, but it is worse in terms of focus
   and the keyboard.
4. **Box width.** The Copy button adds ~50 px to the title row. At 380 px the `Error` title
   is short, so there is room; check on a narrow window and in a long locale (German/
   French `Kopieren`/`Copier`).
5. **Selection across wrapped lines.** egui supports it, but test T1 must drag the mouse
   across the whole label rectangle, not along one line.
6. **Translating the hint.** New `.po` entries will appear empty after
   `cargo i18n update`; this is normal, the UI will show the English text until it is
   translated. It does not block acceptance.

## 11. Order of work and DoD

1. In `warning()` enable `selectable(true)` and add the Copy button.
2. In `clipboard_events()` add the `LabelSelectionState::has_selection()` check
   and leave Copy/Cut in `i.events` only when text is selected.
3. Add the hint string, run `cargo i18n update`.
4. Add tests T1–T9.
5. Run `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
   `cargo clippy -p bimcraft-ui-egui`, `cargo i18n check`.
6. Do not run tests without an explicit request; state in the PR that CI will run them.
7. Split into commits:
   - `ui: select and copy the status message`;
   - `ui: keep Copy/Cut for a label's text selection`;
   - `i18n: the status copy tooltip`;
   - `ui: tests for copying the status message`.
8. Definition of Done: all acceptance criteria are met, the element behaviour and Ctrl+V
   have not changed, there are no new dependencies and no `unsafe`.

## 12. What not to do

- Do not replace `status` with `Copied` after pressing the button: that would destroy the
  message.
- Do not write the error text into `self.copied`/`self.clip_text`: that is BIMCraft's
  internal buffer, and Ctrl+V would start pasting the error text as a model.
- Do not clear the element selection when copying text.
- Do not make all labels of the application selectable globally.
- Do not change the wording of `select what to copy first`.
- Do not add a dependency for the clipboard: `ctx.copy_text` already exists.
- Do not touch the model, the file format, commands and the API.
