# Client-Side Route Guards As UX (Not Security)

> *Day 19: Trainer Dashboard Slice (Frontend) + Polish — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Topic 1 built the server-side gate on `/admin/*` — middleware decodes the JWT, checks the role claim, redirects non-trainers. That's the load-bearing layer for what the browser is *allowed* to render. This topic is about the *client-side* complement: hiding the "Dashboard" link from the sidebar when the logged-in user is a candidate, dimming or removing UI affordances that wouldn't work for them, and generally not advertising the existence of trainer features to people who can't use them. Critically, **none of this is security**. D18's defense-in-depth principle said it: the backend is the only thing standing between a request and a resource. Client-side guards are about *cleanliness* — a candidate who never sees the "Dashboard" link won't click it, won't get a 403, won't have an awkward UX moment. That's a UX win, not a security win. This file explains the distinction explicitly because it's an interview question the cohort *will* be asked, and gives the patterns to implement client guards correctly.

## Objective

Add client-side route guards that hide trainer UI from non-trainers, and articulate clearly why these guards are *not* the security layer.

## The Two Things The Frontend Is Doing

After Topic 1 and this topic, the frontend has two distinct role-aware behaviors:

1. **Server-side hard block (Topic 1).** Middleware + per-page check. Non-trainers cannot reach `/admin/*`. This is *correctness*: even if the link is hidden, somebody might type the URL, paste a bookmark, or follow a stale tab.
2. **Client-side UX hiding (this topic).** Conditional rendering of the "Dashboard" sidebar link, the "Admin" tab, the "View All Reports" button on the homepage, etc. Non-trainers never see them. This is *polish*: don't tease features users can't use.

The two are independent: the server gate stands even if every client guard is removed; the client guards work as polish even if the server gate were somehow bypassed (it isn't, but the layering doesn't depend on that).

## The Pattern: A `useRole` Hook + Conditional Rendering

The auth context from D13 already exposes the logged-in user. Add a tiny convenience hook on top:

```tsx
// frontend/lib/auth/useRole.ts
"use client";

import { useAuth } from "./AuthContext";

export function useRole() {
  const { user } = useAuth();
  return {
    role: user?.role ?? null,
    isTrainer: user?.role === "trainer",
    isCandidate: user?.role === "candidate",
    isAuthed: user != null,
  };
}
```

Use it in the sidebar component:

```tsx
// frontend/components/Sidebar.tsx
"use client";

import Link from "next/link";
import { useRole } from "@/lib/auth/useRole";

export function Sidebar() {
  const { isTrainer, isAuthed } = useRole();

  return (
    <nav className="flex flex-col gap-2 p-4">
      <Link href="/">Home</Link>
      {isAuthed && <Link href="/results">My Results</Link>}
      {isTrainer && (
        <Link href="/admin/reports" data-testid="admin-link">
          Dashboard
        </Link>
      )}
    </nav>
  );
}
```

A candidate sees Home + My Results. A trainer sees Home + My Results + Dashboard. A logged-out user sees Home. No menu items "almost work" — they're either present and functional or absent.

## Why This Is UX, Not Security

The frontend bundle is shipped to the browser. The candidate can:

- Open devtools and edit the `isTrainer` value in React state to force the Dashboard link to appear.
- Read the bundle and find `/admin/reports`, then type the URL directly.
- Inspect the JS source and discover every admin route name.
- Run the conditional logic past a browser extension that mocks `useRole()` to return `isTrainer: true`.

None of these *gets* the candidate into the admin dashboard, because of Topic 1's middleware and D18's backend gate. But they trivially defeat the client-side guard *itself*. The client guard is a courtesy: a candidate operating in good faith doesn't see something they can't use. The candidate operating in bad faith bumps into the server gates the moment they try.

A useful framing for the cohort: **the client guard prevents accidents, the server gate prevents incidents.**

## The Interview Question

A frequent interview prompt for FDE candidates: "Walk me through how you'd protect an admin route in a React app." The wrong answer is "I'd add a `<ProtectedRoute>` wrapper that redirects if the user isn't an admin." The right answer is "I'd enforce auth on the server — middleware or per-page check — and *additionally* hide admin UI from non-admins on the client for UX. The server is the security layer; the client guard is cleanliness."

A candidate who answers with only the client guard has missed the point. A candidate who answers with only the server gate has missed the UX. Both layers, with a clear vocabulary for which does what, is the senior answer.

The cohort should be able to say, in their own words:

> "Client-side guards hide UI from users who can't use it. They are not security because the bundle is in the user's browser and they can modify what runs. Security lives on the server, where the user cannot modify the code path."

That sentence, internalized, is the interview-readiness deliverable for this topic.

## The Componentized Version

For more than one admin link, factor the gate out:

```tsx
// frontend/components/auth/RequireRole.tsx
"use client";

import { ReactNode } from "react";
import { useRole } from "@/lib/auth/useRole";

type Props = {
  role: "trainer" | "candidate";
  children: ReactNode;
  fallback?: ReactNode;
};

export function RequireRole({ role, children, fallback = null }: Props) {
  const { role: currentRole } = useRole();
  if (currentRole !== role) return <>{fallback}</>;
  return <>{children}</>;
}
```

