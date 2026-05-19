# State-Coverage UI (Empty, Loading, Error)

> *Day 19: Trainer Dashboard Slice (Frontend) + Polish — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Most frontend bugs hiding in PEP right now are not "the happy path doesn't work" — D9, D14, D17 already exercised the happy path. The bugs are in the three states the happy path *isn't*: **empty** (the API returned an array with zero items), **loading** (data is in flight, what shows in the meantime?), and **error** (the API returned 500, the network died, the JWT expired). A senior frontend developer covers all three states explicitly on every page; a junior one assumes happy and discovers the gaps in code review. The polish day is when the cohort goes through every page and asks: what does this look like with zero data? what does it look like during the first 200ms? what does it look like when the API fails? This topic codifies the patterns — distinct components for each state, helpful copy (not "no data"), error states with a retry CTA, Suspense for loading — and gives a checklist to run before the capstone demo.

## Objective

Ensure every page covers empty, loading, and error states with distinct, well-designed UI.

## The Four States, Not Three

The typical framing is empty/loading/error, but in practice there are four:

1. **Loading** — request in flight, no data yet.
2. **Empty** — request succeeded, returned a valid response with no items.
3. **Error** — request failed (network, 500, 401, etc.).
4. **Success** — data present and renderable (the happy path).

A page that handles 1, 2, 3, and 4 explicitly is "state-complete." A page that only handles 4 falls apart in the other three.

## Why Empty Is Not An Error

The most common mistake: treating empty as "something went wrong." It isn't — a trainer with no filter matches just has no matches. A candidate who hasn't taken a test has no results. The right empty state is a friendly explanation and a path forward, not an error message.

Compare:

- Bad: `"No data."`
- Bad: `"Error: data array is empty."`
- Good: `"No attempts match the current filters. Try clearing filters or expanding the date range."`
- Good: `"You haven't taken any tests yet. Your trainer will assign one to you."`

The empty-state copy should *help the user* — tell them what to do next. This is one of the cheapest, highest-impact polish wins.

## The Pattern: Distinct Components

Don't conflate the states with conditionals in one component. Make them distinct.

```tsx
// frontend/components/states/EmptyState.tsx
export function EmptyState({
  title,
  message,
  action,
}: {
  title: string;
  message: string;
  action?: { label: string; onClick: () => void };
}) {
  return (
    <div
      role="status"
      className="rounded-lg border-2 border-dashed border-slate-300 p-8 text-center"
    >
      <h3 className="text-base font-semibold text-slate-700">{title}</h3>
      <p className="mt-1 text-sm text-slate-500">{message}</p>
      {action && (
        <button
          onClick={action.onClick}
          className="mt-4 rounded bg-slate-800 px-3 py-1 text-sm text-white hover:bg-slate-900"
        >
          {action.label}
        </button>
      )}
    </div>
  );
}
```

```tsx
// frontend/components/states/ErrorState.tsx
export function ErrorState({
  message,
  onRetry,
}: {
  message: string;
  onRetry?: () => void;
}) {
  return (
    <div
      role="alert"
      className="rounded-lg border border-red-300 bg-red-50 p-6 text-center"
    >
      <h3 className="text-base font-semibold text-red-800">Something went wrong</h3>
      <p className="mt-1 text-sm text-red-700">{message}</p>
      {onRetry && (
        <button
          onClick={onRetry}
          className="mt-4 rounded border border-red-300 bg-white px-3 py-1 text-sm text-red-800 hover:bg-red-100"
        >
          Try again
        </button>
      )}
    </div>
  );
}
```

```tsx
// frontend/components/states/LoadingState.tsx
export function LoadingSkeleton({ rows = 5 }: { rows?: number }) {
  return (
    <div className="space-y-2" role="status" aria-label="Loading">
      {Array.from({ length: rows }).map((_, i) => (
        <div key={i} className="h-12 animate-pulse rounded bg-slate-100" />
      ))}
    </div>
  );
}
```

Three components, three states, each independently styleable, each independently testable. Pages compose them rather than re-implementing.

## Loading Via Suspense

