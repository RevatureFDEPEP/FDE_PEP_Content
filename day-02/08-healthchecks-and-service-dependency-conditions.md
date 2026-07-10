# Healthchecks and Service Dependency Conditions

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
A stack of 7 services that just happens to come up in the right order on your laptop is a stack that will fail on every other laptop in the cohort. Compose's `depends_on` with `condition: service_healthy` is the mechanism that turns "usually works" into "deterministically works" — but it only works if each dependency exposes a real healthcheck. This topic covers writing both halves so today's stack stops fighting startup race conditions.

## Why `depends_on` Alone Is Not Enough

The naive form of dependency in Compose looks like:

```yaml
  user-service:
    depends_on:
      - postgres
```

This only guarantees that the `postgres` *container* starts first — not that the Postgres *process inside* is ready to accept connections. A container can be in the "started" state for 5–10 seconds while Postgres initializes its data directory, replays WAL, and binds the listener. During that window, `user-service` happily fires its first query and crashes.

The fix is two-part:

1. Give Postgres (and Mongo, MinIO, every infra service) a **healthcheck** so Docker can tell when it is actually ready.
2. Change dependents to `condition: service_healthy` so they wait on that signal.

## Writing a Healthcheck

A Compose healthcheck is a command Docker runs periodically; exit code 0 means healthy, non-zero means unhealthy. The block:

```yaml
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "app", "-d", "users"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
```

Field by field:

- **`test:`** — The command. Prefer `["CMD", ...]` (exec form) over `CMD-SHELL` when possible; exec form does not invoke a shell and is more predictable. `pg_isready` is shipped inside the official Postgres image specifically for this purpose.
- **`interval:`** — Time between checks once running.
- **`timeout:`** — Max time a single check can take before being counted as failed.
- **`retries:`** — How many consecutive failures flip the state from "starting" to "unhealthy."
- **`start_period:`** — Grace period at startup during which failures do **not** count toward `retries`. Critical for databases that need 10–30s to come up cleanly.

The state machine: container starts → `starting` for `start_period` → first check → `healthy` (any pass) / `unhealthy` (retries exhausted). Dependents waiting on `service_healthy` block until the first `healthy`.

## Per-Service Healthcheck Recipes

The substrate uses these proven commands:

**Postgres:**
```yaml
healthcheck:
  test: ["CMD", "pg_isready", "-U", "app", "-d", "users"]
  interval: 5s
  timeout: 3s
  retries: 10
  start_period: 10s
```

**Mongo:**
```yaml
healthcheck:
  test: ["CMD", "mongosh", "--quiet", "-u", "app", "-p", "app",
         "--authenticationDatabase", "admin",
         "--eval", "db.adminCommand('ping').ok"]
  interval: 5s
  timeout: 3s
  retries: 10
  start_period: 15s
```

**MinIO:**
```yaml
healthcheck:
  test: ["CMD", "mc", "ready", "local"]   # or curl http://localhost:9000/minio/health/live
  interval: 5s
  timeout: 3s
  retries: 10
  start_period: 10s
```

For MinIO, the simpler `curl -f http://localhost:9000/minio/health/live` works if the image has `curl` (older minio images did, current ones may not — `wget` is the fallback).

**First-party Python services** (every FastAPI service should expose `/health`):
```yaml
healthcheck:
  test: ["CMD", "python", "-c",
         "import urllib.request; urllib.request.urlopen('http://localhost:8000/health').read()"]
  interval: 10s
  timeout: 3s
  retries: 5
  start_period: 15s
```

`python -c` is the dependable check inside slim images that lack `curl`/`wget`. The `/health` endpoint itself is implemented in each service's FastAPI router — just a `return {"status": "ok"}` is enough for Week 1.

## Wiring Dependencies

Now the dependent side:

```yaml
  user-service:
    depends_on:
      postgres:
        condition: service_healthy

  question-management-service:
    depends_on:
      mongo:
        condition: service_healthy

  api-gateway:
    depends_on:
      user-service:
        condition: service_healthy
      question-management-service:
        condition: service_healthy
      test-management-service:
        condition: service_healthy

  minio-init:
    depends_on:
      minio:
        condition: service_healthy
```

With this wiring, `docker compose up` produces a deterministic graph: infra services come up and become healthy → first-party services come up against healthy infra and become healthy → the gateway comes up against healthy services. No race conditions, no `sleep 30` hacks, no "try again."

## Example / Worked Scenario

Inspect a service's current health state:

```bash
docker compose ps
# STATUS column shows "(healthy)", "(unhealthy)", or "(health: starting)"

docker inspect --format '{{json .State.Health}}' postgres | jq
```

The JSON includes the most recent check outputs — useful when a healthcheck is failing and you cannot tell why. To simulate a failure, break the command and observe:

```yaml
    healthcheck:
      test: ["CMD", "false"]            # always fails
      interval: 2s
      retries: 3
      start_period: 2s
```

Run `docker compose up -d postgres` and watch `docker compose ps` cycle through `starting` → `unhealthy`. Any dependent waiting on this with `service_healthy` will sit indefinitely — exactly the diagnostic you want when something legitimately misbehaves.

## Common Pitfalls

- **Skipping `start_period:`.** Without a grace period, slow-starting services fail their first few checks and never recover. 10–15s for databases, 5s for app services is a safe baseline.
- **Healthcheck command depends on a tool not in the image.** `curl` is not in `python:3.11-slim`. Use `python -c "urllib.request..."` or add `curl` explicitly in the Dockerfile. Check what's available with `docker compose exec <svc> which curl`.
- **Using `condition: service_started` (the default).** This is the same as plain `depends_on`. It only confirms the container has been started, not that the process is ready. Always use `service_healthy` for anything that requires the dependency to be functional.
- **Healthcheck that checks the wrong thing.** `["CMD", "ls", "/"]` always passes; it tells you nothing. A check should exercise the same thing your application is about to exercise — a real connection, a real query, a real ping.

## Key Takeaways

- `depends_on` without a `condition:` only orders container *start*, not application *readiness*.
- Every infra service needs a healthcheck; every dependent service should wait on `condition: service_healthy`.
- `start_period:` is the grace window for slow startups — set it generously for databases.
- Pick a healthcheck command that exercises the same surface your application does.

---
*Prerequisites: [03-local-orchestration-with-docker-compose.md](03-local-orchestration-with-docker-compose.md), [05-postgresql-containerization.md](05-postgresql-containerization.md), [07-mongodb-containerization.md](07-mongodb-containerization.md), [06-minio-containerization.md](06-minio-containerization.md)*
