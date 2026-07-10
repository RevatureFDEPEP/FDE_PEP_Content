# Vertical-Slice Integration in a Local Compose Environment

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D10 wired Week 2's authoring slice across `web → api-gateway → question-management-service → mongo + minio` and called it done when the integrated stack passed a smoke test. Today we repeat the *exact same pattern* against a slice with more moving parts: a candidate logs in (`user-service`), starts a session (`test-management-service` writes to Postgres, reads from Mongo), answers questions with the timer/autosave loop from D14, then submits to lock the attempt. Six containers must agree on contracts, ordering, and identity for a single happy-path click. The pattern is unchanged; the surface area is bigger.

Today is not "write more code." Today is "make the code from D11–D14 actually work together against a Compose stack that looks like prod."

## What's New Compared To D10

D10's slice touched two services (`api-gateway`, `question-management-service`) plus the Next.js frontend, persisting to Mongo + MinIO. Today's slice adds:

- **`user-service`** — must be up, must issue a token, must hand off context the gateway can forward (the D13 auth-context propagation work).
- **`test-management-service`** — owns `sessions` and `attempts` in Postgres (D11–D12) and reads question bodies from Mongo (cross-store join in application code).
- **State that changes during the run.** D10's question authoring was one POST per submit; today's session has many `POST /sessions/{id}/answer` calls happening as the candidate clicks, plus the autosave traffic from D14.
- **A time dimension.** The server-anchored timer (D14 Topic 4) means clock skew between client and server matters; Compose's container clocks are normally synced to the host, but it's the first slice that *cares*.

Same four-step delivery pattern from D10 — align contract, deploy in dependency order, smoke test, declare done — just more of each.

## The Deploy Order

`depends_on` with healthchecks (D2) enforces this; today is the first day the full order matters:

```
postgres + mongo + minio
        ↓
user-service + question-management-service + test-management-service
        ↓
api-gateway
        ↓
web (Next.js)
```

- Data stores first; nothing wakes up before its DB is healthy.
- The three FastAPI services come up in parallel — they don't directly depend on each other; they're all behind the gateway.
- `api-gateway` waits for all three downstreams' healthchecks.
- `web` waits for `api-gateway` (its first server-rendered page may hit the API).

If you bring the stack up in the wrong order locally (e.g., `docker compose up web` first), the first page request 502s and the trainer thinks the slice is broken. Insist on `docker compose up -d` against the full stack.

## Aligning The Contracts

Each contract was written in its source-day topic. Today we *check* them across the wire:

| Contract | Source day | What to verify in the integrated run |
|---|---|---|
| `POST /auth/login` → `{ token, user_id }` | (existing in `user-service`) | Frontend stores token; gateway forwards as `Authorization: Bearer` |
| `POST /sessions` → `{ session_id, expires_at, ... }` | D11 | Frontend reads `expires_at` and feeds it to the timer |
| `GET /sessions/{id}` → session + questions | D11/D13 | Server-side fetch in the take page returns the discriminated union the D13 renderer expects |
| `POST /sessions/{id}/answer` (idempotent) | D12, D14 | Idempotency-Key header from D14 round-trips; second submit of same key returns the same response |
| `POST /sessions/{id}/submit` → `{ locked_at }` | D12 | Frontend renders confirmation; subsequent answer POSTs get 409 |
| `X-Request-Id` header (correlation) | D10 | Same id flows through web → gateway → user-service / test-management-service |

If any one of these contracts has drifted since the producing-day PR merged, today is when you find out.

## The Integration Worked Example

Standard Compose reset + bring-up, mirrored on D10's pattern:

```bash
# Known state
docker compose down -v
docker compose up -d --build

# Wait
docker compose ps  # all (healthy)

# Backend smoke (Topic 2 covers depth)
SESSION=$(curl -s -X POST http://localhost:8080/sessions \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{"test_id":"...", "candidate_id":"..."}' | jq -r .session_id)

# Verify the session is in Postgres
docker compose exec postgres psql -U app test_mgmt \
  -c "SELECT session_id, status, expires_at FROM sessions WHERE session_id='${SESSION}';"

# Open the take page in a browser
open http://localhost:3000/take/${SESSION}
# Answer some questions, watch the timer tick down, watch autosave indicators flip,
# submit.

# Verify the attempt locked
docker compose exec postgres psql -U app test_mgmt \
  -c "SELECT status, locked_at FROM sessions WHERE session_id='${SESSION}';"
# Expect: status='submitted', locked_at not null.
```

Each step has a verifiable outcome. If a step fails, you know which boundary to look at.

## What "Done" Means For This Slice

The D10 checklist, restated for Week 3:

