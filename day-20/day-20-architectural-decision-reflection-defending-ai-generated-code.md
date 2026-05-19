# Architectural Decision Reflection — Including Defending AI-Generated Code in the Walkthrough

> *Day 20: Capstone Demo Day — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*
>
> **AI Tooling Thread — Closing Entry**

## Overview

The 15-minute capstone slot's middle block is **4 minutes of architectural decision walkthrough**: one non-trivial architectural choice, explained and defended in front of stakeholders. This is the most consequential 4 minutes of the day. The live demo proves the app works; the architectural walkthrough proves the candidate *understands what they built*. A working app with no architectural narrative reads as "got lucky." A weaker app with a sharp architectural narrative reads as "knows what they're doing." Stakeholders evaluate the difference.

This topic also closes the Unit 0 AI tooling thread that's run across all twenty days. Claude Code drafted real code in this cohort — likely in the scoring engine, the timer, the dashboard SQL, somewhere. The closing discipline is unambiguous: **if Claude Code drafted it, the candidate still owns and defends it.** "I had Claude write it" is not an answer. "Claude drafted the first pass; I considered X; chose Y because Z; the trade-off is W" is the answer.

## Objective

Defend an architectural decision (including AI-assisted code) in a stakeholder setting.

## What Counts As An Architectural Decision

Not every line of code is architectural. The candidate picks *one* decision that satisfies all three:

1. **There was a real choice.** Alternatives existed. The decision was not forced by the framework, the substrate, or a one-line stack overflow snippet.
2. **The choice has consequences.** It affects behavior, performance, scale, or future flexibility in a measurable way.
3. **The candidate can explain the alternatives.** Not just "I did X." Also "I could have done Y or Z, and here's why I didn't."

PEP candidates from the four-week substrate have plenty to choose from. Good candidates:

- **Server-anchored timer** (D13–D14). Alternative: trust client clock. Alternative: poll the server every second for remaining time.
- **Idempotency keys for submission** (D12). Alternative: rely on client-side double-submit prevention. Alternative: server-side dedup by `(user_id, test_id, recent timestamp)` heuristic.
- **MongoDB for questions, Postgres for attempts** (D8). Alternative: all-Postgres with JSONB. Alternative: all-Mongo.
- **Server-side filtering for the dashboard** (D18). Alternative: ship all rows to the client, filter in-browser.
- **Pessimistic row locking on scoring** (D12). Alternative: optimistic concurrency with version columns. Alternative: a job queue with a single consumer.
- **JWT in httpOnly cookies vs. localStorage** (D8/D9 user-service work). Alternative: opaque session tokens in Redis.
- **CSR vs. SSR boundary for the trainer dashboard** (D19). Alternative: all client-rendered.

Weaker candidate choices (avoid for the 4-minute slot):

- "I used Tailwind for styling." Not an architectural choice in this codebase.
- "I used Pydantic for validation." Default of the stack; not a decision.
- "I used `git` for version control." Not architecture.

## The Narrative Pattern

The 4 minutes follows this structure:

```
0:00–0:30  Frame the decision: "The one I want to walk through is X."
0:30–1:30  The constraint / problem the decision had to solve.
1:30–2:30  The alternatives considered.
2:30–3:30  The choice made, and why.
3:30–4:00  The trade-off accepted. What's worse because of this choice.
```

Four minutes is long enough to do this properly. The structure does the heavy lifting; the candidate fills in their specific decision.

## A Worked Example: Server-Anchored Timer

