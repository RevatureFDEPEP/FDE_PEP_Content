# Authentication Context Propagation Through The Component Tree

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The candidate logged in days ago (inherited auth from the brownfield repo). The session backing that login is a cookie — `pep_session=<opaque>` — set HTTP-only by the user-service. Today's page needs that identity in two places: server-side, to attach the cookie when minting the session (Topic 2); and client-side, to attach the cookie when submitting answers (Topic 8/D14). Two different runtimes, one logical identity, propagated through the App Router's server and client trees without leaking the cookie into JavaScript bundles. This file covers the cookies-on-the-server pattern, `credentials: "include"` on the client fetch, and a small `AuthContext` provider that exposes *display* identity (username, role) to the client subtree without ever exposing the token itself.

## What "Auth" Means In This Codebase

The inherited auth is cookie-based:

- Login → user-service sets `Set-Cookie: pep_session=<opaque>; HttpOnly; Secure; SameSite=Lax; Path=/`.
- Every subsequent request from the same origin automatically includes the cookie.
- `HttpOnly` means JavaScript cannot read it. `document.cookie` returns nothing useful for `pep_session`.
- The opaque token is validated server-side on every protected route.

The implication for Day 13:

1. The **server** can read the cookie via `cookies()` from `next/headers` and forward it on outbound `fetch` calls.
2. The **client** cannot read the token. It can *cause* the cookie to be sent on `fetch` (with `credentials: "include"`) but can never see the value.
3. Anything the client needs for UI — "Hi, Aisha", "Role: Candidate" — has to come from a separate source: either a server-rendered prop or a server endpoint like `GET /me`.

## Server-Side Propagation: `cookies()` From `next/headers`

The server component at `/take/[testId]/page.tsx` reads the cookie and forwards it to the backend. In Next.js 16, `cookies()` is async — it returns a Promise of the cookie store.

```tsx
// frontend/app/take/[testId]/page.tsx
import { cookies } from "next/headers";
import { redirect } from "next/navigation";

export default async function TakeTestPage({ params }: PageProps) {
  const { testId } = await params;

  const cookieStore = await cookies();
  const authCookie = cookieStore.get("pep_session");
  if (!authCookie) {
    redirect(`/login?next=/take/${testId}`);
  }

  // Forward to backend in the outbound POST.
  const session = await fetch(`${process.env.API_BASE_URL}/sessions`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Cookie: `pep_session=${authCookie.value}`,
    },
    body: JSON.stringify({ test_id: testId }),
    cache: "no-store",
  }).then((r) => r.json());

  return <TestRunner session={session} />;
}
```

Notes on what we did and didn't do:

- **We didn't pass the auth token as a prop to `TestRunner`.** A prop crosses into the client bundle as serialized JSON in the HTML payload. The token would be visible in "View Source". We forward it only on the server-to-server hop.
- **We did pass the *session* token (`session.session_token`) as part of the session prop.** That's a different token — the D11 short-lived opaque session credential, scoped to this quiz attempt, intended for the client to wield. The auth cookie stays server-side; the session token crosses the boundary.
- **`cookies()` is read-only in server components.** Writing requires a Server Action or Route Handler. We're only reading today.

## Why Not Just Read The Cookie In The Client?

A `fetch` from the client with `credentials: "include"` will *send* the cookie automatically, but the JS code never sees the value:

```ts
// In a client component
await fetch("/api/sessions", {
  method: "POST",
  credentials: "include", // ← include cookies on same-origin/CORS requests
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ test_id: testId }),
});
```

This works perfectly — the browser attaches `pep_session` automatically because the cookie was set with `Path=/`. So why did we do the server-side fetch instead?

Two reasons:

1. **Server-side fetch eliminates the loading flicker** (Topic 2). The page renders with the session already present.
2. **Server-side fetch lets us call the backend directly** (`API_BASE_URL=http://api-gateway:8000` inside Docker, or the public API URL in prod). The client-side fetch has to go through a Next.js proxy or be cross-origin-configured, complicating CORS.

For the initial mint, server-side wins. For subsequent calls (submitting answers, fetching results) on the client, `credentials: "include"` is the answer.

## Client-Side Propagation: `credentials: "include"`

In `TestRunner` (or the answer-submit helper imported into it):

```ts
// frontend/lib/api/client.ts
"use client"; // implied by the use site; the module is client-only

import type { AnswerSubmit, AnswerResponse } from "./types";

export async function submitAnswer(
  sessionId: string,
  sessionToken: string,
  idempotencyKey: string,
  body: AnswerSubmit,
): Promise<AnswerResponse> {
  const res = await fetch(`/api/sessions/${sessionId}/answer`, {
    method: "POST",
    credentials: "include", // sends pep_session cookie automatically
    headers: {
      "Content-Type": "application/json",
      "X-Session-Token": sessionToken,           // D11 session credential
      "Idempotency-Key": idempotencyKey,         // D12 idempotency
    },
    body: JSON.stringify(body),
  });
  if (!res.ok) throw new ApiError(res.status, await res.text());
  return res.json();
}
```

Three credentials in play on this one call, three different origins for each:

| Credential | Where it came from | What it proves |
|---|---|---|
| `pep_session` cookie | Login (days ago), auto-attached by browser | Candidate identity |
| `X-Session-Token` | D11 `POST /sessions` response, server-rendered into the page | This quiz session belongs to this candidate |
| `Idempotency-Key` | Client UUID, fresh per submission attempt | This is a specific submission, not a retry |

The cookie is *implicit* (the browser handles it); the other two are explicit headers. Forgetting `credentials: "include"` is the most common Day 13/14 bug — the request goes out cookie-less, the backend returns 401, and the candidate sees an inscrutable error.

