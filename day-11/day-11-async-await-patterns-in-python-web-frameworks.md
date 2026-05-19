# Async/Await Patterns in Python Web Frameworks

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
FastAPI lets you write `def` or `async def` handlers and runs both successfully, which is part of the appeal and most of the trouble. The session creation endpoint today calls a downstream HTTP service, hits the DB, and serializes a response — three I/O-bound operations that *should* be concurrent. Done correctly, the worker handles dozens of in-flight requests at once. Done wrong, one blocking call freezes every request on the worker and the cohort never sees the problem until production. This topic is about getting it right: when to use `async def`, what blocks the event loop, and how to call sync code from async code safely.

## The Mental Model in One Paragraph

The Python interpreter running your FastAPI app has a single thread per worker process, running an **event loop**. The loop runs one task to its next `await`, suspends it (the underlying I/O is delegated to the OS), and runs the next ready task. While awaited I/O is in flight, other requests get CPU. If a task fails to `await` — if it does CPU work, a blocking syscall, or a sync DB call — the loop is stuck on *that one task* until it returns. Every other in-flight request waits.

Async correctness is about keeping the loop spinning.

## When to Use `async def`

For PEP, the rule is:

- **Use `async def`** for any handler, dependency, or service function that performs I/O.
- **Use plain `def`** for pure-compute helpers (hashing, sampling, validators).

FastAPI handles both — a `def` handler is automatically run on a thread pool so it won't block the loop. But mixing styles in service code is where the bugs come from. The PEP substrate convention: **`async def` everywhere in handler/service/repo layers; sync `def` only for stateless pure helpers**.

```python
# app/services/session_service.py
async def create_session(db, user, request):       # I/O — async
    ...

def hash_session_token(raw: str) -> str:           # pure compute — sync
    return hashlib.sha256(raw.encode()).hexdigest()
```

## Common Blocking Pitfalls

This list is the one to memorize. Each item blocks the event loop:

| Pitfall | Why it blocks | Fix |
|---|---|---|
| Sync DB driver (`psycopg2`, `pymongo`) | Synchronous `recv()` on the socket | Use `asyncpg`/`SQLAlchemy[asyncio]`/`motor` |
| `requests.get(...)` in async code | Sync `socket.read` | Use `httpx.AsyncClient` (topic 5) |
| `time.sleep(2)` | Sync sleep blocks the thread | `await asyncio.sleep(2)` |
| CPU work (parsing 50MB JSON, image resize, bcrypt) | No `await` point until done | `await asyncio.to_thread(...)` or move to a task queue |
| `subprocess.run()` | Blocks until child exits | `await asyncio.create_subprocess_exec(...)` |
| File I/O on big files | Blocks on `read()`/`write()` | `aiofiles`, or `asyncio.to_thread` |
| Calling `.get()` on a sync future / `concurrent.futures.Future` | Blocks the loop | `await asyncio.wrap_future(fut)` |
| Holding the GIL in a tight Python loop | No yield to the loop | Refactor or `asyncio.to_thread` |

If you see any of these in a code review for an `async def` function, stop and fix.

## The Escape Hatch: `asyncio.to_thread`

Sometimes a useful library only ships a sync API (an old SDK, a CPU-bound parser, a CLI wrapper). Don't rewrite the world — push the call onto a thread:

```python
import asyncio
import hashlib

async def hash_large_blob(data: bytes) -> str:
    # SHA-256 over 50MB is CPU work — push it off the loop
    return await asyncio.to_thread(_sha256_hex, data)


def _sha256_hex(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()
```

`asyncio.to_thread` (Python 3.9+) runs the callable on the default thread pool executor and returns a coroutine you await. The event loop keeps spinning.

Caveats:

- The thread pool defaults to ~32 workers. Sustained heavy use → starvation. For CPU-bound bulk work, a process pool or task queue is more appropriate.
- The GIL still applies — `to_thread` helps with **blocking I/O** disguised as sync, less so with CPU-bound Python.

## Concurrent Calls with `asyncio.gather`

Today's session create makes two independent calls to `question-management-service`: list IDs, then fetch the first question. They're sequential because the second depends on the first. But imagine you also need to fetch the user's prior session count from `reporting-and-analytics-service` — independent of either question call. Run them concurrently:

```python
# illustrative — not in the Day 11 deliverable as-is
import asyncio

ids_task = question_client.list_active_question_ids(quiz_id=request.quiz_id)
prior_count_task = reporting_client.count_sessions(user_id=user.id)

ids, prior_count = await asyncio.gather(ids_task, prior_count_task)
```

Two HTTP calls run in parallel; the handler latency is `max(t1, t2)` instead of `t1 + t2`. The catch: if one raises, `gather` cancels the others (or use `return_exceptions=True` and handle each).

## DB Sessions and Async Context

The Day 10 substrate uses `SQLAlchemy[asyncio]`. The repo and service layer functions are `async def` and `await` the queries:

```python
# app/repos/session_repo.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from ..models.session import SessionRow


async def insert(db: AsyncSession, record: SessionRecord) -> None:
    db.add(SessionRow.from_record(record))
    await db.commit()


async def get(db: AsyncSession, session_id: UUID) -> SessionRow | None:
    result = await db.execute(select(SessionRow).where(SessionRow.id == session_id))
    return result.scalar_one_or_none()
```

The session itself is yielded by an async dependency (`get_db` in the DI topic). Mixing a sync `Session` and async route handlers is the most common substrate bug — pin one style per service.

For Mongo in `question-management-service`, the substrate uses `motor` (the async driver). `pymongo` is sync; don't call it from async handlers.

## Testing Async Code

`pytest-asyncio` with `asyncio_mode = "auto"` (set in `pyproject.toml` per the Day 10 scaffold) auto-runs `async def` tests. `httpx`'s `AsyncClient` lets you exercise the app without a server:

```python
# tests/test_sessions.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app


async def test_create_session_returns_token():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        r = await client.post(
            "/sessions",
            json={"quiz_id": "Q1", "question_count": 5, "mode": "graded"},
            headers={"Authorization": "Bearer test-user-token"},
        )
        assert r.status_code == 201
        assert r.json()["session_token"].startswith("sess_")
```

`ASGITransport` runs the FastAPI app in-process without starting a network server — fast, deterministic, and the request goes through real routing.

## Worker Configuration

`uvicorn app.main:app --workers 4` runs 4 separate processes. Each has its own event loop. The PEP substrate default in dev is 1 worker (so a `print` is easy to follow); ECS deploys typically use `--workers $(nproc)` so each CPU core has a loop.

If you see a 25-person cohort overloading one local dev server, it's almost always because someone has a blocking call. Spinning up more workers masks the symptom; find the block.

## A Diagnostic Trick

When latency spikes for no obvious reason, log the time inside the handler vs the time outside:

```python
import time, asyncio
t0 = time.monotonic()
await some_call()
print("inside handler ms:", (time.monotonic() - t0) * 1000)
```

If the inside-handler time is low but client-observed latency is high, another request is blocking the loop. Audit nearby code for the pitfalls above.

For production, install `aiomonitor` or enable Python's `PYTHONASYNCIODEBUG=1` (logs warnings when a coroutine blocks the loop for >100ms).

## Worked Scenario: Today's Endpoint, Annotated

```python
@router.post("", status_code=201, response_model=SessionCreateResponse)
async def create_session(            # async def — handler does I/O
    body: SessionCreateRequest,
    db: DB,                          # injected async DB session
    user: CurrentUserDep,            # injected via async dep
) -> SessionCreateResponse:
    return await session_service.create_session(db=db, user=user, request=body)


async def create_session(            # service layer — async, awaits I/O
    db: AsyncSession, user, request,
) -> SessionCreateResponse:
    session_id = new_session_id()    # sync pure compute — fine
    raw_token = new_session_token()  # sync pure compute — fine
    token_hash = hash_session_token(raw_token)

    question_ids = await question_client.list_active_question_ids(  # awaited HTTP
        quiz_id=request.quiz_id
    )
    rng = secrets.SystemRandom()
    rng.shuffle(question_ids)        # sync — but it's fast on a ~500-item list
    chosen = question_ids[:request.question_count]

    now = server_now()
    record = SessionRecord(..., started_at=now, expires_at=now + SESSION_TTL)
    await session_repo.insert(db, record)              # awaited DB
    first = await question_client.fetch(chosen[0])     # awaited HTTP

    return to_create_response(record, raw_token, first, now)
```

Every I/O point is awaited; pure-compute parts (token mint, ID shuffle) are sync because they finish in microseconds and have no I/O to suspend on.

## Anti-Patterns

- **`async def` handler that calls `requests.get`** — the worst-of-both-worlds combination, blocks the loop while looking async.
- **`time.sleep` for backoff in async retry loops** — silently blocks all other requests on the worker.
- **Defining handlers as `async def` and then awaiting nothing** — fine, but it's a smell; either there's a missing await, or it should be `def`.
- **Calling `asyncio.run()` from inside a handler** — there's already a loop; this crashes with `RuntimeError`.
- **CPU-bound work in the handler** — even pure compute over a 50MB document blocks the loop. Push to `asyncio.to_thread` or a task queue.
- **Mixing sync `Session` and async `AsyncSession` in the same service.** Pick one.

## Key Takeaways
- Single event loop per worker; one blocking call freezes every in-flight request — async correctness is about keeping the loop spinning.
- `async def` everywhere in handler/service/repo for I/O; `def` for stateless pure compute.
- Memorize the pitfall list: sync DB drivers, `requests`, `time.sleep`, big CPU work, `subprocess.run`, file I/O. Each has an async replacement or an `asyncio.to_thread` escape.
- Independent I/O can be parallelized with `asyncio.gather`; sequential dependencies must remain sequential.
- Use `httpx.AsyncClient` + `ASGITransport` for in-process async tests; turn on `PYTHONASYNCIODEBUG=1` when diagnosing loop stalls.

---
*Prerequisites: day-10-fastapi-service-scaffolding-conventions, day-11-cross-service-http-integration-httpx-rest-contracts.*
