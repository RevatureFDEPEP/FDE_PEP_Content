# Server Components For Data-Heavy Pages

> *Day 17: Results Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The candidate submitted the quiz on D14, the backend aggregated the score on D15, and D16 grew a `GET /reports/user/{userId}` endpoint that returns the whole summary-plus-attempts envelope in one round trip. Today the cohort builds the page that consumes it. The right default for that page is a **server component** — read-heavy, low-interactivity work belongs on the server side of the App Router boundary, where the data fetch is one network hop away instead of one across the public internet plus a bundle download plus a hydration tick. D13 introduced server components as the outer chrome of an otherwise client-driven test-taking page; today they're the chassis of nearly the entire screen, with one client island (the chart) carved out.

> **Note on MVF fallback.** Day 17 ends Week 3's content with the **MVF fallback assessment**: candidates clearly off-pace can step down to a Minimum Viable Frontend path — results-only, intentionally unstyled — for the rest of the week. MVF is a defined alternative, not a failure mode. The server-component skeleton below *is* the MVF surface; the styling, chart, and accessibility polish across Topics 4–7 are the standard path on top of it. Trainers triage at EoD; see the ops runbook.

## Why Server Components For This Screen

A page is a good fit for "mostly server component" when it's:

- **Read-heavy.** Page load equals one fetch; user interactions are minor (toggling a tab, expanding a row), not screen-defining.
- **Data-shaped.** Most of the markup is `{report.summary.average_score}`-style interpolation of a JSON payload, not stateful UI.
- **SEO- or share-relevant.** A candidate may bookmark or paste the link; the server-rendered HTML should make sense without JavaScript.
- **Tolerant of network latency on first paint.** The user clicked "View my results"; they expect a short wait, not an instant skeleton.

The results page hits all four. The only piece that *requires* client-side execution is the recharts visualization (Topic 4) — recharts measures the container and renders SVG via React state, neither of which works in a server-only context. Everything else — the summary card, the per-question table, the layout chrome — is text rendered from a JSON object. Shipping that as a client bundle would be paying for hydration that has nothing to hydrate.

The discipline carries D13's rule forward: **push the `"use client"` boundary as far down the tree as you can, then stop.**

## File Layout

```
frontend/app/results/[sessionId]/
├── page.tsx                 ← server component, default export, fetches the report
├── ResultsBreakdown.tsx     ← async server component, renders summary + table
├── ResultsChart.tsx         ← "use client", recharts island (Topic 4)
├── loading.tsx              ← Suspense fallback skeleton (Topic 2)
├── error.tsx                ← error boundary (Topic 3)
└── lib/
    └── fetchReport.ts       ← server-only fetcher; calls reporting service
```

D13 used `app/take/[testId]/page.tsx` for the test runner — the convention is consistent. The new wrinkle is that `page.tsx` here is `async` from the start, not just a thin wrapper that mounts a client island.

## The Page Component

```tsx
// frontend/app/results/[sessionId]/page.tsx
import { Suspense } from "react";
import { fetchReportForSession } from "./lib/fetchReport";
import { ResultsBreakdown } from "./ResultsBreakdown";
import { ResultsChart } from "./ResultsChart";
import { ResultsSkeleton } from "./ResultsSkeleton";

type PageProps = {
  params: Promise<{ sessionId: string }>;
};

export default async function ResultsPage({ params }: PageProps) {
  const { sessionId } = await params;

  return (
    <main
      id="main"
      className="mx-auto flex w-full max-w-5xl flex-col gap-6 px-4 py-6 md:px-6"
    >
      <h1 className="text-2xl font-semibold tracking-tight">Your Results</h1>

      <Suspense fallback={<ResultsSkeleton />}>
        <ResultsBreakdown sessionId={sessionId} />
      </Suspense>
    </main>
  );
}
```

Things to notice that follow from the server-component model:

