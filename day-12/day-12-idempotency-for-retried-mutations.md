# Idempotency for Retried Mutations

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Networks fail. Clients retry. A candidate's browser fires `POST /sessions/{id}/answer`, the server processes the request and scores it, the response packet drops on the way back, the browser sees a timeout, the browser retries — and *the server has now received the same answer submission twice*. If the second request also scores, the candidate either gets double credit, double-charged for a wrong answer, or — worst — has their session advanced past the next question without seeing it. The fix is **idempotency**: design the endpoint so that retrying a mutation does not duplicate its effect. This file walks through the idempotency-key header pattern, server-side dedup, and how it composes with D11's opaque session-token discipline.

## What "Idempotent" Means For Mutations

A mutation is **idempotent** if applying it N times has the same effect as applying it once. For our `POST /sessions/{id}/answer`:

- Applied once: answer recorded, scored, session advanced to next question.
- Applied twice (retry after network failure): same answer recorded (no duplicate row), same score, session at the same next question.
- Applied a third time (paranoid retry): no change, same response returned.

`GET` is idempotent by definition (read-only). `PUT` and `DELETE` are conventionally idempotent (set X to Y, delete X). `POST` is *not* idempotent by default — the server has no way to know that two `POST`s with identical bodies are "the same submission" vs "two separate submissions that happen to look alike." We make it idempotent by adding a key the client supplies.

## The Idempotency-Key Pattern

The client generates a fresh UUID for each *logical* submission and sends it in the `Idempotency-Key` header. The server stores the key alongside the response. If the same key arrives again, the server returns the **stored response** without re-running the side effect.

```
# First attempt
POST /sessions/0193d3a4-.../answer
Idempotency-Key: 7b3e1a2c-9f4d-4c8e-bb12-3f5a8d6e9f01
X-Session-Token: sess_oo3...
Content-Type: application/json

{"selected_options": [0, 2]}

# Response
HTTP/1.1 200 OK
{"score": 0.67, "next_question": {...}}

# Network died; client retries with the SAME key
POST /sessions/0193d3a4-.../answer
Idempotency-Key: 7b3e1a2c-9f4d-4c8e-bb12-3f5a8d6e9f01
X-Session-Token: sess_oo3...

{"selected_options": [0, 2]}

# Response — identical bytes, side effect not re-run
HTTP/1.1 200 OK
{"score": 0.67, "next_question": {...}}
```

The two responses are byte-for-byte identical because the second one was *replayed* from storage, not recomputed.

## What The Server Stores

A separate table — or a column on `answers` — captures the key and the response shape:

```python
# test_management_service/app/models/idempotency.py
class IdempotencyRecord(Base):
    __tablename__ = "idempotency_records"
    key: Mapped[UUID] = mapped_column(primary_key=True)
    session_id: Mapped[UUID] = mapped_column(index=True)
    request_fingerprint: Mapped[str]  # sha256(body) — to detect key reuse with different body
    response_status: Mapped[int]
    response_body: Mapped[dict] = mapped_column(JSONB)
    created_at: Mapped[datetime]

    __table_args__ = (
        # Topic 4 covers the unique-constraint approach for race conditions.
        UniqueConstraint("key", name="uq_idempotency_key"),
    )
```

Three columns matter:

- **`key`** — the client-supplied UUID. Unique constraint enforces "one record per key."
- **`request_fingerprint`** — sha256 of the canonical request body. If the same key arrives with a *different* body, that's a client bug, and the server should refuse with **422 Unprocessable Entity** (not replay a stale response, not accept the new body silently).
- **`response_body`** — the exact JSON returned the first time. Replayed verbatim on retry.

The `session_id` index is for the rare reconciliation query ("show me all idempotent submissions for this session") and is optional. It's not used for dedup — the key alone identifies the operation.

## The Lookup-Then-Insert Dance

```python
# test_management_service/app/routers/sessions.py
@router.post("/sessions/{session_id}/answer")
async def submit_answer(
    session_id: UUID,
    body: AnswerSubmit,
    idempotency_key: Annotated[UUID, Header(alias="Idempotency-Key")],
    db: DB,
    session: LockedSession,  # topic 5 — pessimistic lock on the session row
) -> AnswerResponse:
    fingerprint = sha256(canonical_json(body).encode()).hexdigest()

    existing = await idempotency_repo.get(db, idempotency_key)
    if existing is not None:
        if existing.request_fingerprint != fingerprint:
            raise HTTPException(
                422,
                "idempotency_key_reused_with_different_body",
            )
        # Replay the stored response, do NOT re-score.
        return AnswerResponse.model_validate(existing.response_body)

    # First time — do the real work.
    result = score(
        question_type=session.current_question.question_type,
        correct_options=session.current_question.correct_options,
        selected_options=body.selected_options,
    )
    await answer_repo.upsert(...)
    response = build_response(result, session)

    await idempotency_repo.insert(
        db,
        IdempotencyRecord(
            key=idempotency_key,
            session_id=session_id,
            request_fingerprint=fingerprint,
            response_status=200,
            response_body=response.model_dump(),
            created_at=server_now(),
        ),
    )
    return response
```

