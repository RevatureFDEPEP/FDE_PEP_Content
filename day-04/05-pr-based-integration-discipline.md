# PR-Based Integration Discipline

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Branch protection is the mechanism. PR-based integration discipline is the **culture** that makes the mechanism feel natural rather than bureaucratic. Today is when the cohort flips from "code in a sandbox" mode to "code as a team" mode — and the evening async peer review rotation that starts after standup tomorrow assumes this discipline is in place.

The rule is simple: **every change reaches `main` through a PR, and every PR has at least one peer reviewer who is not the author.** No exceptions. No "this is a tiny fix." No "I'll back-fill the PR later."

## Why Every Change, Even Tiny
The seeded bugs you're fixing today look like one-liners. The temptation to commit-and-push directly is real. The discipline of going through a PR anyway exists because:

1. **Visibility.** Your teammates see what's changing in `main` even when they're not the reviewer. PRs are the activity feed of the codebase.
2. **Auditability.** "Why did this change?" has a single answer: read the PR. Direct commits answer with the much weaker "read the commit body, if there is one."
3. **Habit transfer.** Reverting to direct-push for "obvious" changes is how production incidents start. The muscle is binary — either you always PR or you sometimes don't.
4. **Review surface.** Even a one-line change can be wrong. A second pair of eyes catches the typo in the env var name that turned the seeded bug into a 4-hour debug session in the first place.

In an enterprise FDE role, PR-based delivery is **table stakes**. Revature client teams uniformly expect candidates to operate this way on Day 1. Your evening peer reviews are deliberately scaffolded to build that muscle.

## The Rotation (Starting End of Today)
Each evening, the trainer rotates review assignments. Default scheme for the 25-person cohort:
- Each candidate is assigned 2 peer PRs to review by 8pm local
- Each PR needs 1 approval before the next-morning standup
- The trainer is a backstop reviewer, not the primary

The rotation forces you to read code you didn't write — half the value of this exercise. By Week 2 you'll be reading question-authoring code under time pressure; today is the warm-up.

## What a Reviewable PR Looks Like
The reviewer's job is to evaluate the change. Make their job easy:

**Title** — mirrors the commit subject in conventional-commit form:
```
fix(ci): correct YAML indentation in test job
```

**Description** — three sections, always:
```markdown
## What
Realigns the `steps:` block under the `test:` job from 2-space to
4-space indentation, matching the `build:` job structure.

## Why
GitHub Actions silently skipped the test step because the
mis-indented `steps:` was parsed as a sibling of `jobs:` rather than
a child of `test:`. Confirmed by inspecting the run summary, which
showed the test job completing in 0s with no steps executed.

## How to verify
- CI run on this branch: <link to Actions run>
- The `test` job should now report N steps and exit successfully
  (after dependent bug fixes in PRs #15 and #16 land).
```

**Scope** — one logical change. If your PR has files unrelated to the stated fix, split it.

## What Not to Do
- **"DNM" / "draft" PRs that stay open for days.** Open it when it's ready, mark it draft only if you actively need feedback on a work-in-progress.
- **Self-approval.** GitHub permits an author to also comment on their own PR, but the *approving review* must come from someone else. Branch protection enforces this.
- **Merging your own PR after approval without reading the final diff.** A clean approval doesn't mean you skip the last sanity check before clicking merge.
- **Ghosting reviews.** Your assignment is part of the rotation; not reviewing breaks the cohort's flow. If you can't get to it, say so by 6pm so the trainer can reassign.

## Example / Worked Scenario
End of Day 4. You've authored 2 PRs for CI fixes and been assigned 2 PRs to review.

**As author (PR #14, pin checkout action):**
- Title: `ci: pin actions/checkout to v4`
- Description: what/why/verify, links to Actions run, references hypothesis #2 from your Day 3 deliverable
- Status: 1 review requested (assigned by trainer), CI red because of dependent bugs
- You add a comment: "CI is red on this branch but the failure is in the `test` job from bug #4, which is fixed in PR #15. This PR is ready for review."

**As reviewer (PR #16, fix YAML indentation):**
- You open the Files Changed tab
- You read the change — verify the indentation actually matches the `build:` block
- You read the description — does the "why" match what the diff does?
- You leave one comment ("nit: the trailing whitespace on line 23 could be removed in a follow-up — not blocking") and click Approve
- Time spent: ~10 minutes

That's the rhythm.

## Common Pitfalls
- **Treating PR descriptions as optional.** A PR with no description is a PR that won't get reviewed promptly. Reviewers triage by description quality.
- **Reviewing only the diff, not the description.** The diff tells you *what* changed; the description tells you whether the *what* matches the stated *why*. Both matter.
- **Sitting on a review because you have "nothing constructive to say."** Approve with a quick "LGTM, verified the indentation matches the build job structure." Silent slots block the rotation.
- **Pushing fixup commits in response to comments instead of replying.** Reply first, then push. Otherwise the reviewer doesn't know whether you agreed or just patched something.

## Key Takeaways
- Every change to `main` goes through a PR, period.
- Every PR has at least one peer approver who isn't the author.
- A reviewable PR has a What/Why/Verify description and a single logical scope.
- Evening peer reviews are part of the work, not extra credit.

---
*Prerequisites: [01-git-workflows-feature-branches-pr-discipline-commit-hygiene.md](01-git-workflows-feature-branches-pr-discipline-commit-hygiene.md), [03-branch-protection-and-required-status-checks.md](03-branch-protection-and-required-status-checks.md).*
