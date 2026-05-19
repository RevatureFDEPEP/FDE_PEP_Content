# Demo Preparation — Selection, Sequencing, Risk Mitigation

> *Day 19: Trainer Dashboard Slice (Frontend) + Polish — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Tomorrow is capstone demo day. The cohort has built a working full-stack application across four weeks; today's task is to learn how to *demonstrate* it in 8-10 minutes to an audience that doesn't know the codebase, hasn't seen the assignment, and is judging whether the candidate is ready for the 10-week intensive. Demo prep is its own skill — not engineering, not communication-in-the-abstract, but a specific discipline of **selecting** what to show (most demos try to show too much), **sequencing** the story (the order is the difference between "interesting" and "confusing"), and **mitigating risk** (live demos *will* break; the question is whether you've planned for it). This topic gives a concrete plan template the cohort fills in today, and pairs with Topic 7's backup-recording work. Trainer assesses demo readiness EoD.

## Objective

Plan a demo sequence, identify its risks, and prepare scripted fallbacks for each.

## The Time Budget

Capstone demos are **8 minutes live + 2 minutes Q&A**. Ten minutes total. The cohort routinely tries to fit 20 minutes of material into 8 and the result is rushed, illegible, and runs over.

Concrete budget for an 8-minute demo:

| Segment | Time | Purpose |
|---|---|---|
| Opening | 0:00–0:15 | Hook + what the app does in one sentence. |
| Tour the auth + take a quiz | 0:15–2:30 | Show the candidate flow end-to-end. |
| Show the results page | 2:30–4:00 | The chart + summary, "here's what the candidate sees after." |
| Switch to trainer role | 4:00–5:30 | Login as trainer, show dashboard, demonstrate filters. |
| Code highlight | 5:30–7:00 | One or two pieces of code you're proud of: scoring engine, server components, role gate. |
| Wrap | 7:00–8:00 | What you'd build next + buffer. |

Eight minutes feels long until you start; then it's gone. Practice with a timer.

## The Opening (15 Seconds)

A demo opening that works:

> "I built a quiz-taking application where candidates take timed assessments and trainers see aggregated results across the cohort. Here's the candidate flow."

What the opening does:

- Names the app in one sentence.
- Names the *two roles* (candidate, trainer) so the audience knows there are two flows coming.
- Promises the first demo segment ("Here's the candidate flow").

What the opening *doesn't* do:

- Apologize for things that aren't done.
- Explain the assignment context.
- Recap the four weeks.
- Show a slide deck.

15 seconds. Practiced. The cohort should be able to recite the opening from memory.

## The Sequence

Demo sequencing follows a story arc, not the order you built things. A useful arc for PEP:

1. **Set up the world.** "Candidates take quizzes." (Show login, show the home page with available tests.)
2. **Show the protagonist's journey.** "Here's a candidate taking a quiz." (Walk through one quiz: select test, answer questions, submit.)
3. **Show the payoff.** "Here's what the candidate sees afterward." (Results page with the chart.)
4. **Pivot to the second role.** "Now switch to the trainer." (Logout, login as trainer, show dashboard.)
5. **Show the trainer's leverage.** "Trainers see everyone's data, with filters." (Filter by test, by date range.)
6. **Show the engineering depth.** "Here's the code I'm proud of." (Open one file, explain one decision.)
7. **Close.** "Next steps and what I'd build with more time."

The cohort should *not* sequence as:

- "Let me show you the database schema first..." (Audience tunes out.)
- "I'll start with the CI pipeline because that was Day 3..." (Boring; audience doesn't care about chronology.)
- "Let me explain the architecture..." (Architecture is the *substrate*, not the *story*.)

The user-visible flow is the story. Architecture is mentioned in passing during the code-highlight segment, not the opening.

## Selection: What To Show, What To Skip

Show:

- **Auth flow.** Quick login as a candidate, no more than 10 seconds. Don't dwell on the form.
- **One quiz taken end-to-end.** Three questions, not ten. Use seeded short questions.
- **The chart on the results page.** This is visual and impressive.
- **The trainer dashboard with at least one filter applied.** Show the filtered view, comment on the URL changing.
- **One code highlight.** One. Not three.

Skip:

- **The docker-compose setup, K8s, CI pipeline.** Useful in interviews; not in a demo.
- **Anything that takes >5 seconds to load.** The audience's attention budget is small.
- **Long copy.** "Here's a page with a list of all the candidates who have ever taken any test in the system..." → "Here's the dashboard."
- **Error states.** Show them if asked in Q&A; don't show them proactively. They imply something's wrong.
- **The admin panel for editing questions.** Not load-bearing for the demo story; cut it.

The cohort's instinct is to show everything they built. Resist. **A demo is a curated narrative, not a feature checklist.**

## Sequencing: Don't Toggle Roles More Than Once

A common mistake: candidate → trainer → candidate → trainer to "compare views." Every role toggle costs 10-15 seconds (logout, login form, navigation) and disorients the audience. Plan exactly *one* role switch in the middle and stay in each role for the full segment.

If the cohort feels they need to show "the same data from two angles," show it from the trainer's angle and *describe* the candidate's view in passing. The audience doesn't need to see both — they need to understand both.

## Risk Mitigation

A live demo *will* have something go wrong. Common failure modes for PEP:

| Failure | Likelihood | Mitigation |
|---|---|---|
| Backend container died overnight | Medium | Run `docker-compose ps` 10 min before demo; restart if needed. |
| Database is empty / candidate has no attempts | High | Seed data before demo; verify the candidate user has visible attempts. |
| Wifi flaky | Medium | Local Docker; ensure backend hits `localhost`, not a remote API. |
| JWT expired during the demo | Low | Log in fresh right before starting. |
| Chart renders blank | Low | Test the exact session that will be shown, today. |
| Browser cache shows stale data | Low | Hard-refresh once before starting; close devtools. |
| Forgot the trainer password | Medium | Write it on a sticky note next to the laptop. |
| Live coding breaks the app | High | **Don't live code.** Code highlights are *reading* code, not editing it. |
| Sharing screen shows messy desktop | Medium | Quit Slack, Discord, calendar reminders. Use a clean browser profile. |
| Time runs out before the trainer dashboard | High | Practice with a timer; cut content if you're 30s slow at minute 5. |

The cohort fills in this table for their own demo. Each row needs a *plan*, not just an acknowledgment.

## The "Three Risks" Discipline

Rank the top three risks for *this specific* demo:

1. _____ (most likely thing to go wrong)
2. _____ (second most likely)
3. _____ (third)

For each, write the *exact action* you take if it happens. Two examples:

- **Risk: "Backend doesn't start."** Action: "Switch to the recorded backup (Topic 7), narrate over it." (No fumbling in the live system trying to debug.)
- **Risk: "Filter on dashboard returns empty due to date issue."** Action: "Clear filters, narrate that the unfiltered view is what I'd usually show." (Don't try to fix the filter live.)

The discipline is to *plan* the fallback, not to improvise it. Improvisation under demo pressure looks bad. Planned graceful degradation looks competent.

## The Skip-If-Broken Rule

Identify, before the demo, the segments that *could* be skipped if something breaks:

- Code highlight is skippable; the demo is fine without it.
- The trainer dashboard is *not* skippable; it's the second half of the story.
- The chart on the results page is *not* skippable; it's the visual hook.
- Showing more than three questions in the quiz *is* skippable; cut to "I'll submit now."

Know in advance which segments you can drop. If you're running over at minute 4, the code highlight goes. If the backend is wobbly, the filter demo collapses to "and here's the unfiltered view."

## Scripting Without Sounding Scripted

A demo script is *bullet points*, not paragraphs:

```
0:00 - "I built a quiz app. Two roles: candidate, trainer. Here's the candidate flow."
0:15 - Login as candidate. "I'm logging in."
0:30 - Click "JavaScript Fundamentals." "Three questions, two minutes."
0:45 - Answer Q1. "Multiple choice."
1:00 - Answer Q2. "Code completion."
1:15 - Answer Q3, submit. "Submitting now."
1:30 - Results page. "Here's the breakdown. Chart shows time per question, color is correctness."
...
```

Bullets, not sentences. The cohort speaks in their own words during the demo; the bullets are the safety net.

Read the script aloud once. Time it. Adjust.

## Practice

Two practice runs minimum:

1. **First run** with the script open on a second screen. Where do you stumble? Where does time pile up?
2. **Second run** with no script visible. Does it flow? If not, simplify the script.

The cohort that does *zero* practice runs is the cohort that runs over time, fumbles the login, and forgets the dashboard segment. The cohort that does two is the cohort that hits 7:55.

## EoD Trainer Assessment

The trainer should sit with each candidate for ~5 minutes and watch them run through their demo plan once. Assessment criteria:

- Did they hit time (within ±30s)?
- Did the opening land in 15 seconds?
- Did they pick the right three things to show?
- Do they have a fallback for the top risk?
- Did they avoid live coding?

A "ready for demo day" stamp from the trainer is the deliverable for this topic.

## Anti-Patterns

- **Live coding during a demo.** Even a one-line edit. Demos are for showing finished work.
- **Apologizing.** "I didn't have time to..." undermines the audience's confidence. Don't.
- **Reading from the terminal.** If the demo includes a terminal, the terminal output should be pre-rehearsed and short.
- **Talking over loading spinners.** Plan a sentence to say during each ~2-second pause. "While that loads, let me mention that the dashboard is filtered server-side..."
- **Speeding up when you realize you're behind.** The audience hears the panic. Cut a segment instead; finish on time at a normal pace.
- **Showing the file tree in VS Code as "the architecture."** The file tree is not the story. The user flow is the story.
- **Inviting the audience to "ask anything."** They will. They will derail. Reserve Q&A to after the wrap.

## Connecting Back To The Full Curriculum

The four weeks of work are the substrate. The demo is the *interface* between that work and the people evaluating it. A candidate who built solid code but can't demo it well will be misjudged; a candidate who builds modest code but demos it brilliantly will be overrated. The demo is the cohort's chance to *calibrate the world's view of their work* — and it's a skill worth treating as deliberately as any of the engineering topics across the four weeks.

The 10-week intensive will require demos at the end of multiple sprints. Practicing one well today is leverage for the next ten weeks of professional life.

## Key Takeaways

- 8 minutes of demo, 2 minutes of Q&A. Plan to the budget; practice with a timer.
- Open in 15 seconds with a one-sentence app description. No apologies, no slide deck.
- Sequence as a story (candidate flow → results → trainer flow → code highlight), not as a feature list or architecture tour.
- Select aggressively: one role switch, one code highlight, three quiz questions. Cut the rest.
- Rank the top three risks and plan the *exact action* if each fires. Improvisation under demo pressure looks bad; planned degradation looks competent.
- Don't live code. Don't apologize. Don't toggle roles more than once.
- Two practice runs minimum; trainer signs off on demo readiness EoD.

---
*Prerequisites: day-15-slice-integration-and-end-to-end-validation, day-19-state-coverage-ui-empty-loading-error.*
