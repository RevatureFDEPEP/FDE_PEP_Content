# Retrospective and Continuous Improvement Practice

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D10 introduced the retro habit at the end of Week 2. Today we run the Week 3 retro — and Week 3 is, by design, the *hardest* week of PEP. Backend transactions, pessimistic locks, idempotency keys, server-anchored timers, autosave races, optimistic vs. confirmed updates, double-submit defenses, and finally the integration day with three services in flight. The cohort will have war stories. The retro converts those stories into Week 4 improvements before they're forgotten.

The format and discipline from D10 still apply — pick start/stop/continue or 4Ls, time-box hard, two-to-three action items with owners and dates, review last week's actions first. What's different today is the *content* of what's likely to surface, and how to channel that into concrete W4 changes.

## Open With Last Week's Actions

Same opener as D10 prescribed. Walk through each of Week 2's action items in 30 seconds:

- Did it happen?
- Did it help?
- Carry over, drop, or revise?

If the cohort committed to "bake MinIO CORS into the Compose init script" and nobody did it, surface that — either someone owns it this week, or the cohort drops it explicitly. Lingering uncommitted actions are how retros decay into theater.

## Week 3 Candidate Issues (Use As Priming, Not Substitution)

These are the specific high-rigor issues Week 3 produces. Don't read them out at the start; if the cohort goes silent after the first five minutes, surface one to unstick conversation. Otherwise, let the cohort generate first.

### Backend rigor pain (D11–D12)

- **Idempotency-key misses.** "I thought I added the constraint, but my second submit went through and created a duplicate scoring run." The unique index was on the wrong column tuple, or the migration didn't run, or the code path bypassed the check.
  - *Likely action:* add an integration test (Topic 6) that asserts a duplicate `Idempotency-Key` returns the cached response, not a fresh one. Should have been there.

- **Lock acquired too late.** "I `select`ed first, then `select for update`ed — race window between them." A class of bug that's invisible in dev (no concurrency) and lethal in production.
  - *Likely action:* document "lock before read for any write path" in `CONTRIBUTING.md`; flag in code review (Topic 7 checklist already includes it).

- **Transaction boundary too wide.** "I held the row lock across a 3-second Mongo lookup; second submit waited 3 seconds." The lock should bracket the *mutation*, not the entire handler.
  - *Likely action:* prefetch reads, *then* open the transaction, *then* lock, *then* mutate, *then* commit.

- **`select for update` deadlocks.** Two handlers locking session rows in different orders. Postgres raises a deadlock error; both rolled back.
  - *Likely action:* always lock in a canonical order (e.g., session row first, then attempts); document it.

### Frontend rigor pain (D13–D14)

- **Timer drift.** "My timer said 4:32 but the server thought the session had expired 30 seconds ago." Probably the server-anchored anchor wasn't re-fetched on resume, or the client clock disagrees with the server clock.
  - *Likely action:* fetch `server_now` on every page load and on resume from background tab; never trust `Date.now()` alone for the source of truth.

- **Autosave races against submit.** "I clicked submit; the autosave for the last question fired after; the server saw answer-after-submit; got 409; UI confused." The D14 pattern said wait for in-flight autosaves before submit; that wait wasn't actually implemented.
  - *Likely action:* explicit `await flushPendingAutosaves()` in the submit handler; add Vitest assertion that submit blocks until autosave promise resolves.

- **Optimistic-update inversion.** "I optimistically marked submit as 'done' before server confirmed; server rejected; UI lied." Submit is *confirmed* not *optimistic*; D14 made the distinction explicit but it's easy to forget.
  - *Likely action:* type-system enforcement — submit state machine doesn't expose an `optimistic` state; only `submitting` and `submitted`.

- **Double-click double-submit despite the defense.** "I disabled the button on click, but somehow two requests went out." Probably React batching: the disable state updated *after* the second click landed. Or no `Idempotency-Key` so the server processed both.
  - *Likely action:* generate `Idempotency-Key` once per attempt at submit-state entry; reuse across retries; never let the server be the only line of defense.

### Integration rigor pain (D15)

- **`X-Request-Id` rotted somewhere.** "I tried to follow a failed submit through three services and lost the id at the user-service boundary." Middleware order changed.
  - *Likely action:* assertion in smoke test (already added in Topic 2, step 8); CI fails if correlation rots.

- **Compose-up sequencing surprises.** "Brought up `web` first and got a wall of 502s." Healthcheck `depends_on` not set on the new dependency.
  - *Likely action:* trainer-led review of the Compose file at start of W4; add the missing healthcheck conditions.

- **Tests passing locally, failing in CI.** "My integration test worked on my machine; CI says Postgres connection refused." Almost always: the CI workflow isn't waiting for Postgres healthcheck before running tests.
  - *Likely action:* extract the "wait for healthy" loop into a reusable composite action; use it everywhere.

### Process pain

