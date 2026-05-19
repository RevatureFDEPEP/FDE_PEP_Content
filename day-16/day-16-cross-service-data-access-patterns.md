# Cross-Service Data Access Patterns (Own-Database vs API Call)

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The reporting service needs data it doesn't own. Attempts live in test-management's Postgres; question text lives in question-management's Mongo; user names live in user-service. Three options exist for accessing data owned by another service: (1) **call the other service's API**, (2) **read its database directly**, (3) **maintain your own projected copy** populated by events. Each has real trade-offs. The point of today's topic isn't to memorize "the right answer" — there isn't one — but to give the cohort the vocabulary to make and defend the choice. This is one of the few "architectural decision" topics in W4 that they'll be asked to defend in the capstone demo (D20).

## The Three Patterns

### Pattern A: Cross-Service API Call

Reporting calls test-management's HTTP API to get attempts. The same pattern api-gateway already uses (D11 introduced `httpx.AsyncClient` for cross-service calls).

```python
async with httpx.AsyncClient(base_url=TEST_MGMT_URL, timeout=5.0) as client:
    resp = await client.get(f"/sessions?user_id={user_id}")
    resp.raise_for_status()
    attempts = resp.json()
```

### Pattern B: Direct Database Access

Reporting connects to test-management's Postgres directly with a read-only role and queries the tables.

```python
async with TestMgmtSession() as db:
    rows = (await db.execute(select(Session).where(Session.user_id == user_id))).scalars().all()
```

This is what the Topic 2 SQLAlchemy examples assumed. It's also what the deliverable's `GET /reports/user/{id}` does for Day 16 specifically — and we'll defend why below.

### Pattern C: Own Projected Read Model

Reporting maintains its own copy of the relevant data in its own database, populated by events (Kafka, change-data-capture, or a webhook from test-management on every state change).

```python
# test-management publishes after submit
await event_bus.publish(AttemptSubmitted(session_id=..., user_id=..., score=...))

# reporting subscribes and writes its own row
async def handle(event: AttemptSubmitted):
    await reporting_db.execute(insert(AttemptProjection).values(...))
```

Reporting then queries its own DB, decoupled from test-management entirely.

## The Trade-Off Table

| Concern | A: API Call | B: Direct DB | C: Projected Read Model |
|---|---|---|---|
| **Coupling — what reporting depends on** | API contract (stable, versioned) | Schema (changes when test-management migrates) | Event schema (stable if designed well) |
| **Consistency** | Read-your-writes (call the source) | Read-your-writes (same DB) | Eventually consistent (lag between write and projection update) |
| **Performance — read latency** | One HTTP hop (5–50 ms typical) | One DB query (1–10 ms) | One DB query against local data (1–10 ms) |
| **Performance — load on owner** | Each reporting read is a request to test-management | Each reporting read is a query against test-management's DB pool | Zero load on test-management at read time |
| **Failure modes** | If test-management is down, reporting can't answer | If test-management's DB is down, both services are down | Reporting answers from its own DB regardless of test-management's status |
| **Schema evolution** | Owner can migrate freely; API is the contract | Owner migrating breaks reporting silently | Owner can migrate freely; event schema is the contract |
| **Aggregation across services** | Complex — need to fan out and stitch | Possible via multi-DB joins (ugly) or repeated queries | Trivial — local DB has everything you projected |
| **Operational complexity** | Low (HTTP is plumbed already) | Medium (manage DB credentials, two ORM model sets) | High (event bus, idempotent consumers, replay tooling) |
| **Testability** | Mock httpx in tests | Use a real test DB with fixtures (D15 pattern) | Mock the event bus and the local DB |
| **Bootstrapping** | Works from day one | Works from day one | Need to backfill historical data before launch |

The takeaway: **none of these are wrong; they're choices weighted by what the system needs.** PEP's task today is to pick deliberately and articulate the rationale.

## The Recommendation For PEP, And Why

For the Day 16 deliverable specifically, we go with **Pattern B (direct DB access)** — and the cohort should be able to defend it. Here's the defense:

1. **The data shape we need is heavily relational** (joins across `sessions`, `answers`; aggregates with `group_by`). Expressing those queries as a sequence of HTTP calls in reporting would be N+1 hell: fetch sessions, fetch answers for each, do the join in Python. Bad.
2. **Test-management doesn't expose the bulk query API we'd need.** Its endpoints are CRUD-shaped for the candidate UI (`POST /sessions`, `POST /sessions/{id}/submit`), not bulk-list-shaped for reporting (`GET /sessions?user_id=...&include=answers&limit=1000`). Adding that API surface to test-management would be its own week of work, and it'd be reporting-specific endpoints living on the wrong service — a smell.
3. **The cohort is one team.** The full-shared-DB anti-pattern at Spotify scale doesn't apply when there's one developer team that owns both services. Schema changes in test-management get coordinated with reporting at PR review time, not via formal API contracts.
4. **Postgres is the same physical instance in Compose**, so "another network hop" isn't a real performance argument either way, but the DB-direct path is one query, not a query + an HTTP round trip + JSON serialization + JSON deserialization.
5. **Event-driven projection (Pattern C) is over-engineered for the cohort size and feature scope.** Setting up a Kafka topic, an idempotent consumer, backfill tooling, and replay tooling for a service that serves *one* page in the UI is yak-shaving. The cohort defends not building it today and flags it as Phase-2 work if reporting scales.

Where Pattern B falls down — and what the cohort should *also* be able to say:

- **Coupling on schema is a real cost.** If test-management adds a non-nullable column without a default in its migration, reporting breaks at the next query until reporting's models update. The mitigation in PEP: reporting's models are *duplicated* (not shared) ORM definitions, so a column rename in test-management produces a *clear* `UndefinedColumn` error in reporting rather than mystery behavior. CI runs a smoke query (Topic 6) that catches drift.
- **Cross-service authorization gets fuzzy.** Test-management's API would enforce "user u_42 can only see their own attempts." If reporting bypasses the API, reporting has to enforce it too. The cohort adds the check explicitly: `WHERE user_id = current_user_id` is non-negotiable.
- **The next service that wants attempt data faces the same choice.** If five services start direct-DB-reading test-management, you've recreated the shared-database anti-pattern by attrition. The mitigation: explicitly cap "direct DB read" to one reader (reporting); if a second service needs it, that's the trigger to build the API or the projection.

## When To Pick API Call Instead

Pattern A is correct in several situations the cohort will encounter:

- **Small reads where the owner already has the right endpoint.** Reporting needs the *name* of each test to denormalize into the response. Question-management already has `GET /tests/{id}` returning a test's metadata. One HTTP call per test (cached for the request) is fine. Don't direct-DB-read Mongo just for a name lookup.
- **Cross-domain writes.** Reporting is read-only; if it were ever to *mutate* test-management's data, that goes through the API, full stop. Writes must respect the owner's invariants.
- **Owner is in another organization or team.** If test-management was maintained by a separate team, the API contract is the only safe coupling.

## When To Pick Projection Instead

Pattern C is correct in several situations PEP doesn't hit but Phase 2 might:

- **Read load is so high that calling or querying the owner would degrade it.** A dashboard serving thousands of trainers in real-time would crush test-management's DB if it queried directly.
- **Multi-source aggregation.** A dashboard joining attempts (test-management), user metadata (user-service), and question performance (question-management Mongo + aggregated stats) is much more tractable when reporting projects all three into local tables.
- **Historical/analytical workloads.** Time-series rollups, ML feature stores, anything that wants append-only history. The projection is the analytic copy.

The capstone defense should at least *name* projection as the next step — "if this scales, we project."

## Implementation Discipline For Pattern B

The cohort should write the direct-DB code with the trade-offs visible in the code itself:

```python
# reporting-service/app/external/test_management_db.py
"""Direct DB access into test_management_db.

Coupling note: This service reads test-management's schema directly. The
ORM models below are duplicated from test-management; any breaking change
to test-management's schema requires a coordinated PR here.

If a second service ever needs this data, we should reconsider: either
expose a bulk-list API on test-management, or project the data into
reporting's own DB via events.
"""

from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

test_mgmt_engine = create_async_engine(
    settings.TEST_MGMT_DATABASE_URL,  # uses 'reporting_reader' role with SELECT only
    pool_size=5,
    pool_pre_ping=True,
)
TestMgmtSession = async_sessionmaker(test_mgmt_engine, expire_on_commit=False)
```

The docstring makes the coupling explicit. The role name (`reporting_reader`) flags intent in the connection string. The pool size is small (reporting is one of multiple readers; don't hog the DB pool).

For the test-name lookup that *is* an API call:

```python
# reporting-service/app/external/question_mgmt_api.py
async def get_test_name(test_id: str) -> str:
    async with httpx.AsyncClient(base_url=QM_URL, timeout=2.0) as client:
        try:
            resp = await client.get(f"/tests/{test_id}")
            resp.raise_for_status()
            return resp.json()["name"]
        except (httpx.HTTPError, KeyError):
            return test_id   # graceful fallback: return the id if the lookup fails
```

`timeout=2.0` is deliberate — reporting shouldn't hang on a slow question-management response. Graceful fallback (return the id if the name fetch fails) keeps the report page rendering even when question-management is degraded.

## What The Trainer Should Say Out Loud

This topic is one of the most important "architectural intuition" moments of the curriculum. The trainer should run a 10-minute discussion on the question:

> *"We're going to do direct DB access for attempts and API call for test names. Why those choices? What's the cost if we're wrong?"*

Let the cohort argue. The senior candidates will land on most of the points above; the others learn by listening. Then frame it as the kind of decision they'll make every week in their first job — *the answer matters less than the ability to articulate the trade-off.*

## Anti-Patterns

- **Shared ORM models across services.** Tempting because "don't repeat yourself," catastrophic because it couples deploy cycles. Duplicate the models; pay the maintenance cost; gain isolation.
- **Direct-DB writes from a non-owning service.** Bypassing the owner's invariant checks (D11/D12 validation, locking, status transitions) corrupts state. Reads can be direct; writes go through the owner's API.
- **Using one pattern for everything because "consistency."** Different sub-problems have different right answers. Be deliberate per call site.
- **Skipping the authorization check because "it's behind a reverse proxy."** Defense in depth: every read must enforce `WHERE user_id = current_user_id` (or equivalent). PEP's auth story is thin but the placeholder belongs in the code today.
- **Calling the owner's API in a loop.** N+1 over HTTP is worse than N+1 over SQL. If you're about to write `for x in things: api_call(x)`, batch it (`api_call(list_of_ids)`) or rethink the design.
- **No timeout on cross-service HTTP.** A hung test-management call makes reporting hang too. Always set a tight timeout; degrade gracefully.
- **Pretending eventual consistency away.** If the cohort ever does go to Pattern C, they have to be ready to answer "the user just submitted and the report doesn't show it." It's not a bug; it's the design — but the UI has to communicate it.
- **Building Pattern C "in case we need it later."** Premature event-sourcing is a project-killer. Build the simplest pattern that works; rebuild when the constraints actually change.

## Key Takeaways

- Three patterns: cross-service API call, direct DB access, projected read model. Each has real trade-offs around coupling, consistency, performance, and operational complexity.
- For Day 16: direct DB access to test-management's attempt tables (defended by relational queries + single team + low feature scope); API call to question-management for test names (small lookup, owner has the endpoint).
- Coupling-on-schema is the cost of direct DB access; mitigate with duplicated ORM models, CI smoke queries, and a hard cap of one direct reader.
- Writes always go through the owner's API; reads can take any of the three patterns; pick deliberately and articulate why.
- Project Pattern (C) is where the system goes if it scales — flag it as Phase 2 work, don't build it today.

---
*Prerequisites: day-11-cross-service-http-integration-httpx-rest-contracts, day-1-microservices-architecture-and-trade-offs, day-16-sqlalchemy-queries-joins-aggregates-subqueries.*
