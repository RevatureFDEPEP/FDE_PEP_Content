# Client-Side Temporal State — Timers Anchored To Server Time, Autosave-On-Change

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D11's session endpoint returned `server_now` and `expires_at` precisely because the server is the authority on time and the client is not to be trusted. D14's UI side of that contract is two things working together: a countdown timer that *displays* time remaining without secretly trusting the client clock, and an autosave mechanism that flushes answer changes to the backend on a debounced timer so a refresh or tab close doesn't cost the candidate their progress. Both are temporal state, both have to handle the awkward edges of real browsers — tab blur throttling, system clock skew, network failure, refresh — and both interact with D12's idempotency-key contract so retries are safe.

## The Server-Anchored Timer: The Naive Approach Is Wrong

The instinct from anyone who's written a countdown is:

```tsx
// WRONG
const [secondsLeft, setSecondsLeft] = useState(durationSeconds);
useEffect(() => {
  const id = setInterval(() => setSecondsLeft((s) => s - 1), 1000);
  return () => clearInterval(id);
}, []);
```

This is wrong for the test-taking page in three ways:

1. **`setInterval` drifts.** Browsers don't fire intervals on precise 1000ms boundaries — especially on background tabs, where Chrome throttles to once per minute. The candidate switches tabs for 5 minutes, comes back, and the timer says they have 5 minutes more than they actually do.
2. **It trusts the client clock.** Pause the system, change the system clock, and the countdown obeys. D11 was explicit: the server is the authority.
3. **It re-renders the entire component once per second.** Most of the page doesn't care; only the timer display does. Re-rendering `<TestRunner>` 60 times a minute is wasteful and triggers a cascade of memoization considerations elsewhere.

## The Right Approach: Compute Remaining From A Server-Anchored Reference

At mount, compute the offset between the server's view of "now" and the client's:

```ts
// offset = clientNow - serverNow
// In other words: how much "ahead" the client clock is.
// To map a client timestamp back to "what the server would call this moment":
//   serverEquivalent(clientNow) = clientNow - offset
```

Then on each tick, recompute the remaining time as `expires_at - serverEquivalent(Date.now())`. The display refreshes once per second on a `setInterval`, but the *value* it shows is recomputed each tick from absolute timestamps, so missed ticks don't accumulate error.

```ts
// frontend/app/take/[testId]/useServerAnchoredTimer.ts
"use client";

import { useEffect, useRef, useState } from "react";

type Options = {
  /** Server's view of "now" at session creation (ISO 8601 string from D11). */
  serverNow: string;
  /** Server-computed expiry (ISO 8601). */
  expiresAt: string;
  /** Called once when remaining hits zero. Idempotent — the hook also ensures this. */
  onExpire?: () => void;
};

type State = { remainingMs: number; expired: boolean };

export function useServerAnchoredTimer({ serverNow, expiresAt, onExpire }: Options): State {
  // Capture the offset once, at mount. Don't recompute — that would defeat the point.
  const offsetMsRef = useRef<number>(Date.now() - new Date(serverNow).getTime());
  const expiresAtMsRef = useRef<number>(new Date(expiresAt).getTime());
  const firedExpireRef = useRef(false);

  const compute = (): State => {
    const serverEquivalentNow = Date.now() - offsetMsRef.current;
    const remainingMs = Math.max(0, expiresAtMsRef.current - serverEquivalentNow);
    return { remainingMs, expired: remainingMs === 0 };
  };

  const [state, setState] = useState<State>(compute);

  useEffect(() => {
    const tick = () => {
      const next = compute();
      setState(next);
      if (next.expired && !firedExpireRef.current) {
        firedExpireRef.current = true;
        onExpire?.();
      }
    };

    const id = setInterval(tick, 1000);
    // Also recompute when the tab regains focus — see "Reconciliation On Refocus" below.
    const onVisibility = () => {
      if (document.visibilityState === "visible") tick();
    };
    document.addEventListener("visibilitychange", onVisibility);
    window.addEventListener("focus", tick);

    tick(); // immediate first tick so the display doesn't lag

    return () => {
      clearInterval(id);
      document.removeEventListener("visibilitychange", onVisibility);
      window.removeEventListener("focus", tick);
    };
  }, [onExpire]);

  return state;
}
```

Three things worth highlighting:

- **`offsetMsRef` captured in `useRef`, not state.** Recomputing the offset would let clock skew drift change the timer mid-session — defeating the anchor. Capture once, use forever.
- **`compute()` is pure and reads `Date.now()` fresh each call.** Missed ticks don't matter; the next tick reads the current clock and yields the correct remaining time.
- **`visibilitychange` and `focus` listeners.** When the tab was backgrounded, the interval likely didn't fire at the rate we asked for. Recomputing on refocus snaps the display to truth immediately.

