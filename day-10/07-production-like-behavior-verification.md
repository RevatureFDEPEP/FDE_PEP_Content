# Production-Like Behavior Verification (Smoke Testing, Log Inspection, Env Parity)

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Before today the slice "works on my machine" was sufficient — Day 8 ran the backend with `uvicorn`, Day 9 ran the frontend with `pnpm dev`. Today we promote the slice to "works in Compose," which is the closest pre-prod environment we have. The discipline we want to build is three habits: smoke test the happy path through every layer, read the logs even when things appear to work, and notice the gaps between local Compose and the eventual ECS/RDS/Atlas target. These three together — smoke + logs + env parity — are how you avoid the "passed CI, broke prod" failure mode.

## Smoke Tests vs Other Tests

Quick taxonomy so we use the right tool:

| Test type | Scope | Speed | When it runs |
|---|---|---|---|
| **Unit** | One function/class, mocked deps | ms | Every save (`pytest -x`) |
| **Integration** | One service + its real deps (db, mq) | seconds | PR CI |
| **Smoke** | The whole stack, happy path only | seconds–minute | After deploy / after `compose up` |
| **E2E** | Whole stack, multiple user journeys | minutes | Nightly / pre-release |

A smoke test is deliberately shallow: one path through the system, asserting "alive and basically working." It's the equivalent of plugging in a device and watching the indicator light come on before you start using it.

## A Smoke Test for the Question-Authoring Slice

Concrete script the trainer demos today:

```bash
#!/usr/bin/env bash
# scripts/smoke-question-authoring.sh
set -euo pipefail
BASE=${API_BASE:-http://localhost:8080}

echo "1) Gateway health"
curl -fsS "$BASE/healthz" | jq .

echo "2) Question service health (through gateway)"
curl -fsS "$BASE/api/questions/healthz" | jq .

echo "3) Presign an image upload"
PRESIGN=$(curl -fsS -X POST "$BASE/api/uploads/presign" \
  -H 'content-type: application/json' \
  -d '{"filename":"smoke.png","content_type":"image/png","size":1024}')
echo "$PRESIGN" | jq .
KEY=$(echo "$PRESIGN" | jq -r .key)
URL=$(echo "$PRESIGN" | jq -r .url)

echo "4) PUT a 1KB blob"
head -c 1024 /dev/urandom > /tmp/smoke.png
curl -fsS -X PUT "$URL" -H 'content-type: image/png' \
  --data-binary @/tmp/smoke.png

echo "5) Create a question"
ID=$(curl -fsS -X POST "$BASE/api/questions" \
  -H 'content-type: application/json' \
  -d "$(jq -n --arg k "$KEY" '{type:"single_select",prompt:"smoke",imageKey:$k,
    choices:[{text:"a",correct:true},{text:"b",correct:false}]}')" | jq -r ._id)
echo "Created $ID"

echo "6) Read it back"
curl -fsS "$BASE/api/questions/$ID" | jq .

echo "OK"
```

Six steps, zero secrets, fails fast on any broken layer. Run it every time you bring the stack up.

## Log Inspection as a Diagnostic Tool

When the smoke fails, logs are the first place to look — *not* the source code. Useful Compose commands:

```bash
# Tail everything
docker compose logs -f

# Tail one service since the smoke run started
docker compose logs -f --since=1m question-management-service

# Tail the gateway + backend together (chronological)
docker compose logs -f --timestamps api-gateway question-management-service

# Grep across all containers for a request id
docker compose logs --no-color | grep req_01HZABC
```

Triage rule: read the logs *bottom-up* from the last successful step. The first error line is usually the cause; everything below it is consequence.

What to look for:

- **Stack traces** — Python tracebacks bracketed by `Traceback (most recent call last):` and the exception line.
- **HTTP access lines** — `INFO ... 422 Unprocessable Entity` from FastAPI's access middleware.
- **Connection refusals** — `ECONNREFUSED`, `Connection reset by peer` — almost always means a downstream is not up yet (healthcheck miss).
- **Slow queries** — anything over ~500ms in a smoke run is suspicious; in prod it compounds.

