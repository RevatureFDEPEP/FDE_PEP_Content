# MongoDB Containerization

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
The `question-management-service` uses MongoDB because questions are variable-shape documents (multiple-choice, code, free-response) that resist a fixed relational schema. Today the goal is purely operational: get a local Mongo container running, persisting data, and reachable from `question-management-service` on the Compose network. Week 2 will dig into document modeling; today you only need the substrate working.

## The Official Mongo Image

The cohort uses `mongo:7` from Docker Hub. Like the Postgres image, it reads bootstrap environment variables at first start:

| Variable | Purpose |
|---|---|
| `MONGO_INITDB_ROOT_USERNAME` | Root user name (created on first start) |
| `MONGO_INITDB_ROOT_PASSWORD` | Root user password |
| `MONGO_INITDB_DATABASE` | Default database name (referenced by init scripts) |

It listens on port `27017` and stores data at `/data/db`. The pattern is intentionally parallel to Postgres — same shape, different paths.

## Compose Definition

```yaml
services:
  mongo:
    image: mongo:7
    environment:
      MONGO_INITDB_ROOT_USERNAME: app
      MONGO_INITDB_ROOT_PASSWORD: app
      MONGO_INITDB_DATABASE: questions
    volumes:
      - mongo_data:/data/db
      - ./infra/mongo/init:/docker-entrypoint-initdb.d:ro
    ports:
      - "27017:27017"

volumes:
  mongo_data:
```

The structure mirrors Postgres line-for-line — declare the volume at the top level, mount it at the data path inside the container, optionally mount init scripts. The PostgreSQL Containerization topic explains the named-volume mechanics in detail; the same rules apply here.

## Init Scripts — Slightly Different from Postgres

Mongo's `/docker-entrypoint-initdb.d/` accepts `.js` and `.sh` files, **not** `.sql`. Use JavaScript to create collections, indexes, or seed data:

```javascript
// infra/mongo/init/01-questions.js
db = db.getSiblingDB('questions');
db.createCollection('items');
db.items.createIndex({ slug: 1 }, { unique: true });
```

Same rule as Postgres: scripts run **once**, on first start with an empty data volume. To re-run, `docker compose down -v` and start fresh.

## Connection Strings

From `question-management-service` on the Compose network:

```
mongodb://app:app@mongo:27017/questions?authSource=admin
```

From your laptop:

```
mongodb://app:app@localhost:27017/questions?authSource=admin
```

The `authSource=admin` is the gotcha — when you set `MONGO_INITDB_ROOT_*`, the user is created in the `admin` database, not in the application database. Clients must say so explicitly.

## Example / Worked Scenario

Bring up just Mongo:

```bash
docker compose up -d mongo
docker compose logs mongo | tail -20
```

Expect `Waiting for connections` near the bottom. Now connect using the official `mongosh` shipped inside the image:

```bash
docker compose exec mongo mongosh -u app -p app --authenticationDatabase admin
```

Inside the shell:

```javascript
use questions
db.items.insertOne({ slug: 'smoke', type: 'mcq', prompt: 'pick one' })
db.items.find()
```

Restart to confirm persistence:

```bash
docker compose restart mongo
docker compose exec mongo mongosh -u app -p app --authenticationDatabase admin \
  --eval 'db.getSiblingDB("questions").items.find().toArray()'
```

The smoke doc should still be there. As with Postgres, `docker compose down -v` is the destructive command that wipes the volume.

From `question-management-service` (once it is up), the connection string in `environment:` should resolve `mongo` via Compose DNS:

```yaml
  question-management-service:
    environment:
      MONGO_URI: mongodb://app:app@mongo:27017/questions?authSource=admin
    depends_on:
      mongo:
        condition: service_healthy
```

## Common Pitfalls

- **Missing `authSource=admin`.** Clients connect, then fail authentication with a cryptic message because they tried to auth against the `questions` DB where the user does not exist.
- **Trying to run `.sql` init scripts.** Mongo's init mechanism accepts only `.js` and `.sh`. Mixing the two engines' init script formats is a frequent first-week mistake.
- **Mounting init scripts but volume already exists.** Same as Postgres — init only fires on an empty data dir. If you changed an init script and it "doesn't seem to work," `docker compose down -v` first.
- **Confusing the legacy `mongo` shell with `mongosh`.** Recent images ship `mongosh` (the modern shell); the old `mongo` binary is gone. Stack Overflow answers from before 2022 are misleading.

## Key Takeaways

- Mongo follows the same containerization pattern as Postgres: official image, root credentials via env, named volume mounted at `/data/db`, init scripts via `/docker-entrypoint-initdb.d/`.
- Init scripts are JavaScript (`.js`) or shell (`.sh`), not SQL.
- Connection strings need `?authSource=admin` because the root user lives in `admin`.
- Use `mongosh`, not `mongo`.

---
*Prerequisites: [05-postgresql-containerization.md](05-postgresql-containerization.md), [03-local-orchestration-with-docker-compose.md](03-local-orchestration-with-docker-compose.md)*
