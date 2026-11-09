# SPEC-010. Dimension Chain Editing: a Value Moves the Chosen Reference

Status: **proposal, not implemented**
Date: 2026-11-09
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: discussion [IPedrax/BIMCraft#228 "Top modeling and drawing features missing"](https://github.com/IPedrax/BIMCraft/discussions/228), items 1 and 1.1 and the GIFs attached to them; the refinements for highlighting, the empty input field, the text icon and wall length come from the user's edits to this specification
Related documents: `README.md` (Annotation section, command rows `dimension` and `dimension_value`), `BIMCraft/docs/areas/desktop.md`
Task type: fix the behavior of an existing feature (dimensions and constraints), UX improvement, tests.

> **Summary.** Today `dimension_value` always moves the **second** reference of a segment, regardless of which element the user selected. For a dimension with `EQ`, pressing the button does not equalize the chain in the tested wall scenario, and typing a value does not make all segments equal. A dimension along one wall cannot change that wall's length, the dimension-text window opens only through a non-obvious double-click, and the references of an existing dimension cannot be changed. Below are the scenarios, requirements and change points.

---

## 0. General working rules

- Commits are atomic: one item of the task per commit, each one builds and does not break existing behavior.
- Code, identifiers, code comments, command messages and commit messages are English. This document is the English translation of `ТЗ-010-правка-размерной-цепочки.md`; keep both files in sync when either changes.
- UI strings go only through `t()` / `tf()` / `msg!()`; add new strings to `translations/ui/bimcraft-ui.pot` and to every `translations/ui/*.po`.
- Cover every behavior change with a test: core — `crates/core/src/dimension_tests.rs` (or a new module), UI — `egui_kittest` in `crates/ui-egui/src/tests/dimensions.rs`.
- Do not change the `.bimcraft` file format: `Dimension` already has all the fields needed (`refs`, `locked`, `texts`, `equal`).
- Do not use subagents or parallel agents: the work is done in the current session.
- Before editing, check `git status` and `git diff` so that unrelated files do not get into the commit.

---

## 1. The problem

### 1.1. Scenarios and expected behavior

| No. | Scenario | Current behavior | Expected behavior |
|---|---|---|---|
| 1 | A wrong reference is picked while building a chain | Clicking an already picked reference/face again does nothing; `Esc` clears the **whole** chain | Clicking a picked face again **removes** it; `Backspace` removes the last one; `Esc` still clears the chain |
| 2 | An element attached to a dimension is selected | The text of the attached chain is not highlighted; the dimension must be selected separately to edit its number | The text of a chain whose references include the selected element is highlighted as editable (slightly smaller and dark blue, as in Revit). Clicking the highlighted number opens an input field; the value is typed from a blank field; Enter commits. After Enter the selected element moves so that its distance to the opposite reference becomes the typed value |
| 3 | `EQ` is pressed on a dimension chain | In the tested wall chain nothing happened; the walls did not equalize | The end elements stay in place, the inner ones move so that all distances inside the chain become equal |
| 4 | A value is typed into an `EQ` chain | Nothing happens | All segments become equal to the typed value. The chain re-spaces from one end reference: the opposite end reference moves so that the total length becomes `number of segments × value` |
| 5 | An existing dimension must be corrected (a reference added/removed) | Not possible: the `refs` set cannot be changed after creation | Context menu "Edit Dimension References" and the `dimension {id, refs?}` command: references are added and removed, and `locked`/`texts` move to their reference pairs |
| 6 | Dimension text (prefix/suffix/above/below/replace) | Opens only by double-clicking the number, which is not obvious | With the dimension selected, a small `T`/`Aa` icon appears under every number; clicking it opens the existing `Dimension Text` window. Double-click may remain an additional path, but not the only one |
| 7 | Changing the length dimension of one wall (from its start to its end) | `dimension_value` moves the **whole** wall; the distance between its ends does not change, although the command reports success | 1) both ends free — the wall changes length on both sides by the same amount; 2) one end attached — only the free end moves; 3) both ends attached — the wall extends/shrinks by the same amount on both sides |
| 8 | Thickness dimension of one wall (between its left and right faces) | Same: the wall is translated as a whole, the thickness does not change, and the command silently "succeeds" | Typing a value is forbidden: thickness is changed in the wall properties, not by a dimension. The command returns a clear error and the model does not change |

### 1.2. Why it matters

Without this, a complete plan cannot be dimensioned: the user cannot fix a wrong reference, cannot change a wall's length or position by typing a number, cannot build equal bays, and cannot open dimension text in an obvious way. This is basic Revit mechanics, and it is exactly what the current version lacks.