- [ ] `docker compose down -v && docker compose up -d --build` from a fresh checkout, full stack healthy.
- [ ] Candidate can log in, start a session, answer all questions, submit, see confirmation.
- [ ] Timer counts down from the server `expires_at`; doesn't drift more than a second over a 5-minute window.
- [ ] Autosave indicator flips `idle → saving → saved` on each answer change; survives a brief network outage (DevTools "Offline" toggle).
- [ ] Submit a second time → UI shows "already submitted" (409 path from D12/D14).
- [ ] Logs across all three FastAPI services share an `X-Request-Id` for a single click (D10 wiring still working).
- [ ] `trainer/reference` branch demos Week 3 end-to-end in one take.

Six checks; if any one is red the slice isn't done.

## Failure Modes To Expect Today

Because this is the integration day for the most complex slice in the course, things *will* break. The common ones:

- **Token not forwarded.** Frontend stores the token, gateway middleware strips or forgets `Authorization`. Symptom: every `POST /sessions/...` returns 401 even though login worked. Look at the gateway proxy code from D11.
- **Mongo/Postgres on different clocks.** Rare but happens on Windows + WSL2: container clocks drift from host. Symptom: timer says "5 minutes left" but server thinks the session expired. `docker compose exec test-management-service date` to compare.
- **Autosave race against submit.** Submit fires while an autosave for the last question is still in flight; the server processes them in the wrong order. The D14 "optimistic for autosave, confirmed for submit" pattern is the fix — verify it's actually in place.
- **`X-Request-Id` lost at the auth boundary.** A middleware ordering bug eats the header before it reaches the log filter. Symptom: gateway logs have the id, downstream logs say `request_id: "-"`. Reorder middleware (RequestIdMiddleware first).
- **Session list query in the take page joins Postgres+Mongo on the wrong key.** The session has `question_ids` (Mongo `_id`s), the question docs have `_id`. Easy to misspell either side. Symptom: 200 with `questions: []`.

The first time you see any of these, walk it down with the techniques from Topics 4 (log analysis) and 5 (advanced multi-container debugging) — don't guess.

## Why The Compose Environment Specifically

Same answer as D10: it's the cheapest production-shaped environment we have. Today reinforces it.

- All six processes talk over a real Docker network (not in-memory).
- Postgres and Mongo behave like Postgres and Mongo (not SQLite + a dict).
- The gateway actually proxies (not a function call).
- The browser actually loads JS, hydrates, and clicks (not jsdom).

Anything that works in Compose has a high probability of working in ECS. Anything that *doesn't* work in Compose definitely won't work in ECS.

## Anti-Patterns

- **Skipping the integration day because "all the PRs passed CI."** Unit tests passed in isolation. CI doesn't bring up the full stack with a real browser. Today does.
- **Integrating only the happy path.** Submit twice. Pull the network cable mid-quiz. Refresh during the timer. The slice is only done when the *predictable* breakages behave gracefully (D14 Topics 6, 7, 8).
- **Letting the `trainer/reference` branch lag.** If you don't update it today, Day 16 has nothing to fork from. Update it as the *last* step of the day.
- **Treating "the trainer's machine works" as the bar.** It must work from a fresh `git clone` + `docker compose up`. Have one trainee with no setup do exactly that — that's the real verification.
- **Big-bang integration of all four endpoints on Friday afternoon.** This is the W2 lesson the cohort should already have absorbed (D10 anti-patterns). If the slice wasn't integrated incrementally over D11–D14, today is harder than it needs to be.

## Key Takeaways
- The D10 four-step pattern (align contract → deploy in order → smoke test → declare done) reapplies unchanged; today's slice just touches more boxes.
- Six containers, two data stores, one timer, one token — all must agree. Each pairwise contract was written on its source day; today is the first time they're all checked at once.
- The deploy order (data → services → gateway → web) is enforced by `depends_on` + healthchecks, not by convention.
- "Done" is a six-item checklist, not a feeling. The `trainer/reference` Week 3 demo is the final verification.
- Common failure modes (auth not forwarded, clock drift, autosave race, request-id loss, cross-store join error) are predictable — name them in advance so the cohort recognizes them when they appear.

---
*Prerequisites: [06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md](../day-10/06-multi-service-feature-delivery-impact-mapping-and-cross-cutting-infrastructure-changes.md), [08-healthchecks-and-service-dependency-conditions.md](../day-02/08-healthchecks-and-service-dependency-conditions.md), [07-opaque-token-generation-and-session-identifiers.md](../day-11/07-opaque-token-generation-and-session-identifiers.md), [05-optimistic-updates-vs-server-confirmation.md](../day-14/05-optimistic-updates-vs-server-confirmation.md).*
