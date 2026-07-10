# Effective Code Review Practice

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D4 covered the etiquette and basics of PR review against the small CI fixes the cohort had then. Week 3's PRs are *substantial* — D11's session API and D12's scoring engine span hundreds of lines and touch transactions, locks, idempotency, and concurrent writes. D13 and D14 add a multi-component, stateful frontend slice. Reviewing this work well requires more than spotting typos. Today the cohort reviews each other's slice-integration PR using a vertical-slice-aware checklist and practices the discipline of *substantive feedback* — flagging architectural drift, missing edge cases, contract misalignment, and risky concurrent behavior, not just style.

This topic is the second half of the D4 lesson, not a replacement.

## What's Different About Vertical-Slice PRs

A D4 PR was "fix the broken CI step." A D15 PR is "the quiz-taking session creation flow." That changes what reviewers should look at:

| D4 review | D15 review |
|---|---|
| Single-file change | Multi-service, multi-file, often touching frontend + backend |
| One concern | Many concerns: contract, state, persistence, error paths, tests |
| Mostly mechanical | Architectural — does this design fit the rest of the system? |
| 5-minute read | 30–60 minute read |
| One reviewer plenty | At minimum two; ideally one backend, one frontend |

The shift is from "spot defects" to "build shared understanding of a design." Both matter; today is about the second.

## The Slice-PR Review Checklist

Don't memorize; *use it*. Paste it into the PR comments as a self-review template; the reviewer checks against the same list.

### 1. Contract Alignment

The most expensive bugs in this slice live at service boundaries. Read carefully:

- **Endpoint shape matches the contract doc / OpenAPI?** Method, path, request schema, response schema, error envelope.
- **Both sides updated together?** If the PR is backend-only but changes a field name, the frontend will break silently.
- **Status codes correct?** 200 vs 201 vs 204; 400 vs 422; 409 for idempotency violations and post-submit writes (D12).
- **Headers handled?** `X-Request-Id` forwarded; `Idempotency-Key` accepted on mutating endpoints (D14); `Authorization` propagated downstream (D11).
- **Pagination, filtering, sorting** consistent with the rest of the API?

### 2. State And Persistence

The slice writes to Postgres *and* reads from Mongo *and* maintains session state on the client. State bugs are the second-most-expensive class:

- **Transactions span the right scope?** D12's submit must be one transaction; a partial commit leaves a half-locked attempt.
- **Locks acquired where needed?** `select ... for update` on the session row before mutating it (D12).
- **Idempotency keys stored?** The unique constraint on `(candidate_id, idempotency_key)` from D12 is the safety net; verify migration and code both have it.
- **State machines respected?** Once a session is `submitted`, no further answers accepted (D12, surfaced in D14's lock-out UI).
- **No N+1 queries** in the question-fetch loop; one batched read from Mongo.
- **Migration is reversible?** Alembic down-migration sketched, not just up.

### 3. Error Paths

Happy paths get attention naturally; error paths are what reviewers add value on:

- **What happens if Postgres is unreachable mid-submit?** The handler should 503, not 500-with-stack-trace.
- **What happens if the candidate submits twice in a 100ms window?** The idempotency key plus row lock should make the second one a no-op, not a duplicate scoring run.
- **What happens if the body fails Pydantic validation?** 422 with a structured `detail[]` (D8).
- **What happens if the session is already submitted?** 409, not 500, not silently succeeding.
- **What happens if the timer expired before the submit landed?** Server-authoritative: server rejects the submit (D14).

A good review explicitly names which error cases the reviewer checked. "I verified the 409 path on second submit and the 422 path on missing question_id" — the author knows what *was* checked.

### 4. Logs And Observability

D10's correlation work is only valuable if every new handler uses it:

- **Every new log line carries `request_id`?** Should be automatic via the logging filter; verify the filter is registered for any new logger.
- **Log levels appropriate?** `INFO` for state changes, `WARNING` for recoverable issues, `ERROR` for things that need attention; not `INFO` for everything.
- **No PII or secrets in logs?** Candidate email, token contents, full request bodies — none of these belong in logs.
- **Structured fields, not f-string'd?** `log.info("scored", score=8, max=10)` not `log.info(f"scored 8/10 for {session_id}")`. Aggregators can index the first; the second is a flat string.

### 5. Tests

This is where most cohort PRs are thin. Push back hard if they are:

- **Unit tests for the pure logic?** Scoring functions (D12), the answer-validation schema (D14), reducers (D14) — each should have its own tests.
- **Integration tests for the DB layer?** Today's Topic 6 — row lock, idempotency constraint, cross-store join.
- **Smoke test updated** if the contract changed?
- **Playwright test still passes?** The author should have run it; the reviewer should ask if there's any doubt.
- **Tests assert behavior, not implementation?** "It returns 8/10" not "it calls `_compute_partial_credit` once."
- **Negative cases covered?** Not just "it works" but "it rejects bad input correctly."

### 6. Frontend Specifics

For PRs touching the take page:

- **Server components vs client components correctly chosen?** (D13.) Anything stateful belongs to a client component; static rendering should stay on the server.
- **`'use client'` boundary as narrow as possible?**
- **Form state via RHF + zod?** (D14.) Not raw `useState` for form fields.
- **Autosave + submit interaction correct?** Submit waits for in-flight autosave; double-submit prevented; lock-out engaged on success.
- **Accessibility?** Form labels, ARIA roles on the timer, button names — Playwright's role-based locators (Topic 3) double as accessibility verification.

### 7. Architectural Fit

The hardest category; the most valuable feedback:

- **Does this PR introduce a new pattern that already exists elsewhere?** If `test-management-service` already has a way to forward auth, the new code should use it, not invent a parallel one.
- **Is there a coupling that shouldn't exist?** Does `test-management-service` import from `question-management-service` Python code directly instead of going over HTTP?
- **Could this be simpler?** A 200-line handler with three classes is suspect.
- **Does this make the next PR easier or harder?** If today's choice creates work for tomorrow's author, name it.

## How To Phrase Feedback

The phrasing patterns from D4 still apply; what's new is *substantive* feedback at higher level:

- **Concrete > vague.** "Move the lock acquisition above the Mongo read so the lock window is shorter" beats "the lock seems too broad."
- **Question > assertion when unsure.** "Why call `select for update` here instead of inside the transaction block?" beats "this is wrong."
- **Suggest > complain.** When you flag an issue, offer a direction. "Consider extracting the scoring into a pure function — easier to unit-test." Pull the author toward a better design, don't just gripe.
- **Severity-labeled.** Use prefixes that the cohort adopted on D4: `nit:` (style), `question:` (clarification), `suggestion:` (optional improvement), `blocking:` (must change before merge). Today's review will have more `blocking:` than D4's; that's normal.

## The Reviewer's Workflow

For a slice PR, a 30-minute review beats a 5-minute one. Disciplined sequence:

1. **Read the PR description first.** What is it claiming to do?
2. **Pull the branch and run the smoke script (Topic 2).** If smoke fails, comment and stop — the author should fix that before review continues.
3. **Run the Playwright happy path (Topic 3).** If it fails, same.
4. **Now read the diff.** Work through the checklist; leave comments inline.
5. **Read the tests.** Do they actually exercise the new behavior? Do they have negative cases?
6. **Summary comment.** What overall? Approve, request changes, or "approve with nits."

The summary matters: it tells the author what to focus on first. Twelve inline comments without a summary leaves them parsing priorities; one summary saying "two blocking issues at the lock acquisition and missing the 409 path; rest is nits" is actionable in minutes.

## Common Slice-PR Smells

Things to be alert for in this slice specifically:

- **Idempotency key checked in code instead of with a DB constraint.** Race window. Push for the unique index + `IntegrityError` pattern from D12.
- **Lock acquired after the work, not before.** Defeats the purpose. The lock comes first.
- **Optimistic update inside a confirmed-action handler.** Submit should wait for confirmation; autosave can optimistic-update. Confusing the two means lost submits (D14).
- **`fetch` without timeout / retries.** Network is unreliable; D14 covered the retry-with-backoff pattern; new client code should use the same wrapper.
- **`Idempotency-Key` generated server-side.** Defeats the point. Client must generate per logical attempt, server stores it.
- **Tests that use real wall-clock waits** (`time.sleep(1)`). Flaky in CI; use fake timers or mock time.
- **Log lines without `request_id`.** Either the filter wasn't registered, or someone used a different logger. Both need fixing.

## What Not To Review For Today

To keep reviews focused, *don't* spend time on:

- **Style that linters cover.** ruff and eslint are running (D7); if they pass, don't relitigate quote style.
- **Bikeshedding variable names** in a clean, otherwise-good function.
- **Hypothetical edge cases** that aren't in scope ("what if 10000 candidates submit simultaneously?" — out of scope for a 25-person cohort).
- **Restructuring the PR after the fact.** If the author broke the work into three commits and you'd have done it in five, that's not a review comment.

The reviewer's time is finite. Spend it on contract, state, errors, and tests.

## The 25-Person Cohort Reality

Twenty-five trainees means twenty-five PRs flying through a review window. Ops doc mitigations apply (PR triage, mid-day review pass), but reviewer behavior matters too:

- **Pair reviews.** Two trainees review one PR together — faster, and both learn.
- **Time-box reviews to 30 minutes.** If a PR needs more, leave a "needs synchronous walk-through" comment and schedule a 10-minute screen share.
- **Don't queue.** If you've claimed a review, do it now or release it. Open claims that go stale are a coordination tax.
- **Trainer is the tiebreaker, not the default reviewer.** Cohorts review each other; trainer steps in for architectural disagreements only.

## Anti-Patterns

- **The "LGTM" review with no comments.** Either you read it carefully and have nothing to say (rare on a slice PR), or you didn't read it. Default to "approve with notes" or "request changes," even on good PRs.
- **The dump of 30 nits with no summary.** Author doesn't know what's blocking. Always add a summary.
- **Reviewing without pulling the branch.** You can't catch behavioral bugs by reading code; pull and run.
- **Architectural feedback after merge.** "Wish we'd done X" three days later doesn't help. Surface architectural concerns in review or accept the choice.
- **Treating review as gatekeeping rather than collaboration.** The reviewer's goal is to help the author ship better code, not to prove the reviewer is smarter.
- **Pretending to understand code you don't.** Ask. "Walk me through why the lock is here" is a fine comment; the author either explains it or realizes they can't, both useful.

## Key Takeaways
- Slice PRs need *substantive* review — contract alignment, state and persistence, error paths, tests, architecture — not just style.
- Use the seven-section checklist; paste it into your review or self-review.
- Always pull the branch and run smoke + Playwright before reading the diff; behavioral bugs don't show up in code review of code that doesn't run.
- Phrase feedback with severity prefixes (`nit`, `question`, `suggestion`, `blocking`) and end with a summary comment.
- 30-minute reviews are the right time budget for slice PRs; pair when possible, time-box always.
- Reviewer's goal: collaboration, not gatekeeping. Pull the author toward better design; help them ship.

---
*Prerequisites: [06-code-review-etiquette.md](../day-04/06-code-review-etiquette.md), [02-fastapi-routing-and-dependency-injection-patterns.md](../day-11/02-fastapi-routing-and-dependency-injection-patterns.md), [03-idempotency-for-retried-mutations.md](../day-12/03-idempotency-for-retried-mutations.md), [04-client-components-for-stateful-interactivity.md](../day-13/04-client-components-for-stateful-interactivity.md), [05-optimistic-updates-vs-server-confirmation.md](../day-14/05-optimistic-updates-vs-server-confirmation.md).*
