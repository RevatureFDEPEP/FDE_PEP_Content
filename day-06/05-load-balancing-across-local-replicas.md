# Load Balancing Across Local Replicas

> *Day 6 — Week 2, Monday*

## Overview

The reverse proxy from earlier today has one job per backend service: forward requests to a single container. That is fine for development with one developer hitting one stack — but it is also a useful sandbox for the concept of horizontal scaling. Compose can run multiple replicas of a service; Nginx can distribute traffic across them; and the failure modes that show up at small scale (sticky sessions, in-memory state, uneven distribution) are exactly the ones you will see in production.

This file walks through scaling `user-service` to multiple replicas locally, what the proxy does with them, and where the pattern breaks down.

## Scaling a service in Compose

Compose has two ways to run multiple containers of one service:

### CLI flag

```bash
docker compose up -d --scale user-service=3
```

Three containers come up: `<project>-user-service-1`, `-2`, `-3`. All on the same network, all reachable by the service name `user-service`. Docker's embedded DNS (covered in the name-resolution file earlier today) now returns three IPs in round-robin order for the name `user-service`.

### `deploy.replicas` in `compose.yaml`

```yaml
services:
  user-service:
    build: ./services/user-service
    networks: [backend]
    deploy:
      replicas: 3
```

Equivalent declaratively. `deploy:` keys were originally a Swarm thing; modern Compose honors `replicas` even outside Swarm.

**Important:** you cannot scale a service that has a published `ports:` entry. Two containers cannot bind the same host port. This is why the reverse-proxy pattern is a prerequisite for local scaling — the backend services no longer publish ports, the proxy does, and the proxy distributes traffic.

## How Nginx load-balances

Nginx's `upstream` block can list multiple servers, and by default it uses round-robin:

```nginx
upstream user_service {
    server user-service:8000;
    # No other servers listed — but Docker DNS returns multiple IPs
    # for the name 'user-service' when scaled.
}
```

Nginx resolves `user-service` *once at startup* (or on reload) by default. If you scale up after Nginx starts, the new replicas are invisible. There are two ways around this:

### Option A — restart/reload Nginx after scaling

```bash
docker compose up -d --scale user-service=3
docker compose exec proxy nginx -s reload
```

Crude but works. Tolerable for development.

### Option B — runtime DNS resolution

Tell Nginx to resolve the upstream name at runtime, on each request (or on a TTL):

```nginx
http {
    # Use Docker's embedded DNS resolver. 5s TTL.
    resolver 127.0.0.11 valid=5s ipv6=off;

    server {
        listen 80;

        location /api/users/ {
            # Variable in proxy_pass forces runtime resolution.
            set $upstream_users http://user-service:8000;
            proxy_pass $upstream_users$request_uri;
            proxy_set_header Host $host;
        }
    }
}
```

Two things make this work:

1. `resolver 127.0.0.11` — points Nginx at Docker's embedded DNS.
2. Using a variable (`$upstream_users`) in `proxy_pass` — this disables Nginx's startup-time resolution and forces it to resolve on each request, respecting the resolver's TTL.

The trade-off: you lose the `upstream` block's other features (load-balancing methods, weights, health checks) when using a variable. For development, that is usually fine; for production with multiple replicas, the open-source Nginx limitation is real and is one reason Traefik (which discovers via the Docker socket, not DNS) is appealing.

### Option C — list replicas explicitly

If you know the replica count is fixed, list them:

```nginx
upstream user_service {
    server <project>-user-service-1:8000;
    server <project>-user-service-2:8000;
    server <project>-user-service-3:8000;
}
```

Brittle (you have to know the project name and replica count), but gives you the full upstream feature set: weighted distribution, slow-start, passive health checks.

## Load-balancing methods

Inside an `upstream` block:

```nginx
upstream user_service {
    # least_conn;   # send to the replica with fewest active connections
    # ip_hash;      # always send the same client IP to the same replica (sticky)
    # random;       # random pick

    server <project>-user-service-1:8000;
    server <project>-user-service-2:8000 weight=2;   # 2x the traffic
    server <project>-user-service-3:8000 backup;     # only used if others fail
}
```

