# Modify-vs-rewrite decisions for inherited UI surfaces — analysis frameworks and integration acceptance criteria

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Vertical Slice (the meat)*

## Overview

Day 13 is the highest-stakes brownfield decision day in the course. The work in front of you is the frontend skeleton for the quiz-taking flow — a dynamic route, an initial fetch, a first question render, and navigation between questions. But before any of that is written, you have to answer a question the curriculum deliberately refuses to answer for you: **what do you do with whatever frontend code is already there?**

There may be a partially-built test-taking page in the inherited substrate. There may be a complete-looking flow pointed at endpoints that no longer exist after the AI surface was stripped. There may be nothing — an empty route waiting to be filled. In each case, the path forward — **modify what's there, rewrite from scratch, or hybrid (keep some, replace some)** — is a judgement call you own and defend in the PR. This topic gives you the analysis toolkit to make that call defensibly, and the integration acceptance criteria that any path must satisfy. The result is a decision documented in your PR description, not a pattern handed down by the curriculum.

## The situation: three legitimate paths

You will open the `/take/[testId]` area (or whatever the inherited substrate calls it) and find one of three rough shapes:

1. **An inherited skeleton.** Components exist — maybe a `QuizPage`, a `QuestionCard`, a navigation control. Some of it may already render. Some of it almost certainly calls endpoints that no longer exist (the AI quiz service is gone; references to it are dead). State management decisions have been made by someone else and are visible in the props and hooks.
2. **A near-complete flow that doesn't work.** The original developer got further than a skeleton — there's a full page, multiple components, maybe even submission logic — but it was wired to the stripped AI surface, so nothing actually runs against the Day-11/12 backend you (or a peer) built. The shape exists; the wiring is dead.
3. **Nothing.** The route is empty or absent. There's no decision to make here — the choice has been made for you. (Skip ahead to the integration acceptance criteria section.)

For shapes 1 and 2, you have a real decision in front of you, and three legitimate paths:

- **Modify.** Keep the inherited code; change what needs to change. Rewire dead API calls. Reshape state where the new backend contract demands it. Delete dead references. Add anything missing.
- **Rewrite.** Delete what's there and build the frontend skeleton fresh against the Day-11/12 contract. Lose the inherited UX work; gain a clean fit.
- **Hybrid.** Keep some pieces (a layout, a presentational component, a styling system); rewrite others (state management, API integration, the route handler).

The curriculum's position: **all three are legitimate.** Which is correct depends on what the inherited code actually looks like, how much of the day's remaining time you want to spend on analysis vs. building, and what the integration acceptance criteria demand. The work below is making the call **defensibly**.

## The full analysis toolkit, applied to UI modify-vs-rewrite

The analysis frameworks were introduced at orientation depth on Day 1 — see [07-brownfield-decision-frameworks.md](../day-01/07-brownfield-decision-frameworks.md). You've also applied a subset of them in narrower contexts: [01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md](../day-08/01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md) used technical-debt and ROI-horizon analysis for a Pydantic modeling call; [06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md](../day-10/06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md) applied impact mapping to a cross-cutting infrastructure change; [01-api-contract-design-under-inherited-ui-constraints.md](../day-11/01-api-contract-design-under-inherited-ui-constraints.md) used SWOT and cost-benefit for a backend contract decision. Today combines the full toolkit on a single decision.

Apply each framework. Do not skip ahead to "I think we should just rewrite it" — that's a guess, not an analysis.

### 1. Static dependency mapping (the factual foundation)

Before any of the judgement frameworks (debt, ROI, SWOT) can do useful work, you need facts. Static dependency mapping is the factual layer. The goal is a graph: for every inherited file in the quiz-taking surface, what does it import, who imports it, what API calls does it make, and what state does it own?

The practical procedure:

1. **Enumerate the files.** Walk the `/take` route directory tree. List every `.tsx`, `.ts`, hook file, and helper that lives there or is imported transitively.
2. **For each file, capture four things:**
   - **Imports** — what does this file pull in? Other components, hooks, types, API clients, utilities.
   - **Importers** — who imports this file? Grep the rest of the codebase. A component imported only by a single dead page is in a very different position from one imported by ten surfaces.
   - **API calls** — what endpoints does it hit? Are those endpoints still alive on the Day-11/12 backend, or are they references to the stripped AI surface?
   - **Owned state** — what does this file `useState` / `useReducer` / Zustand-store / Context-provide? Is that state shape compatible with the new backend's response shape?
3. **Draw the graph.** Nodes are files; edges are imports. Annotate API calls and state ownership.

Once the graph exists, classify each node:

- **Load-bearing** — actively used by other surfaces (the dashboard, the question-authoring page, the login flow). Deleting these breaks unrelated work. They are expensive to remove; any "rewrite" path must preserve their interface or migrate their callers.
- **Isolated** — only used by the dead quiz-taking path. These are cheap to delete and cheap to rewrite. The cost of removing them is near zero.

This classification dominates everything that follows. Modifying load-bearing code is risky but contained; rewriting it cascades. Rewriting isolated code is safe and cheap; modifying it may not be worth the analysis effort.

**Worked example — a hypothetical inherited `QuestionCard` component:**

Imagine the inherited substrate has `app/take/[testId]/components/QuestionCard.tsx`. Static mapping turns up:

- *Imports:* `react`, `@/components/ui/card` (shadcn), `@/lib/quiz-api` (a client that calls the stripped AI quiz service at `POST /ai-quiz/grade`), `@/types/question` (the polymorphic question type from Day 8), `useQuestionState` (a custom hook colocated in the same directory).
- *Importers:* grep finds two — the dead `app/take/[testId]/page.tsx`, and `app/preview/QuestionPreview.tsx` (a question-bank preview surface used by the question-authoring page from Day 9).
- *API calls:* one — `quizApi.gradeAnswer()`, which calls a stripped endpoint.
- *Owned state:* a `selectedChoices: string[]` local state, plus a `submitted: boolean` toggle.

Classification: **partially load-bearing**. The component is reused by the live `QuestionPreview` surface, so deleting `QuestionCard.tsx` would break the question-authoring preview. But the API client it uses (`@/lib/quiz-api`) is dead — that's free to delete. The custom hook is isolated. The state shape — `selectedChoices: string[]` — happens to be compatible with the Day-12 answer-submission shape (a list of choice IDs).

