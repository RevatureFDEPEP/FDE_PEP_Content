# Demo Preparation — Selection, Sequencing, What to Skip

> *Day 20: Capstone Demo Day — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Yesterday's D19 work established the demo-prep discipline — time budgeting, story arc, risk mitigation. Today is the final morning of the cohort, the last few hours before demos start, and the work is *narrower*: not "plan a demo" but "make the final selection and sequencing decisions for the demo you are about to give." This is a different kind of pressure than any earlier topic in PEP. The substrate is frozen — no more code changes, no more "let me just add one thing." What remains is editorial: which 8 minutes of the four weeks does the candidate show, and in what order, and what gets cut if they're 30 seconds late at minute 4?

This topic exists because the failure mode of capstone day isn't "the code didn't work." It's "I had 8 minutes and I tried to show 12 minutes of material, so the last third was rushed and the audience never saw the dashboard." Cuts decided in advance, under calm conditions, are very different from cuts panicked into at minute 6.

## Objective

Make final demo selection and sequencing decisions under time pressure.

## The Morning Window

Capstone demos begin after lunch. The morning is for final adjustments, *not* new construction:

- 9:00–9:30 — Cohort gathers, trainers reconfirm the demo schedule, draw slot order if not already drawn.
- 9:30–10:30 — Each candidate runs through their demo once with a peer as audience, peer holds a timer.
- 10:30–11:00 — Final cuts: each candidate decides what to remove based on peer feedback and timing data.
- 11:00–11:45 — Second run-through (solo, with timer). Confirm cuts landed.
- 11:45–12:30 — Lunch, decompress.
- 12:30 onward — Demos.

