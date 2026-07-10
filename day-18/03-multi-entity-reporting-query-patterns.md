# Multi-Entity Reporting Query Patterns

> *Day 18: Trainer Dashboard Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The D16 reports operated on a single axis: one candidate, all their attempts. The trainer dashboard operates on three axes at once — **candidates × tests × time** — and the query has to slice across all three without either exploding into N+1 queries or collapsing into an unreadable response. Today's topic is the design discipline around those multi-axis queries: deciding *what* the report says, designing the SQL that computes it, and shaping the response so the dashboard frontend (D19) can render it without a second round of compute. Get this right and D19 is a Tailwind-and-recharts day; get it wrong and D19 becomes "fix the API."

## A Concrete Multi-Axis Report

Make it concrete. The trainer asks: *"For the last 4 weeks, show me how each cohort performed on each test."* That report has three axes:

- **Candidate** (or, here, grouped into cohorts)
- **Test** (rows or columns of the matrix)
- **Time** (weekly buckets along the time axis)

The output is a 3-D structure compressed into a 2-D table on the dashboard: rows are tests, columns are weeks, cells contain aggregate stats — *and* the trainer can pivot rows to cohort and columns to test, etc.

## Step One: Write The SQL First

Before designing the response, write the query. The query is the constraint — if the SQL can't compute it in one round-trip, the response shape doesn't matter.

```python
from sqlalchemy import select, func, cast, Date

async def per_test_per_week(db: AsyncSession, weeks_back: int = 4):
    week_start = func.date_trunc("week", Session.submitted_at)

    stmt = (
        select(
            Session.test_id,
            week_start.label("week"),
            func.count(Session.id).label("attempts"),
            func.count(func.distinct(Session.user_id)).label("candidates"),
            func.avg(Session.score * 1.0 / Session.max_score).label("avg_pct"),
            func.count(
                case((Session.score * 1.0 / Session.max_score >= 0.7, 1))
            ).label("passed"),
        )
        .where(
            Session.submitted_at.is_not(None),
            Session.submitted_at >= func.now() - timedelta(weeks=weeks_back),
        )
        .group_by(Session.test_id, week_start)
        .order_by(Session.test_id, week_start)
    )
    rows = (await db.execute(stmt)).all()
    return rows
```

`date_trunc('week', submitted_at)` is Postgres's bucket-by-week function — it returns the Monday at 00:00 of the week containing each submission. Same idea for `'day'`, `'month'`. The grouping key is the *pair* `(test_id, week)`, so each row is one cell of the test × week matrix.

This is one query, regardless of how many tests or weeks or candidates. The cohort's first instinct will be "loop over tests in Python and query per test"; the principle is **multi-axis aggregates stay in one query**. Round-trip cost dominates everything else at dashboard latencies.

### Adding A Third Axis: Cohort

If candidates are grouped into cohorts (e.g., joined-at quarter, or training program track), add the cohort to `group_by`:

```python
group_by(User.cohort_id, Session.test_id, week_start)
```

The result is now one row per `(cohort, test, week)` triple. The dashboard pivots them into the visual the trainer asked for. The cohort joining is a join to the `users` table — which in the FDE PEP brownfield substrate lives in the user-service Postgres, not test-management's. That's a cross-service consideration: either denormalize cohort_id onto sessions at session-creation time (cheap, slightly stale), or do the cross-service lookup at report time (slow, always fresh). PEP picks denormalization; flag the trade-off when the cohort sees it.

## Step Two: Shape The Response — Nested vs Flat

Now the design question: how do you ship those rows to the dashboard?

### Option A: Flat list (recommended)

