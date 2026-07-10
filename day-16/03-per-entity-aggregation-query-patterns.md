# Per-Entity Aggregation Query Patterns

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Topic 2 covered SQLAlchemy joins, aggregates, and subqueries in general. This topic narrows in on the *specific* pattern that drives most reporting queries: **aggregating one row per entity** ("one summary row per candidate," "one summary row per test"). The shape is `GROUP BY entity_id`, with `HAVING` to filter the groups, plus the aggregates the UI wants. It sounds simple — and it is, once you've written it twice. The first time, almost everyone gets either the `GROUP BY` columns wrong, the `WHERE`-vs-`HAVING` distinction wrong, the NULL handling wrong, or the join cardinality wrong. Today the cohort writes the pattern, names the gotchas, and primes themselves for D18's trainer-dashboard aggregates ("cohort-wide score distribution," "per-question difficulty") which are the same pattern at larger scale.

## The Standard Shape

The pattern the cohort will write repeatedly:

```sql
SELECT
    entity_id,
    COUNT(*)            AS row_count,
    AVG(numeric_col)    AS avg_value,
    SUM(numeric_col)    AS total_value,
    MAX(numeric_col)    AS best_value
FROM source_table
WHERE filter_clauses
GROUP BY entity_id
HAVING COUNT(*) > 0
ORDER BY best_value DESC
LIMIT 100;
```

In SQLAlchemy:

```python
from sqlalchemy import select, func

stmt = (
    select(
        Session.user_id,
        func.count(Session.id).label("attempt_count"),
        func.avg(Session.score).label("avg_score"),
        func.sum(Session.score).label("total_score"),
        func.max(Session.score).label("best_score"),
    )
    .where(Session.submitted_at.is_not(None))
    .group_by(Session.user_id)
    .having(func.count(Session.id) > 0)
    .order_by(func.max(Session.score).desc())
    .limit(100)
)
rows = (await db.execute(stmt)).all()
```

That's the spine. Everything else in this topic is variation, gotcha, or extension on it.

## WHERE vs HAVING — The First Confusion

Many cohort members will write `WHERE COUNT(*) > 0` and get a SQL error. The distinction:

- **`WHERE`** filters rows *before* they're grouped. It cannot reference aggregates because aggregates don't exist yet at that point in the query plan.
- **`HAVING`** filters groups *after* aggregation. It can (and typically does) reference aggregates.

Examples to drill it in:

- "Only attempts submitted this month" → `WHERE submitted_at >= '2026-05-01'` (filter rows first).
- "Only candidates with more than one attempt" → `HAVING COUNT(*) > 1` (filter groups after).
- "Only complete attempts, and only candidates who completed more than one" → both: `WHERE submitted_at IS NOT NULL ... HAVING COUNT(*) > 1`.

A rule of thumb: if the filter could be applied to a single row without knowing about its neighbors, it's `WHERE`. If it requires knowing the whole group, it's `HAVING`.

## Counting And NULL

Three count variants exist, and they're different:

- **`COUNT(*)`** — count of rows in the group.
- **`COUNT(column)`** — count of rows where `column IS NOT NULL`.
- **`COUNT(DISTINCT column)`** — count of distinct non-null values.

For reporting, this distinction matters a lot:

```python
# Per-candidate: how many attempts? (all sessions, including in-progress)
func.count(Session.id).label("attempt_count")

# Per-candidate: how many completed attempts? (only submitted)
func.count(Session.submitted_at).label("completed_count")

# Per-candidate: how many distinct tests have they tried?
func.count(func.distinct(Session.test_id)).label("tests_attempted")
```

The cohort should know the difference cold; trainers can drill on it with a 30-second quiz.

## Average Of A Nullable Column

`AVG(score)` ignores NULLs. For attempts where `score IS NULL` (in-progress, not yet scored), they're excluded from the average. That's almost always what you want — averaging "score" over rows that don't have a score yet is meaningless — but the cohort should *know* this is happening, because the resulting `avg_score` is "average of completed attempts," not "average of all attempts."

If you want "average over all attempts, treating in-progress as 0," use `COALESCE`:

```python
func.avg(func.coalesce(Session.score, 0)).label("avg_score_with_zeros")
```

But that's almost never the right semantics for a candidate's report — it would punish them for in-progress attempts. The Day 16 deliverable uses `AVG(score)` (excluding NULLs) and is honest about what the number means.

