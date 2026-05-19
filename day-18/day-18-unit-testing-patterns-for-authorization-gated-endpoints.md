# Unit Testing Patterns For Authorization-Gated Endpoints

> *Day 18: Trainer Dashboard Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The aggregate endpoints from earlier today are protected by a chain of dependencies — `HTTPBearer`, `get_current_user`, `require_trainer` — that raise 401 or 403 depending on what's wrong. The cohort has now written *six* topics' worth of carefully-designed code; the seventh is making sure that code can't quietly regress. The discipline today is the **auth matrix**: every authorization-gated endpoint gets exercised under three conditions — *no token*, *candidate token*, *trainer token* — and the tests assert the right status code in each case. This isn't elaborate; it's eight lines of `pytest.mark.parametrize` and a fixture that mints JWTs. The reason it earns a full topic is that the cohort that doesn't internalize the *matrix* will ship endpoints that pass happy-path tests and silently regress on auth. That's the most common backend regression in real codebases.

## What "Auth Matrix" Means

Three axes, but only the first one always matters:

- **Token state:** no token, expired token, malformed token, valid candidate, valid trainer.
- **Endpoint:** every authorization-gated endpoint.
- **Resource:** when relevant, the same vs different user's data.

The minimum bar PEP enforces is the first axis, applied to every gated endpoint:

| Token | Expected status |
|---|---|
| No `Authorization` header | 401 |
| `Bearer <expired-jwt>` | 401 |
| `Bearer <candidate-jwt>` | 403 |
| `Bearer <trainer-jwt>` | 200 |

Four cases per endpoint. Parametrize and reuse.

## Fixtures: Minting Tokens

The conftest builds the JWT helpers from Topic 4 into reusable fixtures:

```python
# reporting-service/tests/conftest.py
import os
import pytest
import jwt
from datetime import datetime, timezone, timedelta
from httpx import AsyncClient, ASGITransport

os.environ.setdefault("JWT_SECRET", "test-secret-do-not-use-in-prod")
os.environ.setdefault("DATABASE_URL", "postgresql+asyncpg://test:test@localhost/test_reporting")

from app.main import app  # noqa: E402  -- import after env is set

JWT_SECRET = os.environ["JWT_SECRET"]

def _mint(user_id: str, role: str, *, expired: bool = False) -> str:
    now = datetime.now(timezone.utc)
    exp = now - timedelta(minutes=1) if expired else now + timedelta(minutes=30)
    payload = {
        "sub": user_id,
        "role": role,
        "iss": "user-service",
        "iat": int(now.timestamp()),
        "exp": int(exp.timestamp()),
    }
    return jwt.encode(payload, JWT_SECRET, algorithm="HS256")

@pytest.fixture
def trainer_token() -> str:
    return _mint("u_trainer_001", "trainer")

@pytest.fixture
def candidate_token() -> str:
    return _mint("u_candidate_001", "candidate")

@pytest.fixture
def expired_trainer_token() -> str:
    return _mint("u_trainer_001", "trainer", expired=True)

@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
```

A couple of choices worth dwelling on:

- **Env is set *before* importing `app`.** The `JWT_SECRET` is read at import time in `auth.py`; if the fixture imports `app` first, the env override doesn't take effect.
- **`ASGITransport` instead of a live server.** `httpx.AsyncClient(transport=ASGITransport(app=app), ...)` calls the FastAPI app in-process, no HTTP socket needed. Faster, isolated.
- **Fixture per role**, not one parametrized fixture. The tests read more cleanly with named fixtures.

## The Auth Matrix Test

The pattern below is the *single most reused test in the codebase* by the end of the cohort. Write it once cleanly and copy it for each new gated endpoint:

```python
# reporting-service/tests/test_auth_matrix.py
import pytest

GATED_ENDPOINTS = [
    ("GET", "/reports/aggregate?from=2026-04-21&to=2026-05-19"),
    ("GET", "/reports/test/test_python_basics"),
]

@pytest.mark.parametrize("method,url", GATED_ENDPOINTS)
async def test_no_token_returns_401(client, method, url):
    resp = await client.request(method, url)
    assert resp.status_code == 401
    assert "WWW-Authenticate" in resp.headers

@pytest.mark.parametrize("method,url", GATED_ENDPOINTS)
async def test_expired_token_returns_401(client, expired_trainer_token, method, url):
    resp = await client.request(method, url, headers={"Authorization": f"Bearer {expired_trainer_token}"})
    assert resp.status_code == 401
    body = resp.json()
    assert "expired" in body["detail"].lower()

@pytest.mark.parametrize("method,url", GATED_ENDPOINTS)
async def test_candidate_token_returns_403(client, candidate_token, method, url):
    resp = await client.request(method, url, headers={"Authorization": f"Bearer {candidate_token}"})
    assert resp.status_code == 403
    body = resp.json()
    assert "trainer" in body["detail"].lower()

@pytest.mark.parametrize("method,url", GATED_ENDPOINTS)
async def test_trainer_token_returns_200(client, trainer_token, method, url):
    resp = await client.request(method, url, headers={"Authorization": f"Bearer {trainer_token}"})
    assert resp.status_code == 200
```

Four test functions, each parametrized across every gated endpoint. Add a new endpoint to the list → all four checks run on it for free. That's the auth matrix discipline.

A subtle but important property: **the test asserts the status code *and* something about the body.** A coding mistake that returns 401 with body `"some unrelated error"` would slip past a status-only assertion. Asserting `"expired"` in the body or `"trainer"` in the body proves the right code path raised the right error.

## Garbage Token Cases

A more paranoid suite covers malformed tokens too. Stack these as additional matrix cases:

```python
MALFORMED_TOKENS = [
    "",                                  # empty
    "not-a-jwt",                         # not three segments
    "a.b.c",                             # three garbage segments
    "Bearer abc.def.ghi",                # duplicated "Bearer" inside the credential
]

@pytest.mark.parametrize("method,url", GATED_ENDPOINTS)
@pytest.mark.parametrize("token", MALFORMED_TOKENS)
async def test_malformed_token_returns_401(client, method, url, token):
    resp = await client.request(method, url, headers={"Authorization": f"Bearer {token}"})
    assert resp.status_code == 401
```

This is also where the cohort proves PyJWT's `algorithms=[...]` argument is doing its job — try minting a token with `alg: "none"`:

```python
def _mint_alg_none(user_id: str, role: str) -> str:
    payload = {"sub": user_id, "role": role, "exp": 9999999999}
    return jwt.encode(payload, key="", algorithm="none")

async def test_alg_none_rejected(client):
    token = _mint_alg_none("u_x", "trainer")
    resp = await client.get(
        "/reports/aggregate?from=2026-04-21&to=2026-05-19",
        headers={"Authorization": f"Bearer {token}"},
    )
    assert resp.status_code == 401
```

If this test ever passes with a 200, the `algorithms=["HS256"]` argument in the decode call was deleted or the version of PyJWT regressed. That assertion is worth its weight.

## Happy-Path Body Assertions

The trainer-token-returns-200 test asserts the status only. A separate suite — running against the same fixtures — asserts the response shape:

```python
async def test_aggregate_response_shape(client, trainer_token, seeded_db):
    resp = await client.get(
        "/reports/aggregate?from=2026-04-21&to=2026-05-19",
        headers={"Authorization": f"Bearer {trainer_token}"},
    )
    assert resp.status_code == 200
    body = resp.json()
    assert "axes" in body
    assert "rows" in body
    assert "meta" in body
    assert isinstance(body["rows"], list)
    if body["rows"]:
        row = body["rows"][0]
        assert {"test_id", "attempts", "candidates", "avg_pct", "pass_rate"} <= row.keys()
```

Two suites, two concerns:

- **`test_auth_matrix.py`** — does the gate work? (Status codes only.)
- **`test_aggregate_endpoint.py`** — does the happy path return the right data? (Shape, values, ordering.)

Keeping them separate makes failures easier to localize.

## Seeded Test Data

The happy-path tests need data in the database. The PEP pattern (carried from D12 onwards) is a `seeded_db` fixture:

```python
@pytest.fixture
async def seeded_db(db_session):
    await db_session.execute(insert(Session), [
        {"id": "s1", "user_id": "u1", "test_id": "t1", "score": 8, "max_score": 10,
         "started_at": datetime(2026, 5, 1, 10), "submitted_at": datetime(2026, 5, 1, 10, 30),
         "status": "submitted"},
        {"id": "s2", "user_id": "u2", "test_id": "t1", "score": 6, "max_score": 10,
         "started_at": datetime(2026, 5, 2, 11), "submitted_at": datetime(2026, 5, 2, 11, 30),
         "status": "submitted"},
        # ... enough rows to exercise group-by and at least one HAVING filter
    ])
    await db_session.commit()
    yield db_session
```

Rule of thumb: seed *enough rows to exercise the aggregation*, not a single happy-path row. A `GROUP BY test_id` query with one row in the database always returns one row; that doesn't tell you the grouping logic is right. Two test_ids and three users each gives the aggregates something to actually aggregate.

## Authorization-Specific Edge Cases

A few cohort-relevant scenarios that the matrix doesn't naturally cover:

- **Trainer JWT but the trainer's id no longer exists in the user-service.** PEP doesn't gate on user existence today (the JWT is trusted alone) — flag this as a Phase 2 consideration; a test isn't required.
- **JWT with role claim missing entirely.** Topic 4's `options={"require": ["role"]}` makes the decode raise. Add a test that mints a token without `role` and confirms 401.
- **JWT with role claim set to an unknown value.** Topic 4 explicitly handles this — the Pydantic `User` model's `Literal` validation rejects it. Add a test minting `"role": "admin"` and confirming 401.

These three tests together cover the realistic ways a JWT can be "valid but wrong" without forging a signature.

## Running The Tests

```bash
# From reporting-service/
docker compose -f docker-compose.test.yml up -d postgres
pytest -v tests/test_auth_matrix.py tests/test_aggregate_endpoint.py
```

PEP's test setup uses a separate test database (`docker-compose.test.yml` from D7). The matrix runs in under a second once fixtures warm up; this is by design — these tests must be cheap enough that every CI run executes them.

CI extension (the D7/D10 GitHub Actions workflow): add `pytest reporting-service/tests/` to the existing matrix step. The auth matrix runs on every PR.

## Anti-Patterns

- **Testing only the happy path.** "It returns 200 for a trainer" tests one quadrant of the matrix. The 401 and 403 quadrants are where real auth bugs live.
- **One fixture that returns a different token based on a parameter.** Reads worse than three named fixtures. Optimize for test legibility.
- **Hardcoding the JWT in the test file.** Tokens carry `exp`; a hardcoded token expires and the test starts failing intermittently. Always mint freshly.
- **Sharing the production `JWT_SECRET` with tests.** Use a separate test secret; if a test secret leaks into prod by accident, prod is still safe.
- **Asserting only the status code on 403 / 401.** A test that passes when the wrong endpoint returns 403 for an unrelated reason is a false positive. Assert something about the body too.
- **Skipping the alg-none test "because PyJWT handles it."** Library defaults change; the test is the contract that says "we still reject this." It costs nothing to keep.
- **Tests that share state.** A test that creates a session and a later test that reads it depends on test order. Fixture-scoped, transaction-wrapped, or truncate-on-teardown — pick one and stick to it.
- **No CI on the auth tests.** If they only run locally, they will rot.

## Key Takeaways

- The minimum auth matrix is **no token → 401, expired token → 401, candidate token → 403, trainer token → 200**, applied to every gated endpoint via `parametrize`.
- Fixtures mint JWTs against a known test secret in `conftest.py`; never reuse the production secret.
- Use `ASGITransport(app=app)` with `httpx.AsyncClient` for in-process FastAPI testing; no real HTTP socket needed.
- Assert *both* the status code and a body marker — status-only assertions miss "right code, wrong reason" bugs.
- Include explicit tests for the alg-confusion attack and missing/unknown role claims; these regressions are silent.
- Keep auth tests separate from happy-path tests; run them on every CI build.

---
*Prerequisites: day-18-jwt-claim-verification-and-role-enforcement, day-18-role-based-authorization-at-the-api-layer, day-16-unit-testing-patterns-for-read-endpoints, day-7-robust-ci-pipelines.*