```json
{
  "axes": ["test_id", "week"],
  "rows": [
    {"test_id": "test_python_basics", "week": "2026-04-27", "attempts": 14, "candidates": 12, "avg_pct": 0.74, "pass_rate": 0.71},
    {"test_id": "test_python_basics", "week": "2026-05-04", "attempts": 18, "candidates": 17, "avg_pct": 0.79, "pass_rate": 0.83},
    {"test_id": "test_python_basics", "week": "2026-05-11", "attempts": 16, "candidates": 16, "avg_pct": 0.81, "pass_rate": 0.88},
    {"test_id": "test_js_basics",      "week": "2026-04-27", "attempts": 11, "candidates": 11, "avg_pct": 0.62, "pass_rate": 0.45},
    ...
  ],
  "meta": {
    "window": {"from": "2026-04-21", "to": "2026-05-19"},
    "total_rows": 12
  }
}
```

Why flat wins for most dashboards:

- **Maps 1:1 to the SQL result.** No backend pivot work; the database produced exactly this shape.
- **The frontend pivots into the visual it wants.** Different views need different pivots (test × week, week × test, candidate × test) — keep that flexibility in the client.
- **Stable contract across views.** When the trainer adds a new pivot tomorrow, the API doesn't change.
- **Easy to paginate.** If the matrix gets huge, the rows array is the natural pagination unit.
- **Trivial to consume in JS** — `rows.filter(r => r.test_id === ...)`, `Object.groupBy(rows, r => r.week)`. The D19 chart libraries (recharts) expect flat arrays anyway.

### Option B: Nested (sometimes appropriate)

```json
{
  "tests": {
    "test_python_basics": {
      "weeks": {
        "2026-04-27": {"attempts": 14, "avg_pct": 0.74, ...},
        "2026-05-04": {"attempts": 18, "avg_pct": 0.79, ...}
      }
    },
    "test_js_basics": { ... }
  }
}
```

Nested looks cleaner at first but creates problems:

- **Schema is dynamic.** Pydantic can't easily type "object whose keys are test_ids whose values are objects whose keys are weeks." `dict[str, dict[str, ...]]` is the best you'll get, which the cohort then can't auto-document.
- **Pivot lock-in.** The shape commits to one pivot; switching to "week is the outer axis" requires a different endpoint or a frontend transform.
- **No empty cells.** Weeks with zero attempts are simply absent, so the frontend has to fill the gaps anyway. Flat output can also be sparse, but it's clearer.

**Default to flat.** Reach for nested only when the response is genuinely a tree (e.g., trainer → assigned candidates → their attempts) and the trainer never wants the inverted view.

## The Pydantic Shape

```python
from pydantic import BaseModel
from datetime import date

class ReportRow(BaseModel):
    test_id: str
    week: date
    attempts: int
    candidates: int
    avg_pct: float
    pass_rate: float

class ReportWindow(BaseModel):
    from_: date
    to: date

    class Config:
        populate_by_name = True
        json_schema_extra = {"example": {"from": "2026-04-21", "to": "2026-05-19"}}

class ReportMeta(BaseModel):
    window: ReportWindow
    total_rows: int

class AggregateReport(BaseModel):
    axes: list[str]
    rows: list[ReportRow]
    meta: ReportMeta
```

Tip the cohort missed last week: `from` is a Python keyword, so the field is `from_` with `alias="from"`. The shape is precise enough to generate accurate OpenAPI for the frontend.

## Filling Sparse Cells

Postgres returns rows only for `(test_id, week)` pairs that *had attempts*. If "test_python_basics, week of 2026-05-04" had zero attempts, the row is missing — not present with `attempts: 0`. Two options:

1. **Let the frontend fill gaps.** Simpler backend; the frontend builds the full grid by iterating axis values and looking up rows. This is what the D19 implementation does.
2. **Backfill on the server.** Use a `generate_series` of weeks and a left join. More complex SQL, less code on the client. Worth it only if many consumers need full grids — for now, frontend fills.

PEP uses (1). The frontend is one consumer; let it do the gap-fill.

## Cross-Test Comparison Endpoint

