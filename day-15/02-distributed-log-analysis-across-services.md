# Distributed Log Analysis Across Services

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D10 wired `X-Request-Id` through the stack and demonstrated grep-by-id as the basic correlation move. Today we *use* that wiring to do real debugging, on a slice where failures can originate in three services and two data stores. Where D10 was "set up the plumbing," today is "you have the plumbing — now read the logs across services and localize a failure in under five minutes." This is a skill the cohort exercises with a guided fault-injection exercise: the trainer breaks one thing in `trainer/reference`, the cohort finds it from the logs.

## The Problem Shape

A `POST /sessions/{id}/submit` returns 500. The candidate stares at "Something went wrong." Without correlated logs you have no idea whether:

- the gateway never reached the downstream,
- the downstream reached Postgres but the transaction failed,
- the downstream reached Mongo but the question lookup failed,
- a middleware crashed before any business logic ran,
- the request timed out somewhere in the chain.

With correlated logs and a five-minute investigative discipline, you have an answer.

## The Standard Investigation Recipe

Always the same four steps. Drill this into the cohort by repetition.

### Step 1: Get The Request ID

From the browser network tab: every response carries `x-request-id` (D10's middleware echoes it on the way out). Copy it.

If the candidate didn't grab it before the page reloaded, grab from the *last failure* in the gateway logs:

```bash
docker compose logs --tail 50 api-gateway | grep -E '5\d\d|4\d\d'
```

Take the most recent failure's `request_id`.

### Step 2: Pull All Lines For That ID, Time-Sorted

```bash
RID="req_a8f3c2..."
docker compose logs --no-color --timestamps 2>&1 \
  | grep "${RID}" \
  | sort -k1,1
```

The `--timestamps` prefix is what makes `sort` work; without it, lines arrive in stream-arrival order which isn't necessarily time order.

What you see for a healthy submit:

```
2026-05-19T14:22:01.123 api-gateway       | {"level":"INFO","msg":"received POST /sessions/{id}/submit","request_id":"req_a8f3..."}
2026-05-19T14:22:01.130 api-gateway       | {"level":"INFO","msg":"forwarding to test-management-service","request_id":"req_a8f3..."}
2026-05-19T14:22:01.142 test-management-service | {"level":"INFO","msg":"received submit, session_id=sess_42","request_id":"req_a8f3..."}
2026-05-19T14:22:01.155 test-management-service | {"level":"INFO","msg":"acquired pg lock on session row","request_id":"req_a8f3..."}
2026-05-19T14:22:01.190 test-management-service | {"level":"INFO","msg":"scored 8/10","request_id":"req_a8f3..."}
2026-05-19T14:22:01.210 test-management-service | {"level":"INFO","msg":"committed submit, locked_at=...","request_id":"req_a8f3..."}
2026-05-19T14:22:01.215 api-gateway       | {"level":"INFO","msg":"200 OK","request_id":"req_a8f3..."}
```

The story reads top-to-bottom; each service hands off cleanly.

### Step 3: Find The Last Successful Line And The First Failing Line

For a *broken* submit:

```
2026-05-19T14:22:01.123 api-gateway       | received POST /sessions/{id}/submit
2026-05-19T14:22:01.142 test-management-service | received submit, session_id=sess_42
2026-05-19T14:22:01.155 test-management-service | acquired pg lock on session row
2026-05-19T14:22:01.180 test-management-service | ERROR: pymongo.errors.NetworkTimeout
2026-05-19T14:22:31.182 test-management-service | ERROR: rolling back transaction
2026-05-19T14:22:31.184 api-gateway       | 500 Internal Server Error
```

Last good: `acquired pg lock`. First bad: `pymongo.errors.NetworkTimeout`. The failure is *in test-management-service* trying to reach Mongo. The 30-second gap between the timeout and the rollback is the Mongo client's retry window. Conclusion: Mongo is unreachable or overloaded; investigate it next (Topic 5).

This step — finding the boundary between healthy and unhealthy in one trace — is the skill. Five minutes if the cohort is fluent.

### Step 4: Confirm Hypothesis With A Targeted Probe

Don't fix anything until you've confirmed. If you suspect Mongo:

```bash
docker compose exec test-management-service python -c \
  "import pymongo; pymongo.MongoClient('mongodb://mongo:27017', serverSelectionTimeoutMS=2000).admin.command('ping')"
```

Healthy: returns `{ok: 1.0}`. Failing: raises `ServerSelectionTimeoutError`, confirming reachability. Topic 5 covers in-network probes in depth.

## Common Slice-Specific Failure Signatures

Pattern-recognition speeds up Step 3. The cohort should learn these:

| Signature in correlated logs | Likely root cause |
|---|---|
| Gateway logs the request; no downstream ever logs it | Gateway route misconfigured (path mismatch, env var pointing to wrong host) |
| Downstream receives but logs `IntegrityError: duplicate key` on `idempotency_key` | D12 idempotency working as designed; second request returning cached response, but maybe the cache layer broken |
| Downstream logs `select... for update` then hangs 30s, then 500 | Postgres row lock contention from D12 — another in-flight submit on the same session |
| Downstream logs answer + 200, but session still shows `in_progress` | Submit handler ran the answer code path instead of the submit code path — routing bug in the handler |
| `request_id: "-"` on downstream logs while gateway has the id | Middleware order broke; RequestIdMiddleware moved below the handler |
| Frontend logs no `x-request-id` at all | Client-side fetch wrapper from D10 Step 5 regressed |

This table belongs in the cohort's wiki — it accelerates the next debug.

## Filtering By Service And Level

When grep on a request id is too narrow (e.g., investigating a class of failures over an hour, not a single request):

```bash
# Errors only, last 10 minutes
docker compose logs --no-color --since 10m 2>&1 | grep '"level":"ERROR"'

# One service, last 10 minutes
docker compose logs --no-color --since 10m test-management-service

# JSON-aware: by jq if your logs are JSON (D10's setup)
docker compose logs --no-color --since 10m 2>&1 | \
  jq -R 'fromjson? | select(.level=="ERROR")' 2>/dev/null
```

The `fromjson?` (with `?`) silently skips non-JSON lines, which is necessary because some container output (Postgres startup banner, Mongo init logs) isn't JSON.

## When Logs Are Insufficient

Logs tell you what happened in the application layer. They don't tell you:

- **Network-layer drops.** A SYN that never gets ACKed leaves no application log line. Use `docker network inspect` and in-network probes (Topic 5).
- **DB-internal contention.** Postgres won't log lock waits by default. `SELECT * FROM pg_stat_activity WHERE wait_event_type='Lock'` shows them live.
- **What the request payload actually was.** If the bug is "frontend sent the wrong body," logs probably don't include bodies (we don't log them by default — PII risk). Reproduce with `curl` and the same headers to see what the service receives.

