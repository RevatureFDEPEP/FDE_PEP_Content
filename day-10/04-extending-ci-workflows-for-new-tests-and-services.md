# Extending CI Workflows for New Tests and Services

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
A new service that isn't in CI is a new service nobody trusts. Today we extend the Day 7 pipeline so the scaffolded `reporting-and-analytics-service` gets the same build/lint/test treatment as every other service — without slowing down the pipeline for changes that don't touch it. The pattern is a **service matrix** (one job runs per service, in parallel) combined with **path filters** (a service's job only runs if files in its directory changed). Once this is in place, adding the next service is a two-line YAML change.

## Where We're Starting

After Day 7, `.github/workflows/ci.yml` runs a per-service matrix for the existing services. Roughly:

```yaml
name: ci
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  python-services:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service:
          - user-service
          - question-management-service
          - test-management-service
          - api-gateway
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
          cache-dependency-path: services/${{ matrix.service }}/pyproject.toml
      - name: Install
        working-directory: services/${{ matrix.service }}
        run: |
          pip install --upgrade pip
          pip install -e .[dev]
      - name: Lint
        working-directory: services/${{ matrix.service }}
        run: ruff check .
      - name: Test
        working-directory: services/${{ matrix.service }}
        run: pytest -q

  web:
    runs-on: ubuntu-latest
    steps:
      # pnpm install, lint, test, build
      ...
```

Two extensions today: add reporting to the matrix, and add path filters so jobs skip cleanly.

## Extension 1: Add Reporting to the Matrix

Single-line change:

```yaml
        service:
          - user-service
          - question-management-service
          - test-management-service
          - api-gateway
          - reporting-and-analytics-service   # ← new
```

That's the whole edit. Because every service follows the scaffolding convention from [01-fastapi-service-scaffolding-conventions.md](01-fastapi-service-scaffolding-conventions.md), the same steps work:

- `pyproject.toml` at the service root → pip cache key works.
- `pip install -e .[dev]` resolves dev deps.
- `ruff check .` and `pytest -q` run from the service directory.
- The placeholder `test_health.py` ensures the matrix entry is not green-by-vacuity.

Push the change, watch a new column appear in the PR check list.

## Extension 2: Path Filters So Jobs Skip Cleanly

A frontend-only change shouldn't run all five backend test suites. Two common approaches.

### Approach A: Workflow-Level `paths`

Coarse but simple. Split the pipeline into multiple workflows, each scoped to a path:

```yaml
# .github/workflows/ci-backend.yml
on:
  pull_request:
    paths:
      - 'services/**'
      - '.github/workflows/ci-backend.yml'
  push:
    branches: [main]
    paths:
      - 'services/**'
      - '.github/workflows/ci-backend.yml'
```

```yaml
# .github/workflows/ci-frontend.yml
on:
  pull_request:
    paths:
      - 'web/**'
      - '.github/workflows/ci-frontend.yml'
```

Pros: simple, GitHub handles the skip natively.
Cons: required status checks need to be configured to *not* require these checks when the path doesn't match — otherwise PRs get stuck "expected to run" forever. Use the `paths-ignore` inverse carefully.

### Approach B: Path Filter Action with Matrix (Recommended)

Use `dorny/paths-filter` to compute which services changed, then run the matrix only for those:

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            user-service:
              - 'services/user-service/**'
            question-management-service:
              - 'services/question-management-service/**'
            test-management-service:
              - 'services/test-management-service/**'
            api-gateway:
              - 'services/api-gateway/**'
            reporting-and-analytics-service:
              - 'services/reporting-and-analytics-service/**'

  python-services:
    needs: changes
    if: needs.changes.outputs.services != '[]'
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service: ${{ fromJSON(needs.changes.outputs.services) }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
          cache-dependency-path: services/${{ matrix.service }}/pyproject.toml
      - name: Install
        working-directory: services/${{ matrix.service }}
        run: |
          pip install --upgrade pip
          pip install -e .[dev]
      - name: Lint
        working-directory: services/${{ matrix.service }}
        run: ruff check .
      - name: Test
        working-directory: services/${{ matrix.service }}
        run: pytest -q
```

Pros: only changed services run; matrix is dynamic; status check works naturally because the job either runs and passes/fails or is `skipped` (which counts as success for branch protection).
Cons: small added complexity in the workflow.

### Shared Code: A "Common" Filter

If you have a shared internal package or a workflow file the matrix depends on, *every* service must run when those change:

```yaml
            shared:
              - '.github/workflows/ci.yml'
              - 'services/_shared/**'
```

Then change the matrix expression to "all services if `shared` changed, otherwise the filtered list" — or just make `shared` set an output that the next job ANDs into the decision.

## Extension 3: Database for Reporting Tests

The reporting service uses Postgres. Tests against a real DB need a service container:

```yaml
  reporting-tests:
    runs-on: ubuntu-latest
    needs: changes
    if: contains(needs.changes.outputs.services, 'reporting-and-analytics-service')
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: reporting
          POSTGRES_PASSWORD: reporting
          POSTGRES_DB: reporting
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U reporting -d reporting"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
    env:
      REPORTING_DATABASE_URL: postgresql+asyncpg://reporting:reporting@localhost:5432/reporting
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Install
        working-directory: services/reporting-and-analytics-service
        run: pip install -e .[dev]
      - name: Run migrations
        working-directory: services/reporting-and-analytics-service
        run: alembic upgrade head
      - name: Test
        working-directory: services/reporting-and-analytics-service
        run: pytest -q
```

This job pairs nicely with the migration discipline from [02-alembic-for-relational-schema-evolution.md](02-alembic-for-relational-schema-evolution.md) — `alembic upgrade head` runs against a real Postgres on every CI run, catching broken migrations before merge.

For today the placeholder test doesn't need a DB, so this job is optional. Add it when the first real model lands in Week 4.

## Caching: Don't Skip This

Pip caching makes the new matrix entry add seconds, not minutes:

```yaml
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
          cache-dependency-path: services/${{ matrix.service }}/pyproject.toml
```

The `cache-dependency-path` per-service means each service has its own cache key — adding a dep to user-service doesn't invalidate reporting's cache. See [04-build-caching-strategies-in-ci.md](../day-03/04-build-caching-strategies-in-ci.md).

## Required Status Checks

Don't forget the branch protection update:

1. Settings → Branches → main → Edit rule.
2. "Require status checks to pass before merging" — add the new check name.
3. If using path filters, set checks as **not required** that may legitimately be skipped (or use "checks marked as skipped count as passed" semantics — already the default in GitHub branch protection rules in 2024+).

A common bug: new check added, branch protection requires it, but `paths` filter prevents it from running on a frontend-only PR → PR is permanently un-mergeable. Fix by switching from `paths`-on-workflow to the matrix filter approach above, where the job runs and reports `skipped`.

## Verifying the Extension Locally

Before pushing:

```bash
# Lint and test the new service the same way CI will
cd services/reporting-and-analytics-service
pip install -e .[dev]
ruff check .
pytest -q

# Verify the workflow YAML is valid
yamllint .github/workflows/ci.yml
# Or use the GitHub CLI:
gh workflow view ci.yml
```

Push to a draft PR; check that:

- The reporting matrix entry appears.
- For a PR touching only `services/reporting-and-analytics-service/`, only that matrix entry runs.
- For a PR touching only `web/`, no backend matrix entries run.

## Anti-Patterns

- **Adding a service to the matrix without verifying it can be linted/tested standalone.** The CI will surface the issue, but a quick local run saves a red CI cycle.
- **Hardcoding the service list in two places.** Keep the source-of-truth single (the filter block) and derive the matrix from its output.
- **Forgetting branch protection.** New check exists, isn't required, fails are ignored, broken code merges.
- **Globbing `services/**` for everything.** Defeats the point of path filters; every PR runs every service.
- **No cache key per service.** One service's dep bump invalidates everyone's cache.
- **Running CI against the Compose stack.** Compose is for integration smoke; CI uses ephemeral service containers per job. Mixing them slows everything down.

## Key Takeaways
- A scaffolded service in the matrix adds one YAML line; the convention does the rest.
- Use `dorny/paths-filter` + dynamic matrix to skip unchanged services without breaking required-check semantics.
- Service containers in CI (`services:` block) replace local Compose for jobs that need a real DB.
- Cache per-service via `cache-dependency-path` so one service's dep change doesn't invalidate others.
- Update branch protection when you add a required check; verify skipped-check behavior with a draft PR.

---
*Prerequisites: day-3 GitHub Actions topics, day-7 mature CI topics, [01-fastapi-service-scaffolding-conventions.md](01-fastapi-service-scaffolding-conventions.md).*
