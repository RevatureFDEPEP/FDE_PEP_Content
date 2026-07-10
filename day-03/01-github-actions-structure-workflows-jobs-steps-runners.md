# GitHub Actions Structure — Workflows, Jobs, Steps, Runners

> *Day 3: Diagnose the CI Pipeline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Today's deliverable is to diagnose five seeded bugs in `.github/workflows/ci-pipeline.yml`. Before you can localize a bug, you have to read a workflow file fluently — knowing which element runs where, in what order, and on which machine. This topic gives you the mental model so that when a step fails on Day 3, you can immediately tell whether the failure is in workflow plumbing, job orchestration, step configuration, or the underlying command.

## The four-layer mental model

A GitHub Actions pipeline is a strict hierarchy:

1. **Workflow** — a single YAML file under `.github/workflows/`. It is the top-level unit of automation, scoped to one repository, triggered by `on:` events.
2. **Job** — a named unit of work inside a workflow. Each job runs on its own fresh runner. Jobs are parallel by default; sequential ordering requires explicit `needs:`.
3. **Step** — an ordered command within a job. Steps share the runner's filesystem and environment within their job, but **nothing** crosses a job boundary unless you upload an artifact or pass an output.
4. **Runner** — the VM (or container) executing one job. `ubuntu-latest`, `windows-latest`, self-hosted. Each job gets a clean runner.

Internalize the boundaries: **steps share state, jobs do not**. Most "why didn't my file from the previous step exist?" bugs are really "I expected job-level state, but the file was written in a different job."

## Anatomy of a workflow file

```yaml
name: CI Pipeline                      # workflow name (shows in Actions UI)

on:                                    # trigger configuration
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:                                   # workflow-level env vars (all jobs see these)
  NODE_VERSION: '20'
  PYTHON_VERSION: '3.11'

jobs:                                  # one or more jobs
  lint-frontend:                       # job ID (referenced by `needs:`)
    name: Lint Next.js frontend        # display name
    runs-on: ubuntu-latest             # runner target

    steps:                             # ordered list of steps
      - uses: actions/checkout@v4      # step using a published action
      - name: Install pnpm             # step with a name + run
        run: npm install -g pnpm@9
      - name: Lint
        run: pnpm --filter frontend lint

  test-backend:
    name: Test Python services
    runs-on: ubuntu-latest
    needs: lint-frontend               # waits for lint-frontend to succeed
    steps:
      - uses: actions/checkout@v4
      - run: pytest services/user-service/tests
```

### Reading order on a failure

When you open a failed run, navigate top-down:

1. **Which workflow ran?** (`name:` at the top — useful if a repo has multiple workflows.)
2. **Which trigger fired it?** (Push to `main`? Pull request? The `on:` block tells you what conditions ran this.)
3. **Which jobs ran, and in what order?** The dependency graph from `needs:` declarations.
4. **Which job failed?** Open that one. Each job is its own log scope.
5. **Which step inside that job failed?** Steps are numbered; the failing one has the red X.
6. **What did the step actually run?** A `uses:` reference (third-party action) or a `run:` (shell command).

Two of the seeded bugs in `ci-pipeline.yml` are misdiagnosed if you stop at "the test step failed" — the real cause is upstream (wrong job dependency, missing env propagation). Always trace back from the failing step through the job and workflow context.

## Triggers and the `on:` block

The `on:` block decides whether a workflow runs at all. A workflow that never runs is the most invisible bug class.

```yaml
on:
  push:
    branches: [main]                   # only pushes to main
    paths:                             # AND only when these paths change
      - 'services/**'
      - '.github/workflows/**'
  pull_request:
    types: [opened, synchronize, reopened]
  workflow_dispatch:                   # manual trigger from the UI
```

Common confusion: `branches:` on a `pull_request:` trigger refers to the **target** branch (where the PR is merging *to*), not the head branch. If you set `branches: [develop]`, PRs targeting `main` will silently skip the workflow.

## Runners and the execution environment