## Hosting The Timer In The Reserved Slot From D13

D13 left a header slot for the timer. Today it's filled by a tiny client component that consumes the hook and renders the formatted string. Isolating the timer in its own component means the once-per-second re-render is *only* in that component, not in `<TestRunner>` or `<QuestionView>`.

```tsx
// frontend/app/take/[testId]/Timer.tsx
"use client";

import { useServerAnchoredTimer } from "./useServerAnchoredTimer";
import { formatRemaining } from "./formatRemaining";

type Props = { serverNow: string; expiresAt: string; onExpire: () => void };

export function Timer({ serverNow, expiresAt, onExpire }: Props) {
  const { remainingMs, expired } = useServerAnchoredTimer({ serverNow, expiresAt, onExpire });

  return (
    <div role="timer" aria-live="off" className={expired ? "text-red-700" : "text-slate-700"}>
      {expired ? "Time's up" : formatRemaining(remainingMs)}
    </div>
  );
}
```

`formatRemaining` is a pure helper — and exactly the kind of pure function that gets unit-tested in Topic 8.

```ts
// frontend/app/take/[testId]/formatRemaining.ts
/** "12:34" or "1:23:45" depending on duration. */
export function formatRemaining(ms: number): string {
  const totalSeconds = Math.floor(ms / 1000);
  const hours = Math.floor(totalSeconds / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = totalSeconds % 60;
  const mm = String(minutes).padStart(2, "0");
  const ss = String(seconds).padStart(2, "0");
  return hours > 0 ? `${hours}:${mm}:${ss}` : `${minutes}:${ss}`;
}
```

## Autosave: Debounced Per-Question Flush

Every time the candidate changes their selection for a question, we want to push the new answer to D12's `POST /sessions/{id}/answer` — but not *every keystroke*. A multi-select with five checkbox toggles in three seconds should produce *one* save, not five. The standard tool is a debounced effect: schedule a save, and if the candidate makes another change before it fires, reset the timer.

```ts
// frontend/app/take/[testId]/useDebouncedEffect.ts
"use client";

import { useEffect, useRef } from "react";

export function useDebouncedEffect(effect: () => void | Promise<void>, deps: unknown[], delayMs: number) {
  const cbRef = useRef(effect);
  cbRef.current = effect;

  useEffect(() => {
    const id = setTimeout(() => {
      void cbRef.current();
    }, delayMs);
    return () => clearTimeout(id);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [...deps, delayMs]);
}
```

400ms is a good default: long enough that rapid typing/clicking coalesces, short enough that the candidate's perception of "saved" matches reality.

## Wiring Autosave Into `<TestRunner>`

Each `QuestionView` mounts a watcher for *its own* question. When the value changes and the field is dirty, the autosave fires.

```tsx
// frontend/app/take/[testId]/AutosaveWatcher.tsx
"use client";

import { useFormContext } from "react-hook-form";
import { useDebouncedEffect } from "./useDebouncedEffect";
import { api } from "@/lib/api";
import { useReducer } from "react";
// answerReducer from Topic 1 is dispatched here too

type Props = {
  sessionId: string;
  questionId: string;
  attemptN: number;
  dispatch: (action: { type: "AUTOSAVE_START" | "AUTOSAVE_SUCCESS" | "AUTOSAVE_ERROR"; questionId: string }) => void;
};

export function AutosaveWatcher({ sessionId, questionId, attemptN, dispatch }: Props) {
  const { watch, formState } = useFormContext();
  const value = watch(questionId);
  const isDirty = !!formState.dirtyFields[questionId];

  useDebouncedEffect(async () => {
    if (!isDirty) return;
    dispatch({ type: "AUTOSAVE_START", questionId });
    try {
      await api.submitAnswer(sessionId, {
        question_id: questionId,
        selected: value,
        // Idempotency-key construction from rules: (session_id, question_id, attempt_n)
        idempotency_key: `${sessionId}:${questionId}:${attemptN}`,
      });
      dispatch({ type: "AUTOSAVE_SUCCESS", questionId });
    } catch {
      dispatch({ type: "AUTOSAVE_ERROR", questionId });
      // Topic 7 handles retries.
    }
  }, [value, isDirty], 400);

  return null; // pure side-effect component
}
```

A few specific decisions:

- **Idempotency-key from `(session_id, question_id, attempt_n)`.** D12 demands an idempotency key per mutation. The triple uniquely identifies "this candidate's Nth answer attempt for this question in this session." `attempt_n` increments each time the candidate *changes* their selection — not each retry. Retries of the same logical change must reuse the same key so the backend dedupes.
- **`AutosaveWatcher` is a render-less component.** Rendering null and doing work in `useEffect` is a clean pattern when you want to attach behavior to a tree position without leaking imperative APIs.
- **The reducer (Topic 1) is the source of truth for autosave status.** The watcher dispatches; the UI displays a small badge (`<AutosaveBadge>`) reading from `autosave.get(questionId)`.

## What `attempt_n` Actually Counts

A subtle point that trips intermediate developers up: `attempt_n` is **not** the retry count. It's the *logical* attempt — incremented each time the candidate's intent changes, so that the idempotency-key construction looks like this:

- Candidate selects option A. attempt_n = 1. Save with key `…:1`. Network fails. Retry: same key.
- Candidate changes mind, selects option B. attempt_n = 2. Save with key `…:2`. Backend sees a new key, processes it, overwrites the answer per D12 rules.

This means `attempt_n` lives in the reducer (Topic 1) keyed by question_id and increments on `SET_SELECTION`. Retries (Topic 7) reuse the current key. Logical changes mint a new one.

## Reconciliation On Refocus

The timer hook already handles refocus by recomputing remaining from absolute timestamps. Autosave needs its own refocus handling: when the tab regains visibility, force-flush any pending dirty answers in case the debounced timer was throttled to oblivion in the background.

```tsx
useEffect(() => {
  const onVisibility = () => {
    if (document.visibilityState === "visible") {
      // For each dirty question, dispatch an immediate save.
      forceFlushAllDirty();
    }
  };
  document.addEventListener("visibilitychange", onVisibility);
  return () => document.removeEventListener("visibilitychange", onVisibility);
}, [forceFlushAllDirty]);
```

The candidate's mental model is "I left the tab, I came back, my work is safe." Without this, a backgrounded `setTimeout` might mean the save was queued but never fired.

## Why Not Just `localStorage`?

A common shortcut: autosave to `localStorage` only, sync to backend on submit. This works until the candidate switches devices, the browser clears storage (Safari ITP, incognito), or — most damagingly — the trainer needs to see in-progress state for the live cohort tracking on D18's dashboard. Backend autosave is the right architecture; `localStorage` is at best a fallback (which we're not implementing today). Server-of-record discipline from D11 carries forward.

## Common Mistakes

- **Recomputing the clock offset on every tick.** This defeats the anchor. Capture once at mount with `useRef`.
- **Hosting the timer in `<TestRunner>` directly.** The component re-renders once per second, dragging everything with it. Isolate into a leaf `<Timer>` component.
- **Debounce delay too long** (>1 second). Candidate clicks Next, navigates away, save never fired, debounce timer cleared. Either flush on navigate or keep the delay short. We do both: 400ms + flush on visibility change.
- **Forgetting `attempt_n` semantics.** Bumping it on every retry instead of every logical change defeats idempotency — the backend sees each retry as a new mutation and the dedup is gone.
- **No refocus handling.** Background-tab throttling will eat your timer accuracy and your autosave timing. Always listen for `visibilitychange`.
- **Trusting `setInterval` to fire on schedule.** Always recompute display values from absolute timestamps, never from "tick count × 1000."

## Key Takeaways
- The timer reads `server_now` and `expires_at` from D11, computes a one-time `offset = clientNow - serverNow`, and renders `expires_at - (Date.now() - offset)` on each tick — drift-free even when the interval is throttled.
- A separate `<Timer>` leaf component absorbs the once-per-second re-render so `<TestRunner>` stays calm.
- Autosave is debounced ~400ms per question, fires only on dirty fields, and uses the D12 idempotency-key triple `(session_id, question_id, attempt_n)` so retries are safe.
- `attempt_n` increments on logical answer changes, not on network retries — that's what makes the idempotency key actually work.
- Refocus and visibility-change handlers reconcile both the timer display and force-flush pending autosaves, because background tabs eat scheduled work.

---
*Prerequisites: [08-server-authoritative-state.md](../day-11/08-server-authoritative-state.md), [03-idempotency-for-retried-mutations.md](../day-12/03-idempotency-for-retried-mutations.md), [02-form-state-collection-with-react-hook-form.md](02-form-state-collection-with-react-hook-form.md). Forward references: [05-optimistic-updates-vs-server-confirmation.md](05-optimistic-updates-vs-server-confirmation.md), [06-submit-and-lock-ux-patterns.md](06-submit-and-lock-ux-patterns.md), [07-graceful-network-failure-handling.md](07-graceful-network-failure-handling.md), [08-component-unit-testing-patterns-vitest.md](08-component-unit-testing-patterns-vitest.md).*