Recognize the gap between "what the logs can tell me" and "what I need to know" — that recognition is when you reach for Topic 5's tools.

## Useful Aliases

The cohort will use these dozens of times today; put them in their shell rc files:

```bash
# Tail aligned logs for a request id
dl-rid() {
  docker compose logs --no-color --timestamps --tail 1000 2>&1 \
    | grep "$1" | sort -k1,1
}

# Tail one service's errors
dl-errs() {
  docker compose logs --no-color --since "${2:-5m}" "$1" 2>&1 \
    | grep '"level":"ERROR"'
}

# Most recent failed gateway request
dl-last-fail() {
  docker compose logs --tail 100 api-gateway 2>&1 \
    | grep -E '5\d\d|4\d\d' | tail -1
}
```

`dl-rid req_a8f3c2...` is the move you'll run constantly today.

## Anti-Patterns

- **Reading logs without a request id.** You're guessing which lines belong together. Always start by getting the id.
- **Reading only one service's logs.** "The frontend is broken" — maybe; maybe the gateway is dropping. Always pull all services for the id; it's three keystrokes.
- **Fixing before confirming.** A hypothesis from logs is a hypothesis; confirm with a probe (Step 4). Cohorts who fix-then-verify waste hours fixing the wrong thing.
- **Ignoring timestamps.** The order of events matters. Without `--timestamps`, stream order can lie. Always sort.
- **`docker compose logs -f` without filtering.** Firehose. Find the line you want with `--since 5m | grep ...` first; only follow if you're actively reproducing.
- **Stripping the id from logs to "clean them up."** The id is the *one* thing that makes distributed debugging tractable. Keep it everywhere.
- **Treating `request_id: "-"` as cosmetic.** It means the wiring rotted in that service. Fix the wiring before debugging anything else.

## The In-Class Drill

The trainer breaks one of these in `trainer/reference` (without telling the cohort which) and gives them 10 minutes to localize from logs:

1. Comment out `httpx.AsyncClient` `mongo` host name; change to `mongodb` (typo). Connection refused.
2. Remove `await session.commit()` from the submit handler. Inserts silently roll back.
3. Reorder middleware so `RequestIdMiddleware` runs *after* error handling. Errors lose the id.
4. Add `time.sleep(35)` to the answer handler. Gateway times out at 30s; downstream still running.

Pattern recognition is the goal. After three drills, the cohort can localize most slice failures without help.

## Key Takeaways
- The investigation recipe is always: get the request id → pull all lines for the id, sorted → find the last-good / first-bad boundary → confirm with a probe.
- D10's `X-Request-Id` wiring is the *only* thing that makes this tractable across services; treat broken wiring as priority-zero.
- Pattern recognition (the failure-signature table) accelerates the next debug; build the cohort wiki.
- Logs don't cover network-layer or DB-internal issues — that's Topic 5.
- Aliases in the shell rc file are a force multiplier; `dl-rid req_...` is the most-used command of the day.

---
*Prerequisites: [05-distributed-log-correlation-across-services.md](../day-10/05-distributed-log-correlation-across-services.md), [09-container-debugging-logs-exec-troubleshooting.md](../day-02/09-container-debugging-logs-exec-troubleshooting.md).*
