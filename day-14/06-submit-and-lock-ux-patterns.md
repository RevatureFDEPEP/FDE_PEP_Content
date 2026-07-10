# Submit-And-Lock UX Patterns

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D12 made the backend authoritative about attempt locking: `SELECT FOR UPDATE` on the attempt row, a one-way `status: in_progress → submitted` transition, idempotency-keyed submission, `409 Conflict` if already submitted, `410 Gone` if the session expired. Today's job is the UX side: render a submit button that visibly transitions through its lifecycle, disable interaction during in-flight requests, prevent double-submit by impatient candidates, surface the locked state clearly and irreversibly, and translate the specific backend status codes (409, 410) into UI states the candidate can act on. This is where the server's correctness gets translated into user trust.

## The Submit Button State Machine

The submit button alone has five distinct visual states, each driven by the submit reducer from Topic 1.

| State | Trigger | Button label | Button disabled? | Other UI |
|---|---|---|---|---|
| `idle` | Initial | "Submit Test" | No | Normal |
| `submitting` | User clicked submit | "Submitting…" + spinner | Yes | All inputs and nav also disabled |
| `submitted` | Server returned 2xx | "Submitted" + check icon | Yes (permanently) | Full locked-state overlay (see below) |
| `error_recoverable` | Network failure (Topic 7) | "Try Again" | No | Inline error message |
| `error_terminal_409` | Server returned 409 | "Already Submitted" | Yes | Banner directing candidate to refresh |
| `error_terminal_410` | Server returned 410 | "Session Expired" | Yes | Banner explaining what happened |

The candidate must always be able to tell which of these states they're in. The button label is the primary signal; color and iconography reinforce.

## The Submit Reducer

Mirroring the answer reducer pattern from Topic 1:

```ts
// frontend/app/take/[testId]/submitReducer.ts
export type SubmitStatus =
  | { kind: "idle" }
  | { kind: "submitting" }
  | { kind: "submitted"; lockedAt: string; sessionId: string }
  | { kind: "error_recoverable"; message: string }
  | { kind: "error_terminal_409" }
  | { kind: "error_terminal_410" };

export type SubmitAction =
  | { type: "SUBMIT_START" }
  | { type: "SUBMIT_SUCCESS"; lockedAt: string; sessionId: string }
  | { type: "SUBMIT_FAILURE_RECOVERABLE"; message: string }
  | { type: "SUBMIT_FAILURE_409" }
  | { type: "SUBMIT_FAILURE_410" }
  | { type: "RESET_RECOVERABLE_ERROR" };

export const initialSubmitState: SubmitStatus = { kind: "idle" };

export function submitReducer(state: SubmitStatus, action: SubmitAction): SubmitStatus {
  switch (action.type) {
    case "SUBMIT_START":
      // Only valid from idle or error_recoverable. Ignore from terminal states.
      if (state.kind === "idle" || state.kind === "error_recoverable") {
        return { kind: "submitting" };
      }
      return state;

    case "SUBMIT_SUCCESS":
      return { kind: "submitted", lockedAt: action.lockedAt, sessionId: action.sessionId };

    case "SUBMIT_FAILURE_RECOVERABLE":
      return { kind: "error_recoverable", message: action.message };

    case "SUBMIT_FAILURE_409":
      return { kind: "error_terminal_409" };

    case "SUBMIT_FAILURE_410":
      return { kind: "error_terminal_410" };

    case "RESET_RECOVERABLE_ERROR":
      return state.kind === "error_recoverable" ? { kind: "idle" } : state;

    default: {
      const _exhaustive: never = action;
      return state;
    }
  }
}
```

The state machine has an explicit "this transition is illegal, ignore" rule for `SUBMIT_START`: once submitted, locked, or terminally failed, no amount of button-clicking should restart the flow. This is the *first* defense against double-submit, before we even talk about disabling the button.

## Classifying Errors From The API Response

The submit handler must inspect the response and dispatch the correct failure action. Don't lump all failures into "something went wrong."

```ts
// frontend/app/take/[testId]/classifySubmitError.ts
import type { SubmitAction } from "./submitReducer";

export function classifySubmitError(error: unknown): SubmitAction {
  if (error instanceof Response) {
    if (error.status === 409) return { type: "SUBMIT_FAILURE_409" };
    if (error.status === 410) return { type: "SUBMIT_FAILURE_410" };
  }
  if (isNetworkError(error)) {
    return { type: "SUBMIT_FAILURE_RECOVERABLE", message: "Network error. Your answers are saved. Try again." };
  }
  // Unknown — treat as recoverable so the candidate isn't permanently stuck.
  return { type: "SUBMIT_FAILURE_RECOVERABLE", message: "Something went wrong. Try again." };
}
```

Two structural decisions worth highlighting:

- **409 and 410 are terminal, not recoverable.** A 409 means the backend has already locked the attempt — clicking "Try Again" is pointless. Surface what's happened and let the candidate refresh into the results page.
- **Unknown errors default to recoverable.** Better to let the candidate retry into a deterministic failure than to lock them out on a transient bug.

