# SPEC-025. File format 20→21 migration: line patterns of the starter grid types

Status: **specification**, implementation and a test run in CI are required
Date: 2026-11-19
Project: BIMCraft (Rust, `BIMCraft-024`, file format 20 → 21)
Source: user request of 2026-11-19: on opening the file
`Budynok-90-KD-03.bimc` (format 20) the application answers
`open: …/Budynok-90-KD-03.bimc: grid type `Grid - ASME Y14.2`: there's no line
pattern `Grid - ASME Chain` (there are: Solid, Centre, Dash, Dash Dot, Dot,
Hidden, Long Dash)`.
Task type: fixing the project-file migration. The model, the commands and the UI
do not change; only what the migration adds to an old file changes.

> **In short.** Format 21 introduced grid types, and when reading a format-20 file
> `Model::migrate()` calls `typed_grids()` (`crates/core/src/model.rs:5546-5560`).
> It creates a starter set of six grid types (`crate::grids::types`) that
> reference five `Grid - …` patterns. But those patterns themselves are **not**
> added to `line_patterns`: the old file holds only the six former patterns
> (`Centre`, `Dash`, `Dash Dot`, `Dot`, `Hidden`, `Long Dash`). Right after
> reading, `open` calls `Model::check()` (`crates/core/src/commands.rs:4921-4923`),
> and `GridType::check` (`crates/core/src/grids.rs:92-106`) fails on the missing
> pattern. The migration must add the missing patterns together with the starter
> grid types, without overwriting project patterns that have the same name.

## 0. General working rules

1. The language of communication is Russian. Code, comments, interface strings and
   log messages are English. The document (`.md`) is Russian. (This is a
   translation of the specification; the original Russian document is
   `ТЗ-025-миграция-паттернов-сеток.md`.)
2. Tests (`cargo test`, `cargo nextest`, doctest) are **not run** in this task
   without a direct request from the user (rule
   `~/.config/opencode/rust-rules.md`). `cargo fmt --check`,
   `cargo check -p bimcraft-core` and `cargo clippy -p bimcraft-core` (without
   `--all-targets`) are allowed.
3. No new dependencies. No `unsafe` is added, lints are not relaxed.
4. The file format stays 21. The behaviour of commands, model checks, the UI and
   the API does not change: only the old-file migration is fixed.
5. Commits are atomic: the migration separately, the regression tests separately.

## 1. Symptom

Reproduction:

1. Take a project saved by a build that predates grid types (format 20), for
   example `Budynok-90-KD-03.bimc` or `Budynok-90-KD.bimc` from
   `/home/kat/Documents/01_ArchProjects`. Such files have:
   - `"format": 20`;
   - no `grid_types` key;
   - no `types.stair` and `types.toposolid` (markers of format 20);
   - exactly six old patterns in `line_patterns`;
   - grids present (`elements` with `"category": "grid"`), but without a `"type"`
     field.
2. Open the file in the current build (format 21).
3. An error appears:

```
open: /home/kat/Documents/01_ArchProjects/Budynok-90-KD-03.bimc:
grid type `Grid - ASME Y14.2`: there's no line pattern `Grid - ASME Chain`
(there are: Solid, Centre, Dash, Dash Dot, Dot, Hidden, Long Dash)
```

Expected behaviour:

- the file opens;
- every grid receives the type `Grid - ISO 128` (or its translation for the
  project language);
- the missing patterns appear in `line_patterns`:
  `Grid - ISO Dash Dot`, `Grid - DIN Dash Dot`, `Grid - ASME Chain`,
  `Grid - Architectural Chain`, `Grid - Compact Dash Dot` (in the names of the
  project language);
- existing project patterns with the same names are not overwritten;
- `Model::check()` passes;
- after saving, the file is read again without errors.

## 2. Diagnosis

### 2.1. The read and check path

`crates/core/src/commands.rs:4897-4923`:

```rust
let (mut m, left_out) = Model::read_within(&text, &budget)…?;
…
m.sync_library();
m.check().map_err(|e| bad(c, format!("{}: {e}", path.display())))?;
```

`Model::read_*` → `after_read` (`crates/core/src/model.rs:5232-5234`) calls
`m.migrate()`. At the end of `migrate()`, `typed_grids()` is called
(`crates/core/src/model.rs:5361-5363`).

### 2.2. What the migration does

`crates/core/src/model.rs:5546-5560`:

```rust
fn typed_grids(&mut self) {
    if self.grid_types.is_empty() {
        self.grid_types = crate::grids::types(self.language);
    }
    let wanted = crate::grids::default_type(&self.grid_types, self.language).cloned().unwrap_or_default();
    for e in self.elements.values_mut() {
        if let Element::Grid(g) = e
            && g.type_name.is_empty()
        {
            g.type_name = wanted.clone();
        }
    }
}
```

`crate::grids::types` (`crates/core/src/grids.rs:125-139`) creates six types;
five of them reference the patterns declared in `grids.rs:27-31`:

| Grid type | Pattern |
|---|---|
| `Grid - ISO 128` | `Grid - ISO Dash Dot` |
| `Grid - DIN 1356` | `Grid - DIN Dash Dot` |
| `Grid - ASME Y14.2` | `Grid - ASME Chain` |
| `Grid - Architectural` | `Grid - Architectural Chain` |
| `Grid - Continuous` | none (`None`) |
| `Grid - Compact` | `Grid - Compact Dash Dot` |

The definitions of these five patterns live in `crate::lines::patterns`
(`crates/core/src/lines.rs:173-197`), but `patterns()` is the starter set of a
**new** project. The migration does not call it: it only fills in
`grid_types`.

### 2.3. Why the check fails

`Model::check` (`crates/core/src/model.rs:8772-8774`):

```rust
for (n, t) in &self.grid_types {
    t.check(n, &self.line_patterns)?;
}
```

`GridType::check` (`crates/core/src/grids.rs:92-106`) requires the type's
`line_pattern` to be a key in `line_patterns`. An old file has no such key —
hence the error. The message names `Grid - ASME Y14.2` because `grid_types` is a
`BTreeMap` and that is the first type alphabetically among the six. In fact all
five grid patterns are missing.

### 2.4. Why the tests did not catch it

`crates/core/tests/grid_types.rs:255-272`
(`a_file_of_format_twenty_opens_with_every_grid_in_the_default_type`):

```rust
let mut s = Session::new();
let g = grid(&mut s);
let mut v: Value = serde_json::from_str(&s.model.to_json().unwrap()).unwrap();
v["format"] = json!(20);
assert!(v.as_object_mut().unwrap().remove("grid_types").is_some(), …);
…
let (m, left_out) = Model::read(&v.to_string()).unwrap();
```

The test takes a **modern** model that already has 11 patterns (6 old +
5 grid ones), changes only `format` to 20 and removes `grid_types`. A real
format-20 file does not have the five grid patterns, so the test passes while the
real file does not.

### 2.5. Why the file cannot simply be "fixed" by hand

A specific `.bimc` can be fixed (add five patterns to `line_patterns`), but that
does not remove the cause: any project saved before grid types existed will fail
the same way. The fix must live in the migration, so that old projects open for
everyone.

## 3. Goals

1. A format-20 file with grids and an arbitrary (including incomplete) set of
   `line_patterns` opens without errors.
2. The migration adds to `line_patterns` those `Grid - …` patterns that the
   starter grid types it creates reference.
3. A pattern of the same name that already exists in the project is kept as it is
   (not overwritten).
4. Format 21 and the model check stay strict: if a grid type in a format-21 file
   references a missing pattern, `Model::check()` still returns an error.
5. The migration does not depend on the project language: the names of the added
   patterns match the names `grids::types` gives to the types.
6. Saving and reopening a migrated project does not change the result.

## 4. Non-goals

- Changing the error text `there's no line pattern …` and the behaviour of
  `GridType::check` for format 21.
- Automatically "healing" format-21 files with missing patterns.
- Adding patterns that the starter grid types do not reference.
- Changing the starter grid types, their names, colours, pen widths and text
  sizes.
- Changing `lines::patterns()` for new projects (the patterns are already there).
- Editing the user's `.bimc` files in the repository or in the application.
- Changing the IFC export, the commands, the UI, the ABI.

## 5. Functional requirements

### F1. Adding patterns during migration

In `lines.rs`, extract the grid patterns into a separate function so they can be
reused in the migration:

```rust
/// The line patterns the starter grid types are drawn in, named in `language`.
pub fn grid_patterns(language: Language) -> BTreeMap<String, LinePattern> {
    use Segment::{Dash as D, Dot, Space as S};
    [
        (msg!("Grid - ISO Dash Dot"), vec![D(12.7), S(3.2), Dot, S(3.2)]),
        (msg!("Grid - DIN Dash Dot"), vec![D(20.0), S(4.0), Dot, S(4.0)]),
        (msg!("Grid - ASME Chain"), vec![D(20.0), S(3.0), D(5.0), S(3.0)]),
        (msg!("Grid - Architectural Chain"), vec![D(15.0), S(2.0), D(2.0), S(2.0), D(2.0), S(2.0)]),
        (msg!("Grid - Compact Dash Dot"), vec![D(6.0), S(1.5), Dot, S(1.5)]),
    ]
    .into_iter()
    .map(|(n, s)| (language.word(n).to_string(), LinePattern(s)))
    .collect()
}
```

`patterns(language)` keeps the same meaning: the six former patterns plus
`grid_patterns(language)`.

### F2. Editing `typed_grids()`

```rust
fn typed_grids(&mut self) {
    if self.grid_types.is_empty() {
        self.grid_types = crate::grids::types(self.language);
        // A file from before grid types holds none of the grid line patterns either:
        // those its starter types draw in are added. One of the same name stays as it is.
        for (name, pattern) in crate::lines::grid_patterns(self.language) {
            self.line_patterns.entry(name).or_insert(pattern);
        }
    }
    …
}
```

Requirements:

- patterns are added only when `grid_types` is empty (that is, the file predates
  format 21); for a format-21 file nothing changes;
- `or_insert` is used, not `insert`: a project pattern with the same name is
  kept;
- all five patterns are added even if none of the grids uses some of the types:
  the starter set of types is created in full;
- the names come from `language.word(...)`, just like the type names in
  `grids::types`;
- `lines::patterns()` and `lines::grid_patterns()` do not diverge: `patterns` is
  built on top of `grid_patterns`, so the definitions are not duplicated.

### F3. Order relative to the language migration

The block `crates/core/src/model.rs:5338-5345` replaces English patterns with the
project-language patterns if the file was saved before names existed in the
language. It runs before `typed_grids()`. After the fix:

- for an English project, English names are added;
- for a non-English project, the names of the added patterns match the names of
  the starter grid types (both come from `language.word`);
- if the language migration has already added the grid patterns, `or_insert`
  changes nothing.

### F4. Keeping the check strict

Do not touch `GridType::check`. A format-21 file in which a grid type references a
deleted pattern must still fail to open. The migration does not mask the error
for new files.

## 6. Key decision and alternatives

### 6.1. Recommended solution: add the missing patterns in the migration

The migration already knows that it creates the starter grid types. It is logical
for it to also create the patterns without which those types do not pass the
check. This is a local change in `typed_grids()` plus extracting the five
definitions into `lines::grid_patterns()`, without changing the model, the
commands or the format.

### 6.2. Rejected options

| Option | Why not |
|---|---|
| Relax `GridType::check`: allow a missing pattern | Hides real format-21 errors and makes the behaviour undefined (what should the grid be drawn with?) |
| Always add all `lines::patterns` patterns when reading | Overwrites patterns the user deliberately deleted and changes the contents of a format-21 project |
| Add the patterns only in `grids::types` | `grids.rs` does not own `line_patterns` and must not modify the model |
| Fix each `.bimc` by hand | Does not help other old projects; the cause remains |
| On a check error, silently delete the problematic grid type | Types are lost and grid behaviour changes; the error must be visible |
| Substitute `line_pattern: None` when the pattern is missing | Changes the appearance of the grid and hides a data error |

## 7. Edit points

| File | What to change |
|---|---|
| `crates/core/src/lines.rs:173-197` | extract `grid_patterns(language)`; build `patterns()` on top of it |
| `crates/core/src/model.rs:5546-5560` | in `typed_grids()`, add the missing grid patterns via `or_insert` |
| `crates/core/tests/grid_types.rs:255-272` | strengthen the format-20 test: remove the grid patterns and their `library` records |
| `crates/core/tests/grid_types.rs` | add tests T1–T5 from section 8 |

`Model::check`, `GridType::check`, `grids::types`, `commands.rs` and the UI do
not need to change.

## 8. Test plan (to be added; run by CI/the user)

### T1. A real format-20 file with six patterns opens

Build the file the way a build before format 21 writes it:

1. `Session::new()`, create a grid.
2. `to_json()`, then:
   - `format = 20`;
   - remove `grid_types`;
   - remove `type` from every grid;
   - keep only `Centre`, `Dash`, `Dash Dot`, `Dot`, `Hidden`, `Long Dash` in
     `line_patterns`;
   - remove from `library` the `kind == "line-pattern"` records whose names are
     `Grid - …` (so the file is like a real old one, not just one with a trimmed
     map).
