# Schema Validation Patterns: Required Fields, Conditional Rules, Custom Validators

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

Topic 1 modeled the *shape* of a question. This topic enforces the *rules* that make a question well-formed: a single-select must have exactly one correct option; a multi-select must have at least two; option IDs must be unique; the stem must not be whitespace-only. These are domain invariants, not type signatures, and Pydantic v2 has a precise vocabulary for each kind.

Today's deliverable hangs on getting these validators right — if they pass, the trainer's PR is approvable; if they leak (or worse, reject valid input), the Day 9 frontend has a bad day. Topic 6 tests every rule on this page; Topic 7's AI-assisted generation will likely *propose* these validators and you will *defend* them.

## The three tiers of validation

Pydantic v2 separates concerns into three layers. Use the lightest one that fits.

### 1. Field-level — `Field()` constraints

For "this string can't be empty", "this list needs at least 2 items", "this int must be positive":

```python
from pydantic import BaseModel, Field

class Option(BaseModel):
    id: str = Field(min_length=1, max_length=8, pattern=r"^[a-z0-9_-]+$")
    text: str = Field(min_length=1, max_length=500)
    is_correct: bool

class QuestionBase(BaseModel):
    stem: str = Field(min_length=1, max_length=2000)
    options: list[Option] = Field(min_length=2, max_length=8)
    tags: list[str] = Field(default_factory=list, max_length=10)
```

These are declarative, self-documenting, and produce great OpenAPI. Reach for `Field()` first.

### 2. Field-level — `@field_validator` for transforms and single-field rules

When you need to strip whitespace, normalize casing, or apply a rule that's just "this one field":

```python
from pydantic import field_validator

class QuestionBase(BaseModel):
    stem: str = Field(min_length=1, max_length=2000)

    @field_validator("stem")
    @classmethod
    def stem_not_whitespace(cls, v: str) -> str:
        stripped = v.strip()
        if not stripped:
            raise ValueError("stem must not be whitespace-only")
        return stripped  # the cleaned value is what gets stored
```

Use `mode="before"` to operate on raw input before type coercion, `mode="after"` (default) once the value is typed.

### 3. Model-level — `@model_validator` for cross-field rules

The interesting rules in a question are *cross-field*: "the count of correct options depends on `type`". That's a model validator:

```python
from pydantic import model_validator
from typing import Self

class SingleSelectQuestion(_QuestionBase):
    type: Literal["single_select"]

    @model_validator(mode="after")
    def exactly_one_correct(self) -> Self:
        correct = [o for o in self.options if o.is_correct]
        if len(correct) != 1:
            raise ValueError(
                f"single_select requires exactly 1 correct option, got {len(correct)}"
            )
        return self

class MultiSelectQuestion(_QuestionBase):
    type: Literal["multi_select"]

    @model_validator(mode="after")
    def at_least_two_correct(self) -> Self:
        correct = [o for o in self.options if o.is_correct]
        if len(correct) < 2:
            raise ValueError(
                f"multi_select requires at least 2 correct options, got {len(correct)}"
            )
        return self
```

`mode="after"` runs once all field-level validators have passed and `self` is a fully typed model — the safe place to express invariants.

## A shared rule: unique option IDs

This belongs on the base because both variants need it:

```python
class _QuestionBase(BaseModel):
    # ... fields ...

    @model_validator(mode="after")
    def option_ids_unique(self) -> Self:
        ids = [o.id for o in self.options]
        if len(ids) != len(set(ids)):
            duplicates = {i for i in ids if ids.count(i) > 1}
            raise ValueError(f"option ids must be unique; duplicates: {sorted(duplicates)}")
        return self
```

Putting it on the base means you only write it once and both variants inherit it. The order of validators is base-first, subclass-second — exactly what you want.

## What FastAPI does with these errors

Every `raise ValueError(...)` inside a validator becomes a 422 response with a structured `detail`:

```json
{
  "detail": [
    {
      "type": "value_error",
      "loc": ["body", "multi_select"],
      "msg": "Value error, multi_select requires at least 2 correct options, got 1",
      "input": { "...": "..." }
    }
  ]
}
```

The `loc` array tells the Day 9 frontend exactly where to highlight in its form. Resist the urge to raise `HTTPException` from inside a validator — Pydantic owns this error path and translates it correctly. The route function only handles exceptions Pydantic can't (Topic 4).

## Example / Worked Scenario

Trainer submits a "single-select" with two correct answers — a common authoring mistake:

```json
{
  "type": "single_select",
  "stem": "What HTTP status is returned on successful creation?",
  "options": [
    {"id": "a", "text": "200", "is_correct": false},
    {"id": "b", "text": "201", "is_correct": true},
    {"id": "c", "text": "204", "is_correct": true},
    {"id": "d", "text": "302", "is_correct": false}
  ]
}
```

Validation order:
1. **Field validators** pass — stem non-empty, options has 4 items, IDs are valid slugs.
2. **Base model validator** (`option_ids_unique`) passes — `a/b/c/d` are unique.
3. **Variant model validator** (`exactly_one_correct`) **fails** — 2 correct, not 1.
4. FastAPI returns 422 with `loc: ["body", "single_select"]` and the clear "got 2".

The frontend (Day 9) parses `loc`, finds the message, and shows it next to the options table. No business logic ran in the route — the model layer rejected the bad shape *before* the database was touched.

## Common Pitfalls

- **Doing validation in the route handler instead of the model.** If `create_question` contains `if len([o for o in payload.options if o.is_correct]) != 1: raise HTTPException(...)`, you've duplicated the rule, dodged the structured `detail`, and split the contract across two files. Push it into a model validator.
- **`@field_validator` for cross-field rules.** Field validators run before the model is assembled; they can't see other fields. Use `@model_validator(mode="after")` instead.
- **Raising `HTTPException` inside validators.** It bypasses Pydantic's `ValidationError` aggregation — instead of one 422 listing every issue, the request fails on the first one with a 4xx that isn't the right shape. Always `raise ValueError(...)` from validators.
- **Mutating `self` in `mode="after"`.** Models are frozen by convention; reassign with `self.model_copy(update=...)` or do the transformation in `mode="before"` on the raw dict. Don't `self.field = ...` and call it a day.
- **Re-running validators on database reads.** When you load a doc from Mongo via `Question.model_validate(doc)`, the model validators run again. If they're expensive (network calls, etc.), that's a problem. Keep validators pure and fast; do expensive checks at the route layer.

## Key Takeaways

- Three tiers: `Field()` constraints, `@field_validator` for single-field transforms, `@model_validator(mode="after")` for cross-field invariants. Use the lightest tier that fits.
- The interesting rules for polymorphic questions are *cross-field* — exactly-one and at-least-two — and belong on the variant models, not the base.
- Shared rules (unique option IDs) live on the base; subclass validators run after, in order.
- Always `raise ValueError(...)` from validators; FastAPI translates it to a structured 422 with `loc` the frontend can use.
- Validators run for every `model_validate(...)` call — including reads from Mongo (Topic 3). Keep them pure.
- Pair each validator with a parametrized pytest case in Topic 6 — the validator and its test are a single unit of work.

---
*Prerequisites: `day-8-pydantic-discriminated-unions-for-polymorphic-schemas`.*