Used in the sidebar:

```tsx
<RequireRole role="trainer">
  <Link href="/admin/reports">Dashboard</Link>
</RequireRole>
```

The wrapper name is honest: `RequireRole` not `ProtectRoute`. "Require" makes the intent (filter rendering) explicit; "Protect" implies security and would mislead a future reader.

A consistent vocabulary helps: in code reviews on PEP, "this isn't security" should be a phrase the cohort uses unprompted when reviewing client guards.

## What About `redirect()` From The Client?

The wrong pattern, but a common one:

```tsx
// BAD: client-side redirect as the "guard"
"use client";
import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useRole } from "@/lib/auth/useRole";

export default function AdminPage() {
  const { isTrainer } = useRole();
  const router = useRouter();
  useEffect(() => {
    if (!isTrainer) router.replace("/403");
  }, [isTrainer, router]);

  return <AdminContent />; // renders for one frame before the redirect fires
}
```

Three problems:

1. **Flash of admin content.** The first paint renders `<AdminContent />`; the `useEffect` fires after, redirect happens, candidate has seen one frame of trainer UI.
2. **The data fetch may have already started.** If `<AdminContent />` server-fetches on mount, a candidate's browser made the request — albeit one that the backend 403s.
3. **It's still UX, not security.** Even done correctly, a candidate with devtools paused on the first render reads the DOM.

The right pattern is Topic 1's server-side redirect. Client-side `useEffect` redirects are at best a fallback, at worst a leak.

## Logged-Out State

A candidate who isn't logged in shouldn't see "My Results" *or* "Dashboard." The `useRole` hook handles this naturally — `isAuthed` is false when there's no user — but the cohort should explicitly think about three states:

1. **Logged out:** Home link only. Login button.
2. **Candidate:** Home + My Results.
3. **Trainer:** Home + My Results + Dashboard.

A test for each, or at least a manual click-through, is part of the deliverable.

## The Loading Beat

The `useAuth` context populates after a "whoami" call on mount. For one tick, `user` is null and `isAuthed` is false even for a logged-in user. Two reasonable strategies:

1. **Show nothing for nav links during loading.** Layout shifts when the user resolves; ugly but safe.
2. **Show a skeleton.** Reserve the space for the nav, render gray placeholders, swap in real links when the user resolves. Better UX.

PEP picks option 2 with a 200ms minimum so a fast network doesn't *also* layout-shift. This is small but real polish work.

## Anti-Patterns

- **Calling the wrapper `ProtectedRoute`.** Implies security; misleads readers. Use `RequireRole`, `WithRole`, or `ShowIfTrainer`.
- **Reading the role from `localStorage` directly in each component.** Reads scatter, role updates don't propagate, you've built a stale-state machine. Use the context-backed hook.
- **Checking `user.email.endsWith("@revature.com")` as a proxy for trainer.** Implicit role inference. Use the explicit `role` claim from D18's JWT.
- **Hardcoding the role list in multiple places.** If trainer changes to admin, you grep-and-pray. Define the role enum once and import it.
- **Putting client guards on data fetches as the only protection.** The data fetch hits the backend; the backend already 403s candidates. Client guards on fetches are pointless if the route is already gated server-side.
- **Telling the candidate "you're not authorized" via a console.log.** They will see it. Either show nothing or show a friendly UI message.

## Connecting Back To D18

D18's principle in one line: **frontend security is UX, not security.** Today the frontend builds exactly that UX, and the cohort can now explain why it's the correct layer to build it at (it's polish, the security lives on the server) and why nobody should mistake it for the load-bearing layer (the bundle is in the user's browser).

The mental model the cohort should leave with:

- Topic 1 of D19: server-side enforcement = security + UX gate.
- This topic: client-side hiding = UX only.
- D18: backend authorization = security.

Three layers. The backend is load-bearing. The Next.js server is a UX gate that happens to also harden against bypass. The client guards are pure UX. Saying that out loud, in interview-ready prose, is the deliverable for this topic.

## Key Takeaways

- Client-side route guards hide UI from users who can't use it. They are UX, not security.
- The frontend bundle is in the user's browser; any client guard can be modified by the user. That's why security lives on the server.
- Pattern: a `useRole` hook backed by the auth context, used directly or via a `RequireRole` wrapper for conditional rendering.
- `useEffect`-based redirects on the client are a leak (flash of admin content) and should not be the primary guard; server-side enforcement (Topic 1) is the primary.
- Vocabulary matters: call wrappers `RequireRole`, not `ProtectedRoute`, so future readers don't mistake UX for security.
- Connect the three layers in interviews: backend authorizes (D18), Next.js server gates (Topic 1), client hides UI (this topic). Each does its part; only the backend is load-bearing.

---
*Prerequisites: day-13-authentication-context-propagation-through-the-component-tree, day-18-defense-in-depth-security-patterns, day-19-role-based-access-enforcement-in-nextjs-server-side.*
