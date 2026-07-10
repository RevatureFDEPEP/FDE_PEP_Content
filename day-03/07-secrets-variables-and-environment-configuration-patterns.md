# Secrets, Variables, and Environment Configuration Patterns

> *Day 3: Diagnose the CI Pipeline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
At least one of today's five seeded bugs in `ci-pipeline.yml` is a configuration plumbing bug: a secret that's referenced but never defined, an env var typo'd at one layer and read at another, or an env block scoped at the wrong level. This topic teaches you how secrets and env vars flow from repository settings into a step's process environment, and how to trace a missing or wrong value back to its source.

## The four sources of values a step can read

When a `run:` step on the runner reads an environment variable, that value came from one of four places. Knowing which is the first move when something is wrong.

1. **GitHub Actions Secrets** — encrypted values stored in repo/org/environment settings, surfaced via `${{ secrets.NAME }}`.
2. **GitHub Actions Variables** — non-encrypted config values, surfaced via `${{ vars.NAME }}`.
3. **Workflow `env:` blocks** — declared in YAML at workflow, job, or step scope.
4. **Runner-default env vars** — `GITHUB_TOKEN`, `GITHUB_SHA`, `GITHUB_REPOSITORY`, etc., set automatically.

Each layer can override the previous. A step-level `env:` wins over a job-level `env:` wins over a workflow-level `env:`. Secrets and vars are *referenced into* env blocks via `${{ }}` interpolation — they aren't environment variables themselves until you wire them in.

## Secrets: how they're defined and consumed

Secrets are configured outside the YAML file, in repository settings (Settings → Secrets and variables → Actions). They have three scopes:

- **Repository secrets** — available to all workflows in this repo.
- **Environment secrets** — available only when a job declares `environment: <name>`. Used for prod vs staging gating.
- **Organization secrets** — defined at the org level, optionally scoped to specific repos.

In YAML, you reference them via the `secrets` context:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production              # required for environment-scoped secrets
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to ECS
        run: ./scripts/deploy.sh
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### What "missing secret" looks like

If `${{ secrets.AWS_ACCESS_KEY_ID }}` is referenced but the secret doesn't exist in the repo's configuration, GitHub Actions **substitutes an empty string** and continues. The `aws-actions/configure-aws-credentials` step then fails downstream with "credentials not found" — and you may misdiagnose it as an AWS-side issue when the real problem is that the secret was never defined (or is defined in the wrong scope).

This is a deliberate design choice: GitHub never echoes whether a secret exists, because that would leak information about which secret names are configured. The cost is that missing secrets fail late and ambiguously.

**Diagnostic move:** in the failing step's log, look for `***` (asterisks) — GitHub redacts secret values in logs. If you expected a redacted value to appear in the log context and instead see empty/blank, the secret may not be defined or may be out of scope.

## Variables (the non-secret counterpart)

Same configuration UI, same scoping rules, but **not encrypted** and visible to anyone with repo access. Used for non-sensitive config — region names, account IDs that aren't sensitive, feature flags.

```yaml
- name: Configure
  env:
    AWS_REGION: ${{ vars.AWS_REGION }}
    ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
  run: ./scripts/configure.sh
```

If you find yourself wanting to commit a value to the YAML file but it might vary across forks/environments, `vars:` is the right tool. Don't waste a secret slot on a region name.

## The three scopes of `env:`

```yaml
name: CI Pipeline

env:                                     # WORKFLOW scope — all jobs, all steps
  CI: 'true'
  NODE_ENV: test

jobs:
  test:
    runs-on: ubuntu-latest
    env:                                 # JOB scope — all steps in this job
      DATABASE_URL: postgres://localhost/test
      PYTHONUNBUFFERED: '1'
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        env:                             # STEP scope — only this step
          PYTEST_ADDOPTS: '-v --tb=short'
        run: pytest
```

Precedence (lower wins on conflict): workflow < job < step. A step-level `env:` overrides the same key set at job or workflow level.

**Common bug:** an env var set at workflow scope is intentionally overridden at step scope for one step. A later refactor moves the step but the override stays — the wrong env value now leaks into a step that didn't expect it.

### Where the env block lives matters

```yaml
# WRONG SCOPE — env is at the step list level, not the step level
- name: Run tests
  run: pytest
env:                                     # ← parses under the job, not the step
  PYTEST_ADDOPTS: '-v'
```

The YAML still parses. The env var is set at job scope, leaking into every step. If a subsequent step is sensitive to `PYTEST_ADDOPTS`, you've created a bug invisible from reading just this snippet. This is the kind of indentation slip that drives seeded YAML bugs (see [02-yaml-syntax-pitfalls.md](02-yaml-syntax-pitfalls.md)).

## The interpolation moment

`${{ ... }}` expressions are interpolated by **GitHub Actions**, before the shell sees the line. By the time `bash` reads your `run:` block, the `${{ }}` has been replaced with the actual value (or empty string for missing references).

This has two important consequences:

**1. Secrets get baked into the shell command.** Anything you put in `${{ secrets.X }}` becomes part of the shell command line. If you write:

```yaml
- run: curl -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" https://api.example.com
```

You're constructing a shell command with the literal token in it. GitHub redacts the token from logs, but anyone who can read the process list on the runner (e.g., a malicious dependency post-install script) can see it. The safer pattern is to pass via `env:`:

```yaml
- env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com
```

Now the token is in the environment, not on the command line, and the shell does the interpolation.

**2. Missing references silently become empty.** `${{ secrets.NOT_DEFINED }}` interpolates to `""`. The next step gets `""` and may not validate it. The failure surfaces deeper in the pipeline as a confusing error.

## Common configuration bug patterns

These are the patterns you'll see in seeded bugs and in real pipelines.

### Typo in the secret name

```yaml
- env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}   # missing _ID
  run: aws s3 cp file.txt s3://bucket/
```

The secret is configured as `AWS_ACCESS_KEY_ID`. The reference asks for `AWS_ACCESS_KEY`. Result: empty string, AWS rejects unsigned request.

### Wrong context (`secrets` vs `vars` vs `env`)

```yaml
- env:
    DATABASE_URL: ${{ env.DATABASE_URL }}          # wrong — env is set FROM env, circular
    REGION: ${{ secrets.AWS_REGION }}              # wrong — should be vars
```

`vars` is read-only at run time. Don't try to mutate it. `env` references work, but only if the variable is set at a higher scope. `secrets` is for sensitive values only.

### Environment-scoped secrets without `environment:`

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    # missing: environment: production
    steps:
      - run: echo "${{ secrets.PROD_API_KEY }}"    # always empty
```

If `PROD_API_KEY` was defined under the `production` environment in settings, the job must declare `environment: production` to access it. Without the declaration, the reference is empty.

### Missing `GITHUB_TOKEN` permissions

```yaml
jobs:
  comment:
    runs-on: ubuntu-latest
    # missing: permissions: pull-requests: write
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({ ... })
```

`GITHUB_TOKEN` has read-only permissions by default in repos with restrictive defaults. Mutating actions need `permissions:` declared.

## Example / Worked Scenario

```yaml
name: CI Pipeline

env:
  AWS_REGION: us-east-1                  # workflow-level fallback

jobs:
  test:
    runs-on: ubuntu-latest
    env:
      DATABASE_URL: postgres://test_user:test_pass@localhost:5432/test_db
      MONGO_URL: mongodb://localhost:27017/test
    steps:
      - uses: actions/checkout@v4
      - run: pytest services/user-service/tests

  build-and-push:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}    # ← suspicious
          aws-region: ${{ env.AWS_REGION }}

      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr

      - name: Build and push
        env:
          ECR_REGISTRY: ${{ steps.ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/user-service:$IMAGE_TAG ./services/user-service
          docker push $ECR_REGISTRY/user-service:$IMAGE_TAG
```

Read the secret references carefully:

- `secrets.AWS_ACCESS_KEY_ID` — looks correct.
- `secrets.AWS_SECRET_KEY` — looks correct *unless* the secret in repo settings is actually named `AWS_SECRET_ACCESS_KEY` (the canonical name). A drift between the YAML reference and the configured secret name produces empty-string substitution and an "unable to locate credentials" error from AWS several steps later.

The fix is to first check what's actually configured: in repo settings, see if the secret name matches the reference. If it doesn't, either rename the secret or fix the reference. **You can't see what's defined from the YAML alone** — this is exactly the kind of bug where the diagnostic step is to read the repo's secrets list, not just the workflow file.

## Common Pitfalls

- **Typos in secret/var names produce empty strings, not errors.** Always cross-check the YAML reference against the actual secret name in repo settings.
- **`env:` at the wrong indentation level changes its scope.** Job-level env leaks into every step; workflow-level env leaks into every job.
- **Environment-scoped secrets require `environment:` on the job.** Without it, references resolve to empty.
- **Interpolating secrets directly into `run:` commands.** Use `env:` mapping instead — keeps the value out of the command line.
- **Assuming `GITHUB_TOKEN` is unrestricted.** Default permissions vary by repo policy; declare `permissions:` explicitly when mutating.

## Key Takeaways

- An env var visible to a step came from one of four sources: secrets, variables, an `env:` block, or runner defaults. Identify the source before debugging.
- Missing or misnamed secret references substitute as empty strings — failures surface downstream, ambiguously.
- The `env:` precedence is workflow < job < step. A step-level override wins.
- Cross-reference YAML against the repo's configured secrets and variables — you cannot diagnose plumbing bugs from the workflow file alone.
- Pass secrets through `env:` mappings, not direct interpolation into `run:` command lines.

---
*Prerequisites: [01-github-actions-structure-workflows-jobs-steps-runners.md](01-github-actions-structure-workflows-jobs-steps-runners.md), [02-yaml-syntax-pitfalls.md](02-yaml-syntax-pitfalls.md).*
