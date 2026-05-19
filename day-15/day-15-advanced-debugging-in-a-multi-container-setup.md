# Advanced Debugging in a Multi-Container Setup

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Topic 4 covers the log-reading discipline that solves 80% of slice failures. The remaining 20% — network-layer drops, hung connections, gateway routing mysteries, environment-variable mismatches — need tools that look *inside* the running containers and *between* them on the Docker network. D2 introduced `docker compose exec` and basic log tailing; today we deepen the toolkit: in-network curl probes from one service to another, attaching debuggers to running FastAPI processes, inspecting the gateway's view of upstream, and reading the Docker network itself. These are the techniques that turn "I have no idea why this is broken" into a diagnosis in 15 minutes.

## When To Reach For These Tools

Logs (Topic 4) tell you what your application *thinks* happened. These tools tell you what *actually* happened at the network and runtime layer. Use them when:

- Logs end abruptly — the request entered, no response logged, no error logged.
- Two services agree their wire is up but requests still time out.
- An env var changed and you suspect (but can't see) which value the running process has.
- A `pip install` was added and you want to confirm it landed in the image vs. in your local venv.
- The gateway returns 502 but the downstream service is healthy from outside the network.

## Tool 1: In-Network `curl` From One Service To Another

The most common slice-debugging question: "Can service A reach service B over the Docker network?" Answer it directly:

```bash
# From inside api-gateway, hit test-management-service's healthcheck
docker compose exec api-gateway curl -fsS \
  http://test-management-service:8000/healthz

# Same, but with an X-Request-Id to also test log correlation
docker compose exec api-gateway curl -fsS \
  -H "X-Request-Id: probe-$(date +%s)" \
  http://test-management-service:8000/healthz
```

Three things this tells you that external `curl localhost:8080` doesn't:

1. **DNS resolves inside the network.** If `test-management-service` doesn't resolve, the service name is wrong in `docker-compose.yml` or the service isn't on the same network.
2. **The port is correct as the *container* sees it.** External port (8080 on host) ≠ internal port (8000 in container). Confusing these is a daily mistake.
3. **The gateway's worldview is real.** If `exec api-gateway curl ...` works but the gateway proxy returns 502, the bug is in the proxy code, not the network.

### Bonus: install `curl` if the image doesn't have it

Many slim images don't include `curl`. Either install at debug time:

```bash
docker compose exec api-gateway sh -c 'apt-get update && apt-get install -y curl'
```

Or use `wget`/`python -c 'import urllib...'`/`httpie` if already present. Better: bake `curl` into your dev image (not your prod image).

## Tool 2: Inspect The Docker Network

When DNS or routing feels wrong:

```bash
# List networks
docker network ls

# Inspect the one your compose project created (usually <projectname>_default)
docker network inspect fdepep_default

# Pretty-print the containers attached
docker network inspect fdepep_default | jq '.[0].Containers | map_values({Name, IPv4Address})'
```

Output shows each container's IP on the bridge. If a service is missing from this list, it didn't join the network — usually because `docker-compose.yml` declared an explicit `networks:` block on some services but not others.

## Tool 3: Inspect The Process Inside The Container

Hung process? Wrong env var? Missing package?

```bash
# What's running
docker compose exec test-management-service ps aux

# Env vars as the running process sees them (not the shell)
docker compose exec test-management-service env | grep -E 'DATABASE|MONGO|JWT'

# Confirm a package is installed at the version you think
docker compose exec test-management-service pip show sqlalchemy

# Open a Python REPL inside the running container with the app's env
docker compose exec test-management-service python
```

The Python REPL inside the container is invaluable: you can `from app.db import engine; await engine.connect()` and see what *actually* happens with the *actual* config. No surprises about which `.env` file got loaded.

## Tool 4: Attach A Debugger To A Running FastAPI Service

Mature pattern; worth knowing. Two approaches:

### A. `debugpy` for VS Code

Bake `debugpy` into the dev image and modify the entrypoint:

```dockerfile
# Dockerfile (dev target)
RUN pip install debugpy
CMD ["python", "-Xfrozen_modules=off", "-m", "debugpy", \
     "--listen", "0.0.0.0:5678", "--wait-for-client", \
     "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Expose `5678` in `docker-compose.yml`:

```yaml
test-management-service:
  ports:
    - "8002:8000"
    - "5678:5678"   # debugpy
```

In VS Code: "Python: Remote Attach", host `localhost`, port `5678`. Set breakpoints; click in the UI; hit them.

The `--wait-for-client` flag makes the container *block at startup* until the debugger attaches. Useful for catching startup bugs; remove for normal dev.

### B. `pdb` Via `exec`

Cheaper, no IDE needed:

```python
# In the handler you're investigating
import pdb; pdb.set_trace()
```

Then run the service in *interactive* mode (this is the gotcha — `compose up -d` won't give you a TTY):

```bash
docker compose stop test-management-service
docker compose run --rm --service-ports test-management-service
# Now stdout/stdin are attached; the next request hits the breakpoint and you get a pdb prompt
```

`pdb` works; it's just less ergonomic than `debugpy`. Both have their place.

## Tool 5: Inspect The Gateway's Upstream View

When the gateway returns 502 / 504 and the downstream service looks fine, the gateway's *configuration* is the suspect.

For our FastAPI-based `api-gateway`, the upstream URLs are env vars:

```bash
docker compose exec api-gateway env | grep _SERVICE_URL
# USER_SERVICE_URL=http://user-service:8000
# TEST_MANAGEMENT_SERVICE_URL=http://test-management-service:8000
# QUESTION_MANAGEMENT_SERVICE_URL=http://question-management-service:8000
```

If any URL is wrong (typo, wrong port, missing `http://`), the gateway can't reach the downstream regardless of what the downstream is doing. Fix the env var; restart the gateway.

For an nginx-style reverse proxy (D6), the equivalent is `docker compose exec api-gateway cat /etc/nginx/conf.d/default.conf` and reading the `proxy_pass` lines.

## Tool 6: Database Health From Inside The Network

Same logic as the curl probe, but for data stores:

```bash
# Postgres reachability and a trivial query
docker compose exec test-management-service python -c \
  "import asyncio, asyncpg; \
   async def main(): \
     c = await asyncpg.connect('postgresql://app:app@postgres:5432/test_mgmt'); \
     print(await c.fetchval('SELECT 1')); \
   asyncio.run(main())"

# Mongo reachability
docker compose exec test-management-service python -c \
  "from pymongo import MongoClient; \
   print(MongoClient('mongodb://mongo:27017', serverSelectionTimeoutMS=2000).admin.command('ping'))"

# Postgres lock contention (the D12 row-lock scenario)
docker compose exec postgres psql -U app test_mgmt -c \
  "SELECT pid, state, query_start, wait_event, query \
   FROM pg_stat_activity WHERE wait_event_type='Lock';"
```

The lock-contention query is the killer move for diagnosing "submit hangs forever" bugs — you literally see the rows that are stuck.

## Tool 7: `docker stats` And Resource Pressure

Sometimes the issue isn't application logic; it's resources:

```bash
docker stats --no-stream
```

If Mongo is sitting at 100% CPU because someone forgot an index from D12, that's a different problem than "Mongo is unreachable." `docker stats` separates them in two seconds.

## A Worked Example: "Submit Hangs Forever"

End-to-end debug using the toolkit, on the failure mode from D12:

```bash
# 1. Reproduce — submit twice quickly from two terminals
# Symptom: second submit hangs

# 2. Logs (Topic 4 first)
docker compose logs --tail 50 test-management-service | tail
# Last line: "acquired pg lock on session row"  → no rollback, no commit

# 3. Confirm: lock contention?
docker compose exec postgres psql -U app test_mgmt -c \
  "SELECT pid, wait_event, query FROM pg_stat_activity WHERE wait_event_type='Lock';"
# Two rows: one holding, one waiting on session row sess_42

# 4. Why is the holder not releasing?
docker compose exec test-management-service ps aux
# The Python process is in the middle of a long-running Mongo query
# (scoring needs question correctness data; Mongo is slow)

# 5. Probe Mongo
docker compose exec test-management-service python -c \
  "from pymongo import MongoClient; \
   db = MongoClient('mongodb://mongo:27017').questions; \
   import time; t=time.time(); list(db.questions.find({'_id':{'$in':['q1','q2','q3']}})); print(time.time()-t)"
# Prints: 8.4
# That's not transient; that's a missing index

# 6. Diagnosis: scoring's Mongo query has no index; the lock is held for 8s
#    while scoring runs; second submit waits the full lock_timeout
```

The toolkit walked us from "hangs forever" to "missing Mongo index on `_id` lookup" in five minutes. Without these tools the same diagnosis takes an hour.

## Anti-Patterns

- **Editing source files inside a running container.** Changes vanish on rebuild; encourages drift between dev and image. Always edit on the host (with a bind mount if needed).
- **Adding `print()` and `docker compose up --build` to redeploy.** Slow loop. Use `pdb` / `debugpy` / `python -c` inside the running container instead.
- **`docker compose down` to "reset" mid-debug.** You lose the state that caused the bug. Inspect first; reset only after you understand.
- **Trusting `localhost` and external ports for in-network probes.** `localhost:8002` from the host hits the published port; `localhost:8000` from inside the container hits the *container itself*. Different things. Use service-name DNS over the network: `http://test-management-service:8000`.
- **Installing tools in production images.** `curl`, `vim`, `pdb` belong in the dev image only. Production stays minimal.
- **`docker compose restart` instead of fixing the underlying issue.** A restart that "fixes" the problem hides whatever leaked. Diagnose first.

## Key Takeaways
- In-network `curl` via `docker compose exec` is the single most useful debugging tool — confirms DNS, port, and reachability in one command.
- The Python REPL inside the running container shows you the *actual* env, not your shell's env.
- `debugpy` for VS Code or `pdb` via interactive `compose run` both attach debuggers to running FastAPI handlers.
- For 502s, suspect gateway env-var misconfiguration first (`exec api-gateway env | grep URL`).
- `pg_stat_activity` exposes lock contention live — the killer query for "submit hangs forever."
- These tools come *after* Topic 4's log analysis, not before; logs are the cheaper layer.

---
*Prerequisites: day-2-container-debugging-logs-exec-troubleshooting, day-6-reverse-proxy-fundamentals, day-12-database-transactions-and-pessimistic-locking, day-15-distributed-log-analysis-across-services.*
