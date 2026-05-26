# Modeling Polymorphic Data — Discriminated Unions, Tagged Enums, Single-Shape with Conditional Rules (a Brownfield Modeling Decision)

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

Today's deliverable adds question authoring to `question-management-service`. The API has to accept *single-select* (exactly one correct answer) and *multi-select* (two or more) through one endpoint, and the substrate may already model this — perhaps well, perhaps not. There is no single "right" Pydantic answer here. Three idiomatic patterns coexist, each defensible under different conditions, and the inherited code may already have committed to one of them.

This is the first explicit *brownfield modeling decision* of the course. The pedagogical work today is not "adopt discriminated unions because they are pretty" — it is to look at what's in the substrate, weigh the patterns honestly, and **make and defend a decision** using technical-debt assessment, cost-benefit, and an explicit ROI horizon. The deliverable requires two contrasting shapes to validate end-to-end; the modeling pattern is your call.

## Three patterns, in parallel

All three patterns are idiomatic Pydantic v2 and all three appear in real production code. None is universally best.

### Pattern A — Discriminated union (separate models, dispatched on a `type` literal)

A `type` field with a `Literal[...]` value selects which model to validate against. Each variant is its own class; Pydantic uses the discriminator to route the payload to the right one.

```python
# services/question-management-service/app/models/question.py
from typing import Annotated, Literal, Self
from pydantic import BaseModel, Field, model_validator

class Option(BaseModel):
    id: str = Field(min_length=1)
    text: str = Field(min_length=1, max_length=500)
    is_correct: bool

class _QuestionBase(BaseModel):
    stem: str = Field(min_length=1, max_length=2000)
    options: list[Option] = Field(min_length=2, max_length=8)
    difficulty: Literal["easy", "medium", "hard"] = "medium"
    image_key: str | None = None

class SingleSelectQuestion(_QuestionBase):
    type: Literal["single_select"]

    @model_validator(mode="after")
    def exactly_one_correct(self) -> Self:
        correct = sum(1 for o in self.options if o.is_correct)
        if correct != 1:
            raise ValueError(f"single_select requires exactly 1 correct option, got {correct}")
        return self

class MultiSelectQuestion(_QuestionBase):
    type: Literal["multi_select"]

    @model_validator(mode="after")
    def at_least_two_correct(self) -> Self:
        correct = sum(1 for o in self.options if o.is_correct)
        if correct < 2:
            raise ValueError(f"multi_select requires at least 2 correct options, got {correct}")
        return self

Question = Annotated[
    SingleSelectQuestion | MultiSelectQuestion,
    Field(discriminator="type"),
]
```

A FastAPI endpoint annotated with `payload: Question` gets a fully-typed `SingleSelectQuestion` or `MultiSelectQuestion`, with `match q:` working cleanly in business logic.

**Trade-offs:**
- *Discoverability:* each variant is a class — easy to navigate, easy for IDE search.
- *OpenAPI shape:* clean `oneOf` with a `discriminator.propertyName` and a per-tag mapping. Frontend codegen and zod-mirror work (Day 9) get a precise schema per variant.
- *Validation ergonomics:* per-variant `@model_validator` reads naturally; no "if type ==" branching inside one big validator.
- *Refactor cost when adding a new type:* one new class, one literal, one line on the `Annotated` union. Fan-out is mechanical.
- *Pitfall surface:* loose-then-strict variant ordering bugs are gone (the discriminator dispatches by tag, not by trial); but every variant must use `Literal` on its discriminator field, and tags must be unique.

### Pattern B — Tagged enum on a single model (one class, conditional validators)

One class, a `type: QuestionType` enum field, and `@model_validator(mode="after")` methods that inspect `self.type` and branch.