---

## 2. Terms

- **Reference** — a line or point a dimension measures to: a wall face/centre line/end, a door or window jamb, a column face, a grid line, a line end. In code — `Reference { id, face, point }`.
- **Station** — the point where the dimension line crosses a reference, ordered along the line. In code — the elements `st: Vec<(f64, Line, Reference)>` from `Model::stations`.
- **Segment** — a pair of adjacent stations `i` and `i + 1`; its length is `st[i+1].0 - st[i].0`.
- **Moving reference (mover)** — the station whose element moves when a value is typed.
- **Anchor (hold)** — the station that stays in place; the moving one is the opposite station of the segment.
- **End reference** — the first or the last station of a chain; for `EQ` only these may be anchors.
- **Chain** — a dimension with three or more references (two or more segments).
- **`EQ` / `equal`** — all segments of the dimension must be equal.
- **Lock (`locked`)** — a segment is fixed at a length; after every command the solver keeps it.
- **Attached end** — a wall end joined to another wall; see F5 for detail.

---

## 3. Functional requirements

### 3.1. F1. `dimension_value` gets a reference choice

Add an optional `hold` parameter:

```
{id, value, segment?, hold?: "start" | "end" | "both" | Reference}
```

- `hold` absent or `"start"` — the station at `segment` stays, `segment + 1` moves. This is the current behavior, kept for backward compatibility.
- `hold: "end"` — the station at `segment + 1` stays, `segment` moves.
- `hold: {id, face?, point?}` — the station whose `Reference` matches the given one stays; the other station of the segment moves. If the given reference is not one of the segment's two stations, an error: `"hold" is a reference this segment ends at, not …`.
- `hold: "both"` — only for `equal`: both end references of the chain stay, and the inner ones are spread evenly. If `value` is not equal to the current total length divided by the number of segments, an error (see 3.3.2).
- Add the actual values to the reply: `{id, moved, by, values}`, where `values` is the result of `Model::dimension_values` after the edit (useful both for the UI and for tests).

Requirements for `hold`:

- for `DimKind::Aligned | DimKind::Linear` and not `alignment` (as today);
- `value` is a positive length, as today;
- with `equal`, `hold` names an **end** reference of the chain, or both end references (see 3.3.2), not an inner station.

### 3.2. F2. Two selection modes: element and dimension

These are two different states, and they do not contradict each other.

#### 3.2.1. An element is selected (not a dimension)

- If the selection is exactly one element (wall, column, grid, door/window, etc.), find the dimensions in the current view whose `refs` include that element.
- The texts of those dimensions are drawn differently on screen: about 10% smaller and in `accent_ink` (dark blue in the light theme, `#2064B5`). This is screen-only: PNG, printing and sheets do not change.
- If the element belongs to several chains, all are highlighted; the chain nearest the cursor is highlighted more strongly.
- Clicking a highlighted number:
  - remembers the selected element as the mover (`pending_mover`) and does not switch the selection to the dimension;
  - opens the number input field for that segment;
  - the field is **empty**; the current value is not prefilled;
  - `Enter` or focus loss commits; `Esc` cancels; `Enter` in an empty field cancels;
  - after the commit, `dimension_value` is called with `hold` naming the opposite station; the selected element moves so that the distance to it becomes the typed value; the selection stays on the element.
- If the selected element supplies both stations of the segment (the length or thickness of one wall), the F5/F6 rules apply.

#### 3.2.2. A dimension is selected

- The numbers are **not highlighted**. This is expected: highlighting is a property of the "element selected" mode, not of the "dimension selected" mode.
- Locks and `EQ` are shown as today; the number can be edited.
- Reference handles (if they remain) are a secondary way to choose the mover: clicking a handle at a station makes it the mover for the adjacent segments; clicking it again clears the choice. The number highlight is not enabled by this.

#### 3.2.3. Priority of choosing the mover

1. If a station handle is selected — the element of that station moves.
2. Otherwise, if the selection contains a reference element (mode 3.2.1) — it moves.
3. Otherwise — the default: `hold = "start"`, the second reference of the segment moves.

#### 3.2.4. Number input field

- Common to permanent and temporary dimensions: on click the field is empty, not prefilled with the current value.
- `Enter` and focus loss commit; `Esc` cancels; an empty `Enter` changes nothing.
- If the typed string is not a length — a status error, the model does not change.

### 3.3. F3. `EQ`: pressing and typing a value are different actions

#### 3.3.1. Pressing `EQ` (without typing a value)