## Env Parity: What's the Same, What's Not

Env parity is the property that local and prod behave the same. Perfect parity is impossible; useful parity is a discipline.

| Concern | Compose (local) | PEP target (AWS) | Parity status |
|---|---|---|---|
| App runtime | python:3.12-slim image | python:3.12-slim image on ECS | Same — good |
| Postgres | postgres:16 container | RDS Postgres 16 | Same engine — good |
| Mongo | mongo:7 container | MongoDB Atlas | Same wire protocol — close |
| Object store | MinIO | S3 | S3-compatible — close |
| Network | Compose bridge network | VPC with security groups | Different — watch CORS, DNS |
| TLS | None (HTTP) | TLS terminated at ALB | Different — auth headers may behave differently |
| Secrets | `.env` files | SSM Parameter Store / Secrets Manager | Different — never check secrets into `.env.example` |
| Time / TZ | Host TZ | UTC | Watch for date-formatting bugs |

The right reaction to a parity gap is *narrowing the gap when cheap, and explicitly tracking it when not*. Examples:

- **MinIO ≈ S3.** Cheap — use the same boto3 client config; both speak S3 SigV4. Already done in Day 8.
- **No local TLS.** Expensive to add for cohort use — note it as a known gap; explicitly test cookie `Secure` flag behavior in staging.
- **Mongo container ≠ Atlas.** Watch for index differences; Atlas auto-creates some, container doesn't.

## A Parity Checklist for Today

Run through this with the cohort once the smoke passes:

1. Image versions in `compose.yml` match the versions the AWS environment runs (Postgres 16, Mongo 7).
2. App container uses the same base image as the Dockerfile that produces the prod image (no `python:3.12` locally vs `slim` in prod).
3. App reads config from env vars only — no local-only `if DEBUG:` branches.
4. Healthcheck endpoints behave the same in container as on the host (`/healthz` returns 200 with no auth).
5. Default port mappings match the ECS task definitions (8000 for app, 8080 for gateway).
6. Log format is structured (JSON) in both — needed for the correlation work in Topic 3.

## When the Smoke Passes But Something's Still Wrong

The trap: green smoke, latent bug. Recognize these signals:

- **Smoke passes but a single field is missing in Mongo.** The model accepted None and persisted it; smoke didn't assert on it. Fix: add the assertion to the smoke script.
- **Smoke passes but takes 30 seconds.** Something is timing out and retrying. Look for `httpx.ReadTimeout` in logs.
- **Smoke passes once, fails on rerun.** Idempotency bug (unique index, duplicate ULID seed, leftover MinIO objects). Either reset state per run or make the script tolerant.

## Anti-Patterns

- **Treating "compose up succeeded" as the smoke test.** It only proves containers started, not that the slice works.
- **Tailing logs only when something breaks.** You miss the warning signs (slow queries, deprecation warnings, retry storms).
- **Mocking what you should run locally.** If Mongo runs in Compose, don't mock the Mongo repo in your integration tests — use the real container.
- **Letting Compose drift from the deploy target.** Pin image tags in both; bump them together.

## Key Takeaways
- A smoke test is a one-path, end-to-end check that the integrated stack basically works — keep it small, fast, scriptable.
- Logs are a primary diagnostic surface; read them bottom-up from the last successful step.
- Env parity is a discipline, not a state — narrow gaps where cheap, name them explicitly where not.
- Compose is the closest local environment to ECS/RDS/Atlas; treat it like pre-prod, not like a sandbox.
- Green smoke + clean logs + known parity gaps = high confidence "done."

---
*Prerequisites: [03-local-orchestration-with-docker-compose.md](../day-02/03-local-orchestration-with-docker-compose.md), [09-container-debugging-logs-exec-troubleshooting.md](../day-02/09-container-debugging-logs-exec-troubleshooting.md), [06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md](06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md).*