This single file is now characterized: keep the component (it's load-bearing via the preview surface), replace its API client (dead), keep the state shape (compatible), keep or fold the custom hook (isolated, cheap either way). A **hybrid** path is forming naturally from the facts. You haven't decided yet — you've just made the facts visible.

Do this for every file in the inherited surface. The graph itself often makes the decision obvious. Where it doesn't, the judgement frameworks (below) kick in.

### 2. Technical debt assessment

With the dependency graph in hand, look for debt smells. Each smell is information — not necessarily a verdict.

**Smells to look for:**

- **Dead imports.** Code references modules, endpoints, or environment variables that no longer exist. The stripped AI surface leaves a trail of these. Cheap to spot, cheap to fix.
- **References to stripped services.** API clients pointing at `ai-quiz-service`, environment variables like `AI_QUIZ_ENDPOINT`, types modeling AI-generated questions. These will not work and cannot be revived in PEP.
- **Tightly coupled state.** A component that knows about the URL, the auth context, the API response shape, and the global Redux store all at once. Hard to test, hard to change, hard to reuse.
- **Missing tests.** Inherited code with no test coverage is harder to modify safely — you don't know what you'd break. Increases the *modify cost* significantly.
- **Pattern divergence.** The rest of the codebase uses react-hook-form; this surface uses raw `useState` chains. The rest uses server components for data fetching; this surface fetches client-side. Divergence isn't always wrong, but it's friction.

**The nuance — debt is a modify cost, not a rewrite trigger.** This is the single most important distinction in this section. A pile of debt makes modifying *more expensive* — it doesn't make rewriting *the correct answer*. Plenty of debt-laden code is also load-bearing and well-tested by its callers, so modify still wins on cost-benefit. Plenty of clean code is isolated and trivial to rewrite, so rewrite wins despite no debt being present. The mistake is reading "this code smells bad" as "therefore we should rewrite it." That's reasoning from aesthetics, not from cost.

Use debt to **price** the modify path. Then compare prices, don't compare aesthetics.

### 3. ROI horizon

Cost-benefit always raises the question: *cost and benefit measured over what time window?* That window is the ROI horizon. You must name it explicitly before any math.

**Two horizons matter for this decision:**

- **PEP horizon (~3 days of Week 3 remaining).** This is the operative horizon for the deliverable. Whatever you choose has to ship by Day 15.
- **Multi-year ownership horizon.** Pretend (or imagine, if you'd like) that this code will be owned for years. Clean architecture amortizes; technical debt compounds.

These two horizons frequently point opposite directions. **Modify often wins over the PEP horizon** because rewriting a multi-component flow from scratch eats Days 13–14 and risks the Day-15 integration deliverable. **Rewrite can win over a multi-year horizon** because three days of cleanup now beats years of working around inherited assumptions.

**You operate on the PEP horizon for this decision.** That's the curriculum's framing. But name it explicitly in your PR — "I chose to modify because over the remaining 3 days, the cost of a clean rewrite exceeds the cost of inherited assumptions" — not as a default. If your dependency mapping reveals isolated, lightly-coupled code with very few load-bearing dependencies, the PEP-horizon math may still favor rewrite.

**A small numeric example:**

Suppose your dependency graph and debt assessment produce these estimates:

| Path | Estimated hours to working state | Confidence in estimate |
|---|---|---|
| Modify | 6 hours | Low — unknowns hidden inside the inherited code |
| Rewrite | 10 hours | High — you control every line |
| Hybrid (keep `QuestionCard`, rewrite page + state + API integration) | 7 hours | Medium |

Over the PEP horizon (you have roughly 8 working hours per day, Days 13–14 are the build window), all three are technically feasible. But **modify is cheapest only if its estimate holds.** Confidence matters — a 6-hour modify with low confidence and a tail risk of "actually 14 hours when I discover the auth context wiring is broken" is more expensive than a 10-hour rewrite with high confidence. We'll formalize this in cost-benefit.

### 4. Cost-benefit (with the asymmetry made explicit)

Cost-benefit on this decision has a characteristic asymmetry you need to name:

- **Modify is often cheaper-but-uncertain.** The expected hours look low because you're leveraging existing UX work. But you don't know what you'll find inside until you're in. Hidden assumptions in the inherited code (a hook that depends on a global Context provider that's no longer mounted; a type that's "almost" the new contract's shape but off-by-one in a way that crashes runtime) can balloon the estimate after you're committed. **Variance is high.**
- **Rewrite is often more-expensive-but-predictable.** You're writing every line yourself, so you know the cost. There are no surprises hidden inside someone else's code. **Variance is low.**

A defensible cost-benefit doesn't just compare expected hours — it compares **risk-adjusted** expected hours.

**Worked estimate:**

Using the numbers from the ROI section:

| Path | Expected hours | Tail-risk hours (worst plausible case) | Risk-adjusted estimate (midpoint) |
|---|---|---|---|
| Modify | 6 | 14 | 10 |
| Rewrite | 10 | 12 | 11 |
| Hybrid | 7 | 11 | 9 |

Under this lens — and these numbers are illustrative, not prescriptive — hybrid is the cheapest risk-adjusted path, modify is competitive on expected value but loses ground when you weigh the tail, and rewrite buys predictability at a small premium.

**The benefit side** is usually shared across paths (the deliverable is the deliverable) but there's one subtle distinction worth naming: rewrite produces code you fully own and can defend in detail in peer review. Modify produces a Frankenstein you'll have to explain piece by piece — "this part is inherited, this part I changed, here's why I kept that." Both are defensible; the PR description will look different.

### 5. SWOT — framing each path

A SWOT table sharpens the trade-offs. It's particularly useful when the cost-benefit numbers are close (as they often are on this decision).

| | **Modify** | **Rewrite** | **Hybrid** |
|---|---|---|---|
| **Strengths** | Leverages existing UX work; preserves visible design decisions someone already thought through; respects the question-authoring preview surface if it depends on shared components | Clean fit to the Day-11/12 backend contract; fully owned by the candidate; no inherited assumptions to navigate around | Captures the cheap wins (keep what works) without paying full rewrite cost; respects load-bearing dependencies surfaced by static mapping |
| **Weaknesses** | Inherited assumptions (auth context shape, state location, API call ergonomics) constrain what you can do without cascade; debt is a modify-cost multiplier | Timeline risk — a multi-component flow takes longer than it looks; deletes UX work that may have been informed by trainer input you don't have | Requires the most careful planning — the seam between kept and rewritten code is where bugs hide; PR is harder to review |
| **Opportunities** | Ship faster; spend saved time on Day-14 interactivity polish | Establish patterns the rest of the team can copy; remove dead references in one sweep | Demonstrate analysis maturity in the PR — surface that you considered both extremes and chose the middle path on evidence |
| **Threats** | Hidden coupling surfaces during integration on Day 15 and breaks the acceptance criteria; debt smells multiply when modified rather than excised | Scope creep — "while I'm rewriting, I'll also fix..."; missing a Day-15 acceptance criterion because the rewrite ran long | The seam is fragile — passing data between kept and rewritten code can break at runtime in ways static analysis misses |

The SWOT is not a scorecard. It's a way of making the trade-offs visible to your future PR reviewer, who has to evaluate your decision without redoing your analysis. **Include this table — or its equivalent — in the PR description.**

### 6. Sunk-cost recognition (the counterweight)

Every framework above is a *forward-looking* tool. Sunk-cost recognition is the *backward-looking* counterweight — and it usually pulls in the opposite direction of where you'd naturally lean.

**The trap.** You've spent time understanding the inherited code. You've drawn the dependency graph. You've assessed the debt. You've felt the pull of "well, I've already invested four hours making sense of this code, I might as well finish modifying it." **Those four hours are sunk regardless of which path you choose.** The forward question is: from where you are right now, with the understanding you have now, what costs the least to ship?

If the answer is rewrite, the four hours weren't wasted — they produced the understanding that let you decide rewrite was cheaper. Treating them as a reason to keep going down the modify path is the sunk-cost fallacy in plain sight.

**The inverse trap.** Equally: don't keep inherited code *because it exists*. "Someone put work into this, it would be wasteful to delete it" is sunk-cost reasoning. The previous developer's work doesn't bind your forward decision either. Keep their code if cost-benefit and ROI horizon say to keep it. Delete it if they don't.

**A concrete recognition cue.** Whenever you find yourself saying "but I've already…" — stop. That phrase is the smell of sunk-cost reasoning. Replace it with "from here, what's cheapest?"

## Integration acceptance criteria — the result-side guardrail

The curriculum gives you wide freedom on the **path**. It is strict on the **result**. Whichever path you take — modify, rewrite, or hybrid — the deliverable must meet all of the following:

1. **Calls the Day-11/12 backend contract** authored by you or chosen on Day 11. The session-creation, question-fetching, and answer-submission endpoints established yesterday and the day before are the contract this UI consumes. No calls to stripped services. No calls to fabricated endpoints that don't exist on the running backend.
2. **Propagates auth context end-to-end.** Whatever auth model the inherited surface uses (cookie session, JWT in header, NextAuth, etc.), it must reach the API calls. A request without auth context fails the criterion regardless of how cleanly the UI renders. See [06-authentication-context-propagation-through-the-component-tree.md](06-authentication-context-propagation-through-the-component-tree.md) for the propagation patterns.
3. **Persists state across navigation.** A learner moving from question 2 to question 3 and back to question 2 must see their previously-selected answer still selected. State must outlive the route transition. See [09-multi-step-ui-navigation-without-state-loss.md](09-multi-step-ui-navigation-without-state-loss.md).
4. **Leaves no broken imports, dead references to stripped services, or non-functional UI elements behind.** If you modified, every reference to the stripped AI surface must be excised. If you rewrote, you must clean up the deleted files' importers. A "Submit AI Grade" button that does nothing is a non-functional UI element and fails the criterion.
5. **Renders whichever question shapes were finalized on Day 8.** The polymorphic-modeling decision from Day 8 — discriminated union, tagged enum, or per-type subclasses — produces a question payload of a specific shape. The UI must render every shape that backend will emit. See [08-polymorphic-component-rendering-for-variant-data-types.md](08-polymorphic-component-rendering-for-variant-data-types.md).

**This is what "fully integrated" means.** All five must be true on Day 15 when the slice is integrated end-to-end. The brownfield course gives the candidate freedom on PATH but is strict on RESULT.

## The decision is recorded in the PR

Your PR description for the Day 13 work includes three artifacts that together let a reviewer evaluate your decision without redoing your analysis:

1. **The chosen path** — one sentence at the top. "I chose to modify the inherited skeleton." or "I chose to rewrite from scratch." or "I chose a hybrid: kept `QuestionCard` and the layout, rewrote the page, state management, and API integration."
2. **The analysis summary that drove the choice** — the dependency graph (or a summary of it), the SWOT, and an explicit naming of your ROI horizon. Anyone reviewing the PR should be able to read this section and reconstruct *why* you chose what you chose. Do not write "rewrite is cleaner" without showing the math. Do not write "modify is faster" without showing the cost-benefit table.
3. **The acceptance-criteria checklist** — the five criteria above, ticked, with one-line evidence for each. ("Calls the Day-11/12 backend contract — see commit `abc123` wiring `fetchSession` to `POST /sessions`.")

This is the artifact the curriculum grades on. The path you chose is yours to defend. The criteria are non-negotiable.

## Example / worked scenario

You open `app/take/[testId]/` and find:

- `page.tsx` — a server component that renders a `QuizContainer`. Calls `getServerSession()`.
- `QuizContainer.tsx` — a client component owning question-index state. Imports `quizApi` (the dead AI client) and `QuestionCard`.
- `QuestionCard.tsx` — renders a question; selects choices; calls `quizApi.gradeAnswer()` on selection.
- `useQuestionState.ts` — a custom hook colocated with `QuestionCard`, manages selectedChoices and submitted state.

You run the analysis:

- **Static dependency mapping.** `QuestionCard` is also imported by `app/preview/QuestionPreview.tsx` — load-bearing. Everything else (`QuizContainer`, `useQuestionState`, `page.tsx`) is isolated. The `quizApi` client is dead.
- **Debt assessment.** Dead `quizApi` references throughout. `QuizContainer` couples auth, routing, state, and API in one component. No tests.
- **ROI horizon.** PEP horizon — 3 days remaining.
- **Cost-benefit.**
  - Modify: ~5 hours expected, ~12 hours tail (the coupling in `QuizContainer` is the unknown).
  - Rewrite: ~9 hours expected, ~11 hours tail.
  - Hybrid (keep `QuestionCard`, rewrite the rest): ~6 hours expected, ~9 hours tail.
- **SWOT.** Hybrid's seam is at `QuestionCard`'s prop interface — well-defined, low-risk. Modify's seam is everywhere. Rewrite forfeits `QuestionCard` (which is also used by question-authoring).
- **Sunk-cost check.** You spent 90 minutes on the analysis. That's sunk. From here, hybrid is cheapest risk-adjusted *and* preserves the load-bearing component.

**Decision: hybrid.** Keep `QuestionCard`. Rewrite `page.tsx`, `QuizContainer.tsx`, and replace `useQuestionState` with state lifted into the page. Delete the dead `quizApi`. Wire to the Day-11/12 contract directly with fetch calls in a server component for initial data and a client component for interaction.

**PR description records:** chosen path, analysis summary above (with the SWOT table and the numeric estimates), and the five-item acceptance-criteria checklist ticked on the integration commits.

## Common Pitfalls

- **Picking before analyzing.** Walking into the file tree, looking around for ten minutes, and declaring "let's rewrite" is not a decision — it's a guess. The dependency graph is non-negotiable; without it the other frameworks have no factual basis to operate on.
- **Treating sunk-cost as a reason to modify.** "I already spent four hours understanding this code" is irrelevant to the forward decision. So is "the previous developer worked hard on this." Both are sunk-cost reasoning and have to be excluded from the call.
- **Treating "rewrite is cleaner" as a reason without doing the ROI math.** Aesthetic preference for clean code is not an analysis output. If rewrite wins on ROI horizon and cost-benefit, defend it on those grounds. If it only wins on "feels cleaner," you haven't done the work yet.
- **Forgetting that integration acceptance criteria are non-negotiable.** "I rewrote it but didn't wire up auth" doesn't meet the bar regardless of how clean the rewrite was. The five criteria apply to the result of every path.
- **Treating "modify" as the default brownfield-friendly answer.** The brownfield mindset of [04-brownfield-mindset-reading-inherited-code-finding-the-seams.md](../day-01/04-brownfield-mindset-reading-inherited-code-finding-the-seams.md) is "read first, change second" — not "always modify." Rewrite is sometimes correct; the curriculum supports it explicitly. What's wrong is rewriting *without* analysis.
- **Skipping the PR analysis summary.** The decision lives in the PR description. A merge that doesn't record the analysis is indistinguishable from a guess, even when the underlying decision was sound.

## Key Takeaways

- Three paths — modify, rewrite, and hybrid — are all legitimate. The curriculum does not prescribe one. You own and defend the choice.
- Analysis precedes the choice. Static dependency mapping is the factual foundation; technical debt, ROI horizon, cost-benefit, SWOT, and sunk-cost recognition are the judgement layers built on top.
- The asymmetry to remember: modify is often cheaper-but-uncertain; rewrite is often more-expensive-but-predictable. Compare risk-adjusted estimates, not raw expected hours.
- Integration acceptance criteria are the result-side guardrail and they are non-negotiable: the Day-11/12 contract, end-to-end auth, state persistence across navigation, no dead references, and rendering every Day-8 question shape.
- The PR description records the chosen path, the analysis summary, and the ticked acceptance-criteria checklist. That's how the decision becomes defensible to your reviewer and to your future self.

---
*Prerequisites: [07-brownfield-decision-frameworks.md](../day-01/07-brownfield-decision-frameworks.md), [04-brownfield-mindset-reading-inherited-code-finding-the-seams.md](../day-01/04-brownfield-mindset-reading-inherited-code-finding-the-seams.md), [01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md](../day-08/01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md), [06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md](../day-10/06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md), [01-api-contract-design-under-inherited-ui-constraints.md](../day-11/01-api-contract-design-under-inherited-ui-constraints.md). Connects within Day 13 to [02-dynamic-routing-with-app-router-parameters.md](02-dynamic-routing-with-app-router-parameters.md), [03-server-components-for-initial-data-fetching.md](03-server-components-for-initial-data-fetching.md), [04-client-components-for-stateful-interactivity.md](04-client-components-for-stateful-interactivity.md), [05-type-safe-api-integration.md](05-type-safe-api-integration.md), [06-authentication-context-propagation-through-the-component-tree.md](06-authentication-context-propagation-through-the-component-tree.md), [08-polymorphic-component-rendering-for-variant-data-types.md](08-polymorphic-component-rendering-for-variant-data-types.md), and [09-multi-step-ui-navigation-without-state-loss.md](09-multi-step-ui-navigation-without-state-loss.md).*
