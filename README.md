# BIMCraft Prompts

**English** | [Русский](README_RU.md)

This folder stores **prompts (specifications) for BIMCraft** — ready-to-use, self-contained task files that are fed to a harness with a neural network (OpenCode, Claude Code, or any other AI coding harness). The harness reads the specification and produces **new functionality in the [BIMCraft](https://github.com/IPedrax/BIMCraft) project**: the model implements the feature, and you get working code instead of a conversation.

Each file is not a short request like "fix the joints", but a full working specification: it describes the current behaviour, points to the exact places in the code that cause the problem, formulates the required behaviour as testable requirements, outlines a technical plan, lists pitfalls, and defines the tests that must prove the result. This is what makes a single file enough input for the harness — the model does not have to guess the context.

## Why this repository exists

- **Spec in, feature out.** One file goes into the harness; implemented functionality comes out of it. The task does not need to be re-explained to the model in chat — everything the model must know is already in the file.
- **Repeatable runs.** The same specification can be fed to the harness again — in a fresh session, with a newer model, or after a failed attempt — and the input stays identical.
- **History and review.** Specifications stay in git next to the code they produced, so it is always clear what was requested, why, and what the scope boundaries were.

## File naming

```
SPEC-<NNN>-<short-kebab-case-title>.md
```

Examples:

- `SPEC-007-strip-foundation-handles-joints-bonding.md`

`NNN` is a running number that keeps specifications ordered and lets them reference each other (`Related specs: SPEC-006-…`).

## Specification structure

Every specification follows the same pattern:

| Section                        | What it answers                                                                                                                |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| Header                         | Status (`specification, not implemented`), date, project/build reference, related specs, task type, optional defect screenshot |
| 1. Context and task            | What the feature is, what is wrong today, scope decisions confirmed by the user                                                |
| 2. Why it happens this way now | Root causes in the current code, file by file — so the agent starts from facts, not guesses                                    |
| 3. Terms                       | Glossary of project-specific vocabulary (run, mitre, grip, glue, …)                                                            |
| 4. Functional requirements     | Numbered, testable requirements the implementation must satisfy                                                                |
| 5. Technical plan              | Where the change goes (modules, commands, data format), without writing the code                                               |
| 6. Pitfalls                    | Known traps that must not be broken (regressions, tolerances, performance, file format, localisation)                          |
| 7. Tests                       | Concrete test scenarios that prove the requirements are met                                                                    |
| Further sections               | Limitations/out-of-scope, and anything task-specific                                                                           |

## How to use a specification

1. Pick the file for the feature you want to get.
2. Start a harness session in the BIMCraft working copy (the header of the specification names the working copy, branch, and project format version the spec was written against).
3. Feed the specification file to the harness as the task input — for example, give it the file path and ask to implement what the file specifies. The model reads the file, follows its requirements, technical plan and pitfalls, and writes the code into the project.
4. When the harness is done, verify the result against the test list from section 7 of the specification.
5. If the spec mentions related specifications, feed them to the harness too — they describe neighbouring behaviour that must not regress.
6. Update `Status:` in the header of the specification once the code is merged.

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