Default (round-robin) is the right starting point. `least_conn` is a safer default if your services have variable request times. `ip_hash` is a poor man's session affinity — useful but a code smell (see "stateful replicas" below).

## Verifying load distribution

```bash
# Scale up
docker compose up -d --scale user-service=3
docker compose exec proxy nginx -s reload

# Hit the endpoint many times and see which container served each
for i in {1..30}; do
  curl -s http://localhost/api/users/whoami
done | sort | uniq -c
# Expect roughly even distribution if /whoami includes a hostname or pod ID.
```

A backend that returns its `$HOSTNAME` (or container ID) from a debug endpoint is the cheapest way to see distribution. Add one if your services do not already have it.

## The stateful-replicas problem

Round-robin works *only if every replica is interchangeable*. The moment a replica holds state that another replica does not — an in-memory session, a local cache, an uploaded file on its filesystem — load balancing breaks.

Concrete failure modes you will see if you scale a stateful service:

- **In-memory sessions.** User logs in, request goes to replica 1, session stored in replica 1's memory. Next request goes to replica 2 — user is "logged out." Fix: move sessions to Redis or signed cookies.
- **Local uploaded files.** User uploads a profile picture, served from replica 2's disk. Next request hits replica 3 — 404. Fix: object storage (S3, MinIO).
- **Local rate-limit counters.** Each replica counts independently. With 3 replicas, the effective rate limit is 3x what you configured. Fix: shared counter in Redis.
- **In-process WebSocket connections.** Client connects to replica 1, then sends a message; the message goes to replica 2 which has no idea the connection exists. Fix: pub/sub broker between replicas, or sticky sessions (`ip_hash`).

The PEP substrate's services are mostly stateless (state lives in Postgres and Mongo, both single instances). They scale fine. When the curriculum reaches the question-authoring slice (Wed-Thu) and you have a service that holds in-flight draft state, be deliberate about where that state lives.

## Worked scenario — scaling `user-service` to 3

Starting from the proxy + single-replica stack:

1. Remove any `ports:` on `user-service`.
2. Add `deploy: { replicas: 3 }`.
3. Update Nginx to use runtime resolution (Option B above) so new replicas appear without a reload.
4. `docker compose up -d`.
5. `docker compose ps` — should show three `user-service-N` containers.
6. Curl the proxy 30 times; confirm hits are distributed across replicas (via a `/whoami` endpoint or via `docker compose logs user-service | grep <request-id>`).
7. Kill replica 2 (`docker compose kill <name>`). Subsequent requests should keep working — Nginx will retry the other replicas (the `proxy_next_upstream` directive controls this; default is "retry on connection error").

Then turn the dial up to 5. Then down to 1. Observe: the proxy keeps serving the whole time, because the abstraction holds.

## Common Pitfalls

- **Scaling a service with `ports:`.** Compose errors out. Remove the published port and put the proxy in front.
- **Forgetting to reload Nginx after scaling.** Without runtime resolution, new replicas are invisible. Either reload or use the variable-in-`proxy_pass` trick.
- **Treating local-disk state as if it were ephemeral.** It is, per-container, but other replicas cannot see it. Files uploaded to one replica are gone from the next request.
- **Using `ip_hash` as a long-term answer.** Sticky sessions paper over a real architecture problem. Move state out, then scale.
- **Assuming "scaled" means "highly available."** Three replicas of a service all on the same Docker host go down together when the host goes down. Real HA is a different (cloud-level) topic; we are practicing the *pattern* locally.

## Key Takeaways

- Compose `--scale` (or `deploy.replicas`) plus a reverse proxy gives you a working horizontal-scale sandbox.
- Default Nginx resolves upstream names once at startup; use runtime DNS resolution (variable in `proxy_pass` with `resolver 127.0.0.11`) for replicas-changing-at-runtime workflows.
- Round-robin assumes interchangeable replicas. State that lives in-process or on local disk breaks that assumption — move it to a shared store before scaling.
- A service that scales cleanly to 3 replicas in development is more likely to scale cleanly in production. Scale early to surface state-bleed bugs.

---

*Prerequisites: Day 6 — Path-based routing, Day 6 — Container name resolution and DNS.*
