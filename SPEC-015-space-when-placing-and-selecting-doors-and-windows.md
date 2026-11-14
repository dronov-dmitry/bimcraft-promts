# SPEC-015. Space while placing and selecting a door/window: Revit parity

Status: **partially implemented** (the foundation is in `main`), must be finished to parity
Date: 2026-11-13
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: [IPedrax/BIMCraft#434 "Nice to have"](https://github.com/IPedrax/BIMCraft/discussions/434), moved from issue [#301](https://github.com/IPedrax/BIMCraft/issues/301) (AwayZz20, 2026-11-07): "other thing will be good to have if I put a door the space bar would be change the position of the door".
Revit documentation the behaviour was checked against: Autodesk "Place a Door" (2027), "Video: Place Doors" (2024), "About Flip Controls" (2025), "Change the Orientation of a Door" (2026); checked 2026-11-13.
Related documents: `ТЗ-016-стрелки-флипа-двери-окна.md` (the same capabilities, but on the canvas), `ТЗ-014-пол-по-грани-ядра-стены.md` (does not overlap), `ТЗ-008-горячие-клавиши.md` (key rules), `ТЗ-012-импорт-клавиш-revit.md`.
Task type: bring the UX to parity (keys, tool state, windows, multi-selection), a small extension of the `flip` command, tests; the file format does not change.

> **In short.** The user asks for Revit behaviour: Space during placement changes the orientation of the door, and after placement Space cycles through the positions; in Revit this works for doors and windows, for one and for several selected elements. In BIMCraft **the foundation already exists** (commit `f54b5c1` in `main`): `door_space` (`lib.rs:3096–3120`) toggles the pending hinge side of a single swing door during placement and cycles the four "Opens" states of the selected one, and there are tests (`tests/inspector.rs:184–299`). What is missing: windows (both while placing and when selected), multi-selection, hints in the status bar, a single rule "what can be flipped", and one command per press for a mixed selection. This specification brings the behaviour up to Revit and does not duplicate ТЗ-016: here it is the keyboard and the command, there it is the arrows on the canvas.

## 0. General working rules

1. Communication language — Russian; code, comments, strings in code, console messages — English. Documentation — English, this file — Russian.
2. Tests (`cargo test` and any execution of test code) are **not run** in this task (rule `~/.config/opencode/rust-rules.md`): verification is `cargo check`, `cargo build`, `cargo clippy` (without `--all-targets`) and CI.
3. Commits are atomic: one commit — one logically complete change; each one builds and does not break the existing behaviour.
4. Identifiers are not translated; user-facing strings go through `msg!`/`ui_msg!`/`t()`, the `translations/` catalogs are updated (`cargo i18n update`).
5. `Model::check` is the only place for validation; the rule "what can be flipped" is not a model check but a capability computed from the type/family; it adds no model errors.
6. Revit behaviour is reproduced from public documentation, without copying Autodesk data.

## 1. The essence of the problem

### 1.1. What exists now (by code)

**Placement**

- Space is handled in `lib.rs:2353–2362`: only a "clean" press (`Key::Space`, `pressed: true`, `repeat: false`, `Modifiers::NONE`), then `door_space()`.
- `door_space` (`lib.rs:3096–3120`):
  - `Tool::Door`, if a plan exists: checks `m.single_swing_door(family, type)` and toggles `self.door_flip_hand` — **the hinge side only**;
  - `Tool::Modify` with exactly one selected door: `modify {id, flip_hand: !o.flip_hand, flip_facing: o.flip_facing ^ o.flip_hand}` — cycles the four states: `(f,f) → (t,f) → (f,t) → (t,t) → (f,f)`;
  - otherwise `false`.
- The opening side during placement (facing) is not set by Space: it is computed from the cursor in `opening_params` (`canvas.rs:2149–2164`): `flip_facing = Axis::of(w).project(cursor).1 < walls::centre(m, w)`, and only for a door with `single_swing_door`; for a window the block does not run, `flip_facing` stays `false` by default.
- `door_flip_hand` is reset when the tool changes (`lib.rs:2953`); Escape/tool change discards the pending state; the model is not changed on hover and on Space before the click (test `door_space_placement_uses_the_physical_wall_centre_and_escape_discards_the_pending_hand`, `tests/inspector.rs:279–300`).
- Hints: Door and Window show "Click a wall to place the door/window" (`lib.rs:5753–5754`); nothing is said about Space and the cursor side. Component/Fixture do mention Space (`lib.rs:5839`), the door does not.

**Selection**

- With a single swing door selected, Space cycles the four "Opens" states; the Properties palette shows the same four tiles (`properties.rs:1588–1638`) and sends `modify` with both flags.
- When **several** doors are selected, Space does nothing: `let &[id] = self.selection.as_slice() else { return false }` (`lib.rs:3109`).
- Windows are not handled at all: `door_space` only looks at `Element::Door`.
- Walls and the rest: after `door_space`, `flip()` is called (`lib.rs:3122–3128`) for `Tool::Modify`, which sends `flip {ids}` for the selected walls. If the selection contains both a door and a wall, `door_space` returns `false` (more than one door / a wall is present), `flip()` flips only the walls, the door stays as it was — a mixed selection behaves unpredictably.
- `flip` in the core (`commands.rs:9639–9699`) accepts **walls only** and refuses on the first non-wall: `"{category} {id} isn't a wall"`. It moves doors/windows only as "passengers" of a flipped wall. There is no separate command to "flip a door/window"; the UI assembles it from `modify`.

**What is already covered by tests** (keep them green)

- `door_space_and_pointer_choose_the_leaf_shown_before_and_after_a_click` (`tests/inspector.rs:184–213`): Space does not place the door, it only affects the preview; facing follows the cursor side for walls drawn in different directions.
- `door_space_cycles_the_opens_choices_with_one_undo_each_and_reopens` (`216–244`): four states in a loop, each change is one undo, the state is saved in the file.
- `door_space_leaves_typing_other_operations_and_multiple_doors_alone` (`247–276`): Shift+Space and Space in a text field do not touch the model; a sliding door is not flipped; with two doors selected Space does nothing; on a wall Space still flips.
- `door_space_placement_uses_the_physical_wall_centre_and_escape_discards_the_pending_hand` (`279–300`): with `location: exterior-finish`, facing is computed from the physical centre of the wall rather than from the location line.

### 1.2. What Revit does (acceptance scenarios)

1. **Placing a door.** The cursor over a wall shows a preview. **Space toggles the hinge side (hand) left/right.** The opening side (facing, inward/outward) is chosen by the **cursor position**: closer to the inner face — opens inward, closer to the outer face — outward. Space does **not** toggle facing (Autodesk "Place a Door").
2. **Placing a window.** Space toggles the orientation of the window (in Revit terms — sill inside/outside, i.e. facing); the side also follows the cursor.
3. **Selected door.** Space **cycles through the opening variants** ("cycle through the various swing orientations", Autodesk tutorial), i.e. the four hand/facing states; the two flip arrows on the canvas do the same.
4. **Selected window.** Space toggles hand/facing depending on the family's control arrows; for a window without them — nothing. (Our families do not have control arrows yet, see §9.)
5. **Several elements.** Space applies to the selection: every door/window changes orientation, walls are flipped; one press — one action (ideally one Undo step).
6. **Text fields and modifiers.** Space in an input field is a space, not a flip; Shift+Space/Ctrl+Space are not intercepted; auto-repeat while holding does not flood the history (already done).
7. **Hint.** Revit writes the hint into the status bar; a user who does not know about Space learns about it from the hint.

### 1.3. Why it matters

- For a Revit user, Space is muscle memory: without it, placing a door needs two extra mouse movements to the palette/arrows. This is not a "feature" but a basic working pace.
- Right now half of the behaviour exists, but **inconsistently**: a door — yes, a window — no; one — yes, several — no; with a wall in the selection — "sometimes yes, sometimes no". The user cannot learn the rule, so they do not trust the key.
- One command per one press is also needed for MCP/an agent: today flipping a door is `modify` with absolute flags, and the agent must know the state itself; a `flip` command with relative toggling makes the intent explicit.
- The foundation is already in `main`, so the work is cheap: not to rewrite, but to generalize the existing `door_space` and give it a single capability model.

## 2. Terms

| Term | Meaning |
|---|---|
| Hand | Hinge side (`Opening.flip_hand`), "left/right" |
| Facing | Opening side/face (`Opening.flip_facing`), "inward/outward" |
| Pending state | Flags that are applied to the next door/window being placed; they do not change the model |
| Cycle | The sequence of four states `(f,f) → (t,f) → (f,t) → (t,t)` |
| Capability | Which flips an element supports: hand/facing/none |
| `single_swing_door` | The "swing door" heuristic based on `Operation` (`model.rs:5028–5034`) |
| `Operation` | Door operation type: Swing, Sliding, Folding, Industrial, etc. (`model.rs`, IFC mapping `ifc.rs:1997`) |
| Mixed selection | The selection contains both openings and walls/other elements |
| `flip` | Core command; today walls only (`commands.rs:9639`) |

## 3. Functional requirements

### 3.1. F1. A single capability model: `opening_flips`

- Add to `Model`:

  ```rust
  #[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
  pub struct Flips { pub hand: bool, pub facing: bool }

  pub fn opening_flips(&self, e: &Element) -> Option<Flips>;
  ```

  - `Element::Door(o)`: find the operation (`family` → `families`, otherwise `types.door`) and return:
    - `Operation::Swing` or `None` (a door without a declared operation is a swing door by default, as in IFC) → `Flips { hand: true, facing: true }`;
    - other operations → `Flips::default()` (no flips);
  - `Element::Window(_)`: `Flips { hand: false, facing: true }` **for now**; when families get control arrows — derive from them (see §9);
  - everything else → `None`.
- Do not remove `single_swing_door`: it is used in rendering (`model.rs:5074`, `6140`), IFC and tests. The new `opening_flips` is a layer for the UI/commands so that there is no second list of operations.
- Replace the checks in the UI:
  - `door_space` (`lib.rs:3103`, `3111`) — with `opening_flips(...).is_some_and(|f| f.hand || f.facing)`;
  - the "Opens" tiles (`properties.rs:1591`) — do not show them for doors without flips (currently `type_subject() == "door"` shows the tiles always);
  - ТЗ-016 uses the same function for the arrows.
- Tests: `opening_flips` for Swing, `None`, Sliding, Industrial, Window, Wall.

### 3.2. F2. Space during placement

**Door (already exists, extend)**

- Keep the behaviour: Space toggles only `door_flip_hand`; facing — by the cursor side; pending is reset on tool change and Escape.
- Compute the cursor side from the **physical centre** of the wall (`walls::centre`), as now; this is correct also for walls with `location: exterior-finish` (the test already exists).
- If the cursor is exactly at the centre (`|d - centre| <= TOL`), facing is taken from the last preview/previous click; on the first one — `false` (inward, as now).
- If the door has no hand flip (sliding, industrial) — Space does nothing and does **not** toggle facing: Revit does not change the orientation in this case either. The status hint: "This door has no swing to flip".

**Window (new)**

- Add a pending state `window_flip_facing: bool` (following the example of `door_flip_hand`, `lib.rs:909`, reset in `set_tool`).
- `opening_params` (`canvas.rs:2156`) for a window:
  - `flip_facing = Axis::of(w).project(cursor).1 < walls::centre(m, w) ^ self.window_flip_facing` (the cursor side, inverted by Space);
  - do not set `flip_hand` (a window does not have it).
- `opening_space` (the renamed `door_space`): `Tool::Window`, if a plan exists and `opening_flips(...).facing` → `window_flip_facing = !window_flip_facing`.
- The window preview is already built from `opening_params`, so the flipped side is visible immediately; the model does not change before the click.

**General rules**

- Pending states do not survive a tool change, Escape or a successful placement (reset after the click, as `door_flip_hand` does now).
- Space must not place the element: keep the `pressed` consumption (`lib.rs:2357–2359`), the test "Space does not place a door" remains.
- Auto-repeat (`repeat: false`) and modifiers — unchanged.

### 3.3. F3. Space on selection

- Rename `door_space` → `opening_space`; branches:
  1. `Tool::Modify` + a selection of **only openings** (one or several):
     - for every door with `flips.hand`: the cycle as now — `hand = !hand`, `facing ^= old_hand` (this yields 4 states);
     - for every window/element with facing only: `facing = !facing`;
     - for a door with facing only (if it appears): `facing = !facing`;
     - apply to **every** selected opening independently; if there are no flips — skip it;
  2. `Tool::Modify` + a selection of openings **and** walls: flip both in one press; preferably with one command (F4), otherwise — one per element and fix the number of Undo steps in a test;
  3. `Tool::Modify` + walls only: the existing `flip()` (`lib.rs:3122–3128`) unchanged;
  4. `Tool::Modify` + mixed with non-flippable elements (column, furniture): the flippable elements are flipped, the rest are left alone; if there are no flippable ones — nothing.
- A mixed selection must not "eat" Space for the walls: today, with two doors, Space does nothing, and `flip()` then flips only the walls; the new logic must be predictable and covered by a test.
- The "Opens" tiles (`properties.rs:1588–1638`) remain the second way to do the same thing; with multi-selection they already work (a common `modify` with all ids) — Space must give the same result.
- One undo per one press — a requirement; see F4.

### 3.4. F4. The `flip` command for doors and windows

- Extend `flip` (`commands.rs:9639`) without breaking the current wall behaviour:

  | Parameter | Type | Default | Meaning |
  |---|---|---|---|
  | `ids` | `[id, …]` | — | Walls and/or openings |
  | `hand` | bool | none | Absolute `flip_hand` value for openings that have a hand |
  | `facing` | bool | none | Absolute `flip_facing` value |
  | `cycle` | `"hand"` \| `"facing"` \| `null` | none | Relative toggle per the Space rule |

- Rules:
  - walls — as today (`commands.rs:9653–9658`), `hand`/`facing`/`cycle` do not apply to them (or an error if given together with walls — decide in code and in a test);
  - openings — only the supported flags are changed (`opening_flips`); an unsupported flag in the parameters → an error naming the element and the available flips;
  - `cycle: "hand"` (doors) — the same formula as in the UI: `hand = !hand; facing ^= old_hand`; `cycle: "facing"` — `facing = !facing`;
  - one execution — **one** Undo step for the whole selection;
  - pinned (`refuse_pinned`) elements — as with walls: refuse, nothing changes;
  - response: `{"flipped": [...], "states": {id: {hand, facing}}}` (states — for tests and the agent).
- The UI uses `flip` instead of `modify` in `opening_space` and in the "Opens" tiles (the tiles stay absolute — `hand`/`facing`), so that Undo is predictable.
- MCP: document `flip` in the command description; examples:
  - `flip {"ids":[12], "cycle":"hand"}` — cycle the positions of the door;
  - `flip {"ids":[13], "facing":true}` — turn the window face to the other side;
  - `flip {"ids":[1,2,12]}` — walls and a door in one action (walls are flipped, the door — by the cycle? no: without `cycle`/flags the door does not change; it is better to require an explicit parameter for openings — decide and record).
- `modify {ids, flip_hand, flip_facing}` stays as is for point edits; do not touch its tests.

### 3.5. F5. Hints and documentation

- Status bar (`lib.rs:5753–5754`):
  - `Tool::Door` → "Click a wall to place the door. Space flips the hinge; the pointer's side of the wall sets the swing." (text through `msg!`);
  - `Tool::Window` → "Click a wall to place the window. Space flips its face; the pointer's side sets which face is out.";
  - `Tool::Modify` with a door/window selected → "Space flips the hand/facing; the flip controls on the plan do the same." (only if `opening_flips` is not empty);
  - `Tool::Modify` with a mixed selection → do not overload: keep the general Modify hint.
- Dialog/help: in `README`/`docs` (English) describe Space for a door/window and the difference from the flip arrows; link to ТЗ-016.
- i18n: new strings in `translations/ui`, `cargo i18n update`; check that the language switcher does not break formatting (there are no placeholders).

### 3.6. F6. Compatibility with IFC, import and `Insert from File`

- `Opening.flip_hand`/`flip_facing` are already read/written by IFC (`ifc.rs:1997`, `import.rs:3700–3727`) and by RVT import (`rvt/src/openings.rs:257`); the format does not need to change.
- `opening_flips` does not affect import: a door with an `Operation` that BIMCraft does not know gets `Flips::default()` in the UI only; the geometry and flags stay.
- `Insert from File` moves the flags as is.
- If in the future a family's control arrows appear (ТЗ-016, §9), `opening_flips` reads them; the command and Space do not change.

## 4. Model and format

- The `.bimcraft` format does not change: `Opening` already stores both flags (`model.rs:917–938`).
- `Model::check` does not change.
- There are no new fields in the file; `Flips` is a computed value.
- There is no migration.

## 5. Tests (describe; do not run)

Unit `core`:

1. `opening_flips`: Swing → (hand+facing); `None` → (hand+facing); Sliding/Folding/Industrial → (false,false); Window → (false,true); Wall/Column → `None`.
2. `flip {cycle:"hand"}` on a door: four calls give `(t,f)`, `(f,t)`, `(t,t)`, `(f,f)`; one Undo per call.
3. `flip {facing:true}` on a window; `flip` on a door without flips — an error with the available flips.
4. `flip` on a mixed list of wall+door: the wall is flipped, the door changes per the requested rule; one Undo; a pinned element causes the whole action to be refused.
5. `flip` with walls and `hand`/`facing` — record the chosen behaviour (error or ignore) and cover it with a test.

UI `ui-egui`:

6. Placing a window: Space toggles the pending facing; the preview shows the other side; the click places the window with this `flip_facing`; the model does not change before the click.
7. Placing a door: the old tests `184–213`, `279–300` stay green; Space still does not place the door.
8. Selected door: the old cycle test `216–244` remains; Space uses `flip` (check the number of Undo steps).
9. Selected window: Space toggles facing; one Undo; a repeated Space restores it.
10. Two selected doors: Space flips both; one Undo; the states are different — each follows its own cycle.
11. Door + wall: Space flips both the wall and the door; one Undo (if F4 provides it); the selection is preserved.
12. Mixed with a column: the column does not change, the door changes.
13. Sliding/industrial door: Space does nothing, the model is the same, the hint says "no swing to flip".
14. Space in the "Mark" field remains a space; Shift+Space does not flip (the old test `247–276`).
15. Hints: Door/Window and Modify with an opening contain the word "Space" in the text (a test on the label/status).
16. The "Opens" tiles are not shown for a door without flips; for a window — shown (if we decide to show them) or hidden (record the decision).

Visual check: the plan `render` — the window and door previews change side on Space, after the click the orientation is the same.

## 6. Scope boundaries

- We do **not** add door/window placement in 3D, sections and elevations: the tool currently works in plan (`plan_level`), this is a separate task; Space is not needed there.
- We do **not** make family control arrows (family editor) — that is ТЗ-016 and the future family editor; here only instance flags.
- We do **not** change the geometry of doors and operations: `Operation` and `single_swing_door` remain the source of truth for rendering.
- We do **not** introduce rotation of free-standing elements via Space (Component/Fixture already does this, `lib.rs:2376–2378`); do not mix it with doors.
- We do **not** touch the IFC export of the flags: it is already correct.
- We do **not** make configurable hotkeys for Space: that is `ТЗ-008`/`ТЗ-012`, here the key is fixed, as in Revit.

## 7. Acceptance criteria

1. While placing a door, Space toggles the hinge side, the opening side follows the cursor; the model does not change before the click; Escape cancels the pending state.
2. While placing a window, Space toggles the side (facing), the cursor sets the base side; the click places the window with the shown position.
3. With a door selected, Space cycles the four positions; with a window selected — toggles facing; with a non-swing door — nothing and a clear hint.
4. With several openings selected, Space applies to all of them; with a mixed selection including walls — to the walls as well; one press = one Undo step.
5. `flip` supports `hand`, `facing`, `cycle`, openings and walls; errors name the element and the available flips; the MCP description is updated.
6. The Door/Window/Modify hints mention Space; the strings are translated.
7. The old Space/flip tests are green; new tests are added; `cargo check`, `cargo build`, `cargo clippy` (without `--all-targets`) pass; CI is green.

## 8. Open questions and decisions made

1. **Space on a selected door: cycle or hand only?** Decided: a cycle of four states, as already in `main` and as in the Revit tutorial ("cycle through the various swing orientations"). This differs from placement, where Space is hand only — that is also how Revit does it.
2. **Window: hand or facing?** Decided: facing (sill inside/outside). The hand of a window will appear together with the family's control arrows; for now `opening_flips` returns `hand: false`.
3. **Multi-selection and Undo.** Decided: one press — one Undo step; for this we extend `flip`. If an implementation via `modify` turns out simpler, it is allowed, but the number of steps is fixed by a test and must not depend on the number of elements.
4. **Mixed selection.** Decided: we flip everything that supports flipping; we leave the non-flippable alone. We do not show an error — this is not a user error.
5. **Walls and `hand`/`facing` in one command.** Decided: walls ignore the opening parameters, and openings ignore the wall geometry; one call does both. If this complicates `flip`, two calls from the UI are allowed, but with a guarantee of one Undo via a future batch — record this in the PR.
6. **Family control arrows.** Decided: out of scope; `opening_flips` is the extension point. dansity's reply in #434 ("control arrows in family editor") does not contradict this: we do instance flips, and we will connect family ones later.

## 9. Work plan by commits

1. `core: opening flip capabilities` — `Flips`, `opening_flips`, unit tests; the UI checks are replaced with it (without changing behaviour).
2. `ui: space flips a window while placing it` — `window_flip_facing`, `opening_params`, reset, preview, tests.
3. `core: flip doors and windows` — extend `flip` (`hand`/`facing`/`cycle`), one Undo, tests; the UI moves Space and "Opens" onto it.
4. `ui: space flips every selected opening` — multi-selection, mixed with walls, tests.
5. `ui: tell the user about Space` — Door/Window/Modify hints, i18n, documentation.
6. `docs: describe flip controls and Space` — `docs/areas`, `README`; link to ТЗ-016.

Each commit builds and does not break the previous one; tests are written in the same commits but are not run locally (CI).