- The function is `async`. Server components are the only kind that may be.
- `params` is awaited — App Router treats it as a `Promise` in current Next.js versions; D13 covered the same shape.
- The page does **not** call `fetch` directly. It delegates to `fetchReportForSession` inside `ResultsBreakdown`, which is the async child the `<Suspense>` boundary is wrapping. That structure is what gives Topic 2 a clean loading state — the parent renders immediately with the page chrome and skeleton; the child suspends on the fetch.
- The only imported client component (`ResultsChart`) is mounted *inside* `ResultsBreakdown`, where the data lives. The page itself doesn't touch the chart.

## The Async Server Component That Fetches

```tsx
// frontend/app/results/[sessionId]/ResultsBreakdown.tsx
import { fetchReportForSession } from "./lib/fetchReport";
import { ResultsChart } from "./ResultsChart";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

type Props = { sessionId: string };

export async function ResultsBreakdown({ sessionId }: Props) {
  const { attempt } = await fetchReportForSession(sessionId);

  return (
    <div className="grid grid-cols-1 gap-6 md:grid-cols-3">
      <Card className="md:col-span-1">
        <CardHeader>
          <CardTitle>Score</CardTitle>
        </CardHeader>
        <CardContent>
          <p className="text-4xl font-semibold">
            {attempt.score}
            <span className="text-base text-muted-foreground">
              {" "}
              / {attempt.max_score}
            </span>
          </p>
          <p className="mt-2 text-sm text-muted-foreground">
            Completed in {formatDuration(attempt.elapsed_seconds)}
          </p>
        </CardContent>
      </Card>

      <Card className="md:col-span-2">
        <CardHeader>
          <CardTitle>Per-question breakdown</CardTitle>
        </CardHeader>
        <CardContent>
          <ResultsChart questions={attempt.per_question} />
        </CardContent>
      </Card>

      <Card className="md:col-span-3">
        <CardHeader>
          <CardTitle>Details</CardTitle>
        </CardHeader>
        <CardContent>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead scope="col">Question</TableHead>
                <TableHead scope="col">Result</TableHead>
                <TableHead scope="col">Time</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {attempt.per_question.map((q, i) => (
                <TableRow key={q.question_id}>
                  <TableCell>Q{i + 1}</TableCell>
                  <TableCell>{q.correct ? "Correct" : "Incorrect"}</TableCell>
                  <TableCell>{formatDuration(q.elapsed_seconds)}</TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}

function formatDuration(seconds: number): string {
  const m = Math.floor(seconds / 60);
  const s = seconds % 60;
  return m > 0 ? `${m}m ${s}s` : `${s}s`;
}
```

What this buys:

- The entire summary card, the entire table, and the layout grid are **server-rendered**. They arrive as HTML, with text already interpolated.
- `ResultsChart` is a client component — it sits inside server-rendered markup as a child, which the App Router handles transparently. The client bundle for this page is just the chart.
- `formatDuration` runs on the server during render. No JavaScript ships for it.
- The `<Table>` component (shadcn primitive — Topic 5) is itself a server-friendly composition of `<table>` semantic markup. Topic 7 returns to why that matters for a11y.

## The Server-Only Fetcher

```ts
// frontend/app/results/[sessionId]/lib/fetchReport.ts
import "server-only";
import { cookies } from "next/headers";
import type { ReportResponse } from "@/lib/api/types";

const REPORTING_BASE_URL =
  process.env.REPORTING_BASE_URL ?? "http://reporting-and-analytics:8000";

export async function fetchReportForSession(
  sessionId: string,
): Promise<ReportResponse> {
  const cookieStore = await cookies();
  const userId = cookieStore.get("user_id")?.value;
  if (!userId) throw new Error("Not authenticated");

  const res = await fetch(
    `${REPORTING_BASE_URL}/reports/user/${userId}?session_id=${sessionId}`,
    {
      headers: { "X-Request-Id": crypto.randomUUID() },
      // No `cache: "force-cache"` — results change as new attempts complete.
      // No `cache: "no-store"` either — the page is server-rendered per request anyway.
    },
  );

  if (res.status === 404) throw new ReportNotFoundError(sessionId);
  if (!res.ok) throw new Error(`Reporting service returned ${res.status}`);

  return res.json();
}

export class ReportNotFoundError extends Error {
  constructor(public sessionId: string) {
    super(`No report for session ${sessionId}`);
    this.name = "ReportNotFoundError";
  }
}
```

