# Service Composition Patterns in docker-compose

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Adding a new service to a working Compose stack is deceptively easy: the wrong way still appears to work and only breaks intermittently. Today we add `reporting-and-analytics-service` and its dedicated `reporting-postgres` to the existing `compose.yml` without breaking any other service. The patterns matter more than the diff: name a private network, give each datastore its own healthcheck, declare `depends_on` with explicit conditions, scope environment variables per service, and avoid port collisions. We covered the primitives in Week 1 (`day-2-local-orchestration-with-docker-compose`, `day-2-healthchecks-and-service-dependency-conditions`); today is the integration exercise.

## Before/After Topology

**Before (end of Day 9):**

```
networks: [pep-net]

services:
  postgres            (5432)  — user, auth data
  mongo               (27017) — questions, tests
  minio               (9000)  — object storage
  user-service        (8001)
  question-management-service (8002)
  test-management-service     (8003)
  api-gateway         (8080)  — public entry
  web                 (3000)  — Next.js
```

**After (end of Day 10):**

```
networks: [pep-net]

services:
  postgres            (5432)
  reporting-postgres  (5433)   ← new, separate DB for reporting
  mongo               (27017)
  minio               (9000)
  user-service        (8001)
  question-management-service (8002)
  test-management-service     (8003)
  reporting-and-analytics-service (8004)   ← new
  api-gateway         (8080)
  web                 (3000)
```

Why a *separate* Postgres for reporting? Service ownership of data. Reporting will eventually hold derived/aggregated data with different access patterns; coupling it to the user-service DB now becomes a hairy split later. Cheap to separate now, expensive to separate later.

## The Diff: Adding `reporting-postgres`

Insert this in the `services:` block alongside the existing `postgres`:

```yaml
  reporting-postgres:
    image: postgres:16-alpine
    container_name: reporting-postgres
    environment:
      POSTGRES_USER: reporting
      POSTGRES_PASSWORD: reporting        # local-only credential
      POSTGRES_DB: reporting
    volumes:
      - reporting-pg-data:/var/lib/postgresql/data
    ports:
      - "5433:5432"                       # host:container — avoid colliding with 5432
    networks: [pep-net]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U reporting -d reporting"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 5s
    restart: unless-stopped
```

And in the `volumes:` block at the bottom:

```yaml
volumes:
  postgres-data:
  reporting-pg-data:    # ← new
  mongo-data:
  minio-data:
```

**Why a different host port (5433)?** The existing `postgres` already maps to 5433? No — to 5432. Map this one to 5433 so `psql -h localhost -p 5433` reaches reporting and -p 5432 reaches the user DB. Inside the network both containers still use 5432; the host port is only for developer convenience.

## The Diff: Adding `reporting-and-analytics-service`

Insert alongside the other backend services:

```yaml
  reporting-and-analytics-service:
    build:
      context: ./services/reporting-and-analytics-service
      dockerfile: Dockerfile
    image: pep/reporting-and-analytics-service:dev
    container_name: reporting-and-analytics-service
    environment:
      REPORTING_ENV: local
      REPORTING_LOG_LEVEL: INFO
      REPORTING_DATABASE_URL: postgresql+asyncpg://reporting:reporting@reporting-postgres:5432/reporting
    ports:
      - "8004:8000"
    networks: [pep-net]
    depends_on:
      reporting-postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/healthz').status==200 else 1)"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 10s
    restart: unless-stopped
    # Optional: run migrations on startup for local dev convenience.
    # In prod we run migrations as a separate task; see day-10 alembic topic.
    command: >
      sh -c "alembic upgrade head &&
             uvicorn app.main:app --host 0.0.0.0 --port 8000"
```

Then (optionally) update `api-gateway` to know about the new downstream once it has real endpoints:

```yaml
  api-gateway:
    # ... existing config ...
    environment:
      # ... existing env ...
      REPORTING_SERVICE_URL: http://reporting-and-analytics-service:8000
    depends_on:
      reporting-and-analytics-service:
        condition: service_healthy
```

For today, the gateway doesn't have to depend on reporting yet — the service has only `/healthz`. Leave the `depends_on` update for when there's a real route to proxy.

## Patterns Worth Naming

### 1. One Network for the Whole Stack

`pep-net` is the user-defined bridge network all services share. Compose creates one automatically, but naming it explicitly:

- Makes Compose attach overrides predictable.
- Lets you connect external containers for debugging (`docker run --rm -it --network fdepep_pep-net ...`).
- Documents intent.

