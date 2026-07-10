# Microservices Architecture and Service Boundaries

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

## Overview

The `rev-eval` substrate you have just cloned is not a monolith — it is a small constellation of independent services that talk to one another over HTTP. Before you can read its code, run its stack, or land changes in it, you need a conceptual map: which services exist, what each one is responsible for, and where the *boundaries* between them sit. These boundaries will constrain everything you do for the next four weeks; the slices you build in Weeks 2–4 cross them, and the bugs you debug in Week 1 often happen exactly *at* them.

## What "Service Boundary" Actually Means

A service boundary is the line at which one independently-deployable unit ends and another begins. Concretely, a boundary involves:

- **A separate codebase folder** (often a separate `pyproject.toml` or `package.json`).
- **A separate process** (its own container in our case).
- **A defined interface** for the outside world to talk to it — usually an HTTP API.
- **Its own data store**, or at least its own *logical* ownership of certain data (the "database-per-service" pattern, sometimes loosely applied).
- **A clear responsibility statement** — what this service is for, and what it is *not* for.

The boundary matters because every cross-boundary call is a network hop: slower than a function call, capable of failing in ways an in-process call cannot, and requiring an explicit contract (request shape, response shape, error codes). Boundaries are where complexity concentrates.

## The `rev-eval-ai` PEP Service Map

After the AI-feature strip, the substrate has the following services:

```
                          +-------------------+
                          |     Frontend      |
                          | (Next.js, port    |
                          |  3000)            |
                          +---------+---------+
                                    |
                                    | HTTP
                                    v
                          +-------------------+
                          |   api-gateway     |
                          | (public entry)    |
                          +---------+---------+
                                    |
              +---------------------+--------------------+
              |                     |                    |
              v                     v                    v
   +-------------------+ +-------------------+ +-------------------+
   |   user-service    | |   question-mgmt   | |   test-mgmt       |
   |   (auth, users)   | |   (CRUD questions)| |   (quizzes/tests) |
   |   -> Postgres     | |   -> Mongo        | |   -> Postgres     |
   +-------------------+ +-------------------+ +-------------------+

                  + reporting-and-analytics (empty service candidate
                    — you build this in Week 4)
```

### Service Responsibilities

- **api-gateway.** The single public-facing HTTP entry point. Routes requests to internal services, handles cross-cutting concerns like auth-token verification at the edge. The frontend talks *only* to the api-gateway; the gateway talks to everything else.
- **user-service.** Identity and authentication. Owns user records (Postgres), password handling, login/JWT issuance. If a feature involves "who is this user," it touches user-service.
- **question-management.** CRUD operations over the question bank. Questions are document-shaped, so this is the service backed by Mongo. If a feature involves authoring, editing, or listing questions, it lives here.
- **test-management.** Quizzes and test instances — the questions assembled into a takeable artefact, plus attempt/scoring state. Postgres-backed because of the relational structure (a test has questions, attempts have answers, attempts belong to users).
- **reporting-and-analytics.** *Empty* in your substrate. You build this in Week 4 — the capstone slice. It will pull data from the other services to produce trainer-dashboard views.
- **Frontend (Next.js 16).** The UI. Talks exclusively to api-gateway; never directly to user/question/test services.

## Why These Boundaries, and Why They Matter to You

You did not draw these boundaries — someone else did, and you have inherited them. Three implications:

1. **The boundaries are load-bearing.** When you build the Week 2 question-authoring slice, you do not just "write some code" — you decide which service the new endpoint lives in, whether the frontend talks to api-gateway (yes, always) which routes to question-management, and what data crosses the boundary. The boundary shape constrains your design.

2. **Bugs cluster at boundaries.** When a feature works in isolation but fails end-to-end, the boundary is usually where to look: wrong request shape, missing field, mismatched error contract, auth token not forwarded, a service unreachable from inside the Docker network. Week 1's CI bugs (Days 3–4) include boundary-related symptoms.

3. **You will not need to change the boundaries themselves.** A common newcomer mistake is to "rationalize" inherited architecture — to refactor the boundaries because they look wrong from your current vantage. Don't, this week. Read the boundaries, work within them, and only suggest reshaping them once you understand *why* they were drawn where they were.

