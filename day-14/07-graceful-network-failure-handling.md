# Graceful Network Failure Handling

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D12's edge-case catalog established the backend's posture: idempotency keys make retries safe, transactions make conflicts coherent, and explicit status codes (409, 410, 422) tell the client *why* something failed. D14 closes that loop on the frontend. A test-taking page that just lets a `fetch` rejection bubble up as an unhandled promise rejection — or worse, silently swallows it — is broken. The candidate's network is going to flake. The CDN is going to hiccup. The Wi-Fi is going to drop for 30 seconds. The page must keep the candidate's work safe through all of it, retry what's safely retryable, surface what isn't, and never make the candidate guess whether their submission succeeded.

## Three Categories Of Failure

Every network failure on this page falls into one of three buckets, each with a different handling strategy.

| Category | Examples | Strategy |
|---|---|---|
| **Transient transport** | DNS resolution failure, TCP timeout, fetch aborted, 502/503/504 | Auto-retry with exponential backoff (for idempotent calls); surface only after persistent failure |
| **Semantic (server-explicit)** | 409 Conflict, 410 Gone, 422 Validation | Don't retry; map directly to UX state per Topic 6 |
| **Connectivity** | Browser offline, network unreachable | Pause new requests, surface persistent banner, resume when back online |

Conflating these is the most common failure mode. A retry on a 422 is pointless (the request was malformed; retrying doesn't fix that). A surface-immediately on a transient 503 is noisy (it was probably going to succeed on the second try). Build the classification *first*, then the handling logic flows from it.

```ts
// frontend/app/take/[testId]/classifyNetworkError.ts
export type FailureCategory =
  | { kind: "transient"; retryable: true; httpStatus?: number }
  | { kind: "semantic"; retryable: false; httpStatus: number }
  | { kind: "connectivity"; retryable: true };

export function classifyNetworkError(error: unknown): FailureCategory {
  if (!navigator.onLine) {
    return { kind: "connectivity", retryable: true };
  }
  if (error instanceof Response) {
    const status = error.status;
    if (status >= 500 && status < 600) return { kind: "transient", retryable: true, httpStatus: status };
    if (status === 408 || status === 429) return { kind: "transient", retryable: true, httpStatus: status };
    return { kind: "semantic", retryable: false, httpStatus: status };
  }
  if (error instanceof TypeError) {
    // fetch network error (DNS, TCP, aborted) is a TypeError in the spec.
    return { kind: "transient", retryable: true };
  }
  // Unknown — treat as transient to err on the side of retrying.
  return { kind: "transient", retryable: true };
}
```

## Retry Policy For Idempotent Calls

D12's autosave endpoint and submit endpoint are both idempotent *if* given the same idempotency key. That's a precondition: retrying with a different key is not a retry, it's a new mutation. The retry helper must enforce key reuse.

A practical retry policy:

- **Max 3 retries** after the initial attempt (4 total attempts).
- **Exponential backoff** with jitter: 250ms, 750ms, 2250ms (×3, plus ±20% jitter).
- **Only retry transient + connectivity** failures.
- **Use the same idempotency key for every attempt.**

```ts
// frontend/app/take/[testId]/retry.ts
import { classifyNetworkError } from "./classifyNetworkError";

type RetryOptions = {
  maxRetries?: number;
  baseDelayMs?: number;
  signal?: AbortSignal;
};

export async function withRetry<T>(
  operation: () => Promise<T>,
  { maxRetries = 3, baseDelayMs = 250, signal }: RetryOptions = {}
): Promise<T> {
  let lastError: unknown;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    if (signal?.aborted) throw new DOMException("Aborted", "AbortError");
    try {
      return await operation();
    } catch (e) {
      lastError = e;
      const category = classifyNetworkError(e);
      if (!category.retryable || attempt === maxRetries) {
        throw e;
      }
      const delay = baseDelayMs * 3 ** attempt;
      const jitter = delay * (Math.random() * 0.4 - 0.2); // ±20%
      await sleep(delay + jitter, signal);
    }
  }
  throw lastError;
}

function sleep(ms: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    const id = setTimeout(resolve, ms);
    signal?.addEventListener("abort", () => {
      clearTimeout(id);
      reject(new DOMException("Aborted", "AbortError"));
    });
  });
}
```

The `signal` parameter matters because the candidate might navigate away mid-retry. Cancelling the retry loop avoids the classic "ghost request fires 10 seconds after the user left the page" bug.

## Wiring Retry Into Autosave

The autosave watcher from Topic 4 wraps its API call in `withRetry`. Because the idempotency key is `(session_id, question_id, attempt_n)` — and `attempt_n` doesn't increment on retries (only on logical changes) — every retry hits the backend with the same key, which D12 dedupes.

```ts
// Inside the autosave flush
try {
  await withRetry(() =>
    api.submitAnswer(sessionId, {
      question_id: questionId,
      selected: value,
      idempotency_key: `${sessionId}:${questionId}:${attemptN}`,
    })
  );
  dispatch({ type: "AUTOSAVE_SUCCESS", questionId });
} catch (e) {
  dispatch({ type: "AUTOSAVE_ERROR", questionId });
  showToast("Couldn't save your answer to question " + (index + 1) + ". We'll try again automatically.");
}
```

After 4 attempts failing, the autosave-error state is set and a toast surfaces. The candidate's selection in the UI is preserved (Topic 5 optimistic posture) — only the save status is degraded. The autosave watcher will try again on the next debounced change.

## Wiring Retry Into Submit

Submit is also retryable on transient failures — same logic, same key (`${sessionId}:submit:1`), but the UI shows the spinner for the entire retry duration. Don't surface intermediate retries to the candidate; only the final outcome.

```ts
const onSubmit = methods.handleSubmit(async (values) => {
  if (inFlightRef.current) return;
  inFlightRef.current = true;
  dispatchSubmit({ type: "SUBMIT_START" });
  try {
    const result = await withRetry(() =>
      api.submitSession(session.session_id, {
        answers: values,
        idempotency_key: `${session.session_id}:submit:1`,
      })
    );
    dispatchSubmit({ type: "SUBMIT_SUCCESS", lockedAt: result.locked_at, sessionId: session.session_id });
  } catch (e) {
    dispatchSubmit(classifySubmitError(e));
  } finally {
    inFlightRef.current = false;
  }
});
```

The candidate sees: button changes to "Submitting…" → after up to ~5 seconds total wait (250 + 750 + 2250 + final attempt), either "Submitted" or a specific error message. They don't see "retry 1 of 3" — that's plumbing noise the candidate doesn't need.

## Offline Detection

`navigator.onLine` is the browser's offline signal. It's not perfectly accurate — it tells you whether the OS thinks the network adapter is connected, not whether there's actually internet — but it's the right primitive to wire into the UX.

```tsx
// frontend/app/take/[testId]/useOnlineStatus.ts
"use client";

import { useEffect, useState } from "react";

export function useOnlineStatus(): boolean {
  const [online, setOnline] = useState(() =>
    typeof navigator !== "undefined" ? navigator.onLine : true
  );

  useEffect(() => {
    const onOnline = () => setOnline(true);
    const onOffline = () => setOnline(false);
    window.addEventListener("online", onOnline);
    window.addEventListener("offline", onOffline);
    return () => {
      window.removeEventListener("online", onOnline);
      window.removeEventListener("offline", onOffline);
    };
  }, []);

  return online;
}
```

The `<TestRunner>` uses this to render a persistent banner across the top while offline:

```tsx
const online = useOnlineStatus();
return (
  <div>
    {!online && (
      <div role="status" className="bg-amber-100 px-4 py-2 text-amber-900">
        You're offline. Your answers are being saved locally and will sync when you reconnect.
      </div>
    )}
    {/* ... rest of test runner ... */}
  </div>
);
```

The phrasing "saved locally" is honest only if we *are* persisting to `localStorage` as an offline fallback. If we're not (the D14 deliverable doesn't strictly require it), say "we'll retry when you reconnect" instead. Don't lie to the candidate about durability.

