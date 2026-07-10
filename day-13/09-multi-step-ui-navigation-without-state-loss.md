# Multi-Step UI Navigation Without State Loss

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The candidate answers question 3, clicks "Next" to glance at question 4, clicks "Previous" to go back to 3 — and the selection has to still be there. Today's deliverable supports this navigation, but the obvious-looking implementation (push `?q=4` to the URL, let App Router re-render) silently destroys all selections on every nav. This topic explains the failure mode, then shows the right pattern: keep `currentIndex` and an `answers` Map in a single client component's state, never navigate at the framework level between questions, and treat the question-switch as a pure state update inside `<TestRunner>`. Sets up D14's autosave and server-anchored timer cleanly because the state is already centralized.

## The State We Have To Preserve

For each question the candidate has visited, we need to remember which options they selected. The structure is:

```ts
// Mapping: question_id → ordered list of selected option IDs
type AnswersMap = Map<string, number[]>;
```

Why a Map (or a plain object keyed by ID), not an array indexed by `currentIndex`:

- **Stable identity.** If the backend re-orders questions across reconnects (it shouldn't, but defensive design), the index changes; the question_id doesn't.
- **Sparse OK.** The candidate might skip question 4 and answer 5. A Map handles that; an array does too but introduces undefined slots that complicate iteration.
- **The D12 answer-submit payload references `question_id`, not index.** Aligning state shape to wire shape avoids translation bugs.

Day 13 only needs to *preserve* state during navigation. Day 14 adds *submitting* it. Today's goal: clicking Prev/Next doesn't lose the Map.

## The Wrong Pattern: URL-Driven Navigation

The first instinct from anyone familiar with multi-step web flows is to put the question index in the URL:

```
/take/0193d3a4-.../q/1
/take/0193d3a4-.../q/2
/take/0193d3a4-.../q/3
```

App Router routing with `/take/[testId]/q/[index]/page.tsx` makes this trivial to wire up. **It also breaks state preservation.**

Here's why. When the URL changes to a new dynamic route, Next.js:

1. Sees a different route segment.
2. Unmounts the current page tree.
3. Re-runs the server component for the new segment (re-fetching the session, in our setup).
4. Re-mounts the client subtree.

Re-mounting means every `useState` resets to its initial value. The Map of answers — gone. The candidate clicks "Previous" expecting to see their selection from question 3; the page renders question 3 with nothing selected.

You can paper over this with `localStorage` or React Query persistence, but you're now solving a problem you created. The right fix is to not navigate at all.

## The Right Pattern: Client-Held Index With `useState`

`<TestRunner>` (Topic 3, client component) owns `currentIndex` and `answers` in `useState`. Prev/Next mutate `currentIndex` locally — no URL change, no remount.

```tsx
// frontend/app/take/[testId]/TestRunner.tsx
"use client";

import { useState, useCallback } from "react";
import type { Session } from "@/lib/api/types";
import { QuestionView } from "./QuestionView";
import { Navigation } from "./Navigation";

type Props = { session: Session };

export function TestRunner({ session }: Props) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [answers, setAnswers] = useState<Map<string, number[]>>(() => new Map());

  const total = session.questions.length;
  const question = session.questions[currentIndex];
  const currentSelection = answers.get(question.question_id) ?? [];

  const setSelection = useCallback(
    (questionId: string, selected: number[]) => {
      setAnswers((prev) => {
        const next = new Map(prev);
        if (selected.length === 0) {
          next.delete(questionId);
        } else {
          next.set(questionId, selected);
        }
        return next;
      });
    },
    [],
  );

  const goPrev = () => setCurrentIndex((i) => Math.max(0, i - 1));
  const goNext = () => setCurrentIndex((i) => Math.min(total - 1, i + 1));

  return (
    <>
      <QuestionView
        question={question}
        selected={currentSelection}
        onChange={(sel) => setSelection(question.question_id, sel)}
      />
      <Navigation
        currentIndex={currentIndex}
        total={total}
        onPrev={goPrev}
        onNext={goNext}
      />
    </>
  );
}
```

Three things to internalize about this shape:

1. **`currentIndex` and `answers` live in the same component.** Sibling-component state would require lifting; child-component state would lose data on unmount. The parent of both is the natural home.
2. **The Map is updated immutably.** `new Map(prev)` clones, then mutates the clone. React relies on reference inequality to detect changes; mutating `prev` in place gives the same reference and skips the re-render.
3. **`useCallback` on `setSelection`** is mild micro-optimization — keeps the function reference stable so memoized children (D14 may add `React.memo` to the question components) don't re-render unnecessarily. Not load-bearing today.

## `useState` vs `useReducer` — When To Reach For Each

Both work. The threshold: if you have **more than two state slices that change together**, or **transitions with non-trivial logic** (e.g., "go to next, but only if the current question is answered"), `useReducer` keeps the logic centralized and testable.

For Day 13 the transitions are trivial (`currentIndex++/--`), so `useState` is fine. For Day 14, when "submit current answer → advance" becomes a single user action with several effects (record answer in Map, call `POST /sessions/{id}/answer`, advance index, clear loading state, handle errors), the right move is to refactor to `useReducer`:

```tsx
type State = {
  currentIndex: number;
  answers: Map<string, number[]>;
  status: "idle" | "submitting" | "error";
  error: string | null;
};

type Action =
  | { type: "select"; questionId: string; selected: number[] }
  | { type: "submit_start" }
  | { type: "submit_success"; nextIndex: number }
  | { type: "submit_error"; error: string }
  | { type: "navigate"; index: number };

function runnerReducer(state: State, action: Action): State {
  switch (action.type) {
    case "select":
      return { ...state, answers: new Map(state.answers).set(action.questionId, action.selected) };
    case "submit_start":
      return { ...state, status: "submitting", error: null };
    case "submit_success":
      return { ...state, status: "idle", currentIndex: action.nextIndex };
    case "submit_error":
      return { ...state, status: "error", error: action.error };
    case "navigate":
      return { ...state, currentIndex: action.index };
  }
}
```

For Day 13, hold off on the reducer. Adopt it tomorrow when the complexity earns it.

## What About Browser Back/Forward?

By keeping question navigation out of the URL, we lose browser-level back/forward navigation between questions. This is the right tradeoff:

- **Browser back from question 5 should mean "leave the test", not "go to question 4".** Mixing the two is confusing and accidental back-presses lose progress.
- **The "Previous" button is the explicit affordance** for moving between questions. It's clearly labeled, it's in the navigation footer, candidates find it.
- **The browser back gesture from the test page should arguably warn before leaving.** Day 14 will hook `beforeunload` to confirm before navigating away — that's the right place to handle accidental backs.

If a future requirement demands deep-linkable per-question URLs (e.g., for proctor review), implement it with searchParams that update without remount (`router.replace`, no `await`) and read the index from URL on first mount only. But don't take that complexity on without a concrete reason.

## What Goes Wrong With State Loss

A short catalog of bugs the centralized client-state pattern prevents:

- **Selections wipe on nav.** Most visible failure; candidate refuses to use Previous, then submits without reviewing.
- **Selections wipe on hot reload during development.** Dev productivity tanks because every code save loses test progress. (`useState` survives Fast Refresh; `localStorage` doesn't help with Fast Refresh; sticking state in a Map you discard on remount is the worst combination.)
- **Selections wipe on accidental browser back.** The candidate hits back, loses everything, the test session itself is still alive server-side but the UI thinks it's fresh.
- **Selections wipe on transient network blip if state is server-derived.** If we fetched per-question and stored only the *current* answer in state, a re-render would re-fetch and lose anything not yet submitted. The all-questions-up-front design from D11 avoids this; the client state pattern reinforces it.

## Composing With The Other Topics

Here's how Topic 8's state shape connects to the rest of Day 13:

- **Topic 1 (dynamic routing):** the route resolves once on page load; never changes during the quiz.
- **Topic 2 (server components):** the session JSON is fetched once on page load; never re-fetched.
- **Topic 3 (client components):** the entire `<TestRunner>` subtree is one client island that never unmounts.
- **Topic 4 (type-safe API):** `Map<string, number[]>` mirrors the wire shape (`AnswerSubmit.selected_options`).
- **Topic 5 (auth):** the auth cookie is implicit; the session token is in `session.session_token` and stays in the client island for D14's submit calls.
- **Topic 6 (layout):** the layout is static; only the `<QuestionView>` slot changes with `currentIndex`.
- **Topic 7 (polymorphic rendering):** `<QuestionView>` swaps internals based on `question.type`; the parent's state shape is uniform.

All seven prior topics converge on this state-centralized client island. The state shape is the contract that makes tomorrow's autosave, timer, and submit-and-lock features land cleanly.

## Common Mistakes

- **URL-driven question navigation.** Remounts on every nav, destroys state. Use local state.
- **Mutating the Map in place** with `prev.set(...)` and returning `prev`. React skips the re-render. Always `new Map(prev)` then mutate the clone.
- **Storing selections in the child component.** `<SingleSelectQuestion>` unmounts when `currentIndex` changes; child state goes with it. Keep selections in the parent.
- **Keying answers by index instead of `question_id`.** Couples your state to a presentation concern that may change.
- **Reaching for Zustand or Redux on Day 13.** `useState` in one component is plenty. Add a state library when there are 3+ components that need to share state and prop-drilling becomes painful — and even then, React's `useContext` often suffices.
- **`useEffect` to sync state to URL.** Re-introduces the remount problem and adds an effect to debug. Don't.

## Key Takeaways
- The test-taking page holds `currentIndex` and an `answers: Map<string, number[]>` in a single client component's state — sibling to neither, child to neither.
- Navigation between questions is a local `setCurrentIndex` call, **not** a URL change; framework navigation remounts the subtree and discards state.
- Immutably update the Map (`new Map(prev).set(...)`) so React detects the change and re-renders.
- Key answers by `question_id` (stable, matches wire shape), not by index (presentation-coupled).
- `useState` is sufficient today; refactor to `useReducer` tomorrow when submit-flow logic justifies the centralization.
- Browser back/forward intentionally doesn't move between questions; the labeled "Previous" button is the affordance.

---
*Prerequisites: [03-form-state-management-with-react-hook-form.md](../day-09/03-form-state-management-with-react-hook-form.md), [04-client-components-for-stateful-interactivity.md](04-client-components-for-stateful-interactivity.md), [08-polymorphic-component-rendering-for-variant-data-types.md](08-polymorphic-component-rendering-for-variant-data-types.md). Forward references: [01-react-state-management-patterns-usestate-vs-usereducer.md](../day-14/01-react-state-management-patterns-usestate-vs-usereducer.md), [06-submit-and-lock-ux-patterns.md](../day-14/06-submit-and-lock-ux-patterns.md), [04-client-side-temporal-state-timers-and-autosave.md](../day-14/04-client-side-temporal-state-timers-and-autosave.md).*