- The end references (the first and the last station) stay in place.
- The inner stations move so that all segments are equal:
  `pos[t] = pos[0] + t * (pos[k] - pos[0]) / k`, where `k` is the number of segments.
- This is possible whenever the inner elements can move. If at least one inner element cannot move (it coincides with an end reference or is held by a lock or a join), the command returns a clear error and rolls back fully; partial equalization is not allowed.
- If the chain is already equal, nothing changes — that is normal.
- Today, in the tested wall chain, pressing `EQ` changed nothing. The cause has not been established; the requirement is: after the press the user must see either equalization or a clear error in the status bar, not silence. The test must reproduce a chain of walls, not only grids.

#### 3.3.2. Typing a value `V` into an `EQ` chain

- All segments become equal to `V`.
- For `EQ`, the `hold` values are redefined: `"start"` and `"end"` mean the **end references of the whole chain**, not the stations of the edited segment.
- The anchor is **one of the end references**:
  - `hold = "start"` — the first station stays, `pos[t] = pos[0] + t * V`;
  - `hold = "end"` — the last station stays, `pos[t] = pos[k] - (k - t) * V`.
- `hold = "both"` — both end references stay, and the inner ones are spread evenly. This is possible only when `V = (pos[k] - pos[0]) / k`; otherwise an error: `"an equal dimension's value would move both ends: …"`. This does not contradict 3.3.1: there no value is typed and the total length is preserved.
- `hold` with an inner station is not allowed for `EQ`: error `"an equal dimension is anchored at one of its ends"`.
- With `hold = "start"` or `"end"`, the opposite end reference moves by `k * V - (pos[k] - pos[0])`.
- Every lock of the dimension gets the length `V`; if they had different lengths, that is a state error — return it.
- After the moves and `constraints::keep`, every segment must equal `V` within `SLACK`.

#### 3.3.3. Choosing the anchor in the UI

- If an element is selected, the anchor is the end reference on the opposite side of the selected station, so that the selected element moves.
- If nothing is selected — `hold = "start"` (the first end reference).
- If the selected station is itself an end reference, the anchor becomes the opposite end reference.

### 3.4. F4. Result verification and full rollback

After the moves the command must:

1. take a snapshot of the model before the changes (`Arc::clone(&s.model)` — cheap, the model is in an `Arc`);
2. perform the moves and the lock updates;
3. call `crate::constraints::keep(s, &before)`;
4. recompute `Model::dimension_values` and check:
   - the edited segment's length equals `value` within `SLACK` (0.01 mm);
   - for `EQ` after a press — all segments equal the original total length divided by their number (`(pos[k] - pos[0]) / k` before the command), and the end references did not move;
   - for `EQ` after typing a value — all segments equal `V`;
   - every `locked` segment keeps its length;
5. if the check fails, return an error explaining which constraint prevented it. The engine rolls the model back to the state before the command (`engine.rs:488-494`), including the `locked` changes.

The error must name the concrete cause, for example:

- `"segment 1 can't be 2500: the wall at [x, y] is held by a locked segment of dimension 123456"`;
- `"the value can't be kept: what it measures to is joined to something that stays"`;
- `"segment 0 lies within one wall: its thickness is its type's"`;
- `"an equal dimension's value would move both ends: 2500 isn't 9000 / 3"`.

Calling `keep` inside the command does not break the engine: the outer `keep` after the command sees that the constraints already hold and moves nothing. This must be covered by a test that the number of undo steps does not double.

### 3.5. F5. Length of one wall

Scenario: the dimension sits between the `point: start` and `point: end` points of one wall.

**What counts as attached.** A wall end is attached if it is joined to another wall: `Wall.join` is true and at the node there is another wall with a shared join over their height. Determine this through `walls::Index::neighbours_at` and `walls::stacked`; add a helper `walls::joined_at(m, id, at_end, index) -> bool`. A wall end merely touching a column or a grid line does not count as attached — this must be stated in the documentation.

Rules:

1. **Both ends free** — both ends move by `delta / 2` in opposite directions, and the length changes by `delta = value - current`. The middle of the wall stays in place.
2. **One end attached** — only the free end moves, by `delta`.
3. **Both ends attached** — both ends move by `delta / 2`, as in case 1; the joined walls stretch/shrink.

Technically:

- for one end — `move` with `end: "start" | "end"`;
- for two — two `move` calls with `end` inside one `dimension_value` command (one undo step), or a dedicated `resize_wall` helper;
- after the moves — verification per F4: the length equals `value`, the free ends moved as expected, and the attached ends stayed in place;
- for a curved wall, the behavior of `relocate` with `end` must be checked by a separate test; if it changes the arc unpredictably, this specification is limited to straight walls and returns a clear error for curved ones.

