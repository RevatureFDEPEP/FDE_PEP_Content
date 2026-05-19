# Storytelling Around Technical Work (What / Why / How)

> *Day 20: Capstone Demo Day — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

A demo isn't "look, it works." The audience knows it works — they can see the screen. What they want, what they remember, and what they evaluate on is the *story* around the technical work. A candidate who can narrate "I built X, because Y, via Z" reads as senior. A candidate who can only narrate "I built X" reads as junior, regardless of the underlying code quality. This topic gives the cohort a framework — **what / why / how**, in that order — and several worked examples drawn directly from the four-week substrate they just built.

The framework matters because under demo pressure most candidates default to *how*-first storytelling ("I used Postgres `SELECT FOR UPDATE` to lock the row..."). The audience that doesn't already understand the problem hears jargon. The candidate who opens with *what* and *why* before *how* gives the audience a place to stand. The same technical content, told in the right order, is the difference between "interesting" and "lost."

## Objective

Narrate technical work in a what / why / how structure.

## The Framework

For any piece of technical work in the demo or the architectural walkthrough:

- **What** — a one-sentence description of the thing built, in plain English. No jargon, no implementation details.
- **Why** — the problem it solves or the design constraint it satisfies. Tied to a real user scenario or system property the audience cares about.
- **How** — the technical mechanism. This is where jargon is allowed because the audience is now oriented.

The order is **what, why, how**. Almost no one defaults to this order; engineers default to *how, what, why* (jumping into implementation, then backfilling the description, then justifying). Reversing the habit is most of the work.

## Worked Example 1: The Server-Anchored Timer (W3, D13–D14)

How the cohort built it: client receives `server_now` and `deadline_at` from the backend, computes the offset once, then ticks the visible clock locally without trusting `Date.now()` as a source of truth.

A *bad* narration of this in a demo (how-first):

> "So I have this `useEffect` hook that computes an offset between `server_now` and `Date.now()`, and then there's a `setInterval` that recomputes the remaining time every second based on that offset, and..."

The audience tunes out at "useEffect." They never learn what problem this solves.

A *good* narration (what / why / how):

> **What:** "The quiz timer you saw counting down is server-anchored — the server decides when time is up, not the browser."
> **Why:** "Client clocks drift, and a candidate could change their system clock to get more time. The quiz is graded server-side, so the timer has to be the server's truth, not the browser's."
> **How:** "On page load, the client captures a one-time offset between server and local clock, then ticks locally against that offset. The server independently rejects submissions past the deadline — the visible timer is a UI hint, not the enforcement."

Three sentences. The audience now knows what they're looking at, why it's not trivial, and broadly how it works. The candidate can follow up with code in the architectural walkthrough segment.

## Worked Example 2: Idempotent Quiz Submission (W3, D12)

> **What:** "When a candidate submits a quiz, the system guarantees that even if the network drops and they click submit twice, the score is only computed once."
> **Why:** "Double-submits happen — flaky wifi, anxious clicks. Without protection, the candidate would see duplicate attempts in their history and we'd have to manually deduplicate. Worse, a scoring engine that ran twice could race against itself if two responses landed simultaneously."
> **How:** "The client generates an idempotency key per submission attempt. The backend stores it with the result; subsequent requests with the same key return the cached response. A unique index on the key enforces it at the database level so two simultaneous requests can't both win."

## Worked Example 3: The Trainer Dashboard's Server-Side Filtering (W4, D18)

> **What:** "The trainer dashboard filters attempts by test, candidate, and date range — and the filtering happens on the server, not in the browser."
> **Why:** "A cohort of 25 candidates taking 20 tests over 4 weeks is small enough to ship to the browser today. But the design needs to scale to multi-cohort views with thousands of attempts, and pushing that down the wire on every filter change is a problem we'd rather not inherit."
> **How:** "Filter state is serialized into URL query params, the Next.js server component reads them, and the API call to the backend includes the filters as query parameters. The backend translates them to SQL `WHERE` clauses with appropriate indexes."

Notice that the *why* names a current scale ("25 candidates, 20 tests") and a future scale ("multi-cohort, thousands of attempts"). That's a senior-coded move — showing awareness that today's working solution and tomorrow's working solution are different, and that the design accommodates both.

## Worked Example 4: The CI Pipeline Bugs (W1, D3–D4)

The W1 bug-fixing arc is hard to demo visually because there's no UI for it. But it's worth narrating in the architectural walkthrough or the retrospective:

> **What:** "The CI pipeline I inherited had five seeded bugs. By day 4, all five were fixed and the pipeline was green."
> **Why:** "The pipeline gates every PR. A broken pipeline means either no one merges, or everyone bypasses the safety net. Both are worse than no pipeline."
> **How:** "I bisected by reading the workflow YAML against the actual failure logs. Four were configuration mismatches — wrong service names, wrong env vars, wrong Python version. The fifth was a missing test fixture that worked locally because Docker had cached state. I wrote a regression test for that one so it can't silently rot again."

The *how* here is a process narrative, not a code walkthrough. That's appropriate — bug fixing is a methodical practice, and saying "I bisected by reading the workflow YAML against the actual failure logs" communicates the practice better than showing diffs.

## Worked Example 5: The Question-Authoring Mongo Schema (W2, D8)

