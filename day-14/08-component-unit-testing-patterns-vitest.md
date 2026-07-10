# Component Unit Testing Patterns (Vitest) — Covering State Reducers And Pure Logic

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Component unit testing closes a gap the frontend has been tolerating: the backend has had pytest from D8 forward, but the frontend's testing story has been implicit. Today changes that. If Vitest isn't already wired into `frontend/`, wiring it up is part of this topic — the Setup section below covers it. The curriculum does not assume the runner is pre-installed; treat both "the runner exists" and "I have to add it" as in-scope today. The reducers from Topic 1, the pure helpers like `formatRemaining` from Topic 4, the network-error classifier from Topic 7, and the answer-validation schema from Topic 3 — all of these are pure functions or easily-isolated state machines, and they are exactly what unit tests are good at. This topic covers the Vitest setup, the testing conventions that make tests legible, and worked examples for the D13 reducer transitions and the `formatRemaining(ms)` helper. React Testing Library gets a brief mention; the heavier component-rendering tests land on D15.

## Why Vitest, Not Jest

Vitest is the unit-test runner aligned with Vite/Next.js's ESM-first tooling. Compared to Jest:

- **No Babel config needed for TypeScript/ESM.** Vitest uses the same transform pipeline as the dev server.
- **Vite-native module resolution.** Path aliases (`@/lib/...`) just work without separate Jest config.
- **Fast.** Workers + native ESM = milliseconds-per-test for unit code.
- **API-compatible with Jest.** `describe`, `it`, `expect` work identically; migration cost is near zero.

The Next.js docs cover both; this project standardized on Vitest because the rest of the frontend toolchain is Vite-adjacent.

## Setup

Install:

```bash
pnpm add -D vitest @vitest/coverage-v8 @testing-library/react @testing-library/dom jsdom
```

`vitest.config.ts` (or extend an existing `vite.config.ts`):

```ts
// frontend/vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",            // for components; "node" for pure logic-only files
    globals: true,                    // expose describe/it/expect without imports
    setupFiles: ["./vitest.setup.ts"], // optional; register Testing Library matchers etc.
    coverage: {
      provider: "v8",
      reporter: ["text", "html"],
      include: ["app/**/*.{ts,tsx}", "lib/**/*.{ts,tsx}"],
      exclude: ["**/*.d.ts", "**/types/**"],
    },
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "."),
    },
  },
});
```

Script in `package.json`:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

Hook this into CI alongside the existing pytest job (see D7 patterns).

## Testing The Pure Helper: `formatRemaining`

Pure functions are the easiest possible test target. Single input, single output, no setup, no mocks. Start every new test file with a pure helper if there is one — it builds muscle memory.

```ts
// frontend/app/take/[testId]/formatRemaining.test.ts
import { describe, it, expect } from "vitest";
import { formatRemaining } from "./formatRemaining";

describe("formatRemaining", () => {
  it("formats sub-minute durations as M:SS", () => {
    expect(formatRemaining(45_000)).toBe("0:45");
    expect(formatRemaining(9_000)).toBe("0:09");
  });

  it("formats minute durations as M:SS", () => {
    expect(formatRemaining(60_000)).toBe("1:00");
    expect(formatRemaining(125_000)).toBe("2:05");
    expect(formatRemaining(599_000)).toBe("9:59");
  });

  it("formats hour-plus durations as H:MM:SS", () => {
    expect(formatRemaining(3_600_000)).toBe("1:00:00");
    expect(formatRemaining(3_725_000)).toBe("1:02:05");
    expect(formatRemaining(36_000_000)).toBe("10:00:00");
  });

  it("returns 0:00 for zero", () => {
    expect(formatRemaining(0)).toBe("0:00");
  });

  it("floors sub-second values down to the lower second", () => {
    expect(formatRemaining(1_999)).toBe("0:01");
    expect(formatRemaining(999)).toBe("0:00");
  });

  it("handles negative input gracefully (clamp to zero)", () => {
    // Defensive: the timer hook should never pass negative, but the helper should not throw.
    expect(formatRemaining(-1)).toBe("0:00");
  });
});
```