## Toast UX Pattern

A toast is a transient notification that auto-dismisses. For this page, toasts are used for *informational* network signals — they shouldn't carry critical state (that's what the save badges and submit button states are for). Use any toast library (sonner, react-hot-toast) or roll a minimal one.

Guidelines for what gets a toast on this page:

| Event | Toast? | Severity |
|---|---|---|
| Autosave succeeds | No (the badge is enough) | — |
| Autosave fails after retries | Yes | Warning |
| Coming back online | Yes | Info, brief |
| Going offline | No (the persistent banner covers it) | — |
| Submit in progress | No (the button shows it) | — |
| Timer expired, auto-submitting | Yes | Info |
| Session expired (410) | No (the persistent banner covers it; toast would be redundant) | — |

The discipline: a toast is a *one-shot announcement*, not a state display. Anything the candidate needs to keep seeing — submission status, locked state, save status per question — belongs in persistent UI, not a toast.

## The "Your Answer Didn't Save" Recovery Flow

When autosave permanently fails for a question, the candidate's selection is still visible (optimistic), but the red save badge is on. The recovery affordance:

```tsx
// In <QuestionView> when autosave status is "error"
{autosaveStatus === "error" && (
  <div className="mt-2 flex items-center gap-2 rounded border border-red-300 bg-red-50 px-3 py-2 text-sm text-red-800">
    <span>Couldn't save this answer.</span>
    <button
      type="button"
      onClick={() => forceFlush(question.question_id)}
      className="underline"
    >
      Retry
    </button>
  </div>
)}
```

