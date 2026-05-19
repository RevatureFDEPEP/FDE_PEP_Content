# FastAPI Routing and Dependency Injection Patterns

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The quiz-taking slice lives in `test-management-service`, and it's where FastAPI's `Depends()` system stops being a convenience and starts being load-bearing. Every endpoint today needs an authenticated user, a database session, and shared validation logic — and we want those three things wired the same way in every route, testable in isolation, and easy to override in tests. This topic shows how to structure routers, providers, and dependency overrides so the rest of Week 3 doesn't drown in boilerplate.

## Routers as the Unit of Composition

The substrate convention (see `day-10-fastapi-service-scaffolding-conventions`) puts one `APIRouter` per resource in `app/routers/`. For Week 3 we'll add:

```
services/test-management-service/app/routers/
├── __init__.py
├── health.py
├── sessions.py        # NEW today
└── attempts.py        # added Day 12
```

`app/main.py` wires them in:

```python
# app/main.py
from fastapi import FastAPI
from .routers import health, sessions

app = FastAPI(title="test-management-service", version="0.2.0")
app.include_router(health.router)
app.include_router(sessions.router, prefix="/sessions", tags=["sessions"])
```

Two conventions worth preserving:

- **Prefix lives at the `include_router` call, not the router itself.** Easier to remount under a different prefix during testing or API-version bumps.
- **One tag per router.** Keeps the generated OpenAPI doc readable.

## `Depends()` — What It Actually Does

`Depends(callable)` runs `callable` for every request that hits a route declaring it, caches the result for the lifetime of that request, and injects the return value into the handler. The callable can be sync or async, can itself declare `Depends(...)` parameters, and can be a generator (for setup/teardown).

That last property is how we hand out DB sessions.

## The `deps.py` Module

By convention, dependency providers live in `app/deps.py`:

```python
# app/deps.py
from typing import Annotated
from fastapi import Depends, Header, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from .db import SessionLocal
from .settings import settings


async def get_db() -> AsyncSession:
    async with SessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise


async def get_current_user(
    authorization: Annotated[str | None, Header()] = None,
) -> "CurrentUser":
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Missing or malformed Authorization header",
            headers={"WWW-Authenticate": "Bearer"},
        )
    token = authorization.removeprefix("Bearer ")
    # In the PEP substrate the user-service validates tokens; for now decode locally.
    user = await _validate_token(token)
    if user is None:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")
    return user


# Type aliases to keep handler signatures readable
DB = Annotated[AsyncSession, Depends(get_db)]
CurrentUserDep = Annotated["CurrentUser", Depends(get_current_user)]
```

The `Annotated[..., Depends(...)]` form is the modern style; it lets us declare `DB` and `CurrentUserDep` once and reuse them everywhere without repeating `= Depends(get_db)` in every signature.

## A Route That Uses the Wiring

```python
# app/routers/sessions.py
from fastapi import APIRouter, status

from ..deps import DB, CurrentUserDep
from ..schemas.sessions import SessionCreateRequest, SessionCreateResponse
from ..services import session_service

router = APIRouter()


@router.post(
    "",
    response_model=SessionCreateResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_session(
    body: SessionCreateRequest,
    db: DB,
    user: CurrentUserDep,
) -> SessionCreateResponse:
    return await session_service.create_session(db=db, user=user, request=body)
```

Three things to notice:

- Handler has **no** auth or DB-session boilerplate; it reads as a contract.
- The handler delegates to a service function — endpoints stay thin (see `day-8` Pydantic/repo split).
- `response_model` triggers Pydantic serialization on the way out and shapes the OpenAPI doc.

## Sub-Dependencies and Shared Validation

When two endpoints need the same lookup ("fetch this session by ID, 404 if missing, 403 if not yours") promote it to a dependency:

```python
# app/deps.py
from uuid import UUID

async def get_owned_session(
    session_id: UUID,
    db: DB,
    user: CurrentUserDep,
) -> "Session":
    s = await session_repo.get(db, session_id)
    if s is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Session not found")
    if s.user_id != user.id:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Not your session")
    return s


OwnedSession = Annotated["Session", Depends(get_owned_session)]
```

Now `GET /sessions/{session_id}`, `POST /sessions/{session_id}/answer`, and `POST /sessions/{session_id}/submit` all share one lookup + one authorization check.

Note that `get_owned_session` itself declares `Depends(get_db)` and `Depends(get_current_user)` — FastAPI walks the graph, resolves each once per request, and caches results.

## Dependency Caching Within a Request

By default, the **same dependency callable is resolved once per request**, even if multiple handlers/sub-dependencies ask for it. That's why three sub-deps asking for `get_db` all see the same `AsyncSession` — exactly what you want for transactional coherence within one HTTP request.

If you need a fresh instance per call (rare), pass `Depends(get_thing, use_cache=False)`.

## Overriding Deps in Tests

This is the killer feature. Tests don't need a real DB or a real auth service — they swap providers:

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.deps import get_db, get_current_user

@pytest.fixture
def fake_user():
    return CurrentUser(id="user-1", roles=["candidate"])

@pytest.fixture
def client(fake_user, db_session):
    async def _override_db():
        yield db_session
    app.dependency_overrides[get_db] = _override_db
    app.dependency_overrides[get_current_user] = lambda: fake_user
    yield TestClient(app)
    app.dependency_overrides.clear()
```

Every test now exercises real handler code through real routing, with fakes injected at the dependency boundary. No monkeypatching, no global mutation that leaks across tests (because of the `.clear()` in teardown).

## Worked Scenario: Creating the Session Endpoint Today

For Day 11's `POST /sessions`, the dependency graph is:

```
POST /sessions
└── create_session(body, db, user)
    ├── db    ← get_db        (AsyncSession)
    └── user  ← get_current_user
                └── authorization header
```

That's all the DI you need to start. By Day 12 we'll add `get_owned_session`; by Day 14 we'll add a `get_server_now()` dep so the timer-related code is testable without freezegun voodoo.

## Anti-Patterns

- **Doing auth checks inline in the handler.** Pull them into `get_current_user` (and `get_owned_session`). One bug fix updates every route.
- **Creating a DB session in the handler with `async with SessionLocal()`.** Bypasses dependency overrides; tests can't inject a fake.
- **Repeating `= Depends(get_db)` in every signature.** Use the `Annotated` alias.
- **Putting business logic in the route function.** Routes should be three lines: validate, call service, return.
- **Forgetting `dependency_overrides.clear()` in test teardown.** Overrides leak between tests; debugging time disappears.

## Key Takeaways
- `APIRouter` per resource; prefixes applied at `include_router` time for remount flexibility.
- Dependency providers live in `app/deps.py`; expose `Annotated` aliases (`DB`, `CurrentUserDep`) so handler signatures stay short.
- Sub-dependencies (`get_owned_session`) compose; FastAPI resolves each once per request and caches.
- `app.dependency_overrides` is the test-time seam — use it instead of monkeypatching.
- Handlers stay thin: parse, authorize via deps, delegate to a service module, return a response model.

---
*Prerequisites: day-10-fastapi-service-scaffolding-conventions, day-8 backend topics.*
