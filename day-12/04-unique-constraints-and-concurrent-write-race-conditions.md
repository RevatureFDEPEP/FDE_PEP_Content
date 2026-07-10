# Unique Constraints and Concurrent-Write Race Conditions

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Two requests arrive at the server at the same moment. Both pass the same "does this row already exist?" check at the same nanosecond. Both decide "no — I should insert it." Both insert. You now have two rows where the application logic assumed at most one. This is a **concurrent-write race condition**, and you cannot fix it in Python — the application code has already done the right thing. The fix is in the database: a **unique constraint** that rejects the second insert with an integrity error the application can catch and respond to. This file shows the race, names the constraints we need for the session/answer schema, and previews how topic 5's pessimistic lock complements (not replaces) constraints.

## The Race, Concretely

Consider a naive idempotency-key insert (topic 3):

```python
async def insert_if_new(db, key: UUID, fingerprint: str, response: dict):
    existing = await db.execute(
        select(IdempotencyRecord).where(IdempotencyRecord.key == key)
    )
    if existing.scalar_one_or_none() is not None:
        return existing  # replay
    db.add(IdempotencyRecord(key=key, ..., response_body=response))
    await db.flush()
```

Two requests, same `key`, arriving concurrently:

```
T=0ms   Request A: SELECT WHERE key=K  → 0 rows
T=0ms   Request B: SELECT WHERE key=K  → 0 rows
T=1ms   Request A: INSERT key=K         → success
T=1ms   Request B: INSERT key=K         → success (no constraint)
                                          ^^ two rows, same key
```

The Python logic was correct. The interleaving wasn't. Without a database-level constraint, every "check then insert" idiom has this race window.

## What A Unique Constraint Does

A `UNIQUE` constraint on a column (or column tuple) tells the database to **reject** any insert or update that would create a duplicate. The check happens **inside the storage engine**, after row-level locks have been acquired — it sees both inserts and serializes them. The second one fails.

```python
class IdempotencyRecord(Base):
    __tablename__ = "idempotency_records"
    key: Mapped[UUID] = mapped_column(primary_key=True)  # PK ⇒ unique
    # ...
```

A primary key is unique by definition; a non-PK column needs explicit:

```python
class AnswerRecord(Base):
    __tablename__ = "answers"
    id: Mapped[int] = mapped_column(primary_key=True)
    session_id: Mapped[UUID]
    question_id: Mapped[UUID]
    # ... selected_options, score, algorithm

    __table_args__ = (
        UniqueConstraint("session_id", "question_id", name="uq_answer_per_question"),
    )
```

The `(session_id, question_id)` pair is unique: a session has at most one answer per question. The race-protected version of "is this candidate's first answer to this question?" is "try to insert; if you get `IntegrityError`, the row already existed."

## The Two Constraints The Substrate Needs Today

Day 12 adds two unique constraints:

| Constraint | Table | Columns | Prevents |
|---|---|---|---|
| `uq_idempotency_key` | `idempotency_records` | `(key)` | Duplicate processing of the same retried submission |
| `uq_answer_per_question` | `answers` | `(session_id, question_id)` | Two answer rows for the same question in the same session |

Both are added in an Alembic migration today (carrying forward the D10 schema-evolution discipline). Both are non-negotiable: without them, the idempotency machinery in topic 3 and the attempt-lock machinery in topic 5 leak.

## Catching The Integrity Error

When the constraint fires, SQLAlchemy raises `IntegrityError` wrapping the underlying Postgres `UniqueViolation`. The endpoint handles it explicitly:

```python
# test_management_service/app/repositories/idempotency_repo.py
from sqlalchemy.exc import IntegrityError
from psycopg.errors import UniqueViolation

async def insert_or_get(
    db: AsyncSession,
    record: IdempotencyRecord,
) -> tuple[IdempotencyRecord, bool]:
    """Returns (record, was_inserted). was_inserted=False means we lost the race."""
    try:
        db.add(record)
        await db.flush()
        return record, True
    except IntegrityError as e:
        if not isinstance(e.orig, UniqueViolation):
            raise
        await db.rollback()
        # Lost the race — fetch what the winner inserted.
        existing = await db.execute(
            select(IdempotencyRecord).where(IdempotencyRecord.key == record.key)
        )
        return existing.scalar_one(), False
```

Three points:

- **Pattern is `try insert / on IntegrityError fetch`**, not "check then insert." The DB does the check atomically with the insert.
- **`await db.rollback()` is mandatory** after the integrity error. Postgres transactions enter a failed state on error and reject all subsequent statements until the transaction is rolled back or closed.
- **`was_inserted=False` is normal flow**, not an exception. The caller checks the flag and either returns the freshly-inserted response or replays the stored one.

## Constraints Catch Races Across All Sources