Five things every test in this file does right:

1. **One concern per `it`.** Sub-minute, minute, hour, zero, sub-second, negative — each is a separate scenario.
2. **Descriptive `it` names that read as English.** Test names show up in CI output; `"formats sub-minute durations as M:SS"` is more useful than `"test1"`.
3. **Multiple assertions per scenario when they're the same conceptual case.** Three sub-minute examples reinforce the rule without inflating the test count.
4. **Edge cases explicitly named.** Zero, negative, sub-second — boundaries the implementation must handle and the test must document.
5. **No mocks, no setup, no async.** This is what makes pure-helper tests fast and trustworthy.

For the "clamp negative" case to pass, `formatRemaining` from Topic 4 needs `Math.max(0, Math.floor(ms / 1000))`. That's the kind of defensive detail that tests force you to think about.

## Testing The Reducer: State Transitions

The D13 reducer (and the D14 extensions in Topic 1) is a pure function: `(state, action) → newState`. Test inputs and outputs; never test the component that *uses* the reducer in the reducer's test file.

```ts
// frontend/app/take/[testId]/answerReducer.test.ts
import { describe, it, expect } from "vitest";
import { answerReducer, initialAnswerState, type AnswerState } from "./answerReducer";

describe("answerReducer", () => {
  describe("NAVIGATE", () => {
    it("updates currentIndex", () => {
      const next = answerReducer(initialAnswerState, { type: "NAVIGATE", index: 3 });
      expect(next.currentIndex).toBe(3);
    });

    it("preserves answers and autosave maps", () => {
      const state: AnswerState = {
        currentIndex: 0,
        answers: new Map([["q1", [42]]]),
        autosave: new Map([["q1", "saved"]]),
      };
      const next = answerReducer(state, { type: "NAVIGATE", index: 1 });
      expect(next.answers.get("q1")).toEqual([42]);
      expect(next.autosave.get("q1")).toBe("saved");
    });

    it("returns a new state object (reference inequality)", () => {
      // Critical for React re-render: state must not be mutated in place.
      const next = answerReducer(initialAnswerState, { type: "NAVIGATE", index: 1 });
      expect(next).not.toBe(initialAnswerState);
    });
  });

  describe("SET_SELECTION", () => {
    it("adds a new answer to an empty map", () => {
      const next = answerReducer(initialAnswerState, {
        type: "SET_SELECTION",
        questionId: "q1",
        selected: [42],
      });
      expect(next.answers.get("q1")).toEqual([42]);
    });

    it("replaces an existing answer", () => {
      const state: AnswerState = {
        ...initialAnswerState,
        answers: new Map([["q1", [42]]]),
      };
      const next = answerReducer(state, {
        type: "SET_SELECTION",
        questionId: "q1",
        selected: [7, 9],
      });
      expect(next.answers.get("q1")).toEqual([7, 9]);
    });

    it("deletes the entry when selected is empty", () => {
      // Per the reducer's invariant: empty selection means unanswered, not "answered with nothing"
      const state: AnswerState = {
        ...initialAnswerState,
        answers: new Map([["q1", [42]]]),
      };
      const next = answerReducer(state, {
        type: "SET_SELECTION",
        questionId: "q1",
        selected: [],
      });
      expect(next.answers.has("q1")).toBe(false);
    });

    it("resets autosave status to idle when selection changes", () => {
      // Invariant: a fresh selection invalidates any prior saved/error status
      const state: AnswerState = {
        ...initialAnswerState,
        answers: new Map([["q1", [42]]]),
        autosave: new Map([["q1", "saved"]]),
      };
      const next = answerReducer(state, {
        type: "SET_SELECTION",
        questionId: "q1",
        selected: [99],
      });
      expect(next.autosave.get("q1")).toBe("idle");
    });

    it("does not mutate the input answers map", () => {
      const answers = new Map([["q1", [42]]]);
      const state: AnswerState = { ...initialAnswerState, answers };
      answerReducer(state, { type: "SET_SELECTION", questionId: "q1", selected: [99] });
      // Original map untouched
      expect(answers.get("q1")).toEqual([42]);
    });
  });

  describe("AUTOSAVE_START / SUCCESS / ERROR", () => {
    it("transitions autosave state through saving → saved", () => {
      let state = initialAnswerState;
      state = answerReducer(state, { type: "AUTOSAVE_START", questionId: "q1" });
      expect(state.autosave.get("q1")).toBe("saving");

      state = answerReducer(state, { type: "AUTOSAVE_SUCCESS", questionId: "q1" });
      expect(state.autosave.get("q1")).toBe("saved");
    });

    it("transitions autosave state through saving → error", () => {
      let state = initialAnswerState;
      state = answerReducer(state, { type: "AUTOSAVE_START", questionId: "q1" });
      state = answerReducer(state, { type: "AUTOSAVE_ERROR", questionId: "q1" });
      expect(state.autosave.get("q1")).toBe("error");
    });

    it("autosave state changes do not touch the answers map", () => {
      const state: AnswerState = {
        ...initialAnswerState,
        answers: new Map([["q1", [42]]]),
      };
      const next = answerReducer(state, { type: "AUTOSAVE_SUCCESS", questionId: "q1" });
      expect(next.answers.get("q1")).toEqual([42]);
    });
  });
});
```