In the App Router, the loading state for server components is best handled with **Suspense boundaries** and a `loading.tsx` adjacent to the page. Recall from D17:

```tsx
// frontend/app/admin/reports/loading.tsx
import { LoadingSkeleton } from "@/components/states/LoadingState";

export default function Loading() {
  return (
    <main className="mx-auto max-w-6xl p-6">
      <LoadingSkeleton rows={6} />
    </main>
  );
}
```

Next.js automatically renders this while the server component's data is fetching. No manual `isLoading` state, no `useEffect` with a spinner. The cohort already saw this pattern on D17; today they extend it to `/admin/reports` and any other page that doesn't have it.

For nested in-page loading (e.g., a filter change re-fetching part of the page), wrap the relevant subtree in `<Suspense fallback={<LoadingSkeleton />}>`. The Suspense boundary scopes the fallback.

## Error Via `error.tsx`

The App Router's parallel to `loading.tsx` is `error.tsx`:

```tsx
// frontend/app/admin/reports/error.tsx
"use client";

import { ErrorState } from "@/components/states/ErrorState";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <main className="mx-auto max-w-6xl p-6">
      <ErrorState
        message="We couldn't load the dashboard. Please try again."
        onRetry={reset}
      />
    </main>
  );
}
```

`error.tsx` *must* be a client component (the App Router requires it). The `reset` function it receives re-renders the boundary's children — that's the retry mechanic. The `error.digest` is the hash Next.js attaches; log it for debugging but don't surface the raw `error.message` to the user (it might leak internal details).

The cohort built error boundaries on D17. Today the discipline is *every page has one*, not just the one we focused on.

## Empty State In The Happy Path Component

`error.tsx` and `loading.tsx` are framework-level. Empty is data-level: the fetch succeeded and the array has length 0. Handle it in the component that renders the data:

```tsx
// frontend/app/admin/reports/AggregateChart.tsx (excerpt)
export function AggregateChart({ data }: { data: TestRow[] }) {
  if (data.length === 0) {
    return (
      <ChartCard title="Correct vs Incorrect by Test">
        <EmptyState
          title="No attempts in this range"
          message="No candidate attempts match the current filters. Try expanding the date range or clearing the test filter."
        />
      </ChartCard>
    );
  }
  return <ChartCard>{/* ...chart... */}</ChartCard>;
}
```

The empty state lives *inside* the same chart card as the chart, preserving layout consistency — the dashboard doesn't visually rearrange depending on whether there's data.

## The Polish-Day Checklist

The cohort goes through every page and answers four questions:

| Page | Loading | Empty | Error | Success |
|---|---|---|---|---|
| `/` (home) | n/a (static) | n/a | n/a | OK |
| `/login` | spinner during submit | n/a | invalid creds copy | redirect |
| `/results` (list) | `loading.tsx` skeleton | "no attempts yet" | `error.tsx` | OK |
| `/results/[sessionId]` | `loading.tsx` | n/a (always populated) | `error.tsx` | OK |
| `/take/[testId]` | `loading.tsx` | n/a | `error.tsx` | OK |
| `/admin/reports` | `loading.tsx` | empty state in chart + table | `error.tsx` | OK |

Tick each cell. If a cell is missing, write the state component before moving on. This is the deliverable.

## Distinguishing Error Types

A single "Something went wrong" message works for most errors but can be sharpened:

- **401 (token expired):** "Your session expired. Please log in again." with a button to `/login`.
- **403 (forbidden):** "You don't have access to this resource." (Likely Topic 1 already redirected to `/403`, but defense in depth.)
- **404 (not found):** "We couldn't find that report." Specific page if the resource is identified by URL.
- **5xx (server error):** "We're having trouble loading this. Please try again in a moment." Plus retry.
- **Network error:** "Check your connection and try again." Plus retry.

In PEP, the fetch wrapper from D13 throws typed errors; the `error.tsx` can branch on the error type and render the right message. Don't go overboard — three or four cases is plenty.

## The "Retry" Affordance

Every error state needs a retry CTA when retry is meaningful. For server errors and network errors, yes. For 401 and 403, no — a retry won't fix it; the action is "log in" or "go back." The cohort should distinguish:

| Error | Retry meaningful? |
|---|---|
| 401 | No (log in) |
| 403 | No (go back) |
| 404 | Sometimes (typo in URL) |
| 5xx | Yes |
| Network | Yes |

A retry button on a 403 page misleads the user into thinking they can try again. Don't ship it.

## Loading State Granularity

Two reasonable choices for "the filter changed, the chart is reloading":

1. **Full-page Suspense fallback.** The whole dashboard area shows the skeleton until the new data is ready. Honest about loading but jarring — the page "blinks."
2. **Per-chart skeleton.** Each chart shows its own skeleton while the new data arrives, but the surrounding chrome (header, filter bar, totals) stays put. Less jarring; harder to wire because each chart needs its own Suspense.

PEP picks option 2 for the dashboard because the chrome is meaningful (the filter bar shows what's selected) and shouldn't blink. For the initial page load, option 1 is fine — the chrome isn't there yet.

## Accessibility For States

- **Loading:** `role="status"` + `aria-label="Loading"` so screen readers announce it. Don't show a spinner alone with no text.
- **Empty:** `role="status"` so it's announced when it appears. The empty-state heading should describe the situation.
- **Error:** `role="alert"` for immediate announcement. The error message should be specific (not "error") and actionable.

These are tiny additions on top of the existing components and add real value for keyboard / screen-reader users.

## Testing State Coverage

Manual testing is fine for a polish day — the cohort manually triggers each state:

- **Empty:** Set a filter that matches nothing (`from=2099-01-01`). Verify empty state appears.
- **Loading:** Throttle the network in devtools to "Slow 3G" and reload. Verify skeleton appears.
- **Error:** Stop the backend (docker-compose stop), reload the page. Verify error state appears with retry.
- **Success:** Normal flow. Verify happy-path UI.

One pass through the checklist per page, signed off by the trainer or a peer. Box checked.

## Anti-Patterns

- **"No data" as the empty-state copy.** Unhelpful, makes the user think the app is broken. Always explain what's missing and what to do.
- **A spinner that lasts forever.** If the fetch hangs, the user is stuck. Suspense boundaries have implicit timeouts only if you wire them; consider a 30s client-side timeout that surfaces the error state.
- **Surfacing raw `error.message` to the user.** Might leak internal paths, stack traces, or API keys. Generic message to the user, full error to the logs.
- **Empty state inside the error state component.** They're different states; don't merge them. An empty fetch is a *success*; an error is a *failure*.
- **Retry buttons on every error.** 401 and 403 don't benefit from retry. A misleading retry is worse than no retry.
- **Skeleton that doesn't match the real layout.** If the skeleton is two boxes and the real content is a table, the layout shifts when content arrives. Skeleton should approximate the real shape.
- **Loading state via `useEffect` + `isLoading` in a server-component page.** Use `loading.tsx` and Suspense; the React-side `isLoading` pattern is for legacy client-only pages.

## Connecting Back To D17

D17 introduced Suspense boundaries and error boundaries on the results page. Today's work is the *generalization* — every page gets the same treatment. The checklist is the deliverable. By the end of D19, no page in the app should have a missing state.

This is the polish-day work that separates "demo-ready" from "actually shippable." A demo can paper over an empty state with seeded data; a real user *will* hit the empty case, and the unhelpful "no data" message will look amateur. Investing the hour now is what makes the capstone demo (D20) feel finished.

## Key Takeaways

- Four states per page: loading, empty, error, success. A senior frontend dev handles all four explicitly.
- Empty is not an error; help the user understand what's missing and what to do next.
- Loading via Suspense + `loading.tsx`. Error via error boundary + `error.tsx`. Empty handled in the component that renders the data.
- Retry CTAs only on errors where retry is meaningful (5xx, network) — not 401, 403, or the empty state.
- Run the polish-day checklist on every page; box every cell before declaring done.
- Accessibility roles (`status`, `alert`) on each state component so screen readers announce them.

---
*Prerequisites: day-17-loading-states-with-suspense-boundaries, day-17-error-boundary-patterns, day-17-accessibility-fundamentals.*
