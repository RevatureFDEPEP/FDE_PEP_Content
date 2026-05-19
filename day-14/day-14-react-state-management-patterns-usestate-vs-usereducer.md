# React State Management Patterns (useState vs useReducer)

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D13's `<TestRunner>` shipped with two `useState` calls — `currentIndex` and an `answers` Map. That worked because the skeleton only had two actions: change the index, change a question's selection. D14 adds autosave status per question, timer state, submit-in-flight, submit error, and a "locked" terminal state. Once five or six pieces of related state are sloshing around the same component, `useState` starts to lie about what the component actually does, and the call sites that update it become a series of disconnected setters that have to be kept consistent by hand. This topic is about recognizing when that line gets crossed and migrating cleanly from `useState` to `useReducer` — and, just as importantly, when *not* to migrate.

## The Two-Question Heuristic

Before reaching for `useReducer`, ask two questions about the state in front of you:

1. **Do multiple pieces of state change together as part of a single user action?**
2. **Are there transitions that are illegal — combinations of values that should never coexist?**

If the answer to both is "yes," you have a reducer. If the answer to both is "no," `useState` is fine and `useReducer` is overkill.

For the D14 test-taking page:

- Clicking "Submit" changes `submitStatus: "idle" → "submitting"`, must clear `submitError`, must disable navigation. **Three setters, one user action.**
- A `submitStatus` of `"submitting"` with `lockedAt: Date` set is incoherent — submission isn't both in flight and finished. **Illegal combination.**

Both yeses. Reducer.

For the timer's `secondsRemaining`:

- Changes every tick, on its own, not tied to other state.
- No illegal combinations — any non-negative integer is valid.

`useState` (or a custom hook backed by `useState`) is the right call there.

## The `useState` Smell On D13's `<TestRunner>`

If we naively grow D13's component, we end up with this:

```tsx
// frontend/app/take/[testId]/TestRunner.tsx — DON'T do this on D14
"use client";

export function TestRunner({ session }: Props) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [answers, setAnswers] = useState<Map<string, number[]>>(() => new Map());
  const [autosaveStatus, setAutosaveStatus] = useState<Map<string, "idle" | "saving" | "saved" | "error">>(() => new Map());
  const [submitStatus, setSubmitStatus] = useState<"idle" | "submitting" | "submitted" | "error">("idle");
  const [submitError, setSubmitError] = useState<string | null>(null);
  const [lockedAt, setLockedAt] = useState<string | null>(null);

  // Submit handler — six setters, easy to forget one:
  const handleSubmit = async () => {
    setSubmitStatus("submitting");
    setSubmitError(null);
    try {
      const result = await api.submitSession(session.session_id);
      setSubmitStatus("submitted");
      setLockedAt(result.locked_at);
    } catch (e) {
      setSubmitStatus("error");
      setSubmitError(String(e));
      // Forgot to clear lockedAt? Did we need to?
    }
  };
  // ...
}
```

Six `useState` calls. The submit handler reads more like a checklist than a state transition. Worse, the *invariants* are nowhere expressed: nothing prevents `submitStatus === "submitted"` and `lockedAt === null` from coexisting, and nothing stops a stray re-render path from setting `submitStatus = "idle"` mid-submit.

## The `useReducer` Refactor

Group the state by what changes together. There are two natural state machines on this page:

1. **Answer state** — `currentIndex`, `answers`, `autosaveStatus`. Driven by navigation and answer changes.
2. **Submission state** — `submitStatus`, `submitError`, `lockedAt`. Driven by the submit lifecycle.

Two reducers, not one. Don't over-merge — combining unrelated state under one reducer gives you a reducer that's just a big switch dispatching to a god-object.

Here's the answer-state reducer. The submission-state reducer follows the same pattern.

```ts
// frontend/app/take/[testId]/answerReducer.ts
export type AutosaveStatus = "idle" | "saving" | "saved" | "error";

export type AnswerState = {
  currentIndex: number;
  answers: Map<string, number[]>;
  autosave: Map<string, AutosaveStatus>;
};

export type AnswerAction =
  | { type: "NAVIGATE"; index: number }
  | { type: "SET_SELECTION"; questionId: string; selected: number[] }
  | { type: "AUTOSAVE_START"; questionId: string }
  | { type: "AUTOSAVE_SUCCESS"; questionId: string }
  | { type: "AUTOSAVE_ERROR"; questionId: string };

export const initialAnswerState: AnswerState = {
  currentIndex: 0,
  answers: new Map(),
  autosave: new Map(),
};

export function answerReducer(state: AnswerState, action: AnswerAction): AnswerState {
  switch (action.type) {
    case "NAVIGATE":
      return { ...state, currentIndex: action.index };

    case "SET_SELECTION": {
      const answers = new Map(state.answers);
      if (action.selected.length === 0) {
        answers.delete(action.questionId);
      } else {
        answers.set(action.questionId, action.selected);
      }
      // Setting a new selection invalidates any prior autosave success.
      const autosave = new Map(state.autosave);
      autosave.set(action.questionId, "idle");
      return { ...state, answers, autosave };
    }

    case "AUTOSAVE_START": {
      const autosave = new Map(state.autosave);
      autosave.set(action.questionId, "saving");
      return { ...state, autosave };
    }

    case "AUTOSAVE_SUCCESS": {
      const autosave = new Map(state.autosave);
      autosave.set(action.questionId, "saved");
      return { ...state, autosave };
    }

    case "AUTOSAVE_ERROR": {
      const autosave = new Map(state.autosave);
      autosave.set(action.questionId, "error");
      return { ...state, autosave };
    }

    default: {
      const _exhaustive: never = action;
      return state;
    }
  }
}
```

