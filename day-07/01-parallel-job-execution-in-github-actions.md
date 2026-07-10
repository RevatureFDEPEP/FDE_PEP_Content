# Parallel Job Execution in GitHub Actions

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

On Day 3 you saw GitHub Actions structure — workflows contain jobs, jobs contain steps. On Day 4 you got the pipeline green. The pipeline you fixed was almost certainly **serial**: one job ran, then the next, then the next. That works, but it wastes wall-clock time. Every minute a developer waits for CI is a minute they're context-switching away from the change they just pushed.

Today's topic is the simplest CI speedup available: run independent jobs in parallel. GitHub Actions does this **by default** — every job in a workflow runs on its own runner, concurrently with its siblings, unless you explicitly chain them with `needs:`. The skill is recognizing which jobs *can* be parallel (no shared state, no ordering dependency) and refactoring a monolithic "build then test then lint" job into a fan-out shape.

## What parallelism buys you in CI

A serial pipeline's wall time is the **sum** of its job durations. A parallel pipeline's wall time is the **max** of its parallel branch durations plus any serial tail. Consider the rev-eval-ai-pep substrate:

- `user-service` (Python, FastAPI) — install deps, run pytest, build image
- `question-management-service` (Python, FastAPI) — install deps, run pytest, build image
- `test-management-service` (Python, FastAPI) — install deps, run pytest, build image
- `api-gateway` (Node/Express) — install deps, run vitest, build image
- `frontend` (Next.js) — install deps, run vitest, build image

Running these one-by-one might take 18 minutes. Running them in parallel takes about 4 minutes — the duration of the slowest single service. The two backends with the heaviest test suites set the floor; everything else hides behind them.

## When jobs cannot be parallel

The `needs:` keyword forces sequencing. You need it when:

- A downstream job consumes the **artifact** of an upstream job (Topic 6 covers this).
- A downstream job should only run if the upstream **passed** (e.g., don't push images if tests failed).
- Two jobs both mutate **shared external state** (e.g., a remote DB schema) and can't safely interleave.

If none of those apply, do not add `needs:`. A common anti-pattern is adding `needs:` "to be safe" — this serializes the pipeline unnecessarily.

## The fan-out / fan-in shape

A mature CI pipeline often looks like:

```
       ┌── lint-python ──┐
       ├── lint-js ──────┤
trigger┤                 ├── gate ── deploy
       ├── test-user ────┤
       ├── test-qm ──────┤
       └── build-images ─┘
```

The leaf jobs are independent and parallel. The `gate` job uses `needs: [lint-python, lint-js, test-user, test-qm, build-images]` and only succeeds when all of them succeed. Downstream jobs depend on `gate` rather than enumerating every leaf.

## Worked Scenario

The starting workflow has one giant `ci` job that runs everything serially. Refactor:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test-user-service:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/user-service
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
          cache-dependency-path: services/user-service/requirements*.txt
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest -q

  test-question-management-service:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/question-management-service
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
          cache-dependency-path: services/question-management-service/requirements*.txt
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest -q

  test-frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: pnpm
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - run: pnpm install --frozen-lockfile
      - run: pnpm test

  ci-gate:
    runs-on: ubuntu-latest
    needs: [test-user-service, test-question-management-service, test-frontend]
    steps:
      - run: echo "All CI checks passed"
```

Three jobs run in parallel. The `ci-gate` job is a synthetic merge point — useful for branch protection rules ("require status check: ci-gate") because it lets you add or rename leaf jobs without re-configuring GitHub's branch protection settings.

## Concurrency control

Parallel jobs across a *single* run is one thing. Parallel *runs* of the workflow (e.g., a developer pushes three commits in 30 seconds) is another. Use `concurrency:` to cancel superseded runs on the same branch:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

For PR branches this is almost always what you want — only the latest push needs to pass. For `main` it's a judgment call; usually leave older `main` runs alone in case they're producing artifacts.

## Common Pitfalls

- **`needs:` cargo-culted onto every job.** Re-read each `needs:` and ask "would this job actually fail or be wrong if the upstream had not run?" If not, delete it.
- **Shared cache key thrashing.** Two parallel jobs writing to the same `actions/cache` key race each other and one's writes are wasted. Use distinct keys per language/service (Day 3's topic on caching is the reference).
- **Runner concurrency limits.** Free GitHub plans cap concurrent Linux runners (20 on Free, more on Team/Enterprise). A 12-job fan-out on a Free org may queue; you'll see runs "waiting for runner" with no progress. Check `Settings → Actions → Usage limits` if jobs sit pending.
- **Forgetting the gate job.** Without `ci-gate`, branch protection has to enumerate every leaf job by name. Add a new service tomorrow and the protection rule silently doesn't cover it. The gate centralizes the contract.
- **Parallel jobs hitting the same external service.** Two pytest suites talking to one shared Postgres on a CI runner will collide. Each parallel job needs its *own* ephemeral dependencies (typically via the job's `services:` block or a fresh container).

## Key Takeaways

- Jobs in GitHub Actions run in parallel **by default**; `needs:` is opt-in serialization.
- Refactor a monolithic "do everything" job into per-service or per-concern jobs to expose parallelism.
- The pipeline's wall time becomes the duration of the slowest parallel branch, not the sum of all branches.
- A `ci-gate` job aggregating `needs:` simplifies branch protection and lets you add new checks without touching repo settings.
- Use `concurrency:` with `cancel-in-progress` on PR branches to avoid wasting runners on superseded pushes.

---

*Prerequisites: Day 3 (Actions ecosystem, workflow structure), Day 4 (fixed pipeline you can now refactor), Day 5 (image build steps that will become parallel jobs).*
