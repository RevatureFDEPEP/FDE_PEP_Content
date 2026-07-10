# Build Caching Strategies in CI

> *Day 3: Diagnose the CI Pipeline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
A misconfigured cache is worse than no cache: it makes builds fast *and wrong*, masking dependency changes and producing flaky failures that surface only on a teammate's machine. One of today's five seeded bugs in `ci-pipeline.yml` is a cache misconfiguration — typically a key that doesn't include the lockfile, so the cache restores stale dependencies after the lockfile has changed. This topic teaches you what a CI cache actually is, how to read a cache configuration, and how to recognize when the cache is the culprit.

## What a CI cache actually is

A GitHub Actions cache is a tarball stored on GitHub's infrastructure, identified by a **key** and scoped to a **path**. On a job run:

1. The action computes the cache key (often by hashing files like `pnpm-lock.yaml`).
2. It asks GitHub: "is there a cache for this key?"
3. **Cache hit:** GitHub downloads the tarball and extracts it to the path. The step finishes quickly. Subsequent build commands skip the work the cache contains.
4. **Cache miss:** the step finishes with no extraction. The job continues. After the job ends, the path is tarballed and uploaded to GitHub as a new cache entry under that key.

The crucial invariant: **the key must change whenever the cached content should change**. If the key stays the same after you upgrade a dependency, the cache restores the old dependency, and your job runs against a stale tree.

## The canonical cache pattern

```yaml
- uses: actions/checkout@v4

- uses: actions/setup-node@v4
  with:
    node-version: '20'

- name: Get pnpm store path
  id: pnpm-store
  run: echo "path=$(pnpm store path)" >> "$GITHUB_OUTPUT"

- uses: actions/cache@v4
  with:
    path: ${{ steps.pnpm-store.outputs.path }}
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-

- run: pnpm install --frozen-lockfile
```

Read the cache key piece by piece:

- `${{ runner.os }}` — `Linux`, `Windows`, or `macOS`. A cache built on Linux is binary-incompatible with macOS, so we partition.
- `pnpm` — a literal namespace tag. Distinguishes this cache from a pip or Maven cache in the same repo.
- `${{ hashFiles('**/pnpm-lock.yaml') }}` — a hash of every `pnpm-lock.yaml` in the repo. **This is the dependency fingerprint.** Any change to any lockfile changes the hash, which changes the key, which forces a fresh install.

Without that lockfile hash in the key, a dependency upgrade in `pnpm-lock.yaml` would not bust the cache — and the job would restore the previous tarball, running against stale dependencies.

## `key:` vs `restore-keys:`: prefix-match fallback

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

- `key:` is the **exact match** the action looks for first. On a hit, the cached tarball is restored, and after the job no new cache is uploaded (the existing one wins).
- `restore-keys:` is a list of **prefix matches** tried in order if the exact key misses. On a prefix hit, the most recent cache matching that prefix is restored, and a *new* cache is uploaded at the end of the job under the exact `key:`.

This dance gives you "warm" caches for first-time installs after a lockfile change: you don't get the exact old cache, but you get a recent-ish one as a starting point, which then installs fewer packages from scratch.

**Bug pattern:** if `restore-keys:` is too greedy (the prefix matches across unrelated content), you can restore the wrong cache. Conversely, if `restore-keys:` is missing entirely, every lockfile change pays the full cold-install cost. Neither is catastrophic — just inefficient — but they're worth recognizing.

## The "cache key that doesn't include the lockfile" bug

This is one of the most common seeded-bug archetypes:

```yaml
# BROKEN
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip                # ← no hash, no lockfile reference
    restore-keys: |
      ${{ runner.os }}-pip
```

What happens:

1. First run: cache miss (no existing cache). Job installs deps from scratch. Cache uploaded.
2. Second run: cache hit on the static key `Linux-pip`. Old deps restored.
3. Someone updates `requirements.txt` (adds a new package). Pushes a PR.
4. CI runs: cache key is still `Linux-pip`. Hit. Old deps restored. The new package is *not* installed. Tests fail with `ModuleNotFoundError`.
5. Developer adds `pip install <missing-package>` to their tests. Tests pass locally because their machine has the package. CI keeps failing.
6. Eventually someone notices the cache and clears it.

The fix is one line:

```yaml
key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
```

When reading a cache config in `ci-pipeline.yml`, the first thing to check is **whether the key includes a hash of the dependency manifest**. If it doesn't, suspect it.

## What to cache vs what not to cache

Cache:

- **Dependency stores.** `~/.pnpm-store`, `~/.cache/pip`, `~/.m2/repository`, `~/.gradle/caches`. These are the big wins.
- **Pre-built artifacts that are expensive to produce.** Compiled Rust crates, native add-ons.

Don't cache:

- **`node_modules/` directly.** Use the package manager's *store* directory instead. `node_modules/` has symlinks and a working-tree layout that doesn't tar/untar reliably across runners. Modern pnpm caching points at `~/.pnpm-store`, not `node_modules/`.
- **Build outputs you ship.** Use artifacts (`actions/upload-artifact`) for things you pass between jobs. Caches are *opportunistic* — a cache miss is fine; an artifact miss is a failure.
- **Anything sensitive.** Caches are visible to anyone who can read repository workflows.
- **Anything that depends on Git history.** Caches are per-key, not per-commit.

## Setup actions with built-in caching

Newer versions of `setup-node`, `setup-python`, and similar actions ship with `cache:` inputs that handle the entire pattern for you:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'pnpm'
    cache-dependency-path: 'pnpm-lock.yaml'

- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'
    cache-dependency-path: 'services/**/requirements.txt'
```

These produce a correct key including the lockfile hash automatically. If you're authoring new CI, prefer the built-in `cache:` input over a hand-rolled `actions/cache` step. When reading existing CI, you may see both patterns side-by-side.

## Reading a cache hit/miss in the log

In the Actions log, a cache step prints a line like:

```
Cache restored from key: Linux-pnpm-abc123def456...
```

or:

```
Cache not found for input keys: Linux-pnpm-abc123def456, Linux-pnpm-
```

At the end of the job, you'll see:

```
Cache saved with the key: Linux-pnpm-abc123def456...
```

(Only when there wasn't an exact-key hit; if there was, no upload happens.)

When diagnosing a cache-related failure, the first move is to find these log lines and see:
- Which key the cache action computed.
- Whether it hit or missed.
- Whether the key looks reasonable (i.e., does it include the lockfile hash?).

## Example / Worked Scenario

A cache misconfiguration in `ci-pipeline.yml`:

```yaml
jobs:
  test-services:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, question-management-service, test-management-service]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-cache                   # ← bug: no OS, no hash, no service
          restore-keys: |
            pip-

      - run: pip install -r services/${{ matrix.service }}/requirements.txt
      - run: pytest services/${{ matrix.service }}/tests
```

Three things wrong with this cache key:

1. **No `runner.os`.** Not a problem today (all jobs are `ubuntu-latest`), but it's a latent landmine if someone adds a Windows job later.
2. **No `hashFiles(...)`.** A change to any `requirements.txt` does not invalidate the cache. Dependency changes silently restore stale state.
3. **One key shared across the matrix.** All three services share `pip-cache`. The first matrix job to run uploads its dependency tree. The other two matrix jobs hit that cache and install the wrong set of deps. Whichever happens to run first "wins" the cache for the others.

The corrected key would look like:

```yaml
key: ${{ runner.os }}-pip-${{ matrix.service }}-${{ hashFiles(format('services/{0}/requirements.txt', matrix.service)) }}
```

That key:
- Partitions by OS.
- Partitions by service (so each matrix entry has its own cache).
- Invalidates when that service's `requirements.txt` changes.

The bug here isn't a parser error or a missing reference — it's that the cache is **succeeding at the wrong thing**, producing a green pipeline that's running against wrong dependency state.

## Common Pitfalls

- **Cache key without a dependency-manifest hash.** Dependency upgrades don't invalidate. This is the most common seeded-bug archetype for caching.
- **Caching `node_modules/` directly.** Tarball/untar of symlink-heavy directories is fragile. Use the package manager's store path.
- **One shared cache across a matrix.** All matrix entries collide on the same key, last-writer-wins semantics produce flaky failures.
- **`restore-keys:` too broad.** Restoring an unrelated cache as a "warm start" can produce wrong state. Keep prefix matches narrow.
- **Forgetting that a cache miss is normal.** A pipeline that builds correctly without a cache should still build correctly *with* one. If introducing the cache changes correctness, the cache config is wrong.

## Key Takeaways

- A CI cache is a tarball keyed by a string. The key must change whenever the cached content should change.
- The key should include a hash of the dependency manifest (`hashFiles('**/pnpm-lock.yaml')`, `hashFiles('**/requirements.txt')`). Without it, dependency changes silently restore stale state.
- Partition by OS, by package manager, and by anything else that affects the content (matrix entries, target architectures).
- Prefer the built-in `cache:` input on `setup-node` / `setup-python` for new workflows. Recognize both patterns when reading existing ones.
- A misconfigured cache is worse than no cache. When CI fails with "module not found" but works locally, suspect the cache before suspecting your code.

---
*Prerequisites: [01-github-actions-structure-workflows-jobs-steps-runners.md](01-github-actions-structure-workflows-jobs-steps-runners.md), [02-yaml-syntax-pitfalls.md](02-yaml-syntax-pitfalls.md), [08-polyglot-development-environment-git-pnpm-python-docker.md](../day-01/08-polyglot-development-environment-git-pnpm-python-docker.md).*
