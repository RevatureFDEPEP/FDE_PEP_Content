# Multi-service feature delivery — impact mapping and recognizing cross-cutting infrastructure changes

> *Day 10: Integrate the Slice + Scaffold Reporting Service + Week 2 Review — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

In a single-process monolith, "ship a new endpoint" means writing a route handler. In the substrate you've been working in for two weeks — Compose-orchestrated services behind an `api-gateway` and (from Day 6) a reverse proxy — "ship a new endpoint" means writing a route handler **and** a gateway routing rule **and** a reverse-proxy path **and** a compose service entry **and** a CI job. Skipping any one means the feature appears to work in isolation and fails in the integrated stack. Today's topic introduces **impact mapping** — a static-analysis technique applied before any code is written — to surface every surface a change touches, so the integration step at the end of the slice is uneventful instead of fragmented.

## The Problem: a New Endpoint Isn't a Route Handler

The reflex from monolith-era development is "endpoint = route handler." In a multi-service substrate that reflex is wrong, and the wrongness is invisible until integration time.

Consider what actually happens when a browser request reaches the reporting service we're scaffolding today:

```
browser → reverse proxy (Nginx) → api-gateway → reporting-and-analytics-service
                ↓                       ↓                       ↓
       location block for           routing rule for     route handler for
       /v1/api/reports/*            /v1/api/reports/*    GET /reports/user/{id}
       (Day 6 substrate)            (path-based fwd)     (the "real" code)
```

That's three routing layers in front of the handler. Each layer is its own config surface, its own deploy target, its own thing that can be out of sync with the others. And that's just the request path — the change also lives in:

- **`docker-compose.yml`**: the new service entry, its `depends_on`, its healthcheck, its env vars, its host port (`day-10-service-composition-patterns-in-docker-compose`).
- **CI workflow**: a new job or matrix entry that builds, lints, and tests the new service; possibly new artifacts to upload (`day-10-extending-ci-workflows-for-new-tests-and-services`).
- **Database**: an Alembic migration for the new schema (Day 10's Alembic topic).
- **Service discovery**: the gateway's environment variable for `REPORTING_SERVICE_URL` so it knows where to forward.

A new endpoint, in this substrate, is the union of all of those. Forgetting any one means the unit tests pass, the local-only smoke test passes, and the integrated stack still 404s in the browser.

## Impact Mapping: the Technique

**Definition.** Before writing code, enumerate every surface in the system the change touches. The output is a tree or table — one row per surface — that becomes the checklist for the PR.

**When to run it.** At the start of any feature that crosses a routing layer. The cost is 10 minutes of paper-thinking. The cost of skipping it is half a day of "why doesn't this work?" during integration.

**How to run it.** Walk the request path end-to-end, and at every layer ask:

1. **Does this layer need a new entry/rule/config to know about the change?**
2. **Does an existing entry need to be modified?**
3. **What's the failure mode if I forget this layer?**

Then walk the supporting infrastructure (Compose, CI, DB, secrets) and ask the same three questions.

## Worked Example: `GET /reports/user/{id}`

The reporting service we scaffold today will eventually expose a per-user report endpoint. Before writing it, run the impact map.

### Tree form

```
GET /reports/user/{id}  (new endpoint on reporting-and-analytics-service)
├── reporting-and-analytics-service (the "real" code)
│   ├── route handler in app/api/reports.py
│   ├── Pydantic response schema in app/schemas/reports.py
│   ├── DB query / repository layer
│   ├── Alembic migration if new tables are needed
│   └── pytest cases (happy path + 404 + auth)
├── api-gateway (path-based forwarding)
│   ├── routing rule for /v1/api/reports/*
│   ├── REPORTING_SERVICE_URL env var (service discovery)
│   └── gateway integration test (or smoke test)
├── reverse proxy / Nginx (browser-facing — only if the frontend hits it directly)
│   └── location block for /v1/api/reports/* (or rely on /v1/api/* catch-all)
├── docker-compose.yml
│   ├── reporting-and-analytics-service entry
│   ├── reporting-postgres entry (own DB — see service-composition topic)
│   ├── depends_on with condition: service_healthy
│   ├── healthcheck for /healthz
│   ├── env vars (REPORTING_DATABASE_URL, REPORTING_LOG_LEVEL)
│   └── api-gateway gets REPORTING_SERVICE_URL added
└── CI (.github/workflows/ci-pipeline.yml)
    ├── new job or matrix entry: build reporting-and-analytics-service image
    ├── lint (Ruff) and test (pytest) steps for the new service
    ├── Trivy scan on the new image
    └── any new test artifacts (coverage, junit)
```

### Table form

| Surface | Change type | Failure mode if skipped |
|---|---|---|
| reporting service handler + schema | new code | 404 from the service itself |
| reporting service tests | new tests | regression invisible |
| Alembic migration | new file | schema drift; runtime errors on query |
| api-gateway routing rule | new rule | 404 at gateway — service never reached |
| api-gateway env (`REPORTING_SERVICE_URL`) | new var | gateway can't resolve the host |
| Nginx location block | new or modified | browser request 404s before gateway |
| `docker-compose.yml` service entry | new block | `docker compose up` doesn't start it |
| `depends_on` + healthcheck | new conditions | gateway boots before reporting; intermittent 502s |
| CI job/matrix | new entry | new service code never lints, builds, or tests in CI |
| Trivy scan | new step | unscanned image ships |

Either form works. Tree is good for thinking; table is good for putting in the PR description as the "what this change covers" checklist.

## Detection Heuristic: When Impact Mapping Pays Off

You don't need to run a full impact map for a typo fix or an internal refactor. The heuristic for when it pays off:

> **Whenever a feature crosses a routing layer, assume the change is multi-touch until impact mapping proves otherwise.**

Routing layers in *this* substrate, ordered by request flow:

1. **Reverse proxy / Nginx** (added Day 6) — browser-facing entry point, path-based routing.
2. **API gateway** — internal routing across backend services.
3. **Compose networks** — DNS-level service-to-service routing.

If your change adds, modifies, or relies on any of those — assume multi-touch. The corollary: features that live entirely inside one service (a new validator on an existing endpoint, a refactor of an internal helper) are usually single-touch and don't need this much ceremony.

A second trigger worth naming: **new infrastructure (new service, new datastore, new env var) is always multi-touch**, even if no routing layer is crossed, because Compose and CI both need to know about it.

## Connecting to Today's Deliverable

Today's deliverable has two cross-cutting pieces, both of which are exactly the kind of multi-touch change this technique exists for:

1. **Scaffold `reporting-and-analytics-service`.** Even though the service has only `/healthz` today, the scaffolding itself touches Compose (new service entry, new `reporting-postgres`), CI (a new job to build/lint the empty service), and — once endpoints exist — the gateway and proxy. Running the impact map *before* scaffolding produces the same checklist `day-10-service-composition-patterns-in-docker-compose` and `day-10-extending-ci-workflows-for-new-tests-and-services` walk through.

2. **Integrate the Week 2 question-authoring slice.** Day 8 produced a backend; Day 9 produced a frontend. By definition the integration touches every layer between them: Nginx → gateway → question-management-service. If the slice's contract was specified up front (it was, on Day 8), the impact map for the integration step is mostly verification — does each layer carry the contract through cleanly? The places to check are exactly the surfaces the map enumerated.

The pattern repeats every time a feature crosses services for the rest of the course. Week 4's results endpoints will need the same map. The capstone defense will assume you can produce one for any cross-cutting change you made.

## Common Pitfalls

- **Doing the map after writing the handler.** The point is to know what you're signing up for *before* you start. Running it retroactively turns it from a planning tool into a regret-cataloguing tool.
- **Mapping only the "obvious" routing layer.** It's easy to remember the gateway and forget the Nginx location block, or vice versa. Walk the request path end-to-end every time, including the layers that "didn't change last time."
- **Forgetting the infrastructure column.** Compose and CI are the two most-skipped surfaces because they don't carry user requests. They still break the feature if neglected — the CI job that never runs is the regression you'll discover three weeks later.
- **Treating the impact map as a one-time artifact.** If the contract changes mid-slice, redo the map for the changed shape. A field renamed on the backend may now require a frontend zod change *and* an updated gateway response transform, depending on the substrate.
- **Mapping without naming failure modes.** "The gateway needs a rule" is half the entry; "if I skip this the frontend gets 404 at the gateway and the service never sees the request" is the full one. The failure mode is what makes the row stick.

## Key Takeaways

- A new endpoint in a multi-service stack is **route handler + gateway rule + proxy path + compose entry + CI job** — not just a route handler.
- **Impact mapping** is a 10-minute static-analysis pass before coding: enumerate every surface the change touches, in a tree or a table, with failure modes named.
- The detection heuristic: **whenever a feature crosses a routing layer, assume multi-touch until impact mapping proves otherwise**. Routing layers in the substrate are Nginx, the api-gateway, and Compose networks.
- New infrastructure (service, datastore, env var) is always multi-touch — Compose and CI both need entries even if no routing layer is crossed.
- Today's deliverables — the reporting-service scaffold and the question-authoring slice integration — are both multi-touch by construction. Running the impact map up front is what turns integration from fragmented into routine.

---
*Prerequisites: day-8 backend slice topics, day-9 frontend slice topics, [[day-10-service-composition-patterns-in-docker-compose]], [[day-10-extending-ci-workflows-for-new-tests-and-services]].*
