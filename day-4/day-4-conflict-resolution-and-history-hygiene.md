# Conflict Resolution and History Hygiene

> *Day 4: Fix CI & Git Discipline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
With 25 candidates landing fixes against the same `ci-pipeline.yml`, conflicts are not hypothetical — they're inevitable today. The interesting question isn't "did you avoid a conflict" but "did you resolve it cleanly and ship a readable history afterward." This topic gives you a working opinion on rebase vs merge, the muscle to resolve a conflict in a YAML file without breaking it, and the habits that keep the log scannable for the next teammate.

## The Inevitable Conflict Today
Multiple candidates will touch `.github/workflows/ci-pipeline.yml` in overlapping regions:
- One PR pins `actions/checkout` (line ~10)
- Another fixes indentation in the `test:` block (lines ~30-45)
- Another adds `NPM_TOKEN` to env (lines ~25-28)

When PR A merges, PRs B and C suddenly need to rebase or merge in the new `main`. The trainer repo is configured with "Require branches to be up to date" — you cannot merge a stale branch.

## Rebase vs Merge — An Opinion
Both work. Both produce a valid result. They produce different shapes of history, which is the actual decision.

**`git merge main` into your feature branch:**
- Creates a merge commit on your branch
- Preserves the original timing of your commits
- Easy to undo (revert the merge commit)
- Log looks like a braid

**`git rebase main` onto your feature branch:**
- Replays your commits on top of the latest `main`
- Produces a linear history when the PR merges
- Loses original commit timestamps
- Requires force-push to update the PR (`git push --force-with-lease`)

**The opinion this course adopts:**
- **Rebase your feature branch onto `main` while it's still your private work.** A linear log per PR is cleaner to review.
- **Merge the PR into `main` using GitHub's "Squash and merge"** (the trainer repo default). Each PR becomes one commit on `main`, regardless of how many commits the PR had during review.
- **Never rebase a shared branch.** If two people are working on the same branch, only merge.

Result: `main`'s log reads like one commit per PR, each with a conventional-commit subject. Your feature branches are linear inside that.

## Resolving a YAML Conflict
YAML is whitespace-sensitive. Conflict markers in a YAML file are a special kind of dangerous because resolving them wrong produces invalid syntax that the editor won't always flag.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
<<<<<<< HEAD
      - uses: actions/checkout@v4
=======
      - uses: actions/checkout@main
        with:
          fetch-depth: 0
>>>>>>> fix/ci-checkout-depth
      - run: npm ci
```

Steps to resolve:
1. **Understand both sides.** `HEAD` (your branch) pinned to v4. The incoming side wants `fetch-depth: 0` for full history.
2. **Decide the intent.** Both changes are wanted — pin AND fetch-depth. Combine them.
3. **Edit by hand, preserving indentation.**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - run: npm ci
```

4. **Stage and continue:**
```bash
git add .github/workflows/ci-pipeline.yml
git rebase --continue
```

5. **Validate the YAML before pushing:**
```bash
# Quick parse check — adjust path as needed
python -c "import yaml; yaml.safe_load(open('.github/workflows/ci-pipeline.yml'))"
# Or use yamllint if installed
yamllint .github/workflows/ci-pipeline.yml
```

A red CI run after a conflict resolution usually means a wrong-indentation fix. The validation step catches it before you waste the round-trip.

## History Hygiene — The Clean Log
By the time your PR is ready to merge, the commits should tell a story. Two acceptable shapes:

**Single-commit PR** (preferred for small fixes — most of today):
```
ci: pin actions/checkout to v4
```

**Multi-commit PR** (for larger PRs):
```
fix(ci): correct YAML indentation in test job
test(ci): add regression case for indentation drift
docs(ci): note YAML indentation convention in CONTRIBUTING
```

What `main` should **never** see (because squash-merge collapses it, but your branch shouldn't have it either before review):
```
wip
fix
fix again
oops
forgot to save
addressing review comments
addressing review comments part 2
```

Use `git rebase -i` (or, in this course's tooling, "amend" via the GitHub UI / your editor's git tooling) to clean up before pushing. If you've already pushed, force-push with `--force-with-lease`:

```bash
# Combine the last 3 commits into 1 with a clean message
git rebase -i HEAD~3
# In the editor: change pick → squash (or fixup) for the trailing commits
# Save, edit the combined message
git push --force-with-lease
```

`--force-with-lease` is the safe variant — it refuses to push if someone else committed to the branch in the meantime.

## Example / Worked Scenario
Your PR #14 (pin checkout) was approved this afternoon. PR #15 (add NPM_TOKEN, by a teammate) merged first. Now your branch is stale and GitHub blocks the merge.

```bash
# 1. Sync main locally
git checkout main
git pull --ff-only origin main

# 2. Rebase your branch onto the new main
git checkout fix/ci-pin-checkout-action
git rebase main
# CONFLICT in .github/workflows/ci-pipeline.yml — env block overlaps

# 3. Open the file, resolve by combining both changes
# (Keep the pin AND the new NPM_TOKEN env entry)

# 4. Validate
python -c "import yaml; yaml.safe_load(open('.github/workflows/ci-pipeline.yml'))"

# 5. Continue the rebase
git add .github/workflows/ci-pipeline.yml
git rebase --continue

# 6. Force-push (safely)
git push --force-with-lease

# 7. CI re-runs against the rebased SHA; reviewer's approval was dismissed
#    by branch protection — ping them for a re-approve since the change
#    is mechanical
```

## Common Pitfalls
- **Resolving a YAML conflict by accepting one side wholesale.** You usually want both intents, recombined. Read both sides before clicking "accept current".
- **`git push --force` (no lease).** Overwrites a teammate's commits without warning. Always `--force-with-lease`.
- **Rebasing a branch others are working on.** Their local copies will diverge from the rewritten history. Communicate or merge instead.
- **Leaving `<<<<<<<` markers in the file.** Run `grep -n '<<<<<<<' .` (or your editor's conflict check) before staging. The CI run will catch it, but a search is faster.

## Key Takeaways
- Rebase your private feature branch onto `main`; squash-merge the PR.
- Never rewrite shared branch history.
- YAML conflicts need a parse-check after resolution.
- A clean log on `main` means one conventional-commit message per PR.
- `git push --force-with-lease`, never bare `--force`.

---
*Prerequisites: `day-4-git-workflows-feature-branches-pr-discipline-commit-hygiene.md`, `day-3-github-actions-structure-yaml-pitfalls.md`.*
