# Partial-Credit Scoring Algorithms (Full-Match, Jaccard, Set-Overlap)

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Single-select scoring is unambiguous. Multi-select scoring is **not** — it's a design choice, and different choices produce different scores for the same answer. This file walks through three algorithms that all reasonably claim to "score multi-select questions correctly": **full-match**, **Jaccard similarity**, and **set-overlap with penalty**. Each is shown on the same worked input set so the differences are visible side-by-side. You will pick one for the substrate today; on Day 20 you'll defend that choice in the architecture review. There is no universally right answer — only an answer you can justify for a 4-week PEP cohort.

## The Setup

A multi-select question with **4 options total**, of which **3 are correct**:

```
correct_options  = {0, 1, 2}
total_options    = 4         # so option index 3 is the wrong one
```

We'll score five candidate answers under each algorithm:

| Case | `selected_options` | Description |
|---|---|---|
| A | `{0, 1, 2}` | Perfect — all correct, none wrong |
| B | `{0, 1}` | Two of three correct, none wrong |
| C | `{0, 1, 2, 3}` | All correct *plus* one wrong |
| D | `{0, 3}` | One correct, one wrong |
| E | `{3}` | Only the wrong one |

Same five cases, three algorithms, one comparison.

## Algorithm 1 — Full-Match (Binary)

The same exact-match rule from topic 1, applied to multi-select. The candidate gets credit only if the selection equals the correct set exactly.

```python
def score_full_match(
    correct: frozenset[int],
    selected: frozenset[int],
) -> ScoreResult:
    score = 1.0 if selected == correct else 0.0
    return ScoreResult(
        score=score,
        is_correct=(score == 1.0),
        algorithm="full_match",
    )
```

Properties:

- **Strictest.** No partial credit. "Almost right" is wrong.
- **Trivially fair.** Everyone is judged by the same line.
- **Punishing for near-misses.** A candidate who picked 2 of 3 correct gets the same score as one who picked all 4 wrong options.

## Algorithm 2 — Jaccard Similarity

The **Jaccard index** is the size of the intersection divided by the size of the union: `|A ∩ B| / |A ∪ B|`. Standard in information retrieval and set-similarity problems.

```python
def score_jaccard(
    correct: frozenset[int],
    selected: frozenset[int],
) -> ScoreResult:
    if not correct and not selected:
        score = 1.0           # vacuous match; not reachable here
    else:
        score = len(correct & selected) / len(correct | selected)
    return ScoreResult(
        score=round(score, 4),
        is_correct=(score == 1.0),
        algorithm="jaccard",
    )
```

Properties:

- **Symmetric.** Treats false-positive and false-negative the same way (both shrink the intersection / grow the union).
- **Bounded [0, 1].** Always.
- **Penalizes over-selection and under-selection equally.** Picking all four options (3 correct + 1 wrong) and picking only two of three correct can score the same.

## Algorithm 3 — Set-Overlap With Penalty

A score that **rewards correct selections and penalizes incorrect ones**, normalized by the number of correct options, clamped at zero:

```python
def score_set_overlap(
    correct: frozenset[int],
    selected: frozenset[int],
) -> ScoreResult:
    if not correct:
        score = 0.0
    else:
        correct_selected = len(correct & selected)
        incorrect_selected = len(selected - correct)
        raw = (correct_selected - incorrect_selected) / len(correct)
        score = max(0.0, raw)
    return ScoreResult(
        score=round(score, 4),
        is_correct=(score == 1.0),
        algorithm="set_overlap",
    )
```

Properties:

- **Asymmetric.** A wrong selection costs `1/|correct|`; a missed selection costs `1/|correct|` (same magnitude, expressed differently in the formula). The clamp matters: scoring three wrong options with one correct selected gives a negative raw score, clamped to 0.
- **Discourages "select everything."** Picking all four options when 3 are correct yields `(3 - 1) / 3 = 0.67`, not 1.0.
- **Linear in the number of correct/incorrect selections.** Easy to explain to candidates.

## Side-by-Side On The Same Inputs

`correct = {0, 1, 2}`, `total_options = 4`:

| Case | `selected` | full_match | jaccard | set_overlap |
|---|---|---|---|---|
| A — perfect | `{0,1,2}` | **1.00** | 3/3 = **1.00** | (3−0)/3 = **1.00** |
| B — partial, no wrong | `{0,1}` | **0.00** | 2/3 = **0.67** | (2−0)/3 = **0.67** |
| C — perfect + 1 wrong | `{0,1,2,3}` | **0.00** | 3/4 = **0.75** | (3−1)/3 = **0.67** |
| D — 1 right, 1 wrong | `{0,3}` | **0.00** | 1/4 = **0.25** | (1−1)/3 = **0.00** |
| E — only wrong | `{3}` | **0.00** | 0/4 = **0.00** | (0−1)/3 = clamp → **0.00** |

A few of the most interesting comparisons:

- **Case B vs C.** Same Jaccard rank? No — C beats B under Jaccard (0.75 > 0.67) because Jaccard rewards "you got all the correct ones." But set-overlap *ties* them (both 0.67) because the wrong selection in C exactly cancels one correct selection. Set-overlap says "you proved you don't know that option 3 is wrong" and treats that ignorance with the same weight as missing a correct answer.
- **Case D.** Jaccard says 0.25 ("you got *something* right"). Set-overlap says 0.0 ("you got one right and one wrong — that's guessing"). Same candidate behavior, very different score.
- **Full-match** treats B, C, D, E identically. From a pedagogy lens, this collapses signal — the trainer dashboard can't distinguish "almost right" from "completely wrong."