`forceFlush` invokes the same autosave call (with the same idempotency key, so it's safe), bypassing the debounce timer. This gives the candidate explicit control when the automatic retries gave up.

## Why Not Roll Everything Into React Query

React Query gives you retries, caching, and offline awareness for free. It's a reasonable choice for a production app. For PEP, building this layer by hand has pedagogical value:

- Candidates *see* the retry policy: it's a function with explicit math, not a config option.
- Candidates *see* the classifier: a switch on `Response.status` and `instanceof TypeError`, not a black-box `retry: (failureCount, error) => …`.
- Candidates *see* the AbortSignal plumbing: needed for cleanup, often glossed over by library users.

When D15 integrates the slice and D17 builds the results frontend, React Query is a fine production-grade swap-in. Today the hand-rolled version makes the mechanism legible.

## Common Mistakes

- **Retrying non-idempotent calls.** Without an idempotency key, every retry risks creating a duplicate. The retry helper assumes the caller has set a key — document this prerequisite.
- **Retrying 4xx errors (except 408/429).** A 400/422 won't succeed on retry; you're just delaying the user's awareness of a problem.
- **Not aborting in-flight retries on unmount/navigation.** Ghost requests can fire long after the user has moved on, causing confusing state writes.
- **Putting critical state in toasts.** Toasts dismiss. "Your test failed to submit" disappearing after 4 seconds is a UX disaster.
- **Treating `navigator.onLine === false` as authoritative "no network."** It only tells you the OS thinks the adapter is unplugged. Captive portals, VPN issues, server outages — none of these show as offline. The retry mechanism is the real defense; the offline banner is a hint.
- **No upper bound on retry delay.** Without a cap or a fixed max, exponential backoff drifts into "user has waited 30 seconds" territory. Cap the total wait at something sensible (5–10 seconds for autosave; longer for less time-sensitive operations).

## Key Takeaways
- Classify failures into three buckets — transient (retry), semantic (route to specific UX state per Topic 6), connectivity (pause + persistent banner) — before choosing a handling strategy.
- Retry policy: max 3 retries with exponential backoff and ±20% jitter, only for idempotent calls, only for transient/connectivity categories.
- Idempotency keys from D12 make retries safe: same key across retries, new key only on logical change of intent.
- Offline detection (`navigator.onLine` + online/offline events) drives a persistent banner; the retry mechanism handles the actual reconnection gracefully.
- Toasts are for announcements, not state — persistent UI carries everything the candidate needs to keep seeing.
- Cancellable retries (AbortSignal) prevent ghost requests after navigation.

---
*Prerequisites: [07-edge-case-handling-for-distributed-clients.md](../day-12/07-edge-case-handling-for-distributed-clients.md), [03-idempotency-for-retried-mutations.md](../day-12/03-idempotency-for-retried-mutations.md), [05-optimistic-updates-vs-server-confirmation.md](05-optimistic-updates-vs-server-confirmation.md), [06-submit-and-lock-ux-patterns.md](06-submit-and-lock-ux-patterns.md). Forward references: [08-component-unit-testing-patterns-vitest.md](08-component-unit-testing-patterns-vitest.md), [01-vertical-slice-integration-in-a-local-compose-environment.md](../day-15/01-vertical-slice-integration-in-a-local-compose-environment.md).*
