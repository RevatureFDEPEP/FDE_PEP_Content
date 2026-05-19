# End-to-End Integration Testing Patterns in Compose

> *Day 5: Advanced Container Builds & Multi-Service Integration — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Today is the first time the **whole inherited slice** has to work together: user-service, question-management-service, test-management-service, api-gateway, the Next.js frontend, Postgres, Mongo, and MinIO — all orchestrated under Compose, with the login flow exercisable and the dashboard shell rendering for a logged-in user.

That's an **end-to-end (E2E) integration test**, even if you're "just" clicking through the UI manually. The Compose stack *is* the test substrate. This topic frames the pattern formally: how to structure an integration test against a Compose-orchestrated stack, what counts as a smoke test versus a full E2E suite, and what the inherited slice's E2E target looks like for the Week 1 retrospective.

Through Day 4 you tested *components* in isolation — services with their direct dependencies, CI lints and unit tests. Today you test the **emergent system**: do the services talk to each other correctly, does the gateway route requests, does the frontend's session cookie survive a round trip?

## The Test Pyramid, Briefly
Standard model:
- **Unit tests** (many, fast, isolated) — already running per-service.
- **Integration tests** (fewer, slower, real adjacent dependencies) — one service + its database, for instance.
- **End-to-end tests** (fewest, slowest, real everything) — what we're framing today.

The pyramid shape exists because E2E tests are expensive: slow, flaky, hard to debug. You don't write one for every code path. You write enough to know that **the system is hung together correctly** and rely on unit/integration tests for everything else.

For PEP, the Week 1 E2E target is a single **smoke test**: a logged-in user can reach the dashboard. That's it. It establishes the seam pattern; Week 2+ will add more.

## The Compose-as-Substrate Pattern
The pattern, abstractly:

```
                       [E2E test runner]
                              |
                              v   HTTP
            +-----------------+-----------------+
            | api-gateway (port 8080)           |
            +---+--------+--------+-------------+
                |        |        |
                v        v        v
       user-svc   question-svc    test-svc
            |        |        |
            v        v        v
         postgres  mongo   postgres
```

The whole graph is brought up by `docker compose up`. The E2E test runner — which can be `curl`, `httpie`, a `pytest` suite, Playwright, or a hand-driven browser — talks **only to the api-gateway**. It doesn't know which downstream service handles which endpoint. That's the point: the gateway is the integration seam, and the test exercises *real routing*, not a mock.

Three properties make this work as a substrate:

1. **Healthchecks gate dependencies.** From Day 2: `depends_on: condition: service_healthy` means the test runner is started *only after* every service it depends on reports healthy. No race-condition flakiness on startup.
2. **Fixed Compose network DNS.** Services reach each other by service name (`http://user-service:3000`) inside the Compose network. The test runner reaches the gateway from outside (`http://localhost:8080`).
3. **Volumes are scoped to the Compose project.** A fresh `docker compose up` starts with seeded test data; tearing down with `docker compose down -v` wipes volumes for a clean re-run.

## A Smoke Test, End to End
The Week 1 deliverable's smoke test, expressed as steps:

1. **Bring up the stack:** `docker compose up -d`
2. **Wait for health:** `docker compose ps` shows all services `(healthy)`.
3. **Login (POST):**
   ```
   POST http://localhost:8080/api/auth/login
   Body: {"username":"trainer@example.com","password":"seed-password"}
   Expected: 200 OK, Set-Cookie header with session token.
   ```
4. **Fetch dashboard data (GET, with cookie):**
   ```
   GET http://localhost:8080/api/dashboard
   Cookie: session=<from step 3>
   Expected: 200 OK, JSON with the trainer's expected payload (course list, recent attempts).
   ```
5. **Tear down:** `docker compose down`

That's the smoke test. Five steps. It exercises:
- The api-gateway is reachable from outside the Compose network.
- The user-service authenticates against Postgres.
- Session cookies round-trip the gateway correctly.
- The dashboard endpoint composes responses from at least two downstream services without erroring.

If those four properties hold, "Week 1 deliverable: inherited slice runs E2E" is satisfied.

## Manual vs Scripted
Today's smoke test can be manual — open the browser, type the login form, see the dashboard. That's a valid E2E verification.