### 3.6. F6. Thickness, door/window width, column size

If both stations of a segment belong to one element, but this is not the length of a wall/run:

1. **Wall thickness** (references — `Face::Left` and `Face::Right` of one wall). Thickness is stored in `WallType::thickness` and is shared by all instances of the type. Typing a value is forbidden. Error: `"segment 0 lies across one wall: its thickness is its Wall Type's, not the dimension's to set"`.
2. **Door/window width** (the jambs of one door/window) and **column size** (the caps of one column). A dimension inside one element cannot be changed by translation. Error with a hint to change the element's type/parameters.
3. On error the model does not change.

### 3.7. F7. Dimension text: an icon instead of double-click

- Today `DimensionTextDialog` opens by double-clicking the number (`canvas.rs:1676-1683`); this is not obvious.
- When the dimension is selected (and while the number field is open), show a small `T` or `Aa` icon under every number (screen-only, not printed).
- Clicking the icon opens the existing `DimensionTextDialog` for that segment (`dimtypes.rs:242-333`).
- Double-click may remain an additional shortcut, but it must not be the only path. If double-click conflicts with the number field, remove it.
- The dialog fields already exist: Prefix, Suffix, Above, Below, Replace With Text. An empty field restores the true number.
- Display priority: a segment's `replace` wins over the `EQ` text, otherwise `EQ`, otherwise the true number. Today in `Model::dimension_labels` (`model.rs:5656-5687`) the `equal` branch comes before `replace` — change the order and cover it with a test.
- No new command is needed: the existing `dimension_text` is used.

### 3.8. F8. Editing the reference set and preserving `locked`/`texts`

Extend the `dimension` command:

```
{id?, refs?, at?, kind?, orient?, type?, prefix?, suffix?, above?, below?, replace?, equality?, level?}
```

- without `id` — create, as today;
- with `id` — edit an existing dimension:
  - `refs` replaces the reference set (at least 2, the same parallelism rules for `aligned`/`linear`);
  - `at` moves the dimension line;
  - `kind`, `orient`, `type`, the text fields and `equality` change when given;
  - `level`/`view` cannot be changed: the dimension stays in its view;
  - a dimension created by `Align` (`alignment == true`) cannot be edited: error `"an alignment lock's dimension can't be edited: unlock it first"`.

When replacing `refs`:

1. build the old stations and segment keys — an unordered pair of references `(Reference, Reference)`;
2. build the new stations and their pairs;
3. for every `locked`/`texts` entry, find the new segment with the same reference pair and move the entry to its index;
4. drop entries whose pair disappeared;
5. if fewer than two segments remain and `equal == true` — an error;
6. if `refs` contains duplicates — an error.

Reply: `{id, values, refs}` — the new set and the segment lengths.

Editing UI:

- for a selected dimension, a right-click context menu item "Edit Dimension References";
- the tool enters edit mode: the existing references are highlighted, clicking an element adds a reference, clicking an already selected one removes it, clicking aside fixes the new dimension-line position;
- `Esc` cancels the edit and the dimension stays as it was;
- after confirmation, `dimension {id, refs, at}` is called; one command — one undo step;
- do not use double-click for this mode: it is taken by the text window (F7).

### 3.9. F9. Removing a wrong reference while building

In `Tool::Dimension` (`canvas.rs:2213`):

- clicking an already selected reference **removes** it from `refs` (toggle) instead of ignoring the click;
- `Backspace` (when the length input field is not active) removes the last selected reference;
- `Esc` still clears the whole chain (keep the current behavior);
- when the removed reference was the only one, the hint returns to "pick the first reference" (`lib.rs:4946-4953`);
- the same behavior is allowed for `Tool::Angular` and `Tool::Round`, but only `Tool::Dimension` is mandatory in this specification.

Update the hints:

- `"Click a picked reference again to take it out, Backspace takes out the last, Esc starts over."`
- with an empty list — as today.

### 3.10. F10. Messages and predictability

All new errors are short, with the dimension id and the segment number. Examples:

- `"dimension 123456 has segments 0 to 2, not 3"`
- `"hold is a reference this segment ends at, not a wall's centre line"`
- `"an equal dimension is anchored at one of its ends"`
- `"hold both is for an equal dimension"`
- `"segment 0 lies within one wall: its thickness is its Wall Type's, not the dimension's to set"`
- `"segment 1 can't be 2500: a locked segment of dimension 123456 keeps what it measures at 3000"`
- `"the value can't be kept: what it measures to is joined to something that stays"`

