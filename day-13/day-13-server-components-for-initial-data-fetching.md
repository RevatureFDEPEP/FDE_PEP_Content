# Server Components For Initial Data Fetching

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Once the dynamic route resolves to `/take/<testId>`, the page needs a session before it can render anything. The naive shape — render an empty page, mount a client component, fire `useEffect` to call `POST /sessions`, show a spinner, hydrate — works but produces a visible flash of empty UI on every load. Next.js 16's App Router gives us a better option: **fetch on the server, render the result, hand a fully-populated object to the client**. This file walks through using a server component as the page root, hitting D11's `POST /sessions` from the server with the auth cookie, and passing the resulting session JSON into a `<TestRunner>` client subtree as a prop.

## Why Server-First For The Session Mint

`POST /sessions` (D11 contract) creates a database row and returns the first question, the opaque session token, and `server_now` / `expires_at` for the timer. Three reasons to fetch it server-side:

1. **No loading flicker.** The page renders with the first question already in the markup — no spinner, no empty container. Time-to-interactive drops noticeably.
2. **No client-side credential exposure.** The session token comes back in the response; if we mint server-side, we can store it as an HTTP-only cookie or pass it through props without ever exposing a long-lived bearer to client JS.
3. **The mint is a once-per-load event.** It's exactly the shape server components were designed for — fetch what you need, render, done. There's no client-side reactivity to it.

The *interactive* parts (selecting an answer, navigating questions) stay client-side. The *initial state* is server-side. This is the D9 split made concrete: server for data, client for interaction.

## The Page As An Async Server Component

Server components are the default in App Router — no directive needed. `page.tsx` is async, awaits the route params (Topic 1), reads the auth cookie (Topic 5), calls the backend, and renders.

```tsx
// frontend/app/take/[testId]/page.tsx
import { cookies } from "next/headers";
import { notFound, redirect } from "next/navigation";
import { z } from "zod";
import { TestRunner } from "./TestRunner";
import type { Session } from "@/lib/api/types"; // Topic 4

const TestIdSchema = z.string().uuid();

type PageProps = {
  params: Promise<{ testId: string }>;
};

export default async function TakeTestPage({ params }: PageProps) {
  const { testId: rawTestId } = await params;
  const parsed = TestIdSchema.safeParse(rawTestId);
  if (!parsed.success) notFound();

  // Topic 5: cookie-based auth flows through next/headers' cookies()
  const cookieStore = await cookies();
  const authCookie = cookieStore.get("pep_session");
  if (!authCookie) redirect("/login?next=/take/" + parsed.data);

  const session = await mintSession(parsed.data, authCookie.value);

  return <TestRunner session={session} />;
}

async function mintSession(testId: string, authToken: string): Promise<Session> {
  const res = await fetch(`${process.env.API_BASE_URL}/sessions`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      // Forward the candidate's auth so the backend knows who is starting.
      Cookie: `pep_session=${authToken}`,
    },
    body: JSON.stringify({ test_id: testId }),
    // Server-side fetch — opt out of Next.js's default request memoization
    // because we want a fresh session on every page load.
    cache: "no-store",
  });

  if (res.status === 401) redirect("/login");
  if (res.status === 404) notFound();
  if (!res.ok) {
    throw new Error(`Session mint failed: ${res.status}`);
  }

  return res.json() as Promise<Session>;
}
```

Three Next.js 16 specifics worth calling out:

- **`cookies()` is async.** Returns a Promise of the cookie store. `await` it before reading.
- **`fetch` default is no longer "force-cache".** In Next.js 15+, `fetch` defaults to `no-store` for dynamic requests, but being explicit (`cache: "no-store"`) documents intent and survives future default changes. For a mutation like `POST` this is moot, but get into the habit.
- **`redirect()` and `notFound()` throw control-flow errors** the framework catches. Don't wrap them in try/catch.

## Why `Promise<Session>` Is The Right Return Type, Not A React Element

A common temptation is to do the fetch inside a sub-component and let Suspense handle it:

```tsx
// Tempting, but wrong for our case.
export default async function TakeTestPage({ params }: PageProps) {
  const { testId } = await params;
  return (
    <Suspense fallback={<Skeleton />}>
      <SessionLoader testId={testId} />
    </Suspense>
  );
}
```

This works, but it puts you back in the world of skeleton screens — exactly what we wanted to avoid by going server-first. For the *test-taking page specifically*, blocking the page on the session mint is the right tradeoff: the candidate is at a clear inflection point ("start the test"), a brief server-side wait is acceptable, and the payoff is a fully-formed first question rendered in the initial HTML.

Use the Suspense pattern when you have *parallel* data that's independently useful — e.g., a sidebar that can render while the main panel loads. For a page whose entire purpose is a single resource, await it at the top.

## Passing The Session To The Client Subtree

