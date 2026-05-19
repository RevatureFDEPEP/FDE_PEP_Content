# PostgreSQL Containerization

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
The `user-service` and Week 4's reporting service both depend on a local Postgres instance. Containerizing it correctly — with a **named volume** for persistence, an init script for the schema, and the right environment variables — is the difference between a stack you can iterate on and one that loses your data every `docker compose down`. This topic establishes the canonical pattern; Mongo and MinIO follow the same shape.

## The Official Postgres Image

The cohort uses `postgres:16-alpine` from Docker Hub. It is well-known, small (~80 MB), and reads three environment variables at first start to bootstrap the cluster:

| Variable | Purpose |
|---|---|
| `POSTGRES_USER` | Superuser name (created on first start) |
| `POSTGRES_PASSWORD` | Superuser password (required) |
| `POSTGRES_DB` | Database to create on first start |

It exposes port `5432` and stores data in `/var/lib/postgresql/data` inside the container. That second fact is what drives the volume design.

## Named Volumes — Why and How

Recall from Docker fundamentals: anything written to a container's filesystem dies when the container is removed. For a database that is unacceptable. The fix is a **named volume**, a Docker-managed storage object that outlives any individual container and can be mounted into one.

In Compose:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: users
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

Three things to note:

1. The top-level `volumes:` block **declares** the volume; the service-level `volumes:` block **mounts** it.
2. The mount target (`/var/lib/postgresql/data`) is the path Postgres writes to inside the container. The volume name (`postgres_data`) is opaque to Postgres.
3. `docker compose down` keeps the volume. `docker compose down -v` destroys it. Memorize this distinction; people lose data to it weekly.

To inspect the volume:

```bash
docker volume ls
docker volume inspect <project>_postgres_data
```

## Init Scripts — Bootstrapping the Schema

The official image runs any `.sql` or `.sh` files mounted into `/docker-entrypoint-initdb.d/` **on first start only** (i.e., when the data directory is empty). The substrate uses this for schema seeds:

```yaml
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infra/postgres/init:/docker-entrypoint-initdb.d:ro
```

A file like `infra/postgres/init/01-users.sql` then runs the first time the volume is created. If you change it later, you must `docker compose down -v` to wipe the volume and re-trigger the init — there is no smart re-run. For ongoing schema changes the substrate uses Alembic from inside `user-service` (covered later in the week); init scripts are strictly for the initial bootstrap.

## Connection Strings — The Compose Pattern

From `user-service` (running on the Compose network):

```
postgresql://app:app@postgres:5432/users
```

From your laptop (host):

```
postgresql://app:app@localhost:5432/users
```

Same database, two different hostnames, because of the ports mapping covered in the orchestration topic. Configure services with the Compose-internal form; use the host form only for ad-hoc `psql` / GUI debugging.

## Example / Worked Scenario

Bring up just Postgres and verify end-to-end:

```bash
docker compose up -d postgres
docker compose ps postgres
docker compose logs postgres | tail -20
```

You should see lines like `database system is ready to accept connections`. Now connect from the host:

```bash
docker compose exec postgres psql -U app -d users -c "\dt"
```

Expected: a list of tables created by the init script (or "Did not find any relations" if no init scripts ran — check that `./infra/postgres/init` exists and has SQL files).

Prove persistence by writing data, restarting, and reading it back:

```bash
docker compose exec postgres psql -U app -d users -c "CREATE TABLE smoke(id int); INSERT INTO smoke VALUES (1);"
docker compose restart postgres
docker compose exec postgres psql -U app -d users -c "SELECT * FROM smoke;"
# -> 1 row
```

Now prove the destructive path:

```bash
docker compose down -v
docker compose up -d postgres
docker compose exec postgres psql -U app -d users -c "SELECT * FROM smoke;"
# -> ERROR: relation "smoke" does not exist
```

That asymmetry — `down` keeps data, `down -v` destroys it — is the entire mental model.

## Common Pitfalls

- **Bind-mounting `./data` instead of a named volume.** Bind mounts use host filesystem semantics; on Windows or macOS with Docker Desktop, Postgres can misbehave (permission and fsync issues). Stick to named volumes.
- **Editing init scripts and expecting them to re-run.** They only run on an empty data dir. `docker compose down -v` (which destroys data) is the only way to re-trigger them.
- **Putting secrets in the Compose file.** `POSTGRES_PASSWORD: app` is fine for local; production uses RDS credentials from AWS Secrets Manager (Week 3). Don't drift those worlds.
- **Forgetting `condition: service_healthy` on dependents.** Without it, `user-service` can start while Postgres is still initializing and crash on the first query. The healthchecks topic covers the fix.

## Key Takeaways

- Postgres state lives in a **named volume** mounted at `/var/lib/postgresql/data`; without it, `down` wipes your database.
- `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` bootstrap the cluster on first start.
- Init scripts in `/docker-entrypoint-initdb.d/` run **once** when the volume is empty.
- Services connect via `postgres:5432`; you connect from the host via `localhost:5432`.

---
*Prerequisites: day-2-local-orchestration-with-docker-compose.md, day-2-docker-fundamentals-images-containers-layers-registries.md*
