# Defense-In-Depth Security Patterns — Backend Authoritative Even When The Frontend Cooperates

> *Day 18: Trainer Dashboard Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D11 established the **server-authoritative** principle for *state*: the server owns time, expiry, and the canonical answer to "is this still valid?" Today that same principle, sharpened to authorization, becomes the second pillar of the cohort's security thinking. The trainer dashboard frontend on D19 will hide `/admin/*` routes from candidates, gray out the menu link, and route candidates straight back to `/results`. That UX work is real and worth doing — but **none of it is security**. A candidate with curl, devtools, or a homemade client can call `GET /reports/aggregate` directly. If the backend isn't itself enforcing the trainer-only gate, the dashboard is open. This file is the argument for why the gate must live on the server independently of any frontend cooperation, and what that looks like in practice for the FDE PEP brownfield variant.

## The Principle In One Sentence

Frontend security is **UX**, not security. The backend is the only thing standing between a request and a resource, and so the backend must enforce every authorization rule that matters — *regardless of whether the frontend was supposed to prevent the request in the first place*.

## The Argument By Counter-Example

A candidate finishes their assessment, clicks around, and notices the trainer's dashboard exists. They can't see the menu link — the frontend's `if (user.role !== "trainer") return null` hid it. So far so good. Now:

- They open devtools, find the API URL the trainer-side fetch would have hit (`/reports/aggregate`), and run it from the JS console.
- Or they pop `curl -H "Authorization: Bearer <their-jwt>" http://api/reports/aggregate` from the terminal — their token is sitting in a cookie or localStorage.
- Or they install a browser extension that flips a feature flag the frontend uses to render trainer UI.
- Or they read the JS bundle, find `/api/reports/aggregate`, and just call it from Postman.

If the *only* gate is the frontend's "should I render this link?", every path above succeeds. The aggregate report — names, pass rates, who's struggling — comes back to a candidate who is not authorized to see it. That is the breach, and it is one curl command away whenever the backend trusts the frontend's cooperation.

## What Makes This Different From D11

D11 was about *state*: the client cannot be the source of truth for "what time is it?" because clocks drift, devtools edit values, and tabs suspend. Today is about *authority*: the client cannot be the source of truth for "am I allowed?" because anyone can write a different client. The principle generalizes:

> Any value that determines whether the server takes an action — time, identity, role, ownership, expiry — must be computed or verified on the server.

State authority (D11) and authorization authority (today) are two specific applications of the same idea. The cohort should see them as one mental model with two faces, not two unrelated topics.

## What The Frontend Does, And What It Doesn't

Defense-in-depth doesn't mean "the frontend does nothing." It means the frontend's checks are *complementary*, not *load-bearing*:

| Layer | What it does | What it does NOT do |
|---|---|---|
| **Frontend route guards** (D19) | Hide admin menu from candidates; redirect to `/login` on missing token; render different chrome for trainers | Prevent unauthorized API access |
| **Network layer** (CORS, CSP) | Limit which origins can call the API from a browser; reduce XSS blast radius | Stop curl, Postman, or non-browser clients |
| **API gateway / reverse proxy** (D6) | Rate limit, terminate TLS, route by path | Enforce business-layer authorization (PEP design) |
| **Backend route dependencies** (today) | Verify JWT signature, check role claim, enforce ownership | Hide UI elements from unauthorized users |

The backend dependency is the **single load-bearing layer for authorization**. Everything else is UX, hygiene, or hardening — useful, but redundant if the backend is enforcing correctly. Conversely, if the backend is *not* enforcing correctly, no amount of frontend polish substitutes.

A useful test: if you removed every line of frontend auth logic right now, would your data be safe? If yes, the backend is doing its job. If no, you've delegated security to UX and there is a 5-minute curl exploit available.

## How The Aggregate Endpoint Looks Today

After topics 3 and 4, the aggregate endpoint is:

```python
@router.get("/aggregate", response_model=AggregateReport)
async def aggregate(
    user: TrainerDep,          # JWT decoded; role checked; 401 or 403 if not trainer.
    params: AttemptListDep,
    db: DBSession,
):
    return await reports_service.aggregate(db, params)
```

The frontend on D19 will:

- Not render the "Dashboard" link in the sidebar for candidates.
- Redirect candidates landing on `/admin/dashboard` to `/results`.
- Show a friendly "you don't have access" page if a 403 leaks through.

All three are pure UX. The candidate who bypasses them hits `/reports/aggregate` with their candidate JWT and gets a 403 from the backend, not the data.

## Defense In Depth In The Other Direction: Server Filters Even For Trainers

The principle has a second corollary the cohort should internalize. Even within a trusted role, the server should not assume the client is asking *only* for the data it's allowed to see.

