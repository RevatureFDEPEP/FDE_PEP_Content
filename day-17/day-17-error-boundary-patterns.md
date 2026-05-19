# Error Boundary Patterns

> *Day 17: Results Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The reporting service is a new piece of infrastructure that talks to two databases and a sibling service. It will fail. The relevant questions are: *how does it fail visibly without blanking the entire page*, and *what does the candidate do after seeing the failure*? React's error boundary mechanism — plus Next's `error.tsx` route convention — answers both. Topic 3 covers the floor: a route-level error boundary that catches anything thrown during render (including the network exceptions thrown by Topic 1's `fetchReportForSession`), shows a useful message, and offers a retry. Then the slightly more advanced pattern: a per-section error boundary so a single panel can fail without taking the rest of the page with it.

## The Mechanism, Briefly

An error boundary is a React component that implements `componentDidCatch` (class-based) or — more commonly now — is supplied by a framework convention. It catches errors thrown during the render of its descendants and renders a fallback UI instead.

What it catches:

- Errors thrown during render of components below it.
- Errors thrown in lifecycle methods of those components.
- Errors thrown by Topic 2's suspended promises that ultimately *reject* (the Suspense boundary handles the *pending* state; the error boundary handles the *rejected* state).

What it does *not* catch:

- Errors thrown in event handlers (`onClick` handlers — you handle those manually).
- Errors in `setTimeout` / `setInterval` callbacks.
- Errors in async callbacks (e.g., `.then` chains that aren't `await`ed in a server component).
- Server-action errors that happen post-render.

For the results page, the failures we need to cover are the rejected-promise variety: `fetchReportForSession` throws `ReportNotFoundError`, throws a generic 500, or the network drops mid-fetch. All of those propagate out of the `await` and land in the nearest error boundary.

## `error.tsx` Route Convention

Next's App Router has a file-system convention for route-level error UIs. An `error.tsx` sibling to `page.tsx` is automatically wrapped around the route segment. It must be a client component (it uses state to support retries).

```tsx
// frontend/app/results/[sessionId]/error.tsx
"use client";

import { useEffect } from "react";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

type Props = {
  error: Error & { digest?: string };
  reset: () => void;
};

export default function ResultsError({ error, reset }: Props) {
  useEffect(() => {
    // Surface to whatever client-side logging the repo uses.
    // The digest is a server-stable ID Next attaches; useful when the
    // candidate reports the error and the trainer wants to find it in logs.
    console.error("Results page error", { message: error.message, digest: error.digest });
  }, [error]);

  return (
    <main className="mx-auto flex w-full max-w-2xl flex-col gap-6 px-4 py-12">
      <Card>
        <CardHeader>
          <CardTitle>We couldn’t load your results</CardTitle>
        </CardHeader>
        <CardContent className="flex flex-col gap-4">
          <p className="text-sm text-muted-foreground">
            Something went wrong fetching your results. This is usually
            temporary. You can retry, or head back to the dashboard.
          </p>
          {error.digest && (
            <p className="text-xs text-muted-foreground">
              Reference: <code>{error.digest}</code>
            </p>
          )}
          <div className="flex gap-3">
            <Button onClick={() => reset()}>Try again</Button>
            <Button variant="outline" asChild>
              <a href="/">Back to dashboard</a>
            </Button>
          </div>
        </CardContent>
      </Card>
    </main>
  );
}
```

The contract:

- **`error` prop** is whatever was thrown. In production, Next strips the message for security — the candidate sees a generic Error with the original message replaced by a digest. In development, the original message is preserved.
- **`reset` prop** is a function that, when called, attempts to re-render the route segment. The component receives this from Next; calling it triggers a fresh server render of the route, which means a fresh fetch. That's the retry mechanism.
- **`digest`** is a deterministic hash of the original error that's safe to expose to the client. Pair it with server-side logs (D10's request-id correlation) to track down what actually went wrong.

## The Retry UX

`reset()` is the key. The candidate sees "we couldn't load your results", clicks "Try again", and the route re-renders. If the failure was transient (a flaky DB connection, a momentary 502), the second attempt succeeds and the candidate sees their results. If the failure persists, the error boundary re-renders the error.

Two design choices worth being explicit about:

- **Don't auto-retry.** The temptation to retry-on-mount with backoff is real and almost always wrong for user-facing errors. If the candidate doesn't take a deliberate action, the page silently flickers between error and loading states and they can't tell what's happening. Manual retry, explicit button.
- **Don't retry forever.** Three failed retries in a row probably means the system is actually down; offering a fourth retry button is unkind. For the PEP scope, count retries client-side and after three consecutive failures swap the button for a "the system appears to be down; contact your trainer" message. (Implementation in the appendix below — optional.)

## Per-Section Error Boundaries

`error.tsx` covers the *whole route*. If the chart panel throws (recharts can throw on malformed data) but the summary fetched fine, do you want to blank the whole page? Probably not. The fix: wrap the chart-rendering region in its own error boundary.

The `error.tsx` convention doesn't help here — that's route-level only. For component-level boundaries, use `react-error-boundary` or hand-roll a small one.

```tsx
// frontend/components/SectionErrorBoundary.tsx
"use client";

import { Component, type ReactNode } from "react";

type Props = {
  fallback: (error: Error, reset: () => void) => ReactNode;
  children: ReactNode;
};

type State = { error: Error | null };

export class SectionErrorBoundary extends Component<Props, State> {
  state: State = { error: null };

  static getDerivedStateFromError(error: Error) {
    return { error };
  }

  componentDidCatch(error: Error) {
    console.error("SectionErrorBoundary caught", error);
  }

  reset = () => this.setState({ error: null });

  render() {
    if (this.state.error) return this.props.fallback(this.state.error, this.reset);
    return this.props.children;
  }
}
```

Usage inside `ResultsBreakdown`:

```tsx
<Card className="md:col-span-2">
  <CardHeader>
    <CardTitle>Per-question breakdown</CardTitle>
  </CardHeader>
  <CardContent>
    <SectionErrorBoundary
      fallback={(_, reset) => (
        <div role="alert" className="flex flex-col gap-2 text-sm">
          <p className="text-muted-foreground">
            Couldn’t render the chart. The data below is still accurate.
          </p>
          <Button size="sm" variant="outline" onClick={reset}>
            Retry chart
          </Button>
        </div>
      )}
    >
      <ResultsChart questions={attempt.per_question} />
    </SectionErrorBoundary>
  </CardContent>
</Card>
```

Now: chart panel throws → only the chart panel shows the fallback → summary card and details table still render normally. The candidate has their score, just no visualization.

This is also where the `role="alert"` attribute earns its keep — Topic 7 covers a11y broadly; for error UIs specifically, `role="alert"` ensures screen readers announce the failure immediately when it appears.

## Server Component Errors vs Client Component Errors

A subtle distinction worth naming for the cohort:

- **Server component throws during render** (e.g., the `await fetchReportForSession(...)` rejects). Next catches this on the server, renders the route's `error.tsx` boundary, and ships *that* HTML to the client. Streaming SSR can still send the page shell first if the error happens inside a Suspense boundary.
- **Client component throws during render** (e.g., recharts chokes on bad data). React's reconciler catches this on the client, walks up to the nearest error boundary in the tree (our `SectionErrorBoundary` or the route's `error.tsx`), and renders the fallback locally.

