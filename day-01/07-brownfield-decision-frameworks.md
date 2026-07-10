# Brownfield Decision Frameworks (Orientation Depth)

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

## Overview

Brownfield work is full of forks where more than one path is defensible — keep the inherited model or refactor it, conform to the existing client or redesign the contract, modify the page in front of you or rewrite it. The previous topic established the *mindset* ("read before you write, find the seams"); this topic previews the *toolkit* you will use when "read first" is not enough on its own and you actually have to make the call. Six small frameworks, defined briefly here at orientation depth, will get applied in context at specific decision points across Weeks 2–4. Today's goal is just to put their names in your head and tell you what kind of decision each is for, so they are not new vocabulary when the curriculum reaches for them later.

## Why a Toolkit at All

When you spot something in inherited code that looks like it might need changing, "should I change it?" decomposes into a set of smaller questions:

- *What is actually here?* (Static analysis)
- *Is what is here a problem, or just unfamiliar?* (Technical debt assessment)
- *If I changed it, would the value justify the cost?* (Cost-benefit)
- *Over what time window am I measuring "value"?* (ROI horizon)
- *When neither path is obviously better, what are the strengths, weaknesses, opportunities, and threats of each?* (SWOT)
- *Am I keeping this only because it already exists?* (Sunk-cost recognition)

Each framework below addresses one of those sub-questions. They do not replace judgment — they are scaffolding for judgment. You will rarely apply all six to a single decision; usually two or three are enough.

## The Six Frameworks

### 1. Static analysis

**What it is.** A factual reading of the code as it actually exists, before any judgment. Trace call trees, build a dependency graph, identify which pieces are load-bearing (used by many things) versus deletable scaffolding (used by nothing live). For a UI, that means greping for every call into a given API surface and mapping URL/method/request shape/response shape. For inherited modules, it means "who imports this, what does it import, what state does it own."

**What kind of decision it suits.** Any decision that depends on knowing the actual surface area of a change — *before* you start judging it. Static analysis produces the factual layer underneath every other framework on this page.

