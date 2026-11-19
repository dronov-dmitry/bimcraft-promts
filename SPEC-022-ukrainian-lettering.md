# SPEC-022. Ukrainian letters in the drawing font: plans without "?"

Status: **draft**, implementation required
Date: 2026-11-19
Project: BIMCraft (Rust; `crates/core/src/font.rs`, `crates/core/src/render.rs`,
`crates/core/tests/drawing_lettering.rs`, `crates/core/src/language.rs`)
Source: user request of 2026-11-19: "make it so Ukrainian letters are readable on
the plan too. right now they are not readable, and a question mark is put in place of them."
Related: SPEC-021 (agent and project) — no overlap.
Task type: fixing the stroke font and diagnostics; the model, project file, format and ABI do
not change.

> **In short.** Drawing lettering (plans, sections, elevations, sheets) is drawn with our own
> single-line font `font.rs`. It has Latin and **Russian** Cyrillic (33 letters), but not the
> Ukrainian `І`, `Ї`, `Є`, `Ґ`. That is why "Кухня-вітальня" comes out as `КУХНЯ-В?ТАЛЬНЯ`, and
> "Тех. приміщення" as `ПРИМ?ЩЕННЯ`, and a note about `U+0456` appears in the render response.
> We need to add four letters to the font (a few strokes on the 8 × 12 grid) without changing the
> style, and keep "?" only for what the font genuinely does not have.

## 0. General working rules

1. Communication language is Russian; code, comments, strings in code, compiler and log
   messages are English. User-facing window strings go through `msg!`/`t()`, catalogs are
   updated (`cargo i18n update`).
2. Tests (`cargo test`, `cargo nextest`, doctest) are **not run** in this task without a direct
   user request (rule `~/.config/opencode/rust-rules.md`). Allowed: `cargo check`, `cargo build`,
   `cargo clippy` (without `--all-targets`), `cargo fmt --check`, `cargo i18n check`; tests run on
   CI and by the user.
3. Commits are atomic: glyphs and `has()` separately, diagnostics in `render.rs` separately, tests
   separately, documentation separately.
4. No new dependencies; `unsafe` is not added.
5. The project format, commands and API do not change. Only the way the stroke font draws
   characters changes.

## 1. The essence of the problem: what exists today

### 1.1. Symptom

In the 2026-11-19 task, plan rendering returned:

```
"notes": ["Drawing lettering has no glyph for U+0456; drawn as a question mark (?)"]
```

and on the plan:

| In the model | On the plan |
|---|---|
| `Кухня-вітальня` | `КУХНЯ-В?ТАЛЬНЯ` |
| `Тех. приміщення` | `ТЕХ. ПРИМ?ЩЕННЯ` |
| `Спальня 1`, `Спальня 2`, `С/В` | unchanged (no such letters) |

### 1.2. Cause

`crates/core/src/font.rs` is "our own single-stroke lettering for plan annotation: capitals, Latin and
Cyrillic" (module header, lines 1–8). But the Cyrillic there is **Russian**: `known()` lists
`А Б В Г Д Е Ё Ж З И Й К Л М Н О П Р С Т У Ф Х Ц Ч Ш Щ Ъ Ы Ь Э Ю Я` (33 letters) and lacks the Ukrainian
`І Ї Є Ґ`.

After that, a single line decides everything:

| Place | Code | Behaviour |
|---|---|---|
| Substitution | `font.rs:36-40` — `fn glyph(c) { known(c).unwrap_or(QUESTION) }` | unknown letter → `?` |
| Check | `font.rs:246-255` — `has(c)` | `to_uppercase()`, then `accent()/tone()/known()` |
| Diagnostics | `render.rs:565` | note "no glyph for U+0456; drawn as a question mark" |
| Language test | `language.rs:227` | "every word its catalog has, the lettering draws, with no question marks" — checks only the words of its own language catalog |

The key point: lettering on the plan is drawn with **strokes** (`font::strokes`), not with a system
font; the system font has Ukrainian letters, the stroke font does not. `char::to_uppercase` on `і`
gives `І` (U+0406), which is not in `known()`, so `has('і')` is false and `strokes()` draws `QUESTION`.

### 1.3. What is missing

| Letter | Code | In the font |
|---|---|---|
| І (і) | U+0406 / U+0456 | **absent** |
| Ї (ї) | U+0407 / U+0457 | **absent** |
| Є (є) | U+0404 / U+0454 | **absent** |
| Ґ (ґ) | U+0490 / U+0491 | **absent** |
| apostrophe ʼ | U+02BC | **absent** (`’` U+2019 exists, `font.rs:159`) |

The remaining letters of the Ukrainian alphabet (`А Б В Г Д Е Ж З И Й К Л М Н О П Р С Т У Ф Х Ц Ч Ш Щ Ь Ю Я`)
are already present. Case does not matter: the font draws lowercase as uppercase.