Both paths funnel into error-boundary UI. The difference is *where* the error happened (which determines what's in logs) and *whether* the page chrome rendered first.

## Differentiating 404 From Generic Failures

`fetchReportForSession` throws `ReportNotFoundError` for a 404 from the reporting service (Topic 1). That's semantically different from a 500: "no report for this session" is a not-found condition the user should see clearly, not a system failure.

Next has a separate convention for this — `notFound()`:

```ts
// inside ResultsBreakdown
import { notFound } from "next/navigation";

try {
  const { attempt } = await fetchReportForSession(sessionId);
  // ...
} catch (err) {
  if (err instanceof ReportNotFoundError) notFound();
  throw err;
}
```

`notFound()` throws a special error Next intercepts and routes to `not-found.tsx`. Build that file too:

```tsx
// frontend/app/results/[sessionId]/not-found.tsx
export default function NotFound() {
  return (
    <main className="mx-auto flex w-full max-w-2xl flex-col gap-6 px-4 py-12">
      <h1 className="text-2xl font-semibold">We don’t have a report for that session</h1>
      <p className="text-sm text-muted-foreground">
        The session ID in the URL doesn’t match any completed test.
        Double-check the link, or head back to the dashboard.
      </p>
      <a href="/" className="text-sm underline">Back to dashboard</a>
    </main>
  );
}
```

Splitting 404 from generic errors keeps the messaging honest: "we couldn't reach the service" and "this thing doesn't exist" are different problems with different user actions.

## What To Log

Both boundaries should funnel into the logs the trainer can grep on D10's request-id correlation. The bare minimum:

- The error message (server-side; client-side has only the digest in production).
- The `X-Request-Id` the fetcher attached.
- The session ID and user ID context.
- A timestamp.

For PEP we keep this simple — `console.error` on the client, structured logs already wired on the server. Phase 2 hooks this into a real error tracker (Sentry, etc.); flag the seam.

## Anti-Patterns

- **`try / catch` around every `await` instead of letting it propagate.** Defeats the point of error boundaries. Let errors bubble to the boundary; catch only where you have a *specific* recovery (e.g., the 404 → `notFound()` translation).
- **Swallowing errors with empty UI.** `{error ? null : <Content />}` shows the candidate a blank panel and no path forward. Always render a fallback.
- **Showing raw error messages in production.** Stack traces and internal class names leak architectural info. Render a friendly message; log the detail.
- **Auto-retry loops.** Silent retries hide problems and waste backend capacity. Manual button only.
- **One boundary for everything.** Whole-page failures for chart-rendering bugs are bad UX. Use route-level for "the data fetch failed"; section-level for "this widget specifically broke."
- **Forgetting `role="alert"` on the fallback UI.** Screen-reader users don't get the visual signal that something failed.
- **Throwing strings or plain objects.** `throw "bad"` is not an Error; the boundary will catch it but `error.message` is undefined. Always throw `Error` instances (or subclasses, like `ReportNotFoundError`).

## Key Takeaways

- `error.tsx` is the route-level error boundary. It receives `error` and `reset` props; the reset is the candidate-visible retry.
- Use `notFound()` + `not-found.tsx` to distinguish "this report doesn't exist" from "the system is broken." Different messages, different user actions.
- Add a section-level `SectionErrorBoundary` around risky regions (the chart) so a single widget can fail without blanking the page.
- Don't auto-retry. Show the error, offer a button, log enough context to debug.
- Use `role="alert"` on fallback UIs so screen readers announce the failure.

---
*Prerequisites: day-17-server-components-for-data-heavy-pages, day-17-loading-states-with-suspense-boundaries, day-10-distributed-log-correlation-across-services.*