The `Response` instance check assumes the API client throws the `fetch` `Response` on non-2xx; adapt to whatever the project's API client actually throws.

## The Submit Handler

```tsx
// In <TestRunner>
const [submitState, dispatchSubmit] = useReducer(submitReducer, initialSubmitState);
const inFlightRef = useRef(false);

const onSubmit = methods.handleSubmit(async (values) => {
  // SECOND defense against double-submit: a ref that survives the brief window
  // before React commits the dispatch.
  if (inFlightRef.current) return;
  if (submitState.kind !== "idle" && submitState.kind !== "error_recoverable") return;

  inFlightRef.current = true;
  dispatchSubmit({ type: "SUBMIT_START" });

  try {
    const result = await api.submitSession(session.session_id, {
      answers: values,
      idempotency_key: `${session.session_id}:submit:1`,
    });
    dispatchSubmit({ type: "SUBMIT_SUCCESS", lockedAt: result.locked_at, sessionId: session.session_id });
  } catch (e) {
    dispatchSubmit(classifySubmitError(e));
  } finally {
    inFlightRef.current = false;
  }
});
```

The `inFlightRef` is belt-and-suspenders: between `dispatchSubmit({ type: "SUBMIT_START" })` and React actually re-rendering with the button disabled, there's a microtask window in which another `onSubmit` invocation can fire. The ref closes that window.

The idempotency key for submit is `${sessionId}:submit:1` — fixed, not incremented, because submit is logically idempotent: D12 will accept the first submission and return the same locked state on every subsequent attempt with the same key.

## The Submit Button Component

```tsx
// frontend/app/take/[testId]/SubmitButton.tsx
"use client";

import { useFormContext } from "react-hook-form";
import type { SubmitStatus } from "./submitReducer";

type Props = { status: SubmitStatus; onSubmit: () => void };

export function SubmitButton({ status, onSubmit }: Props) {
  const { formState } = useFormContext();
  const hasFieldErrors = Object.keys(formState.errors).length > 0;

  const { label, disabled, variant } = (() => {
    switch (status.kind) {
      case "idle":
        return { label: "Submit Test", disabled: hasFieldErrors, variant: "primary" as const };
      case "submitting":
        return { label: "Submitting…", disabled: true, variant: "primary" as const };
      case "submitted":
        return { label: "Submitted", disabled: true, variant: "success" as const };
      case "error_recoverable":
        return { label: "Try Again", disabled: false, variant: "warning" as const };
      case "error_terminal_409":
        return { label: "Already Submitted", disabled: true, variant: "muted" as const };
      case "error_terminal_410":
        return { label: "Session Expired", disabled: true, variant: "muted" as const };
    }
  })();

  return (
    <button
      type="button"
      disabled={disabled}
      onClick={onSubmit}
      aria-busy={status.kind === "submitting"}
      className={buttonClassName(variant)}
    >
      {status.kind === "submitting" && <Spinner className="mr-2 inline" />}
      {label}
    </button>
  );
}
```

Three accessibility-and-correctness details:

- **`type="button"`.** Without it, the button defaults to `type="submit"` and triggers any ancestor `<form>` submit on Enter key from within an input — a famous source of accidental submits.
- **`aria-busy`.** Screen readers announce the in-flight state to assistive tech.
- **`disabled` is the only thing that stops re-firing.** Combined with the `inFlightRef` and the reducer's transition rules, that's three layers of double-submit defense. Belt, suspenders, and a backup belt.

## The Locked-State Overlay

Once `status.kind === "submitted"`, the page needs to make the locked state *unmistakable*. The candidate has finished; the UI must communicate finality.

```tsx
// frontend/app/take/[testId]/LockedOverlay.tsx
"use client";

import Link from "next/link";

type Props = { lockedAt: string; sessionId: string };

export function LockedOverlay({ lockedAt, sessionId }: Props) {
  const lockedAtDisplay = new Date(lockedAt).toLocaleString();
  return (
    <div className="rounded border border-green-300 bg-green-50 p-6">
      <h2 className="text-lg font-semibold text-green-900">Your test has been submitted.</h2>
      <p className="mt-2 text-sm text-green-800">
        Submitted at {lockedAtDisplay}. Your answers cannot be changed.
      </p>
      <Link
        href={`/results/${sessionId}`}
        className="mt-4 inline-block rounded bg-green-700 px-4 py-2 text-white"
      >
        View Results
      </Link>
    </div>
  );
}
```

The whole `<TestRunner>` body switches to this overlay when locked — the questions and answers themselves can stay visible underneath, in disabled form, but the call-to-action shifts entirely. The candidate sees one button now: "View Results."

```tsx
// In <TestRunner>
return submitState.kind === "submitted" ? (
  <>
    <DisabledAnswerReview answers={methods.getValues()} questions={session.questions} />
    <LockedOverlay lockedAt={submitState.lockedAt} sessionId={submitState.sessionId} />
  </>
) : (
  <NormalTestRunnerLayout {...} />
);
```

