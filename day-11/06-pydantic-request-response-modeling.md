# Pydantic Request/Response Modeling

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Day 8 introduced Pydantic v2 for the question-authoring backend — discriminated unions on the polymorphic `Question` model, `model_config = ConfigDict(extra="forbid")`, and request/response separation. Today we apply the same discipline to the session boundary: the request body that starts a session, the response that hands back a session token and first question, and the internal-to-external shape transformation that hides server-only fields. This file assumes you remember the Day 8 patterns; here we're focused on what's *different* when modeling a session resource.

## Why Request and Response Models Differ

A session has both inbound and outbound shapes, and they should not be the same class:

- **Request:** what the client is allowed to send. Small. Validated hard. `extra="forbid"`.
- **Response:** what we promise to return. Includes derived/server-set fields. Stable contract.
- **Internal:** the ORM/document model. Has private fields (audit cols, raw answer history, internal flags).

Conflating these is the most common Pydantic anti-pattern. A client sending `is_admin=true` to a model that also serializes `is_admin` on the way out is a privilege-escalation bug waiting to happen.

## The Session Request Model

```python
# app/schemas/sessions.py
from typing import Annotated, Literal
from pydantic import BaseModel, ConfigDict, Field

QuestionCount = Annotated[int, Field(ge=1, le=50)]


class SessionCreateRequest(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    quiz_id: str = Field(..., description="ULID of the quiz definition")
    question_count: QuestionCount = 10
    mode: Literal["practice", "graded"] = "graded"
```

Notes:

- `extra="forbid"` rejects unknown fields with a 422 — catches typos and prevents clients from smuggling fields the server might later honor by accident.
- `QuestionCount` is a `Annotated[int, Field(...)]` alias — reused on `PATCH /sessions/{id}` later in the week.
- `mode` is a `Literal` rather than an enum import to keep the schema readable in the OpenAPI doc.

## The Session Response Model

```python
# app/schemas/sessions.py (continued)
from datetime import datetime
from uuid import UUID
from .questions import QuestionPublic  # reused from D8


class SessionCreateResponse(BaseModel):
    model_config = ConfigDict(extra="forbid")

    session_id: UUID
    session_token: str  # opaque, see day-11 opaque-token topic
    quiz_id: str
    mode: Literal["practice", "graded"]
    total_questions: int

    # Server-authoritative time fields (see day-11 server-authoritative-state)
    started_at: datetime
    expires_at: datetime
    server_now: datetime

    first_question: QuestionPublic
```

Three things to flag:

- **`QuestionPublic` is reused from Day 8** — that model already strips correct-answer flags and authoring metadata. Don't redefine it; if you find yourself wanting to, fix the original instead.
- **`server_now`** lets the client compute an offset for the countdown display without trusting its own clock. The deeper rationale lives in [08-server-authoritative-state.md](08-server-authoritative-state.md).
- **`session_token` is a plain string in the schema, not a `SecretStr`.** It's a return-to-client value; it needs to serialize as-is, and `SecretStr` would render as `"**********"`.

## Internal Model vs External Model

The MongoDB / Postgres-level session record carries more than we return:

```python
# app/models/session.py (internal — not exposed)
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel


class SessionRecord(BaseModel):
    """Internal representation. Never serialized to a client."""
    session_id: UUID
    session_token_hash: str        # store hash, not raw token
    user_id: str
    quiz_id: str
    mode: str
    question_ids: list[str]        # the full sampled list (D11 topic 4)
    current_index: int
    started_at: datetime
    expires_at: datetime
    submitted_at: datetime | None
    schema_version: int = 1        # convention from D8
```

`SessionRecord` never crosses the HTTP boundary. The service layer converts it to a `SessionCreateResponse`:

```python
# app/services/session_service.py
def to_create_response(
    record: SessionRecord,
    raw_token: str,
    first_question: QuestionPublic,
    server_now: datetime,
) -> SessionCreateResponse:
    return SessionCreateResponse(
        session_id=record.session_id,
        session_token=raw_token,     # raw form, returned ONCE
        quiz_id=record.quiz_id,
        mode=record.mode,
        total_questions=len(record.question_ids),
        started_at=record.started_at,
        expires_at=record.expires_at,
        server_now=server_now,
        first_question=first_question,
    )
```

