# Deterministic Scoring Algorithms (Exact-Match)

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Today's scoring engine starts with the simplest possible algorithm: **exact match**. A single-select question has one correct option, and the candidate either selects it (score = 1.0) or doesn't (score = 0.0). No partial credit, no gradient. This file establishes the baseline scoring contract — the function shape, the determinism guarantees, the data the function does and does not see — before the next topic introduces partial-credit algorithms for multi-select. Get this right and the multi-select layer slots in cleanly; get this wrong and you'll be re-plumbing the scoring engine all week.

## Why Start With Exact-Match

Three reasons:

- **It's the contract the substrate already implies.** Questions in MongoDB have a `correct_options: list[int]`. For `question_type = "single_select"`, that list has exactly one element. Exact-match is the trivially-correct algorithm — no design choices to defend.
- **It pins down the function signature.** Once `score_question(question, answer) -> ScoreResult` is fixed, the partial-credit algorithms in topic 2 plug into the same shape.
- **It surfaces the determinism rules.** Same inputs must always produce the same output. No randomness, no clock reads, no DB lookups inside the scoring function. Today is when you internalize that discipline.

## The Function Signature

```python
# test_management_service/app/scoring/exact_match.py
from dataclasses import dataclass

@dataclass(frozen=True)
class ScoreResult:
    score: float           # 0.0 .. 1.0
    is_correct: bool       # convenience: score == 1.0
    algorithm: str         # "exact_match" — for auditability


def score_exact_match(
    correct_options: frozenset[int],
    selected_options: frozenset[int],
) -> ScoreResult:
    if selected_options == correct_options:
        return ScoreResult(score=1.0, is_correct=True, algorithm="exact_match")
    return ScoreResult(score=0.0, is_correct=False, algorithm="exact_match")
```

Five things to notice:

1. **`frozenset[int]`** — not `list[int]`. Order doesn't matter for scoring, and freezing prevents accidental mutation. The caller normalizes from whatever shape the DB returns.
2. **The function takes *just* the option sets**, not the question or the session. Scoring is a pure function of `(correct, selected)`. Everything else (storing the result, marking the attempt scored) happens outside.
3. **`ScoreResult` is frozen and carries the algorithm name.** When you persist a score, you persist *which* algorithm produced it. On Day 16 the results page renders this; on D20 the architectural defense leans on it.
4. **No I/O, no logging, no time reads.** Pure function. Trivially testable, trivially memoizable, trivially deterministic.
5. **`is_correct` is a convenience field**, not a separate computation. The invariant `is_correct == (score == 1.0)` holds; tests will assert it.

## Determinism — What It Means and Why It's Non-Negotiable

A scoring function is **deterministic** if and only if: for any two invocations with the same input arguments, the output is byte-for-byte identical. Concretely, this rules out:

| Source of non-determinism | Example |
|---|---|
| Wall-clock time | `if datetime.now() > question.deadline: score *= 0.5` |
| Randomness | `score += random.uniform(0, 0.05)` (tiebreakers) |
| Mutable global state | Reading a feature flag inside the function |
| Iteration order over hashes | Building floats by iterating a `dict` in 3.6+ this is fine, but historically a hazard |
| DB lookups | Fetching the correct answer *inside* the scoring function |
| Floating-point summation order | Less of an issue here (we return 0.0 or 1.0) but a real hazard for partial-credit algorithms |

Why does it matter for *exact*-match where everything is 0 or 1? Because the discipline you set today is the discipline you keep tomorrow. When the engine grows to Jaccard scoring in topic 2 and idempotent submission in topic 3, the property "same `(correct, selected)` always returns the same score" is what lets you re-score from stored answers without re-running the entire session — and it's what lets the test suite be parametrized cleanly in topic 8.

## What the Function Does *Not* See

The scoring function is given a `correct_options` set and a `selected_options` set. It is **not** given:

- **The user.** Scoring doesn't know whose answer it is. Identity lives at the layer above (the submission endpoint).
- **The session.** Scoring doesn't know `session_id`, `current_index`, or whether the session has expired. Expiry is enforced at the endpoint (see topic 5 on locking, and D11's `expires_at` discipline).
- **The question text or explanation.** Scoring works on option sets. The text is for display.
- **Prior answers in the session.** Each answer is scored independently. Aggregate session score is a sum over per-question scores, computed at submit time.

This narrow scope is what makes the function pure. Resist the urge to "just pass the whole session in" for convenience.

## Normalization at the Boundary

The DB stores `correct_options` as a `list[int]`. The submission endpoint receives `selected_options` as a `list[int]` in the request body. Both get normalized into `frozenset[int]` before reaching the scoring function:

