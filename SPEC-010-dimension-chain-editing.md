# SPEC-010. Dimension Chain Editing: a Value Moves the Chosen Reference

Status: **implemented**
Date: 2026-11-09
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: discussion [IPedrax/BIMCraft#228 "Top modeling and drawing features missing"](https://github.com/IPedrax/BIMCraft/discussions/228), items 1 and 1.1 and the GIFs attached to them; the refinements for highlighting, the empty input field, the text icon and wall length come from the user's edits to this specification
Related documents: `README.md` (Annotation section, command rows `dimension` and `dimension_value`), `BIMCraft/docs/areas/desktop.md`
Task type: fix the behavior of an existing feature (dimensions and constraints), UX improvement, tests.

> **Summary.** Today `dimension_value` always moves the **second** reference of a segment, regardless of which element the user selected. For a dimension with `EQ`, pressing the button does not equalize the chain in the tested wall scenario, and typing a value does not make all segments equal. A dimension along one wall cannot change that wall's length (nor when it measures to the wall's faces, corner handles or end faces), the dimension-text window opens only through a non-obvious double-click, and the references of an existing dimension cannot be changed from the Modify tab. Below are the scenarios, requirements and change points, including the later work: the temporary (virtual) dimension, chain-segment selection, the centred EQ control, open padlocks, perpendicular dragging, and reference picking by Tab.

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
| 5 | An existing dimension must be corrected (a reference added/removed) | Not possible: the `refs` set cannot be changed after creation; the editor is only in the context menu | "Edit Dimension References" in the context menu and on the Modify tab, and the `dimension {id, refs?}` command: picked references are removed by clicking them, other faces, axes and corners added, and `locked`/`texts` move to their reference pairs. A click in the open confirms and leaves, it does not move the line |
| 6 | Dimension text (prefix/suffix/above/below/replace) | Opens only by double-clicking the number, which is not obvious | With the dimension selected, a large framed `Aa` button appears over every number; clicking it opens the existing `Dimension Text` window. Double-click may remain an additional path, but not the only one |
| 7 | Changing the length dimension of one wall (from its start to its end) | `dimension_value` moves the **whole** wall; the distance between its ends does not change, although the command reports success | 1) both ends free — the wall changes length on both sides by the same amount; 2) one end attached — only the free end moves; 3) both ends attached — the wall extends/shrinks by the same amount on both sides |
| 8 | Thickness dimension of one wall (between its left and right faces) | Same: the wall is translated as a whole, the thickness does not change, and the command silently "succeeds" | Typing a value is forbidden: thickness is changed in the wall properties, not by a dimension. The command returns a clear error and the model does not change |
| 9 | Temporary (virtual) dimension of the selected wall | Only the length is shown and it is edited by moving the far end; it cannot be made permanent | Under the number there is a button that makes it a permanent dimension. The number is edited like a normal dimension: both ends free — both move by half the change; one attached — the free one; both attached — both move evenly in the direction of the chain |
| 10 | Stepping through segments of a dimension chain | A single segment cannot be selected; Tab only cycles elements | Tab over a dimension cycles its segments and highlights the current one; a left click selects a segment; Delete removes it while the chains either side stay on their own references (the chain splits, it does not stretch) |
| 11 | Text of an EQ dimension | With EQ on, the labels are replaced by `EQ` | Labels stay ordinary numbers; `EQ` appears over the middle of the chain when hovered and always while the dimension is selected (screen only). A click on it turns EQ on/off; the button to the right of the chain is gone |
| 12 | Padlock of a free segment | The lifted shackle is barely different from a shut padlock | A free segment wears a clearly open padlock (shackle swung up and out), a locked one a shut padlock |
| 13 | Permanent dimension from the temporary one where the wall joins perpendicular walls | The dimension binds to the selected wall's ends; editing moves only the joined end of the perpendicular wall, and it goes out of square | When an end is joined to a perpendicular wall, the dimension takes that wall's centre line as its reference; editing moves the whole perpendicular wall and it stays perpendicular |
| 14 | A free wall near a perpendicular wall | The real dimension binds to the nearby perpendicular wall even though the wall is joined to nothing | If the end is not joined to a perpendicular wall, the dimension stays between the wall's own ends; a nearby wall that is not joined is ignored |
| 15 | A click in the open while editing references | The dimension chain moves to the click point | A click in the open does not move the dimension, but confirms the edit and leaves the mode; the line moves only by dragging |
| 16 | Dragging a dimension chain | The dimension can be dragged sideways, off its perpendicular | The dimension moves only along the normal to its line; the sideways component of the drag is ignored |
| 17 | Binding several faces and corners of one element | Clicking another face removes the reference of the previous face | Every face, axis and corner binds separately: a click on a new face/corner adds a reference, a click on a bound one removes it |
| 18 | Tab through faces and axes while creating or editing a dimension | Hovering shows only one reference; Tab does not cycle the element's faces | Over an element Tab cycles its faces, axes and corners and highlights the reference a click would bind; it works both when creating a chain and when editing references |
| 19 | Edit References button on the Modify panel | References can only be edited through the context menu | A selected dimension offers Edit Dimension References on the Modify tab; it enters the bind/unbind mode, Enter confirms, Esc cancels |
| 20 | Dimension on the faces, handles or end faces of one wall | Typing a value gives "what it measures to is joined to something that stays" and the wall keeps its length | The wall's length changes when the references are its side faces, corner handles, end faces or a mix: the wall is nudged until the measured distance is the typed value; thickness is still not changed by a dimension |

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
- Locks are shown as today; the EQ control is the label over the middle of the chain (see 3.3.1) and a click toggles equality. The number can be edited.
- A segment lock reads without hovering: a locked segment wears a shut padlock, a free one an open padlock (shackle swung up and out).
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
- The EQ control itself is the label `EQ` over the middle of the dimension chain, on screen only (never in a PNG or on a sheet):
  - on an EQ dimension it is visible while the pointer is over the chain;
  - on a selected dimension it is always visible when the chain has two segments or more;
  - when EQ is off, the selected dimension wears it struck through; when on, it is not struck through;
  - a click on it turns EQ on or off, as the old button to the right did; that button is gone.

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