## 2. Goals

1. Ukrainian `І`, `Ї`, `Є`, `Ґ` (in either case) are drawn on the plan, in a section, on an elevation,
   on a sheet and in exports (PNG, SVG, PDF) — without "?" and without the note.
2. The style does not change: the same 8 × 12 grid, the same stroke weight, the same cap height, the
   same spacing; the new letters are indistinguishable in manner from their neighbours.
3. "?" remains only for characters the font genuinely lacks (Chinese, Greek and the like), and the
   note remains — it is useful.
4. A test that prevents regression: the entire Ukrainian alphabet passes `font::has`.
5. Nothing breaks: Latin and Russian Cyrillic are drawn with the same pixels as before.

## 3. Non-goals

- Full Unicode support in the stroke font (Chinese, Arabic, Indic scripts).
- Changing the typeface, weight, slant or "hand-drawn" quality of the drawing font.
- Translating the interface into Ukrainian (the UI already shows Ukrainian with the system font; this
  is only about drawing lettering).
- Automatically choosing a different font for entire lettering strings (fallback to IBM Plex for
  drawings) — not in this task; see §10, question 2.
- Changing the project format: names are already stored in UTF-8 and survive save/reopen.

## 4. The key decision

**Extend the single-line font with four glyphs and one mark, preserving the style.**

Why not fall back to the system font:

| Option | Consequence |
|---|---|
| Extend `font.rs` (chosen) | 4 letters × a few strokes; a single style, vector export, nothing depends on the system |
| Draw missing letters with the IBM Plex outline | a mix of single-line and outline font on the plan; weight and height do not match; heavier PDF |
| Keep "?" and ask the user to rename | unacceptable: this is user data, Ukrainian is a working language of the project |

## 5. Stage A. Glyphs

### 5.1. What to add to `known()` (`font.rs`, next to `'Г'`, `'Е'`, `'Э'`)

| Letter | Shape | Implementation |
|---|---|---|
| `І` | like a Latin `I` with serifs | `'І' => glyph('I')` (Latin I already exists, `font.rs:38`) |
| `Ї` | `І` + two dots above it | via `accent()`: `'Ї' => return Some(('І', DIAERESIS))` (like `Ё` in `font.rs:192`) |
| `Є` | like `C`, but with a crossbar from the left edge to the middle | `&[&[(8,10),(6,12),(2,12),(0,10),(0,2),(2,0),(6,0),(8,2)], &[(0,6),(5,6)]]` (mirror of `Э`, `font.rs:137`) |
| `Ґ` | `Г` + a short vertical stroke upward from the right end of the crossbar | `&[&[(8,12),(0,12),(0,0)], &[(8,12),(8,14)]]`; if extending past the cap height is undesirable — crossbar at `y = 10`, stroke `(8,10)–(8,12)` |
| `ʼ` | like `’` | `'ʼ' => glyph('’')` (or add it to the line at `font.rs:159`) |

Shape requirements:

- The grid is the same: 8 units wide, 12 high, `ADVANCE = 11` (`font.rs:258`).
- `Ї`: dots at `y = 14–15`, like `DIAERESIS` (`font.rs:176`), so that it matches `Ё`.
- `Ґ`: the vertical must not merge with the crossbar into a "Т"; the corner is a right angle.
- `Є`: the crossbar must not touch the right edge (otherwise it reads as `Э`).
- Lowercase is not added: `strokes()` applies `to_uppercase`.

### 5.2. `has()`

`has()` (`font.rs:246-255`) already consults `accent()`, `tone()` and `known()`, so after §5.1
`has('і')`, `has('ї')`, `has('є')`, `has('ґ')` become true without changes. Verify with a test.

### 5.3. Compatibility

- Old projects: names are already in UTF-8; when opened they are drawn with the new glyphs, the file
  does not change.
- Latin and Russian strings: `strokes()` does not change; the test "pixels before/after match" on the
  control string `AZaz АЯая 0123` (§8).

## 6. Stage B. Diagnostics

`render.rs:565` currently reports only the character code. It needs to:

- name the lettering string itself (up to 40 characters, in quotes) and the character: `Drawing
  lettering has no glyph for «і» (U+0456) in «Кухня-вітальня»; drawn as a question mark (?)`;
- not duplicate the note for the same character in the same lettering string (the `notes.contains`
  check already exists);
- keep the limit of 32 notes;
- for Ukrainian letters the note does not appear at all.

## 7. Stage C. All drawing paths

Verify that the letters are visible also in:

1. the level plan (`render view: "Level 1"`);
2. a section and an elevation (`section`, `elevation`);
3. the reflected ceiling plan (`ceiling: true`);
4. a sheet (`sheet`) and its PDF;
5. SVG and PNG export (`export`/`render`);
6. room tags, door and window marks, grid names, dimensions, text notes
   (`text {font: "BIMCraft Lettering"}`), schedules on a sheet.

All of them go through `font::strokes`/`font::has`; no separate code is required, but the test in §8
should cover at least the plan, a section and a sheet.

## 8. Test plan

1. `font.rs` (unit):
   - `has()` is true for `І і Ї ї Є є Ґ ґ ʼ`;
   - `strokes("ІЇЄҐ", …)` produces non-empty lines and is **not** equal to `strokes("????", …)`;
   - `width()` for the new string is computed from the number of characters (as before).
2. `font.rs` (unit, regression): `strokes("AZaz АЯая 0123")` before and after the change is identical.
3. `render` (integration, following `tests/drawing_lettering.rs`):
   - a grid named `Ґ` and a room `Кухня-вітальня`: `render` contains no `notes`;
   - the pixels of the lettering differ from the same lettering where the letter is replaced by `?`;
   - the pixels are not empty (the letter is actually drawn).
4. `render` (regression): `漢漢ωω` is still drawn by the text pipeline's fallback without a note
   (do not modify the existing test).
5. Export: the PDF of a sheet with Ukrainian lettering opens, the text in it is curves, there is no
   "?" (verify the presence of the note and of pixels).
6. `save`/`open`: the name `Кухня-вітальня` survives saving, and rendering after opening matches
   pixel for pixel.
7. `language.rs`: if a Ukrainian language is added to the catalog (`uk.po`), the same test "every word
   draws without question marks" must pass; in this task the catalog is not added, the test is not
   touched.

## 9. i18n and documentation

- No new interface strings; the note is diagnostics, in English.
- `docs/areas/levels-walls-roofs.md` or `docs/areas/looking-at-results.md`: in the section about the
  drawing font, replace "Latin and Cyrillic" with a list of what is supported (Latin with diacritics,
  Russian and Ukrainian Cyrillic), so that the promise matches the code.
- `font.rs` header (lines 1–8): list the Ukrainian letters.

## 10. Risks and open questions

| Risk | Answer |
|---|---|
| `Ґ` extends past the cap height | the variant with the crossbar at `y = 10`; choose from a screenshot |
| `Є` is confused with `Э` or `С` | distinguishability test: the pixels of `Є`, `Э`, `С` are pairwise different |
| Other Cyrillic languages (Belarusian `Ў`, Serbian `Ђ Љ Њ Ћ Џ`, Macedonian `Ѓ Ќ Ѕ Ј`) | not in this task; the mechanism is the same, add on request |
| The font starts being used for the UI | the UI uses egui and system fonts; no overlap |
| PDF export already contains no "?" today | no, it does: the same substitution; the fix covers PDF as well |

Questions for discussion:

1. Should the full extended Cyrillic set be added right away (Belarusian `Ў`, Serbian `Ђ Љ Њ Ћ Џ`,
   Macedonian `Ѓ Ќ Ѕ Ј`) — "for the future"? Proposal: no, only Ukrainian + `ʼ`, the rest on request.
2. Is a general fallback of the stroke font to IBM Plex needed for arbitrary text (Chinese, Greek)
   instead of "?"? That is a separate task: mixed manner, weight and metrics.
3. Should Ukrainian be added to the project's language list (`new {language: "uk"}`)? It is absent
   now; that is a separate task about the translation catalog, not about the font.

## 11. Acceptance criteria

- [ ] `Кухня-вітальня` and `Тех. приміщення` are fully readable on the plan, without "?".
- [ ] `render` returns no note for Ukrainian letters.
- [ ] `І Ї Є Ґ` are drawn in upper and lower case; `Ї` with dots, `Є` with a crossbar, `Ґ` with a
      stroke.
- [ ] `ʼ` is drawn as an apostrophe.
- [ ] Latin and Russian Cyrillic are unchanged (pixel test).
- [ ] "?" and the note remain for genuinely missing characters.
- [ ] The plan, section, sheet and PDF contain Ukrainian letters.
- [ ] Tests in §8 are green; `cargo fmt --check`, `cargo clippy` without `--all-targets`.

## 12. Definition of Done

- [ ] Glyphs `І`, `Ї`, `Є`, `Ґ`, `ʼ` in `font.rs`; `has()` passes for the Ukrainian alphabet.
- [ ] The note in `render.rs` names the lettering string and the character; it does not appear for
      Ukrainian letters.
- [ ] Tests in §8 (unit + integration + regression).
- [ ] `font.rs` header and documentation updated.
- [ ] Manual check: a project with the room `Кухня-вітальня` → plan, section, sheet, PDF — without "?".
