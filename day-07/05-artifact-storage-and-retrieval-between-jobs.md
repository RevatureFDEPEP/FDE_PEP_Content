# Artifact Storage and Retrieval Between Jobs

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

GitHub Actions jobs run on **independent runners**. Each job starts with a fresh VM, clones the repo, and ends when its steps finish. The filesystem of one job is invisible to every other job. This isolation is a *feature* (clean, reproducible environments) and a *problem* (how does the test job consume the image the build job just built?).

The answer is **artifacts**: a per-workflow blob store baked into Actions. A job uploads files; another job downloads them. `actions/upload-artifact` and `actions/download-artifact` are the two halves of the contract.

Artifacts are also the right way to pass things *out* of a CI run to humans — a coverage XML, a Trivy SARIF report, a screenshot from a failing E2E test. After the workflow finishes, artifacts persist for a configurable retention window and are downloadable from the run's summary page.

## When to use artifacts

Three categories:

1. **Inter-job handoff during the same workflow.** Job A builds a Docker image; Job B scans it; Job C pushes it. A uploads, B/C download.
2. **Persistent build output.** A coverage report, a SARIF file, a built static site, a wheel/sdist. Stored for later download, audit, or for a follow-up workflow to consume via `dawidd6/action-download-artifact` from a sibling workflow.
3. **Debugging breadcrumbs from a failing run.** Server logs, container `docker logs` output, screenshots, browser console captures. Conditionally uploaded `if: failure()` so they only exist when there's a reason to inspect them.

Artifacts are **not** for: passing secrets (they're visible to anyone with read access to the run), or for caching dependencies between *runs* (that's `actions/cache`'s job — different mechanism, different lifecycle).

## upload-artifact basics

```yaml
- name: Upload coverage XML
  uses: actions/upload-artifact@v4
  with:
    name: coverage-user-service
    path: services/user-service/coverage.xml
    retention-days: 14
    if-no-files-found: error  # fail if the file isn't there
```

Key inputs:

- **`name`** — the artifact's identifier. Must be unique within a workflow run (v4 enforces this; v3 silently merged). Including a matrix value in the name avoids collisions.
- **`path`** — file, directory, or glob. Globs use the GitHub Actions glob syntax (similar to gitignore patterns).
- **`retention-days`** — how long GitHub stores it. Default is the repo's configured retention (often 90 days). Set lower for transient handoffs to save storage quota.
- **`if-no-files-found`** — `warn` (default), `error`, or `ignore`. **Always `error` for handoffs** — silently uploading nothing is the worst failure mode. `warn` is OK for optional debug artifacts.

## download-artifact basics

```yaml
- name: Download coverage XML
  uses: actions/download-artifact@v4
  with:
    name: coverage-user-service
    path: ./coverage  # destination directory; created if missing
```

Default `path` is the workspace root. Omitting `name` downloads *all* artifacts produced by the workflow so far into subdirectories — useful for an aggregation job at the end of a fan-out.

## Worked Scenario — image build/scan/push handoff

A common pipeline shape in the substrate: build the image once, scan it, push it. You do not want to build twice — building is the slowest step and producing different bits in the build job vs the push job creates a reproducibility hole.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tag }}
    strategy:
      fail-fast: false
      matrix:
        service: [user-service, question-management-service]
    steps:
      - uses: actions/checkout@v4

      - id: meta
        run: echo "tag=${{ matrix.service }}:${{ github.sha }}" >> "$GITHUB_OUTPUT"

      - name: Build image
        run: |
          docker build \
            -t rev-eval-ai-pep/${{ matrix.service }}:${{ github.sha }} \
            -f services/${{ matrix.service }}/Dockerfile \
            services/${{ matrix.service }}

      - name: Save image to tar
        run: |
          docker save rev-eval-ai-pep/${{ matrix.service }}:${{ github.sha }} \
            -o /tmp/${{ matrix.service }}.tar

      - name: Upload image artifact
        uses: actions/upload-artifact@v4
        with:
          name: image-${{ matrix.service }}
          path: /tmp/${{ matrix.service }}.tar
          retention-days: 1
          if-no-files-found: error

  scan:
    runs-on: ubuntu-latest
    needs: build
    strategy:
      fail-fast: false
      matrix:
        service: [user-service, question-management-service]
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: image-${{ matrix.service }}
          path: /tmp

      - name: Load image
        run: docker load -i /tmp/${{ matrix.service }}.tar

      - name: Trivy scan
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: rev-eval-ai-pep/${{ matrix.service }}:${{ github.sha }}
          severity: CRITICAL,HIGH
          ignore-unfixed: true
          exit-code: "1"

  push:
    runs-on: ubuntu-latest
    needs: [build, scan]
    if: github.ref == 'refs/heads/main'
    strategy:
      fail-fast: false
      matrix:
        service: [user-service, question-management-service]
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: image-${{ matrix.service }}
          path: /tmp
      - run: docker load -i /tmp/${{ matrix.service }}.tar
      # ECR auth + push steps (covered Day 11) follow here
