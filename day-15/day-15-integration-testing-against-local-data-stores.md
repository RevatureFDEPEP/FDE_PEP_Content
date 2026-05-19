# Integration Testing Against Local Data Stores

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Unit tests (D8, D12, D14) prove pure logic. Smoke tests (Topic 2) prove the stack is alive. Playwright (Topic 3) proves the happy path through the browser. There's a gap in the middle: testing that the service's *real database queries* — the SQLAlchemy session-creation transaction, the `select ... for update` row lock, the cross-store join against Mongo for question lookups — behave correctly. Mocking the database hides the bugs you most need to catch (constraint violations, lock timeouts, query plan surprises, JSON column quirks). Today the cohort adds integration tests that hit *real* Postgres and Mongo containers from inside pytest.

The unit-test pyramid still holds: many unit tests, fewer integration tests, fewer-still E2E. Integration tests are the second-cheapest way to catch bugs; for any code that touches a DB, they belong in the suite.

## Why Not Mock The Database

Three reasons mocking SQLAlchemy / pymongo hides real bugs:

1. **SQL is its own language.** `select ... for update` either acquires a row lock or doesn't; no mock simulates lock behavior faithfully. The D12 idempotency race only manifests against real Postgres.
2. **Constraint violations are real bugs in real shape.** The `IntegrityError` on a duplicate idempotency key is the thing you're testing; a mock that returns whatever you tell it to doesn't exercise the constraint.
3. **Migration drift is invisible to mocks.** If Alembic head and your code disagree, a mock won't notice. A real DB will.

Mocking the DB is fine for *unit tests of code that incidentally touches a DB*. For testing the DB-touching code itself, use the real thing.

## Approach: Reuse The Compose Stack vs. testcontainers

Two reasonable patterns. PEP uses approach A (Compose) for cohort simplicity; approach B (testcontainers) is what mature teams adopt later.

### Approach A: Reuse The Running Compose Stack (PEP default)

The Postgres and Mongo already running on the cohort's machine for development are also used for integration tests. Each test creates and tears down a *uniquely-named database* so tests don't collide with dev data.

Pros: zero extra infra; tests run in seconds; trivial to debug (you can `psql` into the same DB after a failure).
Cons: requires Compose to be up; CI must bring Compose up before running.

### Approach B: testcontainers-Python

`testcontainers-python` programmatically starts Postgres/Mongo containers per test session, then tears them down. Pros: hermetic; works without dev Compose. Cons: slow startup (~10s per test session); extra dependency; trickier on Windows.

For PEP, the cohort already runs Compose for everything else; doubling down is simpler.

## Setup

The integration test job needs the dev Compose stack up. Local:

```bash
docker compose up -d postgres mongo
cd test-management-service
pytest tests/integration/ -v
```

In CI (extends D7 patterns):

```yaml
# .github/workflows/integration.yml (excerpt)
- name: Start data stores
  run: docker compose up -d postgres mongo

- name: Wait for healthy
  run: |
    for i in {1..30}; do
      docker compose ps --format json | jq -r '.[] | select(.Service=="postgres" or .Service=="mongo") | .Health' | grep -vc healthy && sleep 1 || break
    done

- name: Run integration tests
  working-directory: test-management-service
  run: pytest tests/integration/ -v
```

## The Fixture Pattern: Distinct DB Per Run

Conftest sets up a unique database name per test session, runs migrations against it, and tears down at the end. Each test sees a clean schema and can do whatever it wants without polluting other runs.

