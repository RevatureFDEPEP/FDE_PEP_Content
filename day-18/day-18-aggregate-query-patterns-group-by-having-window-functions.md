# Aggregate Query Patterns — Group By, Having, Window Functions

> *Day 18: Trainer Dashboard Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D16's reporting endpoints answered questions about *one* candidate at a time — "give me Richard's attempts, give me Richard's per-test averages." That's per-entity work. Today the trainer dashboard asks fundamentally bigger questions: "across all candidates, which tests have the lowest pass rate?", "what's the score distribution per test?", "where does each question rank by difficulty?" Those queries are aggregate-across-the-cohort queries, and they need three SQL tools D16 only brushed past: `GROUP BY` at scale, `HAVING` to filter aggregates, and window functions to rank and percentile *without* collapsing the rows. This is the deepest SQL day of PEP. Slow down on it.

## The Shape Of The Dashboard Queries

The deliverable for today is two endpoints: `GET /reports/test/{id}` (single-test deep-dive) and `GET /reports/aggregate` (cross-test summary). Both fan out across the entire `sessions` and `answers` tables. Concretely:

- **Per-test summary row.** test_id, total attempts, distinct candidates, average score, pass rate (% above some threshold), median time-to-complete.
- **Per-question difficulty.** Inside one test, each question's correct-answer rate, plus its rank from hardest to easiest.
- **Cohort distribution.** A histogram of scores per test bucket.

Each of these is a `GROUP BY` query, a `HAVING` filter, or a window function — sometimes all three.

## GROUP BY: The Per-Test Summary

This is the workhorse query of `GET /reports/aggregate`:

```python
from sqlalchemy import select, func, case
from sqlalchemy.ext.asyncio import AsyncSession

PASS_THRESHOLD = 0.7  # 70%

async def per_test_summary(db: AsyncSession):
    stmt = (
        select(
            Session.test_id,
            func.count(Session.id).label("total_attempts"),
            func.count(func.distinct(Session.user_id)).label("distinct_candidates"),
            func.avg(Session.score * 1.0 / Session.max_score).label("avg_pct"),
            func.count(
                case(
                    (Session.score * 1.0 / Session.max_score >= PASS_THRESHOLD, 1)
                )
            ).label("passed"),
            func.avg(
                func.extract("epoch", Session.submitted_at - Session.started_at)
            ).label("avg_duration_sec"),
        )
        .where(Session.submitted_at.is_not(None))
        .group_by(Session.test_id)
        .order_by(func.count(Session.id).desc())
    )
    rows = (await db.execute(stmt)).all()
    return rows
```

A few things worth dwelling on, because half the cohort writes one of these and half doesn't:

- **Multiply by `1.0` to force float division.** `Session.score / Session.max_score` in Postgres on two `int` columns does integer division and returns 0 most of the time. The `* 1.0` (or `cast(... as float)`) is the standard idiom.
- **`func.count(func.distinct(...))`** is how SQLAlchemy expresses `COUNT(DISTINCT user_id)`. Without `distinct`, a candidate who took the test 5 times counts as 5 candidates.
- **`func.count(case(...))` for conditional counts** — same idiom as D16. `case` returns 1 when the predicate holds and NULL otherwise; `count` ignores NULL. Pass rate is then `passed / total_attempts` in Python.
- **`func.extract("epoch", interval)`** converts a `timestamptz - timestamptz` interval to seconds. Postgres returns intervals natively; the API wants a number.
- **Every non-aggregated column appears in `group_by`.** Postgres enforces this. `Session.test_id` is the only non-aggregate here, so `group_by(Session.test_id)`.

The result is one row per test_id with five precomputed metrics. The trainer dashboard renders one card per row.

## HAVING: Filter On The Aggregate

Sometimes the question is "show me only tests that have been attempted at least 10 times" — a filter that *can't* be expressed as a `WHERE` because it depends on the aggregate. That's `HAVING`:

```python
MIN_ATTEMPTS = 10

stmt = (
    select(
        Session.test_id,
        func.count(Session.id).label("total_attempts"),
        func.avg(Session.score * 1.0 / Session.max_score).label("avg_pct"),
    )
    .where(Session.submitted_at.is_not(None))
    .group_by(Session.test_id)
    .having(func.count(Session.id) >= MIN_ATTEMPTS)
    .order_by(func.avg(Session.score * 1.0 / Session.max_score).asc())
)
```

This drops tests with thin data so the dashboard doesn't show "100% pass rate" cards based on one attempt. The cohort will reflexively try to write `WHERE COUNT(*) >= 10`, which Postgres rejects with `aggregate functions are not allowed in WHERE`. The clearer mental model:

- **`WHERE`** filters input rows *before* grouping.
- **`HAVING`** filters aggregate rows *after* grouping.

You can use both in the same query. `WHERE Session.submitted_at.is_not(None)` filters before; `HAVING func.count(Session.id) >= 10` filters after. Order in the statement is `WHERE → GROUP BY → HAVING → ORDER BY`.

## Window Functions: Per-Question Difficulty Rank

Window functions are the technique that separates the "I wrote SQL in school" cohort from the "I write SQL at work" cohort. They aggregate *without collapsing rows*, which means you can compute a rank, a percentile, or a running total alongside the per-row data.

Example: for one test, rank each question from hardest (lowest correct rate) to easiest, but *return one row per question* with the rank attached.

```python
from sqlalchemy import select, func

async def per_question_difficulty(db: AsyncSession, test_id: str):
    # Subquery: per-question correct rate within this test.
    per_q = (
        select(
            Answer.question_id,
            func.count(Answer.id).label("attempts"),
            func.avg(case((Answer.is_correct.is_(True), 1.0), else_=0.0)).label("correct_rate"),
        )
        .join(Session, Session.id == Answer.session_id)
        .where(Session.test_id == test_id, Session.submitted_at.is_not(None))
        .group_by(Answer.question_id)
        .subquery()
    )

    # Main: attach a rank (1 = hardest) and a percentile across questions.
    stmt = select(
        per_q.c.question_id,
        per_q.c.attempts,
        per_q.c.correct_rate,
        func.rank().over(order_by=per_q.c.correct_rate.asc()).label("difficulty_rank"),
        func.percent_rank().over(order_by=per_q.c.correct_rate.asc()).label("difficulty_pct"),
    ).order_by("difficulty_rank")

    rows = (await db.execute(stmt)).all()
    return rows
```

Anatomy of `func.rank().over(order_by=...)`:

- **`func.rank()`** is the window aggregate — it assigns 1 to the smallest, 2 to the next, etc., with ties getting the same rank and a gap after.
- **`.over(order_by=...)`** is the window frame. Without `order_by` the rank is undefined.
- **`func.percent_rank()`** returns a value in `[0, 1]` — useful for "this question is harder than 87% of the others on this test."
- **The rank is per-question, but each question's full row is preserved.** That's the magic. With a plain `GROUP BY` you'd get back the ranked column but lose the ability to also show `attempts` and `correct_rate` together on the same row.

### Partitioning

For "rank questions within each test" (across multiple tests in one query), add a `partition_by`:

```python
func.rank().over(
    partition_by=per_q.c.test_id,
    order_by=per_q.c.correct_rate.asc(),
).label("difficulty_rank")
```

Now ranks reset at each test boundary — question 1 of test A and question 1 of test B both get rank 1. This is the pattern you reach for whenever you want "top N per group."

## ROW_NUMBER vs RANK vs DENSE_RANK

The cohort will see all three; they differ on ties:

| Function | Behavior on ties |
|---|---|
| `row_number()` | 1, 2, 3, 4 — arbitrary tiebreak |
| `rank()` | 1, 2, 2, 4 — ties share rank, then skip |
| `dense_rank()` | 1, 2, 2, 3 — ties share rank, no skip |

