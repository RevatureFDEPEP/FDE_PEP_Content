# AI-Assisted Test Authoring (Concept Preview)

> *Day 5: Advanced Container Builds & Multi-Service Integration — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*
> *Unit 0: AI Tooling Thread*

## Overview
This topic is a **preview**, not a hands-on. The full exercise — generating a parametrized pytest suite for the scoring engine, with coverage gates and CI integration — lands on Day 12 (Week 3) when there's something meaningful to score. Today's goal is narrower: **see the pattern once, so it isn't new in Week 3.**

The pattern is: **given a specification, ask Claude Code to author parametrized test cases that exercise the spec.** It's a sharper, more disciplined variant of the inline code generation you just practiced. Specs are uniquely well-suited to AI test authoring because the *contract* (input → expected output) is explicit. The agent generates *cases*; you supply the *contract*.

The earlier topic today (inline code generation) targeted infra glue, where the win is speed and the failure mode is loud. Test authoring is different: the *failure mode is silent*. A generated test that passes when it shouldn't is worse than no test — it manufactures false confidence. The mitigation patterns matter, and that's why this preview exists today instead of being dropped on you cold in Week 3.

## The Pattern, At a Glance
```
   [explicit spec]  -->  [agent generates parametrized cases]
                              |
                              v
                  [you review each case for correctness]
                              |
                              v
                [you run the suite; verify intentional failures]
                              |
                              v
                       [commit]
```

Three things differ from glue generation:

1. **The input is a spec, not a vague task.** You don't say "write tests for the scoring service." You say "given a question with N options where exactly K are correct, and the user selects S of those, the score is max(0, |S∩correct| - |S\correct|) / K. Here are 15 representative cases I want covered..."
2. **The output is parametrized, not freeform.** The agent emits a `@pytest.mark.parametrize` table (or equivalent for jest), not 15 separate test functions. This is a strong forcing function — it makes the cases comparable.
3. **You verify with deliberate failures.** You temporarily break the implementation and confirm the new tests catch it. A test that doesn't fail when the code is wrong didn't test anything.

## What a Good Spec-to-Test Prompt Looks Like
A canonical example (preview only — actual Week 3 work will be richer):

```
I have a function `score_question(correct_options, selected_options,
total_options)` that returns a float between 0.0 and 1.0.

The contract:
- Inputs are sets of option indices (0-based integers).
- correct_options is a non-empty subset of {0..total_options-1}.
- selected_options is a (possibly empty) subset of {0..total_options-1}.
- Score formula: max(0, |selected ∩ correct| - |selected \ correct|)
  divided by |correct|.
- Selecting nothing returns 0.0.
- Selecting only correct options returns the fraction of correct
  options selected.
- Selecting any wrong option penalises 1/|correct| per wrong option.
- The function never returns negative; clamp to 0.0.

Generate a pytest.mark.parametrize table covering:
- All-correct selection
- All-incorrect selection
- Empty selection
- Exactly half correct
- Exactly one correct, several wrong
- Single-correct-option question (|correct| = 1)
- Many-correct-option question (|correct| = total_options - 1)
- Over-selection (penalty pushes score to zero)

For each case, include a brief `id` so failures are diagnosable. Don't
invent additional contract rules; if any case requires a rule I haven't
stated, mark it XFAIL with a comment.
```

The "don't invent additional contract rules" instruction is the test-authoring equivalent of "don't invent context" from glue generation. Without it, the agent will quietly fill in plausible scoring rules ("partial credit for ordering"? "bonus for confidence?") that don't match your spec, and the tests will pass against an implementation that doesn't actually match.

## Why You Don't Just Let the Agent Write the Whole Suite
Test code has a particularly nasty failure mode: **a green suite is not evidence of correctness**. It's evidence that the tests and the code agree. If the agent generated both the tests *and* the code from the same prompt, they will agree on whatever misreading of the spec the agent committed to. Both wrong; both green.

This is why the Week 3 exercise will:

1. Specify the contract **in writing** (not just in a prompt).
2. Have a *human* implement the function from the contract.
3. Have the agent generate tests from the **same written contract**, not from the implementation.
4. Run the suite. Confirm failures match the contract's edge cases when the implementation is broken on purpose.

The contract is the source of truth. Both human and agent derive from it independently. Disagreement surfaces as a failing test; agreement means *at least one of them* might be right. (You confirm by mutation testing: introduce a deliberate bug. Does any test fail? If not, the agent generated weak cases.)

## What Today's Preview Asks of You
You will **not** write tests today. You will:

1. **Read this topic.**
2. **Look at one demo prompt + output pair** the trainer will walk through, ideally for one of the simpler endpoints in user-service or test-management-service.
3. **Note the pattern** so when Day 12 says "generate parametrized cases for the scoring engine," you already know what shape that means.

That's it. The preview's job is to remove the *novelty* on Day 12 so the cohort can focus on the *quality* of the cases.

## Example / Worked Scenario (Preview Walkthrough)
The trainer demonstrates with an artificially small example — a `normalize_username` function in user-service:

**Spec:**
```
normalize_username(s: str) -> str
- Strips leading/trailing whitespace.
- Lowercases.
- Rejects (raises ValueError) if empty after stripping.
- Rejects if length > 64 after stripping.
- Otherwise returns the normalized string.
```

**Prompt to Claude Code (abbreviated):**
```
Generate a pytest.mark.parametrize table for normalize_username covering:
- Whitespace-only input (should raise)
- Empty string (should raise)
- Mixed-case input (should lowercase)
- Leading/trailing whitespace (should strip)
- 64-character input (should pass)
- 65-character input (should raise)
- Unicode characters (lowercased correctly)

Don't invent additional rules. If a case needs a rule I haven't stated,
mark it XFAIL with a comment.
```

**Output (paraphrased):**
```python
import pytest
from user_service.normalize import normalize_username

@pytest.mark.parametrize("raw,expected", [
    pytest.param("Alice", "alice", id="mixed-case"),
    pytest.param("  bob  ", "bob", id="trim-whitespace"),
    pytest.param("a" * 64, "a" * 64, id="max-length-64"),
])
def test_normalize_username_valid(raw, expected):
    assert normalize_username(raw) == expected


@pytest.mark.parametrize("raw", [
    pytest.param("", id="empty"),
    pytest.param("   ", id="whitespace-only"),
    pytest.param("a" * 65, id="over-max-length"),
])
def test_normalize_username_invalid(raw):
    with pytest.raises(ValueError):
        normalize_username(raw)


@pytest.mark.xfail(reason="Spec doesn't specify unicode case-folding behaviour")
def test_normalize_username_unicode():
    assert normalize_username("İSTANBUL") == "i̇stanbul"
```

**The trainer points at three things:**

1. **The parametrize tables are split** by outcome (valid vs raises). Cleaner than one giant table with `expected` sometimes being an exception.
2. **The unicode case was correctly marked XFAIL** because the spec didn't define case-folding for non-ASCII. The agent didn't invent a rule — it surfaced the gap. That's the prompt's "don't invent" clause doing its job.
3. **The IDs make failures diagnosable.** When `id="over-max-length"` fails, you know what broke without reading the parameters.

**The trainer then runs a mutation test.** They edit `normalize_username` to skip the length check. Re-run pytest. `test_normalize_username_invalid[over-max-length]` fails. Good — the test would have caught a real bug.

That's the preview. Week 3 expands it to the scoring engine, where the spec is richer and the case-design is the actual exercise.

## Common Pitfalls (To Recognize, Not Yet to Practice)
- **Asking the agent to write tests *from the implementation*.** This produces tests that bless current behavior, not specified behavior. The contract must come from outside the code.
- **Accepting cases without checking the expected values.** "The agent said the answer is `0.6`, I'll trust it." No. For each case, derive the expected value from the spec yourself or you've outsourced your judgment.
- **No mutation test.** If you don't break the implementation and confirm the tests catch it, you don't know the tests work. This is *the* essential validation step for generated tests.
- **Letting the agent fill spec gaps with plausible defaults.** The XFAIL pattern above is the right escape valve. If the agent generates a test for a rule you didn't specify, that's a signal to clarify the spec, not to accept the test.
- **Over-relying on parametrize for everything.** Some cases (complex setup, mocking) deserve standalone test functions. Parametrize is the default, not a mandate.

## Key Takeaways
- This is a preview — full exercise lands on Day 12 with real scoring logic.
- Generate tests from the *spec*, not from the implementation.
- The agent generates *cases*; the spec is *yours*.
- Always validate generated tests with a deliberate mutation. Green tests on a broken implementation = weak tests.
- The "don't invent rules" prompt clause is non-negotiable; XFAIL is the right escape valve for spec gaps.

---
*Prerequisites: `day-1-ai-augmented-development-claude-code-agent-tooling-fundamentals.md`, `day-5-ai-assisted-inline-code-generation.md`. Forward link: `day-12-*.md` (the Week 3 exercise).*
