# Container Scanning (Trivy) in CI

> *Day 7 — Week 2, Tuesday. Robust Continuous Integration Pipelines.*

## Overview

A passing test suite tells you the code you wrote behaves correctly. It does not tell you whether the OpenSSL inside your base image has a remotely exploitable CVE published last week. Container vulnerability scanning is the discipline of treating the *whole filesystem* of your built image as something CI is responsible for inspecting, not just your application code.

The substrate ships images to ECR (Day 11). Anything you ship is something attackers can hit. The earlier you find a known-vulnerable dependency — ideally before merge — the cheaper it is to fix.

**Trivy** (from Aqua Security) is the de facto open-source container scanner. It's fast, has good defaults, runs as a single static binary, and has a maintained GitHub Action. The v3.0 PEP variant explicitly added Trivy to close what the M5.1 audit called the security-scan gap. Today you wire it in.

## What Trivy actually scans

A Trivy scan of a container image inspects several things in one pass:

1. **OS package vulnerabilities** — for each package installed via apt/apk/yum (e.g., `openssl`, `libxml2`, `curl`), look up the installed version against the OS distro's CVE feed. This is where most "you're running an out-of-date base image" findings come from.
2. **Application dependencies** — parse `requirements.txt`, `package-lock.json`, `pnpm-lock.yaml`, etc. found inside the image and check each library version against advisory feeds (PyPI's advisory DB, GitHub Advisory DB, npm audit feed).
3. **Misconfigurations** — Dockerfile lints (running as root, no `HEALTHCHECK`, wildcard `COPY`), Kubernetes manifest issues, IaC patterns.
4. **Secrets** — accidentally committed API keys, AWS credentials, private keys, etc. found in image layers.

For Day 7 the focus is the first two — OS and application CVEs in the images you just built.

## Severity model

Trivy reports findings with a severity (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN`) derived from the CVE's CVSS score and from the source feed's own rating. Each finding also carries:

- **`FixedVersion`** — the version that patches it, if one exists. Empty means no patch is available yet.
- **`PrimaryURL`** — the canonical advisory link (NVD, GHSA, RUSTSEC).
- **`Status`** — `affected`, `fixed`, `will_not_fix`, `fix_deferred`, `end_of_life`.

The actionable rows are roughly "HIGH or CRITICAL **with a FixedVersion**." Those you can resolve today by bumping a version. CRITICAL findings with no fix available are real but the response is different (mitigation, workaround, or accepting the risk) and shouldn't fail the CI build automatically — they'll just stick there until the upstream releases a patch.

## A realistic Trivy report

Running `trivy image rev-eval-ai-pep/user-service:abc123` against a Debian-slim-based Python service might produce (abbreviated):

```
rev-eval-ai-pep/user-service:abc123 (debian 12.5)
=================================================
Total: 14 (HIGH: 11, CRITICAL: 3)

┌──────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┬─────────────────────────────────────────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │                            Title                            │
├──────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┼─────────────────────────────────────────────────────────────┤
│ libxml2      │ CVE-2024-25062 │ HIGH     │ fixed  │ 2.9.14+dfsg-1.3   │ 2.9.14+dfsg-  │ libxml2: use-after-free in XMLReader                        │
│              │                │          │        │                   │ 1.3+deb12u1   │                                                             │
│ openssl      │ CVE-2024-0727  │ HIGH     │ fixed  │ 3.0.11-1~deb12u2  │ 3.0.13-1~deb  │ openssl: denial of service via null deref in PKCS12 parsing │
│ curl         │ CVE-2024-2398  │ HIGH     │ affected│ 7.88.1-10+deb12u5│               │ HTTP/2 push headers memory leak (no fix yet)                │
└──────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┴─────────────────────────────────────────────────────────────┘

rev-eval-ai-pep/user-service:abc123 (python-pkg)
================================================
Total: 2 (HIGH: 2)

┌──────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┬─────────────────────────────────────────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │                            Title                            │
├──────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┼─────────────────────────────────────────────────────────────┤
│ requests     │ CVE-2024-35195 │ HIGH     │ fixed  │ 2.31.0            │ 2.32.0        │ requests: session verify=False persists across requests     │
│ jinja2       │ CVE-2024-22195 │ HIGH     │ fixed  │ 3.1.2             │ 3.1.3         │ jinja2: xss in xmlattr filter                                │
└──────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┴─────────────────────────────────────────────────────────────┘
```

## How to triage that report

Walk the table top to bottom:

1. `libxml2 CVE-2024-25062 HIGH fixed` — Fixable by `apt-get update && apt-get upgrade -y` in the Dockerfile (you already do this on Day 5), or more permanently by **rebasing onto a fresher base image tag**. The point release `python:3.11-slim-bookworm` is updated regularly; rebuilding picks up patched system packages.
2. `openssl CVE-2024-0727 HIGH fixed` — Same fix as above. A single base-image rebuild often resolves a dozen OS findings at once.
3. `curl CVE-2024-2398 HIGH affected` — No FixedVersion yet. Real but un-actionable. Options: (a) ignore via `.trivyignore` with a justification comment, (b) drop curl from the image if your runtime doesn't need it, (c) accept and revisit weekly. **Do not** let this one finding gate every merge until the upstream patch lands.
4. `requests 2.31.0 → 2.32.0` — A pin bump in `requirements.txt`. One-line PR.
5. `jinja2 3.1.2 → 3.1.3` — Same.

The triage skill is reading a long scary table and quickly classifying: *fix now*, *suppress with justification*, *accept and revisit*. Not every finding requires panic; not every finding is safe to ignore.

## Wiring Trivy into CI

Add a job that runs after the image build:

```yaml
  scan-user-service:
    runs-on: ubuntu-latest
    needs: build-user-service
    steps:
      - uses: actions/checkout@v4

      - name: Download image artifact
        uses: actions/download-artifact@v4
        with:
          name: user-service-image
          path: /tmp

      - name: Load image into local Docker
        run: docker load -i /tmp/user-service.tar

      - name: Trivy scan
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: rev-eval-ai-pep/user-service:${{ github.sha }}
          severity: CRITICAL,HIGH
          ignore-unfixed: true
          exit-code: "1"
          format: table
          vuln-type: os,library
          trivyignores: .trivyignore
```

Key flags:

- **`severity: CRITICAL,HIGH`** — only report these tiers. MEDIUM/LOW are noisy and rarely gate-worthy.
- **`ignore-unfixed: true`** — skip findings with no FixedVersion. Combined with `exit-code: 1`, this means "fail the build only on things we can actually fix."
- **`exit-code: "1"`** — non-zero exit when findings remain after filtering, so the CI step fails.
- **`trivyignores: .trivyignore`** — path to the suppression file (see below).

## `.trivyignore` — documented exceptions

```
# .trivyignore — Trivy CVE suppressions for rev-eval-ai-pep
# Format: <CVE-ID>  (one per line, # comments allowed)

# curl HTTP/2 push memory leak — no upstream fix as of 2026-05-12.
# Service is internal-only, behind nginx; risk accepted by Platform.
# Revisit: 2026-08-01.
CVE-2024-2398
```

Every line *must* carry a justification comment. An undocumented entry in `.trivyignore` is technical debt at best, a security-policy violation at worst. Most teams enforce this with a PR-template checkbox or a periodic audit script.

## Output formats and reporting

Beyond the gate, you typically want:

- **SARIF output** for GitHub's Security tab — Trivy can emit `format: sarif` and you upload it via `github/codeql-action/upload-sarif`. Findings then show up in the repo's Security view alongside CodeQL results.
- **JSON output** archived as an artifact for trend tracking.
- **A PR comment summary** so reviewers see the delta vs base.

## Common Pitfalls

- **Failing the build on MEDIUM and LOW findings.** You will drown. The signal-to-noise ratio drops below useful and developers start auto-suppressing everything. Gate on HIGH+CRITICAL.
- **Not setting `ignore-unfixed: true`.** Without it, an upstream library with no patch yet permanently blocks merges. Distinguish "we won't fix this" from "we can't fix this."
- **Suppressing without justification.** Future-you will see a CVE in `.trivyignore` and have no idea whether it was triaged carefully or copy-pasted. The comment is the audit trail.
- **Scanning before the image is built.** Trivy needs a real image, either loaded into the runner's Docker daemon or available via registry. Wire the scan job's `needs:` to the build job.
- **DB rate limiting in CI.** Trivy downloads its vulnerability DB each run; on busy days the GHCR-hosted DB can hit rate limits. Cache the DB across runs (`TRIVY_CACHE_DIR` + `actions/cache`) or pin the trivy-action's `cache: true` option.
- **Treating the scan as one-and-done.** New CVEs are published daily. A clean scan today does not mean a clean scan next week. Run scans on a schedule against `main` (a nightly cron workflow), not just on PR.

## Key Takeaways

- Trivy scans built container images for OS package CVEs, application library CVEs, secrets, and misconfigurations.
- The actionable findings are HIGH/CRITICAL with a `FixedVersion`; the gate posture in CI should be exactly those.
- Triage means classifying findings into fix-now / suppress-with-justification / accept-and-revisit, not panicking at every red row.
- `.trivyignore` is for documented exceptions; every entry needs a justification comment and a revisit date.
- A scheduled `main`-branch scan catches CVEs published *after* your last PR — needed because the vulnerability landscape changes daily, not just when code changes.

---

*Prerequisites: Day 5 (multi-stage Dockerfiles producing the images Trivy scans), Day 7 Topic 6 (artifacts — how the scan job gets the built image from the build job).*