Tomorrow (Day 6's reverse-proxy and Compose-profiles work) and onward, the **same flow** wants to be scripted so CI can run it. Day 7 will introduce a `e2e` Compose profile that brings up the whole stack and runs a scripted version of the steps above against it. The manual smoke test today is the *spec* for that script.

## Test Data and Idempotency
A flaky E2E test is worse than no E2E test — it teaches the team to ignore failures. The biggest source of flakiness is test data:

- **Seeded data, not inserted data.** The smoke test relies on a trainer account that exists in the seed scripts (`scripts/seed.sql`, etc.). It does *not* create one as a setup step. Setup steps introduce ordering and concurrency bugs.
- **Read-only assertions where possible.** The login + dashboard smoke test reads existing data; it doesn't mutate. Mutating tests need cleanup, and cleanup needs to be bulletproof.
- **`docker compose down -v` between runs in CI.** Volumes carry state. A wiped volume guarantees seeded state on the next `up`.

## Where E2E Lives in the Workflow
Recurring through the curriculum:
- **Locally:** developer runs E2E before pushing a non-trivial change.
- **CI (Day 7+):** the `e2e` job runs the scripted smoke test in a dedicated job after build/test pass.
- **Pre-merge:** an explicit gate on `e2e` for the `main` branch — covered when branch protection evolves.

For today, "I brought up Compose, logged in, saw the dashboard" is enough. The retrospective notes the gap (no scripted E2E yet) and Week 2 closes it.

## Example / Worked Scenario
The trainer asks: "Show me the inherited slice running E2E. I want to see a logged-in dashboard, and I want you to narrate what's happening at each layer."

```bash
# 1. Clean state
docker compose down -v
docker compose up -d

# 2. Confirm healthchecks
docker compose ps
# user-service              Up (healthy)
# question-management-svc   Up (healthy)
# test-management-svc       Up (healthy)
# api-gateway               Up (healthy)
# frontend                  Up (healthy)
# postgres                  Up (healthy)
# mongo                     Up (healthy)
# minio                     Up (healthy)

# 3. Hit the gateway directly first — fail fast if routing's broken
curl -i -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"trainer@example.com","password":"seed-password"}'

# HTTP/1.1 200 OK
# Set-Cookie: session=eyJhbG...; HttpOnly; Path=/
# {"user":{"id":"...","role":"trainer"}}

# 4. Use that cookie against the dashboard endpoint
curl -i http://localhost:8080/api/dashboard \
  --cookie "session=eyJhbG..."

# HTTP/1.1 200 OK
# {"courses":[...], "recentAttempts":[...]}

# 5. Now do it in the browser — open http://localhost:3000, log in,
#    confirm the dashboard renders the same data.
```

**Narration to the trainer:**

> "When I POST to `/api/auth/login`, the gateway routes it to `user-service`, which queries Postgres for the trainer record, validates the password hash, and emits a session cookie. The gateway sets that cookie on the response. The browser stores it.
>
> When the frontend loads `/dashboard`, it makes an authenticated request through the gateway to `/api/dashboard`. The gateway routes that to whatever service composes the dashboard payload — which in this slice is `user-service` plus `test-management-service` for recent attempts. If any of those calls fails, the dashboard returns partial data with a logged error.
>
> The fact that this works end-to-end without me intervening means: gateway routing is correct, the user-service Postgres connection is correct, the session cookie format is consistent across services, and dependent services are healthy. If any one were broken, this flow would surface it."

That narration is the muscle being built — system thinking, not component thinking.

## Common Pitfalls
- **Bypassing the gateway in tests.** Hitting `user-service:3000` directly from the host (or even a different port) tests the service in isolation, not the system. Always go through the gateway in E2E.
- **Skipping `docker compose down -v` between runs.** Stale volumes mean stale auth tokens, half-committed test data, mystery state. The wipe is cheap.
- **Setup-creating data.** A test that does "POST /users to create the user, then POST /login" tests the creation path *and* the login path entangled. Use seeded users; test creation paths separately.
- **Long timeouts hiding slow startups.** If your "wait for healthy" loop is 60 seconds, you'll never notice that one service took 55. Tight per-service healthcheck intervals (5-10s) surface degradation early.
- **No teardown on failure.** A failed E2E test that doesn't `docker compose down` leaves a stack running that pollutes the next attempt. Wrap the test in `trap "docker compose down -v" EXIT` or equivalent.

## Key Takeaways
- Compose itself is the E2E test substrate; the gateway is the integration seam to assert against.
- Healthchecks (Day 2) eliminate startup-race flakiness in E2E.
- Week 1 needs *one* smoke test (login → dashboard). Week 2+ scripts it.
- Seeded data, not inserted data; clean teardown every run.

---
*Prerequisites: `day-2-local-orchestration-with-docker-compose.md`, `day-2-healthchecks-and-service-dependency-conditions.md`.*
