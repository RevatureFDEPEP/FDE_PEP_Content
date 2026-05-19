# Branch Protection and Required Status Checks

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
You can write the cleanest commit in the world, but if the repo doesn't enforce CI as a gate, a teammate can still push broken code directly to `main`. The trainer repo has **branch protection** configured precisely so that doesn't happen — and so that your day-4 PRs are forced to demonstrate the pipeline is green before they merge. Understanding what protection rules are doing is what separates "I followed the steps" from "I understand why this works."

## What Branch Protection Is
Branch protection is a GitHub-enforced policy on a specific branch (here, `main`) that overrides individual contributor permissions. Even a repo admin cannot push directly to a protected branch without explicitly bypassing the rules (an audited action).

The trainer repo's `main` branch has these rules enabled:

| Rule | Effect |
|------|--------|
| Require a pull request before merging | No direct pushes to `main` — even from admins |
| Require status checks to pass | The `build` and `test` jobs in `ci-pipeline.yml` must succeed on the PR head before the merge button enables |
| Require branches to be up to date | PR branch must contain the latest `main` before merge (forces rebase/merge from main if `main` advances during review) |
| Require at least 1 approving review | Another candidate must approve the PR before merge |
| Dismiss stale approvals on new commits | If you push a fix after approval, the approval clears — reviewer re-checks the change |
| Restrict who can push | Empty here — combined with "require PR", this means nobody pushes directly |

## Required Status Checks — The Mechanic
A "status check" is any commit status that GitHub Actions (or another CI provider) reports against a commit SHA. When you push to a PR branch, the workflow runs and reports `success` or `failure` for each job that has a name registered as a required check.

In this repo, two checks are required:
- `build` — the build job from `ci-pipeline.yml`
- `test` — the test job from `ci-pipeline.yml`

GitHub looks at the head SHA of the PR. If both checks report `success` against that SHA, the merge button enables. If either fails — or if you push a new commit that hasn't yet been evaluated — the merge button greys out until the new run completes.

This is why **fixing CI today is unblocking for the rest of the course**: until the pipeline is green, no PR can merge, no work flows, the course halts.

## Why This Matters for Day 4
You have 5 broken pipelines and 5 PRs to land. The interplay:

1. You open `fix/ci-pin-checkout-action`. CI runs — still red because of the other 4 bugs.
2. You can't merge. Branch protection is doing its job.
3. You stack the next fix branch off `main` and open a second PR. Still red.
4. Repeat for the remaining bugs. Each PR runs CI; until all 5 root causes are addressed, no PR can merge.

This forces you to either:
- **Sequence** — merge fixes one at a time, but the first 4 PRs can't merge until the 5th lands too (since each PR's CI run still hits all 5 bugs); or
- **Coordinate** — agree on a merge order with the trainer, possibly merging fixes through a temporary integration branch.

The trainer will guide which sequencing strategy fits today's cohort. The point: branch protection is what makes that conversation necessary, which is what makes the discipline real.

## Inspecting the Rules
You can see what's enforced from the GitHub UI (Settings → Branches → main) or via the API:

```bash
gh api repos/:owner/:repo/branches/main/protection \
  --jq '.required_status_checks.contexts, .required_pull_request_reviews.required_approving_review_count'
```

Output for the trainer repo:
```
[
  "build",
  "test"
]
1
```

## Example / Worked Scenario
You open PR #14 to pin `actions/checkout@v4`. The PR view shows:

```
Some checks were not successful
  X build  Failing after 2m
  X test  Skipped — build did not succeed
Merging is blocked
  Required statuses must pass before merging
```

You investigate — the build failure isn't your bug; it's seeded bug #4 (missing `NPM_TOKEN` in the test job env). Your pin fix is correct but can't be validated until bug #4 is also fixed. You note this in the PR description ("blocked on resolving bug #4 — see PR #15") and move on to author PR #15.

This kind of dependency between fixes is normal in brownfield CI work. Branch protection makes the dependency visible rather than letting a "looks fine to me" merge paper over it.

## Common Pitfalls
- **Treating a red CI as permission to skip review.** Branch protection blocks the merge button, not the review. Reviewers still leave comments on red PRs.
- **Force-pushing a PR branch after approval.** "Dismiss stale approvals" is on — your reviewer will have to re-approve. Avoid force-pushing once a PR is in active review unless rebasing for a specific reason.
- **Assuming admin overrides things.** The trainer (admin) can bypass protection in an emergency, but every bypass is logged and visible. Don't ask for a bypass to skip a legitimate failing check.

## Key Takeaways
- Branch protection enforces CI as a quality gate; no green pipeline, no merge.
- Required status checks (`build`, `test`) must succeed against the PR head SHA.
- Stale approvals dismiss on new commits, so reviewers always evaluate the final state.
- Today's 5 fixes interact through the gate — sequencing matters.

---
*Prerequisites: `day-3-github-actions-structure-yaml-pitfalls.md`, `day-4-git-workflows-feature-branches-pr-discipline-commit-hygiene.md`.*