A subtle but important property: unique constraints catch races between *any* two writers — same process, different processes, different services, replay attacks, cron jobs, manual SQL. There's no shared application state to maintain, no distributed lock service to introduce. The DB is the synchronization point.

This is why constraints are the right answer for *correctness*, while application-level locks (topic 5) are the right answer for *performance and orderly serialization*. They solve different problems and you typically want both.

## Constraint vs Lock — When To Use Each

| | Unique constraint | Pessimistic lock (`SELECT FOR UPDATE`) |
|---|---|---|
| What it does | Rejects duplicate inserts/updates | Serializes readers/writers on a row |
| Latency cost | Near-zero on the happy path | Blocks waiters until lock released |
| Scope | A column tuple | A specific row, for a transaction |
| Detect race? | Yes, after the fact (IntegrityError) | No race — prevents concurrent access |
| Recover from race? | Catch + retry/replay | N/A |
| Use for | "At most one of these rows" | "Read-modify-write must be atomic" |

For the answer endpoint:

- **`uq_answer_per_question`** prevents two answer rows for the same question. Catches the race after the fact via IntegrityError.
- **`SELECT FOR UPDATE` on the session row** (topic 5) prevents two concurrent submissions from advancing `current_index` past each other. Serializes them so the scoring and index-advance happen atomically.

Both are needed. The constraint is the safety net; the lock is the orderly path.

## A Common Anti-Pattern The Constraints Prevent

Without constraints, well-meaning code tries to fix races at the application layer with retries or sleeps:

```python
# BAD — does not actually prevent the race
for attempt in range(3):
    if not await record_exists(db, key):
        await db.insert(record)
        return
    await asyncio.sleep(0.05 * 2 ** attempt)
raise SomeException("gave up")
```

This makes the race *less likely* but not impossible — the race window is now 50ms wide instead of 1ms, but it's still there. Worse, you've now built a flaky system that "usually works." Add the constraint and the race becomes structurally impossible, regardless of timing.

## Migration For The Two Constraints

```python
# alembic/versions/2026_05_19_add_uniq_answer_per_question.py
def upgrade():
    op.create_unique_constraint(
        "uq_answer_per_question",
        "answers",
        ["session_id", "question_id"],
    )
    op.create_unique_constraint(
        "uq_idempotency_key",
        "idempotency_records",
        ["key"],
    )


def downgrade():
    op.drop_constraint("uq_idempotency_key", "idempotency_records")
    op.drop_constraint("uq_answer_per_question", "answers")
```

Runs against the dev compose-stack DB (D10 pattern) and against the ECS-RDS substrate (Day 7's pre-provisioned AWS).

## Composite Keys vs Composite Unique Constraints

Two ways to enforce "one answer per (session, question)":

1. **Composite primary key.** `__table_args__` declares `(session_id, question_id)` as PK; no separate `id` column.
2. **Surrogate `id` PK + composite unique constraint.** What we did above.

Pattern 2 is slightly preferred because:

- An autoincrement `id` is convenient for logs ("see answer #4271") and for foreign keys.
- Composite PKs make ORM joins more verbose.
- The behavior is equivalent — duplicates are rejected either way.

For PEP, pick one and be consistent across the schema. The substrate uses pattern 2; stick with it.

## Anti-Patterns

- **"Check then insert" at the application layer with no DB constraint.** Always racy.
- **Swallowing `IntegrityError` without distinguishing the constraint that fired.** A `UniqueViolation` on the idempotency key means "you lost the race, fetch and replay." A `ForeignKeyViolation` means "the session you're answering doesn't exist." Don't conflate them.
- **Catching `IntegrityError` without `await db.rollback()`.** The transaction is poisoned; the next statement fails mysteriously.
- **Using a unique constraint to enforce a *business rule* rather than a *physical invariant*.** "User can only submit one quiz per day" is a business rule that may change; a unique constraint on `(user_id, date)` makes it hard to change. Reserve unique constraints for structural invariants ("one answer row per question per session").
- **Skipping the constraint because "the application code is careful."** Even careful code races; the only defense is the DB layer.

## Key Takeaways
- "Check then insert" is racy by definition; the fix is a **unique constraint** at the DB level.
- Today's substrate adds `uq_answer_per_question` and `uq_idempotency_key`.
- Pattern is `try insert / on IntegrityError fetch the winner's row`; always `await db.rollback()` after the error.
- Constraints catch *all* races (any writer, any source); locks (topic 5) provide orderly serialization. Use both — they solve different problems.
- Application-layer retries and sleeps don't fix races; they narrow the window. Structural fixes belong in the schema.

---
*Prerequisites: [03-idempotency-for-retried-mutations.md](03-idempotency-for-retried-mutations.md), [02-alembic-for-relational-schema-evolution.md](../day-10/02-alembic-for-relational-schema-evolution.md). Forward references: [05-database-transactions-and-pessimistic-locking.md](05-database-transactions-and-pessimistic-locking.md).*
