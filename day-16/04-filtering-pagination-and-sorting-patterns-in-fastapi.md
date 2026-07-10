# Filtering, Pagination, And Sorting Patterns In FastAPI

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The `GET /reports/user/{id}/attempts` endpoint can't safely return every attempt a candidate has ever made — even at PEP scale, a year of activity is hundreds of rows, and on D18 the trainer dashboard queries fan out across an entire cohort. So today the cohort learns the three patterns that turn a naive list endpoint into a well-behaved one: **filtering** (by date range, by test, by status), **pagination** (offset/limit with proper meta), and **sorting** (by submitted_at, by score, ascending or descending). FastAPI makes this pleasant with query-parameter dataclasses injected as dependencies — the same DI pattern from D11 — and the result is endpoints with self-documenting contracts.

## The Endpoint We're Building

```
GET /reports/user/{user_id}/attempts
    ?page=1
    &size=20
    &test_id=test_python_basics       (optional)
    &from=2026-04-01                  (optional, inclusive)
    &to=2026-05-19                    (optional, inclusive)
    &status=submitted                 (optional: in_progress | submitted | locked)
    &sort=submitted_at:desc           (optional, default submitted_at:desc)
```

Response shape:

```json
{
  "items": [ ... ],
  "meta": {
    "page": 1,
    "size": 20,
    "total": 137,
    "total_pages": 7,
    "has_next": true,
    "has_prev": false
  }
}
```

The contract is the same shape the trainer dashboard endpoints will return on D18, so getting it right today amortizes across the week.

## Query Params As A Pydantic Dependency

The clean way to express "this endpoint takes seven optional query params" without polluting the route signature is a dependency-injected Pydantic model:

```python
from typing import Annotated, Literal
from datetime import date
from fastapi import Depends, Query
from pydantic import BaseModel, Field

class AttemptListParams(BaseModel):
    page: int = Field(1, ge=1, description="1-indexed page number")
    size: int = Field(20, ge=1, le=100, description="Items per page; capped at 100")
    test_id: str | None = Field(None, description="Filter to one test")
    from_: date | None = Field(None, alias="from", description="Inclusive lower bound on submitted_at")
    to: date | None = Field(None, description="Inclusive upper bound on submitted_at")
    status: Literal["in_progress", "submitted", "locked"] | None = None
    sort: str = Field("submitted_at:desc", pattern=r"^[a-z_]+:(asc|desc)$")

    class Config:
        populate_by_name = True

# Build the dependency that FastAPI knows how to extract from the query string.
def attempt_list_params(
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    test_id: str | None = Query(None),
    from_: date | None = Query(None, alias="from"),
    to: date | None = Query(None),
    status: Literal["in_progress", "submitted", "locked"] | None = Query(None),
    sort: str = Query("submitted_at:desc", pattern=r"^[a-z_]+:(asc|desc)$"),
) -> AttemptListParams:
    return AttemptListParams(
        page=page, size=size, test_id=test_id,
        from_=from_, to=to, status=status, sort=sort,
    )

AttemptListDep = Annotated[AttemptListParams, Depends(attempt_list_params)]
```

The boilerplate is real, but the payoff is worth it: the route handler signature stays clean, every constraint (ge, le, regex, enum) is validated *before* the handler runs, and FastAPI emits accurate OpenAPI docs for the whole parameter set. The cohort sees the regex on `sort` catch a typo (`?sort=submited_at:desc`) at the framework layer rather than producing a 500 from a bad SQL ORDER BY clause.

A note on `from`: it's a Python keyword, so the field is named `from_` and aliased to `from` for the URL. The `populate_by_name = True` config makes both work in Pydantic v2.

## The Route Handler

