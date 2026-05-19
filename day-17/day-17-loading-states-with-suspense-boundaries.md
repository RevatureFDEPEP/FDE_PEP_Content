# Loading States With Suspense Boundaries

> *Day 17: Results Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The `GET /reports/user/{userId}` endpoint built on D16 is fast, but "fast" isn't "instant" — a SQL aggregation plus a cross-service fetch to question-management for test names is realistically 80–300ms on a warm container, more on a cold one. In a server-component world, that latency lives in the time-to-first-byte: the page can't render until the fetch resolves. If you do nothing, the candidate clicks "View Results" and sees a frozen browser tab until the HTML arrives. Suspense fixes that. Topic 2 covers how to draw Suspense boundaries that turn a single slow fetch into a fast page chrome plus a contained skeleton, without writing any explicit loading-state code.

## The Old Way And Why We're Not Doing It

The pre-Suspense pattern in a single-page React app looks like this — and it is the pattern the inherited frontend used everywhere up through D9:

```tsx
"use client";
function ResultsPage() {
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  const [data, setData] = useState<ReportResponse | null>(null);

  useEffect(() => {
    fetch("/api/reports/...")
      .then(r => r.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <Spinner />;
  if (error) return <ErrorBox error={error} />;
  if (!data) return null;

  return <ActualContent data={data} />;
}
```

Every component that fetches reimplements `loading / error / data`. Every component carries a `useState` triplet. Every component is now a client component because `useEffect` is client-only. And nothing renders until the fetch completes — there's no useful page chrome on screen during the wait.

Server components plus Suspense replace all of that with:

```tsx
<Suspense fallback={<Skeleton />}>
  <AsyncChild />
</Suspense>
```

The async child *suspends* — literally throws a promise React catches at the boundary — and React renders the fallback. When the promise resolves, React swaps in the real content. No state, no effects, no client component.

## Where The Boundary Goes

A Suspense boundary is a contract: *everything below me may take time; render this fallback until they're ready.* The art is deciding where to put it.

Three reasonable positions for the results page:

1. **Around the whole `<main>`.** Page chrome is `<h1>Your Results</h1>`; everything below it suspends together. Simplest. The candidate sees the page title plus a single skeleton until the report arrives.
2. **Around `<ResultsBreakdown />` only.** The page chrome (title) renders immediately; the data-heavy panel suspends with its own skeleton. Slightly better perceived performance.
3. **Multiple boundaries, one per panel** — summary, table, and chart each in their own `<Suspense>`. Only useful if each panel has its *own* data fetch. For a single endpoint that returns everything together (D16's design), this just adds noise.

Day 17 uses option 2. The page title and header chrome render immediately so the user has visual confirmation the navigation worked; the data-driven content swaps in a skeleton until the fetch resolves.

```tsx
// frontend/app/results/[sessionId]/page.tsx
import { Suspense } from "react";
import { ResultsBreakdown } from "./ResultsBreakdown";
import { ResultsSkeleton } from "./ResultsSkeleton";

export default async function ResultsPage({
  params,
}: {
  params: Promise<{ sessionId: string }>;
}) {
  const { sessionId } = await params;

  return (
    <main className="mx-auto flex w-full max-w-5xl flex-col gap-6 px-4 py-6 md:px-6">
      <h1 className="text-2xl font-semibold tracking-tight">Your Results</h1>

      <Suspense fallback={<ResultsSkeleton />}>
        <ResultsBreakdown sessionId={sessionId} />
      </Suspense>
    </main>
  );
}
```

`ResultsBreakdown` is an `async` server component (Topic 1). The `await` inside it is what causes the suspend — React renders the fallback until the promise settles.

## The Skeleton Component

A loading skeleton is *not* a spinner. It's a low-information placeholder that occupies roughly the same screen real estate as the real content, so the page doesn't jump when the data arrives. Empirically, well-shaped skeletons reduce perceived latency more than any other UX optimization.

```tsx
// frontend/app/results/[sessionId]/ResultsSkeleton.tsx
import { Card, CardContent, CardHeader } from "@/components/ui/card";

export function ResultsSkeleton() {
  return (
    <div
      className="grid grid-cols-1 gap-6 md:grid-cols-3"
      role="status"
      aria-live="polite"
      aria-label="Loading your results"
    >
      <Card className="md:col-span-1">
        <CardHeader>
          <div className="h-5 w-20 animate-pulse rounded bg-muted" />
        </CardHeader>
        <CardContent>
          <div className="h-10 w-32 animate-pulse rounded bg-muted" />
          <div className="mt-2 h-4 w-40 animate-pulse rounded bg-muted" />
        </CardContent>
      </Card>

      <Card className="md:col-span-2">
        <CardHeader>
          <div className="h-5 w-48 animate-pulse rounded bg-muted" />
        </CardHeader>
        <CardContent>
          <div className="h-64 w-full animate-pulse rounded bg-muted" />
        </CardContent>
      </Card>

      <Card className="md:col-span-3">
        <CardHeader>
          <div className="h-5 w-24 animate-pulse rounded bg-muted" />
        </CardHeader>
        <CardContent>
          <div className="space-y-2">
            {Array.from({ length: 5 }).map((_, i) => (
              <div key={i} className="h-8 w-full animate-pulse rounded bg-muted" />
            ))}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

Why the specifics:

- **Same grid shape** (`grid-cols-1 md:grid-cols-3`, three cards) as the real `ResultsBreakdown`. Layout doesn't shift when the real content lands.
- **`animate-pulse`** is a built-in Tailwind utility — a low-amplitude opacity oscillation. Don't roll a custom shimmer; the built-in is the right default.
- **`role="status"` + `aria-live="polite"`** announce "Loading your results" to screen readers without interrupting whatever they're saying. Topic 7 covers the broader a11y story; this is the loading-state piece.
- **No spinner.** The shape is the affordance.

## `loading.tsx` Route Convention

Next's App Router has a file-system convention for the page-level Suspense fallback: a `loading.tsx` sibling to `page.tsx` is automatically wrapped around the whole route segment. The pattern:

```tsx
// frontend/app/results/[sessionId]/loading.tsx
import { ResultsSkeleton } from "./ResultsSkeleton";

export default function Loading() {
  return (
    <main className="mx-auto flex w-full max-w-5xl flex-col gap-6 px-4 py-6 md:px-6">
      <h1 className="text-2xl font-semibold tracking-tight">Your Results</h1>
      <ResultsSkeleton />
    </main>
  );
}
```

Two boundary patterns coexist in the route:

- **`loading.tsx`** is the *outer* fallback. It renders during navigation, before `page.tsx` has rendered anything at all (e.g., while route code is being loaded). It's wrapped around the full segment by the router.
- **The explicit `<Suspense>` in `page.tsx`** is the *inner* fallback. It renders when `page.tsx` is rendering but `ResultsBreakdown` is awaiting its data.

For this page they share the skeleton, which is fine — the skeleton is the right shape for both moments. If `loading.tsx` matched what's in `page.tsx`'s `<Suspense fallback={...}>` exactly, you could omit one. We keep both because the explicit `<Suspense>` lets the page chrome (the `<h1>`) render immediately while only the data panel skeletons; `loading.tsx` is broader.

The pragmatic rule for the cohort: **start with `loading.tsx`** for route-level loading; add explicit `<Suspense>` boundaries inside `page.tsx` when you want finer-grained reveal.

## What Causes A Suspend

A common confusion: not every async thing suspends. The mechanism is specific:

- An `async` server component whose `await` hasn't resolved yet — suspends.
- A client component that calls `use(promise)` (React 19+) with an unresolved promise — suspends.
- A library that integrates with Suspense (React Query, Apollo with `useSuspenseQuery`) — suspends.
- A plain `useEffect(() => fetch(...))` — does **not** suspend. That's the legacy pattern; it renders, then `setState`-flips.

For the results page, suspension comes from the `await fetchReportForSession(...)` inside `ResultsBreakdown` (Topic 1). No `use()` hook, no library — just the server component's natural await.

## Streaming, Briefly

App Router uses **streaming SSR** under the hood: HTML is sent to the browser in chunks as Suspense boundaries resolve. The shell of the page (the title, the skeleton) goes out immediately; the data-driven content streams in when the fetch completes; the browser progressively reveals it.

The cohort doesn't need to configure streaming — it's the default. But understanding the consequence matters: the time-to-first-byte for the results page is roughly the time to render the page chrome, not the time to fetch the report. The candidate sees the title and skeleton in <100ms even if the report fetch takes 800ms. That's the user-facing win.

## Testing Suspense Boundaries

In the D15 Playwright suite, the equivalent test pattern is:

```ts
// e2e/results.spec.ts
test("results page shows skeleton then content", async ({ page }) => {
  // Slow the reporting fetch to make the skeleton observable.
  await page.route("**/reports/user/**", async (route) => {
    await new Promise((r) => setTimeout(r, 500));
    await route.continue();
  });

  await page.goto("/results/sess_abc123");

  // Skeleton appears.
  await expect(page.getByRole("status", { name: /loading your results/i }))
    .toBeVisible();

  // Real content arrives.
  await expect(page.getByRole("heading", { name: "Score" })).toBeVisible();
  await expect(page.getByRole("status")).not.toBeVisible();
});
```

The `aria-label="Loading your results"` on the skeleton's `role="status"` is what makes this test stable. Don't query by CSS class.

## Anti-Patterns

- **Wrapping the whole `<main>` in `<Suspense>` for no reason.** If the page chrome doesn't take any work to render, putting it under the boundary just hides it during the load. Render chrome above the boundary.
- **No fallback (`<Suspense>` with no `fallback` prop).** React renders the closest ancestor boundary's fallback, which is usually nothing useful. Always provide a fallback explicit enough to occupy the layout.
- **Reintroducing `loading / setLoading` state inside an async server component.** You can't — server components don't have state — but cohort members sometimes try by converting the component back to a client one. The point of Suspense is to delete that state.
- **Spinner-in-the-corner fallbacks.** Tells the user something is happening but doesn't preserve layout. The page will shift when content arrives. Use a skeleton with the same shape.
- **Forgetting `role="status"` on the skeleton.** Sighted users see the loading state; screen-reader users don't, unless you announce it.
- **Multiple boundaries around a single fetch.** If one endpoint returns all the data, one boundary is correct. Multiple boundaries make sense when each child has its own independent fetch (Topic 1, option 3 — not used here).
- **Suspending forever.** If the fetch hangs, the skeleton hangs. Pair Suspense with the error boundary from Topic 3 and a timeout in the fetcher itself.

## Key Takeaways

- A Suspense boundary turns a slow `await` in an async server component into a fast page-chrome render plus a contained skeleton — no client state, no `useEffect`.
- For the results page, wrap `ResultsBreakdown` (the async child that awaits the report) in `<Suspense fallback={<ResultsSkeleton />}>`. Page title renders immediately; data panel suspends.
- Use the `loading.tsx` route convention for the broad route-level fallback; use explicit `<Suspense>` inside `page.tsx` for finer-grained reveal.
- Skeletons preserve layout. `role="status"` + `aria-live="polite"` make them perceivable for screen readers.
- Streaming SSR is the default — TTFB is the time to render the shell, not to fetch the data.

---
*Prerequisites: day-13-client-components-for-stateful-interactivity, day-17-server-components-for-data-heavy-pages, day-16-rest-api-design-for-read-heavy-endpoints.*