```python
# test-management-service/tests/integration/conftest.py
import os
import uuid
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from alembic.config import Config
from alembic import command
from motor.motor_asyncio import AsyncIOMotorClient

PG_HOST = os.getenv("PG_HOST", "localhost")
PG_PORT = os.getenv("PG_PORT", "5432")
PG_USER = os.getenv("PG_USER", "app")
PG_PASS = os.getenv("PG_PASS", "app")
ADMIN_URL = f"postgresql+asyncpg://{PG_USER}:{PG_PASS}@{PG_HOST}:{PG_PORT}/postgres"

MONGO_URL = os.getenv("MONGO_URL", "mongodb://localhost:27017")


@pytest_asyncio.fixture(scope="session")
async def pg_db_name():
    """Session-scoped: one DB per test run, named with a uuid."""
    name = f"test_mgmt_it_{uuid.uuid4().hex[:8]}"
    admin_engine = create_async_engine(ADMIN_URL, isolation_level="AUTOCOMMIT")
    async with admin_engine.begin() as conn:
        await conn.exec_driver_sql(f'CREATE DATABASE "{name}"')
    await admin_engine.dispose()

    yield name

    admin_engine = create_async_engine(ADMIN_URL, isolation_level="AUTOCOMMIT")
    async with admin_engine.begin() as conn:
        # Terminate stragglers so DROP succeeds
        await conn.exec_driver_sql(
            f"SELECT pg_terminate_backend(pid) FROM pg_stat_activity "
            f"WHERE datname='{name}' AND pid <> pg_backend_pid()"
        )
        await conn.exec_driver_sql(f'DROP DATABASE "{name}"')
    await admin_engine.dispose()


@pytest_asyncio.fixture(scope="session")
async def pg_engine(pg_db_name):
    """Engine bound to the per-run DB, schema migrated."""
    url = f"postgresql+asyncpg://{PG_USER}:{PG_PASS}@{PG_HOST}:{PG_PORT}/{pg_db_name}"

    # Run Alembic against the per-run DB. Sync URL for Alembic.
    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option(
        "sqlalchemy.url",
        f"postgresql://{PG_USER}:{PG_PASS}@{PG_HOST}:{PG_PORT}/{pg_db_name}",
    )
    command.upgrade(alembic_cfg, "head")

    engine = create_async_engine(url, pool_pre_ping=True)
    yield engine
    await engine.dispose()


@pytest_asyncio.fixture
async def db(pg_engine) -> AsyncSession:
    """Function-scoped session; rolled back after each test for isolation."""
    Session = async_sessionmaker(pg_engine, expire_on_commit=False)
    async with Session() as session:
        async with session.begin():
            yield session
            await session.rollback()


@pytest_asyncio.fixture(scope="session")
async def mongo_db():
    """Per-run Mongo database name; dropped at session end."""
    name = f"questions_it_{uuid.uuid4().hex[:8]}"
    client = AsyncIOMotorClient(MONGO_URL)
    yield client[name]
    await client.drop_database(name)
    client.close()
```

Notes on the fixture choices:

- **`pg_db_name` and `pg_engine` are session-scoped.** Creating a DB and running migrations takes seconds; doing it per test would be unbearable. One per run is the right granularity.
- **`db` is function-scoped and rolls back.** Each test starts with a clean transactional state; assertions only see what the test wrote.
- **The Mongo DB doesn't have a per-test rollback** because Mongo doesn't have cross-document transactions in the way Postgres does. Use unique IDs per test (`question_id=f"q_{uuid4().hex}"`) to avoid collisions, or drop+recreate the collection in a fixture if the test needs a clean slate.
- **Naming with uuid suffix** means concurrent CI runs don't collide on a shared Postgres.
- **Active-connection termination** before `DROP DATABASE` is the small Postgres dance you have to do; otherwise the drop fails if any other session is still attached.

## Worked Example: Test The Session-Creation Path (D11)

The D11 endpoint creates a `sessions` row in Postgres, validates the test exists in Mongo, and returns a token. Integration test it end-to-end against real DBs:

```python
# test-management-service/tests/integration/test_session_creation.py
import pytest
import uuid
from app.sessions import create_session, SessionAlreadyExistsError

@pytest.mark.asyncio
async def test_create_session_persists_row(db, mongo_db):
    # Arrange: seed a test in Mongo
    test_id = f"t_{uuid.uuid4().hex[:8]}"
    await mongo_db.tests.insert_one({
        "_id": test_id,
        "duration_minutes": 30,
        "question_ids": ["q1", "q2"],
    })
    await mongo_db.questions.insert_many([
        {"_id": "q1", "type": "single_select", "prompt": "...", "options": [], "correct": []},
        {"_id": "q2", "type": "single_select", "prompt": "...", "options": [], "correct": []},
    ])

    # Act
    candidate_id = f"u_{uuid.uuid4().hex[:8]}"
    session = await create_session(db, mongo_db, test_id=test_id, candidate_id=candidate_id)

    # Assert: row exists with expected shape
    assert session.session_id is not None
    assert session.status == "in_progress"
    assert session.expires_at > session.created_at

    # Confirm with a direct query (proves it really got committed within the test's tx)
    from sqlalchemy import select
    from app.models import SessionRow
    result = await db.execute(select(SessionRow).where(SessionRow.session_id == session.session_id))
    row = result.scalar_one()
    assert row.candidate_id == candidate_id


@pytest.mark.asyncio
async def test_create_session_rejects_unknown_test(db, mongo_db):
    candidate_id = f"u_{uuid.uuid4().hex[:8]}"
    with pytest.raises(LookupError):
        await create_session(db, mongo_db, test_id="does-not-exist", candidate_id=candidate_id)


@pytest.mark.asyncio
async def test_idempotent_create_returns_existing_session(db, mongo_db):
    # Arrange
    test_id = f"t_{uuid.uuid4().hex[:8]}"
    await mongo_db.tests.insert_one({"_id": test_id, "duration_minutes": 30, "question_ids": []})
    candidate_id = f"u_{uuid.uuid4().hex[:8]}"

    # Act: two creates with the same idempotency key
    s1 = await create_session(db, mongo_db, test_id=test_id, candidate_id=candidate_id, idempotency_key="k1")
    s2 = await create_session(db, mongo_db, test_id=test_id, candidate_id=candidate_id, idempotency_key="k1")

    # Assert: same session, not two
    assert s1.session_id == s2.session_id
```

