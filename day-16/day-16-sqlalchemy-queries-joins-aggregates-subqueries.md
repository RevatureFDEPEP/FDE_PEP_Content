# SQLAlchemy Queries — Joins, Aggregates, Subqueries

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D10 scaffolded the reporting service with SQLAlchemy 2 async and Alembic; D11–12 used the ORM in test-management for the session/answer write path. Today the cohort writes the *read* queries that the reporting endpoints depend on — joins across `sessions`, `answers`, and `questions`; aggregates like `avg(score)`, `count(*)`, `sum(elapsed_seconds)`; and at least one subquery to get the answer right. These are the queries the SQL-illiterate side of the cohort has been quietly avoiding all month; today is the day they stop avoiding it.

## The Schema We're Querying

Reporting reads from the test-management Postgres database directly (Topic 4 defends why). The relevant tables (from D11–12 plus migrations through D15):

```
sessions
  id              uuid pk
  user_id         text
  test_id         text
  started_at      timestamptz
  expires_at      timestamptz
  submitted_at    timestamptz   null
  locked_at       timestamptz   null
  score           int           null    -- populated by D12 scoring
  max_score       int           null
  status          text                  -- 'in_progress' | 'submitted' | 'locked'

answers
  id              uuid pk
  session_id      uuid fk -> sessions.id
  question_id     text                  -- mongo _id, opaque string
  question_index  int
  selected_option text          null
  is_correct      boolean       null    -- populated at scoring time (D12)
  answered_at     timestamptz
  elapsed_seconds int           null    -- time spent on this question
  unique(session_id, question_index)
```

`questions` lives in Mongo (question-management service), so reporting can't join to it in SQL. When a response needs question text, Topic 4's cross-service call is how to get it. For Day 16 we mostly don't need question text — `question_id` is enough.

## SQLAlchemy 2 Async ORM, Quickly

The cohort already wrote `session.execute(select(...))` calls in test-management; reporting uses the same style. A reminder of the shape:

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func

async def example(db: AsyncSession):
    stmt = select(Session).where(Session.user_id == "u_42")
    result = await db.execute(stmt)
    rows = result.scalars().all()
```

`scalars()` unwraps one-column results to the entity. For multi-column (e.g., `select(Session.id, Session.score)`), use `result.all()` and unpack tuples, or use `result.mappings().all()` to get dict-like rows. The cohort will hit this distinction immediately on aggregates and should learn the muscle memory.

## Join: Session + Answers

The candidate-detail endpoint needs each attempt with its per-question records. Two ways to do this.

**Eager loading via `selectinload` (preferred for the ORM-natural case):**

```python
from sqlalchemy.orm import selectinload

stmt = (
    select(Session)
    .where(Session.user_id == user_id)
    .options(selectinload(Session.answers))
    .order_by(Session.started_at.desc())
    .limit(10)
)
sessions = (await db.execute(stmt)).scalars().all()
# sessions[0].answers is now populated; no N+1.
```

`selectinload` issues a single follow-up `SELECT ... WHERE session_id IN (...)` after the main query. Two queries total, regardless of how many sessions came back. The default lazy loading in async SQLAlchemy *raises an error* (because lazy-loading in async would require a blocking re-query); `selectinload` is what makes the relationship usable.

**Explicit join (preferred when you're projecting specific columns and skipping the ORM entity):**

```python
stmt = (
    select(
        Session.id,
        Session.test_id,
        Session.submitted_at,
        Session.score,
        Session.max_score,
        Answer.question_id,
        Answer.is_correct,
        Answer.elapsed_seconds,
    )
    .join(Answer, Answer.session_id == Session.id)
    .where(Session.user_id == user_id)
    .order_by(Session.started_at.desc(), Answer.question_index.asc())
)
rows = (await db.execute(stmt)).all()
# rows are tuples of the selected columns.
```

Either pattern is fine; pick based on what the endpoint actually needs. The summary endpoint we're building today returns a Pydantic shape that's a poor fit for the entity tree, so the explicit-join pattern with column projection is closer to the response shape.

## Aggregates: avg, count, sum

The summary block from Topic 1 needs `total_attempts`, `completed_attempts`, `average_score`, `best_score`. One query, multiple aggregates:

```python
from sqlalchemy import case

