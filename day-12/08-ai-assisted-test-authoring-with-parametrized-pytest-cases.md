# AI-Assisted Test Authoring with Parametrized pytest Cases

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*
> *Unit 0: AI Tooling Thread*

## Overview
On Day 5 you previewed the pattern: given an explicit specification, ask Claude Code to draft a parametrized pytest table that exercises it. Today the preview pays off. The scoring engine is *the* canonical use case — three algorithms, dozens of edge cases per algorithm, the contract is small enough to fit in one prompt, and a green test suite has unambiguous meaning. This file walks through writing the prompt, generating the cases, verifying them (the verify-and-own loop from D8), and committing. The point is not "AI wrote tests for me"; the point is "I authored thorough tests in 20 minutes that would have taken 90 minutes by hand, **and I own every assertion**."

## Why Scoring Is The Ideal Target

Test-authoring with AI works well when:

- **The contract is explicit and small.** Topic 2 gave us three algorithms, each definable in one paragraph and one formula.
- **The cases are enumerable.** Empty selection, full selection, partial-correct, partial-incorrect, partial-mixed, single-correct edge, single-incorrect edge, etc.
- **Each case has an unambiguous expected output.** Apply the formula to the inputs; that's the expected value. No "vibes" allowed.
- **The implementation is a pure function.** No mocks, no fixtures, no async.

Scoring hits all four. By contrast, AI test authoring for the answer-submission endpoint (which involves a DB, locks, and idempotency state) is *harder* — you'd want to do that with hand-written integration tests informed by the table in topic 7. Save AI authoring for where it shines.

## The Prompt

A concrete prompt to give Claude Code (paste this into the terminal session, with the scoring module open in context):

```
I have three scoring algorithms in test_management_service/app/scoring/:

  exact_match.score_exact_match(correct, selected)  -> ScoreResult
  partial_credit.score_full_match(correct, selected)
  partial_credit.score_jaccard(correct, selected)
  partial_credit.score_set_overlap(correct, selected)

Where:
  correct, selected : frozenset[int]
  ScoreResult       : (score: float, is_correct: bool, algorithm: str)

Contracts:

  exact_match / full_match
    score = 1.0 if selected == correct else 0.0
    is_correct = (score == 1.0)

  jaccard
    if not correct and not selected: score = 1.0  (vacuous; unreachable here)
    else: score = |correct ∩ selected| / |correct ∪ selected|
    score rounded to 4 decimals

  set_overlap
    if not correct: score = 0.0
    else:
      raw = (|correct ∩ selected| - |selected − correct|) / |correct|
      score = max(0.0, raw)
    score rounded to 4 decimals

Generate a pytest test file `tests/scoring/test_scoring_algorithms.py`
with @pytest.mark.parametrize cases covering, for EACH algorithm:

  1. Perfect match
  2. Empty selection on a non-empty correct set
  3. All-wrong selection (no overlap, non-empty selected)
  4. Partial correct, no incorrect
  5. Perfect correct + one extra incorrect (over-selection)
  6. One correct + one incorrect
  7. Single-correct-option question (|correct| = 1) — selected correctly
  8. Single-correct-option question — selected incorrectly
  9. Many-correct-option question (|correct| = 5) — mix scenarios

Use `correct = frozenset({0, 1, 2})` as the standard fixture for cases
1–6 (with `total_options = 4`, so index 3 is the wrong option).
For case 7 use `correct = frozenset({2})`. For case 9 use
`correct = frozenset({0, 1, 2, 3, 4})`.

Each case must have:
  - a descriptive `id` (e.g., "set_overlap-over_select-3correct-1wrong")
  - the input `(correct, selected)`
  - the expected `score` and `is_correct`
  - a one-line comment explaining the expected value

DO NOT invent additional algorithm rules. If a case would require a
rule I haven't stated, mark it with @pytest.mark.xfail and a comment.

Produce one test function per algorithm. Use `math.isclose(a, b,
rel_tol=0, abs_tol=1e-4)` for the score comparison.
```

The discipline in this prompt mirrors Day 5's preview:

- **"DO NOT invent additional algorithm rules."** Without this, the agent invents partial-credit rules for `full_match` to "be helpful" — and the tests pass against a wrong implementation.
- **Explicit fixture values.** The agent doesn't pick random `correct` sets; we control them so the expected values are pre-computable.
- **One function per algorithm.** Keeps the parametrize tables comparable and the diff manageable.
- **`xfail` for "out of contract"** — forces the agent to flag uncertainty instead of making things up.

