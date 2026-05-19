# Extending CI Workflows For Additional Tests

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D10 scaffolded the reporting service and *added a CI job* for it — but at the time the service had two stub endpoints and a smoke test. Today reporting grows real query logic (Topic 2), real read endpoints (Topic 3), and real unit tests (Topic 8). The CI job has to grow with it: more tests to run, coverage for the new code paths, a Postgres service container so the integration-style tests have a database to talk to, and a fail-fast posture so a broken reporting query doesn't ship hidden behind a green checkmark. D7 covered the foundational CI patterns (matrix jobs, caching, parallelism); today is the targeted, incremental "add tests to an existing pipeline" task that the cohort will repeat dozens of times in their careers.

## What CI Looks Like For Reporting Today (Before Day 16)

The D10 scaffold added a job that looks roughly like this:

```yaml
# .github/workflows/ci.yml (excerpt)
reporting-service:
  name: reporting-service / lint+test
  runs-on: ubuntu-latest
  defaults:
    run:
      working-directory: services/reporting-service
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: "3.12"
        cache: pip
    - run: pip install -r requirements.txt -r requirements-dev.txt
    - run: ruff check .
    - run: pytest -q
```

The test it ran was the D10 smoke: "the app starts and `/health` returns 200." That's not a meaningful guardrail against today's changes; if a SQLAlchemy query has a typo, the smoke test still passes.

## What Today Adds

Three additions:

1. **A real test database** spun up as a GitHub Actions service container, so tests that exercise the query layer have somewhere to query against.
2. **Alembic migrations applied** in CI before tests run, so the schema is real.
3. **A coverage threshold** with a sensible gate, and clear test categorization (unit vs. integration) so the cohort knows what's running where.

### Step 1: Add A Postgres Service Container

GitHub Actions has first-class service containers. Add Postgres to the reporting-service job:

```yaml
reporting-service:
  name: reporting-service / lint+test
  runs-on: ubuntu-latest
  defaults:
    run:
      working-directory: services/reporting-service
  services:
    postgres:
      image: postgres:16-alpine
      env:
        POSTGRES_USER: reporting
        POSTGRES_PASSWORD: reporting
        POSTGRES_DB: reporting_test
      ports:
        - 5432:5432
      options: >-
        --health-cmd "pg_isready -U reporting"
        --health-interval 5s
        --health-timeout 5s
        --health-retries 10
  env:
    DATABASE_URL: postgresql+asyncpg://reporting:reporting@localhost:5432/reporting_test
    TEST_MGMT_DATABASE_URL: postgresql+asyncpg://reporting:reporting@localhost:5432/test_mgmt_test
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: "3.12"
        cache: pip
    - run: pip install -r requirements.txt -r requirements-dev.txt
    - run: ruff check .
    - name: Create test_mgmt_test database
      run: |
        PGPASSWORD=reporting psql -h localhost -U reporting -d postgres \
          -c "CREATE DATABASE test_mgmt_test;"
    - name: Apply migrations
      run: |
        alembic upgrade head                                    # reporting_db
        ALEMBIC_DB=$TEST_MGMT_DATABASE_URL \
          alembic -c alembic.test-mgmt.ini upgrade head         # test_mgmt_db schema
    - name: Run tests
      run: pytest -q --cov=app --cov-report=term-missing --cov-fail-under=70
```

Several pieces worth naming:

- **`services.postgres`** is the GitHub Actions service-container syntax. It runs alongside the job's runner and is reachable on `localhost`. The `health-cmd` makes the job wait for Postgres to be ready before running tests, eliminating a class of flaky-startup failures.
- **Two test databases.** Reporting owns `reporting_test`; it also needs `test_mgmt_test` (the cross-DB read pattern from Topic 4). Create the second DB explicitly with `psql`. In production-like Compose, both databases are on the same Postgres instance with different names — CI mirrors that.
- **Apply migrations explicitly.** Alembic's `upgrade head` runs against both databases. The cohort needs a separate `alembic.test-mgmt.ini` or env-var-driven config to point Alembic at the test-mgmt DB; the D10 Alembic setup made this clean.
- **Coverage threshold.** `--cov-fail-under=70` makes the build fail if coverage dips below 70%. The number is a starting point — the cohort can argue it up or down in retro — but having *a* number prevents silent erosion.

### Step 2: Categorize Tests

`pytest` can be told which tests are unit vs. integration via markers. In `pytest.ini`:

```ini
[pytest]
asyncio_mode = auto
markers =
    unit: pure-Python tests, no DB
    integration: tests that touch the DB
```

Tests opt in:

```python
import pytest

@pytest.mark.unit
def test_response_envelope_shape():
    ...

@pytest.mark.integration
async def test_list_attempts_filters_by_user(db_session, seeded_attempts):
    ...
```

CI can then split:

```yaml
- name: Unit tests (fast)
  run: pytest -q -m unit
- name: Integration tests (DB)
  run: pytest -q -m integration --cov=app --cov-fail-under=70
```

The benefit is twofold: failures categorize themselves (a unit-test failure is application logic; an integration-test failure is DB or query-layer), and local developers can run `pytest -m unit` for fast feedback during development without spinning up Postgres.

### Step 3: Don't Forget The Other Workflows

The cohort's first instinct is to add the test command and stop. But CI usually has *several* workflows: PR validation, main-branch builds, nightly runs. Audit:

- **PR workflow** (`pr.yml` or main `ci.yml`) — needs the new tests.
- **Main-branch build / deploy workflow** (`deploy.yml`) — should also run the new tests before deploy. A test that doesn't run on the deploy path is a test that doesn't gate the deploy.
- **Nightly E2E** (the D15 Playwright job) — independent of the unit/integration changes but should keep passing.

