# Unit Testing Patterns for Input Validation (pytest)

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

Every validator from Topic 2 is a *claim*. "Single-select must have exactly one correct option" is a claim about the system. The way to keep claims true as the code changes is to encode them as tests — and the cheapest, most readable way to encode many similar claims is `@pytest.mark.parametrize`.

Today's deliverable explicitly requires "pytest covers validators." This topic shows the shape that coverage should take: one parametrized test per rule, valid and invalid inputs side-by-side, exception types and messages asserted.

## The shape of a validator test

Every input-validation test answers two questions:

1. **Given a valid input,** does the model accept it and produce the expected normalized output?
2. **Given an invalid input,** does the model reject it with the *right* error in the *right* place?

Both halves matter. A test that only checks rejection lets a regression that silently passes bad input slip through; a test that only checks acceptance lets a too-strict validator block good input.

## A minimal valid factory

Before parametrizing, define the smallest helper that builds a valid payload. Each test mutates *only* the field under test, so failures are unambiguous.

```python
# services/question-management-service/tests/conftest.py
import pytest

def make_options(correct_indices: list[int], total: int = 4) -> list[dict]:
    return [
        {"id": chr(ord("a") + i), "text": f"Option {i}", "is_correct": i in correct_indices}
        for i in range(total)
    ]

@pytest.fixture
def valid_single_select() -> dict:
    return {
        "type": "single_select",
        "stem": "What is 2 + 2?",
        "options": make_options(correct_indices=[1]),
    }

@pytest.fixture
def valid_multi_select() -> dict:
    return {
        "type": "multi_select",
        "stem": "Which are even?",
        "options": make_options(correct_indices=[0, 2]),
    }
```

These fixtures are the *control* against which every parametrized variation is measured.

## Parametrized invalid-input tests

```python
# services/question-management-service/tests/unit/test_question_model.py
import pytest
from pydantic import ValidationError
from app.models.question import Question

# ---------- single-select: exactly one correct ----------
@pytest.mark.parametrize(
    "correct_indices, expected_msg_fragment",
    [
        ([],         "exactly 1 correct"),
        ([0, 1],     "exactly 1 correct"),
        ([0, 1, 2],  "exactly 1 correct"),
    ],
    ids=["zero_correct", "two_correct", "three_correct"],
)
def test_single_select_rejects_wrong_correct_count(
    valid_single_select, correct_indices, expected_msg_fragment,
):
    from tests.conftest import make_options  # or import once at top
    payload = {**valid_single_select, "options": make_options(correct_indices)}

    with pytest.raises(ValidationError) as exc:
        Question.model_validate(payload)

    errors = exc.value.errors()
    assert any(expected_msg_fragment in e["msg"] for e in errors), errors
    assert any(e["loc"][-1] == "single_select" or "single_select" in str(e["loc"]) for e in errors)

# ---------- multi-select: at least two correct ----------
@pytest.mark.parametrize(
    "correct_indices",
    [
        ([]),
        ([1]),
    ],
    ids=["zero_correct", "one_correct"],
)
def test_multi_select_rejects_fewer_than_two_correct(valid_multi_select, correct_indices):
    from tests.conftest import make_options
    payload = {**valid_multi_select, "options": make_options(correct_indices)}

    with pytest.raises(ValidationError) as exc:
        Question.model_validate(payload)

    assert any("at least 2 correct" in e["msg"] for e in exc.value.errors())

# ---------- shared base rule: unique option IDs ----------
@pytest.mark.parametrize("fixture_name", ["valid_single_select", "valid_multi_select"])
def test_option_ids_must_be_unique(request, fixture_name):
    payload = dict(request.getfixturevalue(fixture_name))
    payload["options"] = [
        {"id": "x", "text": "A", "is_correct": True},
        {"id": "x", "text": "B", "is_correct": False},
        {"id": "y", "text": "C", "is_correct": False},
    ]
    with pytest.raises(ValidationError) as exc:
        Question.model_validate(payload)
    assert any("option ids must be unique" in e["msg"] for e in exc.value.errors())
```

A few patterns to internalise:

- **`ids=[...]`** on parametrize makes test output human-readable: `test_single_select_rejects_wrong_correct_count[two_correct]` is much better than `[1]`.
- **One concept per test function.** Combining "wrong correct count" and "stem empty" in one parametrize set hides regressions.
- **Assert the message *fragment*, not the full string.** Full-string asserts break the moment someone improves the wording. Fragments survive editorial changes.
- **Assert the `loc`** to confirm Pydantic blamed the right variant — that's what proves the discriminator (Topic 1) is wired correctly.

## Parametrized valid-input tests

Symmetric to the above — prove the validator accepts what it should:

```python
@pytest.mark.parametrize(
    "correct_indices",
    [[0], [1], [3]],
    ids=["first", "middle", "last"],
)
def test_single_select_accepts_any_single_correct(valid_single_select, correct_indices):
    from tests.conftest import make_options
    payload = {**valid_single_select, "options": make_options(correct_indices)}
    q = Question.model_validate(payload)
    assert q.type == "single_select"
    assert sum(1 for o in q.options if o.is_correct) == 1
```