> **(0:00) Frame:** "The decision I want to walk through is how I built the quiz timer. The visible countdown the candidate sees during a quiz."
>
> **(0:30) Constraint:** "The quiz is graded server-side. If the candidate's browser thinks they have 30 seconds left, but the server has already moved past the deadline, submissions get rejected and the candidate sees a confusing error. So the timer needs to agree with the server's idea of time, even though the *display* is happening in the browser."
>
> **(1:30) Alternatives:** "I considered three approaches. First, just use `setInterval` against `Date.now()` — simple, but client clocks drift and a candidate could manually change their system clock to cheat. Second, poll the server every second for remaining time — guaranteed accurate, but 100 candidates × 60 seconds × 30 minutes = 180,000 requests per quiz, and the server has better things to do. Third, anchor once and tick locally — capture the server-to-client offset on load, then count down using that offset."
>
> **(2:30) Choice:** "I chose the third — anchor once, tick locally. The offset captures any clock drift, and the local tick is free. The server still independently rejects late submissions, so the visible timer is a UI hint, not the enforcement. If the local clock drifts mid-quiz, the worst case is the candidate sees a slightly-wrong countdown but the server still decides."
>
> **(3:30) Trade-off:** "The cost is one more concept the next developer has to understand — the timer isn't authoritative, it's an *anchor display*. I documented this in the component header. If we ever add long-running quizzes — say, multi-hour ones — drift could become visible to the candidate, and we'd need to re-anchor periodically. For 30-minute PEP quizzes, that's not a concern yet."

That's 4 minutes, paced. Notice what the narrative did:

- Named the problem in real-user terms ("confusing error"), not abstract terms ("synchronization").
- Considered three alternatives, not one or two. Three forces the candidate to actually think.
- Justified the choice against the alternatives, not in isolation.
- Owned the trade-off explicitly. Mature engineering acknowledges what's worse, not just what's better.
- Named the future-state condition under which the choice would need to be revisited.

## Defending AI-Generated Code

The closing of the Unit 0 thread. Across the cohort, Claude Code drafted real code — that's the point of the AI Tooling Thread woven through D4, D8, D12, D14, D16, D17, and beyond. The discipline at the capstone is:

**If Claude Code drafted the architectural decision being walked through, the candidate's narration *includes* that fact and *still defends the choice* as their own.**

The pattern:

> "Claude Code drafted the first pass of this. I considered alternative X. I chose Y because Z. The trade-off is W."

Four sentences. The first sentence is the disclosure, the rest is exactly the same defense any other code gets. The disclosure is not an apology and not an excuse — it's a fact about the workflow.

A worked example, applied to the scoring engine (D12):

> "Claude Code drafted the initial structure of the scoring engine — the loop that walks each answered question, looks up the correct answer, and accumulates the score. I considered two alternatives to what it produced: a SQL-side score computation that returns aggregates without round-tripping each answer, and a streaming approach that scores question-by-question as autosaves arrive. I chose to keep the synchronous loop-on-submit approach because the quiz length is bounded (≤30 questions) and the synchronous flow is debuggable in a way the streaming approach isn't. The trade-off is that very long quizzes (which we don't have) would push the submit response latency up — at 30 questions we're at ~80ms, at 300 we'd be at ~800ms and need to revisit."

The candidate clearly owns this. They know what Claude wrote, why they kept it, what they'd have changed, and where the boundary of the design's validity lies. **Stakeholders evaluate that ownership, not the keystroke origin of the code.**

## What Not To Say About AI-Generated Code

| Don't say | Why |
|---|---|
| "Claude wrote it, so I'm not sure why." | Cedes ownership. The candidate's job is to know why. |
| "I had Claude write it the way it always does." | Treats the AI as opaque. The cohort is meant to read and understand the output. |
| "Claude made a mistake and I had to fix it." (without specifics) | Reads as defensiveness, not engineering. Be specific about what was wrong and how the fix worked. |
| "I prompted it well." | The audience cares about the code, not the prompt. |
| Hide the AI assistance entirely. | If asked directly and the candidate says "no AI involved" when AI *was* involved, that's a credibility problem. Disclosure is cheap. |

The healthy frame: AI assistance is part of the modern engineering toolchain. Disclosing it normalizes it. The interesting question is what the candidate *did with* the draft — what they kept, what they changed, what they understood, what they pushed back on.

## What If An Audience Member Asks "Did You Write This?"

Direct question, direct answer. Two examples:

> "Yes, I wrote that section directly — I was trying to understand the request lifecycle myself, so I didn't want to skip the typing."

> "Claude Code drafted that. I read through it line by line, ran it, and modified two things: the error handling for the lock-timeout case, and the way we return the cached idempotency response. The core structure is what Claude produced."