## Per-Candidate Aggregation (The Deliverable Query)

The `GET /reports/user/{id}` deliverable's summary block:

```python
async def candidate_summary(db: AsyncSession, user_id: str) -> dict:
    stmt = (
        select(
            func.count(Session.id).label("total_attempts"),
            func.count(Session.submitted_at).label("completed_attempts"),
            func.avg(Session.score).label("avg_score"),
            func.max(Session.score).label("best_score"),
            func.sum(
                func.extract(
                    "epoch",
                    Session.submitted_at - Session.started_at,
                )
            ).label("total_elapsed_seconds"),
        )
        .where(Session.user_id == user_id)
    )
    row = (await db.execute(stmt)).one()
    return {
        "total_attempts": row.total_attempts,
        "completed_attempts": row.completed_attempts,
        "average_score": float(row.avg_score) if row.avg_score is not None else None,
        "best_score": row.best_score,
        "total_elapsed_seconds": int(row.total_elapsed_seconds or 0),
    }
```

Notes:

- **No `GROUP BY`** because we're filtering to one user and reducing to one summary row. The `WHERE` collapses the result to a single group implicitly.
- **`EXTRACT(EPOCH FROM interval)`** turns a Postgres interval into seconds. This is the standard way to sum durations. The cohort will see it in W4 query code repeatedly.
- **`float(row.avg_score)`** because Postgres returns `Decimal` from `AVG`, which doesn't JSON-serialize cleanly. Cast at the boundary.
- **`row.total_elapsed_seconds or 0`** because if the user has zero attempts, the SUM is NULL; coerce to 0 for the response.

## Per-Test Aggregation (Setup For D18)

The trainer dashboard on D18 will want "one row per test, with cohort-wide stats":

```python
stmt = (
    select(
        Session.test_id,
        func.count(Session.id).label("attempts"),
        func.count(func.distinct(Session.user_id)).label("unique_candidates"),
        func.avg(Session.score).label("avg_score"),
        func.percentile_cont(0.5)
            .within_group(Session.score.asc())
            .label("median_score"),
        func.min(Session.score).label("worst_score"),
        func.max(Session.score).label("best_score"),
    )
    .where(Session.submitted_at.is_not(None))
    .group_by(Session.test_id)
    .order_by(func.count(Session.id).desc())
)
```

The new piece is `percentile_cont(0.5) WITHIN GROUP (ORDER BY score ASC)` — Postgres's median. The cohort doesn't need to write this on Day 16, but they should *see* it and know it's available so they don't reimplement medians in Python.

## Per-Question Aggregation (D18 Trainer Dashboard)

Most interesting and most likely to be wrong on the first attempt: "for each question, how many candidates got it right, and what was the average time to answer?"

```python
stmt = (
    select(
        Answer.question_id,
        func.count(Answer.id).label("times_attempted"),
        func.sum(
            func.case((Answer.is_correct.is_(True), 1), else_=0)
        ).label("times_correct"),
        func.avg(Answer.elapsed_seconds).label("avg_elapsed_seconds"),
    )
    .join(Session, Answer.session_id == Session.id)
    .where(Session.submitted_at.is_not(None))
    .group_by(Answer.question_id)
    .order_by(func.count(Answer.id).desc())
)
```

The `case` inside `sum` is the SQL idiom for conditional counts — count "correct" answers without doing two queries. The cohort saw this in Topic 2's summary query; here it's at the per-entity grain. Equivalent to:

```sql
SUM(CASE WHEN is_correct = TRUE THEN 1 ELSE 0 END)
```

It's worth showing the cohort the SQL output side by side with the SQLAlchemy to anchor the mapping.

## Computed Columns In The Aggregate

The candidate summary wants "accuracy = times_correct / times_attempted." Either compute it in SQL (`100.0 * times_correct / times_attempted`) or in Python after the query returns. Recommendations:

- **In SQL** when the value is used for ordering or filtering (`ORDER BY accuracy DESC`, `HAVING accuracy > 0.5`) — the database can use it; Python can't.
- **In Python** when it's just a presentation concern.

For the candidate summary endpoint, accuracy is presentation; compute in Python:

```python
accuracy = row.times_correct / row.times_attempted if row.times_attempted else 0.0
```

The guard against zero division is non-negotiable; SQL would silently NULL, Python would raise.

