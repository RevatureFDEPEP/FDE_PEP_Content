# FastAPI Service Scaffolding Conventions

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Today the cohort creates `reporting-and-analytics-service` from nothing. Rather than reinvent project structure per service, the substrate has a convention every FastAPI service follows — same module layout, same settings handling, same healthcheck path, same Dockerfile shape. Scaffolding to the convention is a productivity move (every team member knows where to look) and an operational move (the same Compose recipe, the same CI matrix entry, the same monitoring config). We're not building reporting features today; we're standing up the empty house so Week 4 can move in.

## What "Scaffold" Means Here

A scaffolded service has:

- A runnable `app` package with `main.py`, settings, db, routers.
- A `/healthz` that returns 200 with no auth, no DB hit.
- A multi-stage Dockerfile that builds with `pip install` and runs as a non-root user.
- A `pyproject.toml` with pinned dependencies.
- An `alembic/` directory (for relational services — see `day-10-alembic-for-relational-schema-evolution`).
- A `tests/` directory with at least one passing test (proves the harness works).
- An entry in `compose.yml` (covered in `day-10-service-composition-patterns-in-docker-compose`).
- An entry in the CI workflow matrix (covered in `day-10-extending-ci-workflows-for-new-tests-and-services`).

That's all. No business logic, no API endpoints beyond healthcheck.

## The Convention: Directory Layout

```
services/reporting-and-analytics-service/
├── app/
│   ├── __init__.py
│   ├── main.py                # FastAPI app, middleware, router includes
│   ├── settings.py            # Pydantic BaseSettings
│   ├── db.py                  # SQLAlchemy engine + session factory
│   ├── models.py              # SQLAlchemy ORM models (empty for now)
│   ├── deps.py                # FastAPI Depends() providers
│   └── routers/
│       ├── __init__.py
│       └── health.py          # /healthz
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_health.py
├── alembic.ini
├── pyproject.toml
├── Dockerfile
└── README.md
```

The other PEP services already use this layout. Symmetry is the point.

## `app/settings.py` — Config via Environment

```python
# app/settings.py
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="REPORTING_")

    service_name: str = "reporting-and-analytics-service"
    env: str = Field(default="local")  # local | staging | prod
    log_level: str = "INFO"

    database_url: str  # e.g. postgresql+asyncpg://user:pw@reporting-postgres:5432/reporting

    # Convenience for tests
    @property
    def is_local(self) -> bool:
        return self.env == "local"

settings = Settings()
```

Convention notes:

- Every service uses `pydantic-settings` BaseSettings with an `env_prefix` namespaced to that service.
- `database_url` is required (no default) — fail fast at startup if missing.
- One module-level `settings` instance; import it where needed.

## `app/db.py` — Async SQLAlchemy

```python
# app/db.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from .settings import settings

engine = create_async_engine(settings.database_url, echo=False, pool_pre_ping=True)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)

async def get_session() -> AsyncSession:
    async with SessionLocal() as session:
        yield session
```

`pool_pre_ping=True` saves you from "MySQL server has gone away"-style stale-connection bugs in long-running containers.

## `app/main.py` — App Factory

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

from .settings import settings
from .routers import health
# from .logging_setup import configure_logging  # see day-10 log correlation

@asynccontextmanager
async def lifespan(app: FastAPI):
    # configure_logging()
    yield
    # graceful shutdown work here if needed

app = FastAPI(
    title=settings.service_name,
    version="0.1.0",
    lifespan=lifespan,
)

app.include_router(health.router)
```

The `lifespan` context replaces the deprecated `@app.on_event("startup")` / `("shutdown")` hooks.

## `app/routers/health.py` — The Healthcheck

```python
# app/routers/health.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/healthz", tags=["health"])
async def healthz():
    return {"status": "ok", "service": "reporting-and-analytics-service"}
```

Conventions:

- Path is always `/healthz` (Kubernetes/ECS love this).
- No auth, no DB hit. A healthcheck that depends on the DB will mark you unhealthy when the DB blips, causing cascading restarts.
- A separate `/readyz` can include DB checks if needed; we don't add one today.

## `tests/test_health.py` — Smoke Test

```python
# tests/test_health.py
from fastapi.testclient import TestClient
from app.main import app

