# Filter And Sort Parameter Design For Scale

> *Day 18: Trainer Dashboard Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D16 introduced the `?page=`/`?size=`/`?sort=`/`?from=` query-parameter pattern for the per-candidate attempts endpoint. That was filter and sort for *one* user's data — small scale, simple shape, no real performance concerns. Today the trainer dashboard amplifies the same problem along three new dimensions: many filters at once, sorts on *computed* columns (not stored columns), and aggregate result sets the trainer wants to slice in arbitrary ways. The goal of this topic is to teach the cohort how to design a filter/sort surface that's *expressive* enough for the dashboard, *safe* (no SQL injection, no eval), and *fast* (the database does the work, not Python). It is the difference between an API that scales to 25-person cohort weeks and an API that hangs for 12 seconds because someone clicked a sort header.

## The Expanded Query Surface

A representative trainer-dashboard request:

```
GET /reports/aggregate
    ?from=2026-04-21
    &to=2026-05-19
    &test_id=test_python_basics
    &test_id=test_js_basics            (repeated → multi-value)
    &min_attempts=10
    &min_pass_rate=0.5
    &sort=avg_pct:asc
    &page=1
    &size=20
```

Several things going on:

- **Range filter on time** (`from`, `to`).
- **Multi-value exact filter on test_id** (zero or more test_ids).
- **Aggregate filters** (`min_attempts`, `min_pass_rate`) — these become `HAVING`, not `WHERE`.
- **Sort on a computed column** (`avg_pct` doesn't exist on any table; it's `AVG(score * 1.0 / max_score)`).
- **Pagination** on the aggregate result.

Each is a place the cohort can write something that breaks at scale. Each has a discipline.

## Exact-Match Filters: Indexable, Cheap, Safe

`test_id=test_python_basics` is the easiest case. The SQL is `WHERE test_id = :test_id`. The Postgres planner uses the `(test_id)` index. The Pydantic model validates it's a string. Done.

```python
if params.test_id:
    stmt = stmt.where(Session.test_id == params.test_id)
```

Multi-value (`test_id=A&test_id=B`): use `IN`. FastAPI parses repeated query keys as a list.

```python
class AggregateParams(BaseModel):
    test_id: list[str] = Field(default_factory=list)

def aggregate_params(
    test_id: Annotated[list[str], Query()] = [],
) -> AggregateParams:
    return AggregateParams(test_id=test_id)
```

```python
if params.test_id:
    stmt = stmt.where(Session.test_id.in_(params.test_id))
```

SQLAlchemy parameterizes the `IN` clause — no injection risk, no string interpolation. Cohort members who learned SQL "by `format(query, value)`" will need to unlearn the habit; the answer is *always* "use the ORM's parameter binding."

A caution on unbounded `IN`: if the trainer somehow sends 10,000 test_ids, Postgres has to plan a query with 10,000 parameters. Cap the list at the API layer:

```python
test_id: list[str] = Field(default_factory=list, max_length=50)
```

Pydantic enforces the cap before the SQL runs.

## Range Filters: Use Carefully

`from=2026-04-21&to=2026-05-19` becomes `WHERE submitted_at BETWEEN :from AND :to`. Index requirement: a leading column or composite index that includes `submitted_at`.

```python
if params.from_:
    stmt = stmt.where(Session.submitted_at >= params.from_)
if params.to:
    # Exclusive upper bound on the next day to include the whole `to` day.
    stmt = stmt.where(Session.submitted_at < params.to + timedelta(days=1))
```

A subtlety the cohort will hit: `submitted_at` is a `timestamptz`, `params.to` is a `date`. `submitted_at <= '2026-05-19'` excludes events at `2026-05-19 14:32:00` — the date is treated as midnight. The `+ timedelta(days=1)` with `<` is the idiom for "inclusive day boundary."

Range filters are inherently less selective than exact filters; the planner uses indexes less aggressively. Two mitigations:

- **Always require a range bound at the API.** A query with no `from`/`to` would scan all history. Default `from` to "30 days ago" if not provided.
- **Cap the range width.** "Max 90 days" prevents a curious trainer from pulling a year of data in one request.

## Aggregate Filters: HAVING, Not WHERE

`min_attempts=10` and `min_pass_rate=0.5` filter on aggregate values that don't exist until after `GROUP BY` has run. They become `HAVING`, per the first topic of the day:

```python
if params.min_attempts is not None:
    stmt = stmt.having(func.count(Session.id) >= params.min_attempts)

if params.min_pass_rate is not None:
    pass_count = func.count(case((Session.score * 1.0 / Session.max_score >= 0.7, 1)))
    total_count = func.count(Session.id)
    stmt = stmt.having(pass_count * 1.0 / total_count >= params.min_pass_rate)
```