## Window Functions — A Preview

The cohort doesn't need window functions today; they're worth flagging for D18. The shape:

```sql
SELECT
    user_id,
    test_id,
    score,
    RANK() OVER (PARTITION BY test_id ORDER BY score DESC) AS rank_in_test
FROM sessions
WHERE submitted_at IS NOT NULL;
```

Window functions let you compute per-row values that depend on a group (rank within group, running total, percentile, lag/lead) without collapsing the rows. Use case: "show me each attempt with its rank among that test's attempts." Topic 2 mentioned this as the alternative to subqueries; here it's the alternative to "two queries plus Python."

For Day 16: don't introduce. For D18: introduce when it makes sense.

## Working Through A Cohort Drill

Twenty-minute drill the trainer can run:

> *"Write the SQLAlchemy query for 'each candidate's average score, but only for tests where they completed at least 3 attempts on that test, sorted by who has the most consistent (lowest stddev) score.' Make it work, then explain every GROUP BY and HAVING choice."*

The query:

```python
stmt = (
    select(
        Session.user_id,
        Session.test_id,
        func.count(Session.id).label("attempts_on_test"),
        func.avg(Session.score).label("avg_score"),
        func.stddev(Session.score).label("score_stddev"),
    )
    .where(Session.submitted_at.is_not(None))
    .group_by(Session.user_id, Session.test_id)
    .having(func.count(Session.id) >= 3)
    .order_by(func.stddev(Session.score).asc())
)
```

The discussion is what they grouped on (`user_id, test_id` — each *combination* is a group), why HAVING (filter on the aggregate, not row-level), what stddev returns when N=1 (NULL — and that's why `HAVING >= 3` is sensible; otherwise stddev of one row is meaningless). It's the most efficient cohort-learning twenty minutes the trainer has on Day 16.

## Anti-Patterns

- **`WHERE` on an aggregate.** SQL error, often misread as a typo. The right tool is `HAVING`.
- **Forgetting a non-aggregated column in `GROUP BY`.** Postgres errors; MySQL silently returns garbage. Always list every non-aggregate.
- **`COUNT(*)` when you wanted `COUNT(column)`.** Counts NULLs in. Easy mistake on a nullable column.
- **Averaging without acknowledging NULL exclusion.** Document what your `AVG` is averaging *over*. The number is meaningless without that context.
- **Computing aggregates in Python by pulling all rows.** Pulls megabytes, computes in interpreted code, slow and embarrassing. Push aggregates into SQL.
- **Dividing by zero in computed columns.** Always `NULLIF(denom, 0)` in SQL or a Python guard.
- **`ORDER BY` not in `SELECT` and not in `GROUP BY`.** Some DBs accept it (sort by an aggregate not selected); some reject. Be explicit; put the sort key in the SELECT.
- **Aggregating an unfiltered table.** "Average score across all sessions" sounds fine until you realize it includes in-progress attempts with score=NULL (excluded — fine) and locked-but-not-submitted (depends on your status model). Filter explicitly with `WHERE`.
- **Joining before grouping when you only need columns from one side.** Joins multiply row counts; if `sessions` joins to `answers` and you `COUNT(*)`, you're counting answer-rows, not session-rows. Be deliberate about which table's rows the aggregate is counting.
- **Hardcoding "at least N" thresholds in the query.** Make them parameters when the UI controls them.

## Key Takeaways

- The per-entity aggregation pattern is `SELECT entity_id, aggregates FROM ... WHERE filter GROUP BY entity_id HAVING group_filter ORDER BY ...`; almost every reporting query is a variation of this shape.
- `WHERE` filters rows; `HAVING` filters groups; the distinction is non-negotiable and trips up beginners.
- `COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)` are three different things; pick deliberately.
- Conditional aggregates (`SUM(CASE WHEN ... THEN 1 ELSE 0 END)`) keep multi-aggregate queries to one round trip.
- Cast `Decimal` to `float` at the JSON boundary; guard against zero division in computed columns; document NULL handling.

---
*Prerequisites: [02-sqlalchemy-queries-joins-aggregates-subqueries.md](02-sqlalchemy-queries-joins-aggregates-subqueries.md), [01-deterministic-scoring-algorithms-exact-match.md](../day-12/01-deterministic-scoring-algorithms-exact-match.md).*
