# End-to-End Smoke Testing Patterns

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
A smoke test is the fastest, cheapest thing you can run that proves a vertical slice works. It is not a unit test, not an integration test, not a UI test — it's a small scripted walk through the happy path that catches the catastrophic failures (the stack isn't up, the gateway isn't routing, the DB schema is wrong). D10 introduced the idea with a five-step curl-and-click for the authoring slice. Today we write a real, repeatable smoke script for the quiz-taking slice and add it to the trainer's daily verification routine.

Smoke tests are deliberately broad and shallow. They prove *the system is alive*. Depth — the actual UI happy path with assertions — lives in Topic 3's Playwright test.

## The Three Test Levels

It helps to be explicit about what smoke is *not*:

| Level | Question it answers | Speed | When to run |
|---|---|---|---|
| **Unit** | "Does this function/reducer behave correctly?" | <1s each | On every save (D7, D14) |
| **Integration** | "Do my service + its real DB agree?" | ~seconds | On every PR (Topic 6) |
| **Smoke** | "Is the integrated system alive at the boundaries?" | ~10s total | On every `compose up` |
| **E2E (UI)** | "Does the user journey actually work in a browser?" | ~30s–2min | Before merging slice-integration PRs (Topic 3) |

Smoke sits between integration and E2E. It uses real services, real DBs, real network — but talks to them via `curl`, not a browser, and skips the UI entirely. The win is speed: the smoke script runs in seconds and tells you whether running the slower Playwright suite is even worth it.

## What A Smoke Test Asserts

A good smoke test for the quiz-taking slice answers:

1. **Are the containers up and healthy?** (`docker compose ps`, healthcheck status.)
2. **Does the gateway route?** (One `curl` per service through the gateway.)
3. **Can a user authenticate?** (`POST /auth/login` returns a token.)
4. **Does the slice persist to its primary store?** (`POST /sessions` writes a row in Postgres.)
5. **Does the slice read back?** (`GET /sessions/{id}` returns what was written.)
6. **Does the contract round-trip?** (`POST .../answer` accepts the discriminated payload; idempotency key returns the same result twice.)
7. **Does the terminal state lock?** (`POST .../submit` flips status; a second answer attempt returns 409.)
8. **Are logs correlated?** (The `X-Request-Id` we sent in headers appears in all three services' logs.)

Eight checks, all scriptable, all running in under 30 seconds against a fresh `docker compose up`.

## The Smoke Script

Real shell script, lives at `scripts/smoke-quiz-taking.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

GATEWAY="${GATEWAY:-http://localhost:8080}"
RID="smoke_$(date +%s)_$$"

echo "==> 1. healthcheck"
docker compose ps --format json | jq -r '.[] | "\(.Name)\t\(.Health)"' | grep -v healthy && {
  echo "FAIL: not all containers healthy"; exit 1;
}

echo "==> 2. gateway routing"
for svc in user-service test-management-service question-management-service; do
  curl -fsS "${GATEWAY}/_internal/${svc}/healthz" >/dev/null \
    || { echo "FAIL: gateway -> ${svc} routing"; exit 1; }
done

echo "==> 3. auth"
TOKEN=$(curl -fsS -X POST "${GATEWAY}/auth/login" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: ${RID}-auth" \
  -d '{"email":"smoke@example.com","password":"smoke-pass"}' | jq -r .token)
[[ -n "${TOKEN}" && "${TOKEN}" != "null" ]] || { echo "FAIL: no token"; exit 1; }

echo "==> 4. create session"
SESSION=$(curl -fsS -X POST "${GATEWAY}/sessions" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: ${RID}-create" \
  -d '{"test_id":"smoke-test-1"}' | jq -r .session_id)
[[ -n "${SESSION}" ]] || { echo "FAIL: no session"; exit 1; }

echo "==> 5. read back"
GOT=$(curl -fsS "${GATEWAY}/sessions/${SESSION}" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "X-Request-Id: ${RID}-read" | jq -r .session_id)
[[ "${GOT}" == "${SESSION}" ]] || { echo "FAIL: read mismatch"; exit 1; }

echo "==> 6. answer + idempotency"
IDEM="idem-${RID}"
R1=$(curl -fsS -X POST "${GATEWAY}/sessions/${SESSION}/answer" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Idempotency-Key: ${IDEM}" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: ${RID}-ans1" \
  -d '{"question_id":"q1","selected":[1]}')
R2=$(curl -fsS -X POST "${GATEWAY}/sessions/${SESSION}/answer" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Idempotency-Key: ${IDEM}" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: ${RID}-ans2" \
  -d '{"question_id":"q1","selected":[1]}')
[[ "${R1}" == "${R2}" ]] || { echo "FAIL: idempotency violated"; exit 1; }

echo "==> 7. submit + lock"
curl -fsS -X POST "${GATEWAY}/sessions/${SESSION}/submit" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "X-Request-Id: ${RID}-submit" >/dev/null

STATUS=$(curl -fsS -o /dev/null -w '%{http_code}' \
  -X POST "${GATEWAY}/sessions/${SESSION}/answer" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Idempotency-Key: post-submit-${RID}" \
  -H "Content-Type: application/json" \
  -d '{"question_id":"q2","selected":[1]}' || true)
[[ "${STATUS}" == "409" ]] || { echo "FAIL: expected 409 after submit, got ${STATUS}"; exit 1; }

echo "==> 8. log correlation"
HITS=$(docker compose logs --no-color --since 1m 2>&1 | grep -c "${RID}")
[[ "${HITS}" -ge 3 ]] || { echo "FAIL: request id not correlated across services (saw ${HITS} hits)"; exit 1; }

echo "OK — quiz-taking slice smoke passed."
```

Read it as a runbook, not as code: each `==>` line is a step a trainer can re-execute manually if the scripted version fails confusingly.

## Test Data Strategy

Smoke tests need a known-good seed: a fixed user, a fixed test, fixed questions. Two patterns:

- **Seeded at `compose up` time.** A migration or init-container loads a `smoke@example.com` user, a `smoke-test-1` test, and a handful of questions every time the stack comes up. Cheap; only works if smoke data won't pollute manual testing.
- **Created and torn down per run.** Smoke script creates its own user/test, runs, deletes. More hygienic; more fragile because teardown can fail and leave drift.

For PEP we use the first — Compose is ephemeral (`docker compose down -v` resets everything), so a polluted DB is one command away from clean.

## When Smoke Tests Run

Three moments matter:

1. **After every `docker compose up`** — by the trainer at the start of the morning, by every trainee when they reset their stack. If the smoke fails, *stop* and triage before doing any other work.
2. **In CI on the slice-integration PR.** The CI job spins up the Compose stack, runs the smoke script, tears down. This is the gate for merging multi-service PRs.
3. **As the first step of the Playwright suite.** If smoke fails, skip Playwright — the browser test will fail too and waste 90 seconds. Cheap fail-fast.

Smoke does not run on every push to every branch — it's too expensive (requires the full stack). Unit tests cover that level.

## What Smoke Tests Deliberately Skip

Important to be explicit about what *isn't* in scope, because every cohort tries to expand smoke into something it shouldn't be:

- **UI rendering.** That's Playwright. A smoke test that opens a browser stops being a smoke test.
- **Edge cases.** Smoke is the happy path. The 409 and idempotency checks are there to prove the *contract* responds correctly, not to exhaustively probe edge behavior.
- **Performance.** No timing assertions; the test just has to complete.
- **Correctness of scoring algorithms.** That's unit tests (D12).
- **Multi-user scenarios.** One user, one session, one happy path.

When the cohort says "let's add X to the smoke test," ask: "does X belong at a different layer?" Usually yes.

## Anti-Patterns

- **Smoke that takes a minute to run.** Then it's not smoke; it's an integration test wearing smoke's name. Keep it under 30 seconds. If it grows past that, split it.
- **Smoke without a known-good seed.** "It worked yesterday" doesn't tell you anything if the data is different.
- **Asserting on internal implementation.** Smoke checks behavior at the gateway boundary — public contracts only. If you find yourself querying Postgres in the script, you've gone deeper than smoke.
- **Smoke that's a sequence of `curl`s with no assertions.** "It returned 200" isn't enough; assert on the value of fields you care about (`session_id` populated, idempotency response identical, second submit returns 409).
- **Smoke that runs only on the trainer's machine.** It must be in `scripts/`, checked in, and run by the CI job. Otherwise it's a one-person ritual, not a safety net.
- **Smoke that ignores log correlation.** The cheapest end-to-end check that the D10 X-Request-Id wiring still works is step 8 above. Skipping it means the *next* time you need to debug a distributed failure, you'll discover the wiring rotted weeks ago.

## Key Takeaways
- Smoke tests are broad, shallow, and fast — they prove the slice is *alive* without probing depth.
- For the quiz-taking slice, eight assertions cover the relevant boundaries in under 30 seconds.
- Smoke runs after every `compose up`, in CI for slice-integration PRs, and as the fail-fast step before Playwright.
- Test data should be seeded at `compose up`, not created per-run — Compose's ephemerality handles cleanup.
- Smoke is the *contract layer* — gateway-visible behavior. Internal-state assertions belong at integration-test depth (Topic 6).

---
*Prerequisites: day-10-multi-service-feature-delivery-patterns, day-12-idempotency-for-retried-mutations, day-15-vertical-slice-integration-in-a-local-compose-environment.*