3. `Model::read` → `Ok`, `left_out` empty.
4. `m.check() == Ok(())`.
5. `m.line_patterns` contains all five grid patterns with the correct segments.
6. Every grid received `Grid - ISO 128`.
7. `m.grid_types` contains the six starter types.

### T2. A project pattern of the same name is not overwritten

1. Take the file from T1.
2. Put a `Grid - ASME Chain` pattern with a different definition into
   `line_patterns` (for example, `[{"dash": 1.0}, {"space": 1.0}]`).
3. Read the file.
4. Check that the pattern remained the user's one (compare the segments) and that
   `m.check()` passes.

### T3. Format 21 stays strict

1. Take a modern model with grid types.
2. Remove the `Grid - ASME Chain` pattern from `line_patterns`.
3. Read the file: reading succeeds (the migration must not interfere), but
   `m.check()` returns an error with ``there's no line pattern `Grid - ASME Chain` ``.
   This pins down that the check was not relaxed.

### T4. Saving and reopening

1. Read the file from T1.
2. `to_json()` → `Model::read` again.
3. Both times `m.check() == Ok(())`; the set of patterns and grid types does not
   change.

### T5. Project language

1. Repeat T1 for a non-English `language` (for example, `de` or `ru`) that has
   translations of the type/pattern names.
2. Check that the names of the added patterns match the names in `m.grid_types`
   and that `check()` passes.

## 9. Acceptance criteria

- [ ] An old `.bimc` (format 20) with six patterns and grids opens.
- [ ] The error `grid type … there's no line pattern …` does not appear for such
      files.
- [ ] Exactly the five missing grid patterns are added to `line_patterns`.
- [ ] Existing project patterns are neither changed nor deleted.
- [ ] `Model::check()` for format 21 still rejects a missing pattern.
- [ ] Re-reading the saved file is stable.
- [ ] A non-English project receives patterns with names in its own language.
- [ ] `cargo fmt --check`, `cargo check -p bimcraft-core`,
      `cargo clippy -p bimcraft-core` pass.
- [ ] Tests were not run locally unless the user asked; this is reflected in the
      PR.

## 10. Risks and open questions

1. **The user deleted a pattern before migrating.** In format 20 the `Grid - …`
   patterns could not be used by grid types, because the types did not exist yet.
   Adding all five is therefore safe; a conflict is possible only by name, and it
   is resolved via `or_insert`.
2. **Pattern names in the project language.** If a translation appears later, an
   old file will get the name in the language at the moment of migration. This is
   consistent with how `grids::types` names the types at that same moment.
3. **`library` and GUIDs.** The new patterns get `library` records during
   `sync_library()` after reading — just like any pattern added by a command. No
   separate library migration is required.
4. **Files where the pattern exists but with a different definition.** We keep the
   user's definition; the grid will be drawn with it. This is a deliberate choice
   in favour of the project's data.
5. **Testing with a real user project.** We do not add the user's `.bimc` to the
   repository; T1 builds an equivalent file in code.

## 11. Order of work and DoD

1. Extract `lines::grid_patterns(language)`; build `lines::patterns` on top of it.
2. In `typed_grids()`, add the missing patterns via `or_insert`.
3. Strengthen the existing format-20 test (remove the patterns and the `library`
   records).
4. Add tests T1–T5.
5. Run `cargo fmt --check`, `cargo check -p bimcraft-core`,
   `cargo clippy -p bimcraft-core`.
6. Do not run tests without a direct request; state in the PR that CI will run
   them.
7. Split into commits:
   - `core: the grid line patterns of a file from before grid types`;
   - `core: tests for opening a file of format 20 with grids`.
8. Definition of Done: the acceptance criteria are met, the file format and the
   behaviour of format 21 are unchanged, there are no new dependencies and no
   `unsafe`.

## 12. What not to do

- Do not relax `GridType::check` or turn the error into a warning.
- Do not add patterns to format-21 files "just in case".
- Do not overwrite project patterns of the same name.
- Do not change the names or parameters of the starter grid types.
- Do not touch the UI, the commands, IFC or the file format.
- Do not edit the user's `.bimc` in the repository: fixing a specific file is a
  separate one-off operation, not part of the migration.