The structure mirrors the reducer: one `describe` per action type, multiple `it`s covering the dimensions that action affects. *Invariants* — like "SET_SELECTION resets autosave status" and "reducer doesn't mutate input" — get their own `it`s and explicit comments explaining *why* the invariant matters. Future-you reads the comment when something breaks.

The reference-inequality check (`expect(next).not.toBe(initialAnswerState)`) is the test version of the React-render-invalidation rule: forgetting to clone state breaks rendering, and you want a test that catches that regression immediately.

## Testing The Submit Reducer

Same pattern, focused on the state-machine *transitions*. Test the legal ones, then test the illegal ones (which should be no-ops).

```ts
// frontend/app/take/[testId]/submitReducer.test.ts
import { describe, it, expect } from "vitest";
import { submitReducer, initialSubmitState } from "./submitReducer";

describe("submitReducer transitions", () => {
  it("idle → submitting on SUBMIT_START", () => {
    const next = submitReducer(initialSubmitState, { type: "SUBMIT_START" });
    expect(next.kind).toBe("submitting");
  });

  it("submitting → submitted on SUBMIT_SUCCESS", () => {
    const state = submitReducer(initialSubmitState, { type: "SUBMIT_START" });
    const next = submitReducer(state, {
      type: "SUBMIT_SUCCESS",
      lockedAt: "2026-05-19T12:00:00Z",
      sessionId: "sess_123",
    });
    expect(next).toEqual({
      kind: "submitted",
      lockedAt: "2026-05-19T12:00:00Z",
      sessionId: "sess_123",
    });
  });

  it("submitted is terminal: SUBMIT_START is a no-op", () => {
    // Defense against double-submit: even if a stray dispatch arrives, the state doesn't regress.
    const submittedState = {
      kind: "submitted" as const,
      lockedAt: "2026-05-19T12:00:00Z",
      sessionId: "sess_123",
    };
    const next = submitReducer(submittedState, { type: "SUBMIT_START" });
    expect(next).toBe(submittedState); // same reference, same value
  });

  it("error_terminal_409 is terminal: SUBMIT_START is a no-op", () => {
    const state = { kind: "error_terminal_409" as const };
    const next = submitReducer(state, { type: "SUBMIT_START" });
    expect(next).toBe(state);
  });

  it("error_recoverable → submitting on retry", () => {
    const state = { kind: "error_recoverable" as const, message: "Network failed" };
    const next = submitReducer(state, { type: "SUBMIT_START" });
    expect(next.kind).toBe("submitting");
  });
});
```