Scenario: the dimension measures one and the same wall and its direction runs **along** it. References may be on the centre line, a side face, a corner handle, an end face, or a mix of them: all of these are the wall's length.

**What counts as attached.** A wall end is attached if it is joined to another wall: `Wall.join` is true and at the node there is another wall with a shared join over their height. Determine this through `walls::Index::neighbours_at` and `walls::stacked`; add a helper `walls::joined_at(m, id, at_end, index) -> bool`. A wall end merely touching a column or a grid line does not count as attached — this must be stated in the documentation.

Rules:

1. **Both ends free** — both ends move by `delta / 2` in opposite directions, and the length changes by `delta = value - current`. The middle of the wall stays in place.
2. **One end attached** — only the free end moves, by `delta`.
3. **Both ends attached** — both ends move by `delta / 2`, as in case 1; the joined walls stretch/shrink.

Technically:

- for one end — `move` with `end: "start" | "end"`;
- for two — two `move` calls with `end` inside one `dimension_value` command (one undo step), or a dedicated `resize_wall` helper;
- "along the wall" is decided by the dimension's direction: `u × n ≈ 0` — length; `u · n ≈ 1` — thickness (refused); otherwise the error "spans one wall at an angle";
- corner handles and end faces of joined walls move slightly differently from a centre-line point (mitres, joins), so after the first `resize_wall` the command repeats the nudge until the measured distance is `value` within `SLACK`, then verifies per F4;
- after the moves — verification per F4: the length equals `value`, the free ends moved as expected, and the attached ends stayed in place;
- for a curved wall, the behavior of `relocate` with `end` must be checked by a separate test; if it changes the arc unpredictably, this specification is limited to straight walls and returns a clear error for curved ones.

### 3.6. F6. Thickness, door/window width, column size

If both stations of a segment belong to one element, but the dimension does not run along the element's length (its direction is across the wall, or the element is not a wall/run):

1. **Wall thickness** (references — `Face::Left` and `Face::Right` of one wall). Thickness is stored in `WallType::thickness` and is shared by all instances of the type. Typing a value is forbidden. Error: `"segment 0 lies across one wall: its thickness is its Wall Type's, not the dimension's to set"`.
2. **Door/window width** (the jambs of one door/window) and **column size** (the caps of one column). A dimension inside one element cannot be changed by translation. Error with a hint to change the element's type/parameters.
3. On error the model does not change.

