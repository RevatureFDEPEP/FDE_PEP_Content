# Cross-Service HTTP Integration (httpx, REST Contracts)

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
`test-management-service` doesn't own the question bank — `question-management-service` does (from Day 8). To create a session today, `test-management-service` has to call across the network for two things: the list of candidate question IDs for a quiz, and the rendered shape of the first question to embed in the response. This is the first cohort-built cross-service HTTP call in the substrate, and it's the right moment to set the patterns that the rest of Week 3, the reporting service, and the dashboard service will reuse: `httpx.AsyncClient`, explicit timeouts, bounded retries, request-id propagation, and graceful handling of upstream 5xx.

## Why `httpx` and Not `requests`

`requests` is sync. The FastAPI handlers added today are `async def` (see the async/await topic). Calling sync `requests` from an async handler blocks the event loop — one slow upstream call freezes every request in flight on that worker. `httpx` has the same API surface, async support, and HTTP/2 if you need it later.

```python
import httpx  # not: import requests
```

If you find yourself reaching for `requests` in an async file, stop and switch.

## The Client Singleton

A new `httpx.AsyncClient()` per request opens and tears down a TCP connection pool every time. Bad. Create one per process, manage its lifecycle via the FastAPI `lifespan`:

```python
# app/clients/__init__.py
import httpx
from ..settings import settings

question_client_http: httpx.AsyncClient | None = None


def get_question_http_client() -> httpx.AsyncClient:
    assert question_client_http is not None, "client not initialized"
    return question_client_http
```

```python
# app/main.py
from contextlib import asynccontextmanager
from . import clients

@asynccontextmanager
async def lifespan(app):
    clients.question_client_http = httpx.AsyncClient(
        base_url=settings.question_service_url,         # e.g. http://question-management-service:8000
        timeout=httpx.Timeout(connect=2.0, read=5.0, write=5.0, pool=2.0),
        limits=httpx.Limits(max_connections=50, max_keepalive_connections=20),
        headers={"User-Agent": "test-management-service/0.2"},
    )
    yield
    await clients.question_client_http.aclose()
```

Key points:

- **`base_url`** keeps callers from concatenating strings (and helps tests swap envs).
- **Explicit `Timeout`** for each phase. `httpx` defaults to 5s overall; we tighten `connect` to fail-fast on a dead pod and loosen `read` for Mongo-backed reads.
- **`Limits`** caps total open connections — prevents one slow upstream from exhausting fd quota.

## The Typed Client Wrapper

Don't sprinkle raw `httpx` calls around the service. Wrap them in a small client class with typed methods:

```python
# app/clients/question_client.py
from typing import Any
import httpx

from .common import propagate_headers, raise_for_upstream


class QuestionClient:
    def __init__(self, http: httpx.AsyncClient):
        self._http = http

    async def list_active_question_ids(self, quiz_id: str) -> list[str]:
        r = await self._http.get(
            "/questions/ids",
            params={"quiz_id": quiz_id, "active": "true"},
            headers=propagate_headers(),
        )
        raise_for_upstream(r, op="list_question_ids")
        return r.json()["ids"]

    async def fetch(self, question_id: str) -> dict[str, Any]:
        r = await self._http.get(
            f"/questions/{question_id}",
            headers=propagate_headers(),
        )
        raise_for_upstream(r, op="fetch_question")
        return r.json()
```

Now the service layer doesn't know `httpx` exists; it knows `QuestionClient`. Swapping in a fake for tests is one assignment.

## Propagating `X-Request-Id`

Day 10 introduced the request-id middleware for distributed log correlation. Cross-service calls must carry that id forward or the correlation breaks at the network boundary.

```python
# app/clients/common.py
from contextvars import ContextVar

current_request_id: ContextVar[str | None] = ContextVar("current_request_id", default=None)


def propagate_headers() -> dict[str, str]:
    rid = current_request_id.get()
    return {"X-Request-Id": rid} if rid else {}
```

The middleware (from Day 10) sets `current_request_id` from the inbound header at request entry; this helper reads it for every outbound call. End-to-end, a single request id now appears in `test-management-service` logs, `question-management-service` logs, and any further hop downstream — searchable across the whole slice.

## Handling Upstream 5xx Gracefully

A 5xx from `question-management-service` is not a 5xx from us. We should:

1. **Distinguish transient from permanent.** A 502/503/504 may be a pod restart; a 500 with an error envelope is more interesting.
2. **Map upstream errors to our own response shape.** Clients shouldn't see Mongo error bodies leak through three services.
3. **Log enough context to debug.** Method, path, status, request-id, latency.

```python
# app/clients/common.py
import httpx
import structlog

log = structlog.get_logger()


class UpstreamError(Exception):
    def __init__(self, op: str, status: int, body: str):
        self.op = op
        self.status = status
        self.body = body
        super().__init__(f"{op} failed: {status}")


def raise_for_upstream(r: httpx.Response, op: str) -> None:
    if r.is_success:
        return
    log.warning(
        "upstream.error",
        op=op, status=r.status_code, url=str(r.request.url),
        body=r.text[:500],
    )
    raise UpstreamError(op=op, status=r.status_code, body=r.text[:500])
```

