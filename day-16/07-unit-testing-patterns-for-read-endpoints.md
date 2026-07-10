# Unit Testing Patterns For Read Endpoints (pytest)

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D12 covered pytest deeply for the write-path: scoring algorithms, race conditions, locking. The drama there was concurrency and state. Read endpoints have less drama but are not less important to test: a query that silently returns the wrong rows, omits null cases, or breaks pagination math is a bug the candidate sees on their own report and trusts as truth. The cohort's first read-endpoint tests today follow a pattern they'll reuse on D18 (trainer dashboard) and Phase 2 (every read API they ever write): seed a controlled database state, hit the endpoint with httpx, assert on shape *and* content *and* pagination meta. D15's integration testing patterns apply directly; today is the targeted, per-endpoint version.

## What "Unit" Means Here

For a FastAPI endpoint backed by SQL, "unit test" is a slightly fuzzy term. In strict terminology these are **integration tests** — they hit a real DB. In FastAPI/pytest practice the term "unit test" often gets stretched to mean "tests one endpoint in isolation, with all its dependencies wired to test doubles or test data" — distinct from end-to-end Playwright tests that drive the browser.

This topic uses "unit test" in that loose sense: tests of one endpoint, against a real (test-isolated) database, with seeded data, asserting on the HTTP response. They're fast (milliseconds), they're hermetic (no other services running), and they're the foundation of the test pyramid for a backend service.

## The Test Fixture Layer

A good test for `GET /reports/user/{id}` needs:

1. A test database with the right schema (Topic 6's CI handles this; locally, Compose Postgres + Alembic).
2. An async SQLAlchemy session pointed at it, with each test wrapped in a transaction that rolls back at the end.
3. Seeded data — a candidate with some attempts and answers — that's known and small enough to assert against precisely.
4. An httpx client wired to the FastAPI app via ASGI transport (no port, no network).

The fixtures, in `tests/conftest.py`:

```python
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.deps import get_test_mgmt_db
from app.external.test_management_db import Session as Attempt, Answer

TEST_DATABASE_URL = "postgresql+asyncpg://reporting:reporting@localhost:5432/test_mgmt_test"

@pytest_asyncio.fixture
async def db_engine():
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)
    yield engine
    await engine.dispose()

@pytest_asyncio.fixture
async def db_session(db_engine):
    """Each test runs in a transaction that rolls back at the end."""
    async with db_engine.connect() as conn:
        trans = await conn.begin()
        SessionMaker = async_sessionmaker(bind=conn, expire_on_commit=False)
        async with SessionMaker() as session:
            yield session
        await trans.rollback()

@pytest_asyncio.fixture
async def client(db_session):
    """FastAPI app with the DB session dependency overridden to the test session."""
    async def _override():
        yield db_session
    app.dependency_overrides[get_test_mgmt_db] = _override
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

A few design choices to drill:

- **`db_session` rolls back at the end.** No cleanup needed between tests; data inserted in one test is invisible to the next. This is the *most important* fixture in the suite — without it, tests have order-dependent failures that take hours to debug.
- **`dependency_overrides` swaps the production DB session for the test one.** The route handler's `db: TestMgmtDep` resolves to the test session via this override. No global state, no monkeypatching.
- **`ASGITransport` runs FastAPI in-process.** No port binding, no socket; faster, more reliable, and parallelizable.
- **`app.dependency_overrides.clear()` on teardown.** Otherwise overrides leak across tests in surprising ways.

## Seeding Helpers

Tests should *not* repeat fifty lines of "create candidate, create attempt, create answers" boilerplate. Factor:

```python
# tests/factories.py
from datetime import datetime, timezone, timedelta
import uuid

def make_attempt(
    *,
    user_id: str = "u_test",
    test_id: str = "test_t1",
    score: int | None = 8,
    started_at: datetime | None = None,
    submitted_at: datetime | None = None,
    status: str = "submitted",
) -> dict:
    started = started_at or datetime.now(timezone.utc) - timedelta(minutes=15)
    submitted = submitted_at if submitted_at is not None else started + timedelta(minutes=12)
    return {
        "id": uuid.uuid4(),
        "user_id": user_id,
        "test_id": test_id,
        "score": score,
        "max_score": 10,
        "started_at": started,
        "submitted_at": submitted,
        "expires_at": started + timedelta(hours=1),
        "locked_at": submitted,
        "status": status,
    }

async def seed_attempts(db, attempts: list[dict]) -> list[uuid.UUID]:
    for a in attempts:
        await db.execute(Attempt.__table__.insert().values(**a))
    await db.flush()
    return [a["id"] for a in attempts]
```

The factory uses **sensible defaults plus keyword overrides** — the test writes only the fields it cares about, and the rest are filled in. This pattern (factory_boy in spirit, hand-rolled here for simplicity) is what separates tests that read like specifications from tests that read like database scripts.

## The First Test: Empty State

The simplest meaningful test of a read endpoint is the empty case:

```python
import pytest

@pytest.mark.asyncio
async def test_user_report_empty_state(client):
    r = await client.get("/reports/user/u_nonexistent_yet")
    assert r.status_code == 200
    body = r.json()
    assert body["user_id"] == "u_nonexistent_yet"
    assert body["summary"]["total_attempts"] == 0
    assert body["summary"]["completed_attempts"] == 0
    assert body["summary"]["average_score"] is None
    assert body["recent_attempts"] == []
```

This locks in the Topic 1 contract: *empty data is 200 with an empty shape, not 404*. The frontend's empty-state UI depends on this distinction.

## The Happy-Path Test

```python
@pytest.mark.asyncio
async def test_user_report_with_attempts(client, db_session):
    user_id = "u_42"
    await seed_attempts(db_session, [
        make_attempt(user_id=user_id, test_id="t_python", score=8),
        make_attempt(user_id=user_id, test_id="t_python", score=9),
        make_attempt(user_id=user_id, test_id="t_js",     score=6),
    ])

    r = await client.get(f"/reports/user/{user_id}")
    assert r.status_code == 200
    body = r.json()
    assert body["summary"]["total_attempts"] == 3
    assert body["summary"]["completed_attempts"] == 3
    assert body["summary"]["average_score"] == pytest.approx(7.666, rel=0.01)
    assert body["summary"]["best_score"] == 9
    assert len(body["recent_attempts"]) == 3
```

Three attempts seeded; three attempts in the response. Average score asserted via `pytest.approx` because float comparison with exact equality is brittle. Notice the test doesn't care about ordering yet — that's a separate test.

## The Ordering Test

The contract says recent_attempts comes back in reverse chronological order. Test it explicitly:

```python
@pytest.mark.asyncio
async def test_recent_attempts_ordered_by_recency(client, db_session):
    user_id = "u_43"
    base = datetime.now(timezone.utc)
    await seed_attempts(db_session, [
        make_attempt(user_id=user_id, started_at=base - timedelta(days=3)),
        make_attempt(user_id=user_id, started_at=base - timedelta(days=1)),
        make_attempt(user_id=user_id, started_at=base - timedelta(days=2)),
    ])

    r = await client.get(f"/reports/user/{user_id}")
    submitted_ats = [a["submitted_at"] for a in r.json()["recent_attempts"]]
    assert submitted_ats == sorted(submitted_ats, reverse=True)
```

Insert in a non-sorted order on purpose — that way the test would pass if "the database happens to return them in insert order"; a real `ORDER BY` is required to make it work. This is the kind of test that catches the case where the `ORDER BY` is silently missing.

## The "Other Users Don't Leak" Test

A reporting endpoint must scope to the requested user. The cohort *will* forget the `WHERE user_id = :user_id` once during the cohort; this test catches it:

```python
@pytest.mark.asyncio
async def test_report_isolates_users(client, db_session):
    await seed_attempts(db_session, [
        make_attempt(user_id="u_alice", score=8),
        make_attempt(user_id="u_bob",   score=6),
    ])

    r = await client.get("/reports/user/u_alice")
    body = r.json()
    assert body["summary"]["total_attempts"] == 1
    assert body["recent_attempts"][0]["score"] == 8
    # Bob's attempt must not appear
    assert all(a["score"] != 6 for a in body["recent_attempts"])
```

If the route handler forgot the `WHERE`, this test produces `total_attempts == 2` and a clear failure. Every reporting endpoint should have an analogous isolation test.

## Pagination Tests

The `/attempts` list endpoint (Topic 3) needs its own suite:

```python
@pytest.mark.asyncio
async def test_pagination_meta(client, db_session):
    user_id = "u_paginate"
    await seed_attempts(db_session, [
        make_attempt(user_id=user_id) for _ in range(25)
    ])

    # Page 1, size 10
    r = await client.get(f"/reports/user/{user_id}/attempts?page=1&size=10")
    assert r.status_code == 200
    body = r.json()
    assert len(body["items"]) == 10
    assert body["meta"] == {
        "page": 1, "size": 10, "total": 25,
        "total_pages": 3, "has_next": True, "has_prev": False,
    }

    # Page 3 — partial page
    r = await client.get(f"/reports/user/{user_id}/attempts?page=3&size=10")
    body = r.json()
    assert len(body["items"]) == 5
    assert body["meta"]["has_next"] is False
    assert body["meta"]["has_prev"] is True

@pytest.mark.asyncio
async def test_size_cap(client):
    r = await client.get("/reports/user/u_x/attempts?size=1000")
    assert r.status_code == 422   # Pydantic validation, not 200

@pytest.mark.asyncio
async def test_invalid_sort_column(client):
    r = await client.get("/reports/user/u_x/attempts?sort=__table__:asc")
    assert r.status_code == 422   # regex on sort param catches it
```

Three things asserted:

- **Meta math is correct** at boundaries (page 1, partial-page last page).
- **Size cap is enforced at the framework layer** (422, not silently capped).
- **Sort column allowlisting works** — a malicious sort key is rejected, not used to probe internal state.

The "page 3 partial-page" test is the one that catches off-by-one errors. Make it a habit.

## Filtering Tests

```python
@pytest.mark.asyncio
async def test_filter_by_test_id(client, db_session):
    user = "u_44"
    await seed_attempts(db_session, [
        make_attempt(user_id=user, test_id="t_python"),
        make_attempt(user_id=user, test_id="t_python"),
        make_attempt(user_id=user, test_id="t_js"),
    ])
    r = await client.get(f"/reports/user/{user}/attempts?test_id=t_python")
    items = r.json()["items"]
    assert len(items) == 2
    assert all(i["test_id"] == "t_python" for i in items)

@pytest.mark.asyncio
async def test_filter_by_date_range(client, db_session):
    user = "u_45"
    base = datetime(2026, 5, 1, tzinfo=timezone.utc)
    await seed_attempts(db_session, [
        make_attempt(user_id=user, started_at=base + timedelta(days=i),
                     submitted_at=base + timedelta(days=i, minutes=10))
        for i in range(10)
    ])
    r = await client.get(
        f"/reports/user/{user}/attempts?from=2026-05-03&to=2026-05-07"
    )
    assert len(r.json()["items"]) == 5
```

Date-range tests are *especially* worth writing because the date-vs-timestamp inclusivity footgun from Topic 3 only shows up under test. If the cohort wrote `<= to_date` instead of `< to_date + 1 day`, this test reveals it (likely by returning 4 instead of 5).

## Shape Tests Via Pydantic

For the response shape, validating it conforms to the response model is a one-liner that catches regressions if someone edits the handler and forgets to update the model:

```python
from app.schemas import UserReportResponse

@pytest.mark.asyncio
async def test_response_shape_conforms(client, db_session):
    await seed_attempts(db_session, [make_attempt(user_id="u_shape")])
    r = await client.get("/reports/user/u_shape")
    # Will raise ValidationError if the response shape drifts from the schema
    UserReportResponse.model_validate(r.json())
```

Cheap, automatic regression coverage on shape.

## Common Test-Writing Pitfalls For Cohort

A few things the trainer should watch for during in-class review:

- **Tests that pass without the production code being correct.** "If I delete the WHERE clause, does my test still pass?" If yes, the test isn't testing what it claims. The "other users don't leak" test above is the antidote.
- **Asserting on `len(items)` without asserting on *which* items.** A test that passes "returned 3 items" but those are the wrong 3 items is broken. Assert on content too.
- **Forgetting to await async fixtures.** `pytest-asyncio` is forgiving but not infinitely; missing awaits produce confusing errors. The `@pytest.mark.asyncio` decorator on tests + `pytest_asyncio.fixture` on fixtures is the boilerplate.
- **Polluting the global app across tests.** `dependency_overrides.clear()` matters. If a test crashes mid-run, the next test inherits stale overrides.
- **Time-sensitive tests.** `datetime.now()` in a test compared to `datetime.now()` in the handler will eventually be a few microseconds off and fail. Freeze time (`freezegun`) or compare with tolerance.
- **Hardcoded fixtures of 1000 rows for "performance" tests.** Slow tests; if you need to test performance, do it explicitly with `pytest-benchmark`; otherwise keep fixtures small (single digits to low double digits of rows).

## Connecting To D15

D15's integration testing patterns (real Postgres, real Mongo via Compose) apply here. The fixture stack today is similar; the difference is:

- D15's integration tests start the *whole stack* via Compose and exercise *cross-service* flows (frontend → gateway → test-management → DB).
- Today's unit tests run *one service* in-process and exercise *one endpoint* against a test DB.

Both are valuable. Today's tests are faster, more numerous, and catch endpoint-shape regressions. D15's tests catch wiring and integration issues. The pyramid wants more of today's, fewer of D15's, and the cohort should know which they're writing.

## Anti-Patterns

- **Mocking SQLAlchemy.** Tempting to "mock the DB" with `MagicMock`. The mocks pass; the production code doesn't. Use a real test DB; you have one in CI now.
- **Tests with no assertions.** "It didn't crash" is not a test. `assert r.status_code == 200` is the bare minimum, and you should usually assert more.
- **Tests that share state via class attributes or globals.** Order-dependent failures. Each test's data lives in fixtures, scoped to the test.
- **One mega-test that hits every endpoint.** Fails on the first failure, hides everything else, hard to read. One test per assertion-class is better.
- **Snapshot tests for entire JSON responses.** Brittle — any field reorder fails them. Assert on the fields the contract guarantees, not on the byte-for-byte response.
- **Skipping the empty-state test.** "Obviously empty works" — until it doesn't, and an `IndexError` 500s the endpoint when a real user has no attempts.
- **Tests that pass with the wrong return type.** `assert body["score"]` passes for `"score": "8"` (string) and `"score": 8` (int). Use `assert body["score"] == 8` (typed comparison) when the type matters.
- **Comment-only documentation.** A test named `test_pagination` doesn't tell you what about pagination. `test_pagination_meta_on_partial_last_page` does.

## Key Takeaways

- The `db_session` fixture (transaction-per-test, rolled back at teardown) is the foundation of fast, hermetic, parallel-safe tests for read endpoints.
- `dependency_overrides` swaps production deps for test deps without monkeypatching; clean teardown matters.
- Test what's promised: empty state, happy path, ordering, isolation between users, pagination meta math, filter correctness, response shape.
- Insert in non-sorted order to test that the `ORDER BY` is actually doing something.
- Read-endpoint tests are dozens-of-milliseconds fast and dozens-per-endpoint frequent; lean on them as the primary test layer, with D15-style integration tests as the cross-service backstop.

---
*Prerequisites: [08-ai-assisted-test-authoring-with-parametrized-pytest-cases.md](../day-12/08-ai-assisted-test-authoring-with-parametrized-pytest-cases.md), [05-integration-testing-against-local-data-stores.md](../day-15/05-integration-testing-against-local-data-stores.md), [02-fastapi-routing-and-dependency-injection-patterns.md](../day-11/02-fastapi-routing-and-dependency-injection-patterns.md), [04-filtering-pagination-and-sorting-patterns-in-fastapi.md](04-filtering-pagination-and-sorting-patterns-in-fastapi.md).*