Example: a future feature lets each trainer see *only their own cohort's* aggregate. If the frontend filters by cohort_id in the query string and the backend trusts the query string, a malicious-but-authenticated trainer could pass another trainer's cohort_id and read it. The backend must look up the trainer's cohort_id from a server-side mapping and filter regardless of what the client requested. The query string is a *suggestion*; the server is the authority.

PEP doesn't ship cohort-scoped trainers (every trainer sees everything), but the principle should be in the cohort's vocabulary. Whenever a value comes from the client and feeds into authorization, the server *re-derives* it from authoritative state.

## Idempotency And Replay

A related discipline: even authenticated requests should be safe to replay. A trainer dashboard refresh that triggers `GET /reports/aggregate` ten times should not have side effects ten times. GETs are naturally idempotent — they read, they don't write — so this is mostly a vocabulary point on D18, but it's a property the cohort will need on D20 capstone POSTs.

If a future endpoint *does* mutate state, it needs an `Idempotency-Key` header pattern or a server-side replay guard. PEP doesn't ship this today; the cohort should know it exists as a pattern.

## CSRF, CORS, And Bearer Tokens

A few words on adjacent concerns the cohort will hear about in interviews:

- **CSRF (Cross-Site Request Forgery).** Concerns cookie-based authentication where a malicious site can cause a victim's browser to send the cookie automatically. **Bearer-token auth (PEP) is not vulnerable to classic CSRF** because the token must be attached programmatically by JS that can read it — a malicious site can't read tokens from a different origin's localStorage. If PEP later moves to cookie auth, CSRF becomes relevant; add `SameSite=Lax` and CSRF tokens.
- **CORS (Cross-Origin Resource Sharing).** The api-gateway from D6 configures which origins can call the API from a browser. CORS is a browser hygiene mechanism; it does not stop curl. Useful, not load-bearing.
- **CSP (Content Security Policy).** Limits which scripts the frontend can load — reduces XSS blast radius. Adjacent to authorization; not a substitute.

The cohort should be able to articulate which of these are *security* and which are *defense in depth*. CORS, CSP, CSRF mitigations: defense-in-depth layers. Backend authorization on the route: security.

## Rate Limiting And Audit Logging

Two operational habits that round out the defense:

- **Rate limit authentication failures.** If `get_current_user` is raising 401 for the same source IP 100 times a minute, somebody is brute-forcing tokens. The api-gateway should rate-limit on failure response code. PEP defers explicit rate limiting to Phase 2 but logs the count.
- **Audit log authorization denials.** Topic 3 already mentioned this; restating because it's the bridge between "we enforce auth" and "we *know* auth was challenged." When the trainer's manager asks "did anyone try to access the dashboard yesterday?", `authz_denied user=u_123 role=candidate route=/reports/aggregate` in the logs is the answer.

## The Phrase To Internalize

A line worth burning into the cohort:

> **Treat every request as if it came from `curl` — because it might have.**

If the answer to "what stops a candidate from getting this data?" requires the assumption "they are using our React frontend," the answer is wrong. The right answer is "the backend checks the JWT role on every request."

## Anti-Patterns

- **Frontend-only role checks.** "We hide the link, so candidates can't access it." Five minutes with curl says otherwise.
- **Reading the role from a request header.** `X-User-Role` set by the frontend is client-controllable; only the verified JWT claim is trustworthy.
- **Soft-allowlisting via JS bundles.** Putting "admin" routes behind a code-split chunk that "candidates won't download" doesn't prevent access — the bundle URL is in the network tab. Code splitting is performance, not security.
- **Treating the api-gateway as the sole auth layer.** If the gateway decodes the JWT and downstream services trust a header, the services are *one network misconfiguration* away from being reachable directly with forged headers.
- **`debug=True` in production.** FastAPI's debug responses leak stack traces with code paths. Always off in non-dev.
- **CORS allowlist of `*` with credentials.** Browsers won't actually allow this (the standard forbids credentials + wildcard), but the cohort tries it and is confused. Be explicit.
- **Returning 200 with `{"error": "forbidden"}` instead of 403.** Confuses clients and breaks the audit-log signal — 403 in access logs is meaningful, 200 isn't.

## Key Takeaways

- Frontend security is UX; backend authorization is security. Hiding a link does not gate the API.
- Every request should be assumed to have come from curl. If the answer to "why is this safe?" requires a cooperating frontend, the answer is wrong.
- Defense-in-depth means *multiple* layers (UX guards, CORS, gateway, route deps) all do their part — but the route dependency is the single load-bearing layer.
- D11's server-authority principle generalizes: any value the server's decision depends on (time, identity, role, ownership) must be computed or verified on the server, never accepted from the client.
- Even within a trusted role, server-side filtering matters: derive the trainer's scope from server state, not the query string.
- Log every authorization denial. Without audit signal, you can't tell whether the gate has been challenged.

---
*Prerequisites: day-11-server-authoritative-state, day-18-role-based-authorization-at-the-api-layer, day-18-jwt-claim-verification-and-role-enforcement.*