The UI status bar on error goes through the existing `Status { text, error: true }`, with the text from the command reply (as today in `canvas.rs:3340`).

### 3.11. Change map

Not a cause analysis, just a short map: what to change and where.

| What | File | Reference |
|---|---|---|
| `dimension_value`: `hold`, station choice, verification, rollback | `crates/core/src/commands.rs` | ~12382 |
| one-wall length: 3 cases, `move` with `end` | `crates/core/src/commands.rs` | ~12382, ~11767 (`relocate`) |
| `EQ`: press and value input, locks | `crates/core/src/constraints.rs` | ~28 (`targets`), ~112 (`keep`) |
| `stations`, `dimension_values`, `dimension_labels` (`replace`/`EQ` order) | `crates/core/src/model.rs` | ~5631, ~5717, ~5656 |
| `dimension {id, refs}`: editing a dimension | `crates/core/src/commands.rs` | ~8176 |
| attached-end predicate | `crates/core/src/walls.rs`, `walls/index.rs` | add `joined_at` |
| highlight, empty field, text icon, reference toggle | `crates/ui-egui/src/canvas.rs` | ~1676, ~2213, ~3257, ~3290 |
| `Dimension Text` window | `crates/ui-egui/src/dimtypes.rs` | ~242 |
| keys and hints | `crates/ui-egui/src/lib.rs` | ~2008, ~4946 |
| translations and README | `translations/ui/*`, `README.md` | rows 247, 383, 517–519 |

---

## 4. Pitfalls

1. **Station ordering.** `refs` in the file may be in any order; segments are counted by the sorted stations. Never use an index in `refs` as a segment index.
2. **One element — two references.** A wall can supply both the left and the right face, and both ends. The "one element" check must compare station `id`s, not the whole `Reference`. For length — move the ends; for thickness — refuse.
3. **Attached end.** `Wall.join` alone is not enough: there must be a real join with another wall at the node. Touching a column/grid must not hold the end unless agreed separately.
4. **Wall joins.** `relocate` with `end` pulls the joined walls; after the move the actual length may differ from `by`. Verification per F4 is mandatory, otherwise we get a "silent success" again.
5. **Doors and windows.** Moving a door/window slides it along the wall; the distance to its jamb may change by something other than `by` if the jamb hits the wall end. Verify the result.
6. **Neighboring locks.** Moving a reference shared by two segments may require moving other elements to keep the second (locked) segment. The `keep` solver does this, but on conflict it must roll back with a clear error.
7. **`EQ` + locks.** `dimension_constraint` already rejects `equal` when the locks differ in length. When typing a value into `EQ`, all locks must get the same value; if that is impossible — an error.
8. **Pressing `EQ` and silence.** In the wall test the press produced no visible result. During implementation check: is the `EQ` button available (3+ stations are required), is the chain already equal, does `keep` reject the command because of joins. Either way the user must see a result or an error.
9. **`keep` idempotence.** After an `EQ` re-space the outer `keep` must not move the elements again. Cover with a test on 3 and 4 segments.
10. **Undo.** One value edit — one undo step. The internal `keep` call must not create a second step.
11. **Oracle tests.** `crates/core/src/oracle_tests.rs` generates `dimension_value`; the new `hold` values and errors must not break the `Model::check` invariants.
12. **`dimension_segment_at`.** After editing `refs` the segment numbers change; texts and locks must move with their reference pairs, otherwise a click on a number lands on the wrong segment.
13. **Curved walls.** For a curved wall the point references and the chord may behave differently; this specification only needs to cover straight walls and return a clear error for curved ones if `relocate` with `end` changes the arc unpredictably.
14. **Read-only mode.** The command is undoable anyway; in `read_only` the engine rejects it, so no extra checks are needed.
15. **Two selection modes.** Numbers highlighted when an element is selected and not highlighted when a dimension is selected is not a conflict but two states. The test must cover both.

---

## 5. Tests

### 5.1. Core (`crates/core`)

New module `crates/core/src/dimension_edit_tests.rs` (or an addition to `dimension_tests.rs`):