- **PR queue depth on Wednesday.** "Three slice PRs landed at once; review took 4 hours; nobody got code-review on their own work until end of day." The 25-cohort PR-triage mitigation from the ops doc was supposed to handle this.
  - *Likely action:* enforce the morning review window more aggressively; if the queue is over 5, the trainer rotates an extra reviewer.

- **Knowledge silos forming.** "Only one person in my pair knows how the row lock works; the other person just nodded." A slice with this much complexity rewards pairing on the *understanding*, not just the code.
  - *Likely action:* mandate that the *non-author* of a backend PR walks through the lock logic in the review; verifies they actually got it.

- **Documentation lag.** "The `Idempotency-Key` header convention isn't written anywhere; new code adds it inconsistently."
  - *Likely action:* `docs/conventions.md` with one paragraph per cross-cutting convention; owner per convention.

## What's Likely To Get The Most Votes

In dot-voting, expect Week 3 retros to cluster around:

1. **Some form of timer or autosave instability.** It's the most-cursed-at feature of the week.
2. **The integration day's surprises.** Specifically the gap between "my unit tests pass" and "the slice works end-to-end" was wider for some trainees than others.
3. **PR review timing or depth.** Always a Week-3-or-later concern.

Don't *steer* the vote — but if these don't surface organically, ask a clarifying question to invite them.

## Action Items: Cap At Three

Same discipline as D10. Two or three concrete, owned, dated actions. Examples that would meet the bar:

- "Add an integration test asserting `Idempotency-Key` duplicates return the cached response. Owner: A. Done by Wed of W4."
- "Document the lock-before-read convention in `CONTRIBUTING.md` with the deadlock-avoidance ordering. Owner: B. Done by Mon of W4."
- "Extract CI 'wait for healthy' into a composite action and use it in three workflows. Owner: C. Done by Wed of W4."

Pin them in the cohort channel. Open the W4 retro with them.

## Things Worth Continuing (Don't Forget The Positive)

The retro isn't only about what hurt. Name the things that worked so they stick:

- **The smoke + Playwright + integration test triad** (Topics 2, 3, 6) closing the testing gap that haunted W2.
- **Distributed log analysis discipline** (Topic 4) actually saving time when slice debugging — the cohort can localize failures in minutes instead of guessing.
- **D14's autosave UX** once it stabilized — candidates won't lose work mid-quiz; trainers won't field "did my answer save?" questions during demos.
- **`trainer/reference` Week 3** branch as the canonical demo target.

If something worked, *say so*. Otherwise the team only hears about pain and loses morale.

## The Specific W3 Risk: Burn-Out Going Into W4

Week 3 is rigor-heavy and intellectually dense; Week 4 is *output-heavy* with the results slice, trainer dashboard, and the capstone. Cohorts often arrive at Friday of W3 exhausted. The retro should explicitly ask:

- "What energy do you have for W4?"
- "Is there anything in the W3 grind that we should *stop* doing in W4 because it's costing more than it's giving?"

If the cohort says "the Wednesday 1-hour deep-dive on transactions was great but I'm spent" — that's a signal worth acting on, even if it's just "no deep-dives W4 Mondays."

## Facilitator Notes

D10's facilitator rules still apply (trainer goes last, name patterns not people, ruthlessly dot-vote, action items only). Two W3-specific additions:

- **Pre-write the candidate-issues list (above), but keep it hidden.** If the room is fluent, never surface it. If the room is silent at minute 7, surface one item from it as a starter — but only one.
- **Watch for the "we should have tested that" pattern.** It's the most common W3 regret. When it surfaces three or more times, that's the action item: "Where in our workflow does 'we should have tested it' keep happening, and what would catch it earlier?" Often the answer is "integration tests" or "smoke in CI" — and those are concrete actions.

## Today's Specific Output

By end of the W3 retro, captured:

1. **3 things to continue** that emerged this week (the testing triad? log discipline? autosave UX?).
2. **2–3 action items** for Week 4 with owners and dates (likely some mix of testing, documentation, and process).
3. **1 thing to revisit at next retro** if the action doesn't land.

Pin to the cohort channel. Reference at the start of Day 16.

## Key Takeaways
- Week 3 is the hardest week; the retro converts the pain into Week 4 actions before the lessons are forgotten.
- Open with the W2 action items — closed loops are the difference between retro-as-habit and retro-as-theater.
- Expect candidates to name: timer drift, idempotency misses, autosave races, integration-day surprises, PR review timing.
- Cap at 2–3 action items; concrete, owned, dated.
- Name what worked (testing triad, log discipline) explicitly — morale matters going into the W4 sprint.
- Watch for burnout; W4 is output-heavy, so trim non-essential cognitive load.

---
*Prerequisites: day-5-week-1-retrospective-and-continuous-improvement-practice, day-10-retrospective-and-continuous-improvement-practice, the rigor of Days 11–15.*
