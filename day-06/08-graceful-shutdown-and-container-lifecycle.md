# Graceful Shutdown and Container Lifecycle

> *Day 6 — Week 2, Monday*

## Overview

Spin up; serve traffic; spin down. That last word — *down* — is where most container bugs hide. A container that starts cleanly but exits messily will silently drop in-flight requests, corrupt half-written files, leave database connections in CLOSE_WAIT, and produce data inconsistencies that show up days later. With the reverse proxy in front of replicas (today's other work), graceful shutdown becomes load-bearing: scaling down a replica, restarting one for a deploy, or recovering from a crash all need the dying container to stop *politely*.

This file covers the Unix signaling that governs container lifecycle, what "graceful shutdown" actually means in a microservice context, and how to verify your services do it correctly.

## The signal flow on `docker stop`

When you run `docker stop <container>`, the daemon does the following:

1. Sends **SIGTERM** to PID 1 inside the container.
2. Waits up to `--time` seconds (default **10**) for the container to exit.
3. If still running after the timeout, sends **SIGKILL**.

`docker kill` skips step 1 and goes straight to SIGKILL. This is the "hard stop" — the process gets no chance to clean up.

`docker compose down` and `docker compose stop` use the same flow as `docker stop` per container.

The interesting verb is **SIGTERM**: "please clean up and exit." The expectation is that your application traps the signal, refuses new work, drains existing work, and then exits cleanly. If it does — `docker stop` returns quickly and cleanly. If it does not — `docker stop` hangs for 10 seconds then SIGKILLs, and any in-flight requests die mid-flight.

## What "graceful shutdown" means in a web service

For a typical HTTP service like the ones in the PEP substrate, the graceful shutdown sequence is:

1. **Trap SIGTERM** in the application process.
2. **Stop accepting new connections** — close the listening socket so the load balancer immediately stops sending work.
3. **Mark `/health` (or a separate `/ready`) as unhealthy** — so any external orchestrator immediately removes this replica from rotation.
4. **Wait for in-flight requests to finish.** Most HTTP frameworks have a "shutdown" method that does this with a timeout.
5. **Close database connections, flush logs, release file handles.**
6. **Exit cleanly (exit code 0).**

In Node.js with Express, the boilerplate looks like this:

```javascript
const server = app.listen(8000);

let shuttingDown = false;
app.get('/health', (_req, res) => {
  res.status(shuttingDown ? 503 : 200).send(shuttingDown ? 'shutting down' : 'ok');
});

function shutdown(signal) {
  console.log(`Received ${signal}, starting graceful shutdown`);
  shuttingDown = true;

  server.close((err) => {
    if (err) {
      console.error('Error during server close', err);
      process.exit(1);
    }
    // Close DB pool, etc.
    db.end().then(() => {
      console.log('Shutdown complete');
      process.exit(0);
    });
  });

  // Hard deadline — exceed it, force exit
  setTimeout(() => {
    console.error('Shutdown timed out, forcing exit');
    process.exit(1);
  }, 9000).unref();
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

In Python (FastAPI / uvicorn), the framework handles most of this — uvicorn already traps SIGTERM and drains in-flight requests. You add cleanup logic via a `lifespan` context manager or a shutdown event handler.

In Java (Spring Boot), set `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase=20s` — Spring then traps SIGTERM, refuses new requests, and waits for current ones to complete.

The defaults are usually close enough; the gotchas are PID 1 and the signal not reaching your process.

## The PID 1 problem

In a container, the process you start as `CMD` runs as PID 1. PID 1 has unusual behavior:

- **It does not get default signal handlers.** If your code does not explicitly trap SIGTERM, the signal is *ignored*. SIGTERM with no handler in a normal process kills the process; in PID 1, it is dropped.
- **It is responsible for reaping zombie children.** If your app spawns subprocesses (`exec`, `worker_threads`), and never reaps them, they accumulate as zombies.

Two consequences:

1. **Always handle SIGTERM explicitly in application code.** Do not rely on the default.
2. **If your container has a shell-based entrypoint, signals may not reach the app.** This is the most common silent bug.

The shell-based entrypoint trap:

```dockerfile
# WRONG — shell wraps the node process, eats signals
CMD npm start

# Equivalent — shell form invokes /bin/sh -c "npm start"
# /bin/sh becomes PID 1, your app is PID 2.
# SIGTERM goes to sh; sh does nothing with it.
```

```dockerfile
# RIGHT — exec form, node is PID 1 directly
CMD ["node", "server.js"]

# Or use tini as PID 1
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]
```

The **exec form** of `CMD` (JSON array) bypasses the shell and runs the binary directly. The **shell form** (string) is interpreted by `/bin/sh -c`, and the shell becomes PID 1. Always prefer exec form for application containers.

`tini` is a minimal init that runs as PID 1, forwards signals to your app, and reaps zombies. The official `node` images include it (`node:18 --init` or via the `--init` Docker flag); the `python` and `openjdk` images do not by default. Adding `tini` is cheap insurance.

## Compose's stop timeout

By default, Compose gives 10 seconds for graceful shutdown. For long-running requests (file uploads, slow database operations), that may not be enough. Configure it per service:

```yaml
services:
  user-service:
    build: ./services/user-service
    stop_signal: SIGTERM        # default; included for clarity
    stop_grace_period: 30s      # raise from 10s default
