# YAML Syntax Pitfalls — Indentation, Quoting, Anchor References

> *Day 3: Diagnose the CI Pipeline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
At least one of today's five seeded bugs in `ci-pipeline.yml` is a YAML-level mistake: syntactically valid, semantically wrong. The YAML parser accepts it without complaint; GitHub Actions interprets it; the workflow runs; the wrong thing happens. This topic catalogs the YAML traps that produce silent misbehavior and trains you to spot them on a careful re-read.

## Why YAML is uniquely treacherous in CI

JSON has roughly one way to write each value. YAML has many. That flexibility creates failure modes that are invisible to the parser but produce wrong runtime behavior:

- **Whitespace is significant.** Two-space vs four-space indentation isn't cosmetic — it changes the structure.
- **The same characters mean different things in different contexts.** `on` can be a string or the boolean `true`. `no` can be a key or the boolean `false`. `1.20` can be a number or a string.
- **Validity is layered.** YAML can parse cleanly into a structure that GitHub Actions then rejects, or — worse — silently accepts as the wrong shape.

A workflow that "looks right" can be the wrong shape entirely. Read for **structure**, not just **content**.

## Indentation: the one-space difference

YAML uses indentation to define nesting. There is no fixed indent width — only consistency within a block. Two-space indentation is conventional and is what the GitHub Actions docs use.

```yaml
# CORRECT
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: pytest
        env:
          DATABASE_URL: postgres://localhost/test
```

Now examine the same file with one off-by-one indentation:

```yaml
# WRONG — env is now at the step list level, not inside "Run tests"
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: pytest
      env:                             # parsed as a key sibling to `steps:`
        DATABASE_URL: postgres://localhost/test
```

In the broken version, `env:` is now scoped to the job, not the step. The parser is happy. The workflow runs. The behavior is almost the same (because job-level env propagates to steps), but if you'd intended this env to be **only** for one step, you've just leaked it into every step. The bug only manifests if a downstream step is sensitive to that env var.

A subtler case:

```yaml
# WRONG — `with:` inputs lost
steps:
  - uses: actions/setup-python@v5
  with:                                # at step list level, not under setup-python
    python-version: '3.11'
```

This silently provides no inputs to `setup-python@v5`. The action runs with defaults (whatever Python the runner ships with), tests pass on `ubuntu-latest`'s default Python, and you don't notice until you upgrade and a 3.11-specific feature breaks.

**Diagnostic move:** when a step "doesn't seem to be using its config," re-check that the config block is indented one level deeper than the `-` of the step.

## Quoting: when YAML guesses wrong

YAML's type inference (the "Norway Problem") guesses the type of unquoted scalars. The classics:

| Written | Parsed as | Notes |
|---|---|---|
| `on` | `true` (boolean!) | YAML 1.1 legacy. GitHub Actions parsers vary. |
| `no` | `false` | Same. |
| `yes` / `Yes` / `YES` | `true` | Country code for Norway is `NO` — hence the name. |
| `1.20` | `1.2` (float) | Trailing zero lost. Common in version pins. |
| `1.20.0` | `"1.20.0"` (string) | Three dots → not a number. |
| `2024-01-15` | A date object | Most CI tools then stringify it inconsistently. |
| `0123` | `83` (octal!) | Leading zero. |

For CI configuration, the safest rule: **quote anything that looks like a version, a date, or a boolean-ish word**.

```yaml
# RISKY
node-version: 20.10
python-version: 3.11

# SAFE
node-version: '20.10'
python-version: '3.11'
```

The first form has bitten countless pipelines: `20.10` parses as `20.1`, which `setup-node@v4` interprets as "any 20.1.x" — pulling whatever 20.1.0/20.1.1 the runner has cached, not 20.10.x as you intended. The action doesn't error; it just installs a different version than you meant.

### Single vs double quotes

- `'single quotes'` — literal. No escape processing.
- `"double quotes"` — supports escapes (`\n`, `\t`, `\"`).

For shell commands inside `run:`, prefer single quotes on the YAML level so YAML doesn't process `\n` before the shell sees it.

```yaml
# YAML processes \n into a real newline BEFORE the shell sees the command
- run: "echo line1\nline2"

# Shell sees the literal string with a backslash-n
- run: 'echo "line1\nline2"'
```

## Block scalars: `|` vs `>` vs folded vs literal

Multi-line `run:` blocks use block scalar indicators. Get these wrong and your script changes meaning.

