# SPEC-012. Revit shortcuts file import: 1 MiB limit, `ShortcutItem`, multiple elements per line

Status: **planned** (not started)
Date: 2026-11-13
Project: BIMCraft (Rust, `crates/core`, `crates/ui-egui`, egui 0.36)
Source: discussion [IPedrax/BIMCraft#403 "Add configurable keyboard shortcuts with multiple bindings"](https://github.com/IPedrax/BIMCraft/pull/403): the comment by emilianocapasso dated 2026-11-12 (a full shortcuts export from Revit ≈ 300 KiB — above the 256 KiB limit; `<ShortcutItem … Paths=…/>` lines may come several per line; there is a ready-made parser that can be ported onto `parse_shortcuts`) and the reply by dronov-dmitry (accept as a separate small PR: raise the limit to 1 MiB, accept both `<Shortcut …/>` and `<ShortcutItem …/>`, read multiple elements per line, accept and ignore `Paths`, handle XML entities, tests for a multi-item line and a synthetic ~300 KiB file)
Related documents: `SPEC-008-User-Configurable-Keyboard-Shortcuts.md`, `ТЗ-008-горячие-клавиши.md`, `ТЗ-011-скоупы-действий-и-family-ключи.md` (a neighbouring task, does not overlap in code except for `parse_shortcuts`), `BIMCraft/docs/areas/shortcuts.md`
Task type: fix/extension of an existing import feature, tests.

> **In short.** Shortcuts file import (`parse_shortcuts`, `preview_shortcuts_file` in `crates/ui-egui/src/shortcuts.rs`) today: it rejects files larger than 256 KiB (`MAX_FILE`), parses Revit XML line by line via `line.contains("<Shortcut")` with a single attribute lookup per line (several elements on one line are silently lost), and does not decode XML entities in attribute values. A full shortcuts export from Revit is ≈ 300 KiB and its `<ShortcutItem … Paths=…/>` lines therefore are not imported. The task: raise the limit to 1 MiB, parse every XML element of a line separately (accepting both `<Shortcut …/>` and `<ShortcutItem …/>`), ignore `Paths`, decode XML entities, keep all previous safety limits, and cover the changes with tests (multi-item line, synthetic ~300 KiB file).

## 0. General working rules

1. Communication language — Russian; code, comments, lines in code, console messages — English.
2. Tests (`cargo test` and any other runs of test code) are **not run** for this task (rule `~/.config/opencode/rust-rules.md`): verification is `cargo build`, `cargo check`, `cargo clippy` (without `--all-targets`) and CI in the PR. At the end of the task — `bash scripts/debug.sh --non-interactive`.
3. Commits are atomic: one commit — one logically complete change; every commit builds and does not break existing behaviour.
4. Documentation and code comments — in English; this file — in Russian.
5. Action identifiers, XML attribute names, commands and paths are not translated.

## 1. The essence of the problem

### 1.1. What exists now (by code)

File `crates/ui-egui/src/shortcuts.rs`:

- `const MAX_FILE: u64 = 256 << 10;` (line 26) — the file size limit;
- `const MAX_LINES: usize = 10_000;` (line 28) and `const MAX_BINDINGS: usize = 4096;` (line 30), `MAX_PER_ACTION = 32` (line 32) — content limits;
- `preview_shortcuts_file` (~3012): based on `fs::metadata`, a file larger than `MAX_FILE` is rejected with the message "That is too big to be a shortcuts file." (~3021) — this is where the ≈ 300 KiB Revit export lands;
- `parse_shortcuts` (~1633): after the BOM, the line limit and a JSON attempt, the whole text is parsed line by line; a line containing `"<Shortcut"` is handled by the branch (~1652):
  - attributes are looked up across the **whole line** by the `attr(name)` function = the first occurrence of `name="` up to the nearest quote: for a line with several elements only the first one is read, the rest are **silently lost** (no error and no warning);
  - `CommandName` (or `CommandId`) and `Shortcuts` are required; if something is missing — the line is silently skipped (`continue`);
  - `<ShortcutItem …>` lands in the branch because it contains the substring `<Shortcut`, but the same attribute limitations apply to it; the `Paths` attribute is not taken into account (today this is not an error, but there is no deliberate acceptance either — see §1.2, item 3);
  - the `Shortcuts` value is split by `#` into several combinations (this works);
- attribute values are taken **raw**: XML entities (`&amp;`, `&quot;`, `&#xA;`, etc.) are not decoded — they end up in the action name/combination as is and produce an "Unknown action" error / a combination parsing error;
- order of formats: JSON (serde_json over the whole text) → XML lines → `version =`/`profile =` → `action = binding` / `action: binding` / `KEY - Name` / `KEY = Action`; errors — per-line `ParseIssue { line, message }`.

### 1.2. Scenarios and expected behaviour

1. **The Revit export is imported in full.** A file of ≈ 300 KiB (up to 1 MiB) passes preview and import instead of being rejected by size.
2. **Both element formats are read.** Both `<Shortcut CommandName=… Shortcuts=…/>` and `<ShortcutItem … Paths=…/>` are accepted; unknown attributes (including `Paths`) do not break parsing.
3. **Several elements per line.** A line like `<ShortcutItem …/><ShortcutItem …/><ShortcutItem …/>` yields all elements, not just the first one; each element gets its own handling and its own errors.
4. **XML entities are decoded.** `&amp;`, `&lt;`, `&gt;`, `&quot;`, `&apos;` and numeric `&#…;` in attribute values are converted into characters before the action/combination is parsed.
5. **Safety is preserved:** the line/binding/bindings-per-action limits, per-line errors, reserved keys and the assignability check work as before; only `MAX_FILE` grows.
6. **Nothing breaks:** existing formats (v2 text, JSON v2/v1, `KEY - Name`, the old XML `<Shortcut>`) are imported exactly as they are now.

### 1.3. Why this matters

- Import from Revit is a standard user migration scenario: today it hits an artificial 256 KiB limit, although the file itself is small in content (hundreds of lines of XML).
- Silently losing line elements is worse than an explicit error: the user gets a partially filled profile and does not know that part of the shortcuts were not imported.
- A ready-made parser exists (offered by the comment author for porting), and discussion #403 recorded the exact list of techniques — the task is small and isolated.

## 2. Terms

| Term | Meaning |
|---|---|
| `MAX_FILE` | Import file size limit, currently 256 KiB (`shortcuts.rs:26`) |
| `MAX_LINES` / `MAX_BINDINGS` / `MAX_PER_ACTION` | Limits for lines (10 000), bindings (4096) and bindings per action (32) |
| Element | An XML tag `<Shortcut …>` or `<ShortcutItem …>` in a line of the Revit file |
| Multi-item line | A line containing several elements in a row |
| `Paths` | An attribute of Revit export elements that must be accepted and ignored |
| XML entities | `&amp;` `&lt;` `&gt;` `&quot;` `&apos;` `&#…;` `&#x…;` in attribute values |
| `ParseIssue` | An import error/warning with a line number (`shortcuts.rs:1618`) |
| `ParsedProfile` | Parsing result: version, profile, bindings, warnings, errors (`shortcuts.rs:1624`) |
| Preview | Import preview before applying (`preview_shortcuts_file`, `shortcuts.rs:3012`) |

## 3. Functional requirements

### 3.1. F1. File limit 256 KiB → 1 MiB

- `MAX_FILE: u64 = 1 << 20` (1 MiB) in `shortcuts.rs:26`; the size error message does not change.
- The limit applies to the file size in bytes (the `fs::metadata(...).len()` check in `preview_shortcuts_file`, `shortcuts.rs:3019-3021`); the error line "That is too big to be a shortcuts file." remains for files > 1 MiB.
- `MAX_LINES`, `MAX_BINDINGS`, `MAX_PER_ACTION` **do not change** (see §4, item 1).

### 3.2. F2. Accept `<Shortcut …/>` and `<ShortcutItem …/>`

- The XML parsing branch triggers on both element names; for each one the attributes `CommandName` or `CommandId` (as now) and `Shortcuts` are read.
- Unknown attributes (including `Paths`) are not considered an error and do not affect parsing.
- Lines where required attributes are missing (`CommandName`/`CommandId` or `Shortcuts`) produce a per-line warning/error `ParseIssue` instead of a silent skip (see §10, open question 1 — warning or error).

### 3.3. F3. Several elements per line

- Every line is split into all of its elements (by tag start), and each is parsed separately: its own `CommandName`/`CommandId`, its own `Shortcuts`, its own errors with the same line number.
- Combinations of one element are still split by `#`; bindings of all elements of the line land in `ParsedProfile.bindings` in the order they appear.
- A duplicate combination between elements is handled by the existing `add_parsed` logic ("already assigned to …", `shortcuts.rs:1726`).

### 3.4. F4. Accept and ignore `Paths`

- The `Paths` attribute (and any other unknown ones) is parsed as a regular attribute; its value is not used and does not cause an error/warning.

### 3.5. F5. XML entities

- Before an attribute value is used, XML entities are decoded: named ones (`&amp;` `&lt;` `&gt;` `&quot;` `&apos;`) and numeric ones (`&#10;`, `&#x20;`, etc.) into Unicode characters.
- Decoding applies only to XML attribute values; text formats (`action = binding`, `KEY - Name`, JSON) do not change.
- An unfinished/unknown entity does not bring down the import of the whole file (the behaviour decision is in §10, open question 2).

### 3.6. F6. Tests (a mandatory part of the task)

- **Multi-item line:** a line with three or more elements `<ShortcutItem … Paths=…/>` (and one `<Shortcut …/>`) is parsed into the expected number of bindings, the order is preserved, there are no errors.
- **Synthetic export ≈ 300 KiB:** a generated file (hundreds of XML lines, ~300 KiB in total, larger than the old 256 KiB limit) passes `preview_shortcuts_file`/`parse_shortcuts` without a size error and yields the expected number of bindings (within `MAX_BINDINGS`).
- **`Paths`:** an element with `Paths` is imported without errors and warnings.
- **XML entities:** an attribute with `&amp;`/`&#…;` is parsed into the expected value.
- **Regression:** the existing format tests (JSON v2, `KEY - Name`, the old `<Shortcut …/>`, `version = 3` → error, duplicates, reserved keys — `shortcuts.rs` ~3206+) keep passing.

### 3.7. F7. Change map

| What | File | Where/what now |
|---|---|---|
| `MAX_FILE` 256 KiB → 1 MiB | `crates/ui-egui/src/shortcuts.rs` | line 26, `preview_shortcuts_file` ~3019 |
| Parse every element of a line separately, both tag names | `crates/ui-egui/src/shortcuts.rs` | the `line.contains("<Shortcut")` branch ~1652 |
| The `Paths` attribute and other unknown ones are ignored | `crates/ui-egui/src/shortcuts.rs` | `attr()` ~1653 |
| Decoding XML entities in attribute values | `crates/ui-egui/src/shortcuts.rs` | a new helper function next to `parse_shortcuts` |
| An explicit warning instead of silently skipping a line without required attributes | `crates/ui-egui/src/shortcuts.rs` | ~1657-1670 |
| Tests: multi-item, ~300 KiB, `Paths`, entities | `crates/ui-egui/src/shortcuts.rs` (`#[cfg(test)]`) | next to the tests ~3206 |
| Import documentation (formats, limit) | `BIMCraft/docs/areas/shortcuts.md`, `SPEC-008` | the import section |

## 4. Pitfalls

1. **`MAX_LINES = 10_000` stays.** The synthetic 300 KiB file must be built so that it fits within the line limit, or the limit must be deliberately revised: if a real Revit export exceeds it, the F6 test will show that. The decision must be recorded before work starts (§10, open question 3).
2. **`line.contains("<Shortcut")` also captures the root tag `<Shortcuts>`** (it contains `<Shortcut`): currently it is silently skipped because `Shortcuts="` is absent; when switching to tag-based splitting, the root element must be skipped explicitly, without an error.
3. **Order of formats:** JSON is tried first over the whole text; XML lines are processed before the `action = binding` branch. The new logic must not intercept ordinary lines (for example, `cmd.family = Ctrl+K` must not go into the XML branch).
4. **`attr()` currently looks up the first occurrence on a line** — with several elements this is the source of data loss; replacing it with splitting the line into element segments is mandatory, otherwise F3 will not be fulfilled.
5. **Line numbers in `ParseIssue`:** with several elements on a line, all their errors get the same line number — this is normal and matches current behaviour; in the message it is useful to distinguish the element (by its attribute), but not mandatory (§10, question 4).
6. **Entity decoding must not break the `#` separator** in `Shortcuts`: decoding happens after the attribute value is cut out and before splitting by `#`.
7. **Safety:** do not raise `MAX_BINDINGS`/`MAX_PER_ACTION`/`MAX_LINES`; a file up to 1 MiB potentially has more lines and bindings, but the limits — not memory — must be the binding constraint: parsing is per line, JSON is a single `serde_json::from_str` over the whole text (acceptable for files up to 1 MiB, but verify on the ~300 KiB test).
8. **`preview_shortcuts_file` rejects `.md`** (help) and symlink files/directories (`m.is_file()`); this behaviour is preserved.
9. **Separation from ТЗ-011:** both tasks touch `shortcuts.rs`, but different areas (ТЗ-011 — registry/scopes/executor, ТЗ-012 — `parse_shortcuts`/`preview_shortcuts_file`); the merge order should be agreed so as not to conflict in the same file.

## 5. Tests

> Tests are not run locally (project rule); they are written and verified by CI in the PR.

### 5.1. Unit tests (`crates/ui-egui/src/shortcuts.rs`, `#[cfg(test)]`)

1. `multi_item_line`: the line `<ShortcutItem CommandName="Wall" Shortcuts="WA" Paths="…"/><ShortcutItem CommandName="Door" Shortcuts="DR" Paths="…"/>` (and a similar one made of `<Shortcut …/>`) → 2+ bindings, no errors.
2. `three_items_in_one_line`: three elements on one line → three bindings, order as in the line.
3. `revit_export_size`: a synthetic file ~300 KiB (generated in the test) → `parse_shortcuts` without size errors, bindings within `MAX_BINDINGS`; separately — a file > 1 MiB is rejected with the previous message.
4. `paths_attribute_ignored`: an element with `Paths` is imported without warnings/errors.
5. `xml_entities`: `CommandName="A &amp; B"` / `Shortcuts="WA&#x23;"` etc. → values decoded before `resolve_action`/`parse_binding`.
6. `missing_attributes_warn`: the line `<ShortcutItem Paths="…"/>` without `CommandName`/`Shortcuts` → `ParseIssue` with a line number (not a silent skip).
7. Regression: the existing tests ~3206+ (JSON v2, legacy JSON, `KEY - Name`, `version = 3`, duplicates, reserved, `# BIMCraft …` without bindings) — with unchanged expectations.

### 5.2. UI tests (`crates/ui-egui/src/tests/…`)

- Import through the Keyboard Shortcuts dialog (File → Import): choosing a ~300 KiB file → the preview shows the number of bindings, applying gives no errors; a file > 1 MiB → the previous size message.
- (If entering a file manually in a test is inconvenient — unit coverage of `parse_shortcuts` plus one UI test on the preview path is enough.)

### 5.3. Quality control

- `cargo check`, `cargo build`, `cargo clippy -p bimcraft-ui-egui` (without `--all-targets`) — clean.
- `bash scripts/debug.sh --non-interactive` — the debug binaries build.
- CI in the PR — green.

## 6. Documentation and translations

- `BIMCraft/docs/areas/shortcuts.md`: the import section — supported formats (v2 text, JSON, `KEY - Name`, Revit XML `<Shortcut>`/`<ShortcutItem>` with several elements per line), the 1 MiB limit, ignoring `Paths`.
- `SPEC-008-User-Configurable-Keyboard-Shortcuts.md`: the limit and the import format.
- New/changed message strings — in the translation templates (`.po`), if the error texts change.
- Code comments (`parse_shortcuts`, the entity helper) — in English.

## 7. Acceptance criteria

1. A 300 KiB file is imported; a file > 1 MiB is rejected with the previous message; `MAX_LINES`/`MAX_BINDINGS`/`MAX_PER_ACTION` are unchanged.
2. `<Shortcut …/>` and `<ShortcutItem …/>` are read; `Paths` and other unknown attributes are ignored without errors.
3. Several elements on a line are all processed; none is silently lost.
4. XML entities in attribute values are decoded.
5. Lines without required attributes produce a `ParseIssue` with a line number.
6. The tests from §5.1 are written (including multi-item and ~300 KiB); the existing format tests are unchanged in expectations.
7. The documentation from §6 is updated.
8. `cargo check`/`build`/`clippy` are clean, `scripts/debug.sh --non-interactive` passes, CI is green.

## 8. How to verify manually

1. A shortcuts export from Revit (KeyboardShortcuts.xml, ~300 KiB) → File → Import in BIMCraft: the preview opens (no "too big"), the binding count is greater than zero, there are no substantive errors.
2. Apply the import → the Keyboard Shortcuts dialog shows the assigned keys; combinations from several elements on a line are all present.
3. A file larger than 1 MiB → the previous message "That is too big to be a shortcuts file.".
4. A file with `Paths` and entities (`&amp;` in a name) → import without errors / with a warning per the decision in §10.1–10.2.
5. Old files (BIMCraft v2 text, JSON, `KEY - Name`) → import as before.

## 9. Out of scope

- Action scopes, `ui.family.*`, physical family input — **ТЗ-011** (a separate PR; here only avoid conflicts in `parse_shortcuts`).
- Export to the Revit format (reverse conversion) — import only.
- Changing the BIMCraft v2 text/JSON format.
- Raising `MAX_LINES`/`MAX_BINDINGS`/`MAX_PER_ACTION` — only if §10.3 decides otherwise.
- Recognising other shortcuts formats (Autodesk Inventor, Archicad, etc.).
- The import UI (the preview dialog) — except where a preview-path test is needed.

## 10. Decisions made and open questions

**Decided (discussion #403):**

1. A separate small PR (not in #403 and not in the family follow-up): the 1 MiB limit; `<Shortcut …/>` and `<ShortcutItem …/>`; several elements per line; `Paths` is accepted and ignored; XML entities; tests for a multi-item line and a synthetic ~300 KiB export.
2. emilianocapasso's ready-made parser is ported onto `parse_shortcuts` (the offer was accepted: "I'd be happy to port it onto parse_shortcuts").
3. The `#` separator of combinations inside `Shortcuts`, per-line `ParseIssue`, reserved keys and the assignability check — as now.

**Open questions:**

1. **A line without required attributes:** should it be an error or a warning `ParseIssue` (now — a silent skip)? The default option in F2 is a warning, so that foreign XML lines do not bring down the import.
2. **An unfinished XML entity:** skip it as is, treat it as a warning, or as a line error — to be clarified during implementation (the default option: a warning, the value stays raw).
3. **`MAX_LINES = 10_000`:** should it be raised together with `MAX_FILE` (a real Revit export may be longer than 10 000 lines at 300 KiB)? The decision must be made before work starts: either raise it proportionally (e.g. 40 000) or keep it and build the test file within the limit.
4. **Identifying the element in an error message** when there are several elements on a line: whether to add the element name/`CommandId` to the `ParseIssue` text (the line number is shared) — optional, does not block the task.

## 11. Work plan (atomic commits)

1. **Size limit.** `MAX_FILE` → 1 MiB + a test for a file over the limit and for ~300 KiB (without changing parsing).
2. **Parsing an XML line by elements.** Splitting into elements, both tag names, ignoring `Paths`/unknown attributes, an explicit `ParseIssue` instead of a silent skip + tests for multi-item/`Paths`/missing-attributes.
3. **XML entities.** Helper decoding in attribute values + tests.
4. **Documentation and translations** (§6) + a UI preview test if needed (§5.2).

Every commit builds (`cargo check`) and does not break existing import formats; tests are run by CI.

## 12. Scenario traceability

| # | Scenario | Requirement | Test |
|---|---|---|---|
| 1 | A Revit export ~300 KiB is imported | F1, F6 | 5.1 (revit_export_size), 5.2 |
| 2 | A file > 1 MiB is rejected with the previous message | F1 | 5.1 |
| 3 | `<ShortcutItem … Paths=…/>` is read, `Paths` is ignored | F2, F4 | 5.1 (paths_attribute_ignored) |
| 4 | Three elements on a line → three bindings | F3 | 5.1 (three_items_in_one_line) |
| 5 | `&amp;`/`&#…;` in an attribute are decoded | F5 | 5.1 (xml_entities) |
| 6 | A line without `CommandName`/`Shortcuts` → `ParseIssue` | F2 | 5.1 (missing_attributes_warn) |
| 7 | Old formats are imported as before | F2 (regression) | 5.1 (existing tests) |