> **What:** "Questions are stored in MongoDB, not Postgres. The other domain data is in Postgres."
> **Why:** "Question content is schema-loose — multiple-choice, code-completion, true/false, free-response, more types coming. Modeling that in Postgres would mean either a sparse table with lots of nullable columns or a JSONB column doing the same job. Mongo's document model fits naturally."
> **How:** "Each question is a document with a `type` discriminator and a type-specific payload. The backend uses Pydantic discriminated unions to validate at the boundary. The Postgres side holds the *attempts* — relational data that benefits from FKs and transactions."

This narration handles a classic interview question — "why did you pick this database?" — in three sentences. The *why* names the actual constraint (schema-loose, growing variety) and the *how* describes the boundary discipline (Pydantic discriminators).

## The Common Failure Modes

### Failure 1: Skipping *what*

"So the reason I used `SELECT FOR UPDATE` is..." — the candidate has assumed the audience knows what's locked, why, and what's being protected. Open with *what* always.

### Failure 2: Conflating *what* and *how*

"I built a Redis-backed cache..." — "Redis-backed" is implementation. *What* is "I built a cache for the dashboard query." *How* is "backed by Redis." Separating them lets the audience evaluate the *decision* (did caching make sense?) without getting tangled in the *mechanism*.

### Failure 3: Hand-waving *why*

"Because performance" or "because best practice" or "because that's what we do" — none of these answer *why*. The *why* should name a concrete user scenario, system property, or constraint. "Because dashboard queries with five-way joins hit 800ms p95 and the user-perceived target is 200ms" is a *why*. "Because performance" is not.

### Failure 4: Drowning in *how*

"And then the SQL uses a CTE to compute the per-test rollup, and then we join that to the candidate dimension, and then we apply the date filter via a recursive subquery..." — when the *how* runs longer than the *what* and *why* combined, the audience is lost. Keep the *how* to 1-3 sentences in the live demo; the architectural walkthrough is where *how* gets more time.

### Failure 5: Telling the chronology, not the story

"First I tried X, then I realized Y was better, then I switched to Z..." — chronology is for retrospectives. For demos, jump to the final design and narrate *that* as what / why / how. The dead ends go in the retro segment, not the live narration.

## A Template for the Cohort

For each piece of work they'll narrate in the demo, candidates fill in:

```
WHAT (one sentence, plain English, no jargon):
___

WHY (one sentence; what problem, scenario, or constraint):
___

HOW (1-3 sentences; mechanism, allowing jargon now):
___
```

Done for: the timer, the idempotency, the dashboard, one CI fix, one schema decision. Five filled templates. The cohort doesn't read these off the page during the demo — they're rehearsal scaffolding. By demo time the cadence is internalized.

## Story Beats Across the 8-Minute Demo

The narration weaves through the demo, attached to what's on screen:

| Demo segment | Narration beat (what / why / how) |
|---|---|
| Open | *What*: "Quiz app, two roles." *Why*: "Candidates take timed assessments; trainers see aggregate performance." *How*: (implicit; "here's the candidate flow"). |
| Quiz taking | *What*: "Timer is server-anchored." *Why*: "Client clocks drift; grading is server-side truth." *How*: "One-time offset capture." |
| Results | *What*: "Per-question breakdown with a chart." *Why*: "Candidates see *which* answers were wrong, not just a score, so the next attempt is informed." *How*: "Server returns per-question results; frontend renders with Recharts." |
| Dashboard | *What*: "Filtered server-side." *Why*: "Scales past the current cohort." *How*: "Filters serialize to URL params, server component reads them, backend SQL `WHERE`." |

Four beats. Each beat is delivered while the relevant screen is visible. The candidate is not narrating *over* the demo; they're narrating *with* the demo.

## Practice Discipline

Two rounds in the morning window:

1. **Read aloud** with the templates open. Time it. Note where the narration runs long.
2. **Speak from memory** with no templates. Where do you fall back to *how*-first? That's the spot to compress.

The cohort that practices the narration separately from the click-through will deliver a much cleaner demo than the cohort that practices only the clicks.

## Anti-Patterns

- **Reading the templates word-for-word during the demo.** Sounds robotic. Templates are scaffolding, not script.
- **Apologizing for the *why*.** "I know caching is overkill for 25 users, but..." — name the constraint and own the choice. Don't pre-emptively undercut yourself.
- **Skipping the *why* to "save time."** The *why* is the part the audience remembers. Cut *how*, not *why*.
- **Promising "I can go deeper if you want" repeatedly.** Once is fine. Three times reads as fishing for engagement.
- **Saying "obviously" or "of course."** What's obvious to the candidate may not be obvious to the audience. Don't.

## Connecting Back

This framework will be reused across the 10-week intensive, in every demo, sprint review, and stakeholder conversation. Senior engineers narrate work in *what / why / how* without thinking about it. Practicing it deliberately during PEP capstone is the first deliberate rep.

## Key Takeaways

- The order is *what / why / how*. Engineers default to *how, what, why* and lose the audience.
- *What* is plain English, one sentence, no jargon. *Why* names a concrete constraint or scenario. *How* is mechanism, jargon allowed.
- Five templates filled in the morning window: timer, idempotency, dashboard, CI fix, schema decision.
- Common failures: skipping *what*, hand-waving *why* ("because performance"), drowning in *how*, narrating chronology instead of the final design.
- The narration weaves through the demo — one *what / why / how* beat per major segment, delivered while the relevant screen is visible.
- Practice the narration separately from the clicks. The cohort that does only click-throughs delivers click-throughs; the cohort that drills the narration delivers stories.

---
*Prerequisites: day-20-demo-preparation-selection-sequencing-what-to-skip.*