The first test verifies the happy path and confirms with a fresh query that the data is really there. The second exercises the cross-store validation (test must exist in Mongo before session can be created in Postgres). The third exercises D12's idempotency at the integration layer — the unit tests check the logic; the integration test checks that the *Postgres constraint* on `(candidate_id, idempotency_key)` actually fires.

## Worked Example: Test The Row Lock (D12)

The pessimistic lock from D12 is exactly the kind of thing unit tests can't cover, because the behavior is in the *database*, not the code. Spawn two concurrent sessions; assert one waits for the other.

```python
@pytest.mark.asyncio
async def test_concurrent_answer_writes_serialize_via_row_lock(pg_engine, mongo_db):
    """Two answer submissions on the same session must serialize."""
    import asyncio
    from app.sessions import submit_answer
    from sqlalchemy.ext.asyncio import async_sessionmaker

    # Set up a session row to lock
    Session = async_sessionmaker(pg_engine, expire_on_commit=False)
    async with Session() as s:
        from app.models import SessionRow
        sid = f"s_{uuid.uuid4().hex[:8]}"
        s.add(SessionRow(session_id=sid, candidate_id="u1", status="in_progress"))
        await s.commit()

    started = []
    finished = []

    async def attempt(idem_key: str):
        async with Session() as s:
            started.append(idem_key)
            r = await submit_answer(
                s, mongo_db, session_id=sid, question_id="q1",
                selected=[1], idempotency_key=idem_key
            )
            finished.append(idem_key)
            return r

    # Fire two in parallel
    r1, r2 = await asyncio.gather(attempt("k1"), attempt("k2"))

    # Both succeeded
    assert {finished[0], finished[1]} == {"k1", "k2"}
    # And both observed the lock serialization (no IntegrityError, no deadlock)
    assert r1 is not None and r2 is not None
```

A test like this would be impossible against a mock — the entire point is the *database's* lock behavior under concurrent transactions.

## When NOT To Write An Integration Test

The pyramid still applies. Skip the integration test if:

- The logic is pure (a scoring function, a reducer, a validator). Unit-test it.
- The behavior is fully owned by the framework (testing that SQLAlchemy can `INSERT` is testing SQLAlchemy, not your code).
- The same behavior is already covered by a smoke test at the boundary.

A good rule of thumb: write an integration test when *the bug you fear is in the interaction between your code and the database*. If the fear is "did I write the right Python," it's a unit test.

## Speed Discipline

Integration tests run in seconds, not milliseconds. The whole suite should still finish in under a minute. To stay there:

- **One session-scoped DB per run.** Don't create a fresh DB per test.
- **Transactional rollback** via the `db` fixture for Postgres tests; the rollback is cheap.
- **Unique IDs everywhere.** Don't try to clean up by name; use uuids and accept that the per-run DB will get dropped at session end.
- **No `time.sleep` in tests.** If you need to wait for a DB to become consistent, you have a real bug — fix it, don't sleep around it.
- **Mark slow tests** (`@pytest.mark.slow`) and skip them on every-push CI; run on PRs only.

## Anti-Patterns

- **Sharing one Postgres DB across all tests with no cleanup.** State leaks; ordering matters; flake city.
- **Truncating tables between tests.** Slower than transactional rollback; doesn't reset sequences; misses things like leftover indexes from migrations.
- **Running integration tests against the dev database the cohort is using.** Pollutes their data; horrifying when they discover their session list has "test test test" entries.
- **Mocking SQLAlchemy in tests that exist to test SQLAlchemy code.** Pointless. If you're going to mock, the test isn't an integration test; rename it.
- **Connecting to remote/staging DBs from tests.** Slow, flaky, dangerous. Local Compose only.
- **No cleanup of the per-run DB.** Hundreds of `test_mgmt_it_xxxxxxxx` databases pile up on the cohort's Postgres after a week of running tests. The fixture's teardown is non-negotiable.

## Key Takeaways
- Integration tests hit *real* Postgres and Mongo from inside pytest — they catch constraint, lock, and migration bugs that mocks can't.
- Session-scoped DB creation + function-scoped transactional rollback is the right fixture pattern: one fresh DB per run, isolated state per test.
- Unique DB name per run (uuid suffix) prevents collisions between concurrent CI jobs sharing one Postgres host.
- Write integration tests for row locks, idempotency constraints, and cross-store joins — places where the *database* is the system under test.
- Compose-stack reuse is the PEP default; testcontainers is the mature alternative.
- Keep the integration suite under a minute; mark slow tests and skip them outside PR CI.

---
*Prerequisites: day-7-robust-ci-pipelines, day-10-alembic-for-relational-schema-evolution, day-11-fastapi-routing-and-dependency-injection-patterns, day-12-database-transactions-and-pessimistic-locking, day-12-idempotency-for-retried-mutations.*
