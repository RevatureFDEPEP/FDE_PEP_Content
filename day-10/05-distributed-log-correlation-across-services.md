# Distributed Log Correlation Across Services

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
A single "create a question" request crosses three processes — the Next.js server (for the page render), `api-gateway`, and `question-management-service` — and may fan out to MinIO and Mongo. When that request goes wrong, you need to find every log line it produced, in order, across all three services. The technique is **log correlation**: stamp every log line with an identifier that follows the request through the system. Today is an orientation-depth introduction — enough to wire up a request id in our stack and grep logs by it. Full distributed tracing (OpenTelemetry, Jaeger) is Phase 2 material.

## The Problem in Concrete Terms

Without correlation, `docker compose logs -f` looks like this for a failing request:

```
api-gateway       | INFO: 172.18.0.1:54312 - "POST /api/questions HTTP/1.1" 502 Bad Gateway
question-mgmt     | ERROR: pymongo.errors.NetworkTimeout: ...
question-mgmt     | INFO: 172.18.0.5:42188 - "POST /questions HTTP/1.1" 500 Internal Server Error
mongo             | {"t":{"$date":"..."},"s":"I","msg":"Slow query","attr":{...}}
api-gateway       | INFO: 172.18.0.1:54313 - "GET /healthz HTTP/1.1" 200 OK
question-mgmt     | INFO: 172.18.0.5:42189 - "POST /questions HTTP/1.1" 201 Created
```

Question: was the slow Mongo query the cause of the 502? Maybe — they're temporally adjacent. But there might have been 30 concurrent requests; you have no way to be sure. Correlation answers this.

## The Correlation ID Pattern

The minimal pattern:

1. The first service to see a request generates a request id (or reads one from an inbound header).
2. Every log line that service emits while handling the request includes that id.
3. When the service calls a downstream, it forwards the id in a header.
4. Every downstream does the same.
5. To debug, grep all logs by the id.

Common header names: `X-Request-Id`, `X-Correlation-Id`, `traceparent` (W3C standard, more powerful, used by OpenTelemetry).

## Wiring It Up in the PEP Stack

### Step 1: Generate or Accept the ID at the Gateway

`api-gateway` is the entry point. Use middleware:

```python
# api-gateway/app/middleware.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar("request_id", default="-")

class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        rid = request.headers.get("x-request-id") or f"req_{uuid.uuid4().hex[:16]}"
        token = request_id_var.set(rid)
        try:
            response = await call_next(request)
            response.headers["x-request-id"] = rid
            return response
        finally:
            request_id_var.reset(token)
```

The `ContextVar` lives across `await` points within a single request, so any log call made during request handling can read it.

### Step 2: Inject Into the Log Format

Use the standard library `logging` with a filter, or `structlog` (preferred for JSON logs):

```python
# api-gateway/app/logging_setup.py
import logging
import json
from .middleware import request_id_var

class RequestIdFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_var.get()
        return True

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "ts": self.formatTime(record),
            "level": record.levelname,
            "service": "api-gateway",
            "request_id": getattr(record, "request_id", "-"),
            "msg": record.getMessage(),
        })

def configure_logging():
    handler = logging.StreamHandler()
    handler.addFilter(RequestIdFilter())
    handler.setFormatter(JsonFormatter())
    root = logging.getLogger()
    root.handlers = [handler]
    root.setLevel("INFO")
```

Now every log line looks like:

```json
{"ts":"2026-05-19T14:22:01","level":"INFO","service":"api-gateway","request_id":"req_a8f3...","msg":"forwarding POST /api/questions to question-management-service"}
```

### Step 3: Forward Downstream

When the gateway proxies the request:

```python
async def proxy(request: Request, target: str) -> Response:
    rid = request_id_var.get()
    headers = {**dict(request.headers), "x-request-id": rid}
    async with httpx.AsyncClient() as client:
        r = await client.request(
            request.method, f"{target}{request.url.path}",
            headers=headers, content=await request.body(),
        )
    return Response(r.content, status_code=r.status_code, headers=dict(r.headers))
```

### Step 4: Repeat in Each Downstream Service

`question-management-service` adds the same middleware + log filter. The `request_id` it picks up is the one the gateway forwarded. Now all logs share the same id for a single request.

### Step 5: Frontend Participation (Optional Today)

The Next.js client can generate the id and send it on every fetch:

```ts
// lib/api.ts
const rid = `req_${crypto.randomUUID().replaceAll('-', '').slice(0, 16)}`;
const res = await fetch(url, { headers: { 'x-request-id': rid, ... }});
```

Then any frontend console.error or telemetry can be cross-referenced with backend logs.

## Reading Correlated Logs

Once the wiring is in place, debugging becomes mechanical:

```bash
# Pick a failing request from the gateway's access log
docker compose logs api-gateway | grep '502 Bad Gateway' | tail -1
# Note the x-request-id from the response header in your browser's network tab,
# or from the structured log line: "request_id":"req_a8f3..."

# Now pull every log line for that request, across all services, in time order
docker compose logs --no-color --timestamps | grep req_a8f3...
```

You see the full story in one view: gateway received → gateway forwarded → backend started → Mongo query timed out → backend 500 → gateway 502.

## Why Not Just Use Timestamps?

Timestamps narrow the window but don't disambiguate concurrent requests. Under load (25 trainees clicking submit at the same time), every second has dozens of overlapping lines from each service. Without ids you guess; with ids you know.

## Beyond Request IDs: Trace Context (Brief)

A real distributed-tracing system uses **trace context** (W3C `traceparent` header) which carries:

- `trace-id` — unique per request, like our request id.
- `span-id` — unique per service hop within the trace; lets you build a tree.
- `sampled` flag — whether this trace gets exported to the tracing backend.

Libraries like `opentelemetry-instrumentation-fastapi` add this automatically. We're not wiring it today, but the request-id pattern is the conceptual on-ramp.

## Anti-Patterns

- **Logging the id in some lines but not others.** The grep will miss steps. Always go through the configured filter/formatter.
- **Generating a fresh id at every service.** Then nothing correlates. Always accept inbound, generate only when absent.
- **Putting the id in the message text only.** Then it's not queryable as a field by log aggregators (Loki, CloudWatch Insights). Always make it a structured field.
- **Leaking the id to untrusted clients.** It's not a secret, but it's also not user-facing — keep it in headers and structured logs, not page bodies.
- **Reinventing W3C `traceparent`.** If you're standing up something new (Phase 2), use the standard from day one.

## Key Takeaways
- A correlation id is a single value that follows a request through every service it touches.
- The gateway is the natural place to generate or accept the id; downstreams forward it via header.
- Use a `ContextVar` + log filter so every log line gets the id without manual passing.
- Emit logs as JSON so the id is a structured field, not a substring.
- The pattern is the on-ramp to full distributed tracing in Phase 2.

---
*Prerequisites: [09-container-debugging-logs-exec-troubleshooting.md](../day-02/09-container-debugging-logs-exec-troubleshooting.md), [03-reverse-proxy-fundamentals-nginx-traefik.md](../day-06/03-reverse-proxy-fundamentals-nginx-traefik.md).*
