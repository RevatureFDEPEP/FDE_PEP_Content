# Image Tagging Strategy — SHA, Semver, Latest, and Trade-offs

> *Day 5: Advanced Container Builds & Multi-Service Integration — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
You can now push images to a registry. The question this topic answers is: **under what name?** Tagging is the closest thing the container ecosystem has to a release strategy — it determines reproducibility, rollback behavior, debugging speed, and how confusing your registry browser becomes after a month.

There is no single correct answer. Different tag schemes optimize for different things, and most production setups use **two or more in parallel**. In code review (later today and recurring throughout the curriculum), you'll be asked to *defend* the scheme you chose. Knowing the trade-offs is the prerequisite.

## The Three Tag Schemes
### 1. Tag-by-SHA — reproducibility-first
```
localhost:5000/user-service:sha-a1b2c3d
localhost:5000/user-service:sha-a1b2c3d4e5f6789abcdef0123456789abcdef012
```

The tag encodes the git commit that produced the image. Two builds of the same commit produce two images with the same tag (and, modulo non-determinism, the same digest). Rolling back is `docker pull user-service:sha-<oldsha>`.

**Wins:** Perfect traceability — given a running container, you can `git checkout` exactly the code it was built from. CI can reliably re-pull what it built, even days later. No ambiguity about *which build* a deploy refers to.

**Costs:** Human-unfriendly. "Is `sha-a1b2c3d` newer than `sha-9f8e7d6`?" can't be answered without `git log`. Registries fill up fast. Operators reading dashboards see noise.

### 2. Tag-by-Semver — human-friendly releases
```
localhost:5000/user-service:1.4.2
localhost:5000/user-service:1.4
localhost:5000/user-service:1
```

The tag encodes a version number, usually following semver (`MAJOR.MINOR.PATCH`). Common pattern: tag a release with all three (`1.4.2`), `1.4`, and `1` — the latter two get repointed as patches and minor releases ship. Pulling `:1` gets you "the latest 1.x.y".

**Wins:** Reads like a release version because it is one. Communicates *intent* about compatibility (semver convention: major bumps for breaking changes). Pairs naturally with git tags and changelogs.

**Costs:** The "latest 1.x" floating semantics mean `:1` is *not* reproducible — re-pulling tomorrow may give you different bits. Only suitable for genuinely released artifacts, not every CI build. Discipline-dependent: nothing forces semver semantics on the tag.

### 3. Tag-by-`latest` (and friends like `:dev`, `:edge`) — convenience-first
```
localhost:5000/user-service:latest
localhost:5000/user-service:dev
```

A single tag that's repointed every time a new build happens. Pulling `:latest` always gets the newest push.

**Wins:** Easy to type. Good for the *very* fast inner loop (developer rebuild, developer test, developer push, teammate pulls `:latest`). Default behavior of many tools.

**Costs:** Severely. `:latest` is **not reproducible** — what it points to changes constantly. If two team members `docker pull user-service:latest` five minutes apart, they may get different images. Rollbacks are impossible from the tag alone. In production environments `:latest` is widely considered an anti-pattern; in CI it makes flaky failures undebuggable.

`:dev` is slightly better in that the contract is more honest (everyone knows `:dev` floats), but the reproducibility problem is identical.

## The Real Answer: Use Multiple Tags
In practice you tag the same image with **multiple** tags simultaneously, optimizing for different consumers:

```bash
SHA=$(git rev-parse --short HEAD)
docker build \
  -t localhost:5000/user-service:sha-$SHA \
  -t localhost:5000/user-service:dev \
  ./services/user-service

docker push localhost:5000/user-service:sha-$SHA
docker push localhost:5000/user-service:dev
```

The first tag is the **authoritative** reference — CI logs it, deploy systems reference it, rollback uses it. The second is a **convenience** pointer for the inner loop.

For releases, you'd add `:1.4.2`, `:1.4`, and possibly `:1` at the same time.