### 3.7. F7. Dimension text: an icon instead of double-click

- Today `DimensionTextDialog` opens by double-clicking the number (`canvas.rs:1676-1683`); this is not obvious.
- When the dimension is selected (and while the number field is open), show a large framed `Aa` button over every number (screen-only, not printed).
- Clicking the button opens the existing `DimensionTextDialog` for that segment (`dimtypes.rs:242-333`).
- Double-click may remain an additional shortcut, but it must not be the only path. If double-click conflicts with the number field, remove it.
- The dialog fields already exist: Prefix, Suffix, Above, Below, Replace With Text. An empty field restores the true number.
- Display priority: a segment's `replace` wins over the true number. `EQ` no longer replaces numbers: on an `equal`/`equality` dimension the labels stay lengths, and the `EQ` label appears over the middle of the chain while the pointer is near it, and always on a selected dimension; a click toggles equality. It is screen-only, never in a PNG or on paper. `dimension_labels` no longer substitutes `equality_text`; the EQ text comes from the dimension type for that label.
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

- for a selected dimension, a right-click context menu item "Edit Dimension References" **and** a button on the Modify tab; pressing the button again leaves the mode;
- the tool enters edit mode: the existing references are ringed; clicking a bound reference (a specific face, axis or corner) removes it, clicking another face, axis or corner of the same element adds it (faces of one element are independent); `Tab` over the element cycles its faces, axes and corners and highlights the one a click would bind, exactly as when creating a chain;
- a click in the open confirms the edit and leaves the mode; it does not move the dimension line;
- the line itself is moved only by dragging, and only along the normal to its line: the sideways component is ignored;
- `Enter` confirms, `Esc` cancels the edit and the dimension stays as it was; one command — one undo step; if nothing changed, leaving the mode makes no undo step;
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

### 3.11. F11. Temporary (virtual) dimension of the selected element

- A selected wall's temporary dimension (its length between its ends) gets a small dimension-line button under the number, with the tooltip "Make Dimension Permanent".
- A click creates a permanent dimension at the temporary line. Its references are chosen as follows:
  - if the wall's end is **joined by its own end** to a perpendicular wall (`walls::joined_perpendicular`), the reference at that end is that perpendicular **wall's centre line** (`{id}`), not the selected wall's handle; a nearby wall that isn't joined is ignored;
  - if there is no joined perpendicular wall at the end, the reference is the wall's own end (`{id, point: "start"|"end"}`);
  - if both ends give different perpendicular walls, the dimension measures between their centre lines; if the same one or none, between the selected wall's ends.
- One `dimension {refs, at}` command — one undo step; afterwards the dimension behaves like any other.
- Because the reference is the perpendicular wall, changing this dimension moves **that whole wall**, not just its joined end, so it stays perpendicular and the walls at its sides stretch. If the dimension stays between one wall's ends, F5 applies.
- The button is shown only for a straight wall: a curved wall's length along its arc is not an aligned dimension (out of scope).
- The temporary dimension's number is edited by the same empty field as any dimension: click — empty field, Enter commits, Esc cancels, the selection stays on the wall (F2.4).
- The value goes through `wall_length {id, value}`:
  1. both ends free — both move by `delta / 2` in opposite directions;
  2. one end attached (`walls::joined_at`) — only the free end moves;
  3. both ends attached — both move by `delta / 2` in the direction of the chain.
- For a curved wall the previous behaviour stays: the far end moves along the chord, keeping the arc's turn.
- After the moves the command checks the length (within `SLACK`), calls `constraints::keep`, and rolls back fully when it can't (F4).

### 3.12. F12. Selecting and deleting a chain segment

- With the pointer over an aligned or linear dimension, `Tab` cycles the dimension's **segments**, not the whole element: the first `Tab` highlights the segment under the pointer, the next ones step on.
- The segment is highlighted over the dimension line (stronger when selected than when only stepped by Tab).
- A left click on the highlighted segment selects it: `selection` names the dimension, the segment is remembered separately.
- `Delete` calls `dimension_segment {id, segment}` and removes only that segment; the neighbours **do not stretch** into its place and stay bound to their own references:
  - the chain splits: the segments before the removed one stay the original dimension, the segments after it become a new dimension at the same line; at an end segment only one of the parts remains;
  - every remaining segment keeps its reference pair, its lock and its text; the removed segment's references do not move anywhere;
  - a part left with one segment loses `equal`/`equality`;
  - if the segment was the only one, the whole dimension is removed.