1. `dimension_value_moves_the_second_reference_by_default` — backward compatibility (rewrite the existing test `constraints.rs:214`).
2. `dimension_value_moves_the_first_reference_with_hold_end` — two walls; `hold: "end"`; the first wall moved, the second stayed; the value is exact.
3. `dimension_value_moves_the_reference_named_by_hold` — `hold: {id, face}`; verify that exactly that reference moves.
4. `dimension_value_refuses_a_hold_that_isnt_an_end_of_the_segment` — expected error.
5. `dimension_value_resizes_a_free_wall_symmetrically` — both ends free; enter 6000; both ends moved by `(6000 - old) / 2`, the middle stays.
6. `dimension_value_resizes_a_wall_joined_at_one_end` — one end joined; only the free end moves.
7. `dimension_value_resizes_a_wall_joined_at_both_ends` — both ends joined; both move by `delta / 2`; the joined walls do not break.
8. `dimension_value_refuses_the_two_faces_of_one_wall` — thickness dimension; the error mentions `thickness`/`type`; the wall did not change.
9. `dimension_value_refuses_a_door_width` — the jambs of one door; error; the door did not change.
10. `pressing_equal_keeps_the_ends_and_spreads_inner_references_evenly` — a chain of walls, not grids; after `dimension_constraint {equal: true}` the end walls stay and the inner ones are at equal distances; an already equal chain does not change.
11. `dimension_value_with_equal_sets_every_segment_and_anchors_the_first_end` — 4 stations, `hold: "start"`; all segments equal `V`; the first end reference stays and the last moved by `k * V - total`.
12. `dimension_value_with_equal_anchors_the_last_end` — `hold: "end"`; the last stays, the first moved.
13. `dimension_value_with_equal_both_ends` — `hold: "both"`: with `V = total / k` the end references stay and the inner ones are spread; with `V != total / k` — expected error, the model does not change.
14. `dimension_value_with_equal_and_locks_updates_every_lock` — every lock gets the new value; after a `move` the chain stays equal.
15. `dimension_value_rolls_back_when_a_neighbouring_lock_conflicts` — error; the model (including `locked`) is identical to the state before the command.
16. `editing_refs_remaps_locks_and_texts` — 4 references, a lock on segment 1, text on segment 2; remove an end reference; the lock and the text moved to the same reference pairs.
17. `editing_refs_drops_locks_of_a_removed_pair` — the lock of the removed pair disappeared, the others stayed.
18. `editing_refs_refuses_an_alignment_lock` — error.
19. `editing_refs_refuses_fewer_than_two_refs` and `refuses_duplicates`.
20. `editing_refs_keeps_equal_and_its_segments` — `equal` is preserved and the number of segments is >= 2, otherwise an error.
21. `a_segment_text_override_replaces_the_number_and_beats_eq` — `dimension_text {replace}` wins over the `EQ` text; an empty `replace` restores the true number.

Additionally:

- `crates/core/src/constraints.rs` — update the test about the second reference and add a test for `keep` idempotence after an `EQ` re-space.
- `crates/core/src/oracle_tests.rs` — add `hold` to the random `dimension_value` parameters so the generator covers both directions.

### 5.2. UI (`crates/ui-egui/src/tests/dimensions.rs`, `egui_kittest`)

1. `clicking_a_picked_reference_takes_it_out` — build a dimension: two references, a third wrong one, click it again; two remain; the dimension is created.
2. `backspace_takes_out_the_last_reference` — after three clicks `Backspace` leaves two.
3. `selecting_a_wall_highlights_its_dimension_and_enter_moves_it` — an element is selected; the text of the attached dimension is highlighted; clicking the number gives an empty field; after Enter the selected wall moved and the opposite one stayed.
4. `selecting_a_dimension_does_not_highlight_its_numbers` — a dimension is selected; there is no number highlight; the number field still works.
5. `pressing_eq_on_a_wall_chain_moves_inner_walls_and_keeps_ends` — a chain of walls; after pressing `EQ` the ends stay and the inner ones are equalized.
6. `typing_a_value_on_an_equal_chain_updates_every_number` — after typing, all on-screen labels are equal.
7. `text_icon_under_the_number_opens_the_text_dialog` — the `T`/`Aa` icon is visible on the selected dimension; clicking opens the window; `Esc`/Cancel change nothing.
8. `clearing_replace_restores_the_true_number` — typed text replaces the number; clearing restores the true one.
9. `edit_dimension_references_from_the_context_menu` — enter the mode, remove a reference, confirm, the new dimension and segment count.
10. `esc_leaves_an_edited_dimension_as_it_was`.

### 5.3. Quality control

- `cargo test --workspace`;
- `cargo clippy --workspace --all-targets -- -D warnings`;
- `cargo fmt --check`;
- oracle tests and `Model::check` after every command (already built into the engine).

---

## 6. Documentation and translations

