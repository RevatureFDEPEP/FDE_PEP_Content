# CI Pipeline Optimization and Execution Time Reduction

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

A CI pipeline that takes 18 minutes does not just consume runner minutes — it consumes *developer attention*. A 90-second pipeline lets you push, glance at the result, and move on. An 18-minute pipeline forces context-switching, and context-switching is the single most expensive thing a developer does in a day. Optimizing CI is one of the highest-leverage things you can do for a team's throughput.

Today's earlier topics gave you the building blocks: parallelism (Topic 1), matrix (Topic 2), and artifacts (Topic 6). This topic is the synthesizing skill — measuring where your pipeline spends its time and applying the right optimization for each hot spot. The mental model is the same as profiling any program: don't guess, measure; fix the biggest cost first.

## How to measure

Two free signals from GitHub:

1. **Run timing in the UI.** The workflow run page shows total duration and per-job duration. The job view shows per-step duration (collapsed by default; expand the dropdown arrow). For most optimization work, this is enough.

2. **The Actions API or `gh run view`.** Programmatic for trend tracking. `gh run list --workflow=ci.yml --limit 20 --json conclusion,startedAt,updatedAt,databaseId` gives you the last 20 runs' durations, which you can dump into a spreadsheet to see drift.

For deeper analysis, the `actions/timing` actions or homegrown step-timestamp logging can produce per-step gantt charts. Usually unnecessary — the job view tells you 95% of what you need.

## The hierarchy of optimizations

Apply in roughly this order, biggest wins first:

1. **Parallelize what's accidentally serial.** Topic 1's lesson. The cheapest 5x speedup you'll ever get.
2. **Cache dependencies.** Day 3's caching topic. `pip`, `pnpm`, `apt`, Docker layers. A cold install is minutes; a cache hit is seconds.
3. **Skip work that doesn't need to run.** Path filters, draft-PR exclusions, `[skip ci]` markers, conditional jobs.
4. **Reduce what you build/test.** Use buildx layer caching for Docker. Use `pytest --testmon` or pytest's `-k` selection to run only affected tests on PRs (full suite on `main`).
5. **Use faster runners.** Larger GitHub-hosted runners or self-hosted runners. Last resort — you're throwing money at the problem, not engineering it away.

The first three are typically 80% of the available win, at zero infrastructure cost.

## Before / after — a worked optimization pass

**Before — single serial job, no caching, 18 minutes:**

```yaml
# .github/workflows/ci.yml — the starting state from Day 4
name: CI
on: [push, pull_request]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - uses: pnpm/action-setup@v3
        with: { version: 9 }

      # Backend setup + test — all three services serially
      - run: cd services/user-service && pip install -r requirements.txt -r requirements-dev.txt && pytest
      - run: cd services/question-management-service && pip install -r requirements.txt -r requirements-dev.txt && pytest
      - run: cd services/test-management-service && pip install -r requirements.txt -r requirements-dev.txt && pytest

      # Frontend setup + test
      - run: cd frontend && pnpm install && pnpm test

      # Image builds — no cache
      - run: docker build -t user-service services/user-service
      - run: docker build -t qm-service services/question-management-service
      - run: docker build -t tm-service services/test-management-service
      - run: docker build -t frontend frontend
```

Timing breakdown:
- pip install x3: 3 x 90s = 4m 30s
- pytest x3: 3 x 45s = 2m 15s
- pnpm install: 70s
- pnpm test: 40s
- docker build x4 (no cache): 4 x 2m = 8m
- checkout + setups + overhead: 1m
- **Total: ~16m 35s** wall time

**After — parallel matrix, cached, layer-cached Docker, 4m 10s:**

```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  test-backend:
    name: test-${{ matrix.service }}
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
          cache: pip
          cache-dependency-path: services/${{ matrix.service }}/requirements*.txt
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest -q --cov=app --cov-report=xml

  test-frontend:
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
      - run: pnpm test

  build-image:
    name: build-${{ matrix.service }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: [user-service, question-management-service, test-management-service, frontend]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Build (with GHA cache)
        uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.service == 'frontend' && 'frontend' || format('services/{0}', matrix.service) }}
          push: false
          tags: rev-eval-ai-pep/${{ matrix.service }}:${{ github.sha }}
          cache-from: type=gha,scope=${{ matrix.service }}
          cache-to: type=gha,mode=max,scope=${{ matrix.service }}

  ci-gate:
    runs-on: ubuntu-latest
    needs: [test-backend, test-frontend, build-image]
    steps:
      - run: echo "All checks passed"
```

