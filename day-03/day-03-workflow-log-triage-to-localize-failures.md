# Workflow Log Triage to Localize Failures

> *Day 3: Diagnose the CI Pipeline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Your deliverable today is a written list of five bugs in `ci-pipeline.yml` with a root-cause hypothesis for each. You won't find them by re-reading the YAML in isolation — you find them by running the pipeline, watching it fail, and reading the logs methodically. This topic teaches the discipline of CI log triage: how to drill from a red workflow icon down to a specific step, how to separate setup noise from real failures, and how to leave the log with a defensible hypothesis rather than a guess.

## The triage discipline: drill, isolate, hypothesize

Open a failing CI run and resist the urge to immediately scroll the giant red block of text. Triage in this order:

1. **Identify which job failed.** A workflow has multiple jobs; usually one fails first and the rest get skipped or cascade-fail. Find the *first* failure chronologically.
2. **Identify which step inside that job failed.** Each job is a list of steps, each with an icon. Click into the failing step (red X).
3. **Read the step from top to bottom, not bottom to top.** The bottom is the final error. The top is the setup. The real cause is often in the middle.
4. **Distinguish setup failure from execution failure.** Did the step fail before your code ran, or while your code ran?
5. **Form one or two hypotheses.** Write them down. Don't dive into "fix mode" yet.

The goal of triage is to leave with a sentence like: *"`test-services (user-service)` failed because the `pip install` step errored on a missing wheel for `psycopg2-binary==2.9.X`, which I hypothesize is because the cache restored an older `requirements.txt` lockfile."* You can't form that sentence by skim-reading.

## The anatomy of an Actions step log

Each step in the GitHub Actions UI is a collapsible block. Expanded, it shows:

```
##[group]Run pytest services/user-service/tests
pytest services/user-service/tests
shell: /usr/bin/bash -e {0}
env:
  DATABASE_URL: ***
  PYTHONUNBUFFERED: 1
##[endgroup]
============================= test session starts ==============================
platform linux -- Python 3.11.7, pytest-7.4.4
collected 42 items

services/user-service/tests/test_auth.py ............                    [ 28%]
services/user-service/tests/test_users.py .......F.....                  [ 60%]
...
=================================== FAILURES ===================================
_____________________________ test_create_user _____________________________
...
ImportError: cannot import name 'PasswordHasher' from 'argon2'
============================== 1 failed, 41 passed in 12.34s ===================
##[error]Process completed with exit code 1.
```

The structure:

- **`##[group]` ... `##[endgroup]`** brackets the metadata GitHub generates: the command it ran, the shell, the env vars (secrets redacted as `***`).
- The middle is the actual command output.
- **`##[error]Process completed with exit code N.`** is GitHub's terminal log line. The exit code tells you what to look for upstream.

When triaging, the first useful information is in the `##[group]` block — it tells you exactly what command ran with which env. The second useful information is the **first** error in the body (often a chain of errors; the root is at the top).

## Reading errors top-down, not bottom-up

A common mistake: scroll to the bottom, see "FAILED", look at the last few lines, guess.

