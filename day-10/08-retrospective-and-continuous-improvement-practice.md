# Retrospective and Continuous Improvement Practice

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Week 2 ends with a retrospective. The skill isn't running a meeting with a particular template — it's the *habit* of pausing after a stretch of work to ask "what should we change before we do this again." Done well, retros surface the friction that's invisible day-to-day (CI flakes, MinIO CORS pain, unclear handoffs) and convert it into concrete experiments for next week. Done poorly, they're a venting session that changes nothing. Today we run a real Week 2 retro on the cohort's own work and walk away with a small number of action items, owners, and dates.

## Why Retro, and Why Now

The cohort has just finished two weeks of intense work — Week 1 stabilized the inherited stack, Week 2 added the first vertical slice from authoring backend through frontend through integration. Patterns have emerged:

- What worked and should be reinforced (e.g., the trainer/reference branch).
- What caused unnecessary pain (e.g., debugging MinIO CORS for 90 minutes).
- What was unclear that should be clarified (e.g., when to use a Server Action vs client `fetch`).
- What the cohort wants to try differently in Weeks 3–4.

Surfacing these now — while the experience is fresh and Week 3 hasn't started — is the cheapest moment to act on them.

## Two Format Options

Use one. Both work; pick by mood.

### Format A: Start / Stop / Continue (10–15 min)

Three columns, sticky-note style (Miro, FigJam, a shared doc, or actual stickies):

- **Start** — things we aren't doing that we should.
- **Stop** — things we're doing that hurt.
- **Continue** — things that worked, name them so they stick.

Quick to run, easy to compare across weeks. Best when the cohort is tired and you want concrete output.

### Format B: 4Ls — Liked / Learned / Lacked / Longed For (20–25 min)

Four columns:

- **Liked** — what was satisfying or fun?
- **Learned** — what new understanding came out of the week?
- **Lacked** — what was missing? (tools, knowledge, time, clarity)
- **Longed for** — what would have made a real difference?

Richer than start/stop/continue. Better when there's energy to reflect and trainers want texture, not just todos.

## The Run Sheet (25 minutes)

For a cohort of 25, time-box hard. The trainer is the facilitator, not a contributor.

| Time | Activity |
|---|---|
| 0–2 | Trainer frames the retro: scope (Weeks 1–2), goal (find changes for W3/4), rules (no blame, name patterns not people). |
| 2–9 | **Silent generation.** Everyone writes notes into the chosen format in a shared doc. No talking. |
| 9–17 | **Group + cluster.** Trainer reads notes aloud, groups duplicates, asks clarifying questions only. |
| 17–22 | **Dot vote.** Each person gets 3 votes to put on the items they care most about. |
| 22–25 | **Action items.** Top 2–3 voted items get an owner and a "by when." Write them down. |

Twenty-five minutes is enough for a focused retro. More and people zone out; less and you skip the action items.

## Example Candidate Issues from Week 2

To make the retro concrete — these are issues that *typically* show up in cohorts running this curriculum. Use them as priming examples if the cohort goes silent, but let the cohort generate their own first.

**Start candidates:**
- Pair on Compose changes — they bite when one person edits in isolation.
- Write a one-line "what I'm stuck on" in the channel before opening a help thread.
- Capture every "gotcha" in a shared `CHEATSHEET.md` so the second time costs less.

**Stop candidates:**
- Force-pushing to shared branches without warning the team.
- Debugging in the browser console without checking server logs first.
- Submitting PRs with `console.log` / `print` statements left in.

**Continue candidates:**
- Daily standups stayed at 10 minutes.
- The `trainer/reference` branch as a known-good baseline.
- Reviewing each other's PRs before asking the trainer.

