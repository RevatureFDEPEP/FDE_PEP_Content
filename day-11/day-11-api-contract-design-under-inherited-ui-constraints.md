# API Contract Design Under Inherited UI Constraints — Does an Existing Client Dictate the Backend?

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview

Day 11 is the start of the quiz-taking backend. Before you sketch a single endpoint, there is a brownfield-specific question to answer: **is there already a frontend calling this surface, and if so, does its existing shape dictate what you build?** That decision is load-bearing — every subsequent day in Week 3 (D12 scoring, D13 frontend skeleton, D14 interactivity) builds on the contract you ship today. Get it wrong and you'll either burn D13–14 reworking the frontend or burn D11 redesigning the backend after the frontend has already shipped. This topic is about doing the analysis *before* the decision, not letting either bias win by default.

## Why This Matters Today

You're writing `POST /sessions` today. A real-world version of that question goes like: "what shape should the request and response take?" In a greenfield project that question has one input (your judgement) and one output (whatever you design). In a brownfield project — which this course is — that question has a second input: **whatever the existing client already assumes.** The inherited frontend may already have a `fetch('/api/sessions', { ... })` call sitting in a component, with a particular request body and a particular expected response shape. If you ignore it, you've quietly forked the contract; the frontend will compile but the integration will fail. If you blindly conform to it, you may inherit warts that didn't deserve to survive into the new backend.

This is exactly the analysis the brownfield decision toolkit was previewed for. See [[day-01-brownfield-decision-frameworks]] for the orientation-depth introduction to static analysis, SWOT, and cost-benefit; today is the first day you apply all three to a contract-design call.

## Step 1 — Static Analysis of the Inherited Client

The factual layer. **Before opinion, evidence.**

Grep the frontend for every call into the quiz/session surface. The targets are usually:

- `fetch(` or `axios(` invocations whose URL contains `session`, `quiz`, `attempt`, or `test`
- Server actions, route handlers, or API client modules with names like `sessionApi.ts`, `quiz-client.ts`, `useSession*`
- Type definitions in `types/` or `lib/` referencing `Session`, `Attempt`, `Quiz`, `Question`

For each call site, fill in a row of this table:

| URL | Method | Request shape | Response shape | Error handling | Client storage |
|---|---|---|---|---|---|
| `/api/sessions` | POST | `{ quiz_id }` | `{ session_id, first_question, server_now }` | `if (!res.ok) throw` | `localStorage.setItem('sid', ...)` |
| `/api/sessions/:id/answer` | POST | `{ question_id, choice_ids[] }` | `{ next_question \| null, remaining }` | toast on 4xx | — |
| `/api/sessions/:id` | GET | — | `{ session, current_question, expires_at }` | redirect to /login on 401 | — |

That table is **"the contract the client believes in."** It is a fact, not a recommendation — the client *will* send these requests, expect these responses, and behave in these ways on error, regardless of what the new backend wants. Until you have this table, every "should we conform or redesign?" conversation is uninformed.

**If no inherited client exists** (no calls into the surface, or the surface is brand-new), skip directly to clean design and note it explicitly in the PR: "No inherited frontend calls this surface; designed fresh."

## Step 2 — SWOT on Conform vs Redesign

With the table in hand, frame the two extreme options as SWOT. (Hybrid is below; do the extremes first to see the trade space.)

|  | Conform to inherited | Redesign clean |
|---|---|---|
| **Strengths** | Zero frontend churn; ship faster | Clean shape; ownership of the contract |
| **Weaknesses** | Inherits client warts | Forces D13–14 frontend work |
| **Opportunities** | Tighter slice timeline | Better long-term API |
| **Threats** | Locking in awkward shape | Scope creep into frontend timeline |

A few patterns to read off this:

- **"Conform" is strongest when** the inherited client's shape is reasonable and the timeline is short. PEP's 4-week horizon often points this way.
- **"Redesign" is strongest when** the inherited shape has a load-bearing problem you'd refuse to defend in a PR review (e.g., the client posts `{ correct_answers: [...] }` to a public endpoint — a security smell you cannot ship even by accident).
- **Neither is the principled default.** The bias that "redesign = professional, conform = lazy" is a trap. So is the bias that "conform = pragmatic, redesign = perfectionist." The right answer is whichever the evidence supports.

## Step 3 — Cost-Benefit

Estimate both sides. The numbers are rough; the discipline is doing the estimate at all.

**Worked example.** Suppose the inherited client posts `POST /sessions { quiz_id, candidate_name }` and expects `{ token, q }` back, where `q` is the full first question including correct-answer flags. (The `candidate_name` is redundant — you have it from the JWT. The correct-answer leak is a security bug.)

**Conform path cost.**
- Backend builds the awkward shape: ~2h (model the request with the redundant field, ignore it server-side; respond with the leaky shape).
- Security trade-off: cannot ship the leak. So you'd actually have to **partial-conform** here — keep the URL/verb/most of the shape, but strip the correct-answer field. ~3h total.
- Frontend rework: 0–1h (if the frontend was reading the leaked field anywhere, that's a separate fix on D13).

**Redesign path cost.**
- Backend builds clean shape: ~2h.
- Frontend rework on D13: ~3–4h (update the API client, the types, the components that consume the response, and any zod schemas).
- Total: ~5–6h, but spread across two days, and D13's "modify-vs-rewrite" decision (see the curriculum's D13 brief) may absorb some of that cost anyway.

