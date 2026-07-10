# Technical Debt Identification in Own Work

> *Day 20: Capstone Demo Day — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The 2-minute retrospective slot inside each capstone presentation has one job: **show the candidate can identify, articulate, and own technical debt in their own work.** This is one of the highest-signal evaluation moments of the day. Candidates who present their work as flawless read as junior — either they didn't notice the shortcuts, or they noticed but won't say. Candidates who walk through specific shortcuts, name what would need to change to repay them, and rank them by urgency read as senior. The skill being assessed is not "having debt" (everyone does) but **knowing where it is**.

This topic gives the cohort a vocabulary, a few worked examples drawn from the actual PEP substrate, and a structure for the 2-minute retro. The work is reflective, not constructive — no new code, no fixes. The capstone is too late for fixes; the value is in the *naming*.

## Objective

Identify and articulate technical debt in their own work.

## What "Technical Debt" Means Here

A working definition for this slot:

> **Technical debt** is a place where a deliberate or accidental shortcut was taken, the system works today *because of* the substrate's current scale or constraints, but the shortcut will need to be revisited when conditions change.

Three components:

1. **A shortcut was taken** — there is a "right" alternative that wasn't chosen.
2. **The system works today** — debt is not the same as bugs. A bug doesn't work; debt does, for now.
3. **Conditions will change** — debt is debt because the rest of the world won't stay still.

Things that are *not* debt:

- Bugs that aren't fixed yet (those are *bugs*, name them as such).
- Features that weren't built (those are *scope*, name them as such).
- Style preferences the candidate didn't adopt (those are *opinions*, not debt).
- Anything where the candidate would do the same thing again (no shortcut = no debt).

The cleaner the definition, the cleaner the narration.

## The Debt-Naming Structure

For each piece of debt in the 2-minute retro:

```
WHAT: the shortcut, in one sentence.
WHEN IT BREAKS: the future condition that exposes the debt.
WHAT REPLACES IT: the right alternative when that condition is reached.
URGENCY: now / next quarter / next year / hypothetical.
```

Four lines per debt item. In 2 minutes the candidate can credibly walk through 2-3 items. Don't try for more.

## Worked Example 1: Offset/Limit Pagination

> **What:** "The trainer dashboard uses offset/limit pagination — `?page=3&size=20` translates to `OFFSET 60 LIMIT 20`."
>
> **When it breaks:** "Past 10,000 attempts. Offset pagination forces Postgres to walk the rows it's about to discard, so `OFFSET 10000` is doing work proportional to 10,000, not 20. At cohort scale (25 candidates, 20 tests, ~500 attempts), it's invisible. At multi-cohort scale, it's a P95 latency problem."
>
> **What replaces it:** "Cursor-based pagination keyed on `(created_at, id)`. The query becomes `WHERE (created_at, id) > (cursor) LIMIT 20`, which is index-only and constant-time regardless of position. The UI loses the ability to jump to 'page 47' directly, but trainers don't actually use that — they scroll or filter."
>
> **Urgency:** "Next cohort if we scale past 4-5 cohorts of attempts in the same database. Not now."

A clean debt narration. The candidate has shown they know offset pagination scales poorly, they know what cursor pagination is, they know the UX trade-off, and they know the *condition* under which the shift becomes necessary.

## Worked Example 2: No Soft-Delete on Questions

> **What:** "When a trainer deletes a question, the row is removed from MongoDB. There's no soft-delete flag, no archive collection."
>
> **When it breaks:** "When we need to look at historical attempts. A candidate's attempt from two weeks ago references question ID 47; if question 47 was deleted, the results page can't show what they answered against. The current code shows '[Question removed]', which is bad UX and worse for forensic review of disputed scores."
>
> **What replaces it:** "Soft-delete with a `deleted_at` timestamp. Active question lookups filter `deleted_at IS NULL`; historical attempt lookups don't. The data stays."
>
> **Urgency:** "First time a candidate disputes a score and we need to show them the question. Possibly within weeks of going live."

## Worked Example 3: Server-Anchored Timer Doesn't Re-Anchor

