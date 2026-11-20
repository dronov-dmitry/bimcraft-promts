# SPEC-027. Snapping view windows to the edges, and tiling

Status: **task**, implementation and a CI test run are required
Date: 2026-11-20
Project: BIMCraft (Rust, `BIMCraft-026`, `eframe`/`egui` 0.36.2)
Source: user request of 2026-11-20: "write a file ТЗ-027-***.md in which, by analogy
with the other specs, you write the task for implementing the functionality so that open
windows can be snapped not only to the top as now but also to the right and left sides,
and then the screen is split into an equal number of parts. that is, if there was one
window then the split happens into two parts. and if there were two windows then the split
should happen into three equal parts. and so on. also in the views tab there should be the
ability to lay windows out in tiles into an equal number across the screen (if 4 then in
the four corners of the program's work area and so on, evenly) and also there should be
the ability to press a button and fold the windows back into the standard view when the
view names are visible at the top of the work area."
Clarification of 2026-11-20: "also add the bottom magnet to the spec, then the windows are
distributed evenly from top to bottom."
Related PRs: none, the task is new; view window behaviour in the current checkout has not
been changed.
Task type: UI change. The model, the `.bimcraft` format, commands, IFC and the API do not
change; only the layout of view windows inside the views area changes (plus two new ribbon
buttons).

> **In short.** Today a view window has exactly one "magnet" — the tab bar at the top of
> the views area: a window dragged onto it becomes a tab again
> (`crates/ui-egui/src/dock.rs:838-840`, `:899-906`). Three more zones are needed — at the
> **left**, **right** and **bottom** edge of the views area: a window released at the side
> becomes a column and splits the views area into parts of equal width; a window released
> at the bottom becomes a row and splits it into rows of equal height — the windows are
> distributed from top to bottom. The number of parts along the chosen axis is
> **the windows taken off the bar + 1** (the tab bar part): one undocked window — two equal
> parts, two — three, and so on. Sides and bottom can be mixed: then the width is split into
> columns and the height into rows (F3). In the ribbon's **View** tab, in the **Windows**
> panel, two buttons appear: **Tile Views** — every open view tiled into equal cells (four —
> in the four corners of the views area and so on. Tile should try to distribute the windows
> into squares), and **Dock All Views** — to gather everything back into the ordinary view
> with the view names on the tab bar at the top.

## 0. General working rules

1. The communication language is Russian. Code, comments, UI strings and log messages are
   English. The document (`.md`) is Russian.
2. Tests (`cargo test`, `cargo nextest`, doctest) are **not run** in this task without an
   explicit user request (rule `~/.config/opencode/rust-rules.md`). Allowed are
   `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
   `cargo clippy -p bimcraft-ui-egui` (without `--all-targets`) and `cargo i18n check`.
3. There are no new dependencies. No `unsafe` is added, lints are not weakened.
4. The behaviour of the model, the `.bimcraft` format, commands and the API does not change.
   Only the layout of view windows in the views area and two ribbon buttons change; this is
   window state, not project state, and it does not go into the project file.
5. New UI strings are added through `t`/`msg!` and land in `translations/ui/`
   (`cargo i18n update`). An empty `msgstr` is acceptable: while there is no translation the
   English one is shown; `cargo i18n check` accepts that.
6. Commits are atomic: the layout and the magnets separately, the ribbon buttons separately,
   the tests separately, the translation catalogues separately.

## 1. Current behaviour

Reproduction:

1. Open a project and two views (for example, `Floor Plan: Level 1` and `Level 2`).
2. Drag the `Level 2` tab down into the views area: the view tears off into its own window
   over the views area (`FloatView`, `crates/ui-egui/src/dock.rs:442-448`).
3. Bring the window by its title to the **tab bar at the top** and release: the window
   disappears and `Level 2` becomes a tab again. That is today's "snap to the top".

Expected behaviour after the task:

- the views area gains three more drop zones — at the left, right and bottom edge;
- a window released in the left or right zone becomes a **column** for the height of its part
  (with no bottom rows — the full height of the views area); the views area is split into
  parts of equal width;
- a window released in the bottom zone becomes a **row** across the full width of the views
  area; the views area is split into parts of equal height, and the windows are distributed
  from top to bottom;
- the number of parts along the chosen axis is **the windows taken off the bar + 1** (the tab
  bar part). One undocked window — two parts, two — three, `k` — `k+1`;
- sides and bottom can be mixed: the width is split into columns and the height into rows
  (F3);
- in **View > Windows** there is **Tile Views** — every open view in equal cells (four — in
  the four corners), and **Dock All Views** — a return to the ordinary view, when the view
  names are again on the tab bar at the top;
- the top magnet (the tab bar) stays as it is now.

A clarification on counting the parts (so there is no ambiguity): a "window" here is a view
taken off the tab bar into its own window. The tab bar with all its tabs is always **one**
part, no matter how many tabs it holds. Therefore "there was one window — two parts, there
were two — three" holds literally for the axis along which the window was dragged: `k` windows
undocked to the side give `k + 1` equal columns, `k` windows undocked downwards give
`k + 1` equal rows. If windows are undocked both to the side and downwards, equality is
counted per axis separately (F3).

## 2. How the view windows work now

### 2.1. Data

`crates/ui-egui/src/lib.rs:632-646`:

```rust
/// The active view, which takes the tools: a tab of `views`, or one of `floating`.
pub sheet: Sheet,
/// The views docked as tabs, and the one of them the views' area shows: the active view, unless that floats.
views: Vec<Sheet>,
front: Sheet,
/// Views floating in windows of their own.
floating: Vec<dock::FloatView>,
```

- `views` — the tabs; never empty (`lib.rs:2594-2601`);
- `front` — the view the tab area shows when the active view floats;
- `floating` — the windows over the views area, each with its own `at`/`size`
  (`dock.rs:442-448`).

`Sheet` (`lib.rs:151-172`) is both a project and a family document
(`Sheet::Family(DocId, Option<Which>)`). Family tabs do not float: they are given only
`Sense::click()` (`lib.rs:7218-7224`), and `view_body` draws nothing for them
(`lib.rs:2516-2517`).

### 2.2. Frame layout

`crates/ui-egui/src/lib.rs:2416-2445`:

- in Compact, and for schedules (`Sheet::Schedule`/`Sheet::Table`), the tab bar is
  `Panel::top("view-tabs")` and the view takes the remainder;
- in Floating the tab bar is a card drawn over the view
  (`style::card([true, false, true, false])`), its bottom is remembered in `tabs_foot`;
- `self.work` — the rectangle whose edges the palettes dock to (`lib.rs:2444`) — currently
  equals the active view.

The active view is drawn by `view_body` (`lib.rs:2495-2524`): it remembers
`self.view_rect`/`self.pane` and reserves space for the floating tab cards and the view
control panel (`self.canvas.room`).

### 2.3. View windows

`crates/ui-egui/src/dock.rs:700-757` (`view_windows`): for each `FloatView` an
`egui::Window` is drawn without a system title bar, with a 26 pt strip (the view name, the
`Dock {name}` and `Close {name}` buttons), `resizable(true)`, `min_size(240×160)`, the
position from `f.at`, the size from `f.size`; after showing, the current position and size
are written back into `FloatView` (the window can be dragged and resized).

`view_pane` (`dock.rs:759-785`): the active view gets the tools, the others are drawn
passively, a click on them makes them active.

### 2.4. Dragging and drop zones

- A view tab: `Sense::click_and_drag`; dragging sets `Drag::View(sheet)`
  (`lib.rs:7222-7225`, `:7263-7265`).
- A floating window's title: `Drag::ViewWindow(sheet)`, the window position follows the
  pointer (`dock.rs:724-734`).
- The drop zones for views (`dock.rs:838-840`):

```rust
Drag::View(_) | Drag::ViewWindow(_) => {
    vec![(Land::Tabs(self.tab_index(self.pointer)), self.tab_bar, t("Dock in the Views").to_string())]
}
```

- Release (`dock.rs:899-907`): on the tab bar the tab is reordered or the window is docked
  (`dock_view`); nowhere else is there a zone for a view, so a tab or window simply floats
  where it was released.
- `reach` (`dock.rs:912-921`) extends the palettes' side zones beyond the window edge so
  that a palette dropped at the edge docks. For views this is not used yet.

### 2.5. Invariants

- The tab bar never empties: `float_view` refuses if the tab is the last one
  (`dock.rs:674-685`), and `tidy` returns a floated-away active view to the bar if no tabs
  are left (`lib.rs:2594-2601`).
- `tidy` throws views that no longer exist in the model/documents out of `views` and
  `floating` (`lib.rs:2588-2589`).
- `Close Inactive` leaves only the active view (`lib.rs:5623-5628`).
- The view window layout is **not saved** in `settings.json` (only the palettes' `Docking`
  is there, `settings.rs:112`, `:181`) and does not go into `.bimcraft`.
- A new/opened project replaces `BimApp` entirely (`lib.rs:4536-4585`), so the layout and the
  floating windows start over.

## 3. Goals

1. Add zones for snapping a view window to the left, right and bottom edge of the views
   area.
2. On a drop into a side zone, split the views area into **equal-width** columns: one per
   each window taken off the bar plus one for the tab bar.
3. On a drop into the bottom zone, split the views area into **equal-height** rows: the
   windows are distributed from top to bottom, one per each undocked window plus a top row
   with the tab bar.
4. Keep the top magnet (the tab bar) unchanged.
5. Add a **Tile Views** button to **View > Windows**: every open view in equal grid cells,
   4 — in the four corners, 3 — in three cells of a 2×2 grid, and so on.
6. Add a **Dock All Views** button to **View > Windows**: all windows and tiles become tabs
   again, with the view names on the bar at the top of the views area.
7. Recompute the layout when the application window is resized, when a view is opened or
   closed, on docking and undocking — without a restart and without "remembered" stale
   coordinates.
8. Do not change the model, the project format, commands, IFC, the API, the palettes or the
   behaviour of family documents.

## 4. Non-goals

- Snapping to the **OS window** top edge and the system Snap (Aero Snap, Wayland): we speak
  only about the views area inside BIMCraft.
- Quarter layouts by corners (a window in a corner splitting both width and height): a corner
  point belongs to one zone by rule F2, there are no hidden quarters.
- Switching all windows into rows on a drop at the bottom: the bottom magnet adds rows but
  does not discard the side columns already made (F3).
- Changing the palettes, their docking or the palettes' drop zones (`dock.rs:791-837`).
- Dragging a **tile's** title in tiled mode (a tile is rearranged by the
  `Tile Views`/`Dock All Views` buttons and by the tabs).
- Automatic window joining on every mouse move: the layout changes only on explicit actions
  (a drop into a zone, the buttons, opening/closing, window size).
- Saving the layout in `settings.json` or in `.bimcraft`; restoring it after a restart (even
  today the floating windows are not saved).
- New keyboard shortcuts and entries in the `shortcuts` command list.
- Tiling family documents: they have their own ribbon and their own editor.
- Changing the appearance of the tab bar beyond its place in the Split/Tile modes.
- Translating view names: that is project data, not UI strings.

## 5. Functional requirements

### F1. The top magnet does not change

- `Land::Tabs` and the `self.tab_bar` rectangle (`dock.rs:838-840`, `lib.rs:7281`) stay as
  they are: a window/tab released onto the bar docks as today.
- The zone label stays `Dock in the Views`.
- There are no new zones at the top edge in this task: "to the top" is exactly the tab bar.

### F2. The side and bottom magnets

- While a view tab (`Drag::View`) or a view window (`Drag::ViewWindow`) is being dragged,
  three zones are drawn:
  - **Snap Left** — a strip `EDGE` wide (36 pt, `dock.rs:154-157`) along the left edge of
    the views area;
  - **Snap Right** — the same strip along the right edge;
  - **Snap Bottom** — a strip `EDGE` high along the bottom edge, across the whole width of
    the views area.
- The side strips run from the top of the views area to `views_rect.bottom() - EDGE`, that
  is, they stop above the bottom zone. So the three zones do not overlap, and the bottom-left
  corner unambiguously belongs to the bottom zone.
- The views area rectangle is a new `self.views_rect`: the whole views region in the central
  panel (between the palettes), without the tab bar in the Split/Tile modes. It is not
  `self.work` (that one equals the active view) and not the whole application window.
- The zones are drawn with the same style as the palettes' zones (`zone_label`,
  `dock.rs:924-930`): a `pal().accent` highlight, 1–2 px thick. The side zone labels are
  rotated by ±90° (upwards on the left, downwards on the right), the bottom label is
  horizontal, centred on the strip.
- `reach` is extended beyond the views area edge to infinity: the left one — `with_min_x`,
  the right one — `with_max_x`, the bottom one — `with_max_y` (`dock.rs:912-921`). A window
  dropped beyond the edge still snaps.
- Priority: the tab bar is checked first (in the top corner a drop docks rather than splits),
  then the side and bottom zones; between themselves they do not intersect (see above).
- The zones are visible only while a view/view window is dragged, not a palette.

### F3. Rules for equal columns and rows

A new layout mode:

```rust
/// How the views' area is laid out: the standard tabbed view, equal columns and rows round
/// the tabbed view (windows snapped to the edges), or a grid of equal cells (Tile Views).
pub(crate) enum Arrange {
    Tabs,
    Split,
    Tile,
}
```

and a side on the window:

```rust
/// The edge a floating view is snapped to while the workspace is split; none floats freely.
pub(crate) enum Edge { Left, Right, Bottom }

pub struct FloatView {
    pub sheet: Sheet,
    pub at: Pos2,
    pub size: Vec2,
    pub edge: Option<Edge>,
}
```

Rules:

1. Dropping a window into `Snap Left`/`Snap Right`/`Snap Bottom` switches the layout to
   `Arrange::Split` and assigns the side to the dropped window.

2. On **entering** Split (`arrange` was `Tabs` or `Tile`), the remaining free windows
   (`edge == None`) get a side from the drop, so that the first snap does not mix the axes:

   - dropped to the left or right — each free window becomes `Left` or `Right` by position:
     centre left of the middle of `views_rect` — `Left`, otherwise `Right`; there are no
     rows;
   - dropped to the bottom — each free window becomes `Bottom`; there are no columns, one
     tab part.
     So "two open windows" give three parts along the chosen axis. On subsequent drops
     (already Split) only the dropped window changes: the others keep their sides, and the
     free ones stay free.

3. The number of columns is `left + right + 1` (the last part is the tab bar); the number of
   rows is `bottom + 1` (the top row is the columns and the tabs, the lower ones are the
   windows). Examples:

   | Drop side    | Undocked windows | Columns | Rows |
   | ------------ | ---------------- | ------- | ---- |
   | Left / Right | k                | k+1     | 1    |
   | Bottom       | k                | 1       | k+1  |
   | mixed        | l, r, b          | l+r+1   | b+1  |

   - 1 to the left → 2 columns: window · tabs;
   - 2 (left and right) → 3 columns: window · tabs · window;
   - 1 to the bottom → 2 rows: tabs · window;
   - 2 to the bottom → 3 rows: tabs · window A · window B (the new one at the very bottom).

4. Column order: left windows — from the left edge to the middle in `floating` order, then
   the tab part, then the right ones — from the middle to the right edge in `floating`
   order. Row order: the top row (columns and tabs), then the bottom windows from top to
   bottom in `floating` order.

5. The dropped window lands **at the edge it was dropped to**: for `Left` it is moved to the
   start of `floating`, for `Right` and `Bottom` — to the end. For the bottom zone this
   means the new window ends up at the very bottom and the previous ones shift upwards.

6. The column width is `views_rect.width() / cols`, the row height is
   `views_rect.height() / rows`; the last column and the last row take the rounding
   remainder, so that the sums exactly equal the views area dimensions (the discrepancy is
   0, not "±1 px"). The top row with the columns takes exactly one row height; the bottom
   windows span the whole width of the views area.

7. The tab part shows `front` and the tab bar; the active view stays active even if it is in
   a side column or a bottom row, and it gets the tools.

8. The inner padding of the cards (`style::gap()`, `style.rs:328`) is cosmetic: it does not
   change the arithmetic of the equal parts. In Compact the cards are drawn flush, in
   Floating with a gap.

9. A free (not snapped) window in Split stays floating over the columns and rows, as today;
   it adds neither columns nor rows.

10. Equality is per axis: the columns are equal in width, the rows in height. When mixing,
    the areas of the parts differ (the top row is split into columns), and that is expected:
    "equal parts" means an equal split of the chosen axis.

An example of Split with two side windows (three columns):

```
+--------------------------------------------------------------+
| [Level 1] [Level 2] [3D]                           tab bar    |
+------------------+----------------------+--------------------+
|  Level 2 (1/3)   |  tabs: Level 1       |  Level 3 (1/3)     |
|  left            |  (1/3)               |  right             |
+------------------+----------------------+--------------------+
```

An example of Split with two bottom windows (three rows, distributed from top to bottom):

```
+--------------------------------------------------------------+
| [Level 1] [Level 2] [3D]                           tab bar    |
+--------------------------------------------------------------+
|  tabs: Level 1 (row 1 of 3)                                  |
+--------------------------------------------------------------+
|  Level 2 (row 2 of 3), across the full width                 |
+--------------------------------------------------------------+
|  Level 3 (row 3 of 3) — the new window, at the bottom edge   |
+--------------------------------------------------------------+
```

A mixed example (one window each on the left, on the right and at the bottom; 2 rows, the
top one has 3 columns):

```
+--------------------------------------------------------------+
| [Level 1] [Level 2] [3D]                           tab bar    |
+------------------+----------------------+--------------------+
|  window left     |  tabs: Level 1       |  window right      |
|  (row 1 of 2)    |  (row 1 of 2)        |  (row 1 of 2)      |
+------------------+----------------------+--------------------+
|  window at the bottom, across the full width (row 2 of 2)    |
+--------------------------------------------------------------+
```

### F4. Recomputation and leaving Split

- Dropping a window out of a column or row past the zones (into the middle) makes it free
  (`edge = None`), it floats where it was released; there is one column or row less. If no
  snapped windows remain — `Arrange::Tabs`.
- `Dock {name}` in the title of a side or bottom window, and dropping its title onto the tab
  bar, dock it (the existing `dock_view`); the layout is recomputed, and with no snapped
  windows — `Arrange::Tabs`.
- Closing a side or bottom window is the same (`close_view`, `lib.rs:4343-4366`).
- Opening a new view in Split (`open_sheet`, `lib.rs:4206-4243`) adds a tab to the tab part;
  the number of columns and rows does not change.
- The last tab cannot float away (`float_view`, `dock.rs:674-685`) — the "the tab bar never
  empties" invariant is preserved.
- `Close Inactive`, after closing the extra views, sets `Arrange::Tabs` so that no layout
  with a single part is left.

### F5. Tiling (Tile Views)

- The **Tile Views** button — in **View > Windows** next to `Close Inactive`
  (`lib.rs:5623-5628`).
- Disabled when fewer than two project views are open (`views` and `floating` together,
  without family documents).
- Action: all floating windows are first **docked** to the bar (moved from `floating` to the
  end of `views` without calling `open_sheet` for each — otherwise N times per frame the
  canvases and tools would be recreated), then `Arrange::Tile`.
- The tiling is built from all open project views in `views` order; family documents
  (`Sheet::Family`) do not go into cells and remain tabs.
- Grid: `cols = ceil(sqrt(n))`, `rows = ceil(n / cols)`; the cells are equal
  (`views_rect.width()/cols` × `views_rect.height()/rows`), filled row by row from left to
  right and from top to bottom, starting at the top-left corner.
- Empty cells (when `n` does not fill the grid) remain the background of the views area
  without a frame.
- 4 views — 2×2, four corners; 3 views — 2×2, three cells; 2 views — 1×2; 5 views — 3×2.
- Each cell is drawn as a view window with a 26 pt strip (like today's floating window): the
  view name, `Close`; there is no `Dock` button in a tile, and the title is not draggable.
- A click on a cell/title makes the view active; the active cell gets the tools, the others
  are drawn passively (`view_pane`, `dock.rs:759-785`).
- The tab bar stays at the top of the views area and shows the tabs; the tiled views are the
  same tabs, so a click on a tab activates the corresponding cell.
- Pressing **Tile Views** again rebuilds the grid (after opening/closing a view it is
  recomputed on its own too).
- While tiling is on, `Float` in the tab still tears the view off: it leaves the grid and
  floats over it; the grid is recomputed for `n-1`. Dropping such a window into a side or
  bottom zone switches the layout to Split (F3): when dropped to the bottom, the remaining
  tabs stay in the top row and the window takes the bottom one — two equal rows.
- The tile cells do not depend on `edge`: the windows' sides are kept in `FloatView` in case
  of a return to Split, but are not used in `Tile`.

An example of tiling four views:

```
+--------------------------------------------------------------+
| [Level 1] [Level 2] [{3D}] [Section A]             tab bar    |
+---------------------------+----------------------------------+
|  Level 1                  |  Level 2                         |
+---------------------------+----------------------------------+
|  {3D}                     |  Section A                       |
+---------------------------+----------------------------------+
```

### F6. Gathering into tabs (Dock All Views)

- The **Dock All Views** button — in the same `Windows` panel.
- Enabled when there is at least one floating window or the layout is not `Tabs`.
- Action: all floating windows (including side, bottom and free ones) are moved to the end
  of `views` in the order of their on-screen appearance (for Split — the top row from left
  to right, then the bottom rows from top to bottom; for Tile — row by row),
  `Arrange::Tabs`, the active view stays active, `front` is the same (or the last tab if the
  active view is not a tab).
- After that the names of all views are again on the tab bar at the top of the views area;
  there are no floating windows; `floating.is_empty()`.
- Docking happens without `open_sheet` per view: the lists are changed, then the active view
  is restored, so as not to jolt the canvases and the tools.

### F7. Sizes, resizing, minimums

- Snapped and tiled windows **cannot be resized** by the corner: the size is set by the
  column, row or cell (`resizable(false)`), and the resize cursor at the window edge does not
  break the equality of the parts.
- A window's free `at`/`size` are **not overwritten** while it is snapped/tiled: on
  undocking the window returns to its previous size and place (and the position — to the
  drop point).
- The floating window constraint `min_size(240×160)` (`dock.rs:711`) does not apply to
  columns, rows and cells: with a large number of parts the window is drawn narrower than
  240 pt, equality matters more.
- The layout is computed **every frame** from `views_rect`, so a change of the application
  window size, showing/hiding palettes and switching Compact/Floating immediately recompute
  the columns and cells; the sum of the column widths equals the views area width, the sum
  of the row heights equals its height.
- A window position outside the screen is, as now, limited by `FAR` (`dock.rs:158-159`) only
  for free windows.

### F8. Families, the start page, invariants

- Family documents do not float, are not split into columns and rows and do not go into the
  tiling; their tabs stay on the bar and are available in Split/Tile.
- When a family document is active, the family editor is drawn as today (`family_central`,
  `lib.rs:2416-2420`), and the project's windows/tiles are **not** drawn over it.
- The start page has no views area: there are no magnets or layout buttons on it.
- `tidy` (`lib.rs:2541-2611`):
  - if no snapped windows remain — `Arrange::Tabs`;
  - if there are fewer than two project views — `Arrange::Tabs`;
  - `edge` lives in `FloatView` and leaves together with the window; no separate cleanup is
    needed;
  - the layout is recomputed from the lists, so no "dead" rectangles remain.
- `Close Inactive` (`lib.rs:5623-5628`) additionally sets `Arrange::Tabs`.
- A new/opened project replaces `BimApp` (`lib.rs:4536-4585`), `Arrange` defaults to `Tabs`,
  and `floating` is empty.

### F9. Frame layout and the tab bar

- In `Arrange::Tabs` everything is as today: the tab bar is a card over the view (Floating)
  or `Panel::top` (Compact).
- In `Arrange::Split` and `Arrange::Tile` the tab bar is `Panel::top("view-tabs")` across the
  full width of the views area (as in Compact today), `tabs_foot = f32::NEG_INFINITY`, and
  `views_rect` is the remainder below it.
- In Split the tab part is drawn in its own column; in Tile no part is drawn the usual way,
  all cells are drawn as view windows.
- `self.work` (the edges for docking palettes) must point at the **active** view in the new
  modes too: in Split/Tile it is taken from the rectangle of the active cell, column or row,
  not from the previous "view under the tabs". Otherwise the palettes would dock to someone
  else's part.
- `self.canvas.room` in Split/Tile reserves space only for the floating view control panel
  (0 in Compact): the tab bar no longer floats over the view there.
- An empty tile cell is the background of the views area, without a frame and without a
  label.

### F10. Icons, strings, accessibility

- New icons in `crates/ui-egui/src/icons.rs` (24×24 grid, like the neighbours):
  - `Icon::Tile` — a 2×2 square;
  - `Icon::DockAll` — a tab bar above a panel.
    Both are added to `icon_names!` and to `paint`; the test that draws `Icon::ALL`
    (`icons.rs:2020`) picks them up on its own.
- New strings (all through `t`/`msg!`):
  - `Snap Left`, `Snap Right`, `Snap Bottom` — the zone labels;
  - `Tile Views`, `Dock All Views` — the buttons;
  - tooltips: `Tile Views: every open view in an equal cell, four of them in the four
    corners` and `Dock All Views: every floating view back onto the tab bar, in one view
    under the tabs`.
- After the edits — `cargo i18n update` (updates `bimcraft-ui.pot` and all `.po`); unfilled
  `msgstr` are acceptable, English is shown as the fallback.
- The buttons are ordinary `big_button` with a tooltip; the zones have labels like the
  palettes; the windows and cells have `WidgetInfo::labeled` as now (`dock.rs:715-717`).

### F11. Nothing outside the UI

- No command, JSON schema, `.bimcraft`, IFC, MCP/API changes.
- The layout does not create undo steps and does not make the project "dirty".
- `settings.json` does not change: `Arrange`/`edge` are window state, like `floating`.
- There are no keyboard shortcuts or `shortcuts` entries.

## 6. Key decision and alternatives

### 6.1. Recommended solution: one layout computation per frame

The view window layout is a pure function of `views_rect`, the `views`/`floating` lists,
`Arrange` and `edge`. It is computed once per frame in `show()` and stored in

```rust
/// The views' area as this frame lays it out: the tabbed pane and every window beside it.
pub(crate) placements: Vec<(Sheet, Rect)>,
pub(crate) tab_pane: Rect,
```

- `Tabs`: `tab_pane = views_rect`, `placements` is empty; the floating windows are drawn
  from their `at`/`size`, as today;
- `Split`: `tab_pane` — the middle column of the top row; `placements` — the side columns and
  the bottom rows (first the top row, then the bottom windows from top to bottom);
- `Tile`: `placements` — all cells row by row, `tab_pane = Rect::NOTHING`.

The same `placements` are used by the drawing (`view_windows`), by the tests and by
`self.work` (the active view's rectangle). One function eliminates the mismatch "drawn not
where it was computed" and makes recomputation on resize free.

The magnets are new variants of the existing drop zone:

```rust
enum Land {
    Side(Side),          // palettes
    With(Palette),
    Before(Palette),
    Tabs(usize),         // the tab bar, as today
    Snap(Edge),          // new: the left/right column or the bottom row of a view window
}
```

The zone strips do not intersect: the side ones run down to `views_rect.bottom() - EDGE`,
the bottom one occupies the whole width at the bottom. Therefore exactly one zone triggers
in a corner, with no hidden quarters.

### 6.2. Rejected options

| Option                                                            | Why not                                                                                                                                                    |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Snapping to the **OS window** edges (`ViewportCommand`)           | We speak about the views area inside the application; the system Snap already works and has nothing to do with splitting views                             |
| Splitting into parts by the number of all tabs only               | The tab bar is one visible part; "one window — two parts" would then not hold                                                                              |
| Snapping only the dropped window and leaving the rest floating    | The user expects "two windows — three parts"; the free windows must take their columns or rows when entering Split                                          |
| The bottom magnet switching all windows into rows                 | It would discard the columns already made and move windows the user did not touch; the first drop to the bottom gives the same result, and later ones are more predictable |
| A quarter layout in a corner (both width and height at once)      | Not requested; one zone triggers in a corner (the bottom one), quarters stay hidden                                                                        |
| A bottom window in a column rather than across the full width     | "Top to bottom" means rows; a row across the full width reads well and does not split the lower windows into narrow columns                                 |
| Storing the column and row rectangles in `FloatView`              | They go stale on resize and palette changes; the layout must be computed every frame                                                                        |
| Turning the tiling into `floating` with coordinates               | Breaks the "the tab bar never empties" invariant and `tidy`; the open-view lists must stay as they are                                                      |
| Tiling only the floating windows                                  | Four open views must give four corners, even if all of them are tabs                                                                                       |
| Tiling with a different width for the last row                    | "Equal parts" matters more than filling: the last row leaves empty cells, all cells are the same size                                                       |
| Applying the `min_size` limit in columns and rows                 | With 5+ parts the window is wider than its column and equality breaks; drawing the view narrower is fine                                                   |
| Saving the layout in `settings.json`                              | Even today the floating windows are not saved; that is a separate task                                                                                      |
| New keyboard shortcuts                                            | Not requested; the `shortcuts` list and key conflicts are not touched                                                                                      |

## 7. Edit points

| File                                                      | What to change                                                                                                                |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `crates/ui-egui/src/dock.rs:442-448`                      | `FloatView.edge`, `Edge` (Left/Right/Bottom), `Arrange` (or in `lib.rs` next to `Sheet`)                                       |
| `crates/ui-egui/src/dock.rs:674-693`                      | `float_view` (edge = None), `dock_view` (exit to Tabs when no snapped windows remain), silent docking for Tile/Dock All       |
| `crates/ui-egui/src/dock.rs:700-757`                      | `view_windows`: draw `placements` (Split/Tile) with a shared "view window" helper; in Tabs — as today                           |
| `crates/ui-egui/src/dock.rs:791-842`                      | `landings`: new `Land::Snap(Edge)` for `Drag::View`/`Drag::ViewWindow`, three zones                                            |
| `crates/ui-egui/src/dock.rs:849-909`                      | `drag_and_drop`: a drop into `Snap` → `snap_view`; the tab bar — as today                                                      |
| `crates/ui-egui/src/dock.rs:912-921`                      | `reach` for `Snap`: beyond the views area edge to infinity (left, right, bottom)                                               |
| `crates/ui-egui/src/lib.rs:617-661`                       | the `arrange`, `views_rect`, `placements`, `tab_pane` fields                                                                    |
| `crates/ui-egui/src/lib.rs:2416-2445`                     | `show()`: a Split/Tile branch with `Panel::top("view-tabs")`, computing `views_rect` and `placements`, `self.work` from the active view |
| `crates/ui-egui/src/lib.rs:2495-2524`                     | `view_body`: `canvas.room` without the tab bar in Split/Tile; `work` from the active view                                       |
| `crates/ui-egui/src/lib.rs:2541-2611`                     | `tidy`: reset `Arrange` per the invariants                                                                                      |
| `crates/ui-egui/src/lib.rs:4343-4366`                     | `close_view`: recompute the layout                                                                                              |
| `crates/ui-egui/src/lib.rs:5490-5653`                     | `view_tab`, the `Windows` panel: the `Tile Views`/`Dock All Views` buttons, `Close Inactive` → Tabs                             |
| `crates/ui-egui/src/icons.rs:25-84`, `:400+`              | `Icon::Tile`, `Icon::DockAll` and their drawing                                                                                 |
| `crates/ui-egui/src/tests/dock.rs`                        | tests T1–T13                                                                                                                    |
| `translations/ui/bimcraft-ui.pot`, `translations/ui/*.po` | `cargo i18n update` after the edits                                                                                             |

`crates/core`, `crates/i18n/src`, the `.bimcraft` format, the commands and `settings.rs` do
not change.

## 8. Test plan (to add; the run is done by CI/the user)

The tests go to `crates/ui-egui/src/tests/dock.rs`, in the existing style: `app(BOX)`,
`two_plans()`, `drag_screen`, `press`, `h.state()`. The `arrange`, `views_rect`,
`placements`, `floating[i].edge` fields are available to the tests just as `views`,
`tab_bar`, `tab_rects` already are.

A shared helper:

```rust
/// The rectangles of the windows off the tab bar, left to right, top to bottom.
fn cells(h: &Harness<BimApp>) -> Vec<(Sheet, Rect)> { h.state().placements.clone() }
```

### T1. One window undocked to the side — two equal columns

1. `two_plans()`; `open_sheet(two)`; `float_view(two, pos2(500.0, 300.0))`.
2. Drag the `Level 2` window title into the left zone (`views_rect.left() + 8.0`,
   vertically — the middle).
3. `arrange == Split`; `floating[0].edge == Some(Edge::Left)`; there are two parts:
   `placements.len() + 1 == 2`.
4. The widths of `placements[0].1` and `tab_pane` are equal to within 1e-3; their sum equals
   `views_rect.width()`.
5. The window is at the left edge, the tab part is on the right.

### T2. Two windows — three equal columns

1. `two_plans()`; open a third view (`open_sheet(Sheet::View3D)`), undock `two` and `View3D`
   from the bar.
2. Drop `two` to the left, then `View3D` to the right.
3. `placements.len() + 1 == 3`; the order is `[Left, tabs, Right]`; the widths are equal.
4. The second dropped window is at the right edge (a new window lands at the edge it was
   dropped to).

### T3. The top magnet is not broken

1. `two_plans()`; `float_view(two, …)`.
2. Drag the title onto `tab_bar` (the centre of the bar).
3. `floating.is_empty()`; `views == [one, two]`; `arrange == Tabs`.

### T4. Tiling four views — four corners

1. Open four project views (two plans, ceiling, `{3D}`), undock two of them into windows.
2. Press `Tile Views`.
3. `arrange == Tile`; `placements.len() == 4`; the cells are 2×2; the first two are at the
   top edge, the second two at the bottom; the left ones at the left edge, the right ones at
   the right.
4. All widths are equal, all heights are equal; the sum of two widths = `views_rect.width()`,
   of two heights — `views_rect.height()`.
5. `floating.is_empty()`: the tiling gathered the windows onto the bar.

### T5. Tiling three views — a 2×2 grid with an empty cell

1. Open three views; `Tile Views`.
2. `placements.len() == 3`; two cells in the top row, one in the bottom left; the bottom
   right corner is empty.
3. The sizes of the three cells are identical.

### T6. Dock All returns the tabs

1. Build the T4 state (four views, tiled), then press `Dock All Views`.
2. `arrange == Tabs`; `floating.is_empty()`; `views.len() == 4`;
   `tab_rects.len() == 4`; the active view is the same.
3. `placements` is empty, `tab_pane == views_rect`.

### T7. Undocking from a column

1. Build the T2 state (two windows, left and right), then drag the right window's title into
   the middle of the views area.
2. `edge == None`; it floats at the drop point; there are two parts again (the left window +
   the tabs).
3. Remove the left window from the column too — `arrange == Tabs`, one part.

### T8. Resize and equality

1. Build the T2 state (three equal columns), then change the application window size
   (`h.set_size`/a new `Harness` with a different size — as in the existing view tests).
2. `views_rect` changed; the widths of all parts are equal again, the sum equals the width.
3. For the T10 state (three rows) — the same in height: the heights are equal, the sum
   equals the height.
4. `floating[i].at/size` did not change: the window's free size is preserved.

### T9. The bottom magnet: one window — two equal rows

1. `two_plans()`; `open_sheet(two)`; `float_view(two, pos2(500.0, 300.0))`.
2. Drag the window title into the bottom zone (`views_rect.center().x`,
   `views_rect.bottom() - 8.0`).
3. `arrange == Split`; `floating[0].edge == Some(Edge::Bottom)`; there are two rows:
   `placements.len() + 1 == 2`.
4. `tab_pane` is the top row across the full width, the window is the bottom row across the
   full width; the heights are equal to within 1e-3, the sum equals `views_rect.height()`.
5. The top row shows the tabs, the bottom one shows the window (the windows are distributed
   from top to bottom).

### T10. Two bottom windows — three equal rows

1. Three views: `two_plans()` and `open_sheet(Sheet::View3D)`; undock `two` and `View3D`.
2. Drop `two` to the bottom, then `View3D` to the bottom.
3. `placements.len() + 1 == 3`; the order from top to bottom: `tab_pane`, `two`, `View3D`.
4. The new window is at the very bottom, at the bottom edge; the heights of all three rows
   are equal, the widths span the whole views area.

### T11. Mixed layout: columns and rows

1. Open four views (two plans, ceiling, `{3D}`); undock three views into windows.
2. Drop the first window to the left (entering Split: the remaining free ones get
   Left/Right by position), then one of them — to the bottom.
3. `arrange == Split`; there are 3 columns in the top row (`left + right + 1`), 2 rows
   (`bottom + 1`).
4. The top row: three columns equal in width, its height is half the views area. The bottom
   row: a window across the full width, the height being half.
5. The tab part is the middle column of the top row.

### T12. The bottom-left corner belongs to the bottom zone

1. In the Split state, bring the window to the point (`views_rect.left() + 8.0`,
   `views_rect.bottom() - 8.0`) and release.
2. `edge == Some(Edge::Bottom)`, not `Left`: the side strips end above the bottom zone.
3. A drop a little higher (`views_rect.bottom() - 2 * EDGE`) gives `Left`.

### T13. Hygiene

`cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
`cargo clippy -p bimcraft-ui-egui`, `cargo i18n check` pass. The new icons are drawn by the
`Icon::ALL` test (`icons.rs:2020`).

## 9. Acceptance criteria

- [ ] At the left, right and bottom edge of the views area there are the `Snap Left`/
  `Snap Right`/`Snap Bottom` zones.
- [ ] Dropping a view window into the left zone puts it as a column on the left, into the
  right one — on the right.
- [ ] Dropping a view window into the bottom zone puts it as a row at the bottom; the windows
  are distributed from top to bottom.
- [ ] One undocked window gives two equal parts, two — three, `k` — `k+1` along the chosen
  axis.
- [ ] The column widths are equal, the row heights are equal; the sums equal the views area
  dimensions (the last ones take the remainder).
- [ ] Side and bottom windows mix: the width is split into columns, the height into rows; the
  top row contains the columns, the bottom windows span the full width.
- [ ] The bottom-left corner belongs to the bottom zone; the side strips end above it.
- [ ] The tab part shows `front` and the tab bar; the active view stays active.
- [ ] The top magnet (the tab bar) works as before.
- [ ] `Tile Views` in View > Windows: 4 views — in the four corners, 3 — in three cells of a
  2×2 grid, 2 — in two, 5 — in five cells of a 3×2 grid; the cells are equal.
- [ ] `Dock All Views` returns all views to the tab bar, the view names are visible at the
  top.
- [ ] Family documents do not float, are not split and are not tiled; their tabs are
  available.
- [ ] `Close Inactive` and a new/opened project return the ordinary view.
- [ ] The layout is recomputed on window resize and palette changes.
- [ ] Snapped/tiled windows are not resizable, the free `at/size` is preserved.
- [ ] The new strings are translated through the catalogues; `cargo i18n check` is green.
- [ ] The model, the `.bimcraft` format, commands, IFC and the API have not changed; undo is
  not written.
- [ ] `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
  `cargo clippy -p bimcraft-ui-egui` pass.
- [ ] The tests were not run locally unless the user asked; this is reflected in the PR.

## 10. Risks and open questions

1. **What counts as a "window".** In this spec a "window" is a view taken off the tab bar;
   the tab bar is always one part. This gives literally "one window — two parts, two —
   three". If the user meant "all open views, including the tabs", the count would be
   different; then a clarification is needed before implementation.
2. **The top magnet.** In the code "to the top" is the tab bar (`Land::Tabs`), not the
   monitor edge. The task does not change it; if in the user's build the top magnet behaves
   differently, that is a separate fix.
3. **Very narrow columns and low rows.** With 6+ parts the width or height is less than
   reasonable; the window is drawn smaller, without the `min_size` limit. This is a
   deliberate choice in favour of equality; the view is still available, just smaller.
4. **Free windows over Split.** A window undocked past the zones floats over the columns and
   rows and overlaps them; today's floating window over the view behaves the same way.
5. **Mixing axes.** With side and bottom windows "equal parts" are counted per axis
   separately: the columns are equal in width, the rows in height, the areas differ. This is
   a direct consequence of "top to bottom" and "left/right"; if equal areas are wanted, that
   is a different layout (a grid), and `Tile Views` provides it.
6. **The bottom-left corner.** It belongs to the bottom zone: the side strips end above it.
   There is no quarter layout — this is a deliberate simplification, not an omission.
7. **Tiling docks the floating windows.** After `Tile Views` they are already tabs; returning
   to the previous "semi-floating" state is possible only through a new undock. This
   simplifies the invariants and makes `Dock All` predictable; noted as a decision, not a
   defect.
8. **Family documents.** Tiling does not touch them; an active family document takes the whole
   views area, and the project's windows are not drawn over it.
9. **The Windows panel is narrow.** The two new buttons lengthen the panel; when space runs
   out, it goes into the list, as today — the `big_button` behaviour does not change.
10. **Translations.** The English strings appear immediately; translations may stay empty
    and be filled in later — `check` allows that.
11. **The layout is not saved.** After a restart — ordinary tabs; this is consistent with the
    fact that even today `floating` is not saved.

## 11. Implementation order and DoD

1. Add `Arrange`, `Edge` (Left/Right/Bottom), `FloatView.edge`; update
   `float_view`/`dock_view`.
2. Introduce `views_rect`, `placements`, `tab_pane`; move the layout computation (columns and
   rows) into one function.
3. Switch `show()`/`view_body` to the new layout; `work` — from the active view.
4. Add `Land::Snap`, the `Snap Left`/`Snap Right`/`Snap Bottom` zones, `snap_view` and rules
   F2–F4.
5. Teach `view_windows` to draw the side windows, bottom rows and the tiling from
   `placements`.
6. Add `Tile Views`/`Dock All Views` to `Windows`, the `Tile`/`DockAll` icons.
7. Update `tidy`, `Close Inactive`, `close_view`.
8. Add tests T1–T13.
9. `cargo i18n update`; check `cargo i18n check`.
10. Run `cargo fmt --check`, `cargo check -p bimcraft-ui-egui`,
    `cargo clippy -p bimcraft-ui-egui`.
11. Do not run the tests without an explicit request; state in the PR that CI will run them.
12. Split into commits:
    - `ui: snap view windows to the edges in equal columns and rows`;
    - `ui: Tile Views and Dock All Views in the View tab`;
    - `ui: tests for snapped and tiled view windows`;
    - `i18n: the view window magnets and tiles`.
13. Definition of Done: the acceptance criteria are met, the model/format/commands have not
    changed, there are no new dependencies and no `unsafe`.

## 12. What not to do

- Do not touch the system Snap and the OS window title.
- Do not change the drop zones and the behaviour of the palettes.
- Do not save the layout in `settings.json` or `.bimcraft`.
- Do not turn the tiling into a set of floating windows with coordinates.
- Do not change the "parts" formula to the number of all tabs: the tab bar is one part.
- Do not switch the side columns already made into rows on a drop at the bottom: the axes
  mix, they do not replace each other.
- Do not make a quarter layout in the corners: one zone triggers in a corner.
- Do not limit the column width or row height with `min_size` — that breaks equality.
- Do not make a tile's title draggable and do not add `Dock` to a tile.
- Do not tile or tear off family documents.
- Do not add keyboard shortcuts or `shortcuts` entries.
- Do not change the model, the `.bimcraft` format, commands, IFC, MCP/API or undo.
- Do not translate view names: that is project data.
