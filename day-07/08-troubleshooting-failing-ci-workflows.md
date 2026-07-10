# Troubleshooting Failing CI Workflows

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

On Day 3 you triaged a *linear* CI failure: one job, top-to-bottom, the bug was somewhere in the log. That worked because the failure mode was simple — a misconfigured action, a typo in a step, a missing dependency. The pipeline you have now is different. It runs jobs in parallel. It fans out into matrix legs. It hands artifacts between jobs. It scans images. It gates on coverage. The failure surface is much larger.

The mental shift for today's topic: **a failure in a mature pipeline is a tree, not a line.** A single PR shows 8 status checks. Five are green, three are red. The skill is reading the *shape* of the failure across jobs to localize the root cause — not staring at a single log waiting for inspiration.

## The diagnostic posture

Five questions, in order, every time:

1. **Is the failure deterministic?** Re-run the failing job. If it now passes with no code change, you have a flake (covered below). If it consistently fails, it's a real issue.
2. **Does the failure correlate with the diff?** Look at the PR's file changes. A failing `lint-python` on a PR that only touched the frontend is almost always a pre-existing problem in `main`, not something this PR introduced.
3. **Which jobs failed and which passed?** A single matrix leg failing while siblings pass is a different problem from all legs failing. The pattern is a strong hypothesis source.
4. **Is the failure in code under test or in CI infrastructure?** "pytest reported 3 assertion failures" is the former. "Cannot connect to registry-1.docker.io" is the latter.
5. **What changed in CI itself recently?** Sometimes the answer is a workflow YAML change, not an application change. `git log --oneline .github/workflows/` is your friend.

## Reading the failure tree

The Actions UI shows the workflow as a graph: nodes are jobs, edges are `needs:` dependencies. A failure tree looks like this:

```
checkout ───┬─ lint-python (✓)
            ├─ lint-frontend (✓)
            ├─ test-backend (matrix)
            │     ├─ user-service              (✓)
            │     ├─ question-management-service (✗)  ← root cause likely here
            │     └─ test-management-service   (✓)
            ├─ test-frontend (✓)
            └─ build-image (matrix)
                  ├─ user-service              (skipped — needs failed)
                  ├─ question-management-service (skipped)
                  ├─ test-management-service   (skipped)
                  └─ frontend                  (skipped)
```

In that picture, the single failing leaf is the only thing you need to look at. The downstream `build-image` jobs were *skipped*, not failed — `needs:` saw an upstream failure and short-circuited. Don't waste time reading their logs.

Compare with:

```
test-backend (matrix)
  ├─ user-service              (✗ — connection refused to postgres:5432)
  ├─ question-management-service (✗ — connection refused to postgres:5432)
  └─ test-management-service   (✗ — connection refused to postgres:5432)
```

Three matrix legs failing with the **same error message** points squarely at shared infrastructure — the `services:` block for Postgres, network configuration, or the action that wires it. Not at any individual service's code. The pattern is the diagnosis.

## Common parallel-CI failure patterns

### Pattern: one matrix leg fails, siblings pass

**Diagnosis:** the problem is leg-specific. Look at what makes that leg different — its `working-directory`, its specific dependencies, a service-specific test that runs only there.

**Example:** `test-backend / question-management-service` fails with `pymongo.errors.ServerSelectionTimeoutError` while sibling Python services pass. The qm service is the only one that needs Mongo. Either the Mongo service block in the workflow is mis-wired or the qm service's test is connecting to the wrong host.

### Pattern: all matrix legs fail with the same error

**Diagnosis:** shared infrastructure or shared base config. The matrix isn't the problem; whatever the matrix shares is.

**Example:** All four `build-image` legs fail with `failed to compute cache key`. Almost certainly the Docker setup action, the buildx config, or the GHA cache backend is wrong — not any individual Dockerfile.

### Pattern: lint-X passes, test-X fails on the same diff

**Diagnosis:** lint is static; tests are dynamic. The failure is about runtime behavior the linter cannot see — wrong argument types, async/await mistakes, mock fixtures that don't match real signatures.

### Pattern: scan job fails on a PR that doesn't touch the Dockerfile

**Diagnosis:** a new CVE was published against a dependency you already had. The "what changed" is not your code; it's the world. Bump the dependency or add a justified `.trivyignore` entry.

### Pattern: gate job fails but every leaf passed

**Diagnosis:** the gate's `needs:` list includes a job that was skipped due to `if:`. A skipped job is not the same as a passed job — the gate may be misinterpreting it. Use `needs.X.result == 'success' || needs.X.result == 'skipped'` if a skip is acceptable.

### Pattern: works locally, fails in CI

**Diagnosis (in order of probability):** (a) you have local state — uncommitted files, a `.env`, a globally-installed tool — that CI doesn't. (b) Filesystem case-sensitivity: macOS is case-insensitive, ubuntu-latest is case-sensitive, so `import User from './user/User'` works locally and fails in CI. (c) Dependency versions differ; check lockfiles are committed and used.

## Flaky tests — the productivity destroyer

A **flaky** test fails sometimes and passes sometimes on the same code. Flakes erode trust in CI more than any other single thing. A failing build that "you can just re-run" trains the team to re-run on every red status, which means real failures get re-run-and-merged.

Common flake sources:

- **Timing assumptions** — `time.sleep(0.1)` followed by "the queue should be drained by now" is wrong on a slow runner.
- **Test ordering coupling** — Test A writes a row Test B reads; pytest's randomized order eventually surfaces it.
- **Shared external state** — Tests against a shared dev DB. Run them on each job's *own* ephemeral DB instead.
- **Network calls without mocking** — A test that hits a real API endpoint is at the mercy of that endpoint's uptime.
- **Race conditions in the code under test** — the test is fine, the code is genuinely racy. The flaky test caught a real bug; don't shoot the messenger.

**Triage:**

1. **Quarantine immediately.** Mark with `@pytest.mark.flaky` or `.skip` and open a ticket. Do not let a known flake gate every merge.
2. **Run it 100 times in isolation** to estimate the flake rate. `pytest --count=100 tests/test_thing.py::test_flake` (pytest-repeat).
3. **Reproduce locally with the same conditions** — same test ordering (`pytest -p no:randomly --order=...`), same env vars, same timing pressure (`stress -c 4` to load the CPU).
4. **Fix the root cause.** Replace timing assumptions with explicit synchronization. Isolate state. Mock external calls. Add proper locking.

A flake fix is a real PR with a test that *proves* it was deterministic flake (e.g., a regression test that ran 100x in CI before the fix to demonstrate the original failure rate).

## Worked Scenario — a tree of failures, only one root cause

A PR adds a new endpoint to `question-management-service` and bumps `pydantic` from 2.5.0 to 2.7.0. CI shows:

```
lint-python (matrix)
  ├─ user-service                (✓)
  ├─ question-management-service (✗ — ruff RUF012 on a new model)
  └─ test-management-service     (✓)

test-backend (matrix)
  ├─ user-service                (✗ — pytest collection error: ImportError)
  ├─ question-management-service (✗ — pytest collection error: ImportError)
  └─ test-management-service     (✗ — pytest collection error: ImportError)

build-image (matrix)
  ├─ user-service                (skipped)
  ├─ question-management-service (skipped)
  ├─ test-management-service     (skipped)
  └─ frontend                    (✓)
```

Step through:

1. **`lint-python / question-management-service`** failed on a Ruff rule on a new file. That's leg-specific and PR-introduced. Real issue, fix in the PR.
2. **`test-backend` — all three legs** failed with `ImportError` during collection. Same error across the matrix means shared infrastructure. Click into the log: `ImportError: cannot import name 'BaseSettings' from 'pydantic'`.
3. **Diagnosis:** `BaseSettings` moved to `pydantic-settings` in pydantic 2.x. The PR bumped pydantic to 2.7.0 in `question-management-service` only, but all three services share a `requirements.txt` constraint, and the shared base requirements file resolves to 2.7.0 for all three. Two services (`user-service`, `test-management-service`) still import `from pydantic import BaseSettings` — works on 2.5, breaks on 2.7.
4. **Fix:** either revert the pydantic bump, or update all three services' imports to `pydantic_settings.BaseSettings` and add `pydantic-settings` to each `requirements.txt`. The lint failure becomes secondary cleanup.

The reading skill: three "identical" failures aren't three problems, they're one. The build-image skips are noise — they didn't fail, they didn't run. Focus narrows to two issues: one PR-local lint, one shared dependency upgrade.

## Tools that help

- **`gh run view <id> --log-failed`** — print only logs from failed steps. Drastically reduces what you scroll through.
- **`gh run rerun <id> --failed`** — re-run only the failed jobs (faster than the whole workflow). Use for flake hypothesis-testing.
- **`act`** — run GitHub Actions workflows locally in Docker. Imperfect (some actions don't work) but excellent for iterating on workflow YAML without push-wait-fail cycles.
- **`ACTIONS_STEP_DEBUG=true`** as a repo secret — enables verbose step-level debug logging. Use sparingly; the output is voluminous.
- **`tmate` action** — drops an SSH session into the runner mid-job for live debugging. Powerful but auditable; document its use.

## Common Pitfalls

- **Re-running until green.** If a job passes on the second try with no change, you have a flake, not a fix. Treat every "fixed by re-run" as a bug to investigate.
- **Reading the wrong log.** When 5 jobs fail, looking at the log for `build-image` (downstream skip) wastes time. Start at the *upstream* failures.
- **Local-only debugging.** If your hypothesis is "this is a CI environment difference," you need to reproduce in CI. `act` or `tmate` or a workflow-dispatch debug branch.
- **Editing the workflow on `main` to fix CI.** Edit on a branch, push, watch the workflow run on that branch. Workflows commit on `main` apply immediately and can break every other open PR.
- **Suppressing failures with `continue-on-error: true`.** Almost always wrong. It turns a hard signal into a soft one, and the next person doesn't know the job is meant to be advisory. Either gate or don't run it.
- **Treating "works locally" as proof.** Local environments accumulate state you forget about. CI's reproducibility is its value; align local to CI, not the other way around.

## Key Takeaways

- A failure in a parallel/matrix pipeline is a tree; read the *shape* before reading any single log.
- The pattern of failures localizes the cause: one leg vs all legs, one job vs many, real failures vs skipped downstream.
- Diagnostic posture: deterministic-or-flaky, correlated-with-diff, which-jobs-failed, code-or-infrastructure, what-changed-in-CI.
- Flakes are an emergency, not a nuisance — quarantine, reproduce, fix root cause, don't normalize re-running.
- `gh run view --log-failed` and `gh run rerun --failed` are the daily-driver commands for triage.
- Edit workflow YAML on a branch, never live on `main`.

---

*Prerequisites: Day 3 (linear CI triage — this extends it), all of Day 7's other topics (you can only troubleshoot a parallel/matrix/artifact pipeline once you understand how each piece is meant to work).*