```yaml
# LITERAL (|): newlines preserved
- run: |
    set -e
    cd services/user-service
    pytest

# FOLDED (>): newlines become spaces
- run: >
    set -e
    cd services/user-service
    pytest
# Equivalent to: "set -e cd services/user-service pytest"
# This is one long line — set -e and cd run on the same line as pytest.
# bash will try to execute "set -e cd services/user-service pytest" — error.
```

Almost every multi-step shell block you write in CI should use `|`, not `>`. If you see `>` on a `run:` block with multiple commands, that's a seeded-bug candidate.

## Anchors and aliases

YAML supports DRY definitions via anchors (`&name`) and aliases (`*name`):

```yaml
# Define an anchor on a reusable env block
defaults: &python-env
  PYTHONUNBUFFERED: '1'
  PYTHONDONTWRITEBYTECODE: '1'

jobs:
  test-user:
    env: *python-env                   # alias — copies the anchor
    steps: [...]

  test-question:
    env:
      <<: *python-env                  # merge key — extends the anchor
      EXTRA_VAR: 'value'
```

GitHub Actions supports YAML anchors but **not the merge key (`<<`)** in older parser versions, and even modern parsers handle it inconsistently. If you see `<<: *something` in a GitHub workflow, treat it as suspect — it may parse on your local linter but fail or be ignored on the runner.

Anchor pitfalls:
- An alias produces a **reference**, not a deep copy. In tools that mutate the parsed structure, mutations to one location can affect the other.
- `*name` before `&name` is a forward reference and is undefined behavior in YAML 1.1.
- Misspelling the alias (`*python-en` instead of `*python-env`) is a parse error — at least this one fails loudly.

## Lists vs maps: the dash matters

```yaml
# A list of strings
branches:
  - main
  - develop

# A map (will fail or behave oddly where a list is expected)
branches:
  main:
  develop:
```

The first is a list of two strings. The second is a map with two keys whose values are both null. Both parse cleanly. GitHub Actions expects a list under `branches:`, so the second form gets parsed into a map and ignored (or worse — interpreted as no branch filter, meaning the workflow runs on every push).

## Example / Worked Scenario

Spot the YAML pitfall in this snippet from `ci-pipeline.yml`:

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  test-services:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service:
          - user-service
          - question-management-service
          - test-management-service
        python-version: 3.11           # ← bug candidate
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r services/${{ matrix.service }}/requirements.txt
      - run: pytest services/${{ matrix.service }}/tests
        env:
          DATABASE_URL: postgres://localhost/test
        continue-on-error: true        # ← bug candidate
```

Two things to flag on a careful read:

1. `python-version: 3.11` — unquoted, would parse as the float `3.11`, then get string-coerced when interpolated. In practice this usually resolves correctly because `setup-python@v5` accepts numbers, but it's fragile across parser versions and `3.10` → `3.1` is the famous failure case. **Quote it.**
2. `continue-on-error: true` is at the step level. Visually it looks like it belongs to the `env:` block above (same indentation as `env:`), but it actually belongs to the step. It means *this test step will pass even when pytest fails*. That's the kind of YAML-level bug that produces a green CI when tests are broken — exactly the seeded-bug profile.

The first is a low-grade type-inference smell. The second is a real semantic bug: every test failure silently passes, so you have no test signal at all.

## Common Pitfalls

- **Unquoted versions getting truncated.** `node-version: 20.10` becomes `20.1`. Always quote: `'20.10'`.
- **`continue-on-error: true` hiding failures.** Sometimes legitimate, sometimes a debugging shortcut left behind, sometimes a seeded bug. Always question it.
- **Indentation that puts `with:`, `env:`, or `if:` at the wrong scope.** Re-trace the indentation chain from the step `-` upward.
- **Folded scalars (`>`) on multi-command shell blocks.** Use literal (`|`) for shell scripts; folded is for prose-like wrapping only.
- **Merge keys (`<<: *anchor`).** Parser support is uneven across CI platforms. Prefer copy-paste over merge keys for portability.

## Key Takeaways

- YAML parses many wrong structures successfully — syntactic validity is a low bar.
- Indentation defines scope. A one-space slip moves a key from "inside this step" to "inside the job."
- Quote versions, dates, and boolean-looking words. The Norway Problem and float-truncation bite real pipelines.
- Use `|` for multi-line `run:` blocks. `>` collapses newlines and is almost never what you want for shell.
- When a CI behavior doesn't match the apparent intent, suspect indentation or quoting before you suspect logic.

---
*Prerequisites: [01-github-actions-structure-workflows-jobs-steps-runners.md](01-github-actions-structure-workflows-jobs-steps-runners.md).*