## Picking One — The Defensible Choice

For PEP, the recommended choice is **`set_overlap`**:

1. **Discourages "select everything" gaming.** A candidate who selects all 4 options gets 0.67, not 1.0. Jaccard gives 0.75 for the same gaming behavior — meaningfully softer.
2. **Linear and explainable.** Candidates can predict their own score: "3 right minus 1 wrong, over 3 correct = 67%." Jaccard requires explaining intersections-over-unions.
3. **Symmetric penalty for false-positive and false-negative**, normalized by the *correct* set size (not the union). A 4-correct-option question and a 2-correct-option question are scored on the same 0..1 scale, weighted by what *should* have been picked.
4. **Backwards-compatible with exact-match.** For single-select (`|correct| = 1`), set-overlap reduces to `(1 - 0)/1 = 1.0` for the right answer and `max(0, (0 - 1)/1) = 0.0` for any other selection. Same answers as topic 1's exact-match.

Full-match is **too punishing** for a learning context (no partial credit on a slip). Jaccard is **defensible** and you should be able to argue why it's a reasonable alternative — but in a PEP cohort where candidates are learning the material, "select everything" gaming is the bigger risk than "punish the borderline case." Set-overlap wins on that axis.

On Day 20 you defend this choice. Write the reasoning down today while it's fresh.

## Dispatch in the Boundary

```python
# test_management_service/app/scoring/__init__.py
from .exact_match import score_exact_match, ScoreResult
from .partial_credit import score_set_overlap

def score(
    question_type: str,
    correct_options: list[int],
    selected_options: list[int],
) -> ScoreResult:
    correct = frozenset(correct_options)
    selected = frozenset(selected_options)

    if question_type == "single_select":
        return score_exact_match(correct, selected)
    if question_type == "multi_select":
        return score_set_overlap(correct, selected)
    raise ValueError(f"unknown question_type: {question_type!r}")
```

The full-match and Jaccard implementations stay in the module (as `score_full_match`, `score_jaccard`) — they're not dead code, they're the parametrized-test fodder for topic 8 and the alternative-algorithm comparison material for Day 20.

## Persisting the Algorithm Name

The `algorithm` column on the `answers` table is no longer cosmetic. With three algorithms living in the codebase, knowing *which one produced a given score* is essential for audits and dispute resolution.

```sql
SELECT question_id, score, algorithm
FROM answers
WHERE session_id = '0193d3a4-...';
-- algorithm column lets a Week 4 trainer-dashboard query answer:
-- "show me all questions scored with set_overlap where score < 1.0"
```

If the algorithm ever changes (suppose D20's review concludes Jaccard would be better), old answers can be re-scored from `selected_options` and the new algorithm, and the algorithm column tells you *which* rows need re-scoring.

## Floating-Point Discipline

Partial credit means real-valued scores. A few rules:

- **Round at the boundary, not mid-pipeline.** The function rounds to 4 decimal places on output. Intermediate arithmetic stays in full float precision.
- **Don't compare scores for equality with `==`.** For "is this a perfect score?", compare `score == 1.0` is OK because 1.0 is exactly representable. For comparing two partial scores, use `math.isclose`.
- **Don't sum scores across questions and then round.** Each question's score is rounded; the session total is `sum(per_question_scores)` and may have small float artifacts. Render with two decimal places in the UI; store full precision.

## Anti-Patterns

- **Mixing algorithms within a single session.** Use the question-type-driven dispatch; don't have an admin flag that flips Jaccard on for some sessions and not others. Audits become impossible.
- **Hard-coding the algorithm choice deep inside the scoring function.** Keep dispatch at the boundary; the per-algorithm functions are pure.
- **Forgetting the empty-`correct` guard.** If a malformed question has no correct options, divide-by-zero awaits. Guard explicitly.
- **Counting "missed correct options" as a penalty *in addition to* normalizing by `|correct|`.** Double-counting. The denominator already captures the missing ones.
- **Storing only the float score, not the algorithm name.** Two months later you'll have no way to explain a score to a candidate disputing it.

## Key Takeaways
- Three reasonable algorithms exist for multi-select: full-match (binary), Jaccard (|A∩B|/|A∪B|), set-overlap with penalty.
- On the same input set they produce visibly different scores — the choice is a real design decision, not a detail.
- PEP picks **set-overlap**: discourages "select everything" gaming, linear and explainable, reduces to exact-match for single-select.
- Persist the algorithm name with every score so future audits and re-scoring stay unambiguous.
- Day 20 defends this choice; the full-match and Jaccard implementations stay in the module as both test material (topic 8) and comparison material (D20).

---
*Prerequisites: [01-deterministic-scoring-algorithms-exact-match.md](01-deterministic-scoring-algorithms-exact-match.md). Forward references: [08-ai-assisted-test-authoring-with-parametrized-pytest-cases.md](08-ai-assisted-test-authoring-with-parametrized-pytest-cases.md) (parametrized tests over all three algorithms), day-20 architecture defense.*