## Choosing for the PEP Substrate
For PEP v3.0, the recommendation that the curriculum lands on:

| Tag scheme | Used for | Why |
|---|---|---|
| `sha-<short-git-sha>` | Every CI build | Reproducibility; this is what deploy-like jobs pull |
| `dev` | Latest build on `main` | Developer inner loop, integration tests |
| `latest` | **Not used** | Avoid; the trainer will challenge if you propose it |

Semver tags don't appear in PEP v3.0 because the cohort doesn't ship releases. They re-enter in the 10-week intensive when teams begin tagging actual releases.

## Defending the Choice in Review
When reviewers (or trainers) push on your tagging choice, expect questions like:
- "How would you roll back if `:dev` is broken?" *(Pull the previous `:sha-...` tag from CI logs; redeploy.)*
- "Why not just use `:latest`?" *(Reproducibility — two pulls minutes apart could yield different images, which makes flaky CI failures unattributable.)*
- "Why short SHA and not full SHA?" *(Short SHA is unambiguous within any realistic repo and human-readable in logs; the registry stores the full digest separately for the truly paranoid case.)*

Day 4's discipline of owning what you submit applies here too: you have to know *why* the tag scheme is the way it is, not just that it's the default.

## Example / Worked Scenario
The trainer presents a scenario:

> "The `question-management-service` failed in CI integration tests against build `sha-9f8e7d6`. A developer panic-pushed a fix, which got tagged `:dev`. The team is now running tests against `:dev`. A reviewer asks: 'How do we know we're testing the fix and not something else that landed?'"

**Reasoning to defend the answer.**

The reviewer's concern is legitimate. `:dev` floats — between the panic-push and "now" (even a few minutes later), another CI run could have re-pointed `:dev` at a *different* image. If the team is debugging by pulling `:dev`, they may not be pulling what they think.

**Correct procedure:**

1. Identify the fix's commit SHA from `git log`: say `b2c3d4e`.
2. The CI run for that commit produced `localhost:5000/question-management-service:sha-b2c3d4e`.
3. Run the integration tests against the **SHA-tagged image**, not `:dev`:
   ```bash
   docker pull localhost:5000/question-management-service:sha-b2c3d4e
   QM_IMAGE=sha-b2c3d4e docker compose up
   ```
4. Once the fix is confirmed, *then* re-tag for the floating consumers if desired.

**Defense in review:** "We test against the SHA tag because `:dev` is non-reproducible. The SHA tag is immutable in practice — it's named after the commit, and rebuilding the same commit produces a functionally identical image. That gives us a stable target for debugging and an unambiguous rollback target if the fix turns out to be wrong."

## Common Pitfalls
- **Using `:latest` in CI.** Two CI runs interleaving can leave `:latest` pointing at the *older* of two builds. Always pin to a SHA tag in CI.
- **Re-pushing to the same SHA tag.** If you re-build the same commit, the layers may be byte-identical but they may *not* — timestamps, install ordering, mirror state. Treat SHA tags as write-once.
- **Mixing the tag scheme with the *image name*.** `user-service-dev:1.4.2` is a worse pattern than `user-service:1.4.2-dev`. Tag holds variability; repository name should be stable.
- **Forgetting to push the SHA tag too.** Pushing only `:dev` after a build means the reproducible reference is local-only and disappears the moment your laptop reboots.
- **Letting the registry grow unbounded.** SHA-per-build is great until you have 10,000 tags. Day 7 mentions retention; even a "delete tags older than 30 days" rule helps.

## Key Takeaways
- Tags are mutable pointers; only digests are immutable identifiers.
- Use SHA tags for reproducibility, floating tags (`:dev`) for inner-loop convenience, semver tags for releases — often all at once.
- `:latest` is a footgun in CI and review-worthy in production.
- When debugging or rolling back, always reach for the SHA tag, not the floating one.

---
*Prerequisites: `day-5-container-registries-local-registries-docker-hub-image-tags.md`, `day-4-defending-ai-generated-changes-in-peer-review.md`.*