A few specific points to internalize, because they recur every time you write a reducer in TypeScript:

- **The discriminated union `AnswerAction`.** This is what makes the reducer type-safe. Each `case` narrows `action` to the right shape; you can't accidentally read `action.questionId` from a `NAVIGATE` action.
- **The `_exhaustive: never` trick in `default`.** Add a new action type to the union, forget to handle it in the switch, and TypeScript fails to compile because the unhandled action is not `never`. This is your guard against silently ignored actions.
- **Always return a new object reference.** React relies on reference equality to decide whether to re-render. Mutating `state.answers` in place and returning `state` will *not* re-render. Hence `new Map(state.answers)` everywhere.
- **Reducers must be pure.** No `fetch`, no `Date.now()`, no `setTimeout`. Side effects live in the component that dispatches actions — usually inside `useEffect` or event handlers. This purity is what makes reducers trivially testable (Topic 8).

## Wiring It In `<TestRunner>`

```tsx
// frontend/app/take/[testId]/TestRunner.tsx
"use client";

import { useReducer } from "react";
import { answerReducer, initialAnswerState } from "./answerReducer";
import { submitReducer, initialSubmitState } from "./submitReducer";
// ... other imports

export function TestRunner({ session }: Props) {
  const [answerState, dispatchAnswer] = useReducer(answerReducer, initialAnswerState);
  const [submitState, dispatchSubmit] = useReducer(submitReducer, initialSubmitState);

  const question = session.questions[answerState.currentIndex];
  const currentSelection = answerState.answers.get(question.question_id) ?? [];
  const isLocked = submitState.status === "submitted";

  return (
    <div className="space-y-6">
      <QuestionView
        question={question}
        selected={currentSelection}
        disabled={isLocked}
        onChange={(selected) =>
          dispatchAnswer({ type: "SET_SELECTION", questionId: question.question_id, selected })
        }
      />
      <Navigation
        currentIndex={answerState.currentIndex}
        total={session.questions.length}
        disabled={isLocked}
        onPrev={() => dispatchAnswer({ type: "NAVIGATE", index: Math.max(0, answerState.currentIndex - 1) })}
        onNext={() => dispatchAnswer({ type: "NAVIGATE", index: Math.min(session.questions.length - 1, answerState.currentIndex + 1) })}
      />
    </div>
  );
}
```

The component shrinks. Every state change goes through a typed action. The reducer is one file, testable in isolation (Topic 8). The component is one file, focused on wiring actions to UI.

## When Not To Reach For `useReducer`

It is genuinely possible to over-engineer this. Cases where `useState` is still the right call:

- **A single boolean toggle** (`isOpen`, `isExpanded`). The action is the value.
- **A controlled input's local string state.** Use `useState` (or just react-hook-form, Topic 2).
- **Derived values.** Don't put computed-from-other-state into state at all — derive it during render or use `useMemo` if expensive.
- **Independent unrelated state slices.** Two `useState` calls for two unrelated values is clearer than one reducer with a union action type.

The migration is not free: a reducer means an action type, a reducer function, and a dispatch call where you used to call a setter. That overhead pays off when state coordination becomes complex, and is wasted when it doesn't.

## Comparison Table

| Concern | `useState` | `useReducer` |
|---|---|---|
| Single value, simple updates | Yes | Overkill |
| Multiple values that change together | Awkward (must remember every setter) | Natural (one action) |
| Illegal state combinations to prevent | Hard to enforce | Express via action design + reducer logic |
| Testability of state transitions | Test the component | Test the reducer in isolation (pure function) |
| State updates depend on prior state | `setX(prev => ...)` works | Reducer always receives prior state |
| Action history / debugging | No built-in story | Dispatch log is a free audit trail |
| Boilerplate | Minimal | Action union + reducer + dispatch |

## Common Mistakes

- **One mega-reducer for everything.** Two unrelated state machines in one reducer is a code smell. Split them — D14 has at least two: answer state and submit state.
- **Mutating state inside the reducer.** Mutating `state.answers` (instead of cloning) breaks React's change detection. Always return new object references.
- **Side effects in the reducer.** `fetch`, `Date.now()`, `Math.random()` — none of these belong. Dispatch an action *describing* what happened; perform the side effect in the component.
- **Putting derived state into the reducer.** `answers.size` should be computed on read, not stored as `answerCount` in the state.
- **Forgetting the `never` exhaustiveness check.** Without it, a new action type silently goes unhandled.

## Key Takeaways
- `useState` is right when state changes are independent and small; `useReducer` is right when multiple values change together and illegal combinations must be prevented.
- D14's test-taking page has two natural reducers: answer state (index + answers + autosave status) and submission state (status + error + lockedAt).
- TypeScript discriminated unions + `_exhaustive: never` make the reducer compiler-checked: forgetting to handle a new action is a build error.
- Reducers must be pure — no fetches, no `Date.now()`, no mutation — which makes them trivially unit-testable (Topic 8).
- The migration from D13's two `useState`s to D14's two reducers is a cost; pay it only when state coordination justifies it.

---
*Prerequisites: day-13-client-components-for-stateful-interactivity, day-13-multi-step-ui-navigation-without-state-loss. Forward references: day-14-component-unit-testing-patterns-vitest, day-14-submit-and-lock-ux-patterns, day-14-client-side-temporal-state-timers-and-autosave.*
