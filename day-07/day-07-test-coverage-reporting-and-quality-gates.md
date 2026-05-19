# Test Coverage Reporting and Quality Gates

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

"The tests pass" is a weak claim if half the code isn't tested. Coverage tools measure which lines, branches, and functions executed during the test run, and report what percentage of the code was reached. A **quality gate** turns that report into a CI decision: if coverage drops below a threshold, the build fails.

Coverage gates are valuable but also commonly misused. Treated naively they incentivize useless tests written purely to bump the number. Treated thoughtfully they prevent silent regressions where a refactor accidentally bypasses a tested code path, or a new feature lands without tests at all.

The right framing: coverage is a **necessary but not sufficient** quality signal. 80% coverage with thoughtful assertions beats 95% coverage where every test ends in `assert True`. The CI gate exists to prevent the floor from sinking; it does not certify the ceiling.

## Coverage tooling for the substrate

- **Python services** (`user-service`, `question-management-service`, `test-management-service`) — `pytest-cov`, a pytest plugin that wraps `coverage.py`. Reports per-file and total line/branch coverage, supports XML/JSON/HTML output formats.
- **Node services** (`api-gateway`, `frontend`) — `vitest --coverage` using the built-in v8 coverage provider (or `@vitest/coverage-istanbul` for finer-grained branch coverage). Configured via `vitest.config.ts`.

Both ecosystems emit a standard `coverage.xml` (Cobertura) or `lcov.info` file, which is the format Codecov, Coveralls, and the GitHub Action `irongut/CodeCoverageSummary` consume.

## Wiring pytest-cov into a Python service

```toml
# services/user-service/pyproject.toml
[tool.pytest.ini_options]
addopts = "--cov=app --cov-report=term-missing --cov-report=xml --cov-branch"

[tool.coverage.run]
branch = true
source = ["app"]
omit = [
  "app/__main__.py",
  "app/migrations/*",
]

[tool.coverage.report]
fail_under = 80
show_missing = true
skip_covered = false
```

`fail_under = 80` is the threshold. If coverage drops below 80%, `pytest` exits non-zero and the CI step fails. `--cov-branch` adds branch coverage (was every `if`/`else` direction taken, not just every line). `omit` skips files where coverage is meaningless (entry points, generated migration scripts).

## Wiring Vitest coverage into the frontend

```ts
// frontend/vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov", "json-summary"],
      reportsDirectory: "./coverage",
      thresholds: {
        lines: 75,
        branches: 70,
        functions: 75,
        statements: 75,
      },
      exclude: [
        "**/*.config.*",
        "**/*.d.ts",
        ".next/**",
        "app/**/layout.tsx",
      ],
    },
  },
});
```

Vitest's `thresholds` block is the equivalent of pytest's `fail_under`. Below threshold, the run exits non-zero.

## The CI step

```yaml
# .github/workflows/ci.yml — within the matrix test job
- name: Run tests with coverage
  working-directory: services/${{ matrix.service }}
  run: pytest -q --cov=app --cov-report=xml --cov-report=term

- name: Upload coverage XML as artifact
  uses: actions/upload-artifact@v4
  with:
    name: coverage-${{ matrix.service }}
    path: services/${{ matrix.service }}/coverage.xml
    retention-days: 7

- name: Comment coverage summary on PR
  if: github.event_name == 'pull_request'
  uses: irongut/CodeCoverageSummary@v1.3.0
  with:
    filename: services/${{ matrix.service }}/coverage.xml
    badge: true
    format: markdown
    output: both
    thresholds: "70 80"
```

The summary action posts a per-file coverage breakdown as a PR comment. Reviewers can see "this PR drops `app/quiz.py` from 88% to 61%" without leaving the diff.

## Differential coverage — the gate that scales

A flat `fail_under = 80` works for a small project. On a brownfield codebase like the substrate it has a problem: existing files may already sit below 80% and you don't want to retrofit them today. You also don't want a developer to be blocked from merging an unrelated change because some other file is below threshold.

The fix is **differential coverage**: gate on the coverage of *lines changed in this PR*, not the whole repo. Tools like `diff-cover` compute this from a git diff plus a coverage report:

```yaml
- name: Differential coverage gate
  if: github.event_name == 'pull_request'
  run: |
    pip install diff-cover
    diff-cover services/${{ matrix.service }}/coverage.xml \
      --compare-branch origin/${{ github.base_ref }} \
      --fail-under=85
```

Now the rule reads: "any new or modified line must be covered to 85%." Existing uncovered code is not your problem until you touch it. This is the gate posture most mature teams settle on.

## Worked Scenario — gate trips on a real PR

A developer adds a `delete_question` endpoint to `question-management-service` and pushes. CI runs and the diff-cover gate fails:

```
Diff Coverage
Diff: origin/main...HEAD, staged and unstaged changes
-------------
app/routes/questions.py (62.5%): Missing lines 87-92
app/db/questions.py    (100.0%)
-------------
Total:   8 lines
Missing: 3 lines
Coverage: 62%
Fail. Coverage 62% < 85% threshold.
```

The developer reads `app/routes/questions.py` lines 87-92 and finds:

```python
if not question:
    raise HTTPException(404, "Question not found")
if question.author_id != current_user.id and not current_user.is_admin:
    raise HTTPException(403, "Cannot delete questions you didn't author")
```

Both error paths exist; neither has a test. The fix is two small tests covering the 404 and 403 branches. Coverage on the diff goes to 100%, the gate passes, the PR is mergeable. The gate did its job — it caught a real gap in a security-relevant code path before merge.

## Common Pitfalls

- **Setting the threshold to whatever the current number is.** Aspirational thresholds (set the bar slightly above the current floor and ratchet it up) push the codebase forward. Setting it to "wherever we are today" cements mediocrity.
- **Counting integration tests as unit coverage.** A coverage report that includes integration-test runs against a real DB will inflate numbers — but those tests are slow and brittle. Run them but report coverage *separately* from the unit suite.
- **Lines covered, behavior not asserted.** A test that imports a module and runs a function but asserts nothing about the output gives you 100% coverage and zero confidence. Coverage gates plus mutation testing (out of scope for this cohort but worth knowing exists) is the real safety net.
- **Forgetting `--cov-branch`.** Line coverage misses `else` branches that never execute. Branch coverage is the more honest number.
- **Threshold drift on `main`.** If `main` is at 82% and the threshold is 80%, a sequence of PRs each landing at exactly 80% can erode the floor. Differential coverage avoids this trap.
- **Excluded files growing.** Every entry in `omit`/`exclude` is a place coverage is not measured. Periodically audit the list — old entries linger and shield code that should be tested.

## Key Takeaways

- Coverage measures which code executed during tests; quality gates make CI fail when coverage drops.
- Use `pytest-cov` with `fail_under` for Python services, Vitest `coverage.thresholds` for Node services.
- Differential coverage (`diff-cover`) is the right gate posture for a brownfield codebase: enforce on changed lines, leave existing debt alone until it's touched.
- Branch coverage (`--cov-branch`) is more honest than line coverage; turn it on.
- Coverage is necessary, not sufficient. A passing gate does not guarantee good tests, only the presence of tests.

---

*Prerequisites: Day 4 (a green CI pipeline you can extend), Day 7 Topics 1-2 (per-service test jobs you'll add coverage to).*
