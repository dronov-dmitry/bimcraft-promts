# BIMCraft Prompts

This folder stores **prompts (specifications) for BIMCraft** — ready-to-use, self-contained task descriptions that are handed to an AI coding agent (for example, OpenCode) to implement a feature or fix a defect in the [BIMCraft](https://github.com/) project.

Each file is not a short request like "fix the joints", but a full working specification: it describes the current behaviour, points to the exact places in the code that cause the problem, formulates the required behaviour as testable requirements, outlines a technical plan, lists pitfalls, and defines the tests that must prove the result.

## Why this repository exists

- **Repeatable context.** Explaining a BIMCraft task (its scope, the user's confirmed decisions, the relevant code paths) takes a long time. A specification captures that context once and can be replayed in any session.
- **One prompt — one task.** A file can be given to an agent as-is; it does not need to be re-explained.
- **History and review.** Specifications stay in git next to the work they produced, so it is always clear what was asked, why, and what the scope boundaries were.

## File naming

```
SPEC-<NNN>-<short-kebab-case-title>.md
```

Examples:

- `SPEC-007-strip-foundation-handles-joints-bonding.md`

`NNN` is a running number that keeps specifications ordered and lets them reference each other (`Related specs: SPEC-006-…`).

## Specification structure

Every specification follows the same pattern:

| Section | What it answers |
| --- | --- |
| Header | Status (`specification, not implemented`), date, project/build reference, related specs, task type, optional defect screenshot |
| 1. Context and task | What the feature is, what is wrong today, scope decisions confirmed by the user |
| 2. Why it happens this way now | Root causes in the current code, file by file — so the agent starts from facts, not guesses |
| 3. Terms | Glossary of project-specific vocabulary (run, mitre, grip, glue, …) |
| 4. Functional requirements | Numbered, testable requirements the implementation must satisfy |
| 5. Technical plan | Where the change goes (modules, commands, data format), without writing the code |
| 6. Pitfalls | Known traps that must not be broken (regressions, tolerances, performance, file format, localisation) |
| 7. Tests | Concrete test scenarios that prove the requirements are met |
| Further sections | Limitations/out-of-scope, and anything task-specific |

## How to use a specification

1. Pick the file for the task you want to perform.
2. Open a session with the BIMCraft working copy (the header of the specification names the working copy, branch, and project format version the spec was written against).
3. Give the agent the specification file path and ask it to implement the task, following the requirements and tests from the file.
4. When the spec mentions related specifications, read them too — they describe neighbouring behaviour that must not regress.

## Adding a new specification

1. Take the next free number and create `SPEC-<NNN>-<title>.md`.
2. Fill in the header: status, date, project reference, related specs, task type.
3. Keep the structure above. The section *Why it happens this way now* is mandatory — a specification without located root causes is only a wish, not a prompt.
4. State scope decisions explicitly (what is included, what is out of scope).
5. End with tests that a reviewer can run to confirm the task is done.
6. Update `Status:` in the header when the specification is implemented.

## Conventions

- Specifications are written in **English**.
- Status is one of: `specification, not implemented`, `in progress`, `implemented`.
- Requirements are written as numbered, checkable statements; each test refers back to them.
- Everything project-specific (file paths, commands, formats) must be concrete — vague references defeat the purpose of the file.