For same-origin requests (frontend and backend behind the same proxy, both at `https://pep.local`), `"same-origin"` is the default and you can omit the option. The PEP local dev setup runs Next.js on `:3000` and the api-gateway on `:8000` — different ports, technically cross-origin — so `"include"` is required and the backend needs `Access-Control-Allow-Credentials: true` + an explicit `Access-Control-Allow-Origin` (not `*`).

## Exposing Display Identity To Client Components

The client subtree often needs to know *who* the candidate is for UI purposes — header, role-gated buttons, etc. The token is unavailable; pass the display info as data.

Pattern: load `GET /me` server-side in a layout, pass user info into a small `AuthProvider` that exposes it via context.

```tsx
// frontend/app/(protected)/layout.tsx — server component
import { cookies } from "next/headers";
import { AuthProvider } from "@/components/auth/AuthProvider";

export default async function ProtectedLayout({ children }: { children: React.ReactNode }) {
  const cookieStore = await cookies();
  const authCookie = cookieStore.get("pep_session");
  if (!authCookie) redirect("/login");

  const me = await fetch(`${process.env.API_BASE_URL}/me`, {
    headers: { Cookie: `pep_session=${authCookie.value}` },
    cache: "no-store",
  }).then((r) => r.json() as Promise<MeResponse>);

  return <AuthProvider user={me}>{children}</AuthProvider>;
}
```

```tsx
// frontend/components/auth/AuthProvider.tsx — client
"use client";

import { createContext, useContext } from "react";
import type { MeResponse } from "@/lib/api/types";

const AuthContext = createContext<MeResponse | null>(null);

export function AuthProvider({ user, children }: { user: MeResponse; children: React.ReactNode }) {
  return <AuthContext.Provider value={user}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth used outside AuthProvider");
  return ctx;
}
```

Any client component under the protected layout can `useAuth()` and get the *display* identity without ever touching the token:

```tsx
// frontend/app/take/[testId]/TestRunner.tsx
"use client";
import { useAuth } from "@/components/auth/AuthProvider";

export function TestRunner({ session }: Props) {
  const { username } = useAuth();
  return (
    <div>
      <header className="text-sm text-muted-foreground">Logged in as {username}</header>
      ...
    </div>
  );
}
```

The `MeResponse` shape is deliberately narrow — `{ user_id, username, role }`. No token. No secrets. Even if someone screenshots the HTML or inspects the React tree in devtools, there's nothing dangerous to find.

## Wiring The Auth Cookie Into The Test-Taking Page Specifically

Putting it together for the Day 13 deliverable:

1. **Protected layout** (already exists from inherited auth, or you create it as part of Day 13 setup) reads `pep_session`, fetches `GET /me`, mounts `AuthProvider`.
2. **`/take/[testId]/page.tsx`** is a child of that layout. It reads `pep_session` again (cheap — `cookies()` is synchronous on cached store after first read) and forwards it on the server-side `POST /sessions`.
3. **`TestRunner`** (client) uses `useAuth()` for display info, uses `credentials: "include"` on its outbound fetches, and uses the `session.session_token` from props for `X-Session-Token`.

No token ever crosses the server/client boundary. No client code reads the cookie. The flow is end-to-end authenticated and the surface area for accidental leaks is small.

## Common Mistakes

- **Forgetting `await` on `cookies()`** in Next.js 16. Silently returns a Promise; `.get(...)` on a Promise is undefined and you redirect to login on every page load.
- **Forwarding the cookie as an `Authorization: Bearer` header instead of a `Cookie` header.** Different mechanism; the backend will read `Cookie` only.
- **Passing the auth token as a prop to a client component to "make it easier".** Now the token is in the HTML, in the React tree, in devtools, in any error report that includes a stack trace. Don't.
- **Omitting `credentials: "include"` on client fetches.** Request goes out cookie-less; 401 from backend; candidate sees an unhelpful error.
- **Wrapping every fetch in a custom client that takes a `token` argument.** You're recreating bearer auth on top of a cookie system; pick one. Cookies are the inherited choice.
- **Using `AuthContext` to store the token** in client state. The token is HTTP-only on purpose; don't synthesize a JS copy.

## Key Takeaways
- The inherited auth is an HTTP-only `pep_session` cookie; JavaScript cannot read it, but the browser auto-attaches it to same-origin requests.
- Server-side: `cookies()` from `next/headers` (async in Next.js 16) reads the cookie and forwards it as a `Cookie` header on outbound `fetch` calls.
- Client-side: `credentials: "include"` on `fetch` tells the browser to attach cookies on cross-origin calls — required for the local dev setup with separate ports.
- For display identity, fetch `GET /me` server-side in a protected layout and expose the result through a thin `AuthContext`; never expose the token itself.
- The test-taking page composes three credentials: the auth cookie (identity), the D11 session token (session-scoped), and the D12 idempotency key (per-submission) — each with a distinct purpose.

---
*Prerequisites: [01-nextjs-16-app-router-fundamentals.md](../day-09/01-nextjs-16-app-router-fundamentals.md), [07-opaque-token-generation-and-session-identifiers.md](../day-11/07-opaque-token-generation-and-session-identifiers.md), [03-idempotency-for-retried-mutations.md](../day-12/03-idempotency-for-retried-mutations.md), [03-server-components-for-initial-data-fetching.md](03-server-components-for-initial-data-fetching.md). Forward references: [06-submit-and-lock-ux-patterns.md](../day-14/06-submit-and-lock-ux-patterns.md).*
