# Pydantic Discriminated Unions for Polymorphic Schemas

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

Today's deliverable adds question authoring to `question-management-service`: the API has to accept *single-select* questions (exactly one correct answer) and *multi-select* questions (two or more correct answers) through the same endpoint. A naive approach branches on an `if` after parsing; the Pydantic-idiomatic approach is a **discriminated union**, where a `type` field tells the parser which model shape to validate against and you get static-style polymorphism for free.

Once you have a discriminator, the FastAPI request body validates itself, OpenAPI documents both shapes correctly, and downstream code can `match` on `.type` knowing the rest of the fields are already the right shape.

## The shape of the problem

A question is "the same thing" at the API surface — it has a stem, options, and metadata — but its *correctness rules* differ by type. You want one endpoint, one request body, and exactly the right validators applied for each variant. Without a discriminator, Pydantic will try every union member in order and pick the first that parses, which is fragile (loose models eat strict ones) and produces unhelpful error messages.

A discriminator is a literal field whose value names the variant. Pydantic uses it as a router: read `type`, dispatch to the matching model, validate, done.

## Pydantic v2 syntax

```python
# services/question-management-service/app/models/question.py
from typing import Annotated, Literal
from pydantic import BaseModel, Field

class Option(BaseModel):
    id: str = Field(min_length=1)
    text: str = Field(min_length=1, max_length=500)
    is_correct: bool

class _QuestionBase(BaseModel):
    stem: str = Field(min_length=1, max_length=2000)
    options: list[Option] = Field(min_length=2, max_length=8)
    difficulty: Literal["easy", "medium", "hard"] = "medium"
    image_key: str | None = None  # MinIO object key (Topic 5)

class SingleSelectQuestion(_QuestionBase):
    type: Literal["single_select"]

class MultiSelectQuestion(_QuestionBase):
    type: Literal["multi_select"]

Question = Annotated[
    SingleSelectQuestion | MultiSelectQuestion,
    Field(discriminator="type"),
]
```

That `Annotated[... , Field(discriminator="type")]` is the v2 idiom. The literal `type: Literal["single_select"]` on each variant is what the discriminator reads.

## Using it in FastAPI

```python
# services/question-management-service/app/routes/questions.py
from fastapi import APIRouter, status
from app.models.question import Question

router = APIRouter(prefix="/questions", tags=["questions"])

@router.post("", status_code=status.HTTP_201_CREATED, response_model=Question)
async def create_question(payload: Question) -> Question:
    # `payload` is already SingleSelectQuestion or MultiSelectQuestion — no isinstance dance
    return await repository.insert(payload)
```

FastAPI sees the `Annotated` union, generates an OpenAPI `oneOf` with the discriminator mapping, and Swagger UI renders a dropdown for the `type` field. Clients (including the Day 9 frontend) get a clean contract.

## What good error messages look like

Without a discriminator, posting `{"type": "single_select", "stem": "", ...}` might fail with "no variant matched" — useless. With a discriminator:

```json
{
  "detail": [
    {
      "type": "string_too_short",
      "loc": ["body", "single_select", "stem"],
      "msg": "String should have at least 1 character"
    }
  ]
}
```

Pydantic knows exactly which variant to blame because `type` told it.

## Branching on the variant in code

When you do need to act on the variant — for example, applying correctness rules (Topic 2) or producing a public-facing "quiz mode" projection (Day 10) — `match` is the clean idiom:

```python
def correct_option_ids(q: Question) -> list[str]:
    match q:
        case SingleSelectQuestion():
            return [o.id for o in q.options if o.is_correct][:1]
        case MultiSelectQuestion():
            return [o.id for o in q.options if o.is_correct]
```

The static checker (and the human reader) both know what fields exist in each arm.

## Example / Worked Scenario

A trainer posts a multi-select question to `POST /questions`:

```json
{
  "type": "multi_select",
  "stem": "Which of the following are HTTP idempotent methods?",
  "difficulty": "medium",
  "options": [
    {"id": "a", "text": "GET",    "is_correct": true},
    {"id": "b", "text": "POST",   "is_correct": false},
    {"id": "c", "text": "PUT",    "is_correct": true},
    {"id": "d", "text": "DELETE", "is_correct": true}
  ]
}
```

The flow:

1. FastAPI receives the body, hands it to the `Question` annotated union.
2. Pydantic reads `type == "multi_select"`, dispatches to `MultiSelectQuestion`.
3. Base validators run: stem length, options count, option text length.
4. Topic 2's `@model_validator` runs the multi-select rule (>= 2 correct).
5. The route receives a fully-typed `MultiSelectQuestion` and persists it.

A `type: "true_false"` body — not yet supported — gets a 422 before any code in the route runs, with `loc: ["body", "type"]` and an explicit "input tag 'true_false' found using 'type' does not match any of the expected tags".

## Common Pitfalls

- **Forgetting `Literal` on the discriminator field.** `type: str` makes the union ambiguous; Pydantic v2 will refuse to build the discriminator and raise at import time. Always `Literal["..."]`.
- **Using `Union` without the `Field(discriminator=...)` wrapper.** Works, but you lose the dispatch — every payload is tried against every variant in order, error messages degrade, and OpenAPI loses the `discriminator` mapping.
- **Reusing the same `type` value across variants.** The discriminator must be unique per variant or import fails. If you need a variant that "looks like" another, give it its own tag.
- **Adding a third variant later and forgetting to update the `Question` alias.** Build a `_VARIANTS` list and derive the union from it, or accept that you will touch one line — but make sure the route, the repository (Topic 3), and the tests all see the new variant. The discriminator is centralised; the consequences aren't.

## Key Takeaways

- A discriminated union is the right tool when one endpoint accepts several related-but-distinct shapes — like single-select vs multi-select questions.
- Pydantic v2's `Annotated[A | B, Field(discriminator="type")]` plus `Literal[...]` discriminator fields is the modern idiom; FastAPI consumes it directly and generates clean OpenAPI.
- Discriminators give you better error messages because Pydantic knows which variant to validate against before running per-field checks.
- Use `match` (or `isinstance`) on the parsed object to branch on variant in business logic — the static type narrows correctly.
- The discriminator is *structural*, not semantic — Topic 2's validators are where the multi-select-needs-2+-correct rule actually lives.

---
*Prerequisites: Day 1 (FastAPI services in the substrate), Day 2 (MongoDB containerization — where these documents will live).*
