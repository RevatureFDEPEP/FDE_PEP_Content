# Role-Based Access Enforcement In Next.js (Server-Side)

> *Day 19: Trainer Dashboard Slice (Frontend) + Polish — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D18 made the backend the single load-bearing authorization layer: `/reports/test/{id}` and `/reports/aggregate` are gated by a `TrainerDep` that decodes the JWT, reads the `role` claim, and 403s anyone who isn't a trainer. Today the frontend learns the same lesson at its own layer. The `/admin/*` routes that host the trainer dashboard must hard-block non-trainers on the server, before any client JavaScript boots, before any data fetch leaks, before the candidate ever sees the page chrome. The principle from D18 — *defense in depth* — reapplies: the backend is still authoritative, but the Next.js server layer enforces a second time so a candidate who guesses the URL gets a 403 page, not a flash of trainer UI. This file covers Next.js's two server-side enforcement points (middleware and per-page server components), the trade-offs between them, and the specific pattern PEP ships.

## Objective

Enforce role-based access on Next.js routes server-side so that non-trainers cannot reach `/admin/*` even by typing the URL directly.

## The Two Server-Side Hooks

Next.js App Router gives you two places to enforce auth on the server *before* a page renders:

1. **Middleware** (`middleware.ts` at the project root). Runs on the **edge** for every matching request before any route handler or page. Fast, but limited: it runs in the edge runtime (no Node APIs), and cannot easily call your Python backend's `/auth/whoami` because edge runtime + cold start + network hop is the wrong tool for "decode this JWT and check a claim."
2. **Per-page server component** (the default `page.tsx` inside `app/admin/...`). Runs in the Node.js runtime, can call your backend freely, can read cookies, can `redirect()` from `next/navigation`. Slower per request than middleware but vastly more capable.

The cohort should know both exist and the reason PEP picks one over the other.

## Trade-Offs

| Concern | Middleware | Server component |
|---|---|---|
| Runs before page assets stream | Yes | Yes |
| Can decode JWT (small lib, edge-compatible) | Yes (`jose`) | Yes |
| Can call backend `/auth/whoami` | Awkward (edge, cold) | Easy (Node fetch) |
| Granularity | Path pattern via `matcher` | Per-route, can branch on params |
| Failure UX | `NextResponse.redirect` or `.rewrite` | `redirect()` or `notFound()` |
| Centralization | One file gates all admin routes | Each admin page declares its own check |
| Latency cost | Sub-ms (no network) if JWT decode only | One network hop per admin page hit |

**PEP's call: middleware for the cheap path-level "is this user a trainer?" check, plus a per-page server-component check as defense in depth.** The middleware uses `jose` to verify the JWT signature and read the `role` claim — no backend call needed because the JWT is the source of truth (its signature was signed by the backend). The server-component check is belt-and-suspenders for the rare path where middleware is bypassed (e.g., a future API route mounted under `/admin/api/*`).

## The Middleware

```ts
// frontend/middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { jwtVerify } from "jose";

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

export const config = {
  matcher: ["/admin/:path*"],
};

export async function middleware(req: NextRequest) {
  const token = req.cookies.get("auth_token")?.value;

  if (!token) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("next", req.nextUrl.pathname);
    return NextResponse.redirect(url);
  }

  try {
    const { payload } = await jwtVerify(token, JWT_SECRET);
    if (payload.role !== "trainer") {
      const url = req.nextUrl.clone();
      url.pathname = "/403";
      return NextResponse.redirect(url);
    }
  } catch {
    // Tampered, expired, or unsigned token.
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  return NextResponse.next();
}
```

Three behaviors:

- **No token:** redirect to `/login?next=/admin/...` so a successful login lands them back where they tried to go.
- **Token but not trainer:** redirect to `/403`. A candidate guessed the URL; surface a friendly forbidden page, not a login loop.
- **Invalid token (signature failure, expired):** treat as "no token" and redirect to login. `jwtVerify` throws on any of these, the `catch` block handles all of them uniformly.

The `matcher` config limits middleware execution to `/admin/*` — middleware runs on every request that matches, so over-broad matchers add latency to every page load.

## The Per-Page Server Component

The middleware is the primary gate. The per-page check is paranoid layering:

```tsx
// frontend/app/admin/reports/page.tsx
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { decodeJwt } from "jose";

import { AdminReportsView } from "./AdminReportsView";

export default async function AdminReportsPage({
  searchParams,
}: {
  searchParams: { test_id?: string; from?: string; to?: string };
}) {
  const token = cookies().get("auth_token")?.value;
  if (!token) redirect("/login?next=/admin/reports");

  // decodeJwt does NOT verify the signature; that's the middleware's job.
  // Here we just need to read the role claim to short-circuit if middleware
  // was bypassed somehow.
  const claims = decodeJwt(token);
  if (claims.role !== "trainer") redirect("/403");

  return <AdminReportsView searchParams={searchParams} />;
}
```