`runs-on: ubuntu-latest` gives you a clean VM with a known toolchain baseline:
- Git, curl, jq, Docker, Python 3.x, Node 20+ pre-installed.
- A non-root user with sudo.
- ~14 GB free disk, ~7 GB RAM (subject to change).

Implications for your workflow:
- The VM is **destroyed after the job**. No persistent state.
- Anything you `apt-get install` lives only for that job.
- Files in `$GITHUB_WORKSPACE` are gone when the job ends — that's why caching and artifacts exist.

Self-hosted runners change all of these guarantees; the rev-eval-ai-pep pipeline uses GitHub-hosted `ubuntu-latest` runners, so for this cohort treat the runner as ephemeral.

## Job dependencies and concurrency

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    needs: build                       # test waits for build to succeed
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: [build, test]               # waits for both
    if: github.ref == 'refs/heads/main'  # conditional execution
    runs-on: ubuntu-latest
    steps: [...]
```

Three things to notice when reading a workflow:

- **`needs:` defines the DAG.** Without `needs:`, jobs run in parallel. With it, downstream jobs wait *and* are skipped if upstream fails (unless you use `if: always()`).
- **`if:` is evaluated at job start.** Common bug: `if: github.event_name == 'push'` written as `'Push'` — string comparison is case-sensitive, so the job silently skips.
- **Each job is isolated.** If `build` produces a `dist/` directory, `test` will not see it unless `build` uploads it via `actions/upload-artifact` and `test` downloads it.

## Example / Worked Scenario

Here's a realistic excerpt from `ci-pipeline.yml` in the rev-eval-ai-pep brownfield repo:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install -g pnpm@9
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  test-services:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        service: [user-service, question-management-service, test-management-service]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r services/${{ matrix.service }}/requirements.txt
      - run: pytest services/${{ matrix.service }}/tests

  build-frontend:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t frontend:ci ./apps/frontend
```

Reading this top-down:

- One workflow, three jobs: `lint`, `test-services`, `build-frontend`.
- `lint` runs first. `test-services` and `build-frontend` run in parallel after `lint` succeeds.
- `test-services` uses a **matrix strategy** — it expands into three parallel job instances, one per service. Three checkmarks (or X's) in the UI, not one.
- If `lint` fails, both downstream jobs are skipped (shown as gray "skipped" in the UI). That's not a bug — that's `needs:` doing its job.

Now suppose the UI shows: `lint` green, `test-services (user-service)` red, the other two matrix entries green. Where do you click? Into the `user-service` matrix entry, and inside that, find the failing step. Don't open the `lint` job — it succeeded. Don't open `build-frontend` — it's a sibling, unrelated.

## Common Pitfalls

- **Confusing `name:` with the job ID.** `name:` is a display string. The job ID (the YAML key) is what `needs:` references. Renaming the display name doesn't break anything; renaming the job ID breaks every `needs:` that referenced it.
- **Assuming filesystem state crosses jobs.** A step in job A writes `dist/`. A step in job B reads `dist/`. It will fail every time — jobs run on different runners. Use `actions/upload-artifact` / `download-artifact`.
- **Misreading the failure scope.** A red workflow ≠ a red step. A workflow fails because a job failed; a job fails because a step failed. Always drill in.
- **Forgetting `if:` evaluates expressions.** `if: ${{ github.ref == 'refs/heads/main' }}` and `if: github.ref == 'refs/heads/main'` are both valid, but typos inside the expression (single equals, wrong context name) silently evaluate to `false` and skip the job with no error.

## Key Takeaways

- A workflow file has a strict four-layer hierarchy: workflow → jobs → steps → runner. Read top-down, drill in on failure.
- Jobs are isolated VMs. Steps share state within a job; jobs don't share state across each other.
- `needs:` defines execution order and skip-on-failure behavior. Without it, jobs run in parallel.
- When diagnosing a failure, identify which **step** failed inside which **job** inside which **workflow run** before forming any hypothesis.

---
*Prerequisites: [05-codebase-navigation-conventions.md](../day-01/05-codebase-navigation-conventions.md), [03-microservices-architecture-and-service-boundaries.md](../day-01/03-microservices-architecture-and-service-boundaries.md).*