1. `README.md`:
   - row 247 (Annotation) — the number moves the chosen reference; pressing `EQ` equalizes with fixed ends; typing a value re-spaces the chain from an end reference; dimension text opens through the icon;
   - row 383 (Dimension, tag) — clicking a reference again removes it; editing an existing dimension; the text icon;
   - row 517 (`dimension`) — the `id` parameter and the edit rules;
   - row 519 (`dimension_value`) — the `hold` parameter.
2. `BIMCraft/docs/areas/desktop.md` — the section about dimensions, highlighting and the context menu.
3. `crates/core/src/commands.rs` — the registry descriptions of `dimension` and `dimension_value`.
4. `translations/ui/bimcraft-ui.pot` and every `translations/ui/*.po` — new strings:
   - `"Click a picked reference again to take it out, Backspace takes out the last, Esc starts over."`
   - `"Edit Dimension References"`
   - `"Dimension text"` (icon label)
   - `"Keep this reference"` / `"Move this reference"` (if handles get labels)
5. `docs/areas/views-and-graphics.md`, if it has a dimension section — synchronize it.

---

## 7. Acceptance criteria

1. With an element selected, the text of the attached chain is highlighted (smaller and `accent_ink`); with a dimension selected, there is no highlight.
2. Clicking a highlighted number opens an empty field; Enter commits; Esc cancels; the selected element moves and the opposite reference stays.
3. Pressing `EQ` keeps the end elements in place and equalizes the inner distances; when impossible — a clear error, not silence.
4. Typing a value into an `EQ` chain makes all segments equal to `V`; one end reference moves; if both ends must stay, `V` must equal `total / k`, otherwise an error.
5. A one-wall length dimension works in all three cases: both ends free — symmetric; one attached — the free one; both attached — symmetric.
6. A one-wall thickness dimension does not "silently succeed": it returns a clear error and the model does not change.
7. The dimension-text window opens through the icon under the number; double-click is not the only path; `replace` wins over the `EQ` text; clearing restores the true number.
8. A wrong reference can be removed by clicking it again or with `Backspace`; `Esc` clears the chain as before.
9. The reference set of an existing dimension can be changed; `locked` and `texts` stay on their reference pairs, and the orphaned ones are dropped.
10. Any failed edit rolls back fully (model, locks, undo).
11. All existing dimension and constraint tests pass; the new tests from section 5 are added.
12. `cargo test --workspace`, `cargo clippy`, `cargo fmt --check` — no errors.
13. The `.bimcraft` format did not change; old projects open.
14. README, command descriptions and translations are updated.

---

## 8. Manual verification

1. **Scenario 2.** Draw two parallel walls and place an aligned dimension between them. Select the first wall → the dimension text is highlighted. Click the number → the field is empty; type a new number and press Enter → the first wall moved, the second stayed.
2. **Scenario 2, cancel.** The same, but press Esc after clicking the number → nothing changed, the selection stayed on the wall.
3. **Scenario 3.** Build a three-segment dimension between walls, select the dimension, press `EQ` → the end walls stay and the inner ones are equalized.
4. **Scenario 4.** On the same chain, type a value into the middle segment → all three numbers become the typed value; one end wall moved; pressing `EQ` again changes nothing.
5. **Scenario 1.** Start a dimension, pick two references and a third wrong one; click the third again (it disappears); use `Backspace` to remove the last one; place the dimension.
6. **Scenario 5.** Select an existing dimension → context menu → Edit Dimension References → remove one reference → confirm; numbers, locks and texts are updated.
7. **Scenario 6.** Select a dimension → the `T`/`Aa` icon is visible under the number; clicking opens the text window; enter Replace With Text → the number is replaced; clear it → the true number returns.
8. **Wall length, both ends free.** Dimension from the wall start to its end; enter 6000 → both ends moved by the same amount, the middle stays.
9. **Wall length, one end attached.** Join one end to another wall; enter a new value → only the free end moves.
10. **Wall length, both ends attached.** Close the wall into a corner on both sides; enter a new value → both ends move by the same amount and the joins do not break.
11. **Wall thickness.** Dimension between the left and right faces of one wall; try to enter a number → an error about the wall type, the wall did not change.
12. **Conflict.** Lock two adjacent segments and try to change one → an error, nothing changed; the rollback is automatic.
13. **Undo.** After every successful edit, one `Ctrl+Z` restores both the number and the element positions.

---

## 9. Out of scope