## Testing field-level constraints

`Field(min_length=1, max_length=2000)` deserves a test too — not because the constraint might fail, but because the *boundary* might shift accidentally:

```python
@pytest.mark.parametrize(
    "stem, valid",
    [
        ("",            False),
        (" ",           False),  # whitespace-only (custom field_validator)
        ("a",           True),
        ("x" * 2000,    True),
        ("x" * 2001,    False),
    ],
    ids=["empty", "whitespace", "min", "max", "overlimit"],
)
def test_stem_length_bounds(valid_single_select, stem, valid):
    payload = {**valid_single_select, "stem": stem}
    if valid:
        Question.model_validate(payload)  # should not raise
    else:
        with pytest.raises(ValidationError):
            Question.model_validate(payload)
```

Boundary tests (`min - 1`, `min`, `max`, `max + 1`) are where bugs hide. Always include them.

## Testing the discriminator dispatch

A small but important test: the discriminator routes correctly.

```python
@pytest.mark.parametrize(
    "fixture_name, expected_cls_name",
    [
        ("valid_single_select", "SingleSelectQuestion"),
        ("valid_multi_select",  "MultiSelectQuestion"),
    ],
)
def test_discriminator_dispatches(request, fixture_name, expected_cls_name):
    q = Question.model_validate(request.getfixturevalue(fixture_name))
    assert type(q).__name__ == expected_cls_name

def test_unknown_type_rejected():
    payload = {"type": "true_false", "stem": "x", "options": []}
    with pytest.raises(ValidationError) as exc:
        Question.model_validate(payload)
    assert any(e["type"] == "union_tag_invalid" for e in exc.value.errors())
```

## Running the suite in CI

The Day 7 CI matrix (the `test-backend` job) already runs `pytest` per service. These tests slot in directly — `tests/unit/test_question_model.py` runs in the `question-management-service` matrix shard, in parallel with the other two services. Coverage on the validator module should be 100% — there's no branch you can't drive from a parametrize row.

```bash
# Local invocation matches CI
cd services/question-management-service
pytest tests/unit -q --cov=app.models --cov-report=term-missing
```

Any line in `app/models/question.py` that the coverage report flags as `missing` is a validator branch with no test — fill it in before the PR (Topic 7) goes up.

## Example / Worked Scenario

You add a third question type — say, `numeric_input` — next sprint. The change is:

1. Add the model + variant validator in `models/question.py`.
2. Extend the discriminator union.
3. Add a fixture (`valid_numeric_input`) in `conftest.py`.
4. Add three parametrize sets in `test_question_model.py`: invalid cases, valid cases, discriminator dispatch.

If the validator regresses ("accepts negative numbers" when it shouldn't), the parametrize row catches it. If you delete a validator by accident in a refactor, the *acceptance* tests still pass but the *rejection* tests fail loudly — that asymmetry is what makes the suite trustworthy.

## Common Pitfalls

- **Asserting full error message strings.** They break on every wording tweak. Match a stable fragment.
- **One mega-test that loops over a list of cases.** When it fails, you don't know *which* case. Use `parametrize` so each case is its own reported test.
- **Testing only invalid inputs.** A validator that rejects everything passes those tests. Always pair with a parametrized acceptance suite.
- **Not asserting `loc`.** Without `loc` assertions, a validator that *rejects* the right input for the *wrong* reason (e.g., the stem rule fires instead of the multi-select rule) passes. The frontend will then highlight the wrong field.
- **Mutating the fixture dict instead of copying.** `payload = valid_single_select; payload["options"] = ...` mutates the fixture, so subsequent tests in the same session see the change. Always `payload = {**fixture, ...}` or `copy.deepcopy`.
- **Skipping boundary tests.** `min`, `min - 1`, `max`, `max + 1` are where length/range bugs live. Don't skip them because "obviously the type system covers it" — `Field(max_length=2000)` is data, not types.

## Key Takeaways

- Encode each validator from Topic 2 as a parametrized test of *both* the invalid cases (with message and `loc` assertions) and the valid cases.
- A single `make_options(correct_indices=...)` helper plus `valid_single_select` / `valid_multi_select` fixtures keep tests focused on the field under test.
- `ids=[...]` on parametrize makes failure output readable.
- Match message *fragments*, not full strings — fragments survive editorial changes; full strings don't.
- Boundary tests for `min/max` constraints are not optional.
- Coverage on `app/models/question.py` should be 100% after this work; anything less is a validator branch with no test.
- The Day 7 CI matrix already runs this in parallel — no infrastructure changes needed.

---
*Prerequisites: `day-8-pydantic-discriminated-unions-for-polymorphic-schemas`, `day-8-schema-validation-patterns`, Day 4 (CI quality gates that run this suite), Day 7 (matrix parallelization).*