```

Some applications expect a different signal. Nginx, for example, treats SIGTERM as "fast shutdown" (kills connections) and SIGQUIT as "graceful shutdown" (drains then exits). For Nginx-based containers:

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    stop_signal: SIGQUIT
    stop_grace_period: 30s
```

Check each image's documentation for which signal triggers graceful behavior.

## Coordinating with the reverse proxy

When you scale down a replica, the proxy needs to stop sending traffic to the dying container *before* it exits. The interaction works like this:

1. You issue `docker compose up -d --scale user-service=2` (down from 3).
2. Compose sends SIGTERM to the chosen replica.
3. The replica flips its `/health` endpoint to 503.
4. The proxy's next health probe sees 503 and removes the replica from rotation.
5. The replica finishes any in-flight requests it already started.
6. The replica closes DB connections and exits.

For this dance to work, your proxy must do active health checking. Open-source Nginx does only *passive* health checking by default (a replica is removed only after it fails on a real request). For real coordination, either:

- Use Nginx Plus, or
- Use an external check that updates the upstream list, or
- Accept that the first few requests during a scale-down may hit the dying replica and be served (because it is still draining).

For this course's local dev work, passive health checking is fine. In production, this is one of the reasons people pick Envoy or Traefik over open-source Nginx.

## Verifying graceful shutdown

A simple test harness:

```bash
# Start a long-running request to one container, then stop it.
# Time the response.

# In terminal A:
time curl http://localhost/api/users/slow-operation
# /slow-operation sleeps 5 seconds then returns 200

# In terminal B, while A is running:
docker compose stop user-service

# If shutdown is graceful: terminal A returns 200 after ~5s.
# If shutdown is NOT graceful: terminal A returns 502 or connection-reset
# almost immediately.
```

Also useful — watch the container exit:

```bash
docker compose stop user-service
docker inspect <container> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
```

Exit code 0 with no SIGKILL (no OOM, no timeout) is the success condition. Exit code 137 (128 + 9 = SIGKILL) means Docker timed out and force-killed your container — your graceful shutdown is broken.

## Worked scenario — add SIGTERM handling to `user-service`

If the substrate's `user-service` does not handle SIGTERM (a not-uncommon brownfield oversight):

1. Run a long request, stop the container, observe the request dies mid-flight.
2. Add a SIGTERM handler that calls `server.close()` and exits 0.
3. Confirm exit code is 0 after `docker compose stop`.
4. Add a `stop_grace_period: 30s` to give it room.
5. Document the pattern in the service README.

That is a 30-minute change that prevents a class of production incidents.

## Common Pitfalls

- **Shell-form CMD.** `CMD npm start` makes `sh` PID 1 and your app PID 2; signals do not reach your app. Use exec form.
- **No SIGTERM handler in application code.** PID 1 ignores unhandled signals; `docker stop` then has to SIGKILL after 10s. Always handle it explicitly.
- **Closing the listening socket but not waiting for in-flight requests.** Half the work — new requests are refused, but the request that came in 50ms before SIGTERM still dies.
- **Healthcheck stays at 200 during shutdown.** The load balancer keeps sending traffic until the socket closes. Flip health to 503 as the first step.
- **Long `stop_grace_period` masking a real problem.** If you need 5 minutes to drain, your service is probably holding state it should not. Raising the timeout treats the symptom.
- **Forgetting that `docker kill` exists.** Some CI scripts use it instead of `docker stop` to "speed things up." It is a SIGKILL — no draining. Audit your scripts.

## Key Takeaways

- `docker stop` sends SIGTERM, waits up to 10s, then SIGKILL. Your application has that window to clean up.
- The graceful-shutdown sequence is: trap SIGTERM → mark unhealthy → stop accepting new connections → drain in-flight → close resources → exit 0.
- Two PID 1 traps: shell-form CMD eats signals; PID 1 has no default signal handlers. Use exec form and explicit handlers, optionally with `tini`.
- Configure `stop_signal` and `stop_grace_period` per service when defaults do not fit.
- A reverse-proxy topology amplifies the importance of graceful shutdown — without it, scaling and deploys drop traffic.

---

*Prerequisites: Day 2 (Dockerfile CMD/ENTRYPOINT), Day 5 (multi-stage images and Compose lifecycle), Day 6 — Load balancing across local replicas.*