The session JSON is plain data — UUIDs as strings, ISO timestamps, the question object. It serializes cleanly across the server/client boundary that the App Router enforces:

```tsx
// frontend/app/take/[testId]/TestRunner.tsx
"use client";

import { useState } from "react";
import type { Session } from "@/lib/api/types";
import { QuestionView } from "./QuestionView";
import { Navigation } from "./Navigation";

type Props = {
  session: Session;
};

export function TestRunner({ session }: Props) {
  // Topic 8: client-held state for currentIndex + answers.
  const [currentIndex, setCurrentIndex] = useState(0);
  // session.questions[0] is what the server already rendered; subsequent
  // questions come from the same array, no further fetches today.
  const question = session.questions[currentIndex];

  return (
    <div className="mx-auto max-w-3xl space-y-6 p-6">
      <QuestionView question={question} />
      <Navigation
        currentIndex={currentIndex}
        total={session.questions.length}
        onPrev={() => setCurrentIndex((i) => Math.max(0, i - 1))}
        onNext={() => setCurrentIndex((i) => Math.min(session.questions.length - 1, i + 1))}
      />
    </div>
  );
}
```

Three rules at the boundary:

1. **Only serializable data crosses.** No functions, no class instances, no `Date` objects (use ISO strings; the client parses if needed), no `Map`/`Set`. The Next.js dev server will warn loudly if you violate this.
2. **The client component is leaf-first.** `TestRunner` carries `"use client"`; everything it imports and renders is implicitly client-rendered. Server components cannot be nested *inside* client components except via the `children` prop pattern.
3. **The prop is the source of truth at mount.** After hydration, the client component owns the session in its own state. If the user navigates away and back, the page re-mints (new server fetch, new session) — we don't try to cache across navigations today.

## What The D11 Contract Looks Like At This Boundary

For the topic-4 types file, the shape we're consuming is approximately:

```jsonc
// POST /sessions response (D11)
{
  "session_id": "0193d3a4-...",
  "session_token": "sess_oo3...",       // opaque, holds in HTTP-only cookie or pass to client
  "test_id": "...",
  "server_now": "2026-05-19T14:32:01Z",
  "expires_at": "2026-05-19T15:32:01Z",
  "questions": [
    {
      "question_id": "...",
      "type": "single_select",          // discriminator — Topic 7
      "stem": "What does CORS stand for?",
      "options": [{ "id": 0, "label": "..." }, ...]
      // note: NO correct_options — server-authoritative scoring (D11/D12)
    },
    { "type": "multi_select", "options": [...], ... }
    // ... N more
  ]
}
```

The frontend never sees `correct_options`. Scoring happens on submit (D14) and the result comes back from `POST /sessions/{id}/answer` (D12 contract). The D11 design — return *all* questions up front — means today's page doesn't need any further fetches for navigation; switching questions is pure client-side state (Topic 8).

## Trade-Offs To Acknowledge

- **All-or-nothing latency.** The whole session payload (~5–10KB for 20 questions) is on the critical path. Acceptable here; not always.
- **No partial degradation.** If the API is down, the page fails hard. The catch in `mintSession` should render a friendly error page — Day 14 territory.
- **The server fetch is a server resource.** Each page load is a Next.js server-side `POST` to the backend. At 25 concurrent candidates this is trivial; at 2,500 it would be a thundering herd worth load-testing.

## Common Mistakes

- **Forgetting `cache: "no-store"`** on the server-side `POST`. POST isn't memoized by default in 16, but in mixed codebases with GET helpers you can accidentally inherit aggressive caching wrappers. Be explicit on mutations.
- **Reading cookies from `document.cookie` on the client** to forward to a server-side fetch — that's not how server components work. Use `cookies()` from `next/headers`.
- **Returning the session as a React element from the server component instead of passing it as a prop.** Loses type information at the boundary; the client side has no `session` it can store in state.
- **Treating the server component as long-lived.** It runs once per request, renders to HTML, and is gone. No `useState`, no `useEffect`, no event handlers — those belong in the client subtree.

## Key Takeaways
- The page root is an async server component that awaits params, validates them, forwards the auth cookie, and performs the D11 `POST /sessions` server-side.
- Server-first fetching eliminates loading flicker and keeps the session-mint round-trip off the client critical path.
- The fully-formed session JSON is handed to a `"use client"` `<TestRunner>` as a prop — plain serializable data only.
- `cookies()`, `params`, and `searchParams` are all async in Next.js 16 — `await` them.
- Use `redirect()` for auth failures and `notFound()` for missing/invalid IDs; both are framework-level control-flow signals, not exceptions to catch.

---
*Prerequisites: day-9-server-vs-client-components, day-11-opaque-token-generation-and-session-identifiers, day-13-dynamic-routing-with-app-router-parameters. Forward references: day-13-client-components-for-stateful-interactivity, day-13-authentication-context-propagation-through-the-component-tree.*