## Handling 409 (Already Submitted)

A `409 Conflict` happens when a different tab, a stale retry, or a backend race already locked the attempt. The candidate's perspective: they pressed submit, and the system says "already done." The right UX is *not* to throw an error toast — it's to acknowledge that yes, the submission exists, here's where to see results.

```tsx
// In <TestRunner>, when status.kind === "error_terminal_409"
<div role="alert" className="rounded border border-amber-300 bg-amber-50 p-4">
  <p className="font-semibold">This attempt is already submitted.</p>
  <p className="mt-1 text-sm">It looks like you submitted from another tab or window.</p>
  <Link href={`/results/${session.session_id}`} className="mt-2 inline-block underline">
    View your results →
  </Link>
</div>
```

D12's response on 409 should ideally include the submitted timestamp and session id so we can render the link without a second round trip. If the API doesn't yet, file that as a follow-up.

## Handling 410 (Session Expired)

A `410 Gone` means the candidate ran out the clock — either the timer hit zero before submit, or they sat on a stale tab past expiry. The candidate has lost the ability to submit.

```tsx
// status.kind === "error_terminal_410"
<div role="alert" className="rounded border border-red-300 bg-red-50 p-4">
  <p className="font-semibold">This session has expired.</p>
  <p className="mt-1 text-sm">
    The submission window closed before your test could be submitted. Your answers up to the timer
    expiry have been auto-saved and counted.
  </p>
  <Link href={`/results/${session.session_id}`} className="mt-2 inline-block underline">
    View your results →
  </Link>
</div>
```

D12's expiry semantics auto-finalize the attempt server-side when the timer hits zero, so "your saved answers have been counted" is *true* — that's the value of the autosave from Topic 4 paired with the server-authoritative timer logic. Without autosave, this UX would be a lie.

## Timer-Triggered Auto-Submit

The timer hook from Topic 4 fires `onExpire` once when remaining hits zero. The handler dispatches a submit action just like the manual button does — same reducer, same flow, same protections against double-submit. The only difference is the UX banner before the submit fires:

```tsx
// In <TestRunner>
const onExpire = useCallback(() => {
  // Surface a brief "time's up, submitting now" toast.
  showToast("Time's up. Submitting your answers now.");
  onSubmit();
}, [onSubmit]);
```

The `onSubmit` from earlier already guards against double-fires, so even if the candidate clicked Submit at the same moment, only one POST goes out.

## Common Mistakes

- **No `inFlightRef`.** Relying solely on the `disabled` attribute leaves a microtask-width window for double-submit. Belt and suspenders.
- **`type="submit"` on the button** inside a form. Pressing Enter from any input fires the submit handler. Use `type="button"` and call `onSubmit` explicitly.
- **Treating 409 like a recoverable error.** Showing "Try again" on 409 makes the candidate click into a permanent failure loop. 409 is terminal; redirect to results.
- **Showing "Submitted!" before the server response arrives.** Topic 5 — never optimistically claim irreversible state.
- **Letting the locked overlay coexist with active inputs.** Once locked, every input must be disabled and unfocusable. Use a `<fieldset disabled>` wrapper if it's easier than per-input prop threading.
- **No accessibility affordances.** `aria-busy` during submitting, `role="alert"` on error banners, focus management onto the success/error banner after state change. These aren't optional for a real product.

## Key Takeaways
- The submit lifecycle is a six-state machine: `idle | submitting | submitted | error_recoverable | error_terminal_409 | error_terminal_410`, with explicit transitions and terminal absorbing states.
- Double-submit defense is layered: reducer rejects illegal transitions, `inFlightRef` covers the microtask window, and the disabled button prevents UI re-fires.
- 409 (already submitted) and 410 (session expired) are *terminal* states that route the candidate to results — they're not "try again" errors.
- Once locked, the entire page UI shifts: questions become read-only, the action surface collapses to a single "View Results" link, and the locked timestamp is displayed.
- Timer-triggered auto-submit uses the same submit flow, with the same double-submit protections, plus a toast announcing the auto-submit.

---
*Prerequisites: [05-database-transactions-and-pessimistic-locking.md](../day-12/05-database-transactions-and-pessimistic-locking.md), [06-state-finalization-and-immutability-patterns.md](../day-12/06-state-finalization-and-immutability-patterns.md), [03-idempotency-for-retried-mutations.md](../day-12/03-idempotency-for-retried-mutations.md), [01-react-state-management-patterns-usestate-vs-usereducer.md](01-react-state-management-patterns-usestate-vs-usereducer.md), [05-optimistic-updates-vs-server-confirmation.md](05-optimistic-updates-vs-server-confirmation.md). Forward references: [07-graceful-network-failure-handling.md](07-graceful-network-failure-handling.md), day-17-results-frontend.*