```python
from enum import StrEnum
from typing import Self
from pydantic import BaseModel, Field, model_validator

class QuestionType(StrEnum):
    SINGLE_SELECT = "single_select"
    MULTI_SELECT = "multi_select"

class Question(BaseModel):
    type: QuestionType
    stem: str = Field(min_length=1, max_length=2000)
    options: list[Option] = Field(min_length=2, max_length=8)
    difficulty: Literal["easy", "medium", "hard"] = "medium"
    image_key: str | None = None

    @model_validator(mode="after")
    def correctness_rules_per_type(self) -> Self:
        correct = sum(1 for o in self.options if o.is_correct)
        if self.type is QuestionType.SINGLE_SELECT and correct != 1:
            raise ValueError(f"single_select requires exactly 1 correct option, got {correct}")
        if self.type is QuestionType.MULTI_SELECT and correct < 2:
            raise ValueError(f"multi_select requires at least 2 correct options, got {correct}")
        return self
```

**Trade-offs:**
- *Discoverability:* one class. Easier to find "the question model"; harder to find "what does a multi-select require?" — the answer is inside an `if`.
- *OpenAPI shape:* one schema with an enum-valued `type` field. The frontend cannot tell from the schema alone that the validator behaves differently per tag — the contract is *behavioural*, not *structural*.
- *Validation ergonomics:* the conditional chain inside one validator is fine for two types and becomes a smell at four or five. The shape that pushes you out of this pattern is the same shape that signals "this is debt now."
- *Refactor cost when adding a new type:* a new enum member and a new branch in every conditional validator. Easy to forget one.
- *Pitfall surface:* `if self.type == ...` chains drift out of sync with the enum. Static checkers can't catch a missed branch.

### Pattern C — Per-type subclasses, unioned at the API boundary

A shared base class with `type` carried as a class attribute, each subclass declares its rules, and the union is resolved at the FastAPI request body annotation. No discriminator field on the wire — the variant is decided structurally (presence of fields) or by an explicit `type` field that mirrors A but without `Annotated[..., Field(discriminator=...)]`.

```python
from typing import ClassVar, Self, Union
from pydantic import BaseModel, Field, model_validator
from fastapi import APIRouter

class QuestionBase(BaseModel):
    type: str  # subclasses override to a Literal
    stem: str = Field(min_length=1, max_length=2000)
    options: list[Option] = Field(min_length=2, max_length=8)

class SingleSelectQuestion(QuestionBase):
    type: Literal["single_select"] = "single_select"

    @model_validator(mode="after")
    def exactly_one_correct(self) -> Self: ...

class MultiSelectQuestion(QuestionBase):
    type: Literal["multi_select"] = "multi_select"

    @model_validator(mode="after")
    def at_least_two_correct(self) -> Self: ...

router = APIRouter()

@router.post("/questions")
async def create_question(
    payload: Union[SingleSelectQuestion, MultiSelectQuestion],
) -> Union[SingleSelectQuestion, MultiSelectQuestion]:
    return await repository.insert(payload)
```

**Trade-offs:**
- *Discoverability:* subclasses are findable, but the routing is implicit at the endpoint — readers must look at the route signature to learn the surface.
- *OpenAPI shape:* a plain `oneOf` *without* a discriminator mapping. Frontend codegen has to use shape-based discrimination (trial parsing) — workable but lossier than Pattern A.
- *Validation ergonomics:* same per-variant cleanliness as Pattern A.
- *Refactor cost when adding a new type:* a new subclass, **and** every endpoint signature that unions the variants must be updated. If three endpoints declare the union inline, three places need editing.
- *Pitfall surface:* without `Field(discriminator=...)`, Pydantic falls back to "try each variant, take the first that parses." Loose variants can swallow strict ones; payloads that should be `MultiSelectQuestion` may parse as `SingleSelectQuestion` if you're not careful with shared fields.

### Quick comparison

| Concern | A: Discriminated union | B: Tagged enum | C: Subclass union |
|---|---|---|---|
| OpenAPI quality (for zod mirror) | best | weakest | middle |
| Adding a new type | one place | many branches | many endpoints |
| Validator readability | per-class, clean | conditional chain | per-class, clean |
| Refactor cost from current | depends on substrate | depends on substrate | depends on substrate |
| Risk of silent shape collision | none (tag-routed) | n/a (one shape) | real (trial parsing) |

## The decision framework

Three patterns, all legitimate. The job is to choose, defend, and move on.

