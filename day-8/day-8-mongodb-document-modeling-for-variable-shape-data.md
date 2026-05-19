# MongoDB Document Modeling for Variable-Shape Data

> *Day 8: Question Authoring Slice (Backend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview

The substrate already chose Mongo for `question-management-service` — and now you'll see why. Question documents have a *variable shape*: a single-select looks similar to a multi-select but differs in correctness semantics, the option list is nested and variable-length, and future types (drag-and-drop, code questions) will add fields. Forcing this into Postgres means either a join-soup of `questions / options / question_metadata` tables or a `jsonb` column that gives up the schema you wanted from a relational store.

Mongo's natural unit is the *document* — the whole question, options inline, ready to read or write in one operation. Today you'll model it deliberately, place the right indexes, and write queries the API actually needs.

## Why Mongo, not Postgres, for this service

A clean argument in three points:

1. **The aggregate is the document.** A question is *always* read and written together with its options. You never want "the question without its options". Postgres encourages splitting that into two tables (because relational normalization rewards it); Mongo encourages keeping it as one document (because the storage model rewards it). For this aggregate, Mongo's preference matches the application's preference.

2. **Polymorphic schemas without `NULL`-soup or `jsonb` opacity.** Postgres tables either have nullable columns for every variant-specific field (ugly, ambiguous) or a `jsonb` column (then you're using Postgres as a worse Mongo). Mongo just stores the document as Pydantic produces it.

3. **Read patterns are document-shaped.** "Fetch this question to display in the authoring form" is one document. "Build a quiz from N tagged questions" is `find({tags: ...}).limit(N)`. There are no joins to write. Other services in the substrate (user-service, test-management) live in Postgres because *their* aggregates are relational. Pick the store per service, not per program.

The other PEP services that need joins, transactions across rows, or strict referential integrity stay in Postgres. Mongo is right *here* because the data is right for it.

## The document shape

The wire shape from Topic 1 maps directly to a Mongo document. The conventions:

```jsonc
// questions collection — one document per question
{
  "_id": "01J9ZQ8T7K3RXVE0WJPK4M2Q5A",          // ULID string (sortable, URL-safe)
  "type": "multi_select",                          // discriminator (Topic 1)
  "stem": "Which of the following are HTTP idempotent methods?",
  "difficulty": "medium",
  "tags": ["http", "rest"],
  "options": [
    {"id": "a", "text": "GET",    "is_correct": true},
    {"id": "b", "text": "POST",   "is_correct": false},
    {"id": "c", "text": "PUT",    "is_correct": true},
    {"id": "d", "text": "DELETE", "is_correct": true}
  ],
  "image_key": null,                               // MinIO key, Topic 5
  "author_id": "u-7f8a",                           // upstream from user-service
  "created_at": ISODate("2026-05-19T14:21:00Z"),
  "updated_at": ISODate("2026-05-19T14:21:00Z"),
  "schema_version": 1                              // see migrations note below
}
```

A few deliberate choices:

- **`_id` is a ULID string, not an ObjectId.** ULIDs sort lexically by time (useful for cursors), are URL-safe (no quoting), and travel cleanly through JSON without the `$oid` wrapper. Mongo accepts any `_id` value; you don't have to take the default.
- **`options` is embedded, not referenced.** They're owned by the question and have no life of their own.
- **`schema_version: 1`** lets you migrate. When you eventually add `true_false`, bump to 2 and write a one-shot migration script.
- **`tags` is an array** — Mongo indexes arrays natively (multikey index), so `find({tags: "http"})` is fast.

## Indexes you actually need today

Indexes are the one thing you *do* design up front in Mongo; query planners are forgiving but a collection scan on 50k questions is not.

```python
# services/question-management-service/app/db/indexes.py
from motor.motor_asyncio import AsyncIOMotorCollection

async def ensure_indexes(coll: AsyncIOMotorCollection) -> None:
    await coll.create_index("type")                          # filter by type
    await coll.create_index("tags")                          # multikey on the array
    await coll.create_index("author_id")                     # "my questions" view
    await coll.create_index([("created_at", -1)])            # newest-first listings
    await coll.create_index(
        [("tags", 1), ("difficulty", 1)],                    # quiz-build filter
    )
```

A few notes:

- **Don't index everything.** Each index costs write throughput and disk; index for queries you actually run.
- **The compound `(tags, difficulty)` covers the quiz-build query.** Day 11 builds quizzes by tag + difficulty; this is the index for that.
- **Call `ensure_indexes` at app startup,** typically from a FastAPI lifespan (Topic 4). It's idempotent — Mongo will only create what's missing.

## The repository pattern (Motor + Pydantic)

Keep Mongo confined to a repository module; routes never touch the driver directly. This is what testability and the Day 9 frontend's contract both depend on.

```python
# services/question-management-service/app/db/questions_repo.py
from datetime import datetime, timezone
from typing import AsyncIterator
from motor.motor_asyncio import AsyncIOMotorCollection
from ulid import ULID
from app.models.question import Question

class QuestionsRepository:
    def __init__(self, coll: AsyncIOMotorCollection) -> None:
        self._coll = coll

    async def insert(self, q: Question) -> Question:
        now = datetime.now(timezone.utc)
        doc = q.model_dump()
        doc["_id"] = str(ULID())
        doc["created_at"] = now
        doc["updated_at"] = now
        doc["schema_version"] = 1
        await self._coll.insert_one(doc)
        return Question.model_validate(doc)

    async def get(self, qid: str) -> Question | None:
        doc = await self._coll.find_one({"_id": qid})
        return Question.model_validate(doc) if doc else None

    async def list_by_tag(self, tag: str, limit: int = 50) -> list[Question]:
        cursor = self._coll.find({"tags": tag}).sort("created_at", -1).limit(limit)
        return [Question.model_validate(d) async for d in cursor]

    async def delete(self, qid: str) -> bool:
        result = await self._coll.delete_one({"_id": qid})
        return result.deleted_count == 1
```

`Question.model_validate(doc)` re-runs the validators (Topic 2) on the dict from Mongo — defensive but cheap, and it means a corrupted document raises clearly rather than corrupting a downstream response.

## What about migrations?

In Mongo, "migration" usually means *back-fill*, not *DDL*. When you add `true_false` and bump to `schema_version: 2`:

1. Update the Pydantic models to accept both versions on read (a discriminator-with-default, or branch in `model_validate`).
2. Write a one-shot script that reads `schema_version: 1` docs, transforms, writes `schema_version: 2`.
3. Once the back-fill is done, drop the v1 compatibility on read.

For the PEP cohort, you won't run a real migration — but recognising that the `schema_version` field is what makes it *possible* later is the lesson.

## Example / Worked Scenario

The trainer creates a multi-select question, then the quiz-build query (Day 11) needs to find it by tag. Walk the path:

1. `POST /questions` arrives; the validated `MultiSelectQuestion` from Topic 1+2 reaches `QuestionsRepository.insert`.
2. `model_dump()` produces a JSON-compatible dict; the repo adds `_id`, timestamps, and `schema_version`.
3. `insert_one` writes the document. The `tags` index updates (multikey) — `["http", "rest"]` becomes two entries in the index pointing at this doc.
4. Day 11 runs `find({tags: "http", difficulty: "medium"}).limit(20)`. The planner uses the compound `(tags, difficulty)` index — an IXSCAN, not a COLLSCAN, even at 50k documents.
5. Each returned doc is fed to `Question.model_validate(...)`; the discriminator routes to `MultiSelectQuestion`; the validators re-confirm the invariants; the API returns clean polymorphic JSON.

The whole loop, end-to-end, is one collection, one repository, no joins.

## Common Pitfalls

- **Treating Mongo like a JSON blob store.** Documents have shape — design them. Embed what's owned, reference what's shared. `_id`, indexes, and `schema_version` are deliberate decisions.
- **Embedding things that should be referenced.** If `author_id` resolved to a full embedded `author` document, you'd duplicate the user's profile across every question and have a stale-data problem the moment the user renames themselves. Reference (`author_id` only), don't embed.
- **No indexes on filter fields.** A collection scan looks fast at 100 documents and miserable at 100k. Index every field you `.find({...})` on in production.
- **Letting routes touch Motor directly.** Once two endpoints both call `db.questions.find(...)`, you have two versions of the same query drifting. Repository pattern keeps the queries — and the index assumptions — in one place.
- **Forgetting to `await ensure_indexes` at startup.** Indexes that don't exist in dev don't exist in prod either. Make their creation part of the app's lifespan, not a manual `mongo` shell step.

## Key Takeaways

- Mongo fits this service because the question + options aggregate is the natural document — one read, one write, no joins.
- Other PEP services stay in Postgres because their aggregates are relational; pick the store per service.
- Design the document deliberately: `_id` (ULID), embedded `options`, `tags` array, `schema_version` for future migrations.
- Index for queries you actually run; the compound `(tags, difficulty)` index is the workhorse for Day 11's quiz-build query.
- Keep Motor inside a repository module; routes import the repository, not the driver. Tests can swap a fake.
- `Question.model_validate(doc)` on read re-runs validators — defensive, cheap, and worth it.

---
*Prerequisites: `day-8-pydantic-discriminated-unions-for-polymorphic-schemas`, `day-8-schema-validation-patterns`, Day 2 (MongoDB containerization).*