```python
from fastapi import APIRouter
from sqlalchemy import select, func, and_, asc, desc
from app.deps import TestMgmtDep        # AsyncSession dependency to test_management_db
from app.models import Session as Attempt   # ORM model duplicated from test-management
from app.schemas import AttemptOut, ListResponse

router = APIRouter(prefix="/reports", tags=["reports"])

SORT_COLUMNS = {
    "submitted_at": Attempt.submitted_at,
    "started_at":   Attempt.started_at,
    "score":        Attempt.score,
}

@router.get("/user/{user_id}/attempts", response_model=ListResponse[AttemptOut])
async def list_attempts(
    user_id: str,
    params: AttemptListDep,
    db: TestMgmtDep,
):
    filters = [Attempt.user_id == user_id]
    if params.test_id:
        filters.append(Attempt.test_id == params.test_id)
    if params.from_:
        filters.append(Attempt.submitted_at >= params.from_)
    if params.to:
        filters.append(Attempt.submitted_at <= params.to)
    if params.status:
        filters.append(Attempt.status == params.status)

    # COUNT for pagination meta (one query).
    total = await db.scalar(
        select(func.count(Attempt.id)).where(and_(*filters))
    )

    # Sort
    col_name, direction = params.sort.split(":")
    col = SORT_COLUMNS.get(col_name)
    if col is None:
        from fastapi import HTTPException
        raise HTTPException(400, f"Unsortable column: {col_name}")
    order = desc(col) if direction == "desc" else asc(col)

    # Page query
    offset = (params.page - 1) * params.size
    stmt = (
        select(Attempt)
        .where(and_(*filters))
        .order_by(order, Attempt.id.asc())   # tiebreaker for stable order
        .offset(offset)
        .limit(params.size)
    )
    rows = (await db.execute(stmt)).scalars().all()

    total_pages = (total + params.size - 1) // params.size
    return {
        "items": [AttemptOut.model_validate(r, from_attributes=True) for r in rows],
        "meta": {
            "page": params.page,
            "size": params.size,
            "total": total,
            "total_pages": total_pages,
            "has_next": params.page < total_pages,
            "has_prev": params.page > 1,
        },
    }
```

Several decisions in there worth naming explicitly.

### The Allowlist For Sortable Columns

`SORT_COLUMNS` is a hard-coded dict. The cohort's first instinct is often to use `getattr(Attempt, col_name)` — *don't*. That trusts the client's input as an attribute name on your ORM class, which lets a malicious or curious caller probe internal columns (`__table__`, `metadata`, etc.). Always allowlist.

### Tiebreaker On The Order

`order_by(order, Attempt.id.asc())` — the secondary sort on `id` is what makes pagination *stable*. If two rows have the same `submitted_at` (possible at second-resolution timestamps) and the database picks them in arbitrary order, a row can appear on page 2 *and* page 3, or be skipped entirely. The tiebreaker guarantees a total ordering.

### Two Queries, Not One

A `count(*)` query plus a `LIMIT ... OFFSET ...` query is two round trips. There are tricks to fuse them (`COUNT(*) OVER ()` window function, deferred-count strategies) but they trade simplicity for marginal performance. At PEP scale the two-query pattern is fine; flag the optimization for Phase 2.

### `from_attributes=True`

Pydantic v2's way of saying "build this model from an ORM row" (replacing v1's `orm_mode`). Without it, validation fails because the ORM row isn't a dict.

## Cursor Pagination, Mentioned

Offset/limit is what we ship today, and it's the right choice for PEP. Briefly mention the alternative so the cohort isn't surprised when they see it elsewhere:

**Cursor pagination** ships the last row's sort-key in the response and the client sends it back as `?after=<cursor>`. The next-page query is `WHERE submitted_at < cursor ORDER BY submitted_at DESC LIMIT N` — no `OFFSET`, so performance stays constant even at page 500. Trade-off: you lose `total`, `page`, and the ability to jump arbitrarily; the UI becomes infinite-scroll instead of numbered pages.

For PEP, offset/limit is correct: the result sets are small, the UI wants page numbers, and the SQL is simpler. For Phase 2's trainer-of-trainers dashboard (thousands of attempts), cursor pagination becomes attractive.

## Filtering: Defensive Composition

The handler builds `filters` as a list and `and_(*filters)`s them. The cohort should know why this matters:

- **Optional filters compose without ugliness.** Each `if params.test_id:` adds one clause; no branching of the whole `select` statement.
- **`and_(*filters)` with an empty list** generates `TRUE` (a no-op), which is what we want — no filters means no `WHERE`.
- **Don't string-interpolate filters into SQL.** Parameterize via the ORM. The cohort already knows this from W2/W3 but it bears repeating in the context of user-controlled query params.

For range filters, be deliberate about inclusive vs. exclusive:

- `from` is `>=` (start of day inclusive),
- `to` is `<=` (end of day inclusive — but be aware that `date <= 2026-05-19` against a `timestamptz` column means `<= 2026-05-19 00:00:00`, which excludes most of that day; cast to `< 2026-05-20` or use `to_timestamp = datetime.combine(to, time.max)`).

Date-vs-timestamp comparison is a classic footgun; the cohort should at least once produce a "where are my last hour of records?" bug and fix it.

## Sorting: The Trickier Cases

Sorting by `score` is tricky because `score` is nullable (in-progress attempts have no score yet). Postgres's default puts NULLs *last* on `ASC` and *first* on `DESC`. If the UI wants "scored attempts first, unscored at the end regardless of direction":

```python
from sqlalchemy import nulls_last
order = nulls_last(desc(col)) if direction == "desc" else nulls_last(asc(col))
```

`nulls_last` and `nulls_first` are Postgres features SQLAlchemy exposes. The cohort should know they exist; they'll need them on D18's trainer dashboard.

## Validating Cap Behavior

The `size=Field(..., le=100)` cap is what stops a caller from doing `?size=10000` and DOSing the service. Test it:

```python
def test_size_cap(client):
    r = client.get("/reports/user/u_42/attempts?size=1000")
    assert r.status_code == 422
    assert "size" in r.json()["detail"][0]["loc"]
```

The cap belongs in Pydantic (FastAPI returns 422 with a clear error) rather than the handler (which would silently cap and confuse the caller).

## Documenting The Contract

FastAPI's auto-generated OpenAPI shows the parameters with descriptions, defaults, and constraints — *if* you include the `description=`, `example=`, and validation parameters. A few minutes of documentation effort produces a Swagger UI the cohort can hand to the frontend pair without saying anything:

```python
test_id: str | None = Query(
    None,
    description="Filter to a single test ID (e.g., 'test_python_basics')",
    example="test_python_basics",
)
```

This is also where to flag deprecations later: `deprecated=True` on a Query parameter shows up in OpenAPI.

## Anti-Patterns

- **Trusting client-supplied column names for sort.** Always allowlist. `getattr(Model, user_input)` is a vulnerability waiting to happen.
- **`OFFSET 0 LIMIT 100000` because pagination is "annoying."** It works until it doesn't; at 10k rows the page is fine, at 100k the response is hundreds of MB. Cap `size` in Pydantic.
- **No stable tiebreaker on the order.** Rows duplicate or vanish across pages. Always add a unique-column tiebreaker (typically `id ASC`).
- **Filtering after fetching.** Pulling all rows into Python and filtering with a list comprehension is correct but wastes the database. Push filters into SQL.
- **Returning `total_pages` based on a stale `total`.** The count and the page query are two separate transactions; if data changed between them, totals can disagree. PEP accepts this; the alternative (`SERIALIZABLE` isolation or a single window-function query) is overkill here.
- **String-formatting query params into SQL.** A SQLi class on a reporting endpoint is no less a SQLi class than on a write endpoint. The ORM parameterizes for you; use it.
- **Returning `meta: null` when there are no items.** Always return the full envelope; empty `items` is fine, missing `meta` breaks frontend code.
- **Off-by-one in offset math.** `(page - 1) * size`, not `page * size`. Once burned, you remember forever.

## Key Takeaways

- A Pydantic dependency captures the entire query-parameter contract in one place, validates before the handler runs, and feeds OpenAPI docs automatically.
- Pagination is two queries (count + page) plus a stable tiebreaker; the response envelope includes `total`, `total_pages`, `has_next`, `has_prev` so the UI doesn't have to compute them.
- Sort columns must be allowlisted; never `getattr` a user-supplied name onto an ORM class.
- Filters compose with `and_(*filters)`; range filters need care around date vs. timestamp inclusivity.
- Cursor pagination exists and is the right call at scale; offset/limit is correct for PEP.

---
*Prerequisites: [02-fastapi-routing-and-dependency-injection-patterns.md](../day-11/02-fastapi-routing-and-dependency-injection-patterns.md), [06-pydantic-request-response-modeling.md](../day-11/06-pydantic-request-response-modeling.md), [02-sqlalchemy-queries-joins-aggregates-subqueries.md](02-sqlalchemy-queries-joins-aggregates-subqueries.md).*