The terminal-state tests are the most valuable here, because they test a *non-event*: dispatching an action and asserting *nothing happened*. This is the kind of behavior that's easy to break inadvertently when refactoring the reducer.

## Testing The Network Error Classifier

The classifier from Topic 7 is pure and benefits from table-driven tests:

```ts
// frontend/app/take/[testId]/classifyNetworkError.test.ts
import { describe, it, expect, beforeEach, vi } from "vitest";
import { classifyNetworkError } from "./classifyNetworkError";

describe("classifyNetworkError", () => {
  beforeEach(() => {
    vi.stubGlobal("navigator", { onLine: true });
  });

  it("classifies 5xx as transient retryable", () => {
    const res = new Response(null, { status: 503 });
    expect(classifyNetworkError(res)).toEqual({
      kind: "transient",
      retryable: true,
      httpStatus: 503,
    });
  });

  it("classifies 408 and 429 as transient retryable", () => {
    expect(classifyNetworkError(new Response(null, { status: 408 })).kind).toBe("transient");
    expect(classifyNetworkError(new Response(null, { status: 429 })).kind).toBe("transient");
  });

  it("classifies 4xx (other) as semantic non-retryable", () => {
    const res = new Response(null, { status: 422 });
    expect(classifyNetworkError(res)).toEqual({
      kind: "semantic",
      retryable: false,
      httpStatus: 422,
    });
  });

  it("classifies TypeError (fetch failure) as transient", () => {
    expect(classifyNetworkError(new TypeError("Failed to fetch")).kind).toBe("transient");
  });

  it("overrides everything to connectivity when navigator.onLine is false", () => {
    vi.stubGlobal("navigator", { onLine: false });
    const res = new Response(null, { status: 503 });
    expect(classifyNetworkError(res).kind).toBe("connectivity");
  });
});
```

`vi.stubGlobal` is Vitest's mechanism for replacing global objects per-test, scoped so other tests aren't polluted. This is how we test code that reads `navigator.onLine` without touching the real one.

## Testing The Answer-Validation Schema

Zod schemas from Topic 3 are perfect for table-driven validation tests:

```ts
// frontend/app/take/[testId]/answerValidation.test.ts
import { describe, it, expect } from "vitest";
import { answerSchemaFor } from "./answerValidation";

const singleSelectQ = {
  question_id: "q1",
  type: "single_select" as const,
  options: [
    { option_id: 1, text: "A" },
    { option_id: 2, text: "B" },
    { option_id: 3, text: "C" },
  ],
};

const multiSelectQ = { ...singleSelectQ, type: "multi_select" as const };

describe("answerSchemaFor (single_select)", () => {
  const schema = answerSchemaFor(singleSelectQ);

  it("accepts a single valid option", () => {
    expect(schema.safeParse([1]).success).toBe(true);
  });

  it("rejects zero options", () => {
    expect(schema.safeParse([]).success).toBe(false);
  });

  it("rejects multiple options", () => {
    expect(schema.safeParse([1, 2]).success).toBe(false);
  });

  it("rejects an unknown option_id", () => {
    expect(schema.safeParse([99]).success).toBe(false);
  });
});

describe("answerSchemaFor (multi_select)", () => {
  const schema = answerSchemaFor(multiSelectQ);

  it("accepts an empty array", () => {
    expect(schema.safeParse([]).success).toBe(true);
  });

  it("accepts multiple distinct valid options", () => {
    expect(schema.safeParse([1, 3]).success).toBe(true);
  });

  it("rejects duplicates", () => {
    expect(schema.safeParse([1, 1]).success).toBe(false);
  });

  it("rejects unknown option_ids mixed with valid ones", () => {
    expect(schema.safeParse([1, 99]).success).toBe(false);
  });
});
```

