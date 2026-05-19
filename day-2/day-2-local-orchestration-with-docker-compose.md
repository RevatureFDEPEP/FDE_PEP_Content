# Local Orchestration with docker-compose

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
`docker-compose.yml` is the single file that wires Day 2's deliverable together: Postgres + Mongo + MinIO + four backend services + the Next.js frontend, all reachable from each other by name. Once you can read and modify this file, the rest of the day's containerization topics slot into it. This is also the file CI will eventually use, the file Day 6 extends with a reverse proxy, and the file you will edit more than any other in Week 1.

## The Compose Mental Model

A Compose file declares a **project** — a set of services, networks, and volumes that come up and go down together. The default invocation:

```bash
docker compose up        # foreground, with logs
docker compose up -d     # detached
docker compose down      # stop and remove containers + default network
docker compose down -v   # also remove named volumes (destroys data)
```

Each top-level `services:` entry is one container (or a scaled set, which we do not use here). Compose:

1. Creates a default **bridge network** named `<project>_default`.
2. Attaches every service to that network.
3. Registers each service name as a **DNS name** on that network — so `user-service` resolves to the container running `user-service`, no IP juggling required.

This DNS behavior is the single most important thing to internalize today: services do not talk to `localhost`; they talk to each other by **service name**.

## Service Definitions — The Anatomy

A typical service block from the substrate looks like:

```yaml
services:
  user-service:
    build:
      context: ./services/user-service
      target: runtime
    image: rev-eval-ai/user-service:dev
    container_name: user-service
    environment:
      DATABASE_URL: postgresql://app:app@postgres:5432/users
      JWT_SECRET: dev-only-not-a-real-secret
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - default
```

Read it field by field:

- **`build:`** — Compose will run `docker build` for you. `context:` is the directory passed to the build (everything in it becomes part of the build context, so a good `.dockerignore` matters). `target:` selects the multi-stage build's final stage.
- **`image:`** — The tag to apply to the built image. If you specify `image:` without `build:`, Compose pulls from a registry instead. The two together give you a local-build with a stable name.
- **`environment:`** — Environment variables injected at container start. Notice `postgres` as a hostname — that is Compose DNS doing its job.
- **`ports:`** — Maps host port to container port (`host:container`). The host side is how *you* reach the service from your laptop; the container side is what the process listens on.
- **`depends_on:` with `condition: service_healthy`** — Wait for another service's healthcheck to pass before starting this one. The healthchecks topic covers this in detail.

### `build:` vs `image:` — when to use which

| Goal | What to use |
|---|---|
| Run a third-party image (Postgres, Mongo, MinIO) | `image:` only |
| Build your own service from a Dockerfile | `build:` (plus `image:` to tag it locally) |
| Pull a prebuilt image from ECR in CI | `image:` only |

For Day 2, every first-party service uses `build:`; every infrastructure dependency uses `image:`.

## Compose-Internal DNS — The Most Important Detail

When `api-gateway` needs to call `user-service`, the URL is **`http://user-service:8000`**, not `http://localhost:8000`. From the host (your laptop), it is `http://localhost:8000` because of the `ports:` mapping. These two address spaces are easy to confuse:

```
Host (your laptop)        Compose network (inside Docker)
---------------------     -------------------------------
localhost:8000     -->    user-service:8000
localhost:5432     -->    postgres:5432
localhost:9000     -->    minio:9000
```

Inside any container on the Compose network, use service names. From your laptop's browser or `curl`, use `localhost:<host-port>`.

## Example / Worked Scenario

Bring up just two services to see the pattern in isolation:

```bash
docker compose up -d postgres user-service
docker compose ps
docker compose logs -f user-service
```

Now exec into `user-service` and prove Compose DNS works:

```bash
docker compose exec user-service bash
# inside the container:
getent hosts postgres        # resolves to a container IP
python -c "import psycopg; psycopg.connect('postgresql://app:app@postgres:5432/users')"
```

Then prove the host-side mapping:

```bash
# from your laptop, NOT inside the container:
curl http://localhost:8000/health
psql postgresql://app:app@localhost:5432/users -c '\dt'
```

Both work because `ports:` published the container ports to the host, and Compose DNS made `postgres` resolvable from inside `user-service`.

To make a one-line modification — say, add a new env var to `user-service`:

```yaml
    environment:
      DATABASE_URL: postgresql://app:app@postgres:5432/users
      JWT_SECRET: dev-only-not-a-real-secret
      LOG_LEVEL: DEBUG          # <-- added
```

Then:

```bash
docker compose up -d user-service    # recreates just that service
```

Compose detects the config drift and recreates only the affected container.

## Common Pitfalls

- **Using `localhost` between services.** A service calling `http://localhost:5432` from inside its container hits *itself*, not Postgres. Use service names.
- **Forgetting `target:` in `build:`.** Without it, Compose builds the final stage of the Dockerfile by default — which is usually fine, but if you copy a Dockerfile that has a `dev` stage on top, you may unexpectedly ship dev tooling. Be explicit.
- **`container_name:` collisions.** Pinning `container_name:` blocks running two stacks side by side. The substrate uses pinned names for readability; if you fork the file, drop the pin.
- **Editing the file but not recreating the service.** `docker compose restart` re-runs the container with the *existing* config. To pick up changes, use `docker compose up -d <service>` (which recreates if config has drifted) or `docker compose up -d --force-recreate <service>`.

## Key Takeaways

- One Compose file declares the entire local stack as services + networks + volumes.
- Services reach each other by **service name** on the Compose-internal network; the host reaches them via `ports:` mappings.
- `build:` builds first-party images; `image:` (alone) pulls third-party ones; together they tag a local build.
- `docker compose up -d <service>` recreates only what has drifted.

---
*Prerequisites: day-2-docker-fundamentals-images-containers-layers-registries.md, day-2-multi-stage-dockerfiles.md*
