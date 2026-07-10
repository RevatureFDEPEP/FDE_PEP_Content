# Edge Case Handling for Distributed Clients (Timeout, Retry, Double-Submit)

> *Day 12: Scoring Engine & Attempt Locking (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The quiz-taking endpoint sits between a browser on a flaky network and a stateful database. The interesting failure modes don't happen on the happy path — they happen when the candidate's WiFi drops mid-submission, when they double-click "Submit", when they leave the tab open for 35 minutes and come back to a session that expired five minutes ago. Each of these is a recurring, predictable scenario that the endpoint must handle with a **specific HTTP response and a specific server-side effect**. This file catalogs the edge cases as a table — symptom, mechanism, response, why this response — so the implementation can be reviewed against a checklist instead of a vibe.

## The Catalog

Read this table as the **specification** for the endpoint's behavior. Each row is a scenario the implementation must handle deliberately. The mechanisms reference the other Day 12 topics.

| # | Scenario | HTTP response | Why this response | Mechanism (which topic) |
|---|---|---|---|---|
| 1 | Happy-path submission | `200 OK` + next question | Normal flow | All of D12 |
| 2 | Network drop mid-response; client retries with **same** `Idempotency-Key` | `200 OK`, replayed stored response | Retry is safe; no double-score | Topic 3 (idempotency) |
| 3 | Network drop mid-response; client retries with **different** key | `409 Conflict` *or* `200 OK` if state allows | Same answer to same question = `409`; new question = `200` | Topic 4 (`uq_answer_per_question`) + topic 5 (lock) |
| 4 | Double-click on Submit Answer; two requests, same key | `200 OK` to both, identical body | Lock serializes; idempotency replays | Topic 5 + topic 3 |
| 5 | Two open tabs, both submitting | `200 OK` to first, `409 Conflict` to second | First wins the lock and the unique constraint; second is a real conflict | Topic 4 + topic 5 |
| 6 | Submission after `expires_at` | `410 Gone` | Session expired; state is terminal-ish | Topic 6 (state machine) + D11 server-auth time |
| 7 | Submission after submit (already-submitted session) | `409 Conflict` | Mutation rejected; state is `submitted` | Topic 6 |
| 8 | Submission with valid `X-Session-Token` but wrong session_id in URL | `404 Not Found` | Token authenticates its session, not others | D11 token discipline |
| 9 | Submission with no `X-Session-Token` | `401 Unauthorized` | No credential | D11 |
| 10 | Submission with wrong `X-Session-Token` | `401 Unauthorized` | Bad credential — don't reveal whether the session exists | D11 |
| 11 | `Idempotency-Key` reused with a different body | `422 Unprocessable Entity` | Client bug; refuse to replay or accept | Topic 3 |
| 12 | Malformed `selected_options` (negative index, out of range) | `422 Unprocessable Entity` | Pydantic validation; no DB work | D11 pydantic modeling |
| 13 | Deadlock detected during the critical section | `503 Service Unavailable` after 3 retries; `200 OK` if a retry succeeds | Transient; safe to retry | Topic 5 deadlock handling |
| 14 | Question-management-service down (lookup fails) | `502 Bad Gateway` | Upstream failure; not the client's fault | D11 httpx error mapping |
| 15 | DB connection lost during transaction | `503 Service Unavailable` | Infra failure; client retries | Topic 5 |

Fifteen rows. Every endpoint in Week 3 should ship with a table like this in the PR description.

## The Three Headline Cases

Three rows in the table are worth a closer look because they're the ones most likely to be missed by junior reviewers.

### Case 2 — Timeout-then-retry with same key

The candidate clicks "Submit Answer." The server scores the answer in 12ms. The response packet drops somewhere between AWS and the candidate's home network. The candidate's browser sees a 30-second timeout and retries. The retry arrives.

Without idempotency: the server scores *again*. Two answer rows (now prevented by `uq_answer_per_question`), or one row with a confused score, or session index advanced by 2 instead of 1. Topic 4's unique constraint catches the "two rows" version; topic 3's idempotency key catches the "scored twice" version.

With idempotency: the retry's `Idempotency-Key` matches the original. The lookup hits; the response replays. The client sees the same JSON. The candidate is none the wiser. **The right behavior is invisible.**

### Case 5 — Two tabs, double-submit

The candidate opens the quiz in two tabs (intentionally or not). Each tab has its own React state, so each generated a *different* `Idempotency-Key` for the next "Submit." Both submit the same question's answer at the same moment.

Sequence (using topic 5's lock):

```
Tab A: BEGIN; SELECT...FOR UPDATE on session → acquires lock
Tab B: BEGIN; SELECT...FOR UPDATE on session → BLOCKS
Tab A: idempotency miss; score; insert answer; advance index 4→5; COMMIT → lock released
Tab B: lock acquired; idempotency miss; tries to insert answer for question 4
       → uq_answer_per_question fires → IntegrityError
Tab B: catch IntegrityError; return 409 Conflict, "answer_already_submitted_for_question"
```

Tab A sees `200 OK` with question 5; Tab B sees `409` and shows "looks like another session for this quiz is in progress." That's a clean, understandable failure. **The lock + the constraint together produce the right behavior.** Either alone would be insufficient.

### Case 6 — Submission after expiry

The candidate gets distracted for 35 minutes. The session expired 5 minutes ago. They finally click "Submit Answer."

```python
async with db.begin():
    session = await session_repo.get_for_update(db, session_id)
    if session is None:
        raise HTTPException(404, "session_not_found")
    assert_in_progress(session, server_now())   # ← raises 410 here
```

The response is **`410 Gone`** with `{"error": "session_expired", "expired_at": "..."}`. Why 410 and not 409?

- **`410 Gone`** says: "This resource existed and was deliberately retired. Don't ask again."
- **`409 Conflict`** says: "Your request conflicted with current state. Possibly retryable."

Expiry is permanent and one-way; 410 is the precise verb. The client should not retry; the UI should redirect to the results page (or a "session expired" view). The same response shape is used for `expired_at` *and* the lazy-expired transition from topic 6 — both are the same outcome from the client's perspective.

## The Status Code Cheat Sheet

| Outcome | Status | Body shape |
|---|---|---|
| Submitted, scored, here's the next question | `200 OK` | `AnswerResponse` |
| Idempotency replay | `200 OK` | Same `AnswerResponse` byte-for-byte |
| Auth missing or bad | `401 Unauthorized` | `{"error": "unauthorized"}` |
| Session not found *or* token doesn't match | `404 Not Found` | `{"error": "session_not_found"}` |
| Already submitted | `409 Conflict` | `{"error": "session_already_submitted"}` |
| Answer already exists for this question (race lost) | `409 Conflict` | `{"error": "answer_already_submitted_for_question"}` |
| Session expired | `410 Gone` | `{"error": "session_expired", "expired_at": "..."}` |
| Malformed body or idempotency-key reused with diff body | `422 Unprocessable Entity` | `{"error": "validation_error", "details": [...]}` |
| Upstream (question-mgmt-svc) failure | `502 Bad Gateway` | `{"error": "upstream_unavailable"}` |
| Deadlock retries exhausted; DB connection lost | `503 Service Unavailable` | `{"error": "service_unavailable"}` with `Retry-After` |

The "401 vs 404" rule deserves a sentence: when the token is bad, return 401. When the token is valid for a *different* session than the URL, return 404 (don't reveal whether the URL's session_id exists). This is the same don't-leak-existence pattern the user-service uses for login attempts.

## Naming The Error Codes

The `"error"` field is a stable machine-readable string. Two reasons:

- **The frontend (Day 14) switches on this value** to choose the right UX (`session_expired` → redirect to results; `answer_already_submitted_for_question` → soft warning, refresh page; `unauthorized` → boot to login).
- **The log aggregator counts and alerts on these strings.** "Spike in `session_expired` after 14:00 UTC" is a real diagnostic signal.

Names follow snake_case, are stable across versions, and are listed in an `ERRORS.md` next to the OpenAPI spec. Add a new error code in the same PR that emits it; don't introduce new strings via prod logs.

## What Belongs In The Server vs The Client

The server's responsibilities (this file):

- Return the right status and stable error code for each scenario.
- Make the response replayable on retry (idempotency).
- Never corrupt state because of a client retry.

The client's responsibilities (Day 14):

- Generate a fresh `Idempotency-Key` per logical submission.
- Reuse the key on every retry of *that* submission.
- Switch on the `error` field to render the right UX.
- Show a sane error message even for `502`/`503` (don't say "Conflict" for "upstream down").

The Day-14 frontend builds against this exact table. Hand it over as the contract.

## What's Not In Scope Today

- **Auto-extend on network failure.** Some quiz systems give the candidate an extra 30 seconds if the submit fails due to network. PEP doesn't — the session expiry is firm. (If we wanted this, it'd be a server-side policy on the `submitted_at` check, not client-side.)
- **Per-question time limits.** The whole session has one TTL. Per-question timing is a future feature.
- **Anti-cheating heuristics** (DevTools detection, tab-switch counting, etc.). The server is authoritative for the rules it cares about; the client is not trusted, but it's also not surveilled.

Flagging these so PR review doesn't slip them in.

## Anti-Patterns

- **Returning 200 with `{"success": false, "reason": "..."}` for errors.** Use HTTP status codes. The infrastructure (load balancers, monitoring, retry logic in HTTP libraries) is built around them.
- **Returning 500 for any user-visible failure.** 500 means "the server has a bug." If you can predict the failure (expired session, bad token, conflict), it gets a 4xx and an error code.
- **Returning 409 for expired sessions.** 410 is precise; 409 invites a retry the client should not perform.
- **Returning 422 instead of 401 for "token missing".** Auth missing/bad is 401; validation failure on the body is 422. They mean different things to clients and to log alerting.
- **Returning different error strings for the same logical failure across endpoints.** Submit and answer endpoints should both return `session_expired`, not one `expired` and the other `session_timed_out`.
- **No `Retry-After` on 503.** Clients have no signal for when to retry. Set it to something sane (5–10 seconds).

## Key Takeaways
- The endpoint's behavior is specified by a **table of scenarios** mapping to specific HTTP responses; review against the table, not against vibes.
- Idempotency + unique constraint + pessimistic lock combine to make the three headline cases (timeout-retry, double-submit across tabs, expired submission) handle themselves.
- `200` for replays, `409` for real conflicts, `410` for expiry, `422` for validation, `401`/`404` for auth — each has a precise meaning; don't blur them.
- Error codes are stable snake_case strings; the frontend switches on them, the log aggregator alerts on them.
- "Auto-extend on network failure", "per-question timers", and anti-cheating heuristics are explicitly **out** of scope today.

---
*Prerequisites: [03-idempotency-for-retried-mutations.md](03-idempotency-for-retried-mutations.md), [04-unique-constraints-and-concurrent-write-race-conditions.md](04-unique-constraints-and-concurrent-write-race-conditions.md), [05-database-transactions-and-pessimistic-locking.md](05-database-transactions-and-pessimistic-locking.md), [06-state-finalization-and-immutability-patterns.md](06-state-finalization-and-immutability-patterns.md), [09-error-handling-and-http-status-code-discipline.md](../day-11/09-error-handling-and-http-status-code-discipline.md). Forward references: day-14 frontend error UX.*