A frequent failure mode: PR tests added; deploy tests forgotten; broken main-branch code deploys because deploy CI never ran the new tests. The cohort should grep for `pytest` across `.github/workflows/` and confirm every relevant invocation is updated.

## Caching For Speed

The D10 job had `cache: pip` — the simple version. As tests grow, the slower bits are:

- **pip install** — already cached via `setup-python`'s built-in.
- **Postgres service container startup** — about 5 seconds; not worth optimizing.
- **Migrations** — fast for now; on D18 the trainer-dashboard schema additions might add 100ms.

The 80/20 of CI speed for reporting is: don't reinstall Python packages on every run. The pip cache handles that. Don't over-engineer.

## Failing Fast And Loudly

One discipline the cohort should adopt today: `--maxfail=1` on integration tests during heavy development:

```yaml
- run: pytest -q -m integration --maxfail=1 -x
```

When fifty tests share one DB fixture, the first failure often cascades into 49 noisy "could not connect" or "row not found" errors that mask the actual problem. Stopping at the first failure gives a clean error to read. Drop the flag in main-branch CI to see *all* failures; keep it in feature-branch CI where the cohort is iterating fast.

## Test Output Discipline

The default `pytest` output is fine; what often *isn't* fine is what happens when an assertion fails on a paginated response: you see `assert items[0] == ...` and the actual `items[0]` is 200 lines of JSON. Two things help:

- **`pytest -v`** prints test names but truncates assertion output by default.
- **`pytest --tb=short`** prints shorter tracebacks.
- **Custom `__repr__`** on Pydantic models keeps assertion output readable.

For CI, `pytest -q --tb=short` gives compact, scannable output. For local dev, `pytest -v` gives more.

## Reviewing The Workflow As A Cohort

Once today's CI changes are in place, the trainer should pull up the green build and walk the cohort through it:

- What were the steps?
- Which step was the slowest? (Probably pip install on a cold cache.)
- Which test category caught the bug we seeded in `trainer/reference`?
- What would have happened if a typo broke a SQL `GROUP BY` clause? (Would the test catch it?)

The point: CI isn't infrastructure that runs in the background. It's *evidence* that the change is safe to merge. The cohort should read CI output as actively as they read application logs.

## Branch Protection As The Final Step

Adding tests to CI does nothing if `main` doesn't *require* them to pass. Confirm the GitHub branch protection rule on `main` includes the `reporting-service / lint+test` check as required. If it doesn't, a PR can merge with a red check.

The trainer demonstrates this once: open the branch protection settings, show the required-checks list, add the new job, save. The cohort should know how to do this themselves; "we have tests, but they don't gate merges" is a common dysfunction that's worth seeing the fix for live.

## A Note On Local CI Reproducibility

Tests that pass in CI but fail locally (or vice versa) are infuriating. The discipline that prevents this:

- **CI's Python version, Postgres version, and `pip install` should be reproducible locally.** The cohort already has `docker compose` with Postgres; using the same image and version eliminates environment drift.
- **`conftest.py` should NOT depend on host-machine state.** No reading from `/tmp`, no env vars not set in CI, no assumed timezone other than UTC.
- **Random seeds.** Tests using random data (which they shouldn't, but) should seed deterministically.

The cohort should run `pytest -q -m integration` against a local Compose Postgres before pushing, and trust that CI will agree. When they diverge, fix the divergence; don't paper over it with retries.

## Anti-Patterns

- **"It works locally; the CI failure is flaky."** Sometimes true (network-dependent test). Usually it's a real bug or an environmental dependency the cohort hasn't noticed. Investigate before re-running.
- **Skipping tests in CI to make it green.** A `@pytest.mark.skip` with no follow-up issue is permanent debt. If a test is broken, fix it or delete it; don't `skip` it.
- **Tests with no DB cleanup between runs.** Postgres state leaking across tests produces order-dependent failures that are nightmares to debug. Use a fixture that rolls back after each test (Topic 8 covers this).
- **Service-container health-checks omitted.** Without `--health-cmd`, the job sometimes runs `pytest` before Postgres is ready and fails. Always include the health check.
- **Hardcoded `localhost:5432` in test config.** Works in CI, breaks if a local dev has a different port mapping. Read from `DATABASE_URL`.
- **No coverage gate.** Coverage erodes the moment nobody's watching. Even a low gate (50%) catches the case where someone adds 200 lines of untested code.
- **CI changes that aren't tested in their own PR.** A workflow change that doesn't actually run produces a green check that means nothing. Push the workflow change, watch CI run it, confirm the new tests execute, then merge.
- **Forgetting the deploy pipeline.** Test added to `ci.yml` but not `deploy.yml` means main can break the moment it merges. Audit both.

## Key Takeaways

- Adding a new service's tests to CI is a four-part move: provision dependencies (Postgres service container), apply migrations, run the tests, and gate the build on coverage.
- Categorize tests via markers (`unit`, `integration`) so local dev can run fast subsets and CI can split parallelizable work later.
- Update *every* relevant workflow — PR validation and deploy — or main-branch failures slip through.
- Branch protection's required-checks list is the actual gate; adding a CI job without making it required changes nothing.
- Local-vs-CI reproducibility is a discipline: same Python, same Postgres, same fixtures, no host-state dependencies.

---
*Prerequisites: day-7-robust-ci-pipelines-failure-modes-and-mitigation, day-10-extending-ci-workflows-for-new-tests-and-services, day-16-unit-testing-patterns-for-read-endpoints.*
