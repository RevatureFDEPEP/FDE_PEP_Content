# Matrix Builds for Multiple Environments

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

You've just learned that GitHub Actions runs jobs in parallel by default. A **matrix build** is a templated way to *generate* many parallel jobs from one job definition. Instead of copy-pasting `test-py-3.10`, `test-py-3.11`, `test-py-3.12` three times, you declare a matrix axis and Actions fans out the runs for you.

The pattern matters for two reasons. First, it scales: if you support N Python versions, you don't want N near-identical job blocks. Second, it makes the *intent* readable — the matrix block is a literal manifest of the environments you support.

The mistake most people make is overusing it. A matrix is right when you have **one piece of code that must work across multiple environments**. A matrix is wrong when you're tempted to use it to test multiple **different** services in one job definition — that's just per-service parallelism (Topic 1) and trying to cram it into a matrix axis makes the YAML harder to read, not easier.

## What a matrix expands to

A matrix is a Cartesian product of axes. Two axes of 3 values each produces 9 parallel jobs. Each job in the matrix is independent — they each get a fresh runner, run all the steps, and report independently.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
        python: ["3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest -q
```

That single job becomes 4 parallel runs: (ubuntu, 3.11), (ubuntu, 3.12), (macos, 3.11), (macos, 3.12). The UI groups them under the parent job name.

## `fail-fast`: usually false in CI

`fail-fast: true` (the default) cancels all in-progress matrix legs the moment one fails. That sounds efficient but it actively hides information: if your Python 3.11 leg fails, you'd like to also know whether 3.12 had the same problem or a different one. In CI for cross-version compatibility, almost always set `fail-fast: false`.

In a deploy-gating matrix (where you really do want to stop on first failure to save build minutes), `true` is defensible.

## Where matrix fits in the rev-eval-ai-pep substrate

Be honest about where it *doesn't* fit. The substrate runs a fixed Python version and a fixed Node version per service. There is no business case for multi-version testing here — it's an internal app, not a library. So don't fabricate a Python-version matrix to "use" the feature.

Legitimate matrix use cases for this cohort's pipeline:

1. **Matrix over services** as a refactor of Topic 1's fan-out. Once you have 3 backend services all running the same pytest steps, you can collapse them into one matrix job:

   ```yaml
   strategy:
     fail-fast: false
     matrix:
       service: [user-service, question-management-service, test-management-service]
   ```

   This is *only* a win when the steps are truly identical. If `question-management-service` needs a Mongo sidecar that the others don't, the matrix becomes a mess of `if: matrix.service == ...` conditionals and you should split them back out.

2. **Matrix for build-and-push of images** where the only differentiator is the service name and Dockerfile path. Cleaner than 5 hand-written `build-and-push-X` jobs.

3. **Future-proofing the Node version bump.** When Node 20 EOLs and you need to validate the frontend on Node 22, a temporary matrix is the right way to run both side-by-side until you cut over.

## Worked Scenario — collapsing three test jobs into a matrix

Before (three near-identical jobs):

```yaml
jobs:
  test-user-service:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - working-directory: services/user-service
        run: pip install -r requirements.txt -r requirements-dev.txt && pytest -q

  test-question-management-service:
    # ... same shape, different working-directory
  test-test-management-service:
    # ... same shape, different working-directory
```

After:

```yaml
jobs:
  test-backend:
    name: test-${{ matrix.service }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service:
          - user-service
          - question-management-service
          - test-management-service
    defaults:
      run:
        working-directory: services/${{ matrix.service }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
          cache-dependency-path: services/${{ matrix.service }}/requirements*.txt
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest -q
```

Three jobs become one definition, three parallel runs. The `name: test-${{ matrix.service }}` line preserves readable check names in the PR UI ("test-user-service", "test-question-management-service", ...).

## `include` and `exclude` — surgical matrix tweaks

If you mostly want a clean grid but with one variation, use `include` to add a single configuration with extra fields:

```yaml
matrix:
  service: [user-service, question-management-service]
  include:
    - service: question-management-service
      mongo: "7.0"
```

Now the qm leg has a `matrix.mongo` value and the user-service leg does not. Use `exclude` to remove specific combinations from the Cartesian product (e.g., a known-incompatible OS/version pair).

This works but reaches its readability limit fast. Past two or three `include`/`exclude` entries, it's clearer to split the matrix into two separate jobs.

## Common Pitfalls

- **Reaching for matrix when per-service parallelism is the right answer.** If the jobs share *nothing* but the YAML structure, just write separate jobs. Matrix is for "same recipe across different ingredients."
- **Forgetting `fail-fast: false`.** The first time a Python upgrade matrix leg fails and cancels the other legs, you lose the diagnostic signal you set the matrix up for in the first place.
- **Matrix jobs sharing a cache key.** Multiple matrix legs writing to `cache: pip` with no service in the cache-dependency-path can produce a single cache that's wrong for all of them. Use the matrix variable in the path to scope per-leg caches.
- **Status-check names changing.** If you rename a matrix axis value, the GitHub check name changes too — and branch protection that referenced the old name silently stops gating. The gate-job pattern from Topic 1 protects against this.
- **Cost surprise.** A 2x3x4 matrix is 24 parallel runners. On private repos that's billable minutes. Audit matrix sizes before merging.

## Key Takeaways

- A matrix templates one job definition into N parallel runs, one per combination of axis values.
- Set `fail-fast: false` whenever the matrix exists to gather cross-environment signal.
- Best fit in this substrate: collapsing three identical Python-service test jobs into one matrix.
- Use `include`/`exclude` sparingly; past a couple of entries, separate jobs are more readable.
- Matrix is not a substitute for thinking about whether the per-leg work is actually the same.

---

*Prerequisites: Day 3 (workflow/job/step structure), Day 7 Topic 1 (parallel job execution — matrix is just a templated version of that).*