## What The Generated Output Should Look Like

```python
# tests/scoring/test_scoring_algorithms.py
import math
import pytest

from test_management_service.app.scoring.exact_match import score_exact_match
from test_management_service.app.scoring.partial_credit import (
    score_full_match, score_jaccard, score_set_overlap,
)

C123 = frozenset({0, 1, 2})

@pytest.mark.parametrize(
    "selected, expected_score, expected_is_correct",
    [
        pytest.param(frozenset({0, 1, 2}), 1.0, True,   id="exact-perfect"),
        pytest.param(frozenset(),          0.0, False,  id="exact-empty"),
        pytest.param(frozenset({3}),       0.0, False,  id="exact-only-wrong"),
        pytest.param(frozenset({0, 1}),    0.0, False,  id="exact-partial"),
        pytest.param(frozenset({0,1,2,3}), 0.0, False,  id="exact-over-select"),
        # ... etc
    ],
)
def test_exact_match(selected, expected_score, expected_is_correct):
    result = score_exact_match(C123, selected)
    assert math.isclose(result.score, expected_score, rel_tol=0, abs_tol=1e-4)
    assert result.is_correct is expected_is_correct
    assert result.algorithm == "exact_match"


@pytest.mark.parametrize(
    "selected, expected_score, expected_is_correct",
    [
        pytest.param(frozenset({0, 1, 2}),  1.0,    True,   id="jaccard-perfect"),
        # |{} ∩ {0,1,2}| / |{} ∪ {0,1,2}| = 0/3
        pytest.param(frozenset(),           0.0,    False,  id="jaccard-empty"),
        # |{3} ∩ C| / |{3} ∪ C| = 0/4
        pytest.param(frozenset({3}),        0.0,    False,  id="jaccard-only-wrong"),
        # |{0,1} ∩ C| / |{0,1} ∪ C| = 2/3 ≈ 0.6667
        pytest.param(frozenset({0, 1}),     0.6667, False,  id="jaccard-partial"),
        # |C ∩ {0,1,2,3}| / |C ∪ {0,1,2,3}| = 3/4
        pytest.param(frozenset({0,1,2,3}),  0.75,   False,  id="jaccard-over-select"),
        # ... etc
    ],
)
def test_jaccard(selected, expected_score, expected_is_correct):
    result = score_jaccard(C123, selected)
    assert math.isclose(result.score, expected_score, rel_tol=0, abs_tol=1e-4)
    assert result.is_correct is expected_is_correct
    assert result.algorithm == "jaccard"


# ... test_full_match and test_set_overlap similarly
```

The expected values in the comments are the **arithmetic**, not a re-statement of the test name. That's intentional — if a value is wrong, the comment surfaces the bug ("you said 2/3 but wrote 0.75 — which is right?") instead of hiding it.

## The Verify-and-Own Loop

The output is a starting point, not the finished product. The verify-and-own loop from D8 has four steps for test code:

### Step 1 — Read every assertion

For each parametrize row, manually compute the expected value from the formula and confirm it matches what the agent wrote. The comments make this fast — you're verifying arithmetic, not deriving algorithms.

```
jaccard-partial: selected = {0,1}, correct = {0,1,2}
  intersection = {0,1} → size 2
  union = {0,1,2} → size 3
  score = 2/3 ≈ 0.6667 ✓
```

If a row is wrong, fix the expected value, note that you fixed it in the commit message ("[ai-author] corrected jaccard-partial expected from 0.5 to 0.6667"). If many are wrong, scrap the table and re-prompt with sharper constraints.

### Step 2 — Check for missing cases

The prompt asked for 9 categories per algorithm; the agent may have produced 7 or 11. Compare against the topic-2 worked-cases table. Any category from that table missing? Add it by hand.

Specific cases to check for, since they're easy to miss:

- **Set-overlap clamp.** A case where the raw score goes negative (e.g., selected = `{3}` against correct = `{0}` with |correct| = 1). Verify the clamp fires.
- **Jaccard with no overlap, non-empty both sides.** Verify the denominator is the *union* size, not the *correct* size.
- **Single-correct, multi-selected.** `correct = {2}`, `selected = {0, 1, 2}` — under set-overlap, score = `(1 - 2) / 1 = -1`, clamped to 0.

### Step 3 — Run the suite

```bash
cd test_management_service && pytest tests/scoring/test_scoring_algorithms.py -v
```

Expected outcome: **all green**. If anything fails, either the test's expected value is wrong (the common case — fix the test) or the implementation is wrong (the uncommon case — fix the impl). Either way, understand *why* before changing anything.