- Changing wall thickness through a dimension (changing `WallType` would affect every instance of the type).
- Changing door/window width and column size through a dimension.
- Editing the values of angular, radial and diameter dimensions, and arc length.
- 3D dimensions (`dimension_3d`).
- A global constraint solver across several dimensions (today each dimension is solved separately; this is a deliberate limitation).
- Automatic wall dimensioning and automatic dimension chains.
- Persisting the chosen moving reference in the project file.
- Multi-selecting several moving elements.
- A wall end attached to a column/grid as a holding factor (this specification covers only wall-to-wall joins).

---

## 10. Decisions and open questions

Decisions:

1. **The parameter is called `hold`, not `move`.** This way the command states which reference stays and which moves; the default `"start"` keeps backward compatibility.
2. **The two selection modes do not conflict.** The number highlight is enabled only when an element is selected; with a dimension selected there is none.
3. **Pressing `EQ` and typing a value are different actions.** Pressing keeps both end references and equalizes at the current total length. Typing a value makes all segments equal to `V` and moves one end reference by default; the `hold: "both"` mode keeps both but requires `V = total / k`, otherwise it returns an error.
4. **For one wall the length changes in three cases (free / one attached / both attached), and thickness is refused.** Changing the wall type from a dimension is out of scope for this specification.
5. **Dimension text opens through the icon under the number.** The existing `DimensionTextDialog` is reused; `replace` wins over the `EQ` text.
6. **Result verification happens inside `dimension_value` after `keep`.** The model is in an `Arc` and the snapshot is cheap; the engine performs the rollback.
7. **The file format does not change.**

Open questions (to resolve during implementation, not blocking):

1. Reference handles: keep them as a secondary way or remove them now that element selection exists? Proposal — keep them, but do not highlight numbers when a dimension is selected.
2. Highlight all chains of the selected element or only the one nearest the cursor? Proposal — all, with the nearest stronger.
3. Keep double-click as a shortcut for the text window? Proposal — keep it, but the icon is mandatory and documented.
4. Should `dimension_value` return the explicit `values` parameter for every command or only for this one? Proposal — only for this one, for tests and the UI.

---

## 11. Work plan (atomic commits)

1. **core: `dimension_value` moves the chosen reference.**
   - the `hold` parameter, station choice, backward compatibility;
   - result verification and rollback;
   - tests 1–4 from 5.1.
2. **core: one-wall length in three cases.**
   - the `walls::joined_at` helper;
   - one or two `move` calls with `end` inside one command;
   - refusal for thickness/width of one element;
   - tests 5–9 from 5.1.
3. **core: `EQ`.**
   - press: ends stay, inner references evenly;
   - value input: anchor at an end reference, all segments `V`;
   - locks, `keep` idempotence;
   - tests 10–15 from 5.1.
4. **core: editing the reference set `dimension {id, refs}`.**
   - moving `locked`/`texts` by reference pairs;
   - tests 16–20 from 5.1.
5. **core: dimension text.**
   - `replace` wins over the `EQ` text in `dimension_labels`;
   - test 21 from 5.1.
6. **ui: removing a wrong reference and `Backspace`.**
   - toggle in `Tool::Dimension`, hints, translations;
   - tests 1–2 from 5.2.
7. **ui: selected element — highlight and input.**
   - `accent_ink` and smaller text highlight;
   - empty field, Enter/Esc, `pending_mover`, passing `hold`;
   - tests 3–4 from 5.2.
8. **ui: `EQ` on screen.**
   - updating the numbers after a press and after typing;
   - tests 5–6 from 5.2.
9. **ui: text icon and reference editing.**
   - the `T`/`Aa` icon under the number, opening `DimensionTextDialog`;
   - the Edit Dimension References context menu;
   - tests 7–10 from 5.2.
10. **docs: README, command descriptions, `docs/areas`, translations.**
    - no behavior changes.

Every commit must build and pass its own set of tests; do not mix core and UI in one commit unless a UI test needs the new command.

---

## 12. Scenario traceability

| Scenario from 1.1 | Requirement | Tests |
|---|---|---|
| 1. Wrong reference | F9 | UI 1–2 |
| 2. Element selected → highlight and input | F2, item 3.2.1 | UI 3 |
| Mode: dimension selected → no highlight | F2, item 3.2.2 | UI 4 |
| 3. Pressing `EQ` | F3, item 3.3.1 | core 10, UI 5 |
| 4. Typing a value into `EQ` | F3, item 3.3.2 | core 11–13, UI 6 |
| 5. Editing the references of an existing dimension | F8 | core 16–20, UI 9 |
| 6. Dimension text through the icon | F7 | core 21, UI 7–8 |
| 7. Length of one wall | F5 | core 5–7, manual 8–10 |
| 8. Thickness of one wall | F6 | core 8, manual 11 |