summary_stmt = select(
    func.count(Session.id).label("total_attempts"),
    func.count(case((Session.submitted_at.is_not(None), 1))).label("completed_attempts"),
    func.avg(Session.score).label("average_score"),
    func.max(Session.score).label("best_score"),
).where(Session.user_id == user_id)

row = (await db.execute(summary_stmt)).one()
# row.total_attempts, row.average_score, etc.
```

A few notes worth drilling in:

- **`func.count(Session.id)`** counts non-null `id`s (so, all rows). Equivalent to SQL `count(*)` in practice; use it when you mean "how many rows."
- **`func.count(case(...))`** is the SQL idiom for conditional counts. The `case` returns 1 when the predicate holds, NULL otherwise; `count` ignores NULLs. This is *much* faster than two separate queries with different `WHERE` clauses.
- **`func.avg(Session.score)`** returns `Decimal` from Postgres; cast or round in Python before serializing to JSON, or you'll see `"average_score": "7.5"` (string) instead of `7.5` (number). `float(row.average_score)` in the Pydantic mapping is the cleanest fix.
- **`.label(...)` matters** when you have multiple aggregates — the row attribute access depends on the label.
- **`.one()` not `.scalar()`** for multi-column aggregates: `.one()` returns the row, `.scalar()` returns the first column only.

## Group By: Per-Test Aggregates

For the candidate's "per-test performance" breakdown (a Day 17 stretch, but also reused on D18's trainer dashboard):

```python
stmt = (
    select(
        Session.test_id,
        func.count(Session.id).label("attempts"),
        func.avg(Session.score).label("avg_score"),
        func.max(Session.score).label("best_score"),
    )
    .where(Session.user_id == user_id, Session.submitted_at.is_not(None))
    .group_by(Session.test_id)
    .order_by(func.count(Session.id).desc())
)
rows = (await db.execute(stmt)).all()
```

Every non-aggregated column in the `select` must appear in `group_by` — that's a SQL rule, not a SQLAlchemy quirk. Forgetting it in Postgres produces an error; in MySQL it silently picks one row arbitrarily (one reason to be glad we're on Postgres).

Topic 7 expands the group-by patterns specifically; this is the starter version.

## Subquery: "Did Their Most Recent Attempt Pass?"

Some questions naturally decompose into a subquery. Example: "give me the most recent attempt per test for this user."

```python
from sqlalchemy import desc

# Subquery: for each (user, test), the most recent submitted_at.
latest_per_test = (
    select(
        Session.test_id,
        func.max(Session.submitted_at).label("latest_at"),
    )
    .where(Session.user_id == user_id, Session.submitted_at.is_not(None))
    .group_by(Session.test_id)
    .subquery()
)

# Main: join sessions to the subquery to get the matching row.
stmt = (
    select(Session)
    .join(
        latest_per_test,
        (Session.test_id == latest_per_test.c.test_id)
        & (Session.submitted_at == latest_per_test.c.latest_at),
    )
    .where(Session.user_id == user_id)
)
latest_sessions = (await db.execute(stmt)).scalars().all()
```

The `.c` accessor (`latest_per_test.c.test_id`) is how you reference columns of a subquery. The pattern — `max()` in a subquery, then re-join to the original table on the `max` value — is one of the most common SQL idioms; the cohort will use it for "most recent X per Y" countless times.

A note: this *can* be done with a window function (`ROW_NUMBER() OVER (PARTITION BY test_id ORDER BY submitted_at DESC)`), which is more performant on large tables. Window functions are out of scope for Day 16 but flag them for the cohort as the next-level technique when they encounter the subquery-is-slow situation in practice.

## CTE Alternative

The same query can be written with a CTE (Common Table Expression), which is often more readable:

```python
latest_per_test = (
    select(
        Session.test_id,
        func.max(Session.submitted_at).label("latest_at"),
    )
    .where(Session.user_id == user_id, Session.submitted_at.is_not(None))
    .group_by(Session.test_id)
    .cte("latest_per_test")
)