The morning has *one* job: confirm the cuts. It is not for adding polish, fixing bugs, or recording fresh backup videos (those were yesterday's work).

## The 8-Minute Live Demo Timebox

The capstone slot is 15 minutes total: **8 minutes live demo, 4 minutes architectural walkthrough, 2 minutes retrospective, 1 minute Q&A.** The live demo segment, broken down for the PEP substrate:

| Segment | Time | Cumulative | What to show |
|---|---|---|---|
| Setup + opening | 0:00–1:00 | 1:00 | "I built a quiz application. Two roles: candidate and trainer. Here's the candidate flow." Login as candidate. |
| Question authoring | 1:00–2:30 | 2:30 | Switch to trainer briefly, author one question (single MC), show it landed in the bank. *Or* show pre-seeded questions if running tight. |
| Quiz taking | 2:30–4:30 | 4:30 | Switch back to candidate, take a 3-question quiz end-to-end, submit. |
| Results | 4:30–6:00 | 6:00 | Show results page: score breakdown, chart, time-per-question. |
| Trainer dashboard | 6:00–8:00 | 8:00 | Switch to trainer, show aggregate dashboard with at least one filter applied. |

That's a tight 8 minutes. Note what's *not* in the budget:

- No CI pipeline walkthrough.
- No Docker Compose tour.
- No login form for both roles back-to-back (one role switch only, planned).
- No admin-only user management screens.
- No code on screen during the live demo segment (code goes in the 4-minute architectural walkthrough, not here).

## Selection Discipline

What earned its way into the 8 minutes:

- **All four slices appear** (W1 inherit/stabilize is implicit — the app *runs*; W2 authoring; W3 quiz-taking; W4 results + dashboard).
- **Two roles appear** (candidate and trainer).
- **One end-to-end user journey is visible** (login → take quiz → see results).

What did *not* earn its way in:

- The CI pipeline (W1's bug-fixing arc). It's foundational but invisible to a demo audience. Mention it in the architectural walkthrough or retrospective if relevant.
- The Docker Compose setup. Same reason.
- The auth-token refresh flow. Working software is implied; mechanics are a Q&A topic.
- The empty-state UIs from D19. Polish, not story.
- The role-gated route guards. Mention in passing ("trainer dashboard is server-side gated") but don't show the redirect.

The instinct is to show everything that took effort. **Effort does not earn screen time. Story does.**

## Sequencing: Plan Exactly One Role Switch

The canonical sequence for PEP is:

1. Open as candidate (login is fast, the candidate's view is the entry point of the story).
2. Switch to trainer *once*, in the question-authoring segment.
3. Switch back to candidate for quiz-taking and results.
4. Switch to trainer *once more*, for the dashboard finale.

That's two role switches, which is one more than D19's general guidance — but the capstone needs to demonstrate both roles substantively, so the second switch is earned. The discipline is *exactly two*, not three or four. If the candidate finds themselves wanting a third role switch, the story isn't tight enough — collapse the dashboard segment to "and the trainer can also see X" and don't actually toggle.

An alternative sequence skips the live question-authoring and uses pre-seeded data:

1. Open as candidate, take quiz, see results.
2. Switch to trainer once, show dashboard, *mention* "trainers also author questions; the bank you saw was authored through this UI" with a 5-second screenshot or page glimpse.

This is the safer sequence under time pressure — one role switch, no live authoring (authoring is the most failure-prone live segment because it requires keystroke accuracy under stress).

## The Cuts List

Each candidate writes a **cuts list** before lunch. Format:

```
If I'm 30s behind at minute 2:30 — cut the live question-authoring, use pre-seeded questions, mention authoring in passing.
If I'm 30s behind at minute 4:30 — show only 2 questions in the quiz, not 3.
If I'm 30s behind at minute 6:00 — skip the chart deep-dive on results, just show the page.
If I'm 30s behind at minute 7:00 — show only the unfiltered dashboard, don't apply a filter.
```

Four cut points, in order of which segment dies first under time pressure. The cohort writes their own — these are illustrative. The principle: **cuts are pre-decided. They are not improvised.**

A candidate who decides at minute 6 "I'll just skip something" *will* skip the wrong thing (typically the trainer dashboard, because it's last). A candidate who decided at 10:30 AM "if I'm late at minute 4:30, I drop the third quiz question" has already made the decision and just executes it.

## What Cannot Be Cut

Some segments are load-bearing. Cutting them collapses the story:

- **The candidate-side quiz flow.** This is the central user journey. Skipping it means the audience never sees the app work end-to-end.
- **The results page.** The chart is the visual hook; without it the demo feels textual.
- **The trainer dashboard appearance.** Even if filters are skipped, the dashboard *page* must be shown. Otherwise the second role isn't substantiated.

Skippable:

- Question authoring (collapse to seeded data).
- Chart deep-dive (just show the page).
- Filter application (just show unfiltered).
- Code highlights (those belong to the architectural walkthrough segment anyway).

Knowing the difference between load-bearing and skippable is the editorial work of this morning.

## The Final Run-Through

The morning's second run-through (11:00–11:45) is solo, with a timer, no audience. The discipline:

1. Start the timer.
2. Run the demo as if the audience were present.
3. Stop the timer at the end.
4. Record the times for each segment.

If the run lands at 7:55, the cuts list is calibrated correctly. If it lands at 9:10, one *more* cut goes in. If it lands at 7:00, the candidate has 60 seconds of slack — they can restore one cut, *or* leave the slack as a buffer (recommended).

Aim for 7:30. Sixty seconds of buffer is professional; 0 seconds of buffer is anxious.

## What This Morning Is *Not* For

- New code. The substrate is frozen.
- New backup recordings. D19 was for that.
- Fixing bugs noticed during the run-through. If a bug surfaces, *add it to the cuts list* (show the working path, skip the broken one) instead of trying to fix it.
- Reorganizing the architectural-walkthrough segment. That's a separate slot; don't conflate.
- Watching a peer's full demo for entertainment. Time is short. Watch only your own.

## The "Don't Edit During The Demo" Discipline

The most expensive class of failure: candidate notices something small during the run-through, decides to fix it, breaks something else, panics. By morning of capstone day, **the code is frozen.** If something is wrong:

- If it doesn't block the demo: ignore it. Mention it in the technical-debt segment of the retrospective.
- If it does block the demo: route around it via the cuts list, or fall back to the recorded video.

A candidate who shows up to the demo with a fresh git commit from 11:30 is a candidate who shows up to the demo with an unverified change. Don't.

## Synchronizing With The Trainer

By 11:30, each candidate confirms with their trainer:

- Their final demo slot time.
- The browser profile / window setup they'll use.
- Whether they want a peer in the audience as a "friendly face" (some find this calming, others find it distracting).
- Whether they're using the live system or starting with the backup video (yes, this is a legitimate decision — if the candidate's local Docker is wobbly, the backup recording *is* the demo, and that's fine).

The trainer notes the slot and confirms readiness. Demo day starts at 12:30.

## A Note on the Slot Order

Slot order matters for cohort dynamics, not for evaluation:

- First slots see fewer stakeholders (they're still arriving) and have less time to second-guess.
- Middle slots have the full room and the most attention.
- Last slots benefit from having watched everyone else's pacing.

Trainers usually randomize. Candidates who *strongly* prefer a slot can flag it the night before, but by 9:30 AM the schedule is locked.

## Anti-Patterns

- **Adding a new segment in the morning.** "I just realized I should also show the audit log..." No. The plan is the plan.
- **Practicing the demo more than twice.** Diminishing returns; the third run-through is anxiety, not preparation.
- **Cutting load-bearing segments.** Skipping the dashboard to make time. Don't.
- **Improvising cuts during the demo.** The whole point of a cuts list is that the decision is pre-made.
- **Lunch with the laptop open.** Eat. Talk. Decompress. The demo is at 12:30 and a panicked candidate demos worse than a calm one.
- **Showing up at 12:25 with unverified changes.** The substrate must be frozen by 11:30 at the latest.

## Connecting Back

D19 taught demo prep in general — risk mitigation, scripting, timing. Today applies that discipline under final time pressure, with the specific 15-minute capstone format and the four-slice substrate the cohort just built. The morning is editorial: cuts, sequence, pre-decided fallbacks. The afternoon is execution.

## Key Takeaways

- 8 minutes of live demo, broken into 5 segments (open, authoring, quiz, results, dashboard); plan exactly *two* role switches, not three.
- Aim for 7:30 in the final run-through; 60 seconds of buffer is professional, 0 is anxious.
- Write a cuts list with explicit triggers ("if I'm 30s behind at minute X, drop Y"). Cuts are pre-decided, not improvised.
- Load-bearing: candidate quiz flow, results page, dashboard appearance. Skippable: live authoring, filters, chart deep-dive.
- The substrate is frozen by 11:30. New code, new recordings, and bug fixes are not on today's agenda.
- Two run-throughs maximum; lunch with the laptop closed; demos start at 12:30.

---
*Prerequisites: day-19-demo-preparation-selection-sequencing-risk-mitigation, day-19-live-demo-backup-patterns-recorded-fallback.*
