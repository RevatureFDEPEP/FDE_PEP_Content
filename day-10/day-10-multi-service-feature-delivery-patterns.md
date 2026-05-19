# Multi-Service Feature Delivery Patterns

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
A vertical slice that spans more than one service is finished only when the slice works end-to-end against a production-shaped environment — not when each service's PR is merged. Day 8 produced a backend; Day 9 produced a frontend. Today we wire them together through `api-gateway` in local Compose and prove the slice works. The pattern we exercise — contract first, deploy in dependency order, smoke test, then declare done — is how multi-service features actually ship in a microservice system. Treat the "merge → done" reflex from monolithic projects as a bug.

## Why "Each PR Merged" Isn't "Feature Done"

A microservice slice has at least three asymmetries you don't get in a monolith:

1. **Independent deploy boundaries.** The backend and the frontend ship from different repos (or at minimum different images). They can be at incompatible versions at the same time.
2. **Network in the middle.** Localhost behaviors (CORS, TLS, DNS, body size limits, gateway routing) don't appear until things actually talk over the wire.
3. **Schema drift.** The Pydantic model on the server and the zod schema on the client are two separate sources of truth (until you generate one from the other). They must agree by contract, not by hope.

Concretely, after Day 9 the team has *two green PRs* but no proof that a trainer can author a question end-to-end. Today's work closes that gap.

## The Four-Step Multi-Service Delivery Pattern

### 1. Align the Contract

Before any cross-service work merges, the contract is the artifact:

- HTTP path, method, request body shape, response shape, error envelope.
- For our slice: `POST /api/questions` with the discriminated-union body from `day-8-pydantic-v2-discriminated-unions` and the 422 `detail[]` envelope from `day-8-fastapi-request-validation-and-error-shapes`.
- Document the contract somewhere both sides read (OpenAPI spec, a markdown file in the gateway repo, or a generated TS client). For the cohort, the FastAPI `/docs` endpoint is the source of truth.

If the contract changes mid-slice, *both sides update before either ships*. A backend-only field rename will break the frontend silently.

### 2. Deploy in Dependency Order

Inside Compose the order matters because the frontend's first render may try to hit the API:

```
postgres + mongo + minio   →   user-service + question-management-service   →   api-gateway   →   web (Next.js)
```

`depends_on` with healthcheck conditions (covered in `day-2-healthchecks-and-service-dependency-conditions`) enforces this. In real environments the order is the same: data stores first, then leaf services, then aggregating layers, then UI.

### 3. Smoke Test the Integrated Slice

A smoke test is not a unit test or an integration test — it's "does the happy path work at all." For our slice:

1. `curl http://localhost:8080/healthz` — gateway is up.
2. `curl -X POST http://localhost:8080/api/questions -d @sample.json` — backend persists.
3. Open `http://localhost:3000/questions/new` — frontend loads.
4. Author and submit a question through the UI — full path exercised.
5. `docker compose exec mongo mongosh ... db.questions.find({})` — data is actually there.

Five steps, two minutes, catches 80% of integration bugs. See `day-10-production-like-behavior-verification` for depth.

### 4. Verify and Declare Done

"Done" is a decision, not a feeling. The checklist for this slice:

- [ ] Backend smoke test passes against running Compose stack.
- [ ] Frontend smoke test (author both question types) passes via browser.
- [ ] Mongo contains both documents with the expected `schema_version`.
- [ ] Image uploaded via the pre-signed URL flow lives in MinIO.
- [ ] Logs from frontend → gateway → backend are correlatable for one request (see `day-10-distributed-log-correlation-across-services`).
- [ ] `trainer/reference` branch demos Week 2 cleanly.

Only when all six are checked does the slice get the "done" label.

## Worked Example: Today's Slice

Time-boxed walkthrough trainers will lead this morning:

```bash
# Reset to known state
docker compose down -v
docker compose up -d --build

# Wait for healthchecks
docker compose ps  # everything should show (healthy)

# Backend smoke
curl -s -X POST http://localhost:8080/api/questions \
  -H 'Content-Type: application/json' \
  -d @scripts/sample-single-select.json | jq .

# Frontend smoke (manual, in browser)
open http://localhost:3000/questions/new
# Author one single_select, one multi_select; upload an image each.

# Verify persistence
docker compose exec mongo mongosh questions --quiet \
  --eval 'db.questions.find({}, {type:1, prompt:1}).pretty()'

# Verify object storage
docker compose exec minio mc ls local/questions
```

If any step fails, that's the integration bug. Triage with logs (Topic 3) before reaching for code changes.

## Contract Drift: What It Looks Like

The most common multi-service bug is contract drift. Symptoms during today's integration:

- **422 on a payload zod thought was valid.** Backend added a required field; frontend schema didn't follow. Fix: update both schemas; consider generating zod from OpenAPI.
- **201 with a missing field on the response.** Backend renamed `_id` to `id`; frontend still reads `_id`. Fix: align serializer aliases.
- **Image upload succeeds, save fails.** Frontend stores `image_key`, backend expects `imageKey` (camelCase vs snake_case). Fix: pick one — recommend camelCase on the wire, snake_case in Python via Pydantic `Field(alias=...)`.

Each of these *would have* been caught by step 1 (contract alignment) if it had been a real review, not a glance.

## Anti-Patterns

- **"Big bang" integration.** Three services merge to main on Friday afternoon; integration starts Monday. Always integrate as you go.
- **Skipping the smoke test because "the unit tests pass."** Unit tests don't run over the wire.
- **Hot-fixing the contract in one service only.** Always update producer and consumer in the same PR or back-to-back, with the producer first.
- **No `trainer/reference` branch.** Without a known-good integrated state, regressions are invisible.
- **Treating Compose as "just dev."** If Compose passes and prod fails, your env parity is broken (Topic 2).

## Key Takeaways
- A multi-service feature is "done" when the integrated slice passes a smoke test against a prod-shaped stack — not when individual PRs merge.
- The four-step pattern is: align contract → deploy in dependency order → smoke test → verify and declare done.
- The contract is the artifact. Both sides update together when it changes.
- Compose `depends_on` with healthchecks is what enforces deploy order locally.
- Keep a `trainer/reference` branch as a known-good integrated baseline.

---
*Prerequisites: day-8 backend slice topics, day-9 frontend slice topics, day-2-healthchecks-and-service-dependency-conditions.*
