# CORS Configuration For Browser-To-API Integration

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

CORS — Cross-Origin Resource Sharing — is the browser's "before-I-let-this-page-call-that-API, the API has to *say* it's okay" mechanism. The substrate's D6 reverse proxy was set up specifically so the frontend and all backend services appear on the *same origin* in production-like usage, which makes CORS a non-event. But local dev sometimes runs the Next.js dev server on `localhost:3000` while hitting reporting on `localhost:8004` directly, and that *is* cross-origin, and the cohort will hit a CORS error and freeze up if they haven't seen one before. Today's topic teaches the conceptual model, the FastAPI middleware that solves the dev case, and — critically — what *not* to do (the `*` everywhere reflex that papers over the issue and creates security problems).

## What CORS Actually Is

CORS is a **browser security policy**, not a server security feature. A page loaded from `https://app.example.com` is, by default, *not allowed* by the browser to make `fetch` requests to `https://api.example.com` — the origins differ (different subdomain counts as different origin). The browser enforces this regardless of what the server does.

CORS is the protocol that lets the server *opt in* to allowing those cross-origin requests. The server says, via response headers:

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Authorization, Content-Type, X-Request-Id
```

The browser reads those headers and decides "okay, the server consents — I'll let the JavaScript code see the response." Without the right headers, the browser blocks the response from being visible to JS even though the request and response physically traveled.

**The server does not protect itself by setting CORS headers.** The server protects itself with authentication, authorization, rate-limiting. CORS is consent for the *browser's* benefit. A non-browser client (curl, Postman, a Python script) ignores CORS entirely — it'll happily call any endpoint with any origin. So:

- "We need CORS to be secure" — wrong framing. You need *auth* to be secure.
- "We need CORS to allow our frontend to call us" — correct framing.

The cohort needs to internalize this distinction. CORS errors look like security errors and people respond by either disabling all security (the `*` mistake) or by re-architecting their app to avoid them. Neither is the right move when you understand what CORS actually is.

## Same-Origin Via Reverse Proxy — The PEP Default

In the production-like Compose setup (D6), the reverse proxy fronts everything:

```
Browser → http://localhost   (single origin)
  /                → frontend (Next.js)
  /api/users/*     → user-service
  /api/questions/* → question-management
  /api/tests/*     → test-management
  /api/reports/*   → reporting-service
  /api/gateway/*   → api-gateway
```

All requests are same-origin (`http://localhost`). CORS does not apply. This is *the reason* the reverse proxy was set up the way it was — not just routing convenience, but origin unification. The cohort should be able to explain that on D20.

So in production-like usage, you don't need CORS at all.

## Where CORS Bites: Local Dev With Two Ports

The case where CORS *does* matter is local dev when the cohort runs the Next.js dev server outside Compose for fast hot-reload:

```bash
# Terminal 1: backend stack
docker compose up

# Terminal 2: frontend dev server, hot-reloading
cd frontend && npm run dev    # listens on :3000
```

Now the browser is on `http://localhost:3000` (the dev server) and the reporting API is on `http://localhost:8004` (Compose port-mapped) or `http://localhost/api/reports/...` (reverse proxy). The first is cross-origin (different port), the second is same-origin. The cohort will inevitably point `fetch` at the first and hit:

```
Access to fetch at 'http://localhost:8004/reports/user/u_42' from origin
'http://localhost:3000' has been blocked by CORS policy: No
'Access-Control-Allow-Origin' header is present on the requested resource.
```

This is the moment to teach the middleware.

## FastAPI CORS Middleware

FastAPI ships `CORSMiddleware` from Starlette. Wire it in `main.py` early in the middleware stack:

```python
# reporting-service/app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings

app = FastAPI(title="reporting-service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,        # list of exact origins, no '*'
    allow_credentials=True,                     # needed for cookies / Authorization
    allow_methods=["GET", "POST", "OPTIONS"],   # explicit, not '*'
    allow_headers=["Authorization", "Content-Type", "X-Request-Id"],
    expose_headers=["X-Request-Id"],            # so frontend code can read it
    max_age=600,                                # cache preflight 10 minutes
)

# Then your routes
app.include_router(reports_router)
```

`settings.CORS_ORIGINS` is read from env, typically:

```python
# config.py
class Settings(BaseSettings):
    CORS_ORIGINS: list[str] = ["http://localhost:3000", "http://localhost"]
```

In production-like Compose, this still doesn't matter (same-origin via proxy), but having the right `localhost:3000` entry makes hybrid-mode dev work.

### What Each Parameter Does

- **`allow_origins`** — explicit list. Browsers send their `Origin` header on every cross-origin request; the middleware echoes the value back as `Access-Control-Allow-Origin` *only if* it's in the allowlist. Other origins get blocked.
- **`allow_credentials=True`** — if your frontend sends cookies or `Authorization` headers, you need this *and* you cannot use `allow_origins=["*"]` (the spec disallows credentials with wildcard origin). The cohort will hit this exact combination and get confused; the answer is: enumerate origins.
- **`allow_methods`** — list the HTTP methods the API actually serves. `OPTIONS` is required (for preflight). For reporting, `GET` and `OPTIONS`; for write services, add `POST`/`PUT`/`DELETE`.
- **`allow_headers`** — headers the client may send. `Content-Type` for JSON bodies, `Authorization` for auth, `X-Request-Id` because the D10 correlation pattern requires the frontend to send the request id through. Without it on the allowlist, the browser strips it from cross-origin requests.
- **`expose_headers`** — by default the browser hides *response* headers from JS for cross-origin responses, except for a CORS-safelist. To let the frontend read `X-Request-Id` from the response (useful for displaying it in error messages — "tell support request id X"), expose it.
- **`max_age=600`** — preflight (the `OPTIONS` request the browser sends before some real requests) is cached for 10 minutes. Without it, the browser preflight-requests every cross-origin call, doubling round trips. Don't go crazy here (some browsers cap at 2 hours anyway), but 10 minutes is sane.

## Preflight, Briefly

The cohort will hear about "preflight requests" and should know what they are. For "simple" requests (`GET` or `POST` with text/plain bodies, no custom headers), the browser just sends the request. For anything else (`PUT`, `DELETE`, `Authorization` header, `Content-Type: application/json`, custom headers like `X-Request-Id`), the browser first sends an `OPTIONS` request asking for permission:

```
OPTIONS /reports/user/u_42 HTTP/1.1
Origin: http://localhost:3000
Access-Control-Request-Method: GET
Access-Control-Request-Headers: x-request-id
```

The server responds with what it allows; only if the browser is satisfied does the real request follow. This is why `allow_methods=["OPTIONS"]` matters and why `max_age` matters (cache the preflight to avoid doing it every time).

In the browser network tab, the cohort will see `OPTIONS` requests immediately before their real `GET` requests; they should recognize what they are rather than mistaking them for application traffic.

## The "`*` Everywhere" Anti-Pattern

The cohort's first instinct when hitting a CORS error is `allow_origins=["*"]`. Avoid this:

- **It can't coexist with `allow_credentials=True`** (spec rejects the combination).
- **It says "any website can call this API and get the response visible to its JS."** For a public read-only API that's fine; for an authenticated reporting endpoint it means a malicious page can trigger reads on behalf of an authenticated user. Combined with credentials disabled this is *less* bad (no cookies sent), but still wrong as a posture.
- **It hides genuine misconfiguration.** If you allowlist explicit origins and a deploy moves the frontend to a new origin, you get a CORS error in dev and fix it. If you allow `*`, you don't, and then you ship a security gap to prod.

For PEP, `allow_origins=["http://localhost:3000", "http://localhost"]` covers dev and production-like Compose. If the cohort is asked "what about staging?" — add the staging origin too, explicitly. Never `*`.

## Debugging CORS Errors

When the cohort hits a CORS error, the recipe:

1. **Read the actual error.** Browsers print specific reasons: "no `Access-Control-Allow-Origin`," "credentials flag is true but `Access-Control-Allow-Origin` is `*`," "method `PUT` is not in `Access-Control-Allow-Methods`." The message tells you exactly what's wrong.
2. **Check the request is actually cross-origin.** Same protocol, host, *and port* — all three must match for same-origin. `http://localhost:3000` and `http://localhost:8004` differ by port and are cross-origin. `http://localhost` and `http://127.0.0.1` differ by host and are cross-origin (yes, even though they resolve to the same IP).
3. **Look at the preflight in the network tab.** If the `OPTIONS` request returns the wrong headers, the real request never fires. Open the `OPTIONS` request, look at its response headers, identify what's missing.
4. **Test with curl to confirm the server side is working.** A successful `curl -i http://localhost:8004/reports/user/u_42 -H 'Origin: http://localhost:3000'` should produce a response with `Access-Control-Allow-Origin: http://localhost:3000` in the headers. If it doesn't, the middleware is misconfigured.
5. **Compare against a working endpoint.** If user-service's CORS works and reporting's doesn't, diff the middleware configs.

The most common cohort errors:

- Forgot to add `CORSMiddleware` at all.
- Added it *after* the routes (FastAPI middleware order matters; CORS should be early).
- `allow_headers` missing `X-Request-Id` so the frontend's request-id header strips off.
- `allow_credentials=True` with `allow_origins=["*"]` — invalid combination.
- Frontend `fetch` not passing `credentials: "include"` when the API uses cookies.

## A Note On The Frontend Side

The browser is half the protocol. The frontend `fetch` for a cross-origin request that needs cookies looks like:

```js
const res = await fetch("http://localhost:8004/reports/user/u_42", {
  credentials: "include",
  headers: {
    "X-Request-Id": crypto.randomUUID(),
  },
});
```

Without `credentials: "include"`, the browser doesn't send cookies on cross-origin requests (a separate guard from CORS). With it, the *server* must respond with `Access-Control-Allow-Credentials: true` and a specific origin (not `*`). The pair of settings must agree on both sides.

## Production Posture

For PEP and the production-like Compose deploy, the recommended posture:

- **Same-origin in production:** route through the reverse proxy. CORS is irrelevant.
- **Same-origin in production-like local stack:** route through the reverse proxy on `localhost`. CORS is irrelevant.
- **CORS configured for hybrid local dev** (Next.js dev server on `:3000` hitting Compose-exposed backend ports): explicit allowlist, credentials enabled, narrow methods/headers.
- **`*` never appears in the codebase.** It's a smell. If you see it in a PR, ask why.

The trainer should grep the repo for `allow_origins=["*"]` during PR review and fail any PR that introduces it without justification.

## Anti-Patterns

- **`allow_origins=["*"]` "to make it work."** Hides misconfiguration; incompatible with credentials; bad security posture. Always enumerate origins.
- **Treating CORS errors as a server-side bug to "fix" with auth changes.** CORS is a browser policy; auth changes don't help. Read the error, fix the middleware.
- **Adding `CORSMiddleware` after `app.include_router(...)`.** Middleware applies in registration order; CORS must be registered before routes (and especially before any auth middleware that might 401 before CORS headers are added).
- **Forgetting `X-Request-Id` in `allow_headers`.** The frontend strips the header on send; D10's correlation pattern silently breaks for cross-origin requests.
- **Mismatched `credentials` between frontend and backend.** `fetch` without `credentials: 'include'` *plus* server `Access-Control-Allow-Credentials: true` — fine. `fetch` *with* credentials *plus* server `allow_origins=['*']` — broken. The pair must agree.
- **Letting CORS leak into integration tests.** Tests run with httpx (server-to-server), which ignores CORS. If a feature works in tests but breaks in the browser, suspect CORS first.
- **Believing CORS authenticates.** It doesn't. A motivated attacker uses curl or a script; CORS is bypassed in one step. Auth + authz + rate-limiting are the security layer.
- **Production deploy that requires CORS to be on at all.** If the prod deploy goes same-origin via proxy, you don't need CORS in prod. Disable it there (or leave a `localhost:3000`-only dev config) and reduce attack surface.

## Key Takeaways

- CORS is a browser-side consent mechanism, not a server security feature; you configure it so legitimate cross-origin browsers can call your API.
- PEP's production-like setup goes through the D6 reverse proxy, making everything same-origin — CORS is not needed there.
- The hybrid local-dev case (Next.js on `:3000`, backend on Compose ports) is where CORS matters; FastAPI's `CORSMiddleware` with an explicit allowlist solves it.
- Never use `allow_origins=["*"]`; it can't coexist with credentials, it hides misconfiguration, and it's a poor security posture.
- Always include `X-Request-Id` in `allow_headers` so the D10 correlation chain survives cross-origin calls.

---
*Prerequisites: [03-reverse-proxy-fundamentals-nginx-traefik.md](../day-06/03-reverse-proxy-fundamentals-nginx-traefik.md), [02-fastapi-routing-and-dependency-injection-patterns.md](../day-11/02-fastapi-routing-and-dependency-injection-patterns.md), [04-client-components-for-stateful-interactivity.md](../day-13/04-client-components-for-stateful-interactivity.md).*
