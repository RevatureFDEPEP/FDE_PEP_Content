# Actions Ecosystem and Version Pinning Discipline

> *Day 3: Diagnose the CI Pipeline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
`uses: actions/checkout@v3` and `uses: actions/checkout@v4` are not the same workflow. One of today's five seeded bugs is a version-drift bug: an action reference that resolves differently than the rest of the pipeline expects, producing a failure mode that looks unrelated to the action itself. This topic teaches you how the Actions ecosystem versions code, why pinning matters, and how to recognize a drift-induced failure when you see it in a log.

## The `uses:` reference: what it actually points to

Every `uses:` in a workflow points to code that runs on the runner. There are three reference styles:

```yaml
- uses: actions/checkout@v4              # major version tag (rolling)
- uses: actions/checkout@v4.1.7          # exact version tag
- uses: actions/checkout@b4ffde65f46336ab  # commit SHA (immutable)
- uses: actions/checkout@main            # branch — DON'T do this
```

What each reference means:

- **`@v4`** — a Git tag that the action maintainer **moves** as they release `v4.1.0`, `v4.1.7`, `v4.2.0`, etc. You always get the latest within major version 4. Backward-compatible by convention (semver), but the convention is not enforced.
- **`@v4.1.7`** — a specific release. Stable, but you have to bump it manually for fixes.
- **`@<full-SHA>`** — points at an exact commit. Immutable. Cannot be changed by the maintainer. Highest reproducibility, but loses readability.
- **`@main`** — points at whatever the maintainer pushed to `main` ten minutes ago. CI breaks the moment they merge a breaking change. **Avoid in production workflows.**

Most workflows use the major-version-tag form (`@v4`). The trade-off: you get patch fixes for free, but you implicitly trust the maintainer not to introduce breaking changes within a major version.

## How version drift bites

The rev-eval-ai-pep `ci-pipeline.yml` was authored against `actions/checkout@v3`. Over time, GitHub published `v4`, which changed default behavior in a way the workflow depended on. A well-meaning prior contributor "modernized" one of the seven `uses: actions/checkout@v3` references to `@v4` but missed the others.

Now the pipeline has:

```yaml
jobs:
  lint:
    steps:
      - uses: actions/checkout@v4        # newer
      - run: pnpm lint

  test-services:
    steps:
      - uses: actions/checkout@v3        # older
      - run: pytest

  build:
    steps:
      - uses: actions/checkout@v3        # older
      - run: docker build .
```

The pipeline runs. Most jobs pass. One specific job fails on something that *looks* unrelated — a submodule isn't initialized, or a tag isn't fetched, or the Node version detection picks up a different working tree.

**This is the classic drift pattern:** the failing job has a different action version than its working siblings, and the difference is invisible unless you check.

Concrete `actions/checkout` v3 → v4 changes that bite:
- v4 requires Node 20 on the runner; v3 worked on Node 16.
- v4 default `fetch-depth: 1` behavior is identical to v3, but the `submodules` and `lfs` defaults shifted in some versions.
- v4 changed how it handles workflows triggered by other workflows (token scoping).

You won't memorize the changelogs. The skill is to **notice the version mismatch** when reading the file, then check the release notes for the specific transition.

## The actions you'll see in rev-eval-ai-pep

The repo's `ci-pipeline.yml` uses a small, conventional set:

| Action | Purpose | Common pin |
|---|---|---|
| `actions/checkout` | Clone the repo onto the runner | `@v4` |
| `actions/setup-node` | Install a specific Node.js version | `@v4` |
| `actions/setup-python` | Install a specific Python version | `@v5` |
| `actions/cache` | Cache dependency directories between runs | `@v4` |
| `actions/upload-artifact` / `download-artifact` | Pass files between jobs | `@v4` |
| `docker/login-action` | Authenticate to a container registry | `@v3` |
| `docker/build-push-action` | Build and push Docker images | `@v5` |
| `aws-actions/configure-aws-credentials` | Assume an AWS role / configure creds | `@v4` |

Two boundaries to know:

- `actions/upload-artifact@v3` and `@v4` are **mutually incompatible**. v4 changed the storage backend. A v3 uploader and a v4 downloader will not find each other. This is a frequent seeded-bug class.
- `actions/setup-python@v4` → `@v5` dropped support for some older Python versions. If you pin Python 3.7 and bump the setup action, you'll get an "unsupported version" error.

## Why pinning matters: supply chain + reproducibility

Two distinct concerns, often conflated:

**Supply chain.** When you `uses: someorg/some-action@v2`, you're executing whatever code lives at that tag on whichever runner the job lands on. If the maintainer's account is compromised and they move `@v2` to point at a malicious commit, your next CI run executes the malicious code with access to your repository's secrets. SHA pinning (`@<full-SHA>`) defeats this attack because the SHA cannot be moved.

**Reproducibility.** A pipeline that succeeded last week may fail this week because `@v4` now points at a newer commit. Without pinning, you cannot reproduce a green build from history — the dependencies floated underneath you.

The defensible practices, in order of strictness:

1. **Minimum:** pin to a major version (`@v4`). Almost everyone does this.
2. **Better:** pin to an exact version (`@v4.1.7`) with a Dependabot/Renovate config to bump it on a schedule.
3. **Strictest:** pin to a SHA with a comment indicating the human-readable version. Used by security-conscious orgs and required for some compliance regimes.

```yaml
# Strict pinning style
- uses: actions/checkout@b4ffde65f46336ab63a395dffa00cd4e2bc3d5b7  # v4.1.7
```

The PEP cohort doesn't need to adopt SHA pinning today, but you should be able to **recognize** all three styles in the wild.

## Diagnosing a drift-induced failure

Signs that a CI failure is version-drift:

- The failing step is `actions/checkout`, `actions/cache`, `actions/upload-artifact`, or another well-known action — not custom code.
- The error message references an internal API change ("Node 16 actions are deprecated," "artifact format mismatch," "cache key not found").
- A sibling job using a different version of the same action passes.
- The workflow worked "last week" — but you can't pin down what changed in your own code, because nothing in your code changed.

Diagnostic moves:

1. `grep -n "uses:" .github/workflows/ci-pipeline.yml` — list every action reference.
2. Look for inconsistent versions of the same action across jobs.
3. For each mismatched action, open its GitHub release notes and skim for breaking changes between the two versions you have.

## Example / Worked Scenario

A realistic excerpt with a drift bug:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - uses: actions/cache@v4
        with:
          path: ~/.pnpm-store
          key: pnpm-${{ hashFiles('pnpm-lock.yaml') }}
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  test-services:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v3        # ← drift candidate
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - uses: actions/cache@v3           # ← drift candidate
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('**/requirements.txt') }}
      - run: pip install -r requirements.txt
      - run: pytest

  upload-coverage:
    runs-on: ubuntu-latest
    needs: test-services
    steps:
      - uses: actions/download-artifact@v4   # ← mismatch with v3 uploader
        with:
          name: coverage-report
```

Read this carefully:

- `lint` uses `actions/checkout@v4` and `actions/cache@v4`.
- `test-services` uses `actions/checkout@v3` and `actions/cache@v3`.
- `upload-coverage` uses `actions/download-artifact@v4` — but there's no `upload-artifact` step shown that uploads `coverage-report`. If the missing upload step was on v3, the v4 download cannot find the artifact (v3/v4 are mutually incompatible).

A failure here would manifest as `upload-coverage` reporting "Artifact 'coverage-report' not found" — pointing the unwary at the download step. The real cause is upstream: the upload step is either missing or on a different artifact version.

## Common Pitfalls

- **Mixing major versions of the same action in one workflow.** `actions/checkout@v3` in one job, `@v4` in another. Either standardize, or pin and document the exception.
- **`@main` or `@master` references.** A live wire — breaking changes land in your CI the moment they're merged upstream.
- **`actions/upload-artifact@v3` paired with `@v4` download (or vice versa).** Silent failure mode: artifact appears "not found."
- **Not reading the action's `README.md` before bumping a major version.** Most breaking changes are documented there; skipping the read is how drift bugs land.
- **Assuming `@v4` means "version 4.0.0".** It means "whatever the maintainer last tagged as v4" — which today might be 4.2.7.

## Key Takeaways

- Every `uses:` reference points at executable code. The pin determines how reproducible and how secure your reference is.
- Major-version tags (`@v4`) are the common-case pin. SHA pins are the secure pin. Branch refs (`@main`) are dangerous.
- Drift bugs hide in inconsistency: one job uses `@v3`, another uses `@v4`, the failure shows up somewhere downstream.
- When you see a failure in a well-known action, list every `uses:` in the file and check for version mismatches before debugging anything else.
- `actions/upload-artifact` and `actions/download-artifact` must match major versions across the same workflow.

---
*Prerequisites: [01-github-actions-structure-workflows-jobs-steps-runners.md](01-github-actions-structure-workflows-jobs-steps-runners.md), [02-yaml-syntax-pitfalls.md](02-yaml-syntax-pitfalls.md).*