### Technical debt assessment — is the inherited model debt or fit-for-purpose?

Walk the inherited code before you judge it. Distinguish "this code is ugly to me" from "this code is debt."

**Signals that the inherited model is debt (favours refactor):**
- A `type` field exists but isn't load-bearing — same `if/elif` chains repeat across services, validators, serializers, and the frontend.
- New variants get added by copy-pasting an existing one and editing one or two lines, with branches forgotten in 1–2 places per change.
- Comments like `# TODO: add real validation when we add true_false`.
- Test coverage is per-type but split awkwardly because there's only one class to mock around.
- A previous reviewer left a comment like "we should split this into separate models" that nobody actioned.

**Signals that the inherited model is fit-for-purpose (favours extend):**
- The variants share *most* fields and differ only in validation rules; the conditional logic is short and centralised.
- The substrate has consistent patterns elsewhere using the same shape — refactoring this one in isolation creates inconsistency.
- The OpenAPI consumers (the existing frontend) work fine with the current shape.
- Tests are clean and the validators read well at the current variant count.

**Smell vs design.** A two-branch conditional in one place is style. A two-branch conditional repeated in eight places, with three of them subtly wrong, is debt. Be honest about which one you're looking at.

### Cost-benefit of refactor vs extend

If the inherited substrate already implements one of the three patterns and you're tempted to refactor to another, name the costs concretely before the benefits.

**Refactor costs (concrete, this PEP):**
- *API churn:* every endpoint that types a question payload changes. Routes, response models, repository signatures.
- *Test rewrites:* parametrized tests against the old shape (Topic 6) all touch the model module; some need restructuring.
- *Day 9 frontend zod-mirror churn:* the zod schemas (Day 9, Topic 4) mirror the backend. If the backend's OpenAPI shape changes — and refactoring from Pattern B to A changes it materially — the frontend pays the cost too, and that cost lands on you tomorrow.
- *Migration of stored documents (if any):* if Mongo already has documents in one shape, a refactor that changes how they deserialize may require a back-fill.
- *Review friction:* peers reviewing the PR have to track both "the shape changed" and "the feature was added" simultaneously. Hard to review well.