The cohort's first impulse: "let me filter in Python after the query runs." That works for small result sets but is the wrong instinct — the database is faster than Python at filtering, and filtering in SQL keeps pagination correct. (Filter in Python and `page=2&size=20` returns the wrong page.)

## Sort Parameter Design: The Allowlist

Sort is where the cohort most often opens a hole. The naive design — `?sort_by=` accepts any column name — is the classic SQL injection / data exposure vector. Two of three pitfalls:

- **Sort by a column the user shouldn't see** (`?sort=password_hash:asc` — okay, no password hash on `sessions`, but the pattern is what matters).
- **Inject SQL into the sort expression** if the implementation uses string interpolation.

The discipline: **sort_by is an allowlist**, not a free-text field.

```python
ALLOWED_SORTS = {
    "avg_pct": "avg_pct",          # column label in the SELECT
    "attempts": "total_attempts",
    "candidates": "distinct_candidates",
    "pass_rate": None,              # computed in Python, not sortable
    "test_id": "test_id",
}

def parse_sort(sort: str) -> tuple[str, str]:
    # Pydantic regex ensures the shape is "field:dir"; split safely.
    field, _, direction = sort.partition(":")
    if field not in ALLOWED_SORTS or ALLOWED_SORTS[field] is None:
        raise HTTPException(400, f"sort field not allowed: {field}")
    if direction not in ("asc", "desc"):
        raise HTTPException(400, f"sort direction not allowed: {direction}")
    return ALLOWED_SORTS[field], direction
```

Apply it:

```python
column_label, direction = parse_sort(params.sort)
order_col = getattr(some_subquery.c, column_label)
stmt = stmt.order_by(order_col.desc() if direction == "desc" else order_col.asc())
```

Two patterns explicitly avoided:

- **`stmt.order_by(text(f"{user_sort} {user_dir}"))`** — interpolates user input into raw SQL. Classic injection.
- **`getattr(Session, user_field).desc()`** — slightly safer (no SQL injection) but exposes *every* column on the model, including columns the API should not be sortable by. The allowlist is the right layer.

The cohort should write the allowlist *once per endpoint*. It's the smallest piece of code that prevents the biggest class of mistake.

### Sort On Computed Columns

The interesting case for D18: sort by `avg_pct`, which is `AVG(score * 1.0 / max_score)`, not a real column. SQLAlchemy handles it via the label:

```python
stmt = (
    select(
        Session.test_id,
        func.avg(Session.score * 1.0 / Session.max_score).label("avg_pct"),
        ...
    )
    .group_by(Session.test_id)
    .order_by(text("avg_pct DESC"))   # references the label
)
```

`order_by(text("avg_pct DESC"))` is acceptable here *only because* the label was set by us and the sort direction is from a validated allowlist. The cohort should pause: "I'm using `text()`, am I sure the string is server-controlled?" Yes — both pieces are. Safe.

Alternatively, reference the column expression by name:

```python
avg_pct = func.avg(Session.score * 1.0 / Session.max_score).label("avg_pct")
stmt = select(Session.test_id, avg_pct, ...).group_by(Session.test_id)
stmt = stmt.order_by(avg_pct.desc() if direction == "desc" else avg_pct.asc())
```

This is the cleanest pattern; no `text()` at all.

## Indexes For The Filter+Sort Combinations

The dashboard's hot paths determine the indexes:

| Filter / Sort | Index |
|---|---|
| `WHERE submitted_at >= :from AND submitted_at < :to` + `GROUP BY test_id` | `(test_id, submitted_at)` composite |
| `WHERE test_id = :id` + per-question aggregation | `answers(session_id)` (already exists for FK), `(question_id)` |
| `WHERE user_id = :id` (D16 endpoint, also used for trainer drilldown) | `(user_id, submitted_at)` composite |

The composite index `(a, b)` covers:

- `WHERE a = ...` (uses index on `a`).
- `WHERE a = ... AND b BETWEEN ... AND ...` (uses index on both).
- `WHERE a = ... ORDER BY b` (index supports the sort, no extra sort step).
- Sometimes `ORDER BY a, b` alone (depending on planner).

It does *not* cover `WHERE b BETWEEN ...` alone (no leading column). The cohort should know that index *order matters*: `(test_id, submitted_at)` is not the same as `(submitted_at, test_id)`.

Add the indexes in an Alembic migration, today. Two or three indexes is plenty for D18; resist over-indexing — every index slows down writes.

## Pagination At Aggregate Scale

D16's pagination was straightforward: `LIMIT/OFFSET` on a `WHERE user_id = ...` query. The aggregate endpoint paginates *after* aggregation, which means the total count is the number of `GROUP BY` rows, not session rows:

```python
total_stmt = select(func.count()).select_from(
    select(Session.test_id).where(...).group_by(Session.test_id).subquery()
)
total = (await db.execute(total_stmt)).scalar_one()
```

Then `LIMIT :size OFFSET (:page - 1) * :size` on the main aggregate.

At very large aggregate result sets (hundreds of tests), `OFFSET` gets expensive — Postgres still computes all the prior rows to skip them. Cursor pagination (keyset) is the production pattern but overkill at PEP scale. Keep `OFFSET/LIMIT` and a max page count.

## Putting It Together

A complete aggregate-endpoint dependency:

```python
from typing import Annotated, Literal
from datetime import date, timedelta
from pydantic import BaseModel, Field
from fastapi import Depends, Query

ALLOWED_SORTS = {"avg_pct", "attempts", "candidates", "pass_rate", "test_id"}

class AggregateParams(BaseModel):
    from_: date = Field(default_factory=lambda: date.today() - timedelta(days=30), alias="from")
    to: date = Field(default_factory=date.today)
    test_id: list[str] = Field(default_factory=list, max_length=50)
    min_attempts: int | None = Field(None, ge=0)
    min_pass_rate: float | None = Field(None, ge=0, le=1)
    sort: str = Field("avg_pct:desc", pattern=r"^[a-z_]+:(asc|desc)$")
    page: int = Field(1, ge=1, le=100)
    size: int = Field(20, ge=1, le=100)

    class Config:
        populate_by_name = True

def aggregate_params(
    from_: Annotated[date | None, Query(alias="from")] = None,
    to: Annotated[date | None, Query()] = None,
    test_id: Annotated[list[str], Query()] = [],
    min_attempts: Annotated[int | None, Query(ge=0)] = None,
    min_pass_rate: Annotated[float | None, Query(ge=0, le=1)] = None,
    sort: Annotated[str, Query(pattern=r"^[a-z_]+:(asc|desc)$")] = "avg_pct:desc",
    page: Annotated[int, Query(ge=1, le=100)] = 1,
    size: Annotated[int, Query(ge=1, le=100)] = 20,
) -> AggregateParams:
    field = sort.split(":")[0]
    if field not in ALLOWED_SORTS:
        raise HTTPException(400, f"sort field not allowed: {field}")

    # Enforce max range width.
    today = date.today()
    f = from_ or (today - timedelta(days=30))
    t = to or today
    if (t - f).days > 90:
        raise HTTPException(400, "range too large; max 90 days")
    if t < f:
        raise HTTPException(400, "to must be >= from")

    return AggregateParams(
        from_=f, to=t, test_id=test_id, min_attempts=min_attempts,
        min_pass_rate=min_pass_rate, sort=sort, page=page, size=size,
    )

AggregateParamsDep = Annotated[AggregateParams, Depends(aggregate_params)]
```

Defaults provided, bounds enforced, allowlist applied, error codes 400 (not 500), max range capped. That's a defensible API surface for the dashboard to call.

## Anti-Patterns

- **`text(f"ORDER BY {sort_field} {sort_dir}")`** — SQL injection. Never.
- **`getattr(Model, user_field)`** — exposes every column to sort; not injection but data leak.
- **Unbounded `IN` list.** Cap with `max_length`.
- **No range bound on time filters.** A missing `from` scans all history. Default it.
- **Filtering in Python after the query.** Breaks pagination, slow at scale. Filter in SQL.
- **Aggregate filters via `WHERE`.** Use `HAVING` for filters on aggregates.
- **Indexes for every column "just in case".** Each index slows writes; index the actual hot paths.
- **Returning `Pydantic` errors as 422 with the user's raw input.** Pydantic does this by default and it's mostly fine, but for the cohort's API, audit what the validation error surface looks like; sensitive default values shouldn't leak into error messages.
- **`page=1&size=1000000` accepted.** Always cap `size`. PEP caps at 100.

## Key Takeaways

- Exact filters are indexable and cheap; range filters need an upper bound and should be capped; aggregate filters become `HAVING`.
- Sort parameters are an **allowlist**, never a free-text field. The allowlist is one dict; the safety it buys is enormous.
- Sort on computed columns by labeling the expression in the SELECT and reordering by the label; no `text()` interpolation of user input.
- Composite indexes `(filter_col, sort_col)` cover the common dashboard queries; index order matters.
- Validate at the Pydantic / FastAPI layer; reject malformed input with 400, not 500.
- Cap pagination size and range width at the API; defaults matter at least as much as maximums.

---
*Prerequisites: day-16-filtering-pagination-and-sorting-patterns-in-fastapi, day-18-aggregate-query-patterns-group-by-having-window-functions, day-10-alembic-for-relational-schema-evolution.*