- `Esc` and a click aside clear the segment selection; the element selection and the model do not change.
- `Tab` doesn't cycle segments when the pointer is not over a dimension: as before, it cycles elements.

#### Reference editing mode

- A selected aligned or linear dimension offers the `Edit Dimension References` button on the Modify panel; it enters the same mode as the context menu item. Pressing it again leaves the mode.
- In edit mode:
  - hovering and clicking a bound reference (a concrete face, axis or corner) **removes just it**; clicking another face or corner of the same element **adds** a reference — faces of one element are independent and do not replace each other;
  - clicking a new element, its face, axis or corner **binds** it (adds a reference);
  - `Tab` over the element under the cursor cycles its faces, axes and corners, highlighting the reference a click would bind; the same works while creating a chain;
  - a click in the open moves nothing but **confirms the edit and leaves the mode** (if the reference set didn't change — no undo step); `Enter` also confirms with one `dimension {id, refs, at}` command, `Esc` cancels.
- Fewer than two references cannot be left.
- The dimension line's position changes **only by dragging**, and only along the normal to its line: the sideways component is ignored. This holds both in normal mode and while editing references.

### 3.13. Change map

Not a cause analysis, just a short map: what to change and where.

| What | File | Reference |
|---|---|---|
| `dimension_value`: `hold`, station choice, verification, rollback | `crates/core/src/commands.rs` | ~12382 |
| one-wall length: 3 cases, `move` with `end` | `crates/core/src/commands.rs` | ~12382, ~11767 (`relocate`) |
| one-wall length through faces/end faces: iterative nudge | `crates/core/src/commands.rs` | `dimension_value`, `resize_wall` |
| `wall_length`: wall length in three cases | `crates/core/src/commands.rs` | near `resize_wall` |
| `dimension_segment`: removing a segment splits the chain | `crates/core/src/commands.rs` | near `dimension_constraints` |
| `EQ`: press and value input, locks | `crates/core/src/constraints.rs` | ~28 (`targets`), ~112 (`keep`) |
| `stations`, `dimension_values`, `dimension_labels` (numbers, no EQ substitution) | `crates/core/src/model.rs` | ~5631, ~5717, ~5656 |
| `dimension {id, refs}`: editing a dimension | `crates/core/src/commands.rs` | ~8176 |
| attached-end predicate | `crates/core/src/walls.rs`, `walls/index.rs` | add `joined_at` |
| perpendicular joined wall | `crates/core/src/walls.rs` | add `joined_perpendicular` |
| highlight, empty field, text button, reference toggle | `crates/ui-egui/src/canvas.rs` | ~1676, ~2213, ~3257, ~3290 |
| EQ over the chain: indicator and toggle | `crates/ui-egui/src/canvas.rs` | `dimension_eq_hint` |
| open/shut padlock | `crates/ui-egui/src/canvas.rs` | `padlock` |
| temporary dimension: permanent button, references | `crates/ui-egui/src/canvas.rs` | `temporary_dimension` |
| Tab over segments, click, Delete | `crates/ui-egui/src/canvas.rs`, `crates/ui-egui/src/lib.rs` | `canvas`, `delete` |
| Tab over faces, axes and corners | `crates/ui-egui/src/canvas.rs` | `dimension_references`, `dimension_reference_highlight` |
| drag square to the line only | `crates/ui-egui/src/canvas.rs` | `dragged` |
| Edit Dimension References button on Modify | `crates/ui-egui/src/lib.rs` | `modify_tab` |
| `Dimension Text` window | `crates/ui-egui/src/dimtypes.rs` | ~242 |
| keys and hints | `crates/ui-egui/src/lib.rs` | ~2008, ~4946 |
| translations and README | `translations/ui/*`, `README.md` | rows 247, 383, 517–519 |

---

## 4. Pitfalls

1. **Station ordering.** `refs` in the file may be in any order; segments are counted by the sorted stations. Never use an index in `refs` as a segment index.
2. **One element — two references.** A wall can supply both the left and the right face, and both ends. The "one element" check must compare station `id`s, not the whole `Reference`. Decide length versus thickness by the dimension's direction (`u × n ≈ 0` means length), not by which faces were picked; for thickness — refuse.
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
16. **A click in the open while editing references.** It confirms and leaves the mode but must not move the line. Moving the line only by dragging; keep the position from the current model, not from a stale edit draft.
17. **Dragging a dimension.** Project the drag onto the line's **normal**, not onto the line's direction: the dimension stays square to what it measures and never slides along itself.
18. **Faces of one element.** Do not remove a reference by element id when another face of the same element is clicked: every face, axis and corner is a separate reference. Only an exact bound reference is taken out.
19. **Near versus joined.** A perpendicular wall is a reference for a permanent dimension only when the wall's end is joined to it; `neighbours_at` alone finds nearby walls, so the join must be confirmed by the model's own arm/join geometry.
20. **Face corners at joined ends.** Moving a wall by the location line's own points is exact; mitred face corners and end faces may lag, so `dimension_value` repeats the nudge until the measured distance is the value.

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
21. `a_segment_text_override_replaces_the_number` — `dimension_text {replace}` wins over the number; an empty `replace` restores the true number. An equal chain shows its lengths, not `EQ`.
22. `wall_length_resizes_a_free_wall_evenly`, `wall_length_moves_only_the_free_end_when_one_is_joined`, `wall_length_moves_both_joined_ends_evenly` — the three F11 cases.
23. `deleting_a_segment_leaves_the_chains_either_side_measuring_their_own_references` — an inner segment of a four-reference chain: the left part stays the original dimension, the right becomes a new one, each on its own references; measures are not stretched.
24. `deleting_the_last_segment_of_a_chain_keeps_the_other_references`.
25. `deleting_the_first_segment_of_a_chain_keeps_the_other_references`.
26. `deleting_a_single_segment_dimension_takes_the_dimension_away`.
27. `deleting_a_segment_of_an_equal_chain_turns_its_equality_off_when_one_is_left`.
28. `resizing_by_a_walls_face_corners_works_where_its_ends_are_joined` — references on a side face (corner handles), both ends joined; the value applies.
29. `a_dimension_between_a_walls_end_faces_resizes_it`.
30. `a_dimension_from_an_end_face_to_the_other_end_resizes_it`.

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
7. `text_icon_under_the_number_opens_the_text_dialog` — the large framed `Aa` button over the number is visible on the selected dimension; clicking opens the window; `Esc`/Cancel change nothing.
8. `clearing_replace_restores_the_true_number` — typed text replaces the number; clearing restores the true one.
9. `edit_dimension_references_from_the_context_menu` — enter the mode, remove a reference, confirm, the new dimension and segment count.
10. `esc_leaves_an_edited_dimension_as_it_was`.
11. `the_temporary_wall_dimension_has_a_button_that_makes_it_permanent` — the button is under the temporary number; clicking creates a normal dimension between the wall's ends.
12. `typing_a_temporary_wall_dimension_moves_both_joined_ends_evenly` — both ends joined; typing a new length moves both evenly.
13. `tab_picks_a_dimension_segment_and_delete_takes_it_out` — Tab highlights the segment under the cursor, a click selects it, Delete removes only it.
14. `an_equal_chain_shows_its_numbers_and_eq_over_it` — an equal chain shows lengths; EQ shows while hovering and on a selected dimension, disappears for an unselected one when the pointer leaves; a click toggles equality and keeps the selection.
15. `a_permanent_dimension_at_a_joined_end_measures_to_the_perpendicular_wall` — the references are the perpendicular walls' centre lines; changing it moves the whole perpendicular wall, which stays square.
16. `deleting_an_inner_segment_leaves_two_chains_on_their_own_references` — two dimensions, each on its previous references; measures are not stretched.
17. `the_modify_tab_edits_a_selected_dimensions_references` — the Modify tab offers Edit Dimension References; it enters the mode, a click on a bound wall removes its reference, Enter confirms.
18. `a_click_in_the_open_leaves_the_edit_and_does_not_move_the_line`.
19. `dragging_a_dimension_moves_it_only_square_to_its_line`.
20. `a_second_face_of_a_bound_wall_is_added_not_taken_out`.
21. `a_free_wall_makes_a_dimension_to_its_own_ends_not_to_a_wall_nearby` — a nearby, unjoined perpendicular wall is not a reference.

### 5.3. Quality control

- `cargo test --workspace`;
- `cargo clippy --workspace --all-targets -- -D warnings`;
- `cargo fmt --check`;
- oracle tests and `Model::check` after every command (already built into the engine).

---

## 6. Documentation and translations

1. `README.md`:
   - row 247 (Annotation) — the number moves the chosen reference; pressing `EQ` equalizes with fixed ends; typing a value re-spaces the chain from an end reference; dimension text opens through the `Aa` button; the temporary dimension, wall length and references;
   - row 383 (Dimension, tag) — clicking a reference again removes it; editing an existing dimension; the text button; segment deletion; dragging square to the line;
   - row 517 (`dimension`) — the `id` parameter and the edit rules; rows for `dimension_segment`, `wall_length`;
   - row 519 (`dimension_value`) — the `hold` parameter.
2. `BIMCraft/docs/areas/desktop.md` — the section about dimensions, highlighting, the Modify button, dragging and the context menu.
3. `crates/core/src/commands.rs` — the registry descriptions of `dimension`, `dimension_value`, `dimension_segment` and `wall_length`.
4. `translations/ui/bimcraft-ui.pot` and every `translations/ui/*.po` — new strings:
   - `"Click a picked reference again to take it out, Backspace takes out the last, Esc starts over."`
   - `"Edit Dimension References"`
   - `"Dimension text"` (button label)
   - `"Make Dimension Permanent"`
   - `"Delete Dimension Segment"`
   - `"Wall Length"`
5. `docs/areas/views-and-graphics.md`, if it has a dimension section — synchronize it.

---

## 7. Acceptance criteria

1. With an element selected, the text of the attached chain is highlighted (smaller and `accent_ink`); with a dimension selected, there is no highlight.
2. Clicking a highlighted number opens an empty field; Enter commits; Esc cancels; the selected element moves and the opposite reference stays.
3. Pressing `EQ` keeps the end elements in place and equalizes the inner distances; when impossible — a clear error, not silence.
4. Typing a value into an `EQ` chain makes all segments equal to `V`; one end reference moves; if both ends must stay, `V` must equal `total / k`, otherwise an error.
5. A one-wall length dimension works in all three cases: both ends free — symmetric; one attached — the free one; both attached — symmetric.
6. A one-wall thickness dimension does not "silently succeed": it returns a clear error and the model does not change.
7. The dimension-text window opens through the `Aa` button over the number; double-click is not the only path; `replace` wins over the number; clearing restores the true number; an equal chain shows lengths, not EQ.
8. A wrong reference can be removed by clicking it again or with `Backspace`; `Esc` clears the chain as before.
9. The reference set of an existing dimension can be changed; `locked` and `texts` stay on their reference pairs, and the orphaned ones are dropped.
10. Any failed edit rolls back fully (model, locks, undo).
11. All existing dimension and constraint tests pass; the new tests from section 5 are added.
12. `cargo test --workspace`, `cargo clippy`, `cargo fmt --check` — no errors.
13. The `.bimcraft` format did not change; old projects open.
14. README, command descriptions and translations are updated.
15. The selected wall's temporary dimension has a "make permanent" button; clicking creates a normal dimension between the wall's ends; the temporary number is edited in the three F5/F11 cases, both joined ends moving evenly.
16. Tab over a dimension chain cycles and highlights segments; a click selects one, `Delete` removes only it: neighbours do not stretch (the chain splits, or one part remains; the only segment removes the dimension); `Esc` and a click aside clear the selection.
17. On an EQ dimension the labels are ordinary numbers; EQ appears over the middle of the chain only on screen, always on a selected dimension, struck through until clicked; a click toggles equality, and there is no button to the right.
18. A free segment wears an open padlock, a locked one a shut padlock.
19. Where a wall end is joined to a perpendicular wall, the permanent dimension takes that wall's centre line, and a change moves the whole wall (it stays perpendicular), not only its joined end.
20. The Edit Dimension References button on the Modify tab enters the reference mode; a click on a bound reference removes it, a click on another binds it, Enter confirms with one command, Esc cancels.
21. A click in the open while editing references does not move the chain but confirms and leaves the mode; the line moves only by dragging, and only along the normal — the sideways component is ignored.
22. A click on another face or corner of an already bound element adds a reference instead of removing the previous one; every face, axis and corner binds independently.
23. Over an element, Tab cycles its faces, axes and corners and highlights the reference a click would bind, both when creating and when editing references.
24. A free wall near a perpendicular wall that isn't joined makes its permanent dimension between its own ends, not to the nearby wall.
25. Typing a value into a dimension whose references are the wall's side faces, corner handles, end faces or a mix changes the wall's length and no longer fails with "joined to something that stays"; the measured distance equals the typed value. Thickness still does not change through a dimension.

---

## 8. Manual verification

1. **Scenario 2.** Draw two parallel walls and place an aligned dimension between them. Select the first wall → the dimension text is highlighted. Click the number → the field is empty; type a new number and press Enter → the first wall moved, the second stayed.
2. **Scenario 2, cancel.** The same, but press Esc after clicking the number → nothing changed, the selection stayed on the wall.
3. **Scenario 3.** Build a three-segment dimension between walls, select the dimension → a struck-through `EQ` is visible over the middle of the chain; click it → it is no longer struck through, the end walls stay and the inner ones are equalized. There is no button to the right of the chain.
4. **Scenario 4.** On the same chain, type a value into the middle segment → all three numbers become the typed value; one end wall moved; pressing `EQ` again changes nothing.
5. **Scenario 1.** Start a dimension, pick two references and a third wrong one; click the third again (it disappears); use `Backspace` to remove the last one; place the dimension.
6. **Scenario 5.** Select an existing dimension → context menu or the Modify tab's Edit Dimension References → remove one reference → confirm; numbers, locks and texts are updated.
7. **Scenario 6.** Select a dimension → the large `Aa` button is visible over the number; clicking opens the text window; enter Replace With Text → the number is replaced; clear it → the true number returns.
8. **Wall length, both ends free.** Dimension from the wall start to its end; enter 6000 → both ends moved by the same amount, the middle stays.
9. **Wall length, one end attached.** Join one end to another wall; enter a new value → only the free end moves.
10. **Wall length, both ends attached.** Close the wall into a corner on both sides; enter a new value → both ends move by the same amount and the joins do not break.
11. **Wall thickness.** Dimension between the left and right faces of one wall; try to enter a number → an error about the wall type, the wall did not change.
12. **Conflict.** Lock two adjacent segments and try to change one → an error, nothing changed; the rollback is automatic.
13. **Undo.** After every successful edit, one `Ctrl+Z` restores both the number and the element positions.
14. **Virtual dimension.** Select a wall → under the temporary dimension's number is the dimension-line button; click it → a normal dimension between the wall's ends is made. Click its number → empty field; type a new length → the wall changes in the three cases.
15. **Chain segment.** Point at a dimension, press Tab → the segment is highlighted; click → selected; `Delete` → the segment is gone and the neighbours stay on their own references (an inner segment splits the chain). `Esc` before deleting leaves the dimension as it was.
16. **Section mark.** If a section mark runs near a temporary dimension's number, a double click on the mark still opens the section.
17. **Padlock.** A free segment wears an open padlock, a locked one a shut padlock; the state reads without hovering.
18. **Perpendicular join.** Close a rectangle; select the bottom wall, press the permanent-dimension button → the references stand on the perpendicular side walls. Type a new length: the side wall moves as a whole and stays perpendicular, the opposite wall stays.
19. **Edit button.** Select a dimension → the Modify tab has Edit Dimension References. Press it → edit mode; a click on a bound face removes it; a click on a new face/corner binds it; a click in the open confirms and leaves, without moving the line; Enter confirms.
20. **Dragging.** Select a dimension and drag diagonally: it moves only along the normal to its line, never sideways; the same while editing references.
21. **Faces and Tab.** Over an element, press Tab: its faces, axes and corners are highlighted in turn; a click binds the highlighted one. Create a chain this way and add a reference the same way while editing.
22. **Free wall.** Put a perpendicular wall near a free end without joining it; make the temporary dimension permanent → it measures only the selected wall's ends.
23. **Several faces.** While editing references, click the left face of an already bound wall: it is added, the previous reference is not removed.
24. **Dimension on the wall's faces.** Place a dimension between side-face handles or between end faces of one wall (with joined ends); type a new length → the wall changes length, no "joined to something that stays" error.

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
3. **Pressing `EQ` and typing a value are different actions.** Pressing keeps both end references and equalizes at the current total length. Typing a value makes all segments equal to `V` and moves one end reference by default; the `hold: "both"` mode keeps both but requires `V = total / k`, otherwise it returns an error. The control is the label over the middle of the chain (struck through when off), not a button to the right.
4. **For one wall the length changes in three cases (free / one attached / both attached), and thickness is refused.** Changing the wall type from a dimension is out of scope for this specification. References may be the faces, corner handles, end faces or a mix; the command nudges until the measured value is reached.
5. **Dimension text opens through the `Aa` button over the number.** The existing `DimensionTextDialog` is reused; `replace` wins over the number, and EQ is a hover/selection label, not a replacement for numbers.
6. **Result verification happens inside `dimension_value` after `keep`.** The model is in an `Arc` and the snapshot is cheap; the engine performs the rollback.
7. **The file format does not change.**
8. **Deleting a segment splits the chain.** The parts either side keep their references, locks and texts; they are not merged across the deleted segment.
9. **A permanent dimension from a temporary one takes a perpendicular wall only when the end is joined to it.** Then it moves that whole wall and keeps it square; a nearby unjoined wall is ignored.
10. **Dragging is square to the line, and a click in the open leaves reference editing without moving the line.** Every face, axis and corner of an element is a separate reference; Tab steps through them.

Open questions (resolved during implementation):

1. Reference handles: the selected dimension still shows them; numbers are not highlighted when a dimension is selected. Kept as a secondary way.
2. All chains of the selected element are highlighted; the nearest is not emphasized separately.
3. Double-click stays as a shortcut for the text window; the `Aa` button is the documented path.
4. Only `dimension_value` returns the explicit `values`.

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
9. **ui: text button and reference editing.**
   - the large `Aa` button over the number, opening `DimensionTextDialog`;
   - Edit Dimension References in the context menu and on the Modify tab;
   - tests 7–10 from 5.2.
10. **core+ui: virtual dimension.**
    - `wall_length`: the three F5/F11 cases, verification and rollback;
    - `joined_perpendicular`: the permanent dimension binds to a joined perpendicular wall so that the whole wall moves;
    - the permanent button under the temporary number;
    - core test 22, UI tests 11–12 and 15.
11. **core+ui: dimension chain segments.**
    - `dimension_segment`: deleting a segment splits the chain, neighbours stay on their references; locks/texts follow; a lone part loses equality;
    - Tab over segments, highlight, click, `Delete`;
    - the Edit Dimension References button on the Modify tab; faces, axes and corners bind independently; Tab cycles them with a highlight;
    - a click in the open confirms without moving; dragging moves only along the normal;
    - a free wall does not bind to a nearby unjoined wall;
    - core tests 23–30, UI tests 13, 16–21.
12. **docs: README, command descriptions, `docs/areas`, translations.**
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
| 9. Virtual dimension | F11 | core 22, UI 11–12, 15, 21, manual 14, 18, 22 |
| 10. Dimension chain segment | F12 | core 23–27, UI 13, 16, manual 15 |
| 11. Text of an EQ dimension | F3, F7 | UI 14 |
| 12. Padlock of a free segment | F2.2 | — (visual) |
| 13. Permanent dimension at a perpendicular wall | F11 | UI 15 |
| 14. Free wall near a perpendicular wall | F11 | UI 21, manual 22 |
| 15. Click in the open while editing references | F12 | UI 18, manual 19 |
| 16. Dragging square to the line | F12 | UI 19, manual 20 |
| 17. Independent faces and corners | F8, F12 | UI 20, manual 23 |
| 18. Tab through faces and axes | F12 | manual 21 |
| 19. Edit References button on Modify | F12 | UI 17, manual 19 |
| 20. Dimension on faces, handles and end faces of a wall | F5 | core 28–30, manual 24 |
