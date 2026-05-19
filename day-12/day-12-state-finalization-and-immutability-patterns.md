# State Finalization and Immutability Patterns

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
A session moves through a small set of states: it's created, the candidate is taking it, they submit, the system scores it, the result is final. Once a session is **submitted**, no further answers may be recorded, no answers may be edited, no rescoring is permitted from the candidate's side. The session is **immutable from that moment**. This file makes the state machine explicit, names every transition and what triggers it, and shows how to enforce immutability at the application layer (with the topic 5 lock providing the atomicity guarantee). The substrate's deliverable today is "attempts lock once submitted" — and that means having a state machine that knows what "locked" *is*.

## The States

Four states for a session attempt:

| State | Meaning | Mutations allowed | Reads allowed |
|---|---|---|---|
| `in_progress` | Candidate is taking the quiz | Answer submissions, index advance | Full session + answers |
| `submitted` | Candidate finished (or session expired) | None | Full session + answers |
| `scored` | Aggregate score computed | None | Full session + answers + score |
| `expired` | Session expired without submission | None | Full session + answers (whatever was captured) |

Three of the four states are **terminal** (no outgoing transitions): `submitted` and `scored` differ only by whether the aggregate has been computed yet, and `expired` is a dead-end for an unfinished attempt. Only `in_progress` allows writes.

## The Transitions

```
  [created]
      |
      v
 in_progress ----(submit)----> submitted ---(score job)---> scored
      |
      |--(expires_at passes)--> expired
```

Four transitions:

| Transition | Trigger | Side effects |
|---|---|---|
| → `in_progress` | `POST /sessions` succeeds (D11) | Session row created, expires_at set |
| `in_progress` → `submitted` | `POST /sessions/{id}/submit` succeeds | `submitted_at` set; lock released; further answers rejected |
| `in_progress` → `expired` | `server_now()` passes `expires_at` and a write attempts | `expired_at` set on the row (lazily, on next attempted write); answer attempt rejected with 410 |
| `submitted` → `scored` | Aggregate score computation (synchronous today, async-eligible later) | `aggregate_score` column populated |

The `expired` transition is **lazy**: we don't have a cron job marking sessions expired the instant their clock runs out. Instead, the next mutation that touches the session computes `server_now() > expires_at` (under the lock from topic 5) and transitions to `expired` if so. Reasons:

- No background process to maintain.
- The "moment of expiry" is unobservable from the candidate's perspective until they try to act.
- The trainer dashboard (Week 4) can derive expired status with a `WHERE expires_at < now() AND submitted_at IS NULL` query.

## Representing State In The Row

Rather than a single `status` column (which encodes the state as a string and tempts you to forget to update it), encode the state via **timestamps**:

```python
# test_management_service/app/models/session.py
class SessionRecord(Base):
    __tablename__ = "sessions"
    session_id: Mapped[UUID] = mapped_column(primary_key=True)
    session_token_hash: Mapped[str]
    user_id: Mapped[UUID]
    quiz_id: Mapped[UUID]
    question_ids: Mapped[list[UUID]] = mapped_column(JSONB)
    current_index: Mapped[int]
    started_at: Mapped[datetime]
    expires_at: Mapped[datetime]
    submitted_at: Mapped[datetime | None]
    scored_at: Mapped[datetime | None]
    aggregate_score: Mapped[float | None]
    # No `status` column — derive it.
```

The state is a function of the timestamps:

```python
def session_state(s: SessionRecord, now: datetime) -> str:
    if s.scored_at is not None:
        return "scored"
    if s.submitted_at is not None:
        return "submitted"
    if now > s.expires_at:
        return "expired"
    return "in_progress"
```

Three benefits:

1. **Impossible to be in two states at once.** A boolean `is_submitted` plus a string `status` can disagree; timestamps can't.
2. **Audit information for free.** *When* was it submitted? Look at the column.
3. **No migration needed when a state is added.** Add a timestamp column.

## Enforcing Immutability At The Endpoint

Every mutation endpoint that touches a session re-validates the state under the lock:

```python
# test_management_service/app/services/session_service.py
def assert_in_progress(session: SessionRecord, now: datetime) -> None:
    if session.submitted_at is not None:
        raise HTTPException(409, "session_already_submitted")
    if session.scored_at is not None:
        raise HTTPException(409, "session_already_scored")
    if now > session.expires_at:
        raise HTTPException(410, "session_expired")
```

Called from inside the lock in the answer endpoint:

```python
async with db.begin():
    session = await session_repo.get_for_update(db, session_id)
    if session is None:
        raise HTTPException(404, "session_not_found")
    assert_in_progress(session, server_now())   # ← gate
    # ... critical section
```

And from the submit endpoint:

```python
@router.post("/sessions/{session_id}/submit")
async def submit_session(
    session_id: UUID,
    db: DB,
    auth: SessionAuth,
) -> SubmitResponse:
    async with db.begin():
        session = await session_repo.get_for_update(db, session_id)
        if session is None:
            raise HTTPException(404, "session_not_found")
        assert_in_progress(session, server_now())     # ← same gate

        session.submitted_at = server_now()
        aggregate = await score_aggregator.compute(db, session_id)
        session.scored_at = server_now()
        session.aggregate_score = aggregate

    return SubmitResponse(...)
```

The same gate (`assert_in_progress`) is the only function that knows what "may mutate" means. Centralizing this avoids the bug where one endpoint forgets to check `submitted_at`.

## Idempotent Submit

