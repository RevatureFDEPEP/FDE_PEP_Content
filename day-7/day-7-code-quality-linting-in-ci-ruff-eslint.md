# Code Quality Linting in CI (Ruff, ESLint)

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

A linter is a static analyzer that reads your source code and flags patterns: stylistic inconsistencies, likely bugs, dead code, dangerous idioms, security-relevant constructs. It runs in milliseconds, requires no test fixtures, and catches an enormous class of issues before a reviewer ever has to read the diff.

CI linting closes the social loop. A linter that only runs on a developer's laptop catches mistakes only if the developer remembered to run it. A linter wired into CI as a gating check catches them every time, with no human required, and removes "could you fix the formatting?" from PR reviews entirely.

Today you add two linters: **Ruff** for the Python services and **ESLint** for the Next.js frontend (plus the api-gateway). They share an organizational pattern — config in a single file, run locally and in CI with identical settings — but their philosophies differ. The substrate uses both.

## Style errors vs quality errors

Lint findings split into two broad classes, and the CI posture for each is different:

- **Style errors** — `line too long`, `missing trailing comma`, `inconsistent quote style`, `unused import`. Mechanical. Auto-fixable. Reasonable people disagree about the *right* answer but consistent application is more valuable than the answer. **In CI: fail the build, but the fix is `ruff format` / `eslint --fix` and a re-push, not a thoughtful refactor.**

- **Quality errors** — `mutable default argument`, `comparison to None with ==`, `await inside non-async function`, `unreachable code`, `bare except`, `floating promise`, `var instead of let/const`. These flag real or likely bugs. Not auto-fixable. Require thinking. **In CI: fail the build; the fix requires the developer to understand what they wrote and correct the underlying issue.**

A mature lint config tunes both classes for the project. Style: pick a convention and enforce it ruthlessly. Quality: enable the high-signal rules, suppress the false-positive-prone ones with documented overrides.

## Ruff for Python

Ruff (from Astral) is a Rust-implemented Python linter that subsumes flake8, isort, pyupgrade, pydocstyle, and several others into one fast tool. It also formats (replacing Black) with `ruff format`. One config, one binary, one CI step.

```toml
# pyproject.toml — works at the repo root or per-service
[tool.ruff]
target-version = "py311"
line-length = 100
extend-exclude = ["migrations", "alembic/versions"]

[tool.ruff.lint]
# Selected rule families:
# E/W — pycodestyle (style)
# F   — pyflakes (likely bugs: unused imports, undefined names)
# I   — isort (import ordering)
# B   — flake8-bugbear (likely-bug patterns: mutable defaults, etc.)
# C4  — flake8-comprehensions
# UP  — pyupgrade (use modern Python idioms)
# S   — flake8-bandit (security)
# RUF — Ruff-specific lints
select = ["E", "W", "F", "I", "B", "C4", "UP", "S", "RUF"]
ignore = [
  "E501",  # line length — let the formatter handle, don't double-error
  "S101",  # assert in tests is fine
]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "S105", "S106"]  # asserts and hardcoded passwords are fine in tests

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
line-ending = "lf"
```

Local usage:

```bash
ruff check .           # quality + style check
ruff check --fix .     # auto-fix what's auto-fixable
ruff format .          # apply formatting
ruff format --check .  # CI mode: fail if not already formatted
```

The CI step:

```yaml
  lint-python:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: [user-service, question-management-service, test-management-service]
    defaults:
      run:
        working-directory: services/${{ matrix.service }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install ruff==0.6.9
      - name: Ruff lint
        run: ruff check --output-format=github .
      - name: Ruff format check
        run: ruff format --check .
```

`--output-format=github` makes Ruff emit findings as GitHub Actions annotations, which renders inline on the PR diff. Reviewers see the lint error right next to the line that produced it.

## ESLint for the Next.js frontend

ESLint is older, more configurable, and more painful. Next.js ships with its own ESLint preset that gives you sensible defaults; you augment from there.

```js
// frontend/eslint.config.mjs (flat config, ESLint 9+)
import next from "@next/eslint-plugin-next";
import tseslint from "typescript-eslint";

export default tseslint.config(
  {
    ignores: [".next/**", "node_modules/**", "coverage/**"],
  },
  ...tseslint.configs.recommended,
  {
    plugins: { "@next/next": next },
    rules: {
      ...next.configs.recommended.rules,
      ...next.configs["core-web-vitals"].rules,

      // Quality rules (must-address)
      "no-floating-promises": "error",
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
      "@typescript-eslint/no-explicit-any": "warn",
      "react-hooks/rules-of-hooks": "error",
      "react-hooks/exhaustive-deps": "warn",

      // Style — handled by Prettier separately, keep ESLint focused on quality
      "semi": "off",
      "quotes": "off",
    },
  }
);
```

`package.json` scripts:

```json
{
  "scripts": {
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

Note the split: ESLint handles quality, Prettier handles formatting. Having ESLint also enforce semicolons and quotes is doable but creates rule conflicts with Prettier; the modern convention is to let them each own one job.

The CI step:

```yaml
  lint-frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: pnpm
          cache-dependency-path: frontend/pnpm-lock.yaml
      - run: pnpm install --frozen-lockfile
      - name: ESLint
        run: pnpm lint
      - name: Prettier check
        run: pnpm format:check
```

## Worked Scenario — a PR that the linter catches

A developer adds an export endpoint to `question-management-service`:

```python
# services/question-management-service/app/routes/export.py
import json, os
from fastapi import APIRouter
from app.db import questions

router = APIRouter()

def write_export(path, items=[]):  # mutable default — B006
    items.extend(questions.all())
    with open(path, 'w') as f:      # S108 if /tmp, also single-quote vs double — formatter
        json.dumps(items, fp=f)     # F841 typo: should be json.dump

@router.post("/export")
def export():
    write_export("/tmp/export.json")
    return {"ok": True}
```

Ruff output in CI:

```
::error file=services/question-management-service/app/routes/export.py,line=1,col=1::F401 [*] `os` imported but unused
::error file=services/question-management-service/app/routes/export.py,line=7,col=33::B006 Do not use mutable data structures for argument defaults
::error file=services/question-management-service/app/routes/export.py,line=9,col=29::Q000 [*] Single quotes found but double quotes preferred
::error file=services/question-management-service/app/routes/export.py,line=10,col=9::F841 Local variable might be wrong: did you mean `json.dump`?
```

The `[*]` markers indicate auto-fixable. The developer runs `ruff check --fix` locally and `os` import + quote style fix themselves. The mutable default (B006) and the `json.dumps` vs `json.dump` typo (F841 — though Ruff's heuristic for this varies) require thought:

```python
def write_export(path: str, items: list | None = None) -> None:
    items = list(items) if items else []
    items.extend(questions.all())
    with open(path, "w") as f:
        json.dump(items, f)
```

The linter caught a real bug (`json.dumps` writes to a string and ignores the `fp=` kwarg in some versions; this would have shipped a route that always wrote `null` to the file). Style cleanup was automatic. Quality cleanup needed the developer.

## Auto-fix on PR — a productivity boost

A common pattern: a GitHub Action that runs `ruff check --fix` and `pnpm lint --fix` on PR branches and commits the fixes as a follow-up commit. This eliminates the "the linter failed; I fixed it; please re-review" round-trip for purely mechanical issues.

```yaml
  autofix:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with: { ref: ${{ github.head_ref }} }
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install ruff
      - run: ruff check --fix . || true
      - run: ruff format .
      - uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "chore: ruff autofix"
```

Use with judgment — auto-commits to PR branches can be confusing for the original author if they're mid-edit. Some teams prefer a `/lint` PR comment trigger instead of automatic.

## Common Pitfalls

- **Linter enabled in CI but not locally.** Developers push, CI fails, they fix and push again. Every PR. Provide a `pre-commit` config or a Makefile target so the same linter runs on save or pre-commit locally.
- **Disabling rules without justification.** Same anti-pattern as `.trivyignore` without comments. Inline `# noqa: B006` should always carry a reason: `# noqa: B006 — empty default is a sentinel; mutation is intentional and guarded`.
- **Running the linter only on changed files locally and the whole repo in CI.** Divergent rule application leads to surprise CI failures. Run on the whole repo in both places, or use `--select` with the same scope.
- **ESLint version drift between local and CI.** Different ESLint versions enforce different rules. Pin ESLint and all plugins in `package.json`; same for Ruff via `pip install ruff==X.Y.Z`.
- **Style flame wars.** Tabs vs spaces, single vs double quotes — these debates produce zero value. Pick whatever the auto-formatter does, commit it, never discuss again. The point of the linter is to take this conversation off the table.
- **Forgetting the formatter.** `ruff check` does not auto-format unless you also run `ruff format`. Add both to CI. Same for ESLint + Prettier.

## Key Takeaways

- Ruff handles Python (lint + format) and ESLint + Prettier handles JS/TS in the substrate.
- Distinguish style errors (auto-fixable, just re-run the tool) from quality errors (need a thoughtful fix).
- Configure once in `pyproject.toml` / `eslint.config.mjs`; run locally and in CI from the same config.
- `--output-format=github` (Ruff) and the default ESLint format render findings as inline PR annotations.
- Suppressions must carry a justification comment; an undocumented `# noqa` is technical debt.
- CI-enforced linting eliminates style debate from PR review and catches a real class of bugs before they ship.

---

*Prerequisites: Day 4 (a green CI you can extend with lint jobs), Day 7 Topic 1 (parallel jobs — lint jobs run alongside test jobs, not after).*