Two specifics worth slowing down on:

- **`import "server-only"`.** The package is a build-time tripwire: if a client component ever imports this module, the build fails. That prevents the next cohort member from accidentally pulling the fetcher (and its cookies + service URL) into the client bundle.
- **Service-to-service URL is `http://reporting-and-analytics:8000`.** Inside Compose, the frontend container reaches the reporting container by service name — the D6 reverse-proxy rewrite only applies to *browser* traffic. Server components fetch directly. The cohort fought this distinction on D13 and again on D15; refresh the mental model: server-side fetches go service-to-service; client-side `fetch` goes through the same-origin proxy.

## What Goes Wrong If You Skip This

Concretely, what you lose if you build the same page as a single `"use client"` tree:

1. **First paint blocks on hydration.** Client component → empty page until the bundle arrives, parses, and runs the `useEffect` that fetches data. With server components, the HTML has the score baked in.
2. **Bundle bloat.** The shadcn `<Table>` is small; ten of them plus the `formatDuration` helper plus the cookies-reading logic plus the auth check is not. Server components ship none of that.
3. **Auth cookie exposure.** The cookies-based auth pattern from D13 is server-only. Doing it in a client component means either reading a JS-accessible cookie (security regression) or POSTing through an API route just to read your own auth state.
4. **Cross-service fetch from the browser.** A `useEffect(() => fetch("/reports/..."))` works only because the D6 proxy is doing the same-origin rewrite. Forget about that proxy for a second and the page is broken in a way that's hard to debug.

The architectural payoff of the App Router is exactly this: you get to pick the right execution context per region of the page. Don't give it away for one component that needs `useState`.

## Anti-Patterns

- **`"use client"` at the page root.** Forces the entire tree client-side; you lose every benefit of server components for no gain. The chart needs to be client; everything else does not.
- **Fetching in `useEffect` instead of `await` in the server component.** Adds a network round trip after hydration, breaks suspense, and exposes the service URL to the browser.
- **Importing the server-only fetcher from a client component.** The `server-only` package catches this at build time, but if you've removed the import for some reason, the runtime crash is opaque. Keep the discipline of separate `lib/` modules per execution context.
- **Mixing data fetch and rendering in the same async component when the fetch is slow.** Better to split: a parent that renders chrome, an async child that suspends on the fetch. Topic 2 makes that explicit.
- **Calling `cookies()` or `headers()` "just in case" from inside what should be a pure render.** Reading those server-only APIs forces the route to be dynamic; do it where you actually need them (the fetcher), not in every nested component.
- **Forgetting the `X-Request-Id` header.** D10 wired log correlation across services; pass the header so the reporting service's logs line up with the frontend request.

## Key Takeaways

- The results page is a server component by default; only the chart needs `"use client"`. That keeps the bundle minimal and the data fetch one hop from the database.
- Split the page into a chrome-rendering `page.tsx` and an async `ResultsBreakdown` child so Topic 2's Suspense boundary has something concrete to wrap.
- Server-side fetches go service-to-service (`http://reporting-and-analytics:8000`); the D6 reverse proxy is for browser traffic.
- Use `import "server-only"` on the fetcher to make accidental client-side imports a build failure, not a runtime surprise.
- MVF fallback at EoD: the bare server-rendered skeleton built here is the assessment floor. Styling, chart, and a11y polish are the standard-path additions on top of it.

---
*Prerequisites: day-13-client-components-for-stateful-interactivity, day-13-dynamic-routing-with-app-router-parameters, day-16-rest-api-design-for-read-heavy-endpoints.*