### 2. Service-to-Service Communication Uses Service Names

Inside `pep-net`, the hostname for a service is its Compose service name: `reporting-postgres`, `api-gateway`, etc. Always use the service name in URLs, never `localhost` (which means "this container") and never the host IP (changes per machine).

Wrong:
```
REPORTING_DATABASE_URL=postgresql://reporting:reporting@localhost:5433/reporting
```

Right:
```
REPORTING_DATABASE_URL=postgresql+asyncpg://reporting:reporting@reporting-postgres:5432/reporting
```

### 3. Healthchecks Gate Dependencies

`depends_on: { reporting-postgres: { condition: service_healthy } }` makes Compose wait for Postgres's healthcheck to pass before starting reporting. Without `condition: service_healthy`, Compose only waits for the container to start — not for Postgres to accept connections. The app would crash on its first connection attempt.

Conditions available:

- `service_started` — container is up (default).
- `service_healthy` — healthcheck passes.
- `service_completed_successfully` — for one-shot jobs like migrations.

Use `service_healthy` for every datastore dependency.

### 4. Environment Variables Are Service-Scoped

Each service has its own `environment:` block. Don't use top-level `env_file:` for shared config — it leaks secrets to services that shouldn't see them. If multiple services need the same value, repeat it or use an `x-common-env: &common-env` YAML anchor:

```yaml
x-common-env: &common-env
  LOG_LEVEL: INFO
  TZ: UTC

services:
  reporting-and-analytics-service:
    environment:
      <<: *common-env
      REPORTING_DATABASE_URL: ...
```

### 5. Port Mappings Are Host-Only

Inside the network, services reach each other on container ports (8000, 5432). `ports:` only matters for host access. Map each service to a unique host port to avoid collisions; pick a convention (8001 user, 8002 questions, 8003 tests, 8004 reporting, 8080 gateway).

### 6. Volumes Are Named, Not Bind-Mounted (for data)

`reporting-pg-data:/var/lib/postgresql/data` uses a named volume — managed by Docker, persists across `compose down`, deleted by `compose down -v`. Don't bind-mount `./pg-data:/var/lib/postgresql/data` for databases; Postgres on bind mounts has permission and performance issues on macOS/Windows.

Source code is the exception — bind-mount it during development for hot reload:

```yaml
    volumes:
      - reporting-pg-data:/var/lib/postgresql/data    # named volume
      # - ./services/reporting-and-analytics-service/app:/app/app   # bind, dev-only
```

## Verifying the Composition

```bash
# Bring it up
docker compose up -d --build reporting-postgres reporting-and-analytics-service

# Watch healthchecks resolve
docker compose ps reporting-postgres reporting-and-analytics-service

# Hit the healthz from the host
curl -s http://localhost:8004/healthz | jq .

# Hit it from inside the network (proves service discovery)
docker compose exec api-gateway python -c \
  "import urllib.request; print(urllib.request.urlopen('http://reporting-and-analytics-service:8000/healthz').read())"

# Confirm nothing else regressed
./scripts/smoke-question-authoring.sh
```

## Common Pitfalls

- **Port collision** on the host (5432 vs 5432). Always pick a new host port for added databases.
- **Forgetting `condition: service_healthy`.** App boots, immediately crashes on DB connect; restart loops for ~10s until DB is ready.
- **Using `localhost` inside a container.** It means "this container," not "this machine."
- **`depends_on` doesn't propagate transitively.** If A depends on B and B depends on C, A doesn't wait for C unless A says so. Be explicit.
- **Hard-coding credentials per service.** Even for local, use env vars so the pattern carries to ECS task definitions.
- **`docker compose down` (without `-v`).** Data persists. To reset: `docker compose down -v`.
- **Running migrations in the app's `command:` in prod.** Fine for local convenience; in prod, run migrations as a separate task that completes before the app starts (so a failed migration doesn't crash-loop the app).

## Key Takeaways
- Adding a service is "one block in `services:`, one block in `volumes:` if it's stateful, one entry in the network."
- Service names are hostnames inside the Compose network — never use `localhost` for service-to-service calls.
- `depends_on` with `condition: service_healthy` is what makes startup reliable.
- Pick a host-port convention for each service so collisions are visible at review time.
- Named volumes for data, bind mounts only for source-code hot reload.

---
*Prerequisites: day-2-local-orchestration-with-docker-compose, day-2-healthchecks-and-service-dependency-conditions, day-10-fastapi-service-scaffolding-conventions.*
