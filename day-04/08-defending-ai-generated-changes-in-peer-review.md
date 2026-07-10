# Defending AI-Generated Changes in Peer Review — Owning the Output

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*
> *Unit 0: AI Tooling Thread*

## Overview
Yesterday's topic on AI-assisted drafting ended with a promise: tomorrow you defend it. Tomorrow is today. The peer review rotation starts at end of day, and your reviewer will ask questions about your fix that the agent's PR description glossed over. "Why pin to `@v4` instead of a SHA?" "Why fix the indentation rather than add yamllint?" If the agent helped author the change, the agent is not in the room. **You are.**

This topic is the muscle for being the human in the loop when the loop is being scrutinized.

## The Standard
A change is yours when you can do all of the following, in your own words, without rereading the PR description:

1. **State what the change does** at the file/line level.
2. **State why this approach was chosen over at least one alternative.**
3. **State what would break if the change were wrong.**
4. **Point to the evidence that the change is correct** (test, log, manual check).

If you can't do any of those four, you didn't own the change — you forwarded the agent's output. Today's discipline is closing that gap *before* the review, not during it.

## The Pre-Review Self-Check
Before clicking "Ready for review," run this prompt against your own work. The five questions take 2-3 minutes:

```
For each non-trivial line in my diff, can I answer:
1. What does this line do?
2. Why is it phrased this way (not the obvious alternative)?
3. What test or check would catch it being wrong?
4. Where did this idea come from — my reasoning, the agent, or
   copied from elsewhere?
5. If a reviewer asks "why not X instead?" do I have an answer?
```

Question 4 is the one most people skip. The honest answer for an agent-drafted change is often "the agent suggested it and I didn't think hard about it." When that's the answer, go back and think hard about it *now*, before the reviewer makes you do it in real-time.

## A Prompt to Force the "Own It" Check
You can also have Claude Code stress-test the change for you before you submit. The prompt:

```
I'm about to submit this PR for peer review. I need to defend the
change in my own words. Read the diff and the description below, then
ask me the 5 hardest questions a reviewer might ask. Do not answer
them — just ask them. After I respond to each, tell me whether my
answer would satisfy a senior reviewer or whether I'm hand-waving.

Diff:
<git diff main...HEAD>

PR description:
<paste the description you drafted>
```

The agent generates questions like:
- "Why did you pin to `@v4` rather than to a specific SHA? What guarantees does `@v4` give you that `@main` didn't?"
- "If GitHub deprecates v4 mid-cohort, what's your rollback plan?"
- "The diff also changes whitespace on line 22. Was that intentional or a stray editor change?"

You answer each in plain language. The agent grades you. If you're hand-waving, you have a chance to study before the reviewer notices.

This inverts the normal use of the agent — instead of generating output, it's grilling you on output. Same tool, different posture.

## What "Defend" Doesn't Mean
Defend does not mean "refuse to change the code." A good defense often ends with "you're right, I'll update it." The point of the defense is that the change reflects *your* understanding, not the agent's, so the conversation with the reviewer is grounded in real reasoning rather than recital.

Compare:

**Recital (bad):**
> Reviewer: "Why not pin to a SHA?"
> Author: "Uh… because the team convention is major-version pins. That's what the PR description says."

**Defense (good):**
> Reviewer: "Why not pin to a SHA?"
> Author: "Two reasons. One, the rest of the workflow uses major-version pins, so a SHA here would be inconsistent — we'd lose the regularity that lets us scan the file. Two, `actions/checkout` is a first-party action that GitHub actively maintains; the risk of an unannounced breaking change inside v4.x is low enough that I traded reproducibility-by-SHA for readability-by-tag. If the team wants to standardize on SHA pins, I'd be on board, but that's a separate PR."

The second answer reveals reasoning that survives the question. The first reveals that the PR description was the only source of truth, and the author hasn't internalized it.

## When the Reviewer Catches You Out
It happens. The fix is the same in either direction:

1. **Acknowledge specifically.** "You're right — I didn't think about the case where the test job runs in parallel."
2. **Don't blame the agent.** "Claude suggested it" is not a defense. You submitted it.
3. **Patch the gap and re-push.** Add the fix in a new commit (or amend if pre-review).
4. **Update the PR description.** If the original "Why" is now wrong, fix it.

The cohort will respect the second outcome — caught, acknowledged, fixed — far more than a defensive non-response. Engineers who can say "I missed that" without breaking stride are the engineers who scale.

## Example / Worked Scenario
You used Claude Code yesterday to draft the PR description for PR #16 (fix YAML indentation). Before submitting, you run the self-check prompt:

```
You: <prompt + diff + description>
Claude: Here are 5 hard questions a reviewer might ask:
  1. The diff fixes indentation in the test job but not the build job
     — are you sure the build job is correctly indented?
  2. YAML supports both 2-space and 4-space; why is 4-space the
     correct choice here?
  3. What test or check confirms the test job actually runs now, vs.
     just being syntactically parseable?
  4. Did you check whether other workflows in the repo have the same
     drift? If yes, why isn't this PR fixing them too?
  5. The PR description references "GitHub Actions silently skipped"
     — where in the run log is that visible to someone verifying?
```

You answer:
- Q1: "Yes — build job was 4-space; that's why I aligned to it."
- Q2: "Repo convention is 4-space per `.editorconfig`."
- Q3: "Locally re-ran with `act`; remote CI run linked in description."
- Q4: "I only checked `ci-pipeline.yml`. Will scan others as a follow-up issue rather than expanding this PR."
- Q5: "The run summary showed `0 steps executed` for the test job — link added to description."

Claude grades you:
> Q1, Q2, Q3 strong. Q4 acceptable as long as you actually file the follow-up before merging. Q5 was implicit in your draft but the link wasn't there — recommend adding it before submitting.

You add the missing link, file the follow-up issue, and submit. When the reviewer asks Q3 the next morning, you answer in 5 seconds.

## Common Pitfalls
- **Reading "AI-assisted" as "AI-responsible."** The agent is a tool. You sign the commit, you defend the commit.
- **Memorizing the PR description as the defense.** Internalize the reasoning instead. Reviewers ask questions the description doesn't answer.
- **Defensive posture.** If the reviewer is right, agree and patch. The goal is shipping a correct change, not winning the conversation.
- **Skipping the self-check on "small" changes.** The seeded bugs are one-line fixes. The reasoning behind each is *not* one line.

## Key Takeaways
- A change is yours when you can defend the *what*, the *why*, the *failure mode*, and the *evidence* in your own words.
- Run a pre-review self-check with the agent grilling you, not generating for you.
- Don't blame the agent in review; the commit is signed by you.
- A caught miss + clean fix is a positive signal, not a negative one.

---
*Prerequisites: [02-ai-assisted-pr-and-adr-drafting.md](02-ai-assisted-pr-and-adr-drafting.md), [06-code-review-etiquette.md](06-code-review-etiquette.md), [05-pr-based-integration-discipline.md](05-pr-based-integration-discipline.md).*