Both answers are fine. The second is fine *because* the candidate can immediately follow with what they did with the draft. A candidate who can only say "Claude wrote it" and stop there fails the question.

## The Q&A Minute

The final 1 minute of the capstone is open Q&A. The architectural walkthrough usually generates the questions ("why didn't you do X?"), so the candidate should anticipate one follow-up per alternative they mentioned.

For the timer example above:

- Anticipated Q: "Why not WebSockets for the countdown?" — A: "Same overhead concern as polling; for a 30-minute window the anchor-and-tick gets us the accuracy without holding a connection per candidate."
- Anticipated Q: "What if the candidate refreshes the page mid-quiz?" — A: "On resume, the offset re-anchors from the server; the visible countdown jumps to the server's truth. I added a brief 'reconnecting' indicator so the jump isn't surprising."
- Anticipated Q: "Could a candidate cheat by intercepting the offset?" — A: "They could — but the server is the enforcement, not the timer. They'd just see a wrong countdown and still get rejected on a late submit."

Each anticipated question rehearsed in advance, in the *what / why / how* cadence from the storytelling topic.

## Picking The Decision: Which One

Candidates often want to pick the *coolest* decision. Wrong frame. The right frame: pick the decision the candidate can defend in the most detail. The most *defensible* choice beats the most *impressive* choice.

If the candidate did three months of research into idempotency patterns and built a beautifully justified key-based dedup system, walk through *that*. If the candidate copy-pasted the timer code from somewhere, do not walk through the timer — pick a decision they actually wrestled with.

Self-test: can the candidate name two alternatives they didn't pick, and one trade-off they accepted? If yes, the decision is walkable. If no, pick a different decision.

## Anti-Patterns

- **Walking through a decision with no alternatives considered.** "I used Postgres because it's a database." Not architectural; not defensible.
- **Naming alternatives but not explaining why they were rejected.** "I could have used Redis instead..." then nothing. Why didn't you?
- **Pretending there was no trade-off.** Every choice has a cost. A candidate who can't name the cost hasn't fully reasoned about the choice.
- **Hiding AI assistance.** Disclose. Then defend the choice on its merits.
- **Choosing the decision that's hardest to explain.** Wrong incentive. Choose the decision you can explain *clearly*, not the decision that sounds smartest.
- **Treating the walkthrough as a code reading.** It's not. Code is incidental; the *decision* is the subject. Open the file briefly if it helps illustrate the trade-off, but the audience should be looking at the candidate, not the editor.

## Connecting Back to the AI Thread

The thread that ran across PEP:

- D4 introduced AI-assisted Git operations and the discipline of reviewing diffs before commits.
- D8 / D9 used AI for question-authoring scaffolding and FastAPI routing.
- D12 / D14 used AI for scoring algorithms and Vitest test authoring, with explicit "read it, run it, modify it" discipline.
- D16 / D17 used AI for aggregation queries and chart configuration.
- D18 / D19 used AI for dashboard SQL and Next.js server components.

Each of those days emphasized **own the output**. Today closes the loop: in front of stakeholders, narrate the ownership explicitly. The 10-week intensive will use AI tooling more aggressively (LangChain, LangGraph, agentic patterns), and the "own the output" muscle is exactly the muscle that survives the scale-up.

## Key Takeaways

- The 4-minute architectural walkthrough follows: frame → constraint → alternatives → choice → trade-off.
- Pick the decision you can defend in the *most detail*, not the one that sounds the most impressive.
- Always name at least two alternatives considered and one trade-off accepted.
- For AI-drafted code, disclose with the pattern: "Claude drafted this; I considered X; chose Y because Z; trade-off is W." Disclosure is cheap; ownership is the point.
- Anticipate the follow-up Q&A per alternative mentioned. Rehearse one-sentence answers in *what / why / how* cadence.
- The Unit 0 thread closes today. The discipline carries into Phase 2: AI assists, candidate owns.

---
*Prerequisites: day-04-ai-assisted-git-operations-and-pr-discipline, day-12-ai-assisted-test-authoring-with-parametrized-pytest-cases, day-20-storytelling-around-technical-work.*