The second big endpoint, `GET /reports/aggregate`, summarizes across *all* tests with one row per test. It's just the D18-topic-1 per-test summary query, ordered by whatever the trainer's view chose. The response is flat: `{"rows": [{test_id, attempts, avg_pct, pass_rate, ...}, ...]}`.

The single-test deep-dive `GET /reports/test/{id}` returns:

```json
{
  "test_id": "test_python_basics",
  "summary": { "attempts": 48, "candidates": 31, "avg_pct": 0.78, "pass_rate": 0.81 },
  "by_question": [
    {"question_id": "q_001", "attempts": 48, "correct_rate": 0.92, "difficulty_rank": 1},
    {"question_id": "q_002", "attempts": 48, "correct_rate": 0.37, "difficulty_rank": 12}
  ],
  "by_week": [ { "week": "2026-04-27", "attempts": 14, "avg_pct": 0.74, ... } ]
}
```

That's a *small* nested response — three sibling lists, not a generic nested tree. The shape is fixed and known; Pydantic types it cleanly. Acceptable nesting.

## Cross-Service Considerations

A test_id is opaque from test-management's view; the *name* of the test lives in question-management's Mongo. Two ways to handle it:

- **Return only test_ids; frontend resolves names via question-management.** Cheap; one extra fetch on the dashboard. This is the D18 default.
- **Backend resolves names server-side and includes them in the response.** More complete, but the reporting service now calls question-management on every dashboard request. Slow at cohort load.

Stick with the first; the D16 cross-service-data-access pattern applies. The frontend already caches question/test metadata.

## Performance Sketch

Three things matter:

1. **Single query per axis.** Don't loop in Python.
2. **Indexes on group-by columns.** From the previous topic: `(test_id, submitted_at)` covers test × week. `(user_id, submitted_at)` covers candidate × week.
3. **`LIMIT` the time window.** "Last 4 weeks" is always present in the dashboard's default query. Unbounded scans across all-time data don't scale; the API should reject a missing `weeks_back` or default it to 4.

## Anti-Patterns

- **Looping in Python over tests and issuing one query per test.** The classic mistake. One SQL query with `GROUP BY test_id, week` produces the same answer in a single round-trip.
- **Returning a deeply-nested tree the frontend then flattens anyway.** Wasted server CPU. Stay flat.
- **Letting `test_id` go un-validated.** `GET /reports/test/{id}` is trusted user input; constrain to known tests or return 404 if no sessions exist for it.
- **Tying the response shape to the current dashboard layout.** The trainer asks for a new pivot in two weeks; if the shape is the layout, the API breaks. Keep the response orthogonal to the rendering.
- **Computing pass rate twice — once for the per-test summary, once for the by-week breakdown.** Compute once in SQL, reuse the row.
- **Forgetting the time bound.** A query that scans 18 months of sessions to compute "last 4 weeks" because the filter is missing is the most common new-hire SQL mistake. The `WHERE submitted_at >=` is mandatory.

## Key Takeaways

- Multi-axis reports = `GROUP BY` on multiple columns + bucketing functions like `date_trunc`. One query.
- Default to flat response shapes (`{axes, rows, meta}`); reach for nested only when the data really is a tree.
- Let the frontend pivot and gap-fill — keep the backend's response orthogonal to the visual.
- Time windows are always required; never let an aggregate endpoint scan unbounded history.
- Cross-service name resolution stays on the frontend for read endpoints; the backend ships opaque IDs.
- Indexes on `(group_by_column, time_column)` are what makes these queries fast at cohort scale.

---
*Prerequisites: [05-cross-service-data-access-patterns.md](../day-16/05-cross-service-data-access-patterns.md), [01-rest-api-design-for-read-heavy-endpoints.md](../day-16/01-rest-api-design-for-read-heavy-endpoints.md), [01-aggregate-query-patterns-group-by-having-window-functions.md](01-aggregate-query-patterns-group-by-having-window-functions.md), [02-filter-and-sort-parameter-design-for-scale.md](02-filter-and-sort-parameter-design-for-scale.md).*