stmt = (
    select(Session)
    .join(
        latest_per_test,
        (Session.test_id == latest_per_test.c.test_id)
        & (Session.submitted_at == latest_per_test.c.latest_at),
    )
    .where(Session.user_id == user_id)
)
```

`.subquery()` produces an inline derived table; `.cte()` produces a `WITH latest_per_test AS (...)` block. Postgres handles both well; pick on readability.

## Logging The SQL

When a query isn't behaving, see what was actually sent. The reporting service should have an env-gated `SQLALCHEMY_ECHO` flag:

```python
engine = create_async_engine(DATABASE_URL, echo=os.getenv("SQL_ECHO") == "1")
```

Run with `SQL_ECHO=1` locally to see every query in the logs. Don't enable it in production (PII, performance), but in dev it's the fastest way to confirm "wait, why is this issuing 47 queries?"

## Connecting To Test-Management's Postgres

The reporting service has its own database (`reporting_db`, scaffolded D10). For Day 16 we *also* configure a read-only connection to `test_management_db`:

```python
# reporting-service/app/db.py
test_mgmt_engine = create_async_engine(
    os.getenv("TEST_MGMT_DATABASE_URL"),  # different DB, same Postgres instance in Compose
    pool_size=5,
    pool_pre_ping=True,
)
TestMgmtSession = async_sessionmaker(test_mgmt_engine, expire_on_commit=False)
```

The cohort should know:

- **Read-only by convention.** The reporting service is a *reader* of test-management's data. Operationally, that means create a `reporting_reader` Postgres role with `SELECT` only and use its credentials. PEP defers role-based DB access to Phase 2 but the cohort should add a TODO.
- **Shared models or duplicate models?** Defining `Session` and `Answer` ORM models in the reporting service that mirror test-management's models is a form of duplication. Topic 4 discusses this; the short answer for PEP is "duplicate the models, accept the maintenance cost, gain isolation."
- **Pool size is small.** Reporting is read-mostly and traffic is low; `pool_size=5` is plenty. Don't over-provision pools — they consume connections on the Postgres side that other services need.

## Performance Sketch

The cohort doesn't need to be performance experts today, but they should know enough not to write `O(n)` queries:

- **Index `(user_id, started_at)` and `(test_id, started_at)` on `sessions`.** The reporting queries filter on these constantly. The D10 Alembic migration should add the indexes today.
- **Index `(session_id, question_index)` on `answers`.** Already unique-indexed for the D12 constraint; reporting reuses it.
- **`EXPLAIN ANALYZE` is your friend.** Once during the cohort, the trainer should run `EXPLAIN ANALYZE` on one of these queries to show a sequential scan vs. an index scan in the actual output. It's the moment SQL performance becomes concrete.

Premature optimization is real, but for a reporting service that fans out across attempt history, missing indexes turns a 5ms query into a 5-second query the moment cohort size grows.

## Anti-Patterns

- **N+1 from lazy loading.** Async lazy loading raises, but synchronous-style `for session in sessions: print(session.answers)` after a `selectinload`-less query is a classic mistake. Always think about how many queries this loop runs.
- **`func.count()` without an argument.** Works in raw SQL (`COUNT(*)`); in SQLAlchemy, write `func.count(Session.id)` or `func.count()` (the latter compiles to `COUNT(*)`). Be consistent.
- **Forgetting `.label()`.** Without it you'll do `row[1]` instead of `row.average_score` and your code becomes unreadable.
- **`order_by` without `desc()` for "most recent" queries.** Default is ascending; "most recent" wants `.desc()`. The cohort will get this wrong once and notice immediately because the test breaks.
- **Selecting whole entities when you need three columns.** Wastes memory and bandwidth. Project the columns you actually return.
- **Subquery when a join works.** Some cohort members reach for subqueries reflexively; if a plain join expresses the intent, use the join. The subquery in this topic is for the *specific* case of "most recent per group" which can't be expressed as a plain join.
- **Using `session.execute(text("SELECT ..."))`.** Falling back to raw SQL strings loses the type-safety and refactoring benefits of the ORM. Only acceptable for pathological cases where the ORM can't express the query (rare; window functions are about the threshold).

## Key Takeaways

- `selectinload` is the async-SQLAlchemy way to populate relationships without N+1; explicit column-projection joins are preferred when the response shape doesn't match the entity tree.
- Aggregates compose: `func.count(case(...))`, `func.avg`, `func.max` together produce summary rows in one query.
- `group_by` requires every non-aggregated column; Postgres enforces this, which is a feature.
- Subqueries via `.subquery()` (or `.cte()`) handle "most recent per group" cleanly; window functions are the more-advanced alternative.
- Add indexes for the columns reporting filters on; without them, query time grows linearly with the table.

---
*Prerequisites: day-10-alembic-for-relational-schema-evolution, day-10-fastapi-service-scaffolding-conventions, day-12-database-transactions-and-pessimistic-locking.*