### Step 4 — Mutation test (the "deliberate failures" check)

Temporarily break the implementation and confirm tests catch it:

```python
# test_management_service/app/scoring/exact_match.py
def score_exact_match(correct, selected):
    # MUTATION: always return 1.0
    return ScoreResult(score=1.0, is_correct=True, algorithm="exact_match")
```

Run the suite. It must fail. If it doesn't, your tests aren't asserting what they claim to assert. Revert the mutation, fix the tests, repeat.

Run the mutation check at least three times with different mutations:

| Mutation | Should fail at... |
|---|---|
| Always return 1.0 | Most non-perfect cases |
| Always return 0.0 | The perfect-match cases |
| Use `len(correct) - len(selected)` instead of intersection | The over-select and under-select cases |

If a mutation produces a green suite, you have a coverage hole. Add a case.

## What "Own" Means For Generated Tests

After the verify-and-own loop, you should be able to:

- **Explain every parametrize id without looking at the code.** "jaccard-partial means jaccard of {0,1} against {0,1,2}, expected 2/3."
- **Predict, from the formula, what `set_overlap` returns for any new (correct, selected) pair.** If you can't, you don't yet own the algorithm — re-read topic 2.
- **Defend the choice of `set_overlap` in PR review.** Same defense as topic 2.

If you can't do these three, you have not done the verify step properly. **The point of AI test authoring is not to skip the thinking — it's to skip the typing.**

## Commit Message Convention

The Day-4 AI-PR convention applies. Commit message:

```
test(scoring): parametrized cases for exact_match / full_match / jaccard / set_overlap

Generated with Claude Code from the algorithm contracts in topic 2,
verified against the worked-cases table, mutation-tested with three
deliberate impl bugs (always-1, always-0, len-diff). All caught.

Corrections from generated output:
  - jaccard-partial expected: 0.5 → 0.6667 (agent miscomputed union size)
  - set_overlap-single-many-wrong: added (missing from generated set)
```

The "Corrections from generated output" section is *non-optional*. If you didn't correct anything, you didn't verify hard enough — go back to step 1.

## When AI Test Authoring Doesn't Apply Today

Don't use AI authoring for these Day 12 test surfaces:

- **The lock-and-state integration tests.** These need real DB transactions, real concurrency. Author by hand using `asyncio.gather` of two competing requests.
- **The idempotency replay tests.** Stateful, end-to-end-ish. Author by hand.
- **The HTTP-status-code edge case tests** from topic 7. The mapping table *is* the spec; convert it to tests yourself so you internalize it.

Reserve AI authoring for the pure-function scoring algorithms. The other surfaces have hidden state that the agent can't see, and a wrong test gives false confidence.

## Anti-Patterns

- **Pasting generated tests into the repo without reading them.** Green suite with wrong assertions = worst possible state.
- **Generating tests *and* the implementation in the same prompt.** They agree on whatever the agent misread; both wrong, both green.
- **Skipping the mutation check "because the tests look thorough."** They look thorough. Until you break the impl, you don't know.
- **Pasting the full source of the scoring module into the prompt.** Then the agent generates tests against the *implementation*, not the *contract*. The implementation may be wrong; the tests will agree.
- **"Iterate" by asking the agent "add more cases" without specifying which ones.** You get random additions, not coverage gaps closed. Specify.

## Key Takeaways
- Scoring algorithms are the ideal target for AI test authoring: small explicit contracts, enumerable cases, pure functions, unambiguous expected outputs.
- The prompt specifies the contract, fixture values, and required case categories — and explicitly forbids inventing rules.
- Verify-and-own has four steps: read every assertion, check for missing cases, run, mutation-test.
- The commit message documents corrections made to the generated output; if none, you didn't verify.
- AI authoring is for pure-function tests today; the integration and edge-case tests from topic 7 are hand-authored.

---
*Prerequisites: [07-ai-assisted-test-authoring-concept-preview.md](../day-05/07-ai-assisted-test-authoring-concept-preview.md), [07-ai-assisted-code-generation-in-practice.md](../day-08/07-ai-assisted-code-generation-in-practice.md), [01-deterministic-scoring-algorithms-exact-match.md](01-deterministic-scoring-algorithms-exact-match.md), [02-partial-credit-scoring-algorithms.md](02-partial-credit-scoring-algorithms.md), [07-edge-case-handling-for-distributed-clients.md](07-edge-case-handling-for-distributed-clients.md). Forward references: day-15 integration test polish.*
