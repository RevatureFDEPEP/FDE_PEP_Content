# FastAPI Request / Response Patterns

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

Topic 1 built the request model; Topic 2 enforced its rules; Topic 3 persisted it. This topic is the *glue*: how FastAPI wires the validated model into an idiomatic CRUD surface that the Day 9 frontend can consume without surprises. The goal is a route file that looks boring — because boring means correct status codes, predictable error shapes, response models that don't leak internals, and dependency-injected resources that tests can swap.

By the end you should be able to read `routes/questions.py` and see exactly which responses each endpoint can produce and why.

## The five things every endpoint should get right

1. **Request body** — typed via the Pydantic models from Topics 1+2.
2. **Response model** — declared explicitly with `response_model=`, so the response shape is documented and filtered.
3. **Status code** — `201` on create, `204` on delete, `200` on read/update; never silently `200` for everything.
4. **Errors** — `HTTPException` for app-level failures (not found, conflict); Pydantic owns 422 (Topic 2).
5. **Dependencies** — repository injected via `Depends(...)`, lifespan owns its construction.

## A clean CRUD surface

```python
# services/question-management-service/app/routes/questions.py
from typing import Annotated
from fastapi import APIRouter, Depends, HTTPException, Query, status
from app.models.question import Question
from app.db.questions_repo import QuestionsRepository
from app.deps import get_questions_repo

router = APIRouter(prefix="/questions", tags=["questions"])
RepoDep = Annotated[QuestionsRepository, Depends(get_questions_repo)]

@router.post(
    "",
    status_code=status.HTTP_201_CREATED,
    response_model=Question,
    responses={422: {"description": "Validation failed"}},
)
async def create_question(payload: Question, repo: RepoDep) -> Question:
    return await repo.insert(payload)

@router.get("/{qid}", response_model=Question)
async def get_question(qid: str, repo: RepoDep) -> Question:
    q = await repo.get(qid)
    if q is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail=f"question {qid} not found")
    return q

@router.get("", response_model=list[Question])
async def list_questions(
    repo: RepoDep,
    tag: Annotated[str | None, Query(min_length=1, max_length=40)] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 50,
) -> list[Question]:
    if tag:
        return await repo.list_by_tag(tag, limit=limit)
    return await repo.list_recent(limit=limit)

@router.put("/{qid}", response_model=Question)
async def replace_question(qid: str, payload: Question, repo: RepoDep) -> Question:
    updated = await repo.replace(qid, payload)
    if updated is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail=f"question {qid} not found")
    return updated

@router.delete("/{qid}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_question(qid: str, repo: RepoDep) -> None:
    deleted = await repo.delete(qid)
    if not deleted:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail=f"question {qid} not found")
```

A few specifics worth pointing at:

- **`Annotated[QuestionsRepository, Depends(get_questions_repo)]`** is the v2 idiom — alias it once (`RepoDep`) and the route signatures stay readable.
- **`response_model=Question`** uses the discriminated union: FastAPI documents `oneOf` and *filters* the response (any extra fields on the in-memory object are stripped before sending).
- **Query parameter validation** (`Query(min_length=..., ge=..., le=...)`) is declarative — invalid `limit=999` returns 422 before the route runs.
- **`204 No Content`** has no body — FastAPI knows because the return annotation is `None`.

## Dependency injection done properly

Resources (DB connections, repositories, settings) live in `app/deps.py` and use the lifespan:

```python
# services/question-management-service/app/deps.py
from fastapi import FastAPI, Request
from motor.motor_asyncio import AsyncIOMotorClient
from app.config import settings
from app.db.questions_repo import QuestionsRepository
from app.db.indexes import ensure_indexes

async def lifespan(app: FastAPI):
    client = AsyncIOMotorClient(settings.MONGO_URL)
    db = client.get_database(settings.MONGO_DB)
    await ensure_indexes(db.questions)
    app.state.questions_repo = QuestionsRepository(db.questions)
    yield
    client.close()

def get_questions_repo(request: Request) -> QuestionsRepository:
    return request.app.state.questions_repo
```

```python
# services/question-management-service/app/main.py
from fastapi import FastAPI
from app.deps import lifespan
from app.routes.questions import router as questions_router

app = FastAPI(title="question-management-service", lifespan=lifespan)
app.include_router(questions_router)
```

Why this matters for tests (Topic 6 + Day 9 integration):

- The route depends on `get_questions_repo`, not on a global.
- A test can `app.dependency_overrides[get_questions_repo] = lambda: FakeRepo()` and the routes work unchanged.
- Production wiring (real Mongo) and test wiring (fake) share *zero* code paths and *one* contract.

## Error handling that doesn't leak

Two rules:

1. **`HTTPException`** for things the route is responsible for: 404 not found, 409 conflict, 403 forbidden. Always pass `detail=` — never let it default.
2. **Don't catch `ValidationError`** in routes. FastAPI's exception handler already turns it into a 422 with the structured `detail` from Topic 2. If you `try/except` it, you'll either re-raise (pointless) or produce a worse response (harmful).

For domain exceptions that bubble out of the repository (e.g., "duplicate option ID" from a custom Mongo unique-index violation), wrap them in an exception handler:

```python
# services/question-management-service/app/main.py
from fastapi import Request
from fastapi.responses import JSONResponse
from pymongo.errors import DuplicateKeyError

@app.exception_handler(DuplicateKeyError)
async def _dup_key(_: Request, exc: DuplicateKeyError) -> JSONResponse:
    return JSONResponse(status_code=409, content={"detail": "duplicate key"})
```

Keep the route handlers free of try/except for known errors; the global handler is the right place.

## Response shape: `response_model` is doing real work

When you declare `response_model=Question`, FastAPI:

1. Runs the discriminated union through Pydantic's serializer on the way out.
2. Strips any fields that aren't in the model (e.g., internal `_id`, `schema_version` from the Mongo document).
3. Uses the model's `model_config` for things like alias generation and JSON encoding.

This is the only thing standing between "internal Mongo document" and "frontend JSON". Without `response_model=`, FastAPI returns whatever Python object you give it through a generic encoder — including private fields. Always declare it.

A subtle gotcha: `response_model=Question` and a return type annotation of `Question` are *not* the same thing. The annotation guides static checking; `response_model` controls serialization. Set both.

## Example / Worked Scenario

The trainer (Day 9 frontend) creates a question, fetches it, then deletes it:

1. `POST /questions` with a valid multi-select body. Route receives a `MultiSelectQuestion`; `repo.insert` returns the persisted document; FastAPI serializes via `response_model=Question`, returns `201 Created` with the JSON.
2. `GET /questions/01J9ZQ8T7K3R...` — the route asks the repo, gets a `Question` or `None`; if `None`, raises `HTTPException(404, "question ... not found")`. The frontend's 404 path is exercised.
3. `GET /questions?tag=http&limit=10` — query validators clamp `limit` to `[1, 100]`; `tag=http` flows through to the index from Topic 3.
4. `DELETE /questions/01J9...` — repo returns `True`; route returns `None`; FastAPI sends `204 No Content` with empty body.
5. Frontend submits a broken payload (`type: single_select`, 0 correct). FastAPI's Pydantic layer rejects with 422, route never runs, `detail` has `loc: ["body", "single_select"]` and the validator message from Topic 2.

Every endpoint has one observable status code per outcome — nothing surprising.

## Common Pitfalls

- **Returning the Mongo document dict directly.** It contains `_id`, `schema_version`, and whatever else you added; without `response_model` filtering, that all leaks. Always go via the Pydantic model.
- **Using 200 for everything.** A create that returns 200 confuses HTTP clients and intermediaries. Use 201 for create, 204 for delete-no-content, 200 for read/update. Status codes are part of the API.
- **Global DB clients constructed at import time.** Import-time side effects (network connections) break tests and surprise readers. Use `lifespan` to construct, `Depends` to inject.
- **Mixing validation logic into routes.** If a route has `if len(payload.options) < 2: raise HTTPException(400, ...)`, that logic belongs in a Pydantic validator (Topic 2). Routes orchestrate; models validate.
- **Catching `Exception` and returning 500 with a string.** Lets a real bug masquerade as a controlled error and hides the stack trace from the logs. Let unexpected exceptions bubble — FastAPI logs them and returns 500 with no body, which is the right answer.

## Key Takeaways

- A clean route file is mostly: declared status codes, declared `response_model`, injected repo, and `HTTPException` for app-level failures.
- `Annotated[..., Depends(...)]` aliased as a `TypeAlias` keeps signatures readable.
- `response_model=` is the filter between Mongo documents and frontend JSON — never skip it on a public endpoint.
- Don't catch `ValidationError`; Pydantic and FastAPI already do the right thing. Do catch *domain* exceptions in global handlers.
- Use `lifespan` to construct DB clients and repositories once; routes ask for them via `Depends`. Tests swap them via `app.dependency_overrides`.
- 201 / 200 / 204 / 404 / 409 / 422 — pick the right one for each outcome. Status codes are documentation.

---
*Prerequisites: [01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md](01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md), [02-schema-validation-patterns.md](02-schema-validation-patterns.md), [03-mongodb-document-modeling-for-variable-shape-data.md](03-mongodb-document-modeling-for-variable-shape-data.md).*