## React Testing Library — A Brief Note

For component-level tests (rendering `<TestRunner>` and simulating clicks), React Testing Library is the standard partner to Vitest. The setup is minimal:

```tsx
// vitest.setup.ts
import "@testing-library/jest-dom/vitest"; // matchers like toBeInTheDocument

// Example
import { render, screen, fireEvent } from "@testing-library/react";
import { Timer } from "./Timer";

it("renders the timer text", () => {
  render(<Timer serverNow="2026-05-19T12:00:00Z" expiresAt="2026-05-19T13:00:00Z" onExpire={() => {}} />);
  expect(screen.getByRole("timer")).toBeInTheDocument();
});
```

These are valuable, but they're slower and more brittle than the pure-logic tests above — and they're the focus of D15's component testing pass, not today's. For D14, keep the testing pyramid weighted toward unit tests on reducers, helpers, classifiers, and schemas. The component-level tests come tomorrow.

## What To Run In CI

```yaml
# .github/workflows/frontend.yml (excerpt)
- name: Run frontend unit tests
  working-directory: frontend
  run: pnpm test
```

Coverage thresholds for D14's deliverable are pragmatic: 100% on reducers and pure helpers (they're trivial), >70% overall as the bar to push above. Don't chase 100% coverage on UI components today — that's chasing a false metric.

## Common Mistakes

- **Testing the reducer through the component.** The reducer is pure; test it directly. Reaching for `render(<TestRunner>)` to assert state transitions is slow, brittle, and conflates two test concerns.
- **Sharing mutable state between tests.** Each `it` should start fresh. If you find yourself writing a `beforeEach` that resets a module-level variable, the test is probably depending on global state it shouldn't.
- **Asserting on implementation details.** Test the reducer's output, not which internal helper it calls. Behavior is the contract.
- **Skipping the reference-inequality check.** Forgetting that returning the same state object breaks React rendering is a regression waiting to happen.
- **No tests for terminal-state no-ops.** The "submitting cannot go back to idle" rule is precisely the kind of behavior that breaks under careless refactor and is invisible without a test asserting "this dispatch is a no-op."
- **Treating Vitest like Jest with all the Jest baggage.** Don't add `ts-jest`, `babel-jest`, etc.; you don't need them.

## Key Takeaways
- Vitest aligns with the project's ESM/Vite tooling and is API-compatible with Jest — fast tests, minimal config, no transform pipeline to fight.
- Reducers (Topic 1) and pure helpers (`formatRemaining`, `classifyNetworkError`, zod schemas) are the right test targets for D14 — they're pure, fast, and they document the invariants.
- Each reducer action gets its own `describe` block; each invariant (immutability, idempotency, no-op on illegal transitions) gets its own `it`.
- Terminal-state no-op tests catch a class of bugs that's invisible in normal usage — write them explicitly.
- React Testing Library is the partner for component-level tests, but those land on D15; today, lean into the testing pyramid's broad pure-logic base.

---
*Prerequisites: [03-code-quality-linting-in-ci-ruff-eslint.md](../day-07/03-code-quality-linting-in-ci-ruff-eslint.md), [06-unit-testing-patterns-for-input-validation.md](../day-08/06-unit-testing-patterns-for-input-validation.md), [01-react-state-management-patterns-usestate-vs-usereducer.md](01-react-state-management-patterns-usestate-vs-usereducer.md), [04-client-side-temporal-state-timers-and-autosave.md](04-client-side-temporal-state-timers-and-autosave.md), [07-graceful-network-failure-handling.md](07-graceful-network-failure-handling.md). Forward references: [06-end-to-end-ui-testing-patterns-playwright-happy-path.md](../day-15/06-end-to-end-ui-testing-patterns-playwright-happy-path.md).*