**Lacked / Pain points commonly named:**
- *CI flakes* — intermittent Postgres healthcheck timeout in CI; never reproducible locally. (Action: bump healthcheck retries; if still flaky, file an issue with logs from three flaky runs.)
- *MinIO CORS pain* — Day 9's CORS setup took longer than the rest of the upload work. (Action: bake the CORS policy into a Compose init script so it applies on `compose up`.)
- *Contract drift between zod and Pydantic schemas* — kept getting bitten by camelCase vs snake_case. (Action: document the on-the-wire convention in `CONTRIBUTING.md`; consider generating zod from OpenAPI in Week 3.)
- *422 errors disappearing into root.serverError* — the `loc.body` strip was undocumented. (Action: comment it in `applyApiErrorsToForm`; add a test.)
- *Long PR review queues on Friday afternoons.* (Action: review-window in mornings; nothing merged after 3pm Friday.)
- *Slow `compose up` cycle on Windows.* (Action: build a minimal compose profile for the slice you're working on; full stack only when needed.)

## Action Items: What Makes Them Real

A retro produces *action items*, not just lists. An action item has three properties:

1. **Specific** — "improve CI" is not actionable; "bump Postgres healthcheck retries from 5 to 10 and re-run the last three flaky branches to verify" is.
2. **Owned** — exactly one name, not a team. Owner can delegate; owner is accountable for outcome.
3. **Time-boxed** — has a date (typically "by end of next week"). No date = no action.

Write the three (or fewer) action items in a place the team will see them — pinned in Slack, in the cohort wiki, in the retro doc that's reviewed at the *next* retro.

## The Critical Habit: Reviewing Last Week's Actions First

A retro that ignores last week's commitments is theater. Open the retro with **30 seconds per action item from last week**:

- "Did it happen?"
- "Did it help?"
- "Carry over, drop, or revise?"

This is what converts retros from venting into a feedback loop. If the same issue shows up three retros in a row with no action taken, that's a signal worth naming.

## Facilitator Anti-Patterns

- **Trainer talks first.** Suppresses cohort voices. Trainer goes last, if at all.
- **Naming individuals.** "X broke main" turns the room defensive. Reframe to "main broke; here's what would prevent it." Patterns, not people.
- **Letting one issue dominate.** Use the dot vote ruthlessly; not every issue gets airtime.
- **Generating 20 action items.** Nobody owns 20 things. Three is the cap.
- **No follow-up.** The retro doc is read once and forgotten. Pin it; revisit at next retro.

## Process Anti-Patterns

- **Retro every day.** Becomes ritual, generates no signal. Once per week is plenty for a cohort.
- **Retro right after a frustrating event.** Tempting but produces lists of complaints, not actions. Schedule it as a standing slot.
- **Retro with no time-box.** Expands to fill available time, then runs out of energy at the action-item step.
- **Action items with no owner.** They simply don't happen.
- **Treating the retro as performance evaluation.** Different meeting, different room, different time.

## Continuous Improvement Beyond the Meeting

The retro is the structured moment. The disposition is continuous:

- When something hurts, name it in the moment (in the channel, in the PR). The retro just collects what's already been surfaced.
- Try small experiments week-over-week; keep the ones that help, drop the ones that don't.
- Track which experiments worked; build a team playbook of "things we've learned the hard way."

In a working team, the retro mostly confirms what people already knew was up. In a struggling team, it surfaces what nobody felt safe saying. Both outcomes are valuable.

## Today's Specific Output

By end of Day 10's retro, capture:

1. **3 things to continue** (so they keep happening).
2. **2 action items** for Week 3 with owners and dates.
3. **1 thing to revisit at next retro** if the action doesn't land.

Pin the result to the cohort's home channel. Reference it at the start of Day 11.

## Key Takeaways
- Retro is a habit, not a template — pick a format (start/stop/continue or 4Ls) and run it consistently.
- Time-box hard: 25 minutes is enough for 25 people with discipline.
- Output is **2–3 action items with owners and dates** — anything else is theater.
- Open every retro by reviewing last week's actions; otherwise the loop never closes.
- Patterns, not people; trainer facilitates, doesn't dominate.

---
*Prerequisites: Week 1 review (day-5), Week 2 vertical slice (day-8 through day-10).*