For "most recent attempt per user" (the D16 subquery), `row_number()` partitioned by user, ordered by `submitted_at DESC`, then filter to `row_number = 1`. That's the more performant alternative to the `MAX(submitted_at)` + self-join we used on D16, and the cohort can now write it.

## Putting Aggregates In The Response

The endpoint returns the rows mapped into a Pydantic shape. Don't return SQLAlchemy `Row` objects directly:

```python
from pydantic import BaseModel
from typing import Annotated
from decimal import Decimal

class PerTestSummary(BaseModel):
    test_id: str
    total_attempts: int
    distinct_candidates: int
    avg_pct: float
    pass_rate: float
    avg_duration_sec: float

def to_summary(row) -> PerTestSummary:
    return PerTestSummary(
        test_id=row.test_id,
        total_attempts=row.total_attempts,
        distinct_candidates=row.distinct_candidates,
        avg_pct=float(row.avg_pct or 0),
        pass_rate=(row.passed / row.total_attempts) if row.total_attempts else 0.0,
        avg_duration_sec=float(row.avg_duration_sec or 0),
    )
```

Note the explicit `float(...)` — Postgres returns `Decimal` for `AVG`, and shipping `Decimal` through `model_dump_json` raises. The same fix from D16; it comes up again every time.

## Performance: Indexes For Group-By At Scale

D16 talked about indexes for filter queries. Aggregate queries care about *different* indexes:

- **`(test_id)` on `sessions`** — covers `GROUP BY Session.test_id` and the `WHERE test_id = ...` for single-test reports.
- **`(test_id, submitted_at)` on `sessions`** — covers the common "this test, this date range" filter.
- **`(session_id)` on `answers`** — the join key; should already exist as the FK index.
- **`(question_id)` on `answers`** — covers `GROUP BY Answer.question_id` in per-question queries.

Without these, the per-test summary scans the whole `sessions` table on every dashboard load. Add the indexes in an Alembic migration today as part of the deliverable.

`EXPLAIN ANALYZE` once in dev to see "Seq Scan" become "Index Scan" — same demo as D16 but with bigger queries.

## Anti-Patterns

- **`SELECT *` with `GROUP BY` of one column.** Postgres rejects it; MySQL silently picks one row arbitrarily. Always project explicitly.
- **Using `WHERE` for aggregate filters.** `WHERE COUNT(*) >= 10` doesn't work; the cohort writes this once, gets the error, learns `HAVING`.
- **Window function without `ORDER BY`.** `func.rank().over()` is meaningless without an order. SQLAlchemy will let you write it; Postgres returns garbage.
- **Computing pass rate in Python with a separate query.** Two roundtrips for `total` and `passed` is twice the latency. Use the conditional-count idiom.
- **Returning rows that haven't been ordered.** Without `ORDER BY`, Postgres returns rows in physical-storage order, which changes when the planner changes its mind. Always order dashboard data.
- **Integer division.** `score / max_score` on two `int` columns returns 0. `* 1.0` or `cast(... as float)`.

## Key Takeaways

- `GROUP BY` collapses rows into one row per group; `HAVING` filters those grouped rows by aggregate; `WHERE` filters input rows before grouping.
- Conditional aggregates via `func.count(case(...))` and `func.avg(case(...))` compute multiple slices in one query.
- Window functions (`rank`, `dense_rank`, `row_number`, `percent_rank`) attach aggregate-like values *without* collapsing rows. Use them for ranking and percentile work.
- `partition_by` resets the window per group — the pattern for "top N per group."
- Indexes for aggregate workloads target the `GROUP BY` and join keys, not just `WHERE` columns.
- Add Alembic migrations for the indexes the dashboard depends on, today.

---
*Prerequisites: day-16-sqlalchemy-queries-joins-aggregates-subqueries, day-16-per-entity-aggregation-query-patterns, day-10-alembic-for-relational-schema-evolution.*
