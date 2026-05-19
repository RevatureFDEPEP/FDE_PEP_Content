# Random Sampling Patterns from Document Stores

> *Day 11: Test Session Creation (Backend) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
When a candidate starts a session, we need to pick N questions from the bank — randomly, without replacement, weighted by neither client nor user. The naive answer ("load all questions and pick in Python") works at 50 questions and embarrasses you at 5,000. MongoDB's `$sample` aggregation stage and a few alternative patterns each have different trade-offs. This file covers `$sample` mechanics, what it does *not* guarantee, and when to reach for a pre-shuffled-id pattern in Postgres instead.

## The Default Tool: `$sample`

The `question-management-service` stores questions in MongoDB (see Day 8). The straightforward sample query:

```python
# services/question-management-service/app/repos/question_repo.py
async def sample_questions(
    db: AsyncIOMotorDatabase,
    quiz_id: str,
    count: int,
) -> list[dict]:
    pipeline = [
        {"$match": {"quiz_id": quiz_id, "active": True}},
        {"$sample": {"size": count}},
        {"$project": {"_id": 0, "id": 1, "stem": 1, "kind": 1, "choices": 1}},
    ]
    cursor = db.questions.aggregate(pipeline)
    return [doc async for doc in cursor]
```

The order matters: `$match` first to shrink the candidate set, then `$sample`, then `$project` to trim the payload over the wire.

## What `$sample` Actually Guarantees

`$sample` returns `size` documents from the input pipeline, **without replacement** within a single invocation, using a pseudo-random selection. That's the entire contract. Things it does **not** guarantee:

- **Uniformity across calls.** Different invocations may favor or disfavor particular documents over time. Don't use `$sample` for statistical sampling work.
- **Stability under concurrent writes.** Documents inserted during the aggregation may or may not be eligible.
- **Cost independent of collection size.** See below.

## The Cost Model

`$sample`'s implementation picks one of two strategies depending on `size` relative to collection size:

1. **Random cursor** (cheap) when `size` is small (≤ 5% of collection) and the documents come from a single collection (no `$match`/`$project` that would prevent it). It uses a pseudo-random cursor that touches roughly `size` documents.
2. **In-memory shuffle** (expensive) otherwise. It pulls candidate documents into a sort-style stage and shuffles. Has a 100MB in-memory limit unless `allowDiskUse=True`.

The kicker: **once you put a `$match` before `$sample`, the optimizer typically can't use the random-cursor path** for that match's output. You're paying the in-memory-shuffle cost over the filtered set. For a 5k-question bank with 500 matching a quiz, that's fine. For 500k with 50k matching, it's a noticeable hit and you should add `allowDiskUse=True`.

```python
cursor = db.questions.aggregate(pipeline, allowDiskUse=True)
```

## Quick Sanity Check

Time the aggregation against representative data:

```bash
mongosh "$MONGO_URL" --eval '
db.questions.aggregate([
  {$match: {quiz_id: "01HSY8...", active: true}},
  {$sample: {size: 10}}
], {explain: "executionStats"}).executionStats.executionTimeMillis'
```

A range under ~50ms for a single-quiz sample on the PEP substrate is fine. If you see hundreds of ms, the bank has grown and you may want the alternative pattern below.

## Alternative: Pre-Shuffled IDs in Postgres

`test-management-service` already owns the session record in Postgres. An alternative pattern co-locates the randomization with the session:

1. At session create, **list all candidate question IDs** for the quiz (cheap projection from Mongo — one round trip, IDs only).
2. **Shuffle in Python** using `random.SystemRandom().shuffle()`.
3. **Store the full shuffled list** as a Postgres array on the session row.
4. Each subsequent call (`GET /sessions/.../current`) reads the array by index.

```python
# services/test-management-service/app/services/session_service.py
import secrets

async def sample_questions(
    question_client: "QuestionClient",
    quiz_id: str,
    count: int,
) -> list[str]:
    all_ids = await question_client.list_active_question_ids(quiz_id=quiz_id)
    if len(all_ids) < count:
        raise InsufficientQuestionsError(
            quiz_id=quiz_id, available=len(all_ids), requested=count
        )
    # SystemRandom uses os.urandom — better than Mersenne Twister for anti-cheat
    rng = secrets.SystemRandom()
    rng.shuffle(all_ids)
    return all_ids[:count]
```