Timing breakdown (cache warm):
- Backend matrix (3 parallel): max(setup + pip-cached + pytest) = ~70s
- Frontend (parallel): pnpm-cached + test = ~75s
- Build matrix (4 parallel) with GHA layer cache: max ~3m 50s
- **Total: ~3m 50s** (gated by the slowest parallel branch — the image builds)

The pipeline went from 16:35 to 3:50. A **4.3x speedup**, no money spent, no new infrastructure. The remaining floor is the Docker builds — if you want to drop further, that's where the next round of work goes (smaller images, more aggressive layer cache hits, parallel multi-stage builds with `--mount=type=cache`).

## Specific techniques worth knowing

### Path filters

Don't run the frontend test job if only Python files changed:

```yaml
on:
  pull_request:
    paths:
      - "frontend/**"
      - ".github/workflows/ci-frontend.yml"
```

Or split workflows entirely by area. The trade-off: separate workflows means separate status checks in branch protection. The `ci-gate` pattern smooths this.

### Skip CI on draft PRs

```yaml
jobs:
  test-backend:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
```

Saves runner minutes while a developer is iterating. They'll mark the PR ready for review when they want CI to run.

### Concurrency cancellation

Already shown above. The single most impactful one-liner for PR-heavy workflows — every superseded push frees a runner immediately rather than burning it on stale code.

### Docker BuildKit layer caching

`cache-from: type=gha` plus `cache-to: type=gha,mode=max` makes BuildKit push intermediate layers to GitHub's Actions cache backend. Subsequent runs reuse layers whose inputs haven't changed. Pair with multi-stage Dockerfiles (Day 5) where `COPY requirements.txt` precedes `COPY .`, so dependency installation caches but code changes don't invalidate it.

### Setup-action built-in caches

`actions/setup-python` with `cache: pip`, `setup-node` with `cache: pnpm`, `setup-java` with `cache: maven` all take care of dependency caching with one line — easier than rolling `actions/cache` by hand. Use these by default.

### Test sharding

For test suites that have outgrown a single runner, split tests across N parallel runners:

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: pytest --splits 4 --group ${{ matrix.shard }}  # pytest-split plugin
```

For the rev-eval-ai-pep substrate this is overkill. Worth knowing exists for the day a service's test suite grows past 2-3 minutes.

## Common Pitfalls

- **Optimizing the wrong step.** A 30-second saving on a 5-second step is invisible. Profile first. Always.
- **Caching the wrong thing.** Caching `node_modules` directly (rather than pnpm's content-addressed store) is fragile across pnpm versions. Cache the package-manager store, not the resolved tree.
- **Cache key collisions across services.** Two services with cache key `pip-${{ hashFiles('**/requirements.txt') }}` end up with one cache that's right for one and wrong for the other. Always scope by service in the key.
- **Aggressive concurrency cancellation on `main`.** Cancelling a `main` build because someone pushed a second commit can lose deployment artifacts. Only cancel on PR branches.
- **Self-hosted runners as a first move.** Self-hosted runners add ops overhead, security boundaries, and capacity-planning headaches. Exhaust GitHub-hosted optimization first. They become correct around the point where build minutes cost matters more than runner setup cost.
- **Premature parallelism.** A 12-way matrix that spends 90% of its time on runner startup and dependency install isn't faster than a 4-way matrix that pays those costs four times instead of twelve. There is a sweet spot.

## Key Takeaways

- Measure before you optimize: use the run/job/step timing in the Actions UI to find the actual hot spots.
- Apply optimizations in order of impact: parallelize, then cache, then skip, then shrink, then upgrade runners.
- The combination of matrix-parallel jobs + setup-action dependency caching + Docker BuildKit layer cache (`type=gha`) is usually a 4-5x win against an unoptimized pipeline.
- `concurrency: cancel-in-progress` on PR branches is one line that frequently doubles effective runner throughput.
- Path filters, draft-PR skips, and test sharding are surgical tools — apply when targeted measurement says they help.
- A 4-minute pipeline is a tool the team uses. An 18-minute pipeline is a thing the team avoids. Optimization is a developer-experience investment, not a vanity metric.

---

*Prerequisites: All of Day 7 Topics 1-6 (this synthesizes parallelism, matrix, caching, artifacts, scanning, and linting into one optimized whole), Day 3 (caching strategies), Day 5 (multi-stage Dockerfiles whose layer cache you exploit here).*