Why decode and not verify? The middleware already verified the signature; this is a redundant claim read, not a security check. If the middleware was bypassed (it shouldn't be), the worst case is a candidate with a tampered JWT — but D18's backend will still 403 the data fetch, so the page renders empty. The server-component check is about *clean UX on a bypass*, not about being the last line of defense.

## Why Both, Not Just One

A pure-middleware design works until somebody adds a new admin route and forgets to extend the `matcher` glob. A pure-per-page design works until somebody forgets to add the check at the top of a new admin page. Together, the two layers fail-safe: a missed matcher is caught by the page; a missed page check is caught by the matcher. Defense in depth applied to *frontend* defense in depth.

The cost is ~10 lines of duplicated logic. Worth it for a polish-day investment.

## Redirecting Vs Returning 403

Two reasonable failure modes:

1. **Redirect** the candidate to `/403` (the choice above). The URL bar updates, the back button works sensibly, the page is bookmarkable as "I'm forbidden."
2. **Rewrite** to `/403` (keep the URL `/admin/reports` but render the forbidden page). Less surprising for the user — they don't see a URL change — but they can refresh and hit the check again, which is fine.

Either is defensible. PEP picks redirect because the back button behavior is cleaner and the `/403` URL is easier to reason about in logs. Whatever the cohort picks, pick it consistently across all admin routes.

The `/403` page itself is just a server component under `app/403/page.tsx` with a friendly "you don't have access to this page" copy and a link back to `/results` (the candidate's expected landing).

## What About `/login` Loops?

A common pitfall: a candidate is logged in but somehow ends up at `/login?next=/admin/reports`. After login, they're sent to `/admin/reports`, middleware checks role, candidate is not a trainer, redirect to `/403`. They click "go home," land on `/results`, all good. The `next` parameter must be **validated** on the login page — never blindly redirect to an arbitrary URL from `next`, or you've built an open redirect that lets phishers point `next` at evil.com. Allowlist next values to internal paths only.

```ts
const next = searchParams.next;
const safeNext = next && next.startsWith("/") && !next.startsWith("//")
  ? next
  : "/results";
```

Small detail; easy to miss on the polish day.

## How To Test It

Three manual tests the cohort should run before checking the box on the deliverable:

1. **Candidate hits `/admin/reports` directly in the URL bar.** Expected: redirect to `/403`. They never see trainer chrome.
2. **Logged-out user hits `/admin/reports`.** Expected: redirect to `/login?next=/admin/reports`. After login as a trainer, lands back on the admin page.
3. **Trainer hits `/admin/reports`.** Expected: page loads, data renders, no extra redirects.

Add one unit test for the middleware that mocks a `NextRequest` with each role and asserts the response — but the manual tests are what catch the UX papercuts.

## Anti-Patterns

- **Relying on `useEffect` to redirect non-trainers.** The page renders, candidate sees the trainer UI for 200ms, *then* the redirect fires. That's a leak — a fast screenshot captures the dashboard. Server-side enforcement renders nothing until the role is verified.
- **Checking role in `getServerSideProps` (Pages Router) but mixing App Router elsewhere.** PEP is App Router throughout; using mixed routing patterns confuses the cohort.
- **Decoding the JWT in middleware without verifying the signature.** `decodeJwt` reads claims of any string; only `jwtVerify` checks the signature. In middleware, *verify*. (In the redundant per-page check, decode is fine because middleware already verified.)
- **Putting `JWT_SECRET` in a client-side env var (`NEXT_PUBLIC_*`).** Then it's shipped to every browser and signing forgeries is trivial. Server-only env vars.
- **Returning a 403 status from `redirect()`.** `redirect()` is always a 307; setting an explicit 403 page is fine, but don't expect the redirect itself to be a 403.
- **Using `redirect()` from a try block.** `redirect()` throws a special exception that Next.js catches; wrapping it in your own try/catch swallows the redirect. Let it bubble.

## Connecting Back To D18

D18's principle: backend is authoritative even when the frontend cooperates. Today's middleware is the frontend *cooperating well* — it shouldn't leak admin UI to candidates. But the candidate who bypasses the middleware (curl, an old browser, a misconfigured matcher) still hits D18's `TrainerDep` and gets 403'd from the data. Two layers, both doing their part, neither alone sufficient.

The mental model: middleware = UX gate; backend = security gate. Both are required for a well-built app, but they answer different questions.

## Key Takeaways

- Server-side enforcement of `/admin/*` means middleware and/or server-component checks — never `useEffect`, which leaks the page for a frame.
- Middleware on the edge with `jose` to verify the JWT is the right primary tool. Per-page server-component checks are belt-and-suspenders for missed matchers.
- The `matcher` config scopes middleware to the routes that need it; broader scopes add latency to every page.
- Redirect non-trainers to `/403`, logged-out users to `/login?next=...`, validating `next` against open-redirect abuse.
- Backend gates from D18 remain the load-bearing layer. Middleware is UX hardening, not a substitute for backend authorization.

---
*Prerequisites: [06-authentication-context-propagation-through-the-component-tree.md](../day-13/06-authentication-context-propagation-through-the-component-tree.md), [05-jwt-claim-verification-and-role-enforcement.md](../day-18/05-jwt-claim-verification-and-role-enforcement.md), [06-defense-in-depth-security-patterns.md](../day-18/06-defense-in-depth-security-patterns.md).*