Trade-offs:

| | `$sample` in Mongo | Pre-shuffled in Postgres |
|---|---|---|
| Round trips at session create | 1 | 1 (id-list fetch) |
| Round trips per question fetch | 1 (cached) | 1 (cached) |
| Server-side cost at scale | grows with bank size | grows with bank size, but in Python where it's cheap |
| Predictability for tests | hard to seed | trivial to seed (`random.Random(42)`) |
| Anti-cheat (random source) | depends on Mongo build | `SystemRandom` is CSPRNG |
| Resumption after a process restart | requires re-sample | already persisted in Postgres |

For PEP, the **pre-shuffled-IDs pattern wins** because:

- The cohort runs 25 candidates concurrently — replays after a pod restart should be deterministic.
- The Day 12 scoring engine needs the exact same question order for `current_index`-based progression.
- It's testable without a Mongo fixture (seed the RNG).

Use `$sample` when you genuinely don't care about replayability — e.g., the trainer's "show me a random recent submission" widget in Week 4.

## Why Not `ORDER BY RANDOM() LIMIT N`?

Tempting in Postgres-native designs. It works but it's a full-table scan + sort for every call. On a 50k-row table that's a noticeable warm-CPU spike. Pre-shuffling in app code is cheaper and gives you the seedable RNG.

## Worked Scenario: Day 11's Choice

`POST /sessions` with `question_count=10` from a bank of 500 quiz-tagged questions:

1. `session_service` calls `question_client.list_active_question_ids(quiz_id=...)` — one HTTP call (topic 5), returns 500 ULIDs (~12KB).
2. `secrets.SystemRandom().shuffle(ids)` — microseconds.
3. Take first 10, store full shuffled list on the session record for future `current_index` lookups.
4. Call `question_client.fetch(ids[0])` for the first question to include in the response.

Two HTTP calls to question-management-service total at session create; everything else reads from the local session row.

## Statistical Aside: Sampling Without Replacement

Both `$sample` and Python's `random.shuffle` give you sampling without replacement *within one call*. Across calls, neither is guaranteed to be uniform — a question that's been in 12 prior sessions can still be in the next one. If you need "no question repeats for a user within a quiz," that's a separate concern: filter the candidate list by `question_id NOT IN (user's prior questions)` before shuffling. Don't try to encode it into the sampler itself.

## Anti-Patterns

- **Loading all questions into Python and slicing.** Fine at 50 questions, painful at 50k. Make the projection do the work.
- **Using `random` (Mersenne Twister) for question selection.** Seedable from observed outputs in theory; use `secrets.SystemRandom` or a properly-seeded `random.Random(seed)` only in tests.
- **Calling `$sample` per-question in a loop.** Five `{size: 1}` calls is ~5x the work of one `{size: 5}` call and gives you no de-duplication.
- **Forgetting `active: true` (or equivalent) in the `$match`.** Soft-deleted questions appear in sessions. Embarrassing.
- **Sampling at fetch time, not at create time.** Means the session has a different question set every time someone reopens it. The whole feature breaks.

## Key Takeaways
- `$sample` is fine for one-shot, low-stakes sampling; it does *not* guarantee uniformity across calls and an upstream `$match` pushes you onto the in-memory-shuffle path.
- For session creation, pre-shuffle question IDs in Python with `secrets.SystemRandom` and persist the full list on the session row — replayable, testable, resumable.
- Validate the bank has enough questions before shuffling; raise a domain error the route layer converts to a 409 or 422 (see error-handling topic).
- Avoid `ORDER BY RANDOM() LIMIT N` and per-question `$sample` loops — both scale badly.
- Keep `active: true` (or equivalent soft-delete flag) in every sample query.

---
*Prerequisites: day-8 question-authoring backend topics, day-11-cross-service-http-integration-httpx-rest-contracts.*
