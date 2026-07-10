# Git Workflows — Feature Branches, PR Discipline, Commit Hygiene, Conventional Commits

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Yesterday you produced a list of 5 root-cause hypotheses for the seeded bugs in `ci-pipeline.yml`. Today you turn those hypotheses into merged fixes. That conversion happens through a specific delivery vehicle: a feature branch, a tight commit history, and a peer-reviewed pull request. The discipline you build here is the substrate every later day assumes — by Day 5 you'll be opening PRs without thinking about the mechanics.

This topic covers the four habits that make PR-based delivery sustainable:
1. One branch per logical change
2. Conventional commit messages
3. Small, reviewable commits
4. PRs scoped to a single concern

## Feature Branch Naming
Each CI bug fix gets its own branch. The trainer repo follows the `<type>/<short-slug>` convention:

```bash
# One branch per bug — not one branch for "all CI fixes"
git checkout main
git pull --ff-only origin main
git checkout -b fix/ci-pin-checkout-action
```

Common prefixes used in this course:
- `fix/` — bug fixes (most CI work today)
- `feat/` — new functionality (W2+)
- `chore/` — tooling, config, no behavior change
- `docs/` — documentation only
- `refactor/` — restructure without behavior change

Slugs are kebab-case and specific: `fix/ci-pin-checkout-action` not `fix/ci` or `fix/stuff`.

## Conventional Commits
[Conventional Commits](https://www.conventionalcommits.org) gives commit messages structure that humans and tools both read. Format:

```
<type>(<scope>): <subject>

<body — optional, wrapped at 72 chars>

<footer — optional, e.g. "Closes #42">
```

Types match the branch prefixes above (`fix`, `feat`, `chore`, `docs`, `refactor`, `test`, `ci`).

Examples that fit today's seeded bugs:

```
ci: pin actions/checkout to v4

The workflow used actions/checkout@main, which broke when upstream
shipped a breaking change to input handling. Pinning to v4 restores
reproducibility per the repo pinning policy.
```

```
fix(ci): correct YAML indentation in test job

The `steps:` block under `test:` was indented two spaces instead of
four, causing GitHub Actions to silently skip the test step. Realigned
to match the build job structure.
```

```
ci: add NPM_TOKEN to test job env

Private package install was failing because NPM_TOKEN was only exposed
to the build job. Added it to the test job env so the install step
can authenticate.
```

Subject line rules: imperative mood ("add", not "added"), under 72 chars, no trailing period.

## Commit Hygiene — Small, Reviewable Units
A good commit is **one logical change a reviewer can understand in 30 seconds**. The reviewer should not have to ask "what does this commit do?"

Bad:
```
fix stuff and also bump node version and reformat
```

Good — three separate commits:
```
ci: pin node-version to 20.11.1 in setup-node
fix(ci): correct YAML indentation in test job
chore: reformat ci-pipeline.yml with prettier
```

If you find yourself writing "and" in a commit subject, split the commit. Use `git add -p` to stage hunks selectively when you've made multiple unrelated changes in one editing session.

## Scoping the PR
One PR = one concern. For today's deliverable that means **five PRs, not one mega-PR**. Each PR:
- Fixes one root-cause hypothesis from yesterday
- Has a title that mirrors the commit subject (`fix(ci): correct YAML indentation in test job`)
- Has a description explaining *why* (the hypothesis) and *what* (the change), with a link to the failing CI run

Smaller PRs are reviewed faster, merged faster, and revert cleanly if they break something downstream.

## Example / Worked Scenario
You're fixing seeded bug #2: the workflow pinned `actions/checkout@main`, which started failing after an upstream change.

```bash
# 1. Start from a clean main
git checkout main
git pull --ff-only origin main

# 2. Branch
git checkout -b fix/ci-pin-checkout-action

# 3. Edit .github/workflows/ci-pipeline.yml — change @main to @v4

# 4. Stage and commit
git add .github/workflows/ci-pipeline.yml
git commit -m "ci: pin actions/checkout to v4

The workflow used actions/checkout@main, which broke when upstream
shipped a breaking change. Pinning to v4 restores reproducibility."

# 5. Push and open PR
git push -u origin fix/ci-pin-checkout-action
gh pr create --title "ci: pin actions/checkout to v4" \
  --body "Fixes hypothesis #2 from yesterday's diagnosis..."
```

CI runs against the PR. If green for this fix in isolation, you move to bug #3 on a new branch. If red, you iterate on the same branch.

## Common Pitfalls
- **Stacking unrelated work on one branch.** "While I was in there I also fixed X" — now the PR can't be reviewed cleanly. Branch off main again for X.
- **Force-pushing main or shared branches.** Force-push only your own feature branches, and only when the PR isn't actively being reviewed.
- **Vague commit subjects.** `fix bug`, `update ci`, `wip` — reviewers can't scan the log. Be specific.
- **Branching off a stale main.** Always `git pull --ff-only origin main` before creating a branch, or you'll inherit conflicts from work merged after your last sync.

## Key Takeaways
- One logical change per branch, one branch per PR.
- Conventional commits (`type(scope): subject`) make history scannable.
- Commit subjects are imperative, specific, under 72 chars.
- If you write "and" in a subject, split the commit.
- Five seeded bugs means five PRs today, not one.

---
*Prerequisites: [01-github-actions-structure-workflows-jobs-steps-runners.md](../day-03/01-github-actions-structure-workflows-jobs-steps-runners.md), [06-hypothesis-driven-debugging-with-ai-agents.md](../day-03/06-hypothesis-driven-debugging-with-ai-agents.md).*
