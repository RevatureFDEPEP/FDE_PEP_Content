# Server-Authoritative State — Why Time and Authority Live on the Server

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
This is the single most important concept in Week 3. The whole quiz-taking slice — session creation today, attempt locking on Day 12, the visible countdown timer on Day 14 — only works if the **server** owns the answers to "what time is it?", "did this session expire?", "is this answer accepted?", and "what's the next question?" The client gets to *display* state, not *determine* it. Every junior-engineer instinct says "compute expiry on the client and tell the server" because it's simpler; every production incident report about cheating, drift, and replay attacks says "don't." This file establishes the principle and ties it forward to the Week 3 deliverables it constrains.

## The Principle in One Sentence

If a state value matters for correctness, authorization, or trust, the **server** computes it, **persists** it, and the **client** displays a copy that may be stale.

## Why "Just Trust the Client" Fails

A counter-example. Imagine `POST /sessions` returned `{"expires_at_ms": now + 30*60*1000}` computed *in the browser*, and `POST /sessions/{id}/submit` accepted any submission tagged with `client_timestamp_ms < expires_at_ms`. Failure modes:

- **Clock drift.** The candidate's laptop is 6 minutes ahead of the server. The session "expires" 6 minutes early. Support ticket.
- **Clock drift, the other direction.** Laptop is 6 minutes behind. Candidate gets an extra 6 minutes for free.
- **Devtools.** Candidate opens devtools, edits `expires_at_ms`, gets infinite time.
- **`Date.now()` mocking.** Browser extensions or `Performance.now()` jitter.
- **Tab suspension.** Background tab's clock pauses on some platforms; resumes "in the past."
- **Different timezones in the same cohort.** The bug shows up only for the Manila candidate.

None of these are exotic. The fix isn't "validate harder"; it's "don't trust the client for authoritative time at all."

## The Correct Shape

```python
# app/services/session_service.py
from datetime import datetime, timedelta, timezone

SESSION_TTL = timedelta(minutes=30)


def server_now() -> datetime:
    return datetime.now(timezone.utc)


async def create_session(...) -> SessionCreateResponse:
    now = server_now()
    expires_at = now + SESSION_TTL

    record = SessionRecord(
        ...,
        started_at=now,
        expires_at=expires_at,
        submitted_at=None,
    )
    await session_repo.insert(db, record)
    return SessionCreateResponse(
        ...,
        started_at=now,
        expires_at=expires_at,
        server_now=now,           # for client display offset
    )
```

The client receives `server_now`, `started_at`, and `expires_at` *all from the server*. The client can:

- Compute `display_remaining = expires_at - (client_now - offset)` where `offset = client_now_when_response_received - server_now`.
- Render a ticking countdown.
- **Show the user** "time's up — submitting…".

The client cannot:

- Decide whether the submission is *accepted*. The server checks on `POST /sessions/{id}/submit` whether `server_now() <= expires_at` and rejects with a 410 Gone otherwise.

## The Two Sides of the Timer

There are two clocks. They are not the same thing and must not be confused:

| | UX clock (client) | Authority clock (server) |
|---|---|---|
| Purpose | Show the candidate a countdown | Decide what's accepted |
| Allowed to be wrong by | A few seconds | Zero |
| Lives in | React state, refreshed from a ref | The DB row `expires_at` |
| Re-checked on | Animation frame | Every mutation request |

Day 14 builds the UX clock. Day 12 enforces the authority clock during answer mutations. Today (Day 11) we *establish* both by writing `expires_at` server-side and returning `server_now` for the client's offset calculation.

## "Authority" Beyond Time

The same principle applies to anything the server must decide:

- **Which question comes next?** The server holds `question_ids` and `current_index`. The client says `POST /sessions/{id}/answer`, the server returns `next_question`. The client does **not** index into a local array and ask for "question 5" — that's how candidates cheat by skipping ahead.
- **Is this answer correct?** Server-only. The client never sees correct-answer flags (see Day 8's `QuestionPublic`). Scoring lives in Day 12's engine, computed at submit time.
- **Can this user start a session for this quiz?** Authorization is server-side from the JWT, not from a `?is_admin=true` query param.
- **Is the session locked because another tab is also taking it?** Day 12's attempt-locking topic. The lock state lives in the DB, not in localStorage.

The PEP rule: **if the client can edit it in devtools, you cannot use it for an authoritative decision.**

## Forward Links

Today's `expires_at` and `current_index` are the seeds for two later concepts:

- **Day 12 — Attempt locking.** When a user has two tabs open, both posting answers, we need an optimistic-concurrency check on the session row. The lock token is server-issued; the client carries it but cannot mint it. Without today's server-authoritative session shape, there'd be nothing to lock.
- **Day 14 — Visible timer.** The Next.js component computes the displayed countdown from `server_now` + `expires_at`. The component itself never decides "time's up, submit"; it triggers a submit and the server is free to accept or 410.

If the cohort gets the principle wrong today, both downstream features become impossible to implement without rework.

## Worked Scenario: Reviewing a PR

A candidate's PR adds:

```python
# bad
class SessionCreateRequest(BaseModel):
    quiz_id: str
    expires_at: datetime    # client-supplied
```

The reviewer's response: "Why does the client send `expires_at`? The server should compute it from a fixed TTL." Push back, link this topic, ship the corrected version. This is the *exact* PR pattern to expect in this slice.

A subtler version:

```python
# also bad
@router.post("/sessions/{session_id}/answer")
async def answer(
    session_id: UUID,
    body: AnswerSubmit,
    db: DB, user: CurrentUserDep,
):
    session = await session_repo.get(db, session_id)
    # The check is here, good.
    if body.client_now > session.expires_at:
        raise HTTPException(410)
    # ... store answer
```

The check exists, but it uses `body.client_now`. A devtools edit and the check is meaningless. The right version uses `server_now()`:

```python
if server_now() > session.expires_at:
    raise HTTPException(410, "session_expired")
```

## A Note on `datetime.now()` vs `time.monotonic()`

Use `datetime.now(timezone.utc)` for **wall-clock** timestamps stored in the DB and exposed to clients. Use `time.monotonic()` only for **measuring durations within a single process** (e.g. "this DB query took 12ms"). They aren't interchangeable: monotonic time has no calendar reference and can't be compared across processes.

## Anti-Patterns

- **Computing `expires_at` in the browser.** Trivially bypassable.
- **Accepting `client_timestamp` in any mutation endpoint as authoritative.** The server's clock is the only one that matters.
- **Echoing `is_admin` (or any authz claim) from the request body.** Authority comes from the verified JWT, not the request body.
- **Putting the timer's "is time up?" check on the client only.** The client may not even be running (tab closed, laptop asleep) — the server must enforce on the mutation.
- **Returning `now` from a separate `/time` endpoint and asking the client to poll it.** Just include `server_now` in the response of the call the client already had to make.

## Key Takeaways
- Authoritative state — time, authorization, scoring, progression — lives on the server, full stop.
- The client gets a snapshot to display; it never gets the right to decide.
- Today: compute `started_at` and `expires_at` server-side at session create, return `server_now` so the client can render a countdown with a known offset.
- Every mutation in Week 3 re-checks the authoritative state on the server — never trusts the body's `client_now`.
- Forward links: Day 12 attempt locking and Day 14 visible timer both *depend* on today's server-authoritative session shape.

---
*Prerequisites: day-11-opaque-token-generation-and-session-identifiers, day-11-pydantic-request-response-modeling. Forward references: day-12 attempt locking, day-14 visible timer.*
