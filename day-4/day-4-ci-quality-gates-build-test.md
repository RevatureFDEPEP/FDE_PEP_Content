# CI Quality Gates (Build → Test)

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
You diagnosed the workflow yesterday and you're fixing it today. This topic zooms out and asks: *what is the pipeline actually for?* The answer in v3.0 of the trainer repo's pipeline is deliberately narrow — **build then test, no deploy** — and understanding why that scope was chosen matters for how you reason about CI failures and what "green" actually proves.

## The Gate Defined
A "quality gate" is a checkpoint that a change must pass before progressing. The trainer pipeline has exactly two gates, in order:

```
PR opened / updated
        |
        v
   [ build gate ]   <-- does the code compile/transpile cleanly?
        |
   pass | fail
        |       \--> PR cannot merge; author iterates
        v
   [ test gate ]   <-- do the unit tests pass against the build?
        |
   pass | fail
        |       \--> PR cannot merge; author iterates
        v
   merge button enables
```

That's the entire pipeline today. v3.0 deliberately omits a deploy gate. Earlier versions of `rev-eval-ai` had a `deploy` job that pushed to ECS; in the PEP brownfield variant that's stripped out (per the scope map). Why:
- A failing deploy step muddies CI signal — "is the code wrong, or is AWS flaky?"
- Deploy is a Week 4 topic in the 10-week intensive, not Week 1 of PEP
- Build+test is enough to enforce code quality for PR-based delivery

You'll add deploy gates later. Today's gate is build+test, and that's what "green" means.

## What Each Gate Proves
**Build gate (`build` job):**
- Source code is syntactically valid in every service
- All declared dependencies resolve and install
- TypeScript / Java / etc. compile cleanly
- Container images build (if the workflow exercises that)

Failure modes blocked: missing imports, syntax errors, broken `package.json`, missing `tsconfig` references, Dockerfile errors.

**Test gate (`test` job):**
- Unit tests in each service execute against the built code
- All tests pass (no skipped tests counting as pass)
- Exit code 0 from the test runner

Failure modes blocked: regressions in existing logic, new code that doesn't match the contracts existing tests assert.

What the gate **does not prove** (yet):
- That the code does what the PR description says (still requires human review)
- That services integrate correctly (no integration tests in v3.0)
- That the change is deployable (no deploy stage)
- That security/perf is acceptable (no SAST/load-test stage)

Knowing what the gate doesn't prove is as important as knowing what it does. A green CI run is **necessary but not sufficient** for merge — peer review covers the rest.

## Gate Ordering and Short-Circuit
The test job depends on the build job. In `ci-pipeline.yml`:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.11.1
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  test:
    runs-on: ubuntu-latest
    needs: build      # <-- explicit dependency
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.11.1
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: npm ci
      - run: npm test
```

The `needs: build` line is the short-circuit: if build fails, test is skipped entirely (reported as "skipped", not "failed"). This saves CI minutes and produces a cleaner signal — you see "build failed" rather than "build failed and 47 cascading test failures because dist/ doesn't exist."

## Demonstrating the Gate Passes
By end of day, your evidence package for the deliverable is:

1. **All 5 fix PRs merged to `main`.**
2. **CI run on `main` showing build + test green** — a link to the Actions run.
3. **A short note in the cohort channel** confirming the pipeline is green and naming any caveats (e.g., "test job runs 12 specs in 38s; one test is `.skip`'d with a TODO from before the cohort started").

The note matters. Future-you (and future cohorts) need to know what "green" included.

## Example / Worked Scenario
After PR #18 (the fifth fix) merges, you click into the post-merge Actions run on `main`:

```
ci-pipeline / build      ✓ 1m 47s
ci-pipeline / test       ✓ 0m 52s
```

You click into `test` and see:
```
Test Suites: 7 passed, 7 total
Tests:       1 skipped, 89 passed, 90 total
Snapshots:   0 total
Time:        38.221 s
```

You note the 1 skipped test. You inspect it:
```javascript
it.skip('integrates with reporting-and-analytics service', () => {
  // TODO: re-enable when reporting service has an HTTP handler
});
```

That's a legitimate skip — the `reporting-and-analytics` service is empty by design (it's a Week 3 build target). You include the skip in your end-of-day note: *"Pipeline green. One test skipped (reporting-and-analytics integration) — expected; that service is empty per scope map."*

This is what a real CI green-light looks like in a brownfield repo: not "zero noise" but "all noise accounted for."

## Common Pitfalls
- **Reading "skipped" as "passed".** A skipped test is a test that didn't run. Track them and have a reason for each.
- **Trusting cached green.** If your branch was green yesterday and you didn't push, the green is stale. Re-run after rebasing.
- **Treating CI as the only quality signal.** Build+test passing means the code compiles and unit tests pass. It does not mean the change is correct. Reviewers cover the rest.
- **Adding a "fix CI" commit that just re-runs the workflow.** If the workflow is flaky, fix the flake. Don't paper over it with retries.

## Key Takeaways
- v3.0 pipeline is **build then test, no deploy** — by design.
- Build proves the code compiles; test proves units pass. Nothing more.
- Test depends on build (`needs: build`) — fail-fast on build errors.
- Green CI is necessary but not sufficient for merge; peer review covers correctness.
- Account for skipped tests; don't let them silently grow.

---
*Prerequisites: `day-3-github-actions-structure-yaml-pitfalls.md`, `day-4-branch-protection-and-required-status-checks.md`.*
