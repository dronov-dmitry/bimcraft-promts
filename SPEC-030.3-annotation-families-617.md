# SPEC-030.3. Annotation families: follow-ups from discussion #617

Status: **draft**, implementation required
Date: 2026-11-25 (shortened: items duplicated from SPEC-030.2 replaced by references)
Project: BIMCraft (Rust; `crates/core/src/commands.rs` — `annotation`, `family`, `modify`,
`elements`, the new `family_from_group`; `crates/core/src/family/annotation.rs`,
`crates/core/src/model.rs`, `crates/core/src/groups.rs`, `crates/core/src/render.rs`,
`crates/core/src/sheet.rs`; `crates/ui-egui/src/browser.rs`, `crates/ui-egui/src/properties.rs`,
`crates/ui-egui/src/canvas.rs`)
Source: discussion IPedrax/BIMCraft#617 "Parametric 2D detail families (annotation components
with types)" — author `linjiahljh`, November 24, 2026; review of the `annotation-on-main` branch
(PR #600).
Related: SPEC-030.1 (what already works), SPEC-030.2 (the overall list of what is left and the
only status tracker), SPEC-030 (the original requirements), PR #600, discussion #617.
Task type: follow-up work on annotation families for office symbols; there are no new document
kinds, except converting a detail group into a family.

> **In short.** About 90% of the #617 wishes are already done in PR #600: a 2D family with
> parameters, formulas, types and fills, placement, rotation, mirroring and instance parameters.
> What is missing is the reason office symbols exist at all: sheet fields in labels, detail view,
> nesting, limits and tests. A separate Detail Item category, parameters on detail groups and
> instance scaling are deliberately rejected (see §3). Legacy libraries move over with the
> `family_from_group` command.
>
> **This file contains only what is missing from SPEC-030.2**: the mapping to #617, the rationale
> for the non-goals and the conversion of a detail group into a family. Sheet fields, detail view,
> placement, Properties, nesting, `Reload`, limits, tests, export and documentation are described
> and tracked in SPEC-030.2 — here there are only references to it.

## 0. General working rules

The working rules (language, tests, atomic commits, CI, thin UI, format compatibility) — SPEC-030.2
§0. In addition:

1. **Work is not duplicated with SPEC-030.2.** If an item is already there, it is not described
   again here but referenced. Status is tracked only in SPEC-030.2.
2. The `.bimc`/`.bimf` format does not change; new fields — `serde(default)`.

## 1. Summary

| #617 wish | Status | Where |
|---|---|---|
| Family document: parameters (length, angle, text, yes/no), a sketch of lines/arcs/fills/text, types | **done** | SPEC-030.1 §5–§7 |
| Placement, rotation, mirroring; parameters per instance | **done** (except detail views) | SPEC-030.1 §8 |
| No bodies and no IFC for a 2D family | **done** | SPEC-030.1 §6 |
| A separate Detail Item category | rejected | §3 |
| Parameters on detail group types | rejected, replaced by conversion | §3, §6 |
| Instance scale | rejected: scale is a property of the view | §3 |
| Office revision stamp, level marker with a tail, section number | not done: no sheet fields | SPEC-030.2 §7 |
| Placement in drafting/detail views | not done | SPEC-030.2 §5.1 |
| Composite symbols | not done: no nesting | SPEC-030.2 §8.1 |
| A frame/stamp with hundreds of symbols | not done: `MAX_SYMBOLS` = 64 | SPEC-030.2 §9.2 |
| Tests: a parameter moves the members; place/rotate/mirror; save/open; no IFC | not done | SPEC-030.2 §11 |

The discussion author works like this: the model is not finished, the drawing is completed with
annotation; right now he keeps two sets of symbols — built-in tags and static detail groups. The
family closes the second set, but only after sheet fields and detail views.

## 2. Goals

1. Answer #617 in terms of the existing category: the family already exists, the follow-ups are
   targeted.
2. Existing detail groups convert into families without redrawing.
3. The remaining goals (sheet fields, detail view, placement, nesting, limits, test plan) —
   SPEC-030.2 §2.

## 3. Non-goals

- **A separate Detail Item category.** Not introduced: the set of tools, parameters and placement
  rules matches `Annotation`; a second category would add a second document kind in the browser,
  Visibility/Graphics, filters, schedules, export and i18n with no functional difference. If the
  word "detail" is needed in the UI, it is a label, not an architecture.
- **Parameters on `GroupType`.** Not done: `GroupType { members: Vec<Member>, levels }` stores
  concrete `Element`s with concrete coordinates. Parameterisation would require replacing the
  geometry of every element with a "number-or-formula" and recomputing it per instance — that is,
  repeating the family mechanism (sketch + parameters + eval) inside groups and losing member
  identity, Edit Group and the conflict rule. Instead — conversion (§6).
- **Instance scale.** Not done: scale is a property of the view. Everything that looks like
  scaling a symbol is expressed by parameters (arrow length, tail length, text height).
- Other non-goals (raster images and logos, replacing `title_block`, annotations in 3D/IFC/GLB, a
  family library of its own and cloud editing) — SPEC-030.2 §3.

## 4. Key decision

**All follow-ups build on `Category::Annotation` and the existing family engine** (in detail —
SPEC-030.2 §4). The only decision that is not in 030.2:

- Legacy libraries move over with the `family_from_group` command: lines, arcs, text and fills map
  1:1, dimensions are dropped with a warning.

## 5. The remaining work

Described and tracked in SPEC-030.2; here — only a map of references.

| Work | Where it is described |
|---|---|
| Sheet fields in labels: keys, rules, family parameter precedence | SPEC-030.2 §7 |
| Detail view as a placement target | SPEC-030.2 §5.1 |
| Drag from the browser, `Space`/`Tab`/`Esc`, snapping | SPEC-030.2 §5.2–5.3 |
| Instance Properties, context menu, the Annotations group in the browser | SPEC-030.2 §6 |
| Nested annotation family | SPEC-030.2 §8.1 |
| `Reload`, dimensions in paper mm | SPEC-030.2 §8.2–8.3 |
| PDF/SVG/PNG and `publish`; absence from IFC/GLB | SPEC-030.2 §9.1 |
| `MAX_SYMBOLS`/10 000 limits and the `PaperMark` cache | SPEC-030.2 §9.2 |
| Line styles and visibility | SPEC-030.2 §9.3 |
| Format compatibility | SPEC-030.2 §10 |

## 6. Stage E. Converting a detail group into a family

Closes "two parallel sets of symbols" without manual redrawing.

### 6.1. Command

`family_from_group {id | type, name?, scale_with_view?, place?}` — builds a `.bimf` family of the
Annotation category from a detail group type.

### 6.2. Mapping the members

| Group member | Family symbol | Note |
|---|---|---|
| Detail line (any curve) | `SymbolKind::Line` | paper mm: `paper = model / view_scale` |
| Arc | `SymbolKind::Arc` | centre and radius in paper mm |
| Text | `SymbolKind::Text` | text height is divided by the view scale |
| Filled region | `SymbolKind::Fill` | each ring in paper mm |
| Dimension | — | dropped with a warning and a list of ids |

- Coordinates are taken relative to the group's `at`; the group's rotation and mirroring are
  removed.
- Line styles and fill types are carried over by name; a missing style — a warning and
  `Thin Lines`.
- `place: true` places one instance where the group stood and returns its id; without the flag
  the original group stays untouched.
- The result opens in the family editor (or is selected in the browser).

### 6.3. Acceptance

- A group of lines, arcs, text and fill converts 1:1 to the paper sizes of the view.
- A group with dimensions: the family is created, the dimensions are listed in `skipped`.
- A group with rotation and mirroring: the family is straight, the instance stands at the same
  angle.
- Save/Open `.bimf` and reloading do not change the geometry.

## 7. Test plans

The test plan — SPEC-030.2 §11. In addition to it:

1. `family_from_group`: lines/arcs/text/fill, dimensions in `skipped`, rotation and mirroring,
   `.bimf` round trip.

> Tests in this SPEC are described but are not run without a separate request; CI is separate.

## 8. i18n and documentation

- `docs/areas/groups.md` — `family_from_group` and why parameters on group types did not appear.
- The remaining strings and documents — SPEC-030.2 §12.

## 9. Acceptance criteria

- [ ] `family_from_group` carries geometry 1:1 and lists the losses.
- [ ] No separate Detail Item category, no parameters on `GroupType`, no instance scale have
      appeared.

The remaining criteria — SPEC-030.2 §13.

## 10. Risks and questions

| Risk | Answer |
|---|---|
| Model→paper conversion in `family_from_group` | divide by the view scale once, test at 1:5 and 1:100 |
| Dimensions in a group are lost | list them in `skipped`; the user decides whether they are needed |
| A line style or fill type is missing in the project | a warning and `Thin Lines`/default |
| A group stands in one view, the family is everywhere | it is geometry that is converted, not the binding to a view |

1. Is `family_from_group` needed with `place` straight away, or is it enough to create the family
   and leave placement to the user?
2. What to do with dimensions during conversion: drop them with a warning or turn them into tick
   lines (the look is preserved, the meaning is lost)?

The remaining questions — SPEC-030.2 §14.

## 11. Definition of Done

- [ ] `family_from_group` with a loss map and a `place` option.
- [ ] No separate Detail Item category, no parameters on `GroupType`, no instance scale have
      appeared.
- [ ] `check`, `build`, `clippy`, `fmt --check`, `i18n check` — green (per the project rules).

The remaining items — SPEC-030.2 §15.

## 12. Code map

- Commands: `crates/core/src/commands.rs` (`family_from_group`).
- Groups: `crates/core/src/groups.rs` (`GroupType`, `Member`, `capture`, `place`).
- Family content: `crates/core/src/family/annotation.rs` (`PaperMark`, `marks`, `bounds`,
  `check`), `crates/core/src/model.rs` (`Annotation`, `DetailView`).
- Documentation: `docs/areas/groups.md`.
- The remaining files (rendering, sheets, UI) — SPEC-030.2 §16.
