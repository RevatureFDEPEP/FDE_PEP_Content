# Container Debugging — Logs, Exec, Troubleshooting

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Most of the time the stack will come up cleanly. The other times, you need to be quick with `docker compose logs`, `docker compose exec`, and `docker inspect` — these are the tools you fall back on every day for the rest of the cohort. This topic catalogues the common operational failures from a real cohort stack-up and the diagnostic recipe for each.

## The Three Tools

### `docker compose logs` — what is the container saying

```bash
docker compose logs                  # all services, combined
docker compose logs user-service     # one service
docker compose logs -f user-service  # follow (tail)
docker compose logs --tail 50 user-service
docker compose logs --since 10m
```

This is your first move on any failure. The logs are stdout/stderr from the container's main process — if the service crashes on startup, the stack trace is here.

Combined logs are colour-coded per service and useful for cross-service issues ("did the API gateway 500 because user-service died?"). Single-service logs are quieter when you know where to look.

### `docker compose exec` — get a shell inside

```bash
docker compose exec user-service bash
docker compose exec user-service sh          # bash often missing in slim images
docker compose exec postgres psql -U app -d users
docker compose exec mongo mongosh -u app -p app --authenticationDatabase admin
```

`exec` requires the container to be **running**. If it has crashed, `exec` returns "container not running." For a crashed container, either fix the crash or `run` a one-off container from the same image:

```bash
docker compose run --rm --entrypoint sh user-service
```

This starts a new container with the service's image but bypasses the original entrypoint — useful when the entrypoint itself is broken.

### `docker inspect` — what is the container's config, actually

```bash
docker inspect user-service
docker inspect --format '{{.State.Status}}' user-service
docker inspect --format '{{json .State.Health}}' postgres | jq
docker inspect --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}' postgres
docker network inspect <project>_default
```

This is the "what does Docker think the configuration actually is" tool — distinct from "what does the Compose file say." Useful when an env var seems wrong, a volume seems missing, or DNS is misbehaving.

## A Triage Checklist

When the stack misbehaves, walk this list:

1. **`docker compose ps`** — what is the status of each service? Anything `Exited` or `unhealthy`?
2. **`docker compose logs <service>`** — read the logs of the unhappy one. Most problems explain themselves here.
3. **Healthcheck failing?** `docker inspect --format '{{json .State.Health}}' <service> | jq` shows the last few check outputs verbatim.
4. **Container running but request failing?** `docker compose exec` in, hit the endpoint with `curl` or the language's equivalent. Distinguishes "service is broken" from "networking is broken."
5. **Networking issue?** From inside one container, `getent hosts <other-service>` should resolve. If it doesn't, the services are not on the same Compose network.
6. **Volume issue?** `docker volume inspect <project>_<volume>` shows mount point and labels; `docker compose exec <svc> ls -la <mount-target>` shows what is actually there.

## Common Failure Modes — and Their Tells

### "Connection refused" between services
Symptom: `user-service` logs show `connection refused` against `postgres`. Diagnosis: either Postgres is not actually healthy yet (healthcheck/dependency wiring problem — see previous topic), or you wrote `localhost` instead of `postgres` in the connection string.

```bash
docker compose exec user-service getent hosts postgres
# should resolve to a container IP
```

### Container exits immediately
Symptom: `docker compose ps` shows the service as `Exited (1) 2 seconds ago`. Diagnosis: process crashed on startup. The error is in the logs.

```bash
docker compose logs --tail 100 <service>
```

Common causes: missing env var (`KeyError: 'DATABASE_URL'`), bad import (`ModuleNotFoundError`), broken healthcheck command running on PID 1 if you misused entrypoints.

### Build failure mid-Dockerfile
Symptom: `docker compose up --build` fails during a `RUN` step. Diagnosis: rebuild with `--no-cache` to rule out stale layers, then target the failing stage and exec in.

```bash
docker compose build --no-cache <service>
docker build --target builder -t debug <service-path>
docker run --rm -it debug sh
```

### Volume permissions
Symptom: service logs "permission denied" writing to its mounted volume. Diagnosis: usually a UID mismatch between the in-container user and the volume owner. For named volumes Docker handles this; for bind mounts you may need `chown` in the Dockerfile or to run as a specific UID.

### Image is old, despite editing the Dockerfile
Symptom: code changes "do nothing." Diagnosis: Compose only rebuilds when you ask. Use `docker compose up -d --build <service>` or `docker compose build <service>` first.

### Port already in use on host
Symptom: `bind: address already in use` on `compose up`. Diagnosis: another process (a previous Compose run, a local Postgres install, another dev server) owns the host port. Either stop it or change the host side of the `ports:` mapping.

## Example / Worked Scenario

A learner reports "`user-service` keeps restarting." Walk the triage:

```bash
docker compose ps user-service
# STATUS: Restarting (1) 5 seconds ago

docker compose logs --tail 50 user-service
# ...
# psycopg.OperationalError: connection to server at "postgres" (172.18.0.2),
# port 5432 failed: FATAL: password authentication failed for user "app"
```

Cause located: password mismatch between `POSTGRES_PASSWORD` on the postgres service and the password in `DATABASE_URL` on user-service. Fix the env var, then:

```bash
docker compose up -d --force-recreate user-service
docker compose logs -f user-service
```

Watch the new logs come clean. Total time from "it's broken" to "it's fixed" with this pattern is rarely more than a minute or two — *if* you start at the logs.

## Common Pitfalls

- **Reading the wrong service's logs.** Combined `docker compose logs` is great, but easy to misread when many services are noisy. When in doubt, isolate to one service.
- **`exec`-ing into the wrong container instance.** After a `--force-recreate`, the container ID changes. Stale terminal sessions point at a container that no longer exists.
- **Trusting `restart: always` to fix things.** Auto-restart hides the actual error in a tight loop. Drop it temporarily when diagnosing, or use `restart: on-failure:3` so it gives up.
- **Forgetting that `docker compose down -v` exists.** When state looks corrupt, fully tearing down (including volumes) and starting fresh is sometimes the fastest path — at the cost of wiping local data.

## Key Takeaways

- `logs`, `exec`, `inspect` — in that order — solves the majority of container problems.
- `docker compose ps` is the dashboard; always glance at it first.
- For crashed containers, use `docker compose run --rm --entrypoint sh <service>` to poke at the image without its entrypoint.
- Most cross-service failures are either DNS (wrong hostname) or ordering (dependency not yet healthy).

---
*Prerequisites: [03-local-orchestration-with-docker-compose.md](03-local-orchestration-with-docker-compose.md), [08-healthchecks-and-service-dependency-conditions.md](08-healthchecks-and-service-dependency-conditions.md)*
