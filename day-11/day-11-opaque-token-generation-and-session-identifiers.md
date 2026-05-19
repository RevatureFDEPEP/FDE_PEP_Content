# Opaque Token Generation and Session Identifiers

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
When `POST /sessions` succeeds today, the response carries two identifiers: a `session_id` (UUID) used in URLs and logs, and a `session_token` used for authentication on subsequent calls within the session. Both must be **opaque** — meaningless to the holder, impossible to guess, indistinguishable from any other token of the same shape. This file explains why opacity matters, how to generate identifiers correctly in Python, and where the `session_id` vs `session_token` line gets drawn.

## What "Opaque" Means

An opaque identifier:

- Carries **no business meaning** in its bytes (no embedded user id, no timestamp the client can read, no quiz id).
- Is **infeasible to guess** by enumeration or pattern (no sequential IDs, no monotonic counters).
- Looks the **same as every other identifier** of its class regardless of source, environment, or user.

A non-opaque identifier — say, `session-1234` — leaks the count of total sessions, lets an attacker probe `session-1233`, `session-1235`, etc., and tells anyone watching the URL bar what kind of system is behind it.

## Why Sequential IDs Are a Problem for Sessions

Imagine `POST /sessions` returns `{"session_id": 4271}`:

- **Enumeration:** an attacker scripts `GET /sessions/1..4271` and probes for IDOR bugs.
- **Information leak:** session number 4271 vs 4272 tells competitors how many candidates took the quiz today.
- **Cache poisoning surface:** small ID space, easy to fingerprint.
- **Collisions across environments:** dev's `session-12` and prod's `session-12` are different things but the same string in logs.

UUIDs (specifically UUIDv4 or v7) eliminate all four.

## UUIDv4 vs UUIDv7 — Which to Use

| | UUIDv4 | UUIDv7 |
|---|---|---|
| Source of bytes | random | timestamp prefix + random |
| Sort order | unordered | time-sortable |
| Index locality (DB) | poor (random across pages) | good (recent inserts cluster) |
| Information leak | none | leaks creation timestamp (ms precision) |
| Stdlib (3.12+) | `uuid.uuid4()` | not in stdlib yet — use `uuid_utils` or `uuid6` package |

For PEP, **UUIDv7** is the right call for the `session_id`:

- We *want* recent sessions to cluster in the index (the trainer dashboard in Week 4 reads "today's sessions" constantly).
- The timestamp leak is fine — `started_at` is already in the response.
- Sort-by-id approximates sort-by-creation-time without an extra index.

If you don't have a UUIDv7 library handy, UUIDv4 is the safe default and the substrate accepts it.

## Generating the `session_id`

```python
# app/services/session_service.py
import uuid

try:
    import uuid_utils  # pip install uuid-utils
    def new_session_id() -> uuid.UUID:
        return uuid.UUID(str(uuid_utils.uuid7()))
except ImportError:
    def new_session_id() -> uuid.UUID:
        return uuid.uuid4()
```

The fallback to v4 means a misconfigured dev box still works — it just loses the index-locality benefit.

## Generating the `session_token`

The `session_token` is the **bearer credential** for subsequent calls in the session. It must:

- Be cryptographically random (not derived from the session_id).
- Have enough entropy to resist brute-force (≥128 bits).
- Be safe to embed in headers and URLs.

Python's `secrets` module is the only correct choice:

```python
# app/services/session_service.py
import secrets

def new_session_token() -> str:
    # 32 bytes = 256 bits of entropy; URL-safe base64 ≈ 43 chars
    return "sess_" + secrets.token_urlsafe(32)
```

A few notes:

- **`secrets.token_urlsafe`**, not `random.choices` and not `uuid.uuid4().hex`. `random` is a Mersenne Twister — fast but predictable. `uuid.uuid4().hex` is technically fine but only 122 bits and looks like a UUID, which invites confusion with the session_id.
- **Prefix the token** (`sess_`, `pat_`, `sk_live_`, etc.). Cheap, and lets log scrubbers and secret scanners (e.g. GitHub's push protection) recognize the type at a glance.
- **Never log the raw token.** See the storage section below.

## `session_id` vs `session_token` — Two Things, Two Jobs

| | `session_id` | `session_token` |
|---|---|---|
| Purpose | identify the session | authenticate calls to it |
| Where it appears | URLs, logs, dashboards | `Authorization: Bearer` header |
| Sensitive? | no (opaque but not secret) | yes (treat like a password) |
| Stored as | plain | hash (sha256) |
| Returned to client | every response | exactly once, on create |
| Lifetime | forever (audit trail) | until session expires/submits |

The split mirrors the GitHub pattern: a PR has a number (`#1234`) and a PAT used to call its API. You'd never authenticate with the PR number, and you'd never put the PAT in the URL.

## Storing the Token as a Hash

```python
# app/services/session_service.py
import hashlib

def hash_session_token(raw: str) -> str:
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()
```

We persist `session_token_hash`, not `session_token`. On subsequent requests:

```python
async def authenticate_session(db, raw_token: str) -> SessionRecord | None:
    h = hash_session_token(raw_token)
    return await session_repo.get_by_token_hash(db, h)
```

Why hash a randomly-generated token? Defense in depth. If the DB dumps (backup, log leak, error message that includes a row) the dumped data can't be replayed. SHA-256 is fine here — no need for bcrypt/argon2 because the input is already 256 bits of random.

## Tying It Together in `create_session`

```python
# app/services/session_service.py
async def create_session(
    db: AsyncSession,
    user: CurrentUser,
    request: SessionCreateRequest,
) -> SessionCreateResponse:
    session_id = new_session_id()
    raw_token = new_session_token()
    token_hash = hash_session_token(raw_token)

    question_ids = await question_client.sample_questions(  # see topic 4 + topic 5
        quiz_id=request.quiz_id,
        count=request.question_count,
    )
    now = datetime.now(timezone.utc)  # server-authoritative — see topic 6

    record = SessionRecord(
        session_id=session_id,
        session_token_hash=token_hash,
        user_id=user.id,
        quiz_id=request.quiz_id,
        mode=request.mode,
        question_ids=question_ids,
        current_index=0,
        started_at=now,
        expires_at=now + SESSION_TTL,  # configurable
        submitted_at=None,
    )
    await session_repo.insert(db, record)

    first = await question_client.fetch(question_ids[0])
    return to_create_response(record, raw_token, first, now)
```

The raw token escapes the function exactly once — in the response object. After that, only the hash exists server-side.

## Logging Discipline

```python
# Good
log.info("session.created", session_id=str(session_id), user_id=user.id)

# Bad — leaks the credential into log aggregation
log.info("session.created", session_id=str(session_id), token=raw_token)
```

Pair this with a log scrubber that redacts strings matching `sess_[A-Za-z0-9_-]{40,}` as a safety net.

## Worked Scenario: A Curl Test

```bash
# Create a session
curl -s -X POST http://localhost:8003/sessions \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"quiz_id":"01HSY8X4ZJN0...","question_count":5,"mode":"graded"}' | jq

# Response (abbreviated)
# {
#   "session_id":   "0193d3a4-1234-7abc-9def-0123456789ab",
#   "session_token":"sess_oo3...redacted...",
#   ...
# }

# Use the session token on subsequent calls (Day 12+)
SESSION_TOKEN=sess_oo3...
curl -s http://localhost:8003/sessions/0193d3a4-.../current \
  -H "X-Session-Token: $SESSION_TOKEN"
```

Notice we use `X-Session-Token` for the session credential, separate from the `Authorization: Bearer` user JWT. Two different auth contexts, two different headers — no ambiguity.

## Anti-Patterns

- **`session_id = id(some_object)`** — Python id, not unique across runs, leaks pointer info, no.
- **Using the same value for `session_id` and `session_token`.** Forces you to either treat the ID as secret (logs become a hazard) or treat the token as public (auth is broken).
- **`random.choice` for token generation.** Not cryptographically random.
- **Storing the raw token "just for debugging".** It's not for debugging; it's for impersonation. Always hash.
- **Returning the token in every response.** It's a `create`-only field. Subsequent endpoints take it via header and don't echo it.

## Key Takeaways
- Session IDs are opaque (no enumeration, no embedded meaning); UUIDv7 if possible, v4 as a safe fallback.
- Session **id** identifies; session **token** authenticates — keep them separate and prefix the token (`sess_`).
- Generate tokens with `secrets.token_urlsafe(32)`, never `random` or `uuid.hex`.
- Store the SHA-256 hash of the token, return the raw value exactly once, accept the raw value on subsequent calls and hash-compare.
- Never log the raw token; scrub it as a safety net.

---
*Prerequisites: day-8 backend topics, day-11-pydantic-request-response-modeling.*