**Benefit.**
- Conform: feature in working state by Day 11 EOD; D13 has less to do.
- Redesign: cleaner contract that the candidate authored; better defense in the Day 20 capstone; ownership of the surface.

In this scenario the **partial-conform / hybrid** (next section) often wins: conform on URL and verb, redesign the response payload to remove the leak and the redundancy.

## Step 4 — Hybrid is a Valid Third Path

Conform-vs-redesign is a false binary. The real space is:

- **Conform on URLs and verbs, redesign payload shapes.** Frontend's network code (the call sites) keeps working; the request/response *bodies* change. Most "the URL is fine but the payload was sketchy" cases land here.
- **Redesign URLs and verbs, conform on payloads.** Rarer, but valid — sometimes the URLs are wrong (`/api/quiz/start` should be `/api/sessions`) but the body shape is fine.
- **Conform on the happy path, redesign error responses.** The frontend's "happy path" call sites work unchanged, but you upgrade `{ error: "bad" }` to a proper RFC-7807 problem document.

**Name the boundary explicitly in the PR.** Don't ship a hybrid that the reviewer has to reverse-engineer. The PR description should read like: *"Conformed on `POST /sessions` URL and request body to avoid frontend churn. Redesigned the response to remove the `correct_answers` field (security) and to include `server_now` (for the D14 timer). Frontend types updated in the same PR."*

**Example where hybrid often makes sense in PEP:** the inherited client calls `POST /sessions` with a fine request shape but expects a flat response. You want the response to include `server_now` (for [[day-11-server-authoritative-state]]'s offset calculation, which D14's timer depends on) and you want to strip any leaked correct-answer fields. Hybrid: conform URL/verb/request, redesign response. One-line frontend type update on D13; no behavioural change to the call site.

## Step 5 — The Decision Lives in the PR

Whichever path you choose, the **PR description records the reasoning.** A reviewer should be able to read the PR and see:

1. **What you found.** A short version of the static-analysis table — the inherited call sites and what they expect.
2. **What you chose.** Conform / redesign / hybrid, and if hybrid, where the boundary sits.
3. **Why.** The SWOT and cost-benefit, briefly. Two or three sentences is fine.
4. **What changes downstream.** "D13 frontend will need X" or "D13 frontend unchanged."

This matters because the rest of Week 3 is load-bearing on today's call:

- **Day 12 — scoring engine** writes `POST /sessions/{id}/answer`. Its shape is constrained by today's choice.
- **Day 13 — frontend skeleton** consumes today's contract. If you chose redesign, D13 absorbs the rework; if conform, D13 is freer to focus on its own modify-vs-rewrite decision.
- **Day 14 — interactivity** (timer, submission) reads `server_now` and `expires_at` from today's response. The fields have to be there.
- **Day 20 — capstone** asks you to defend the choices you made. "Why this contract shape?" is one of the standard questions.

If the PR doesn't record the reasoning, you'll be reconstructing it from memory in Week 4. Don't.

## Common Pitfalls

- **Skipping the static-analysis step and guessing what the client expects.** The most common failure mode. You think "the client probably wants `{ session_id, first_question }`" and ship that, then discover on D13 that the inherited client wanted `{ token, q }`. Now you've broken the frontend without realizing it, and the diagnosis is "why isn't the call working?" instead of "I never checked the contract." Grep first.
- **Treating "conform" as the lazy default.** "There's a frontend, so I'll just match it" sidesteps the analysis. If the inherited shape has a real problem (security leak, wrong verbs, redundant fields), conforming bakes the problem into your new code and you'll defend it forever.
- **Treating "redesign" as the principled default.** "The inherited shape isn't to my taste, so I'm rewriting it" sidesteps the cost-benefit. PEP's 4-week timeline doesn't have slack for vanity redesigns. Redesign when there's a defensible reason, not because clean code is its own reward.
- **Choosing hybrid without naming the boundary.** Hybrid is the most common right answer and the easiest to do sloppily. If your PR says "I changed some things and kept some things," the reviewer has to reverse-engineer the boundary. Spell it out.
- **Letting the decision drift through D12–D14.** The contract changes mid-week because nobody locked it in on D11. Then D13's frontend doesn't match D14's mutations and nobody knows which is right. The fix is discipline: today's PR is the source of truth, and any later change comes with a follow-up PR that updates the source of truth too.

## Key Takeaways

- The analysis precedes the choice. Grep the frontend before you sketch the backend; produce the "contract the client believes in" table before opinion enters.
- All three paths — conform, redesign, hybrid — are legitimate. None is the default. Pick the one the SWOT and cost-benefit support for *this* contract under *this* horizon.
- Hybrid is often the right answer, and it requires you to name the boundary explicitly (URL/verb vs payload; happy path vs error; etc.).
- The call is the candidate's, and it is defended in the PR description. The rest of Week 3 builds on today's contract — D12 scoring, D13 skeleton, D14 interactivity all assume it's authoritative.
- If no inherited client exists, say so in the PR and design clean; don't invent constraints that aren't there.

---
*Prerequisites: [[day-01-brownfield-decision-frameworks]] (orientation-depth introduction to static analysis, SWOT, cost-benefit). Sibling topics: [[day-11-pydantic-request-response-modeling]] applies the chosen contract; [[day-11-server-authoritative-state]] constrains what fields must appear in the response. Forward references: D12 scoring contract, D13 frontend modify-vs-rewrite, D14 timer and submission.*
