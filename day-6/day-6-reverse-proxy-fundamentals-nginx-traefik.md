# Reverse Proxy Fundamentals (Nginx / Traefik)

> *Day 6 — Week 2, Monday*

## Overview

The Week 1 stack worked, but it had no front door. Four services, each on its own published port (`:8000`, `:8001`, `:8002`, `:8003`), each addressed differently from the frontend, each independently responsible for CORS, TLS, and rate limiting. That sprawl is fine for one developer poking at one service, but it does not scale to a product. Today's deliverable replaces it with one consistent entry point: a reverse proxy that accepts every request and forwards it to the right backend service.

This file establishes the concept — what a reverse proxy *is*, why microservice topologies put one in front, and the trade-off between Nginx (static config) and Traefik (dynamic discovery). The next file (path-based routing) builds the concrete config.

## Forward vs reverse proxy

The terminology trips people up, so settle it now.

- A **forward proxy** sits between clients and the wider internet. The client is configured to send requests through it; the proxy fetches resources on the client's behalf. Corporate web filters, Squid, and the proxy your work laptop uses for outbound HTTP are forward proxies.
- A **reverse proxy** sits between the wider internet and your servers. The *client* sees a single hostname; the proxy decides which backend handles each request. The client has no idea (and should have no idea) how many services are behind it.

Same software (Nginx, HAProxy, Envoy, Traefik) can do either job; the difference is where it sits and whose interest it serves.

## What a reverse proxy gives you

In the architecture you build today, the proxy is doing several jobs at once:

1. **Single entry point.** Clients send everything to `http://localhost/` — one hostname, one port. The frontend no longer has to know that users live at one URL and questions at another.
2. **Routing.** Based on path (`/api/users/*` → user-service) or host (`api.example.com` → backend, `admin.example.com` → admin-ui), the proxy picks the upstream.
3. **Cross-cutting concerns done once.** TLS termination, gzip compression, request logging, rate limiting, CORS headers, request ID injection — all configured in one place instead of replicated across four services.
4. **Backend abstraction.** You can split a service in two, swap an implementation, or scale to multiple replicas without changing what the client calls. The proxy hides the topology.
5. **Health-aware load balancing.** When you scale `user-service` to three replicas (later in the day), the proxy distributes traffic and skips replicas that fail their health checks.
6. **Security boundary.** Combined with the `internal: true` network from earlier today, the proxy is the *only* thing on the host's network. Backends are unreachable except through it.

## Nginx vs Traefik — the choice for this course

You will use **Nginx** as the default for this cohort because the substrate config is small, well-understood, and identical to what most production teams ship. Traefik is worth knowing about because it represents a different philosophy.

### Nginx — static config, explicit, battle-tested

```nginx
# nginx.conf — minimal reverse proxy
events {}

http {
    upstream user_service {
        server user-service:8000;
    }

    server {
        listen 80;

        location /api/users/ {
            proxy_pass http://user_service/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

Mount this file into the official `nginx:1.27-alpine` image and you have a working reverse proxy. Add a `location` block per service. Reload (`nginx -s reload`) or restart the container to pick up changes.

**Strengths:** explicit, debuggable (one file), excellent performance, vast operational knowledge in the wider world. The `proxy_pass`, `upstream`, and `location` primitives have not changed in a decade.

**Weaknesses:** every new service needs a config edit and a reload. No native service discovery. Per-service config can become repetitive.

### Traefik — dynamic, label-driven

```yaml
services:
  traefik:
    image: traefik:v3.1
    command:
      - --providers.docker=true
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  user-service:
    build: ./services/user-service
    labels:
      - traefik.enable=true
      - traefik.http.routers.users.rule=PathPrefix(`/api/users`)
      - traefik.http.services.users.loadbalancer.server.port=8000
```

No central config file. Traefik watches the Docker socket and configures routes from labels on each service. Add a service with the right labels and it appears in the router automatically.

**Strengths:** zero touch to add a service, sensible defaults, built-in Let's Encrypt, slick dashboard.

**Weaknesses:** more magical (routing decisions live in labels scattered across the Compose file), Docker socket access is a security trade-off, and operational knowledge in the wider world is thinner than Nginx.

For this course, **Nginx is the default**. The path-based routing file walks through a complete config; the Traefik snippet above is here so you recognize the alternative when you see it in the wild.

## The request lifecycle through Nginx

Trace a single request to make the picture concrete:

1. Browser sends `GET http://localhost/api/users/123` to the host's `:80`.
2. Docker forwards `:80` on the host to the proxy container's `:80` (via the published port).
3. Nginx in the proxy container matches `location /api/users/`.
4. Nginx rewrites the path (strips the prefix), looks up `user-service` via Docker DNS, picks an IP (if multiple replicas, round-robin), and opens a TCP connection to `user-service:8000`.
5. `user-service` responds; Nginx forwards the response back to the client.

The client only ever saw `localhost:80`. Steps 3-5 are invisible.

## Common Pitfalls

- **Forgetting the trailing slash on `proxy_pass`.** `proxy_pass http://user_service;` (no slash) preserves the full incoming path. `proxy_pass http://user_service/;` (with slash) strips the `location` prefix. The next file covers this in detail; for now, the rule is "trailing slash means rewrite."
- **Not setting `X-Forwarded-*` headers.** Backends behind a proxy see the *proxy's* IP as the client. If the backend logs request origin or does rate limiting, you need `X-Real-IP`, `X-Forwarded-For`, and `X-Forwarded-Proto` set explicitly. The snippet above does this.
- **Using `localhost` inside the Nginx config.** From inside the proxy container, `localhost` is the proxy itself. Always use the Docker service name (`user-service`).
- **Treating the proxy as stateless and free.** It is the choke point for every request. A misconfiguration here breaks every backend at once. Treat changes to proxy config with the same care as DB migrations.
- **Skipping the proxy on the frontend.** If the Next.js frontend hardcodes `http://user-service:8000` in its API client, the moment the proxy is in place those calls bypass it. The frontend should call `/api/users/...` relative to the proxy.

## Key Takeaways

- A reverse proxy is the single front door to a microservice topology — clients see one hostname; the proxy decides which backend serves each request.
- Beyond routing, the proxy is where you centralize TLS, logging, rate limiting, and CORS — concerns that would otherwise be replicated in every service.
- Nginx (static config, explicit) and Traefik (dynamic, label-driven) are the two mainstream options. This course uses Nginx by default; recognize Traefik when you see it.
- The proxy combined with `internal: true` backend networks gives you a real security boundary, not just a convenience.

---

*Prerequisites: Day 6 — Docker networks, Day 6 — Container name resolution and DNS.*