```python
# test_management_service/app/scoring/__init__.py
from .exact_match import score_exact_match, ScoreResult

def normalize_options(options: list[int]) -> frozenset[int]:
    return frozenset(options)


def score(
    question_type: str,
    correct_options: list[int],
    selected_options: list[int],
) -> ScoreResult:
    correct = normalize_options(correct_options)
    selected = normalize_options(selected_options)

    if question_type == "single_select":
        # Today's deliverable: exact match for single-select.
        return score_exact_match(correct, selected)

    if question_type == "multi_select":
        # Topic 2: partial-credit algorithm goes here.
        raise NotImplementedError("multi_select scoring lands in topic 2")

    raise ValueError(f"unknown question_type: {question_type!r}")
```

The boundary function does three jobs: normalize types, dispatch on question type, and surface unknown question types as a real error (not a silent zero).

## Worked Cases

| `correct_options` | `selected_options` | Result | Notes |
|---|---|---|---|
| `{2}` | `{2}` | `score=1.0, is_correct=True` | Trivial correct |
| `{2}` | `{0}` | `score=0.0, is_correct=False` | Wrong single-select |
| `{2}` | `set()` | `score=0.0, is_correct=False` | Empty — no credit |
| `{2}` | `{0, 2}` | `score=0.0, is_correct=False` | Multi-selected on single-select — wrong (the endpoint should also reject this at validation time, but if it gets through, exact-match returns 0) |
| `{1}` | `{1}` | `score=1.0, is_correct=True` | Different index, same shape |

The last row matters: scoring depends only on set equality, not on which specific indices appear. The same function works for `{1}` vs `{1}` and `{47}` vs `{47}`.

## Where This Lives in the Submission Flow

Once the answer arrives at `POST /sessions/{id}/answer`, the flow is:

```python
# test_management_service/app/routers/sessions.py
@router.post("/sessions/{session_id}/answer")
async def submit_answer(
    session_id: UUID,
    body: AnswerSubmit,
    db: DB,
    session: LockedSession,  # topic 5 — pessimistic lock
) -> AnswerResponse:
    question = await question_client.fetch(session.current_question_id)

    result = score(
        question_type=question.question_type,
        correct_options=question.correct_options,
        selected_options=body.selected_options,
    )

    await answer_repo.upsert(
        db,
        session_id=session_id,
        question_id=question.id,
        selected_options=body.selected_options,
        score=result.score,
        algorithm=result.algorithm,
    )
    # ... advance index, return next question
```

The scoring function is one line in the middle of a 20-line endpoint. That's the right ratio — the algorithm is small, the orchestration around it is the engineering.

## Persisting the Result

Two columns matter on the `answers` table:

```python
class AnswerRecord(Base):
    __tablename__ = "answers"
    session_id: Mapped[UUID] = mapped_column(primary_key=True)
    question_id: Mapped[UUID] = mapped_column(primary_key=True)  # composite — see topic 4
    selected_options: Mapped[list[int]] = mapped_column(JSON)
    score: Mapped[float]
    algorithm: Mapped[str]   # "exact_match" today; richer names from topic 2 on
    scored_at: Mapped[datetime]
```

Persisting `algorithm` per row means future re-scoring (audit, algorithm change, dispute resolution) is unambiguous. You can tell what produced any given score.

## Anti-Patterns

- **`return 1 if selected == correct else 0`.** Integers, not floats. Today they're indistinguishable; tomorrow's Jaccard scores need fractional output and your aggregation breaks if exact-match returns int.
- **List equality (`list == list`) instead of set equality.** Ordering should not affect correctness. `[1, 2]` and `[2, 1]` are the same answer.
- **Fetching `correct_options` inside the scoring function.** Pulls a DB dependency into a pure function. Pass the data in.
- **Branching on `question_type` inside `score_exact_match`.** That dispatch belongs in the boundary function. `score_exact_match` does one thing.
- **Returning just a float.** No room for the algorithm name, the `is_correct` flag, or future fields like `partial_credit_basis`. The `ScoreResult` dataclass costs nothing and pays off in auditability.

## Key Takeaways
- Exact-match scoring returns 1.0 on set equality and 0.0 otherwise — that's the entire algorithm.
- The scoring function is **pure**: takes two frozensets, returns a `ScoreResult`, does no I/O.
- Normalize `list[int]` to `frozenset[int]` at the boundary; the scoring function never sees lists.
- Persist the `algorithm` name with each score so future audits and re-scoring stay unambiguous.
- The function signature established today (`(correct, selected) -> ScoreResult`) is the same shape topic 2's partial-credit algorithms plug into.

---
*Prerequisites: day-11 session model, day-8 question schema. Forward references: day-12 partial-credit scoring, day-16 results aggregation.*