**Refactor benefits:**
- Cleaner OpenAPI (Pattern A specifically).
- Lower per-variant cognitive load when adding the third or fourth type.
- Static checking catches missed branches (the conditional validator chain in Pattern B can't).

**Extend costs:**
- The smell, if it is one, gets larger.
- If the inherited pattern is C and you add a fourth variant, several endpoint signatures need updating, not just one.

**Extend benefits:**
- Day 9 frontend contract is stable; the zod-mirror doesn't change shape.
- The PR reviews as a feature add, not a refactor — easier to land, easier to revert.
- Time is not spent on cleanliness when functionality is the deliverable.

### ROI horizon — name your horizon before deciding

The same trade-off resolves differently under different horizons. Say which horizon you are reasoning over before you reason.

- **PEP 4-week horizon.** A refactor that costs two days and pays back over six months is a net loss inside PEP. A refactor that costs two hours and removes a friction the rest of the week will trip over is a clear win. The default disposition over four weeks is **extend unless the refactor is small or the inherited shape actively blocks today's work**.
- **Multi-year ownership horizon.** Over years, the per-variant cognitive load and the OpenAPI quality dominate. The default flips: **refactor toward Pattern A or C unless something specific argues against it.**
- **Mixed horizon (this PEP, with the cohort's PR history feeding the 10-week intensive).** Reviewers in the next phase will inherit your decision. A clean decision defended in PR — even "I extended Pattern B because the refactor cost outweighed the benefit over four weeks" — is more valuable than a perfect pattern adopted without defence.

The point is not that one horizon is correct. The point is to *name the horizon explicitly* and tie the decision to it.

## Example / Worked Scenario

Suppose the inherited `question-management-service` already uses Pattern B — one `Question` class, a `type: QuestionType` enum, one big `@model_validator` that branches. Today's deliverable requires two contrasting shapes to validate end-to-end (single-select and multi-select). Walk the decision:

1. **Static read.** The inherited model is one file, 80 lines. The conditional validator branches on two enum values today; there are no other variants. The endpoint signatures use `Question` directly. The Day 9 frontend's zod schema (which you can preview in the substrate) mirrors the shape: one zod object with conditional refinements.

2. **Debt assessment.** The conditional chain is short, the validator reads fine, the OpenAPI is one schema with an enum field. The frontend already mirrors it. Score: **style, not debt.**

3. **Cost-benefit, PEP horizon.**
   - *Refactor to A:* ~3 hours of backend changes, ~2 hours of frontend zod mirror changes tomorrow, plus review. Benefit: cleaner OpenAPI, easier to add the third variant later (which PEP probably won't need).
   - *Extend B:* ~30 minutes — the validator already handles both branches. Benefit: nothing changes on the wire; Day 9 is unaffected.
4. **ROI horizon.** PEP 4-week. The third variant is not on the deliverable list. The OpenAPI quality improvement does not unblock anything before Friday.

5. **Decision.** Extend Pattern B. Defend it in the PR description: *"The inherited model is Pattern B (tagged enum, conditional validators). I considered refactoring to a discriminated union for cleaner OpenAPI but the cost-benefit over the PEP timeframe didn't justify it: the validator chain is short at two variants, the Day 9 zod mirror would need to change shape, and there's no third variant on the deliverable list. I'd revisit this if a third type were added, or if the cohort were on a multi-year ownership horizon."*

The same scenario with the inherited code at Pattern B but already showing branches in three other places (the serializer, a quiz-build filter, a frontend client) flips the answer — that's debt, the cost-benefit reverses, and the refactor to A is the right call. The framework didn't change; the inputs did.

## Common Pitfalls

- **Silently choosing "refactor" because Pattern A is prettier.** Pattern A is genuinely the cleanest pattern; that does not make refactoring to it free, and it does not exempt you from doing the cost-benefit. If you find yourself reaching for the refactor without writing down the costs, stop and write them down.
- **Conflating "this code is ugly" with "this code is debt."** Style preferences and debt are different categories. Debt has measurable cost (repeated bugs, missed branches, slow onboarding); ugliness is taste. Refactoring debt is engineering; refactoring ugliness on someone else's timeline is rework.
- **Ignoring the Day 9 frontend zod-mirror cost.** The OpenAPI shape your backend produces today is consumed verbatim by the zod schema tomorrow. A backend refactor that changes shape doubles into a frontend refactor the next day, and that cost lands on you, not on a future maintainer. Account for it before deciding.
- **Choosing a pattern without naming the horizon.** "Refactor to A" or "extend B" without "over what timeframe?" is incomplete. The same evidence supports opposite conclusions depending on the horizon — so the horizon belongs in the PR description, not assumed.
- **Treating the curriculum's example as a prescription.** Patterns A, B, and C are presented as equal options. The deliverable does not specify which to use; the PR description specifies *why you chose what you chose*.

## Key Takeaways

- Three patterns — discriminated union, tagged enum on a single model, per-type subclasses unioned at the boundary — are all legitimate Pydantic v2 idioms. None is universally best.
- The choice is a **decision the candidate makes and defends**, not a default the curriculum assigns. The PR description records the chosen pattern and the reasoning.
- Apply the analysis toolkit in order: read the inherited code, assess whether what's there is debt or style, cost out refactor-vs-extend concretely (including the Day 9 frontend zod-mirror impact), and **name the ROI horizon explicitly** before judging.
- Over PEP's 4 weeks, the default disposition is *extend* unless the refactor is small or the inherited shape blocks today's work. Over a multi-year horizon, the default flips toward Pattern A or C.
- The deliverable requires two contrasting question shapes to validate end-to-end. *Which pattern you used* is the candidate's call; *that you defended the call* is the curriculum's requirement.

---
*Prerequisites: Day 1 Topic 6 (brownfield decision frameworks — orientation depth), Day 2 (FastAPI services in the substrate, MongoDB containerization). Followed by `day-08-schema-validation-patterns` (the rule layer on top of whichever shape you chose).*
