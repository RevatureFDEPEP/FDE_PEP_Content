# Dynamic Routing With App Router Parameters

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The candidate clicks "Start Test" on the dashboard and lands on `/take/0193d3a4-...`. That URL has to resolve to a page, the page has to know *which* test, and that ID has to flow into the D11 `POST /sessions` call we'll make in Topic 2. Next.js 16's App Router models this with **dynamic segments** — folder names wrapped in brackets — and gives the page a typed `params` prop. This file covers the file-system convention, the `params` vs `searchParams` split, the Next.js 16 async-params requirement, and the typing discipline that makes the test-taking page's URL contract a compile-time check rather than a string-fishing exercise.

## The File-System Convention

App Router routes are folders under `app/`. A folder named `[testId]` becomes a parameterized segment:

```
frontend/app/
├── (dashboard)/
│   └── tests/page.tsx              → /tests
└── take/
    └── [testId]/
        ├── page.tsx                → /take/:testId
        ├── layout.tsx              (optional, wraps the page)
        └── loading.tsx             (optional, Suspense fallback)
```

The bracketed segment name becomes the key in the `params` object the framework passes to the page. So `[testId]` → `params.testId`. A bracketed segment matches *exactly one* URL segment — `/take/abc` matches, `/take/abc/def` does not (that would need a nested route).

Variant bracket forms exist but we don't need them here:
- `[...slug]` — catch-all, matches any number of segments.
- `[[...slug]]` — optional catch-all, matches the parent route too.
- `[[testId]]` — optional single segment.

For the test-taking page we want exactly one ID, required, so plain `[testId]` is right.

## The Page Signature In Next.js 16

In Next.js 15 the `params` prop became a `Promise`, and 16 keeps that contract. The page is an async function that awaits `params` before reading the segment:

```tsx
// frontend/app/take/[testId]/page.tsx
type PageProps = {
  params: Promise<{ testId: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
};

export default async function TakeTestPage({ params }: PageProps) {
  const { testId } = await params;
  // testId is a string — narrow further before passing to APIs.
  return <TestRunnerShell testId={testId} />;
}
```

Two things to internalize:

1. **`params` is a Promise.** You `await` it inside the server component. Trying to read `params.testId` directly (the pre-15 style) is a runtime error in dev and a type error if you've typed it correctly.
2. **It's already typed as `string`** because URL segments are always strings. There is no automatic coercion to `UUID` or `number` — that's your job at the boundary (Topic 4 covers schema validation; for now a `z.string().uuid()` guard is sufficient).

## `params` vs `searchParams` — Different Lifecycles

Both arrive as Promises, both contain strings, but they mean different things:

| | `params` | `searchParams` |
|---|---|---|
| Source | Dynamic segments in the path (`/take/[testId]`) | Query string after `?` (`?from=dashboard`) |
| Required | Yes — a missing segment is a 404 | No — always optional |
| Cacheability | Treated as part of the route key; full static optimization possible | Marks the page as dynamic on read |
| Use for | Resource identifiers, primary keys | Filters, UI state, opt-in flags |

For the test-taking page the only thing in the URL is the `testId` — no query params today. On Day 14 we may add `?resume=1` to differentiate "fresh start" from "resume an in-progress session", which would be a `searchParams` read.

Reading `searchParams` opts the page out of static rendering. `params` does not, on its own — a `[testId]` route with no `searchParams` access and no `cookies()`/`headers()` call could still be statically generated for known IDs via `generateStaticParams`. We won't use that here (test IDs are user-specific and known only at request time), but it's worth knowing where the line is.

## Validating The Segment

`testId` arrives as `string`. The D11 contract says session/test IDs are UUIDs. Hand the framework's `string` straight to a `fetch` call and you'll discover the typo at the network layer, far from the source. Validate at the boundary:

```tsx
import { z } from "zod";
import { notFound } from "next/navigation";

const TestIdSchema = z.string().uuid();

export default async function TakeTestPage({ params }: PageProps) {
  const { testId: rawTestId } = await params;
  const parsed = TestIdSchema.safeParse(rawTestId);
  if (!parsed.success) {
    notFound(); // renders app/not-found.tsx, returns 404
  }
  const testId = parsed.data; // now typed as a validated UUID string

  return <TestRunnerShell testId={testId} />;
}
```

`notFound()` is the App Router idiom — it throws a special error the framework catches and converts to a 404 response with your `not-found.tsx` UI. Don't `throw new Error("bad id")`; that becomes a 500.

## How `testId` Flows Into The Day Today

The page is the entry point. Today the chain is:

```
URL /take/<uuid>
   │
   ▼
[testId]/page.tsx (server component)
   │  • await params, validate
   │  • Topic 5: read auth cookie
   │  • Topic 2: POST /sessions { test_id: testId }, get back session JSON
   │
   ▼
<TestRunner session={...} /> (client component, Topic 3)
   │
   ▼  user navigates between questions (Topic 8)
```

The `testId` lives in the page for ~1 statement — it's the input to `POST /sessions`. Once we have a session, the session_id takes over as the identifier for all subsequent calls (the D11 contract is session-id-centric, not test-id-centric).

## Common Mistakes

- **Reading `params.testId` synchronously.** Works in older codebases, fails in Next.js 16. Always `await params`.
- **Typing `params` as `{ testId: string }` instead of `Promise<{ testId: string }>`**. Compiles, then misbehaves at runtime when the page tries to `.then` something it can't.
- **Treating the string as already-validated.** A URL is user input. Validate UUID-ness at the page boundary; don't push that responsibility to the backend ("the API will reject it") — by then you've already done a wasteful network call and need user-visible error handling for a case `notFound()` handles cleanly.
- **Putting auth/session logic in the segment folder name.** `[testId]` is for the resource ID, not for `[userId]/[testId]` cuteness — the user identity comes from the auth cookie (Topic 5), not the URL.
- **Naming the segment something generic like `[id]`.** When this page renders inside a layout that also has a `[sessionId]` sibling, the params object becomes ambiguous. Use the specific name (`testId`) and the params object self-documents.

## Key Takeaways
- Dynamic segments are folders wrapped in brackets: `app/take/[testId]/page.tsx` matches `/take/:testId`.
- In Next.js 16, `params` arrives as a `Promise` — async pages `await` it before reading segments.
- `params` and `searchParams` are both Promises of string maps; reach for `params` for resource IDs, `searchParams` for filters/UI flags.
- Validate the segment with a schema (`z.string().uuid()`) and call `notFound()` on failure — don't let invalid input reach the API.
- The page is a thin server-component entry point that consumes the segment, performs the D11 session fetch, and hands a typed session object down to a client subtree.

---
*Prerequisites: [01-nextjs-16-app-router-fundamentals.md](../day-09/01-nextjs-16-app-router-fundamentals.md), [07-opaque-token-generation-and-session-identifiers.md](../day-11/07-opaque-token-generation-and-session-identifiers.md). Forward references: [03-server-components-for-initial-data-fetching.md](03-server-components-for-initial-data-fetching.md), [06-authentication-context-propagation-through-the-component-tree.md](06-authentication-context-propagation-through-the-component-tree.md).*