Pattern: **hash on the way in, return raw once, never echo it again**. Same as a password reset link.

## Field Validators for Cross-Field Rules

Pydantic v2 uses `@field_validator` and `@model_validator`:

```python
from pydantic import model_validator

class SessionCreateRequest(BaseModel):
    # ... fields ...

    @model_validator(mode="after")
    def practice_mode_caps_count(self) -> "SessionCreateRequest":
        if self.mode == "practice" and self.question_count > 20:
            raise ValueError("practice mode capped at 20 questions")
        return self
```

This raises a 422 at the FastAPI boundary with `loc=["body"]`. The Day 9 frontend's `setError` flow handles it without any new code path.

## OpenAPI Examples — Document the Contract

`response_model` already shapes the schema, but add examples so frontend devs can see realistic payloads:

```python
class SessionCreateResponse(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        json_schema_extra={
            "examples": [
                {
                    "session_id": "0193d3a4-1234-7abc-9def-0123456789ab",
                    "session_token": "sk_live_QQwK...redacted...",
                    "quiz_id": "01HSY8X4ZJN0...",
                    "mode": "graded",
                    "total_questions": 10,
                    "started_at": "2026-05-18T14:02:00Z",
                    "expires_at": "2026-05-18T14:32:00Z",
                    "server_now": "2026-05-18T14:02:00.137Z",
                    "first_question": {"...": "see questions schema"},
                }
            ]
        },
    )
    # ... fields ...
```

The auto-generated `/docs` page now shows the example payload — invaluable when level-3 strong candidates record their Next.js walkthrough this week and need a realistic shape to mock against.

## Serialization Gotchas

- **`datetime` serializes to ISO-8601 with offset.** Make sure your DB returns UTC-aware datetimes (`timezone.utc`); naive datetimes serialize without `Z` and confuse the frontend.
- **`UUID` serializes as a string with hyphens.** Good. Don't manually `str(uuid)` in the service layer.
- **`extra="forbid"` on response models** seems paranoid but protects against future refactors where someone adds an internal field to the response object by accident.

## Worked Scenario: The Day 11 Contract

For today's `POST /sessions`:

- Client posts `SessionCreateRequest`.
- Service creates a `SessionRecord`, samples question IDs (topic 4), fetches the first question (topic 5), mints a token (topic 3).
- Service returns `SessionCreateResponse` — and *only* that shape escapes the service.

If a reviewer asks "where do we decide what's safe to send to the client?" the answer is `SessionCreateResponse` in `app/schemas/sessions.py`. One file.

## Anti-Patterns

- **Reusing the internal `SessionRecord` as the response model.** Leaks `session_token_hash`, `schema_version`, and anything added later.
- **Returning the raw question objects from the DB.** Use `QuestionPublic` (D8) — it already strips the correct answer.
- **Letting `extra` default to `"allow"` on requests.** Means a typo in a field name silently passes validation.
- **Using `Any` to avoid writing a schema.** You're trading 10 minutes today for a debug session in Week 4.
- **Computing `expires_at` on the client and trusting it.** See [08-server-authoritative-state.md](08-server-authoritative-state.md).

## Key Takeaways
- Request, response, and internal models are three different classes — never collapse them.
- `extra="forbid"` on requests catches client typos and blocks privilege smuggling.
- Reuse Day 8's `QuestionPublic` rather than redefining a question shape for sessions.
- Hash the session token at rest, return the raw value exactly once.
- `json_schema_extra` examples make the OpenAPI doc useful for frontend pairing.

---
*Prerequisites: day-8 question-authoring backend topics. Sibling topic: [07-opaque-token-generation-and-session-identifiers.md](07-opaque-token-generation-and-session-identifiers.md) details the `session_token` field modeled here (read after this).*