The lookup-then-insert pattern is **race-prone on its own** — two requests with the same key can both pass the `existing is None` check and both attempt to insert. Topic 4 (unique constraints) and topic 5 (pessimistic locking) close that race. The structure above is the *logical* shape; the implementation needs the DB-level guard.

## Where The Key Comes From — Client Responsibility

The client mints the key. Two patterns:

1. **One key per logical submission attempt.** The client generates `crypto.randomUUID()` when the user clicks "Submit Answer", stashes it in component state, and reuses it for any retry of *that submission*. A fresh click for the *next* question gets a fresh key.
2. **One key per (session, question_index) pair.** The client builds the key deterministically from `session_id` + `current_index`. Simpler — no state to track — but introduces a constraint: the client cannot re-submit a different answer for the same question (which is what we want anyway).

For PEP, **pattern 1** is the choice: the frontend (Day 14) holds the key in component state for the lifetime of one "Submit" interaction. On Day 14 you'll see this materialize as a `useRef<string>` that's reset to a new UUID each time the question advances.

## Why The Client Supplies The Key, Not The Server

Naively, the server could hash the request body and use that as the dedup key. Two reasons not to:

- **Two semantically-distinct requests can have identical bodies.** Candidate selects `{0, 2}` for question 1, scores 0.67. The session advances. Question 2 is also a multi-select. Candidate selects `{0, 2}` for question 2. Same body, different intent — and if the server dedups on body hash, the second submission silently replays the first.
- **The client knows when a retry is a retry.** Only the client knows "I sent this and didn't get an ack — try again." The server can't infer that.

The client-supplied key encodes the intent ("this is the same submission as before") in a way the body cannot.

## Composition With D11's Session Token

The `Idempotency-Key` is **orthogonal** to the session token from D11:

| Header | Purpose | Scope |
|---|---|---|
| `X-Session-Token: sess_...` | Authenticate the holder against the session | Lifetime of the session |
| `Idempotency-Key: <uuid>` | Identify a specific submission attempt for dedup | A single logical request |

Both headers are required on every mutation. The session token authorizes "you may submit to this session"; the idempotency key says "and this is *which* submission." Reusing the session token across submissions is correct (it's a credential); reusing the idempotency key across submissions is a bug (it conflates intents).

## TTL And Storage Cost

Idempotency records are not forever. A reasonable TTL:

- **24 hours.** Long enough to cover any plausible retry window (the candidate's network is down for hours, the client retries after recovery). Short enough that the table doesn't grow without bound.
- **Cleanup job.** A Day-20-relevant cron or `DELETE WHERE created_at < now() - interval '24 hours'`. Not Day 12's concern, but flag it as a TODO.

Storage is cheap. A typical record is ~2KB; a 25-person cohort generating ~50 answers each over a session is 1,250 records per cohort, ~2.5MB. Negligible.

## What Goes Wrong Without Idempotency

A short catalog of incidents that an idempotency key prevents:

- **Double-scored answer.** Network blip, client retries, server scores twice, candidate either gets 2× credit or has wildly inflated total.
- **Question skipped.** First submission scores and advances index from 4 to 5. Retry scores again (against question 5 this time, with question 4's selections — nonsense data) and advances to 6. Candidate sees question 6 next and never sees question 5.
- **State corruption on the "submit final answer" call.** The very last submission triggers the attempt lock (topic 6). Without idempotency, the retry hits a locked attempt and gets a 409, but the client has no way to recover the actual result of the first call.

The first two are immediately user-visible. The third surfaces as "candidate finished the quiz but the dashboard doesn't show their final score" — much harder to debug after the fact.

## Anti-Patterns

- **Server-generated dedup key (body hash).** Conflates distinct requests with identical bodies.
- **Storing the response as a string but parsing/re-serializing on replay.** Float precision drift, key ordering differences — replays no longer byte-equal. Store the original JSON exactly.
- **Treating "key seen, different body" as success.** The client thinks the new body was accepted; the server returned the old one. Silent data divergence. Return 422.
- **No TTL on the idempotency table.** Unbounded growth. Set a TTL; document it.
- **Skipping idempotency on the "final submit" because it's a one-shot operation.** That's the *most* important operation to make idempotent — the cost of a duplicate is the highest.
- **Using the session token as the idempotency key.** Conflates auth with deduplication. Two different concerns, two different headers.

## Key Takeaways
- Idempotency makes `POST` safe to retry: the client supplies an `Idempotency-Key` header; the server stores key + request fingerprint + response and replays on retry.
- Keys are client-generated UUIDs, one per logical submission; reusing a key with a different body is a 422.
- Persist the response **as-stored**, not as a re-serialization, so replays are byte-equal.
- Composes cleanly with D11's session-token discipline: token authenticates, key deduplicates — orthogonal.
- The lookup-then-insert pattern is race-prone on its own; topic 4's unique constraint and topic 5's pessimistic lock close the race.

---
*Prerequisites: day-11-opaque-token-generation-and-session-identifiers, day-11-error-handling-and-http-status-code-discipline. Forward references: day-12-unique-constraints, day-12-database-transactions-and-pessimistic-locking, day-14 frontend retry handling.*
