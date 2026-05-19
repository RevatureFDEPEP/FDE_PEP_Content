# Code Review Etiquette (Giving and Receiving)

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
The peer-review rotation starts at the end of today. Tomorrow morning you'll walk into standup with reviews assigned, given, and received. The mechanics (branch protection, PR-based delivery) are in place — this topic is about the *behavior* on either side of a review. The goal: useful, kind, technically sharp reviews that ship code faster than they would have without the review, not slower.

Code review is the single most leveraged habit in a brownfield environment. You will spend more time reading code than writing it for the rest of your career.

## The Two Roles

### Giving a Review
Your goal is to make the PR **better and merged**, in that order. Not to demonstrate that you spotted something. Not to gatekeep.

Three questions to ask in order:
1. **Does the change do what its description says it does?** If not, that's the comment — "the description says X but I see Y."
2. **Will this break something?** Look at the surrounding context, not just the diff. A YAML indentation fix on line 30 might shift line 50 in ways the diff doesn't highlight.
3. **Is it readable for the next person?** Naming, comments, structure. Polish, not correctness.

### Receiving a Review
Your goal is to **absorb the review without ego**. The reviewer is helping you ship; they are not auditing your worth.

Three habits:
1. **Reply to every comment.** Even "good catch, fixed in next push." Silence reads as ignoring.
2. **Push back when you disagree, with reasoning.** "I considered that — went with X because Y. Open to Z if you feel strongly."
3. **Update the PR description if the scope shifts.** If review comments reveal you misunderstood the bug, edit the description; don't leave it stale.

## Useful Review Comment Patterns
Borrow these phrasings. They lower defensiveness and make the comment actionable.

| Bad | Good |
|-----|------|
| "This is wrong." | "I think this misses the case where the env var is unset — would `process.env.NPM_TOKEN ?? ''` work better?" |
| "Why did you do it this way?" | "Curious about this choice — was there a reason not to use the existing `withRetry` helper?" |
| "Reformat this." | "nit: could you align this with the indentation style in the build job? Not blocking." |
| "LGTM" with no read of the change | "Verified the indentation now matches the build job and re-ran the workflow locally with `act`. LGTM." |

**Prefix conventions** used in this course:
- `nit:` — minor, non-blocking style point
- `question:` — asking for context, not requesting a change
- `suggestion:` — concrete alternative the author can take or leave
- `blocking:` — must be resolved before merge

Use prefixes; they tell the author what response shape you expect.

## Tone — Specific Phrases
Read out loud. If it sounds like you're talking to a teammate at lunch, it's right. If it sounds like a teacher grading a test, rephrase.

**Soft-but-clear disagreement:**
> "I'd push back on this — pinning to `@v4` rather than a SHA still allows v4.x.y bumps. That might be fine here, but worth a line in the PR description on the trade-off."

**Praise (yes, leave these):**
> "Nice — the description here makes it obvious why this was the right fix vs. just bumping the action version."

**Asking for changes without sounding harsh:**
> "Could we get a quick test or at least a manual verification note in the description? Right now I can't tell whether this was confirmed end-to-end."

## Receiving — Specific Phrases
**Agreeing and fixing:**
> "Good point, didn't consider the unset case. Pushed a fix in `c4d2f8a` — added a default."

**Disagreeing with reasoning:**
> "I considered using `withRetry`, but it adds a 2s backoff that would make CI slower on every run. The failure mode here is non-flaky, so I think a single attempt is correct. Happy to revisit if you've seen this be flaky."

**Asking for clarification:**
> "Not sure I follow — do you mean the indentation under `steps:` specifically, or the whole `test:` block? Want to make sure I fix the right thing."

## Example / Worked Scenario
Excerpt from a peer review on PR #16 (`fix(ci): correct YAML indentation in test job`):

**Reviewer comment on line 32:**
```
question: any idea why the original was 2-space? Wondering if it was
copy-pasted from a tab-indented source vs. a deliberate choice. Not
blocking — just want to understand if we should add yamllint to
prevent regression.
```

**Author reply:**
```
Good question — checked git blame, the indentation drift happened in
PR #8 from last week (commit a3f2c1d), which looks like an editor
auto-indent issue. I think adding yamllint is a great follow-up but
out of scope here. Filed as issue #22 so we don't lose it.
```

**Reviewer follow-up:**
```
Perfect, thanks. Approving — and following the yamllint issue.
```

Notice: the conversation produced a follow-up issue, not scope creep. The PR stayed focused. The reviewer learned something. The author didn't get defensive about an old commit. That's the loop working.

## Common Pitfalls
- **Reviewing for taste, not correctness.** "I would have named this differently" without a specific reason wastes the author's time. Save taste comments for the times the naming actively misleads.
- **Bikeshedding.** Long thread about a one-line style point on a 200-line PR. If it's not blocking, prefix `nit:` and move on.
- **Approve-bombing.** Clicking Approve in 30 seconds without reading the diff. Reviewers who do this are noticed. So are the bugs they let through.
- **Treating "Request changes" as personal.** Use "Request changes" only when there's a blocking issue. For non-blockers, comment without the status change.

## Key Takeaways
- Reviewer's job: make the PR better and merged.
- Author's job: absorb feedback without ego; reply to every comment.
- Prefix comments (`nit:`, `question:`, `suggestion:`, `blocking:`).
- Disagree with reasoning, not silence or capitulation.
- Praise good moves explicitly — it shapes the cohort culture.

---
*Prerequisites: `day-4-pr-based-integration-discipline.md`.*
