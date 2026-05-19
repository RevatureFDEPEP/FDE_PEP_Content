# AI-Assisted Code Generation in Practice — Own the Output, Defend in Review

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*
> ⚡ *AI Tooling Thread*

## Overview

Until now, Claude Code has been a comprehension and debugging partner — reading the substrate (Day 1), proposing diagnostic hypotheses (Day 3), drafting a PR description (Day 4). Today is the first day it generates *non-trivial production code* that ships in a PR: the Pydantic models, validators, repository, routes, and tests of Topics 1-6. Tomorrow a peer reviews it.

The job has not changed. *You* author the code. Claude Code drafts; you verify; you defend. If a reviewer asks "why this regex?" and your only answer is "Claude wrote it," the PR isn't ready. This topic is the discipline that turns AI-assisted generation from a productivity gimmick into something a brownfield team actually trusts.

## The verify-and-own loop

Every AI-generated chunk passes through four checkpoints before it goes into a commit:

1. **Read it.** Slowly. Not skim. Every line that ships, you have read.
2. **Run it.** Tests, the actual endpoint, edge cases. If it has zero tests, it didn't run.
3. **Edit it.** At least one substantive change — naming, structure, comment, error message — so the code is yours, not a transcript.
4. **Defend it.** Be able to answer "why this approach?" for every non-trivial choice. If you can't, you don't understand it well enough to merge it.

The four steps are not negotiable, and the order matters. People who skip step 1 ("I'll read it during review") merge code they don't understand. People who skip step 2 ship code that looks plausible and isn't. People who skip step 3 produce PRs that read like AI output — and reviewers notice.

## A concrete prompt for today's deliverable

A good prompt is *specific* about context, *concrete* about scope, and *explicit* about constraints. Vague prompts get vague code.

> **Prompt to Claude Code, run from `services/question-management-service/`:**
>
> Read `app/models/`, `app/db/`, `app/routes/`, and `tests/`. We're adding question authoring to this service. I need:
>
> 1. `app/models/question.py` — Pydantic v2 models for `single_select` and `multi_select` questions using a discriminated union on `type`. Shared base with `stem`, `options: list[Option]`, `difficulty`, `tags`, `image_key`. Each `Option` has `id`, `text`, `is_correct`.
> 2. Cross-field validators: single-select requires exactly 1 `is_correct: true`; multi-select requires >= 2; option IDs must be unique. Use `@model_validator(mode="after")`.
> 3. `app/db/questions_repo.py` — a `QuestionsRepository` class around a Motor collection with `insert`, `get`, `list_by_tag`, `replace`, `delete`. ULID `_id`, `created_at`/`updated_at`, `schema_version: 1`.
> 4. `app/routes/questions.py` — full CRUD with proper status codes (201 create, 204 delete, 404 on missing), `response_model=Question`, `Annotated[..., Depends(get_questions_repo)]`.
> 5. `tests/unit/test_question_model.py` — `@pytest.mark.parametrize` tests for every validator with `ids=[...]`, both valid and invalid cases, asserting message fragments and `loc`.
>
> Constraints: Python 3.11, Pydantic v2, Motor (not pymongo), no global state, no `print`. Match the import and naming style in the existing `app/main.py`.
>
> Please ask before generating if anything is unclear. Then produce the files one at a time, and pause for me to check before moving on.

Three things this prompt does deliberately:

- **It points Claude Code at the existing files** (the `Read app/models/...` line). The substrate's conventions are the strongest signal for "what good looks like here."
- **It enumerates each file with constraints**, so the output is reviewable in chunks instead of a 1000-line wall.
- **It asks Claude Code to pause** between files. That's the verify checkpoint built into the prompt itself.

## What "verify" looks like in practice

After Claude Code produces `app/models/question.py`:

```bash
# Run the new tests Claude is about to write — but first, write one yourself
cd services/question-management-service
pytest tests/unit/test_question_model.py -v
```

Pick two or three rows of the parametrize sets you can reason about from first principles. Trace what happens manually:

- "If `correct_indices=[]` on a single-select, which validator fires? `exactly_one_correct`. Does the test assert that message fragment? Yes. Does `loc` include `single_select`?  Yes. OK, this row is correct."
- "What if Claude regressed and the multi-select validator allows zero correct? The `at_least_two_correct` test with `correct_indices=[]` should fail. Let me sabotage the validator temporarily — comment it out, rerun, confirm the test now fails. Restore. Good — the test actually defends the rule."

This *mutation testing by hand* is the single best way to catch tests that pass for the wrong reason — including tests Claude Code generated by pattern-matching without understanding.

## What "edit" looks like

You won't keep Claude's code byte-for-byte. Things you'll likely change today, even if they "work":

- **Error messages** — Claude's "Validation failed" becomes "single_select requires exactly 1 correct option, got 2" because the frontend (Day 9) needs to show the user what's wrong.
- **Imports** — Claude often imports things multiple ways; consolidate to match `app/main.py`.
- **Variable names** — `q` is fine inside a comprehension, but a function parameter named `q` instead of `question` is worse on second read.
- **Comments** — delete tutorial-style comments that explain what Pydantic is; keep comments that explain *why* a specific choice was made.
- **Sequencing** — Claude sometimes puts the discriminator union *after* the variants in a way that works but reads oddly. Reorder for clarity.