**When it shows up later.** D11 (mapping the inherited frontend's calls into the quiz/session surface to know what contract the client believes in); D13 (dependency mapping of inherited UI components to distinguish load-bearing from isolated patterns before the modify-vs-rewrite call).

### 2. Technical debt assessment

**What it is.** Distinguishing a debt *smell* — code that is genuinely worse than it should be (growing chains of `if type == ...`, dead imports, tight coupling, missing tests, inconsistent patterns vs the rest of the codebase) — from a *design decision that fits the domain*. Not every awkward-looking piece of code is debt; some of it is scar tissue with reasons, and some of it is style you happen to disagree with.

**What kind of decision it suits.** "Is this thing worth changing on its own merits?" Used to *name* what's worth changing and what isn't, before you start arguing about cost.

**When it shows up later.** D8 (is the inherited polymorphic model a debt smell or a deliberate design?); D13 (is the inherited UI debt, or simply unfamiliar?).

### 3. Cost-benefit

**What it is.** Estimate the effort of a change (hours, churn, test rewrites, downstream breakage) against the value it produces (cleanliness, learner ownership, future velocity, defect reduction). The point is not to produce a precise number — your estimates will be wrong — but to *force the comparison to be explicit* rather than leaving it as a gut feeling.

**What kind of decision it suits.** Any single, scoped change where "refactor or extend" is the question. Tightly paired with ROI horizon (next framework), because cost-benefit without a time window is incomplete.

**When it shows up later.** D8 (refactor the inherited polymorphic model vs extend it — estimate API churn, test rewrites, frontend zod-mirror churn against the value); D11 (estimate frontend rework hours under "redesign" vs backend shape compromises under "conform").

### 4. ROI horizon

**What it is.** The question "value over what time window?" — and the discipline of *naming the horizon before deciding*. The PEP cohort runs for four weeks. A refactor whose payoff is "the codebase is cleaner in 18 months" has effectively zero return inside a 4-week horizon, but might be the right call over a multi-year ownership horizon. Same change, different answer, because the horizon changed.

**What kind of decision it suits.** Layered on top of cost-benefit. Whenever someone says "but in the long run, X is better," ROI horizon asks "*which* long run, and is that the run we are scoped for?"

**When it shows up later.** D8 (a polymorphic-modeling refactor often loses over 4 weeks but wins over a multi-year horizon — name your horizon before the call); D13 (rewriting a multi-component UI flow often loses over the ~3 remaining days of Week 3 but might win over longer ownership).

### 5. SWOT

**What it is.** Strengths / Weaknesses / Opportunities / Threats — a four-quadrant framing of a single path. Useful when cost-benefit alone collapses too much into a single number and you want to keep qualitative texture (e.g., "redesign's strength is a clean contract, but its threat is scope creep into the frontend work two days from now").

**What kind of decision it suits.** Modify-vs-rewrite calls where neither path is obviously better, and where the trade-offs include qualitative things (clarity, scope risk, team norms) that don't sit neatly in an hours-vs-value table.

**When it shows up later.** D11 (SWOT of conform-vs-redesign for the inherited client's API expectations); D13 (SWOT of modify-vs-rewrite for the inherited test-taking UI).

### 6. Sunk-cost recognition

**What it is.** The counterweight to all of the above. Sunk-cost fallacy is the bias toward keeping something because of what has already been spent on it, rather than because it earns its keep going forward. In a brownfield course this looks like: *"there is already a half-built test-taking page in the repo, so we should modify it."* That is not a reason on its own. The right question is whether keeping it is the best path from here on out — the past investment is not recoverable either way.

**What kind of decision it suits.** Any decision where "but this already exists" is creeping into the argument. Sunk-cost recognition is the brake. It does not say "throw out inherited code" — it says "keep it only if the *forward* math says to."

**When it shows up later.** D13 (the explicit counterweight on modify-vs-rewrite — do not keep the inherited UI just because it exists; keep it because cost-benefit and ROI horizon say so).

## How They Combine

You don't pick one framework — you stack them in the order their outputs feed each other:

1. **Static analysis** first — what's actually here? (Facts, no judgment yet.)
2. **Technical debt assessment** — is what's here a problem or just unfamiliar?
3. **Cost-benefit** and **ROI horizon** together — if changing it is worth doing, is the value worth the effort *over our time window*?
4. **SWOT** — if cost-benefit is close, what qualitative factors break the tie?
5. **Sunk-cost recognition** — sanity-check: am I keeping this only because it already exists?

Most decisions are settled by steps 1–3. SWOT and sunk-cost are reserves for the genuinely-balanced cases. Orientation depth is enough today; the *application* of these frameworks happens in context at the days that need them.

## Example / Worked Scenario

A miniature worked scenario, to put one framework on the page concretely — not the full toolkit, just a glimpse.

Suppose on Day 8 (next week) you are looking at the inherited `question-management-service` and find that the existing Pydantic model handles question types via a tagged enum with conditional validators: one `Question` class with a `type: QuestionType` field and a chain of `@model_validator` calls that fire per type. Your instinct says "this should be a discriminated union; it would be cleaner." Apply the frameworks:

- **Static analysis** — grep for every place `Question` is imported. You find: the API layer, three tests, and the frontend's zod schema (mirrored). That's the surface a refactor would touch.
- **Technical debt assessment** — is the current shape a smell or a design? Look at the conditional validators. Three of them, each ~5 lines. Not a sprawling chain. Tests cover them. It is *acceptable*, even if not your preferred style.
- **Cost-benefit** — refactor cost: rewrite the model, rewrite three tests, rewrite the zod mirror (touches the Day-9 frontend work). Maybe 4–6 hours. Refactor value: a cleaner type system, marginally less branching.
- **ROI horizon** — you have a 4-week PEP horizon. The cleaner type system pays off mostly in *future* maintenance, which the PEP horizon does not extend into. Over a 12-month ownership horizon, the math might flip.
- **Conclusion** — extend, don't refactor. Document the call in the PR description, so reviewers can see the reasoning.

That whole reasoning chain takes maybe 10 minutes once you are fluent. Today, it might take longer. By Week 2, with these tools named, it should feel less like staring at a problem and more like working through a checklist.

## A Note on Acceptance Criteria

The curriculum supplies *integration acceptance criteria* — the guardrails the result must meet regardless of which path you chose. The frameworks help you *make* the decision; the acceptance criteria *check* the result. On Day 13 (the highest-stakes decision day of the course), the acceptance criteria are spelled out explicitly: the chosen path must call the Day-11/12 contract, propagate auth, persist navigation state, leave no broken imports, and render finalized question shapes. Pick whichever path the frameworks recommend; ship a result that meets those criteria. The path is yours; the bar is fixed.

## Common Pitfalls

- **Applying every framework to every decision.** Six tools is a toolbox, not a checklist. Most decisions are settled by two or three; reaching for all six is over-engineering the decision-making itself. Pick the ones that address the specific sub-question you are stuck on.
- **Skipping static analysis and going straight to judgment.** Without the factual layer (what is actually here, what is it connected to), every subsequent framework runs on assumptions. Build the dependency picture first, then judge.
- **Not naming the ROI horizon.** Arguments about "the long run" are unresolvable until both sides agree on the time window. The PEP horizon is four weeks for course-internal decisions; production work has different horizons. Name yours explicitly.
- **Treating sunk cost as "throw out inherited code."** Sunk-cost recognition is *symmetric* — it argues against keeping things for bad reasons, but it equally argues against throwing things away just because you didn't write them. The point is forward-looking math, not a bias either direction.
- **Using cost-benefit estimates as if they were precise.** Your estimates will be wrong. The discipline is not precision — it is *forcing the comparison to be explicit* rather than left as a hunch. Two rough numbers beat one strong opinion.

## Key Takeaways

- Brownfield decisions decompose into smaller questions, and each of the six frameworks addresses one of those questions: static analysis (what's here?), technical debt assessment (is it a problem?), cost-benefit (worth the effort?), ROI horizon (over what window?), SWOT (qualitative trade-offs?), sunk-cost recognition (am I keeping it only because it exists?).
- These are scaffolding for judgment, not a replacement for it. Most decisions use two or three frameworks; reaching for all six is over-engineering the decision-making.
- Stack them in order: facts (static analysis) → debt judgment → cost/value over a named horizon → SWOT for close calls → sunk-cost sanity check.
- The PEP curriculum applies these in context across Weeks 2–4: D8 (polymorphic-modeling decision), D11 (conform-vs-redesign for an inherited client), D13 (modify-vs-rewrite for an inherited UI). Each later topic reaches back to the framework names introduced here.
- Integration acceptance criteria (where the curriculum supplies them) check the *result*, not the *path*. The frameworks help you decide; the acceptance criteria fix the bar the decision has to clear.

---
*Prerequisites: [04-brownfield-mindset-reading-inherited-code-finding-the-seams.md](04-brownfield-mindset-reading-inherited-code-finding-the-seams.md). These frameworks show up in context across W2–W4 at the following decision points: D8 (polymorphic modeling — technical debt, cost-benefit, ROI horizon); D11 (conform-vs-redesign for an inherited client — static analysis, SWOT, cost-benefit); D13 (modify-vs-rewrite for inherited UI — all six, with explicit integration acceptance criteria).*