The actual failure is almost always **earlier** in the log. Python tracebacks show the root error at the top of a traceback chain (or, for Python's "during handling of the above exception" format, alternating). Linker errors print the offending object first. `pip install` failures print the failing package early and then a long resolution graph.

The discipline: **scroll up from the bottom until you find the first `Error:`, `FAILED:`, or `Traceback`**, and start reading there. The bottom is rarely the cause.

## Distinguishing setup failure from execution failure

This is the single most useful distinction for today's exercise. Five seeded bugs in `ci-pipeline.yml` are *pipeline* bugs, not *application* bugs — they fail before your application code's tests run.

**Setup failure** smells like:

- `actions/checkout` error.
- `actions/setup-python` or `setup-node` error.
- `pip install` / `pnpm install` error.
- `Error: Unable to locate executable file: pytest`.
- `permission denied` on a binary.
- `No such file or directory` on a config file.
- Cache restore/save errors.

**Execution failure** smells like:

- A test failed (`assert ... == ...`).
- An assertion error from your code.
- A timeout.
- A runtime exception that's specifically about your domain logic (`UserNotFound`, `InvalidPayload`).

For the seeded-bug exercise, **expect every failure to be a setup failure**. If you see "assertion failed in test_users.py", look harder — the upstream setup probably did the wrong thing (restored stale deps, set wrong env vars, used wrong Python version) and the test is collateral.

## The summary view: which job failed first

The Actions UI's "Jobs" sidebar shows every job with its state. For a failing run:

- Green check — job passed.
- Red X — job failed.
- Yellow circle — job is still running (or being retried).
- Gray dash — job was skipped (usually because a `needs:` dependency failed).

When `lint` fails and `test-services` is gray, **don't open `test-services`** — it didn't run. Open `lint`. The cascade is misleading if you skim.

When two jobs in parallel both fail, find the one with the earlier failure timestamp. They're independent failures, but one of them is your starting point — and often, fixing one reveals the other was a knock-on effect.

## Matrix expansions and partial failures

A matrix job expands into multiple parallel job instances. Each gets its own log.

```yaml
strategy:
  matrix:
    service: [user-service, question-management-service, test-management-service]
```

In the UI, you'll see three entries: `test-services (user-service)`, `test-services (question-management-service)`, `test-services (test-management-service)`. If only one is red and the others are green, **the failure is specific to that matrix entry**. The bug is in that service's config, deps, or tests — not in the workflow plumbing (because the same plumbing works for the other two).

Conversely, if all three matrix entries fail with the same error, the bug is in the **shared** plumbing — the cache config, the setup step, an env var. Read the failing step once, not three times.

## Reading the redacted lines

Secrets show as `***` in the log. This is a diagnostic signal:

- `***` present in the env block → the secret is set (has some value).
- Empty value or completely missing line → the secret reference resolved to empty string.

For diagnosing the secrets-related seeded bug, expand the `##[group]` block of the failing step and look at the `env:` section. If you expected `${{ secrets.AWS_ACCESS_KEY_ID }}` to populate `AWS_ACCESS_KEY_ID` and the env block shows it as empty (no `***`), the secret is misnamed or out of scope.

## The `actions/upload-log` trick (and its alternatives)

When a step fails and you wish you had more diagnostic info, your options on a re-run are:

1. **Re-run with debug logging enabled.** Click "Re-run jobs" → "Enable debug logging." Sets `ACTIONS_RUNNER_DEBUG=true` and `ACTIONS_STEP_DEBUG=true`. The log becomes ~3x longer but includes runner-internal info.
2. **Add diagnostic `run:` steps.** For your own PR, add `- run: env | sort` or `- run: pip list` before the failing step to dump state. Useful for one-off debugging, remove before merge.
3. **Use `tmate` action for an interactive shell.** `mxschmitt/action-tmate@v3` opens an SSH session into the runner. Powerful, dangerous, treat secrets carefully.

For today's exercise, debug logging and inline `env | sort` are usually sufficient.

## Example / Worked Scenario

You push a PR. The CI fails. The UI shows:

```
Jobs:
  lint                            ✓ passed
  test-services (user-service)    ✗ failed
  test-services (question-mgmt)   ✓ passed
  test-services (test-mgmt)       ✓ passed
  build-frontend                  — skipped
```

First observation: matrix is partial-failure. Only `user-service` failed. `lint` (a sibling) passed. `build-frontend` was skipped because of `needs:` cascade.

You click into `test-services (user-service)`. Five steps:

```
✓ Set up job
✓ Run actions/checkout@v4
✓ Run actions/setup-python@v5
✓ Restore cache
✗ Run pip install -r services/user-service/requirements.txt
— Run pytest services/user-service/tests
```

The failing step is `pip install`, not `pytest`. The test step is dashed (didn't run).

You expand `Restore cache`. Log says:

```
Cache restored from key: pip-cache
```

You expand `pip install`. Top of the log:

```
Collecting psycopg2-binary==2.9.9
  Could not find a version that satisfies the requirement psycopg2-binary==2.9.9
ERROR: No matching distribution found for psycopg2-binary==2.9.9
```

Quick check: `services/user-service/requirements.txt` does list `psycopg2-binary==2.9.9`. The other two services' requirements don't.

Hypothesis: the cache key `pip-cache` is shared across all three matrix entries (no `matrix.service` partition). The cache was last written by `question-management-service`, whose `requirements.txt` does not pin `psycopg2-binary==2.9.9`. When `user-service` restored that cache, it got the wrong pip wheel index state, leading pip to fail to find a wheel it should have found.

Note the chain: matrix partial-failure → setup failure (not execution) → cache step succeeded but produced wrong state → pip install failed. The hypothesis points at `key: pip-cache` (which lacks `matrix.service`) as the root cause, not at pip or psycopg2.

The deliverable for this bug: *"Cache key `pip-cache` is shared across all matrix entries; the matrix produces last-writer-wins cache state, so `user-service` restores a cache populated by a sibling service with different dependencies. Fix candidate (Day 4): include `matrix.service` and `hashFiles(...)` in the cache key."*

## Common Pitfalls

- **Skim-reading the bottom of the log.** The actual error is almost always earlier in the step. Scroll up from the bottom until you find the first error.
- **Opening skipped jobs.** A gray dash means the job didn't run. Don't waste time looking for an error there — find the job whose failure caused the cascade.
- **Confusing matrix partial-failure for plumbing bugs.** If one matrix entry fails and others pass, it's specific to that entry. If all fail identically, it's shared plumbing.
- **Treating the test failure as the root cause.** In a brownfield CI exercise, a test failure is often the *symptom*; the upstream setup did the wrong thing.
- **Not re-running with debug logging when stuck.** Free diagnostic info that most people skip.

## Key Takeaways

- Triage in order: which workflow → which job → which step → which line. Always drill, never skim.
- Read errors top-down within a step. The first `Error:` or `Traceback` is usually the root; everything after is cascade.
- Distinguish setup failure from execution failure. In the Day 3 seeded-bug exercise, expect setup failures dominantly.
- Matrix partial-failure points at config specific to one matrix entry; matrix total-failure points at shared plumbing.
- Leave triage with a written hypothesis — a sentence identifying the failing step *and* the suspected upstream cause. No fixes yet.

---
*Prerequisites: `day-3-github-actions-structure-workflows-jobs-steps-runners.md`, `day-2-container-debugging-logs-exec-troubleshooting.md`.*
