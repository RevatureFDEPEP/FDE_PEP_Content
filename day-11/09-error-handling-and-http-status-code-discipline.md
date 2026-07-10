# Error Handling and HTTP Status Code Discipline

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
`POST /sessions` has more failure modes than success modes — the user might not be authorized, the quiz might not exist, the question bank might not have enough items, a concurrent request might have already created the session, the body might be malformed. Each one is a different HTTP status code, and picking the right one is the difference between a usable API and a frustrating one. This file covers the close-but-not-the-same status pairs (400 vs 422, 401 vs 403, 404 vs 410, 409 vs 422), gives a decision table for today's specific failure modes, and shows how to wire a consistent error envelope through FastAPI's exception machinery.

## The Pairs People Get Wrong

### 400 vs 422

- **400 Bad Request** — the request is malformed at the protocol or basic-syntax level. Bad JSON, missing `Content-Type`, body too large.
- **422 Unprocessable Entity** — the request is syntactically valid but semantically wrong. Field missing a value, value out of range, fails a cross-field validator.

FastAPI emits **422 by default** when Pydantic validation fails — leave that alone. Use **400** only for things FastAPI/Pydantic can't see (e.g., body fails a malformed-JSON check before reaching the model).

### 401 vs 403

- **401 Unauthorized** — "I don't know who you are." Missing or invalid credentials. Includes `WWW-Authenticate` header to tell the client how to authenticate.
- **403 Forbidden** — "I know who you are, and you're not allowed to do this." Authenticated, but lacks permission.

A common mistake: returning 403 when the JWT is missing. That's a 401 — there's no identity to forbid. Conversely, returning 401 when the user is *known* but lacks permission gives the client no way forward (it'll re-prompt for login that won't help).

### 404 vs 410

- **404 Not Found** — "this resource doesn't exist or you can't see it." Default for missing things.
- **410 Gone** — "this resource existed and has been intentionally removed; don't retry." Use for **expired sessions** in Week 3: the session was real, you used to be able to act on it, you no longer can. Tells the client "stop polling, this is permanent."

The session-expired case is the canonical 410 use case for the PEP slice. Day 12's submit endpoint and Day 14's answer endpoint both raise 410 when `server_now() > session.expires_at`.

### 409 vs 422

- **409 Conflict** — the request can't be processed because of a conflict with the *current state of the server*. Idempotency violation, version mismatch, concurrent modification.
- **422 Unprocessable Entity** — the request's *own* content is bad.

If a candidate POSTs `POST /sessions` twice with the same idempotency key, the second is a 409. If they POST with `question_count=0`, it's a 422.

## Decision Table for Today's Endpoint

For `POST /sessions`, here's the mapping of failure → status → error code:

| Situation | Status | `error` code | When detected |
|---|---|---|---|
| Body isn't valid JSON | 400 | `malformed_body` | FastAPI body parser |
| Missing or invalid bearer token | 401 | `unauthorized` | `get_current_user` dep |
| User authenticated but lacks `candidate` role | 403 | `forbidden` | role check after auth |
| `question_count` out of range, unknown field, wrong type | 422 | `invalid_request` | Pydantic |
| `mode=practice` + `question_count > 20` | 422 | `invalid_request` | `@model_validator` |
| `quiz_id` references a quiz that doesn't exist | 404 | `quiz_not_found` | upstream 404 from question service |
| Quiz exists but is archived | 410 | `quiz_archived` | upstream returns archive flag |
| Quiz exists but has fewer than `question_count` active questions | 422 | `insufficient_questions` | service-layer check after listing IDs |
| Concurrent create with same idempotency key already succeeded | 409 | `duplicate_session` | repo unique-key violation |
| Upstream question-management-service returns 5xx | 502 | `upstream_unavailable` | client wrapper (topic 5) |
| Database unreachable | 503 | `database_unavailable` | DB driver exception handler |
| Anything else | 500 | `internal_error` | last-resort handler |

A few judgement calls worth flagging:

- **Insufficient questions is 422, not 409.** The body itself asks for something the data doesn't support — semantic problem with the request. 409 would imply concurrent modification, which isn't the case.
- **Quiz not found is 404, not 422.** It's a referenced resource that doesn't exist, like asking for `GET /quizzes/nope`.
- **Archived quiz is 410, not 404.** It existed; it's intentionally gone for new sessions. This signals "don't show this quiz in the menu again" to the client.

## A Consistent Error Envelope

Every error response should have the same shape, so the Day 9 frontend's error handling doesn't grow a special case per service:

```json
{
  "error": "insufficient_questions",
  "message": "Quiz has 7 active questions; 10 requested.",
  "request_id": "0193d3a4-1234-7abc-9def-0123456789ab",
  "details": {
    "available": 7,
    "requested": 10
  }
}
```

- `error` is a stable, machine-readable code. Frontends switch on it.
- `message` is a human-readable string. Logs surface it; UIs may show it.
- `request_id` is the Day 10 correlation id. Lets candidates paste it into support tickets.
- `details` is optional, structured context.

