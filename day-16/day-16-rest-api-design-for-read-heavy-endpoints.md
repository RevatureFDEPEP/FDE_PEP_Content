# REST API Design for Read-Heavy Endpoints

> *Day 16: Results Slice (Backend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The reporting service that was scaffolded on D10 (FastAPI + Postgres + Alembic, plumbed into Compose, wired through the D6 reverse proxy with `X-Request-Id`) so far returns nothing useful. Today it grows real endpoints. Unlike the write-heavy contracts the cohort built in W2 (question authoring) and W3 (sessions and submits), reporting is **read-only and read-heavy** — every endpoint is a `GET`, every request shape is "show me a slice of historical data," and every response shape is "the JSON the UI needs, in the order it wants it, with the minimum number of round trips." Designing those endpoints well is a different skill than designing CRUD endpoints, and that's what this topic covers.

## The Read-Heavy Mindset

A write endpoint is shaped by *correctness*: the contract has to capture what the caller wants to mutate, the server has to validate it, the state transition has to be atomic. A read endpoint is shaped by *fit-to-consumer*: the contract has to give the UI exactly the data it needs to render a screen, in one trip, without forcing it to do additional joins or aggregations client-side. A read endpoint that returns "all the raw rows" and forces the frontend to group, sort, and aggregate is technically correct and operationally terrible — slow page loads, duplicated logic, brittle UI code.

So the design question is always: **what screen does this endpoint serve, and what's the minimum shape the screen needs?**

For Day 16, the screen is the candidate's own results page (Day 17 builds it). It needs to show:

- a summary line ("you scored 8/10 on Quiz X, taken 2026-05-19, 12 minutes elapsed"),
- a list of attempts in reverse chronological order,
- per-attempt drill-down (score, time elapsed, per-question time).

That shape — summary + list + nested per-item detail — drives the endpoint.

## Resource Naming For Reads

The substrate's mutation endpoints follow patterns like `POST /sessions`, `POST /sessions/{id}/submit`. Reporting is a **separate read model**, so it lives under a `/reports/` prefix to make the boundary obvious:

```
GET /reports/user/{user_id}                  # per-candidate summary + recent attempts
GET /reports/user/{user_id}/attempts         # paginated list of all their attempts
GET /reports/test/{test_id}                  # per-test aggregates (D18 — trainer dashboard)
GET /reports/test/{test_id}/attempts         # paginated attempts for a test (D18)
```

The naming conveys: *this is the reporting view of the user/test resource, not the canonical record.* The canonical record (the `sessions` row, the `answers` rows) still lives in test-management. Reporting is a *projection* — it reads from somewhere (today, directly from the test-management Postgres; see Topic 4 for the trade-off discussion) and shapes it for a UI.

Avoid temptations like:

- `GET /user/{id}/report` — looks like the user resource owns the report; it doesn't.
- `GET /reports/get-user-report?user_id=...` — `get-` verbs in URLs are a smell; `GET` is the verb.
- `POST /reports/query` with a JSON body — RPC-flavored, breaks caching, breaks bookmarkability, breaks the browser network tab as a debugging tool.

REST conventions exist because they make the system easier to reason about, cache, log, and proxy. Stay inside them.

## Response Shape Optimized For The UI

The Day 17 results page needs the summary and the list together — that's one render, one fetch. So the `GET /reports/user/{user_id}` response is shaped as a nested envelope:

```json
{
  "user_id": "u_42",
  "summary": {
    "total_attempts": 5,
    "completed_attempts": 4,
    "average_score": 7.5,
    "best_score": 9
  },
  "recent_attempts": [
    {
      "session_id": "sess_103",
      "test_id": "test_python_basics",
      "test_name": "Python Basics",
      "submitted_at": "2026-05-18T14:22:01Z",
      "score": 8,
      "max_score": 10,
      "elapsed_seconds": 723,
      "per_question": [
        {"question_id": "q_1", "correct": true,  "elapsed_seconds": 45},
        {"question_id": "q_2", "correct": false, "elapsed_seconds": 78}
      ]
    }
  ]
}
```

A few specific design choices in there worth naming:

- **`summary` is a sub-object, not flat fields.** Future summaries (per-topic accuracy, trend lines) can be added as new keys under `summary` without flattening into a wide root object.
- **`recent_attempts` is bounded** (default 10, configurable via Topic 3's pagination). The full list lives at `/reports/user/{id}/attempts`. Don't return unbounded arrays from a "summary" endpoint — that's how a 200ms endpoint becomes a 30-second timeout when a power-user has 500 attempts.
- **`test_name` is denormalized into the response.** The UI needs the name; reporting fetches it once from question-management (Topic 4 covers how) and embeds it. The UI doesn't make a second hop to look up names.
- **`max_score` is included.** The UI doesn't have to know that all PEP quizzes are out of 10 — that's a magic number. Include the denominator.
- **`per_question` is inline, not a separate endpoint.** The detail panel is shown on the same screen; making it a separate fetch per attempt would be N+1 from the client. Inline it, paginate it if it grows beyond ~50.
- **Times are ISO 8601 UTC strings, durations are integer seconds.** Always. The cohort fought this fight on D14 with the timer; don't relitigate it in reporting.

The shape is *driven by the screen*. If a different screen needed a different shape, that's a different endpoint — don't try to make one endpoint serve every consumer with a `?fields=` query param. That's a different design (GraphQL, or sparse fieldsets) and it's not what we're building today.

## Sizing The Response

A read endpoint that returns 5 MB of JSON is a design failure even if the data is "correct." Some discipline:

- **Bounded by default.** Every list inside a response has a default cap. `recent_attempts: 10`, `per_question: 50`. Topic 3 covers how to make the cap configurable cleanly.
- **No raw blobs.** If a record has long-form text (e.g., free-text answers, which PEP doesn't have but Phase 2 will), return a `truncated_text` + `length` and a link to the full record.
- **Project, don't return-the-row.** Reporting doesn't return `attempts.*`; it returns *only the columns the UI needs*. The SQLAlchemy queries in Topic 2 select specific columns, not entire ORM rows.

A rough target: the `GET /reports/user/{id}` response for a typical PEP candidate (5–10 attempts, 10 questions each) should be well under 50 KB. If the cohort sees responses pushing 500 KB, they're doing it wrong.

## HTTP Status Codes For Reads

Simpler than mutations, but still worth being deliberate:

| Status | When |
|---|---|
| `200 OK` | Resource exists; body is the report. (Even if `recent_attempts` is empty — empty array is a valid result, not an error.) |
| `404 Not Found` | The `user_id` doesn't exist in the source system. Distinct from "user exists but has no attempts." |
| `400 Bad Request` | Malformed query params (bad date format, `page=abc`). Pydantic's validation produces these automatically (D11). |
| `401 / 403` | Caller isn't authenticated, or is asking for someone else's report without permission. PEP defers full authz to Phase 2 but the cohort should at least log a TODO. |
| `429 Too Many Requests` | Out of scope for PEP, but flag it for Phase 2 — reporting endpoints are a classic abuse vector. |
| `500 Internal Server Error` | Upstream (test-management API call, Postgres) failed. Don't leak the underlying error message; log with `request_id` and return a generic envelope. |

Critically: **empty data is not 404.** A candidate who hasn't taken any quizzes yet gets a 200 with `total_attempts: 0` and `recent_attempts: []`. The screen renders an empty-state message. 404 means *the user doesn't exist*; 200 with an empty list means *the user exists and has no data*. Conflating those breaks the frontend's ability to render the right message.

## Idempotency, Safety, And Caching

`GET` is **safe** (no server-side mutation) and **idempotent** (same request, same response — modulo new data arriving). These properties unlock several capabilities the cohort should know about:

- **Browser and intermediary caches** can cache `GET` responses based on `Cache-Control` headers. For reporting, set `Cache-Control: private, max-age=30` — private (per-candidate, don't share across users), 30 seconds (short, because new attempts arrive). Skip caching entirely for the trainer dashboard endpoints where staleness matters more.
- **Retry safety.** The frontend can retry a `GET` on network failure without worrying about double-execution. Write endpoints can't (D12's idempotency keys were the W3 mitigation). The Day 17 results page can use simple `fetch` + retry; no token gymnastics.
- **Conditional requests.** `ETag` + `If-None-Match` lets a client say "give me the body only if it changed since last time" and get a 304 otherwise. Out of scope for PEP but worth naming as a future optimization for the trainer dashboard.

The `Cache-Control` header is one line of middleware code; add it on Day 16 even if the frontend doesn't yet exploit it.

## Versioning, Briefly

The substrate doesn't version its APIs (`/sessions`, not `/v1/sessions`). That's an architectural choice the trainer can defend in W4 retro: the system is small, the consumers are co-located, version-bump-via-deploy is acceptable. Reporting follows the same convention — `/reports/user/{id}`, not `/reports/v1/user/{id}`. If Phase 2 introduces external consumers (the AI surface, third-party integrations), versioning gets reintroduced then. Don't add it preemptively just because.

## Anti-Patterns

- **Returning ORM rows directly.** The Pydantic response model (D11) is the contract; map to it explicitly. Don't `return session.scalars(...).all()` and hope the serializer figures it out — it'll leak internal column names and timestamps the frontend doesn't need.
- **Single mega-endpoint with `?fields=`.** The temptation to build one endpoint serving every screen via field selection is real and almost always wrong at this scale. One endpoint per screen, shaped to the screen.
- **Unbounded list responses.** "It's only 10 candidates today" is a sentence that ends careers. Cap every list, document the cap, paginate beyond it (Topic 3).
- **POST for queries with bodies.** Breaks caching, breaks the browser, breaks the principle of least surprise. If a query truly cannot fit in URL params, that's a sign the design needs rethinking — not a justification for `POST /reports/query`.
- **404 for "no data yet."** Frontend can't distinguish "user doesn't exist" from "user has nothing." Always return 200 with an empty shape unless the resource itself doesn't exist.
- **Verbs in URLs.** `GET /reports/get-user-report` — no. The HTTP verb is the verb. The URL is the noun.
- **Returning `null` for things the UI iterates.** `recent_attempts: null` requires every frontend caller to add a null check. `recent_attempts: []` doesn't. Empty arrays for lists, empty objects for objects, omit-the-key for genuinely optional fields.
- **Leaking internal IDs the UI doesn't need.** If the frontend doesn't use `internal_sequence_no` or `pg_oid`, don't include them. The contract is what you commit to maintaining.

## Key Takeaways

- Read-heavy endpoints are shaped by the screen they serve, not by the underlying tables. One endpoint per consumer use case; nest summary + list + detail when the screen renders them together.
- `/reports/...` prefix signals a projection; the canonical record lives in the owning service (test-management). The cohort defends this boundary on Topic 4.
- Bound every list, project only the columns the UI needs, denormalize cross-service names to avoid client-side N+1.
- Use HTTP semantics deliberately: 200-with-empty for "no data," 404 for "resource missing," `Cache-Control` for the safe-and-idempotent properties of `GET`.
- Don't return ORM rows — Pydantic response models are the contract (D11 reinforced).

---
*Prerequisites: day-10-fastapi-service-scaffolding-conventions, day-11-pydantic-request-response-modeling, day-11-fastapi-routing-and-dependency-injection-patterns.*