```

The flow: build once, save as a tar, upload. Scan and push jobs both download and `docker load` the same bits. The push job only runs on `main`, but the scan still happens on PRs — so a vulnerability detected on a PR blocks merge before the image ever reaches the registry.

`retention-days: 1` is appropriate here. The artifact is a handoff, not a record. Letting it sit for 90 days burns the org's artifact storage quota for no benefit.

## Conditional artifacts for failure diagnosis

```yaml
- name: Capture container logs on failure
  if: failure()
  run: |
    docker compose logs --no-color > compose-logs.txt
    docker ps -a > docker-ps.txt

- name: Upload failure artifacts
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: failure-logs-${{ github.run_id }}-${{ matrix.service }}
    path: |
      compose-logs.txt
      docker-ps.txt
      services/${{ matrix.service }}/test-results/
    retention-days: 7
    if-no-files-found: warn
```

`if: failure()` runs the step only when an earlier step in the job failed. Pairs perfectly with artifact upload — you get diagnostic breadcrumbs exactly when you need them, and pay no storage cost when the run succeeded.

## Artifacts vs caches — a crisp distinction

| Concern               | Artifacts                               | Cache                                       |
| --------------------- | --------------------------------------- | ------------------------------------------- |
| Scope                 | Single workflow run                     | Across runs (keyed by hash)                 |
| Lifetime              | `retention-days` (default 90)           | Up to 7 days, evicted by LRU at 10 GB/repo  |
| Intent                | Pass outputs between jobs / to humans   | Speed up dependency restoration             |
| Determinism           | Always present if upload succeeded      | May miss; code must handle cache-miss path  |
| Visibility            | Downloadable from run summary           | Internal; surfaces only in logs             |

If you find yourself uploading an artifact named "node-modules-cache", you want `actions/cache` instead.

## v4 changes worth knowing

`actions/upload-artifact@v4` and `download-artifact@v4` are not API-compatible with v3:

- **Unique names required.** Two matrix legs trying to upload `coverage.xml` both as `coverage` will fail in v4 (v3 merged). Include `${{ matrix.X }}` in the name.
- **Immutability.** You cannot append to an existing artifact mid-run. To aggregate, use `actions/upload-artifact/merge@v4` at the end.
- **No artifact downloads from in-progress jobs.** A job cannot download an artifact that hasn't been uploaded yet; this was always logically true but v4 enforces it more strictly.
- **Faster.** v4 is dramatically faster than v3 for large artifacts (new backend).

Pin to v4 for new workflows; existing v3 usage works but is on a deprecation path.

## Common Pitfalls

- **Two matrix legs uploading the same artifact name.** v4 fails noisily, v3 silently merged with last-write-wins. Always interpolate matrix vars into the name.
- **Forgetting `if-no-files-found: error`.** A step that produced no output (because of a glob typo or a path mistake) uploads an empty artifact and a downstream job downloads nothing. Hours of debugging avoidable by setting `error`.
- **Using artifacts for cross-workflow communication.** Workflow A's artifacts are not auto-visible to Workflow B in the same repo; you need a separate action (`dawidd6/action-download-artifact`) and a `workflow_run` trigger. Often a custom workflow design or a registry push is cleaner.
- **Uploading huge artifacts you'll never read.** Verbose log dumps, gigabytes of E2E video recordings — costs storage quota. Either filter before upload or set a short `retention-days`.
- **Treating an artifact as durable storage.** They expire. If you need something forever, put it in S3 or a release asset.
- **Secrets in artifacts.** Artifacts are visible to anyone with `Read` access to the repo's Actions tab. Don't upload `.env` files, kubeconfigs, or anything sensitive even "temporarily."

## Key Takeaways

- Artifacts move files between jobs and out to humans; uses are inter-job handoff, persistent build outputs, and conditional failure diagnostics.
- `upload-artifact@v4` + `download-artifact@v4` with unique names per matrix leg.
- Always set `if-no-files-found: error` for handoffs; the silent-empty-upload failure mode is brutal to debug.
- Use `retention-days` to scope artifact lifetime — short for handoffs, longer for audit-trail outputs.
- Artifacts are not caches and are not durable storage; they're a workflow-scoped blob store with a retention window.
- Conditional `if: failure()` artifacts for log/diagnostic capture pay only when something went wrong.

---

*Prerequisites: Day 7 Topic 1 (parallel jobs need a way to pass data — that's this), Day 5 (image builds — what you're packing into a tar and uploading).*