> **What:** "The quiz timer captures the server-client offset once on page load. If the page stays open for 30 minutes without re-anchoring, clock drift accumulates."
>
> **When it breaks:** "Two scenarios. First, if quiz lengths grow beyond an hour, drift becomes visible (~1 second per 15 minutes typical). Second, if the candidate's tab is backgrounded by the OS, the `setInterval` is throttled, and the visible time gets badly out of sync."
>
> **What replaces it:** "Re-anchor on every page visibility change, and periodically (every 5 minutes) for very long quizzes. The server is still the source of truth for the deadline enforcement, so even today the worst case is a confusing UI, not a wrong score."
>
> **Urgency:** "Visibility-change re-anchor is a small fix; should be next sprint. Periodic re-anchor is hypothetical until quizzes get longer."

## Worked Example 4: API Gateway Is Just Path Routing

> **What:** "The api-gateway service routes by path prefix (`/users/*` → user-service, `/questions/*` → question-management, etc.) and forwards the request. It doesn't authenticate, rate-limit, or transform requests."
>
> **When it breaks:** "First time a downstream service is exposed to the internet without auth, *or* the first time we want a single rate limit across all services. Today each service does its own JWT validation, which means changing the auth scheme means changing four services. The gateway *should* be the auth boundary."
>
> **What replaces it:** "Move JWT validation into the gateway as a middleware. Downstream services trust the gateway's verified claims (passed through as headers). Add per-IP and per-user rate limits at the gateway."
>
> **Urgency:** "Next major auth change. Not now — but the next time someone proposes changing the auth flow, that's the moment to consolidate."

## Worked Example 5: Test Coverage Gaps

> **What:** "The scoring engine has unit tests for the happy path and the duplicate-submit path. It doesn't have tests for: a question that no longer exists when scoring runs, an attempt with missing answers (network drop mid-quiz), or a tied-score edge case."
>
> **When it breaks:** "Any of those three scenarios in production produces an undefined behavior. Today it probably 500s; tomorrow when the database has been migrated and the assumptions are slightly different, it could silently miscount."
>
> **What replaces it:** "Three parametrized test cases, one per scenario. I'd estimate an afternoon of work."
>
> **Urgency:** "Before any meaningful production traffic. Probably first or second sprint of the 10-week intensive."

## Worked Example 6: The Dashboard's Cohort-Level Aggregation Is N+1-Adjacent

> **What:** "The dashboard's per-test summary view calls one SQL query for the list of tests, then iterates over the tests in Python calling one query per test for the per-test stats. That's N+1 — fine for the 20 tests we have, painful past 200."
>
> **When it breaks:** "When a single cohort has more than ~100 tests, or when the dashboard view aggregates across multiple cohorts. Today the dashboard renders in 80ms; with 500 tests it'd be ~2 seconds."
>
> **What replaces it:** "Single query with `GROUP BY test_id` returning all the stats in one round trip. The Python becomes one pass over the result set instead of N queries."
>
> **Urgency:** "Cross-cohort feature; not in the current scope. The fix is small (~30 lines) and well-understood when needed."

## The 2-Minute Retro Structure

The candidate's 2-minute retro slot, framed around debt:

```
0:00–0:20  "Three pieces of technical debt I'd flag if I were handing this codebase over."
0:20–0:40  First item — what, when it breaks, what replaces it.
0:40–1:00  Second item.
1:00–1:20  Third item.
1:20–1:40  Honorable mentions ("there are more, but these are the top three").
1:40–2:00  Close: "Highest-priority is X because Y."
```

Two minutes is short — practice with a timer. Three items, ~20 seconds each. The format trains compression.

## Why Owning Debt Reads As Senior

A senior engineer's mental model includes the system's future. A junior engineer's mental model includes only the system's present. The candidate who can name where the substrate will break under future conditions is demonstrating the senior mental model.

Stakeholders watching the capstone are not evaluating "did the candidate take any shortcuts?" — everyone takes shortcuts. They're evaluating "does the candidate *know* where the shortcuts are?" A candidate who says "I'd ship this as-is" is signaling either that they didn't notice the shortcuts or that they noticed and won't say. Both are worse than "here are three places I'd revisit."