A FastAPI exception handler maps `UpstreamError` to a 502:

```python
# app/main.py
from fastapi import Request
from fastapi.responses import JSONResponse
from .clients.common import UpstreamError

@app.exception_handler(UpstreamError)
async def on_upstream(req: Request, exc: UpstreamError):
    return JSONResponse(
        status_code=502,
        content={"error": "upstream_unavailable", "op": exc.op},
    )
```

See the error-handling topic for the broader 4xx/5xx decision table.

## Retries — Bounded, Idempotent, Backed Off

`GET` is safe to retry; `POST` generally isn't. For Day 11's two outbound calls — both `GET` — a bounded retry with jitter handles transient pod blips:

```python
# app/clients/common.py
import asyncio
import random
import httpx

RETRYABLE_STATUS = {502, 503, 504}


async def get_with_retries(
    http: httpx.AsyncClient,
    path: str,
    *,
    params: dict | None = None,
    headers: dict | None = None,
    attempts: int = 3,
) -> httpx.Response:
    last_exc: Exception | None = None
    for i in range(attempts):
        try:
            r = await http.get(path, params=params, headers=headers)
            if r.status_code not in RETRYABLE_STATUS:
                return r
            last_exc = httpx.HTTPStatusError(
                f"{r.status_code}", request=r.request, response=r
            )
        except (httpx.ConnectError, httpx.ReadTimeout) as e:
            last_exc = e

        if i < attempts - 1:
            backoff = (2 ** i) * 0.1 + random.uniform(0, 0.1)
            await asyncio.sleep(backoff)

    assert last_exc is not None
    raise last_exc
```

Three attempts max, exponential backoff (100ms / 200ms), jitter to avoid thundering-herd. Don't loop more than that — the candidate is staring at a spinner.

For `POST` calls later in the week, use an **idempotency key** header so a retry doesn't create two resources.

## The REST Contract Discipline

A cross-service call lives or dies by the contract. Three rules:

1. **Don't reach into the database directly.** It's tempting to have `test-management-service` query Mongo with the same connection string. Don't. The whole point of a service is that its data shape is private.
2. **Treat the called service's response schema as a public API.** Use a Pydantic model on the *consumer* side to validate — catches breaking changes immediately.

   ```python
   # app/clients/question_client.py
   from pydantic import BaseModel

   class _IdsResponse(BaseModel):
       ids: list[str]

   async def list_active_question_ids(self, quiz_id: str) -> list[str]:
       r = await self._http.get("/questions/ids", params={"quiz_id": quiz_id})
       raise_for_upstream(r, op="list_question_ids")
       return _IdsResponse.model_validate(r.json()).ids
   ```

3. **Version the endpoint when you change it.** `GET /v1/questions/ids` → `GET /v2/questions/ids`. Run both for a deprecation window. The PEP substrate doesn't enforce versioning yet, but Week 4 cleanup is a good time to introduce it.

## Worked Scenario: Today's Two Calls

`session_service.create_session()` makes two HTTP calls:

```
test-management-service                  question-management-service
─────────────────────────                 ───────────────────────────
create_session()
  → list_active_question_ids(quiz)
                     ───────────GET /questions/ids?quiz_id=…──────────►
                     ◄──────────200 {"ids":[ulid, ulid, …]}────────────
  shuffle, take 10
  → fetch(ids[0])
                     ────────────GET /questions/{ulid}────────────────►
                     ◄──────────200 {"id":..., "stem":..., …}─────────
  build SessionCreateResponse
```

Both calls carry `X-Request-Id`. Both fall back to retry on 502/503/504. The session create stays well under our 1s p95 target as long as the upstream is healthy.

## Anti-Patterns

- **`new httpx.AsyncClient()` per request.** Trashes the connection pool.
- **No timeout, or a giant timeout.** One stuck upstream call now ties up a worker forever.
- **Retrying `POST` without an idempotency key.** Creates duplicates.
- **Letting upstream error bodies leak through verbatim.** Now your API contract includes their bug.
- **Skipping `X-Request-Id` propagation.** Logs become a scavenger hunt across services — undoing Day 10.
- **Sync `requests` from `async def`.** Blocks the event loop; see the async topic.

## Key Takeaways
- `httpx.AsyncClient` as a process-level singleton with explicit `Timeout` and `Limits`, lifecycle-managed by FastAPI's `lifespan`.
- Wrap raw HTTP in a typed `QuestionClient` so the service layer stays clean and tests can swap a fake.
- Propagate `X-Request-Id` on every outbound call so Day 10's distributed log correlation survives the network hop.
- Bounded, jittered retries on idempotent calls only; map upstream errors to a 502 with a stable shape — don't echo their body.
- Validate consumed responses with Pydantic models on the consumer side to catch contract drift.

---
*Prerequisites: [05-distributed-log-correlation-across-services.md](../day-10/05-distributed-log-correlation-across-services.md), [03-async-await-patterns-in-python-web-frameworks.md](03-async-await-patterns-in-python-web-frameworks.md).*
