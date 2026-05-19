# Database Transactions and Pessimistic Locking (`SELECT FOR UPDATE`)

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
A candidate's browser submits an answer. The handler reads the session row (`current_index = 4`), scores the answer, advances the index to 5, writes it back. Meanwhile a second request — same session, double-click, retry, two open tabs, doesn't matter — *also* reads `current_index = 4`, scores its answer (against question 4, the same one), advances to 5, writes. Both handlers think they advanced the session by one; the session in fact advanced by one, and one answer silently overwrote the other. This is a **read-modify-write race**. The fix is to lock the session row for the duration of the transaction so the second handler waits for the first. This file shows that pattern with SQLAlchemy 2 async, defines the precise lock scope, and explains where pessimistic locking fits relative to D11's optimistic concepts and topic 4's constraints.

## The Vocabulary

- **Transaction.** A unit of work that is *atomic* (all-or-nothing), *consistent* (constraints hold), *isolated* (concurrent transactions don't see each other's intermediate state), and *durable* (committed work survives crashes). The "I" is the one most relevant today.
- **Isolation level.** How much one transaction can see of another's in-flight work. Postgres default is **Read Committed** — you don't see uncommitted writes, but a row you read can change between two reads inside your transaction. That's the default we assume; it's adequate when paired with explicit locks.
- **Pessimistic lock.** "I'm about to modify this row; nobody else gets to read-modify-write it until I'm done." Acquired with `SELECT ... FOR UPDATE`. Holds until the transaction commits or rolls back.
- **Optimistic concurrency.** Opposite approach: don't lock, but include a version column; on write, check the version hasn't changed; if it has, abort and retry. Useful when conflicts are rare; for the answer endpoint, conflicts are *common* enough (retries, tab clicks) that pessimistic is the right default.

## What `SELECT FOR UPDATE` Does

```sql
BEGIN;
SELECT * FROM sessions WHERE session_id = '...' FOR UPDATE;
-- This row is now locked. Any other transaction issuing
-- SELECT ... FOR UPDATE on the same row BLOCKS here, waiting.
UPDATE sessions SET current_index = 5 WHERE session_id = '...';
COMMIT;
-- Lock released. The waiting transaction unblocks.
```

The key behaviors:

- The lock is **row-level**, not table-level. Other sessions of other candidates are unaffected.
- The lock is held until **commit or rollback**, not just until the next statement.
- Concurrent `SELECT ... FOR UPDATE` on the same row **blocks** (waits). A plain `SELECT` without `FOR UPDATE` does **not** block — it returns the last-committed version.
- If a deadlock is detected (two transactions waiting on each other's locks), Postgres aborts one with a `DeadlockDetected` error. The application catches and retries.

## The SQLAlchemy 2 Async Pattern

```python
# test_management_service/app/repositories/session_repo.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

async def get_for_update(
    db: AsyncSession,
    session_id: UUID,
) -> SessionRecord | None:
    stmt = (
        select(SessionRecord)
        .where(SessionRecord.session_id == session_id)
        .with_for_update()           # ← issues SELECT ... FOR UPDATE
    )
    result = await db.execute(stmt)
    return result.scalar_one_or_none()
```

Used inside the answer endpoint:

```python
# test_management_service/app/routers/sessions.py
@router.post("/sessions/{session_id}/answer")
async def submit_answer(
    session_id: UUID,
    body: AnswerSubmit,
    idempotency_key: Annotated[UUID, Header(alias="Idempotency-Key")],
    db: DB,
    auth: SessionAuth,  # validates X-Session-Token (D11)
) -> AnswerResponse:
    async with db.begin():                  # explicit transaction boundary
        session = await session_repo.get_for_update(db, session_id)
        if session is None:
            raise HTTPException(404, "session_not_found")
        if session.submitted_at is not None:
            raise HTTPException(409, "session_already_submitted")
        if server_now() > session.expires_at:
            raise HTTPException(410, "session_expired")

        # Idempotency check (topic 3) — uses same transaction
        existing = await idempotency_repo.get(db, idempotency_key)
        if existing is not None:
            return AnswerResponse.model_validate(existing.response_body)

        # Critical section: read question, score, advance index, persist.
        question_id = session.question_ids[session.current_index]
        question = await question_client.fetch(question_id)
        result = score(
            question_type=question.question_type,
            correct_options=question.correct_options,
            selected_options=body.selected_options,
        )
        await answer_repo.upsert(db, ...)
        session.current_index += 1
        await idempotency_repo.insert(db, ...)
        # Implicit COMMIT on exiting `async with db.begin():` block —
        # lock released here.

    return build_response(result, session)
```

Five disciplines visible in this pattern:

1. **Explicit transaction boundary** — `async with db.begin():`. The lock's scope is exactly this block.
2. **Lock acquired first.** Before any other DB work in the critical section. Once held, the row is ours.
3. **Validation immediately after lock.** Submitted, expired, etc. — checked while holding the lock so the decision can't change underneath us.
4. **All writes inside the block.** Scoring, answer insert, index increment, idempotency record. One atomic unit.
5. **Lock released on commit.** Implicit when the `async with` exits cleanly. On exception, it rolls back and releases.

## Explicit Lock Scope — Hold It Short

The lock blocks every other request for this session. Hold it for **microseconds**, not milliseconds. Anything that takes time goes *outside* the lock:

```python
# BAD — fetches question over HTTP while holding the row lock
async with db.begin():
    session = await session_repo.get_for_update(db, session_id)
    question = await question_client.fetch(question.id)   # ← network call!
    # ... rest of critical section
```

If the question-management-service is slow, every concurrent submission to *any* session in the queue stalls because the row lock is held during the network round-trip. Yes, the lock is per-row — but the cohort has 25 candidates and one slow upstream multiplies failure.

```python
# GOOD — fetch outside the lock, then lock + recheck
question_id_guess = await session_repo.peek_current_question_id(db, session_id)
question = await question_client.fetch(question_id_guess)

async with db.begin():
    session = await session_repo.get_for_update(db, session_id)
    if session.question_ids[session.current_index] != question.id:
        # Index advanced between peek and lock; fetch the real one.
        question = await question_client.fetch(
            session.question_ids[session.current_index]
        )
    # ... rest of critical section
```

The peek-then-lock-then-recheck pattern adds complexity in exchange for not holding the row lock during a network call. For PEP at 25 candidates the simpler pattern (network inside the lock) is fine *if* the question fetch is fast and bounded; if it ever isn't, refactor. Flag it as a known shortcut.

## What The Lock Protects — And What It Doesn't

The `SELECT FOR UPDATE` on the `sessions` row protects:

- The session row itself (`current_index`, `submitted_at`).
- The atomicity of the "read session → decide → write session" sequence.

It does **not** protect:

- The `answers` row uniqueness — that's `uq_answer_per_question` from topic 4.
- The `idempotency_records` row uniqueness — that's `uq_idempotency_key` from topic 4.
- Cross-session state (no race exists; different sessions = different rows).
- State outside the database (in-memory caches, S3 objects, MinIO uploads).

The lock and the constraints **work together**. The lock serializes the orderly path; the constraint catches anything that slips through.

## Isolation Levels — Briefly

Postgres defaults to **Read Committed**. With explicit row locks, this is adequate for our use case. For completeness:

| Level | What you see | When to use |
|---|---|---|
| Read Uncommitted | Treated as Read Committed in Postgres | Don't (Postgres treats them the same anyway) |
| Read Committed | Latest committed data; row can change between reads | Default; fine with explicit locks |
| Repeatable Read | Snapshot at transaction start; same row always returns same value | Reports, analytics queries with multiple reads |
| Serializable | Acts as if transactions ran sequentially; can abort with serialization failure | When you want correctness without explicit locks; cost: retries |

For the answer endpoint, **Read Committed + explicit `FOR UPDATE`** is the right combination. Don't reach for Serializable to avoid writing the lock — you trade an explicit, debuggable lock for a hidden retry loop.

## Lock Variants — `FOR UPDATE` vs `FOR SHARE` vs `SKIP LOCKED`

| Variant | Behavior | Use case |
|---|---|---|
| `FOR UPDATE` | Exclusive; blocks other `FOR UPDATE` and `FOR SHARE` | Read-modify-write |
| `FOR SHARE` | Shared; blocks other `FOR UPDATE` but not other `FOR SHARE` | Read consistency without modification |
| `FOR UPDATE SKIP LOCKED` | Doesn't block; skips rows held by other locks | Work-queue patterns ("give me a job nobody else is working on") |
| `FOR UPDATE NOWAIT` | Doesn't block; raises immediately if locked | When waiting would be worse than failing fast |

Today we use **`FOR UPDATE`** (the default, via `with_for_update()`). The work-queue variants are interesting for the trainer dashboard's background grading job (Week 4) but not for the answer endpoint.

## Handling Deadlock

If two transactions hold locks the other one wants, Postgres detects the cycle and aborts one with `psycopg.errors.DeadlockDetected` (wrapped by SQLAlchemy as `OperationalError`). The application catches and retries:

```python
from sqlalchemy.exc import OperationalError
from psycopg.errors import DeadlockDetected

async def submit_answer_with_retry(*args, **kwargs):
    for attempt in range(3):
        try:
            return await submit_answer(*args, **kwargs)
        except OperationalError as e:
            if not isinstance(e.orig, DeadlockDetected):
                raise
            if attempt == 2:
                raise HTTPException(503, "deadlock_retry_exhausted")
            # Brief backoff; in practice deadlocks on a single-row pattern are rare.
            await asyncio.sleep(0.02 * (attempt + 1))
```

For the single-row lock pattern in the answer endpoint, deadlock is *theoretically possible* (two transactions each locking session A then session B, in opposite order) but in practice doesn't occur because each request only locks one session. Keep the retry wrapper anyway — it's free insurance against future endpoints that lock multiple rows.

## Worked Scenario: The Double-Click

A candidate impatiently double-clicks "Submit Answer." Two requests fire ~50ms apart, both with the **same** `Idempotency-Key` (the frontend held the key in a ref):

```
T=0ms     Req A: BEGIN; SELECT...FOR UPDATE on session row  → acquires lock
T=0ms     Req A: idempotency lookup → not found
T=10ms    Req B: BEGIN; SELECT...FOR UPDATE on session row  → BLOCKS, waits
T=15ms    Req A: score, write answer, write idempotency record, increment index
T=20ms    Req A: COMMIT → lock released
T=20ms    Req B: lock acquired
T=20ms    Req B: idempotency lookup → FOUND (Req A's record)
T=21ms    Req B: replay stored response, COMMIT → no side effect
T=25ms    Req B: returns same JSON to client
```

The combination of lock + idempotency key produces the right behavior: only one set of side effects, both responses are byte-equal, the candidate sees the next question.

Now the *different* idempotency keys variant (two open tabs):

```
T=0ms     Req A (Key1): BEGIN; lock; idempotency miss; score; commit → index=5
T=10ms    Req B (Key2): BEGIN; lock; idempotency miss; ???
```

When Req B acquires the lock, the session it reads has `current_index = 5`, not 4. The question Req B's body is meant to answer is question 4; the session is now on question 5. Req B should return a **409 Conflict** ("answer submitted for wrong question index") — that's the topic 7 edge case.

## Anti-Patterns

- **Holding the lock during a slow upstream call.** Network call inside `async with db.begin():` after `with_for_update()` stalls every other writer on this row.
- **Acquiring the lock with `.first()` instead of `.scalar_one_or_none()`.** Same idea, but `.first()` doesn't raise on missing rows the same way; explicit is better.
- **Forgetting the explicit transaction boundary.** Without `async with db.begin():`, the lock is held for the lifetime of the session (the connection's autocommit/autobegin behavior varies); be explicit.
- **Reaching for `Serializable` to avoid writing the lock.** Trades an explicit serialization point for hidden retries; harder to reason about.
- **Using a Python `asyncio.Lock` to serialize writes.** Only works within a single process; useless behind multiple Uvicorn workers. The DB is the only consistent synchronization point.
- **Locking on the wrong row.** `SELECT FOR UPDATE` on `answers` (the row you're about to insert) doesn't exist yet — there's nothing to lock. Lock the parent (`sessions`), which exists and is the read-modify-write target.

## Key Takeaways
- Pessimistic locking via `SELECT ... FOR UPDATE` (SQLAlchemy: `.with_for_update()`) serializes read-modify-write on a row across concurrent transactions.
- Hold the lock for **microseconds**: acquire, validate, write, commit. Network calls go outside the lock.
- Pair with topic 4's unique constraints — lock is the orderly path, constraint is the safety net.
- Default Read Committed isolation + explicit row lock is the right combination for the answer endpoint; don't reach for Serializable.
- Deadlock is theoretically possible; wrap submission in a 3-attempt retry on `DeadlockDetected` for cheap insurance.

---
*Prerequisites: day-12-unique-constraints-and-concurrent-write-race-conditions, day-11-server-authoritative-state, day-11-async-await-patterns-in-python-web-frameworks. Forward references: day-12-state-finalization-and-immutability-patterns, day-12-edge-case-handling-for-distributed-clients.*