## The Substrate's Common Debt Patterns

For PEP candidates, these are the substrate-specific debts that almost always exist and are worth scanning for:

- **N+1 in any dashboard summary.** Did the candidate use a per-row query inside a loop?
- **Offset/limit pagination.** Did they ever write `OFFSET N` for any list endpoint?
- **Optimistic UI updates without rollback.** Did anywhere assume the server will succeed?
- **Frontend filter state that doesn't survive page refresh.** Filters in component state instead of URL params.
- **Missing error states.** Empty / loading is covered (D19), but what about "the API returned 500"?
- **Hardcoded strings.** Test names, role names, color values that should be constants or config.
- **Client-side role checks without server-side enforcement.** Looks like security but isn't.
- **No structured logging.** `print()` instead of structured log lines with request IDs.
- **Migration scripts that haven't been tested in reverse.** Can the schema change be rolled back?
- **Auth tokens with long expiry and no refresh.** Convenient for development, dangerous in production.

A candidate who picks three of these from their own codebase, narrates them in the structure above, and ranks them by urgency, is demonstrating fluent engineering judgment.

## Honesty Calibration

Some candidates will be too harsh on themselves; others too generous. The honest framing:

- **Don't catastrophize.** "My code is held together with duct tape" is not useful self-criticism; it's not specific and it's not actionable. If the duct tape is real, name the *specific* piece.
- **Don't whitewash.** "I'd ship as-is" is not honest; there's always *something*. If the candidate truly can't find debt, they haven't looked hard enough.
- **Don't blame the substrate.** "The codebase I inherited was the problem" — fine for context, not for the retro. The candidate's own contributions are what's being evaluated.
- **Don't blame Claude.** "Claude wrote it that way" — see D20's defending-AI-code topic; the candidate owns the output.

The healthy frame: "I made choices under time pressure. Some I'd revisit. Here are the ones I'd revisit first."

## Anti-Patterns

- **Listing more than three items.** Diminishing signal. After three, the audience can't track which is most important.
- **Listing bugs as debt.** "There's a bug in the chart" is not debt; it's a bug. Different category.
- **Naming debt without the *when it breaks* condition.** Without the condition, debt sounds like an apology. With it, it sounds like engineering judgment.
- **Promising to fix it.** "I'll fix that next sprint" is a commitment the candidate may not be in a position to make. "Here's what would need to change when X happens" is more honest.
- **Showing the code with the debt.** The 2-minute slot is too short for that. Name the debt; the code is incidental.
- **Apologizing for debt.** "I'm sorry I didn't have time to..." — debt is not a personal failure. Don't apologize.

## Connecting Back

This is reflective practice that closes the four-week loop. Every architectural decision made under time pressure during PEP — server-anchored timer, idempotency, dashboard aggregation, role gates — has a debt corner the candidate can now see. Walking through three of them in 2 minutes is a fluency exercise that will be re-used across every sprint review and code handoff for the rest of the candidate's career.

The 10-week intensive will produce more debt, faster, because the AI surface returns and AI-assisted code generates surface area faster than humans review it. The discipline of *naming* the debt before it's noticed by someone else is a survival skill for that environment.

## Key Takeaways

- Debt = shortcut + works today + future condition will expose it. Bugs, scope cuts, and style preferences are not debt.
- Each debt item gets four lines: what, when it breaks, what replaces it, urgency.
- Pick three items for the 2-minute retro, ~20 seconds each. Rank by urgency in the close.
- Common substrate debts: offset pagination, N+1 dashboard queries, missing error states, hardcoded strings, client-only role checks, no soft-delete.
- Owning debt reads as senior; pretending it doesn't exist reads as junior. Stakeholders evaluate the difference.
- Don't catastrophize; don't whitewash; don't blame the substrate, the AI, or anyone else. Own it.

---
*Prerequisites: [03-architectural-decision-reflection-defending-ai-generated-code.md](03-architectural-decision-reflection-defending-ai-generated-code.md).*
