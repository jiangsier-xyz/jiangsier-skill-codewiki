---
name: codewiki
description: |
  Trigger a one-shot workflow that clones (or pulls) a Git repository and
  generates a structured wiki from it using the `codewiki` CLI. Use this
  skill whenever the user asks to "generate a wiki", "document this repo",
  "create a code wiki", or invokes `/codewiki <input>`.
dependencies:
  - name: CodeWiki
    url: https://github.com/FSoft-AI4Code/CodeWiki
    required: true
    binary: codewiki
---

# Skill: `codewiki`

## When to trigger

Invoke this skill when the user:

- Types `/codewiki <input>`, **or**
- Asks in natural language to generate a wiki / documentation from a repository
  (e.g. *"generate a wiki for anthropics/claude-code"*).

## Trigger command

```
/codewiki <user_input>
```

## Dependencies

This skill depends on **[CodeWiki](https://github.com/FSoft-AI4Code/CodeWiki)**
(`FSoft-AI4Code/CodeWiki`). The `codewiki` CLI — provided by that project —
must be installed and available on `$PATH` before the underlying script can
run. The script's startup `check_dependencies` step will fail with exit code
`2` and a clear message if `codewiki` is missing.

If the user has not installed CodeWiki yet, direct them to
<https://github.com/FSoft-AI4Code/CodeWiki> and have them follow the install
instructions there **before** confirming execution.

## Agent execution logic

Follow these steps in order. Do **not** execute the underlying script until
the user has explicitly confirmed.

### 1. Parse `<user_input>`

Extract three fields from the user's input. Accept natural-language phrasing
and shorthand.

| Field        | Required | Accepted forms                                              | Default  |
|--------------|----------|------------------------------------------------------------|----------|
| repository   | yes      | `group/repo`, `https://github.com/...`, `git@github.com:...` | —        |
| output dir   | no       | any local path                                              | `.`      |
| instructions | no       | free-form text describing focus / constraints               | (none)   |

Detection heuristics:

- **repository**: the first token matching `^\w[\w.-]*/[\w.-]+$`, or starting
  with `https://` or `git@`.
- **output dir**: a token following words like `into`, `to`, `under`, `at`,
  or a path-looking token (`./`, `/`, or containing `/`).
- **instructions**: everything remaining after repository and output are
  removed. Treat quoted strings as a single instruction block.

### 2. Present a structured confirmation overview

Render the parsed options back to the user in a compact block before doing
anything. Example:

```
╭─ codewiki — parameters ──────────────────────────────╮
│ Repository   : anthropics/claude-code                │
│   (expanded) : https://github.com/anthropics/        │
│                claude-code.git                       │
│ Output dir   : .                                     │
│ Instructions : "Focus on the auth module; skip       │
│                vendored code."                       │
╰──────────────────────────────────────────────────────╯
```

If a field was inferred from natural language (not stated literally), mark it
with `(inferred)` so the user can correct it.

### 3. Prompt for confirmation or modification (STRICT)

Present exactly two actionable paths. Do not run the script yet.

> **Confirm to execute, or tell me what to change.**
>
> - Reply `yes` / `confirm` / `go` → I will run:
>   ```bash
>   ./scripts/codewiki.sh -r anthropics/claude-code \
>               -o . \
>               -i "Focus on the auth module; skip vendored code."
>   ```
> - Reply `modify: <field> = <value>` → I will update the parameter table
>   and re-prompt.
> - Reply `cancel` → abort without side effects.

### 4. Execute (only after confirmation)

Run the underlying script with the exact flags agreed above. Stream its
output to the conversation. On completion, report:

- The path to the generated wiki (typically `<output>/<repo>/wiki`).
- Any warnings from `codewiki` worth the user's attention.

## Hard rules

- **Never** run `codewiki.sh` before the user confirms. This skill clones
  remote repositories — that is a side effect with network and disk cost.
- **Never** invent a repository when none was provided. If parsing yields no
  repository, ask the user for one.
- **Always** show the expanded clone URL when the user supplied a shorthand
  `group/repo` form, so they can verify the target before cloning.
- **Preserve** instruction text verbatim. Do not reword, summarize, or trim
  it before passing it to `-i`.