## The HTTP Contract Layer

Services talk over HTTP — typically JSON request/response — and each service exposes its API at a base URL inside the Docker network. The api-gateway holds the routing table: "this URL path goes to this service." Internal traffic uses container hostnames (e.g., `http://user-service:8000`); external traffic comes in through the gateway.

You do not need to memorize every endpoint today. You *do* need to understand that:

- Every cross-service call is an HTTP request with a request body, a response body, and a status code.
- Contracts are real things — if user-service changes the shape of its `/login` response, every consumer breaks until they are updated.
- Services should not reach into each other's databases — the only legitimate way to read another service's data is through its API. This is a discipline, not a technical enforcement; nothing stops you from doing the wrong thing, but the boundary is meaningless if you violate it.

## Example / Worked Scenario

Consider what happens when a user logs in:

1. The user submits the login form in the Next.js frontend.
2. The frontend POSTs to `/api/auth/login` on the api-gateway.
3. The api-gateway routes this to user-service's `/login` endpoint.
4. user-service checks the credentials against Postgres, generates a JWT, and returns it.
5. The gateway returns the JWT to the frontend.
6. The frontend stores it (cookie or localStorage, depending on how the inherited code does it) and includes it on subsequent requests.

Now consider the user taking a quiz:

1. The frontend requests a quiz from the api-gateway (`GET /api/tests/123`), with the JWT.
2. The gateway verifies the JWT (or asks user-service to), then routes to test-management.
3. test-management needs the *questions* for this quiz — it does *not* read them out of Mongo directly. It calls question-management's `/questions/{ids}` endpoint.
4. question-management returns the question payloads.
5. test-management assembles and returns the full quiz to the gateway.
6. The gateway returns it to the frontend.

Notice: every dotted line in the service map corresponds to a real HTTP call. Every one is a place where things can go wrong. Internalizing this is the difference between "I think the question-management service is broken" and "I can see the question-management call from test-management is returning 502."

## Common Pitfalls

- **Reaching into another service's database.** A tempting shortcut — "I just need user emails for this report, I'll query Postgres directly." Don't. It looks fine until user-service changes its schema and your code silently breaks. Go through the API.
- **Treating the api-gateway as optional.** The frontend talks *only* to api-gateway. Never bypass it. The gateway is where auth, routing, and CORS live; bypassing it is how staging-vs-prod surprises happen.
- **Confusing service boundaries with module boundaries.** Inside a single service, you have modules/files — those are intra-process boundaries, cheap to refactor. Service boundaries are cross-process, network-spanning, and *expensive* to change. Don't draw them lightly, and don't redraw them this week.
- **Assuming services are reachable from your host.** Inside the Docker network, `http://user-service:8000` works. From your laptop's browser, it does not — you need to go through the published port of the api-gateway (or a directly-exposed port for debugging). Network namespaces are not the same as `localhost`.
- **Designing as if there were a single shared database.** There isn't. user data is in user-service's Postgres; question data is in question-management's Mongo; test data is in test-management's Postgres. Plan accordingly.

## Key Takeaways

- The PEP substrate has 4 backend services (user, question-management, test-management, api-gateway), 1 empty service to build (reporting-and-analytics), and a Next.js frontend; they talk over HTTP via the api-gateway as the public entry point.
- A service boundary = separate codebase + separate process + defined HTTP interface + its own data ownership + a clear responsibility statement.
- Boundaries are load-bearing: they constrain your design, they cluster bugs, and you should *not* try to redraw them this week — read them first.
- Cross-service calls are real HTTP calls with contracts, latency, and failure modes; services never reach into each other's databases.
- The frontend talks only to the api-gateway; inside the Docker network, services address each other by container hostname.

---
*Prerequisites: [01-fde-role-and-pep-positioning-in-the-three-phase-programme.md](01-fde-role-and-pep-positioning-in-the-three-phase-programme.md). Connects forward to Day 2 (Docker Compose stack-up brings these services up together) and Week 4 (you build the reporting-and-analytics service).*