Submission itself should be idempotent (topic 3 applied to the submit endpoint). If the candidate clicks "Submit Final" and the network drops, the client retries. The second submit must produce the same response, not a 409:

```python
@router.post("/sessions/{session_id}/submit")
async def submit_session(
    session_id: UUID,
    idempotency_key: Annotated[UUID, Header(alias="Idempotency-Key")],
    db: DB,
    auth: SessionAuth,
) -> SubmitResponse:
    async with db.begin():
        # Idempotency replay path — same as topic 3.
        existing = await idempotency_repo.get(db, idempotency_key)
        if existing is not None:
            return SubmitResponse.model_validate(existing.response_body)

        session = await session_repo.get_for_update(db, session_id)
        if session is None:
            raise HTTPException(404, "session_not_found")
        if session.submitted_at is not None:
            # Already submitted by a prior request without an idempotency key
            # (or with a different key). Return the stored result, not 409.
            response = build_submit_response(session)
        else:
            if server_now() > session.expires_at:
                raise HTTPException(410, "session_expired")
            session.submitted_at = server_now()
            aggregate = await score_aggregator.compute(db, session_id)
            session.scored_at = server_now()
            session.aggregate_score = aggregate
            response = build_submit_response(session)

        await idempotency_repo.insert(db, ..., response_body=response.model_dump())
        return response
```

The fork at "already submitted" is the subtle part: if the session is already in a terminal state when a fresh submit comes in (different idempotency key), we **don't** return 409 — we return the stored result. The state being immutable doesn't prevent *reading* it; it prevents *changing* it. A second submission of an already-submitted session is functionally a re-read with redundant intent.

The 409 case is reserved for situations where the *answer* mutation arrives after submit — that's a real conflict (the answer never made it before the finalization), not a benign replay.

## Why Not A Database `CHECK` Constraint

Tempting to add:

```sql
ALTER TABLE answers ADD CONSTRAINT
  no_writes_after_submit CHECK (
    NOT EXISTS (
      SELECT 1 FROM sessions
      WHERE sessions.session_id = answers.session_id
        AND sessions.submitted_at IS NOT NULL
    )
  );
```

Don't. `CHECK` constraints can only reference the row being inserted/updated (not other tables), so this doesn't work in standard SQL anyway. The Postgres workaround is a trigger, and triggers move business logic into the database where it's hard to test, hard to version, and surprising to future maintainers. The application-layer `assert_in_progress` gate (under the row lock from topic 5) gives equivalent guarantees and lives where the rest of the logic lives.

## Immutability Of Individual Answer Rows

A subtler immutability rule: even *during* `in_progress`, an answer row should not be updated. If the candidate "changes their mind" on question 3 after answering it, the system either:

- Refuses (the answer is final once submitted; the candidate moves forward only).
- Allows changes via a separate endpoint (`PATCH /sessions/{id}/answers/{question_id}`) that explicitly authorizes the change.

For PEP, the substrate refuses: there's no "change answer" UX. The Day-14 frontend hides the previous-question option after submission. The `uq_answer_per_question` constraint from topic 4 enforces this physically — a second `POST /sessions/{id}/answer` for the same question (same body or different) hits the unique constraint and the application returns **409 Conflict, "answer_already_submitted_for_question"**.

This is a deliberate UX choice, defensible: a quiz isn't a survey; "go back and change it" turns scoring into an interactive optimization game instead of a knowledge check. Doc the choice in the PR description.

## The Lock-Once Property

The phrase "attempts lock once submitted" in the deliverable maps to this property:

- Before submit, the session row is **locked transiently** for each answer (topic 5).
- At submit, the session row is **transitioned to `submitted`** atomically (topic 5 + this topic).
- After submit, **no transient lock matters** because the immutability check (`assert_in_progress`) fails before any write logic runs. The state, not the lock, is what prevents further mutation.

So "locked" is shorthand for "no longer in the only writable state." The DB-level row lock from `SELECT FOR UPDATE` is a means, not an end.

## Anti-Patterns

- **A `status` enum column independent of timestamps.** Drift between status and timestamp = bugs.
- **Re-implementing the gate inline in every endpoint.** Use `assert_in_progress` (or a FastAPI dependency that wraps it). One place for the rule.
- **Returning 409 on idempotent submit replay.** A retry of a submit is a replay, not a conflict.
- **Allowing answer mutations after expiry "because the user couldn't help it."** The server is authoritative (D11). If the policy is "auto-extend by 30s on network blip," encode that explicitly in `expires_at`, don't paper over it in the gate.
- **Using a Python `if session.is_submitted: raise` without holding the row lock.** Without the lock, the state can flip between the check and the write. Always gate inside the lock.
- **Database triggers enforcing state transitions.** Hidden logic, hard to test. Keep it in the application.

## Key Takeaways
- The session state machine has four states (`in_progress`, `submitted`, `scored`, `expired`) and four transitions; only `in_progress` is writable.
- Represent state via **timestamps** (`submitted_at`, `scored_at`), not an explicit `status` column — impossible to be in two states at once, audit info for free.
- A single `assert_in_progress` gate, called inside the `SELECT FOR UPDATE` lock, enforces immutability.
- Submit must itself be idempotent; a replay of a submitted session returns the stored result, not 409.
- "Attempts lock once submitted" = the gate refuses further mutation; the row lock is the mechanism, not the meaning.

---
*Prerequisites: day-12-database-transactions-and-pessimistic-locking, day-12-idempotency-for-retried-mutations, day-11-server-authoritative-state. Forward references: day-16 results aggregation reads scored sessions, day-12-edge-case-handling.*
