# AI-Assisted PR and ADR Drafting — Structured Templates, Agent as Drafting Partner

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*
> *Unit 0: AI Tooling Thread*

## Overview
You're authoring 5 PRs today, and on top of that the trainer will ask you to write at least one short ADR (Architecture Decision Record) capturing a non-obvious choice — e.g., "why we pinned `actions/checkout@v4` instead of pinning to a SHA." Writing the PR description and the ADR by hand from a blank page is slow. Claude Code is excellent at producing a structured first draft from a diff, freeing your time for the parts that actually need your judgment: the *why*, the trade-offs, and the voice.

The rule, repeated for emphasis: **draft with the agent, own the final content.** Tomorrow's topic on defending these changes in peer review depends on you actually understanding what you submitted.

## The Drafting Pipeline
The pattern is the same for PR descriptions and ADRs:

```
   [your fix on a branch] --> [agent reads the diff + your intent]
                              --> [structured first draft]
                              --> [you revise: voice, accuracy, trade-offs]
                              --> [submit]
```

The agent is fast at the *structure*. You're fast at the *judgment*. Each does what it's good at.

## PR Description — Drafting Prompt
A repeatable prompt for Claude Code, given a branch with one or more commits:

```
You are drafting a pull request description for me. I'll give you the
diff and the intent. Produce a PR description in this exact format:

## What
<one-paragraph factual summary of what the diff changes, no opinions>

## Why
<one-paragraph explanation tying the change to a root cause; reference
any hypothesis from yesterday's diagnosis if I name one>

## How to verify
<bulleted checklist a reviewer can follow to confirm the change works;
include the link placeholder "<Actions run>" for me to fill in>

## Trade-offs considered
<one or two bullets covering alternatives I considered and why I didn't
pick them; if I haven't told you about alternatives, write "None
articulated — author to fill in" so I don't forget>

Constraints:
- Imperative present tense in subject; past/present indicative in body.
- No marketing language. No "we" or "I" — describe the change.
- Under 250 words total.

Here is the diff:
<diff>

Here is my intent in 1-2 sentences:
<intent>
```

Then you paste the diff (`git diff main...HEAD`) and one sentence of intent.

The output is a draft. Read it line by line. Check:
- Is the "What" accurate to the diff?
- Does the "Why" match the actual root cause, not a plausible-sounding-but-wrong one?
- Did the agent hallucinate a trade-off you didn't make? Strike it.
- Is the verification list something a reviewer can actually do?

Revise. Submit.

## ADR Drafting — Prompt
ADRs are short (one page or less). The trainer uses the [MADR](https://adr.github.io/madr/) lite format:

```
# ADR-NNN: <Title>

## Status
Accepted | Proposed | Superseded by ADR-XYZ

## Context
What was the situation that prompted this decision?

## Decision
What did we decide?

## Consequences
What follows from the decision — positive and negative?
```

Drafting prompt:

```
Draft an ADR using the MADR-lite format with sections Status, Context,
Decision, Consequences. Title it "ADR-NNN: <short imperative>".

The decision being recorded: <one sentence>

Background facts (use only these, do not invent context):
- <fact 1>
- <fact 2>
- <fact 3>

Alternatives I considered and rejected:
- <alternative 1>: <why rejected>
- <alternative 2>: <why rejected>

Keep it under 300 words. Use bullet points in Consequences. Mark the
Status as "Accepted" unless I told you otherwise.
```

The prompt is intentionally restrictive — the agent only uses facts you provide. This prevents the common failure mode where an agent fills in plausible context that doesn't match your actual situation.

## Owning the Final Content
Tomorrow you might be in peer review defending why you pinned to `@v4` instead of a SHA. If the agent wrote that paragraph and you skimmed it, you'll stumble. The fix:

After accepting the draft, **read it as if you were the reviewer**. For each sentence, ask: *would I be able to defend this if challenged?* If not, either learn the answer or strike the sentence.

This is non-negotiable. The next topic (`day-4-defending-ai-generated-changes-in-peer-review.md`) is the muscle for doing exactly this in real-time. Today's habit prevents tomorrow's embarrassment.

## Example / Worked Scenario
You're authoring the PR description for PR #14 (pin checkout action).

**Step 1.** You stage the diff and write your intent:
> "Pinning actions/checkout from @main to @v4 because @main floated and broke when upstream shipped a breaking change yesterday. Considered pinning to a full commit SHA but rejected because the team's stated convention is major-version pins."

**Step 2.** You invoke Claude Code with the prompt above. Output:

```markdown
## What
Pins the `actions/checkout` action in `.github/workflows/ci-pipeline.yml`
from the floating `@main` ref to the fixed `@v4` major-version tag.

## Why
The workflow used `actions/checkout@main`, which began failing yesterday
after upstream shipped a breaking change to the action's input handling.
Floating refs offer no guarantee of compatibility. Pinning to `@v4`
restores reproducibility under the repo's stated pinning convention
(hypothesis #2 from the Day 3 diagnosis).

## How to verify
- CI run against this branch: <Actions run>
- The `build` job's checkout step should report `actions/checkout@v4`
  in its logs.
- Failure mode reproduces if `@main` is restored locally.

## Trade-offs considered
- Pinning to a full commit SHA was rejected because the team convention
  is major-version pins; SHA pins would inconsistent with the rest of
  the workflow.
- Pinning to a patch version (`@v4.1.1`) was rejected as over-specified
  for a stable, well-versioned official action.
```

**Step 3.** You read it as the reviewer. The "Trade-offs considered" second bullet — *did you actually consider patch-pinning, or did the agent infer it?* You didn't. You strike it:

```markdown
## Trade-offs considered
- Pinning to a full commit SHA was rejected because the team convention
  is major-version pins; SHA pins would be inconsistent with the rest
  of the workflow.
```

Note also: "would inconsistent" was an agent typo, fixed to "would be inconsistent". Always read for surface errors too.

**Step 4.** You fill in the Actions run link, paste the description into `gh pr create`, and submit.

Total time: ~3 minutes. Without the agent: ~10. The 7 minutes you saved go into reading the next teammate's PR more carefully.

## Common Pitfalls
- **Submitting the draft unchanged.** "It looked right" is not ownership. Read line by line.
- **Letting the agent invent facts.** Restrict the prompt to facts you supply. If the output contains a claim you didn't tell it, strike or verify.
- **Using the agent for the "Why" without thinking through the why yourself.** The agent will produce a plausible *why*. Your job is to make sure it's the *actual* why.
- **One-shot prompting.** If the draft is off, refine the prompt or ask follow-up turns. Don't manually rewrite something the agent could fix in one more iteration.

## Key Takeaways
- The agent drafts the structure; you own the judgment.
- Restrict prompts to facts you supply; mark unknowns explicitly.
- Read every sentence as if you'll defend it tomorrow — because you will.
- 3 minutes with the agent beats 10 by hand, but only if you actually revise.

---
*Prerequisites: `day-1-claude-code-agentic-tooling.md`, `day-4-git-workflows-feature-branches-pr-discipline-commit-hygiene.md`.*