The 422 case follows the same shape but with the Pydantic-style location list:

```json
{
  "error": "invalid_request",
  "message": "Validation failed.",
  "request_id": "...",
  "details": {
    "errors": [
      {"loc": ["body", "question_count"], "msg": "Input should be less than or equal to 50"}
    ]
  }
}
```

## Implementation in FastAPI

### Domain Exceptions, Not Magic Strings

Service code raises typed exceptions, not `HTTPException` directly. Keeps services framework-independent and tests cleaner:

```python
# app/errors.py
class DomainError(Exception):
    status_code: int = 500
    error_code: str = "internal_error"

    def __init__(self, message: str = "", **details):
        self.message = message or self.error_code
        self.details = details
        super().__init__(self.message)


class QuizNotFound(DomainError):
    status_code = 404
    error_code = "quiz_not_found"


class QuizArchived(DomainError):
    status_code = 410
    error_code = "quiz_archived"


class InsufficientQuestions(DomainError):
    status_code = 422
    error_code = "insufficient_questions"


class DuplicateSession(DomainError):
    status_code = 409
    error_code = "duplicate_session"
```

Service code raises them naturally:

```python
# app/services/session_service.py
if len(all_ids) < request.question_count:
    raise InsufficientQuestions(
        f"Quiz has {len(all_ids)} active questions; {request.question_count} requested.",
        available=len(all_ids),
        requested=request.question_count,
    )
```

### One Handler That Maps Them All

```python
# app/main.py
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

from .errors import DomainError
from .clients.common import UpstreamError


def _envelope(error_code: str, message: str, request_id: str, **details):
    body = {"error": error_code, "message": message, "request_id": request_id}
    if details:
        body["details"] = details
    return body


@app.exception_handler(DomainError)
async def on_domain(req: Request, exc: DomainError):
    return JSONResponse(
        status_code=exc.status_code,
        content=_envelope(exc.error_code, exc.message, _rid(req), **exc.details),
    )


@app.exception_handler(RequestValidationError)
async def on_validation(req: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content=_envelope(
            "invalid_request", "Validation failed.", _rid(req),
            errors=exc.errors(),
        ),
    )


@app.exception_handler(UpstreamError)
async def on_upstream(req: Request, exc: UpstreamError):
    return JSONResponse(
        status_code=502,
        content=_envelope("upstream_unavailable", f"{exc.op} failed.", _rid(req)),
    )


def _rid(req: Request) -> str:
    return getattr(req.state, "request_id", "")
```

Every error shape now flows through the same envelope, with the same request id, with stable `error` codes the frontend can switch on.

### What About `HTTPException`?

FastAPI's built-in `HTTPException` is fine for trivial cases inside a handler (`raise HTTPException(401, "Missing token")`) but tends to encourage strings-as-protocol. The pattern above scales better for a service that grows: domain exceptions for service-layer errors, `HTTPException` only at the auth boundary where there's no domain meaning to attach.

## Two Things to Get Right Today

1. **The 410 for `quiz_archived`** — establishes the pattern Day 12 uses for `session_expired`. If the cohort hand-rolls "use 404 for archived" today, Day 12 inherits the confusion.
2. **The 422 for `insufficient_questions`** — exercising a domain exception that isn't a Pydantic violation but still uses 422 reinforces "semantic-but-not-syntactic" intuition.

## Anti-Patterns

- **Returning 200 with `{"success": false}`.** The HTTP status code *is* the success signal. Reserve 200 for success. Caches, monitors, and middlewares all key off the status code.
- **500 for "user didn't have permission."** Internal logs fill with false alarms. Use 403.
- **Different error shapes per endpoint.** Each one becomes a special case in the client.
- **Letting raw exceptions through.** A stack trace in the response body is an information leak and an unstable contract. Always go through a handler.
- **Omitting the `request_id`.** Support tickets become unsearchable. Day 10's middleware put it in `req.state.request_id` — use it.
- **Swallowing upstream errors as 200s.** Hides the failure from the caller and from monitoring.

## Key Takeaways
- 400 = protocol/syntax, 422 = semantics; 401 = unknown identity, 403 = known but forbidden; 404 = missing, 410 = intentionally gone; 409 = state conflict.
- Use a single error envelope (`error`, `message`, `request_id`, optional `details`) across every failure.
- Define domain exceptions with `status_code` and `error_code` attributes; raise them in service code; one FastAPI handler maps them to responses.
- Today's specific picks: insufficient questions → 422, archived quiz → 410, duplicate session → 409, upstream 5xx → 502.
- Reserve 200 for success; never encode failure inside a 200 body.

---
*Prerequisites: day-9 frontend error handling topics, [06-pydantic-request-response-modeling.md](06-pydantic-request-response-modeling.md), [04-cross-service-http-integration-httpx-rest-contracts.md](04-cross-service-http-integration-httpx-rest-contracts.md).*