def test_healthz():
    with TestClient(app) as client:
        r = client.get("/healthz")
        assert r.status_code == 200
        assert r.json()["status"] == "ok"
```

One passing test means the test harness works. Future tests anchor here.

## `pyproject.toml` — Pinned Dependencies

```toml
[project]
name = "reporting-and-analytics-service"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi==0.115.0",
    "uvicorn[standard]==0.32.0",
    "pydantic==2.9.2",
    "pydantic-settings==2.6.0",
    "sqlalchemy[asyncio]==2.0.36",
    "asyncpg==0.30.0",
    "alembic==1.14.0",
]

[project.optional-dependencies]
dev = [
    "pytest==8.3.3",
    "pytest-asyncio==0.24.0",
    "httpx==0.27.2",  # for TestClient
    "ruff==0.7.0",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

Pin exact versions; bump deliberately. The CI cache (see Day 3) keys on this file.

## Dockerfile — Multi-Stage Build

```dockerfile
# Dockerfile
# ---- builder ----
FROM python:3.12-slim AS builder
WORKDIR /build
RUN pip install --no-cache-dir --upgrade pip
COPY pyproject.toml ./
RUN pip install --no-cache-dir --prefix=/install . \
 && pip install --no-cache-dir --prefix=/install .[dev]  # remove for prod image variant

# ---- runtime ----
FROM python:3.12-slim AS runtime
RUN useradd --create-home --shell /bin/bash app
WORKDIR /app

COPY --from=builder /install /usr/local
COPY app/ ./app/
COPY alembic/ ./alembic/
COPY alembic.ini ./

USER app
EXPOSE 8000

HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request,sys; \
    sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/healthz').status==200 else 1)"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Multi-stage rationale (covered in `day-2-multi-stage-dockerfiles`): keep build tools out of the runtime image. The healthcheck uses stdlib `urllib` so we don't add `curl` to the image.

## Running It

```bash
cd services/reporting-and-analytics-service

# Local
export REPORTING_DATABASE_URL=postgresql+asyncpg://reporting:reporting@localhost:5433/reporting
uvicorn app.main:app --reload

# Test
pytest

# Build the image
docker build -t reporting-and-analytics-service:dev .
```

Today we'll wire it into Compose (Topic 6) and CI (Topic 7).

## Why the Convention Matters

- **Cognitive load.** New service feels familiar in five seconds.
- **Tooling reuse.** One `Makefile`, one Compose pattern, one CI matrix entry per service.
- **Operability.** Same `/healthz` path means one ALB target group config template, one Datadog monitor template.
- **Reviewability.** Reviewers know what they're looking at; deviations stand out.

When you deviate from the convention, write it down in the service README — *why* the deviation exists. Otherwise the next person normalizes back to the convention and breaks something.

## Anti-Patterns

- **Copy-paste from another service then forget to rename.** `app/main.py` says `title="question-management-service"`. Run a grep for the old name after copying.
- **Adding a DB hit to `/healthz`.** Causes cascading restarts when DB blips. Use `/readyz` instead.
- **Skipping the `tests/test_health.py` placeholder.** First real test takes longer because you're also setting up the harness.
- **Vendoring deps via `requirements.txt` + `pyproject.toml`.** Pick one. The substrate uses `pyproject.toml`.
- **Running as root in the container.** Always `USER app` after copying files.

## Key Takeaways
- Every FastAPI service in the substrate follows the same layout — scaffold to it, don't invent.
- Settings come from env vars via `pydantic-settings`, namespaced per service.
- `/healthz` is auth-free, DB-free, always 200 when the process is alive.
- Multi-stage Dockerfile + non-root user + `pyproject.toml`-pinned deps is the substrate baseline.
- One placeholder test makes the test harness real on day one.

---
*Prerequisites: day-2-multi-stage-dockerfiles, day-8 backend slice topics.*