You don't edit for editing's sake. You edit because reading the code reveals choices you'd make differently, and a PR that's `<your name>` should reflect *your* choices.

## What "defend" looks like

Tomorrow a peer reviewer will leave comments. The Day 4 file on *defending AI-generated changes in peer review* is the protocol — go read it again before your PR. The short version:

- **"Why a discriminated union and not `Union[A, B]`?"** Because Pydantic's error messages and OpenAPI's `oneOf` mapping both require the discriminator to be wired explicitly. (You wrote this in Topic 1; rehearse it.)
- **"Why ULID and not ObjectId?"** Because ULIDs sort by time, are URL-safe, and don't need the `$oid` wrapper. (Topic 3.)
- **"Why path-style addressing in boto3?"** Because MinIO doesn't do virtual-hosted-style. (Topic 5.)
- **"Why this regex on option IDs?"** Because option IDs are exposed in URLs and frontend keys; the slug pattern prevents `../` and Unicode pitfalls.

If the answer to any of these is "Claude generated it that way," the right response is to *go back and understand it*, not to merge. The reviewer is doing their job; you do yours.

## A note on what Claude Code is bad at

Be honest about the failure modes you'll see today:

- **Fabricated APIs.** Claude might confidently call `pydantic.discriminated_union(...)` — which doesn't exist. The fix is `Annotated[..., Field(discriminator="type")]`. Always cross-check unfamiliar API names against Pydantic v2 docs.
- **Mixing Pydantic v1 and v2 idioms.** `@validator` (v1) instead of `@field_validator` (v2). `Config` inner classes instead of `model_config`. If you see this, regenerate with "Pydantic v2 only" or hand-correct.
- **Tests that pass for the wrong reason.** A test that asserts `raises(Exception)` will pass for any exception, including an unrelated import error. Make assertions specific.
- **Over-engineering.** Claude can produce factory patterns, abstract base classes, and dependency injection containers when a 30-line module would do. If the code grows abstractions that don't pay rent, push back: "Simplify; we don't need an abstract repository, just a class."

Recognising these failure modes is what separates "using AI" from "shipping AI-generated bugs."

## Example / Worked Scenario

You run the prompt above. Claude generates `app/models/question.py` in 90 seconds. You:

1. Open the file. Read it line by line. Notice it used `@validator` (v1 style) for one of the rules.
2. Ask Claude Code: "You used `@validator` on `option_ids_unique` — that's Pydantic v1. Rewrite as `@model_validator(mode="after")` for v2."
3. Run a test you write yourself: `Question.model_validate({"type": "single_select", "stem": "x", "options": [{"id": "a", "text": "1", "is_correct": True}, {"id": "b", "text": "2", "is_correct": False}]})`. It returns a `SingleSelectQuestion`. Good.
4. Sabotage the validator (delete the `at_least_two_correct` body, replace with `return self`), run Claude's tests, confirm one fails, restore.
5. Edit the validator's error message from "Invalid count" to "multi_select requires at least 2 correct options, got {n}" so the frontend can show it.
6. Stage, commit, push, open PR. In the PR description (which you also drafted with Claude per Day 4, then edited), call out the three non-obvious decisions: discriminated union, ULID `_id`, embedded options.
7. Tomorrow's reviewer asks "why embedded options instead of a separate collection?" You answer in one paragraph from your own understanding (Topic 3). The reviewer approves.

That's the loop. Generation is the cheap part; verify-edit-defend is the work.

## Common Pitfalls

- **Treating Claude's output as the final answer.** It's the *first* answer. Editing is mandatory.
- **Generating, glancing, committing.** The most common failure mode. Force yourself to read every line before staging.
- **Asking Claude vague questions like "build the question service."** You get vague, generic code. Specific prompts get specific code.
- **Skipping the test-the-test step.** A green test suite doesn't prove the tests test the right thing — sabotage and rerun is the cheap insurance.
- **Defending choices with "Claude said so" in PR comments.** Reviewers will reject. The choice is yours to defend, by name.
- **Letting Claude introduce new dependencies silently.** "Claude added `email-validator` to requirements.txt for one model" — but the substrate didn't have it, and you added a new dep without discussion. Read the diff in `requirements.txt` as carefully as the code diff.

## Key Takeaways

- The loop is read - run - edit - defend. Skip any step and you're not actually using AI-assisted generation; you're hoping.
- Specific prompts beat vague ones. Point Claude Code at existing files, enumerate the deliverables, set the constraints, ask it to pause between files.
- Verify by writing one test yourself and by sabotaging code to confirm Claude's tests catch the regression.
- Always edit — at minimum, error messages and naming. Code that ships as "Claude-shaped" tells the reviewer you didn't read it.
- Defend every non-trivial choice in your own words. "Claude wrote it that way" is not a defense; it's an admission. The Day 4 *defending AI-generated changes in peer review* file is the protocol — re-read it before your PR.
- Recognise the failure modes: fabricated APIs, v1/v2 mixing, tests that pass for the wrong reason, over-engineering. Catching these is the senior-developer skill the program is teaching you.

---
*Prerequisites: All of Day 8 (Topics 1-6 are what you're generating today), Day 1 (Claude Code agent fundamentals), Day 2 (AI-assisted brownfield comprehension), Day 3 (hypothesis-driven debugging with AI agents), Day 4 (defending AI-generated changes in peer review — re-read today).*
