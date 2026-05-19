# Docker Networks — Bridge, Host, and Internal Routing

> *Day 6 — Week 2, Monday*

## Overview

Up to now, the Compose stack from Day 2 and the multi-image build pipeline from Day 5 have leaned on Docker's defaults: every service joined the same auto-created bridge network and "it worked" because services could see each other by name. Today that default becomes a design surface. Once a reverse proxy sits in front of the stack (the day's deliverable), we want backend services to be reachable from the proxy *and nothing else* — not from the host, not from a curl against `localhost:8000`. Docker networks are the primitive that enforces that.

This file covers the three network drivers you will encounter in this course (`bridge`, `host`, and the `internal: true` modifier), what they actually do at the packet level, and how to express the topology in a Compose file.

## What a Docker network actually is

A Docker network is a virtual L2/L3 segment managed by the daemon. When you attach a container to a network, Docker:

1. Creates a `veth` pair — one end inside the container's network namespace, the other on a host bridge.
2. Assigns an IP from the network's subnet.
3. Registers the container's name (and any aliases) with Docker's embedded DNS resolver scoped to that network.
4. Programs iptables rules to allow intra-network traffic and (for bridge networks) NAT outbound traffic to the host.

Two containers on the **same** user-defined network can reach each other by name on every port. Two containers on **different** networks cannot reach each other at all unless one is attached to both.

## The three drivers you will use

### `bridge` (the default)

Each Compose project gets an isolated bridge network named `<project>_default` unless you declare your own. Containers on it share a private subnet (typically `172.x.0.0/16`), get DNS-resolvable names, and can reach the outside world via NAT. Anything you publish with `ports:` is exposed on the host; anything you don't is reachable only from inside the network.

This is the workhorse driver. Use it for everything in this course unless you have a specific reason not to.

### `host`

The container shares the host's network namespace directly — no `veth`, no NAT, no port translation. A service that binds `0.0.0.0:8000` inside the container is literally on the host's `:8000`. There is no name-based discovery because there is no Docker network.

`host` networking is rare in development. It exists for high-throughput workloads or tools that need to sniff host traffic. **It does not work on Docker Desktop for Mac or Windows the way it does on Linux** — on Desktop, `host` mode still runs inside the Linux VM, so the "host" is the VM, not your laptop. Mention it for awareness; do not reach for it on this course.

### `internal: true` (a modifier on bridge networks)

Not a driver — a flag. An internal network has **no route to the outside world**. Containers on it can talk to each other, but they cannot reach the internet and they cannot be reached from the host. This is exactly what you want for databases and other backend services that should only be reachable through the proxy.

## Declaring networks in Compose

A two-tier topology — public services on a routable network, backend services hidden — looks like this:

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    networks:
      - edge
      - backend

  user-service:
    build: ./services/user-service
    networks:
      - backend
    # NOTE: no `ports:` — service is unreachable from host

  postgres:
    image: postgres:16
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: dev
    # internal-only — no port published

networks:
  edge:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

The proxy straddles both networks: it accepts traffic from the outside on `edge` and forwards to backends on `backend`. The `user-service` and `postgres` are unreachable except through the proxy. A `curl localhost:8000` from your laptop now fails — and that's the design.

## Inspecting what Docker actually built

```bash
docker network ls                          # list networks
docker network inspect <project>_backend   # show subnet, attached containers, IPs
docker compose exec proxy getent hosts user-service   # confirm DNS resolution
docker compose exec user-service curl -s https://example.com   # should fail if internal: true
```

If `getent hosts` returns nothing, the two containers are not on the same network. If `curl https://example.com` succeeds from an `internal: true` network, the network is not actually internal — check the network name in `docker network inspect`.

## Worked scenario — locking down `user-service`

Starting from the Day 5 stack where `user-service` was published on `:8000`:

1. Remove the `ports:` block from `user-service`.
2. Add an `edge` network and a `backend` network (latter `internal: true`).
3. Attach the new `proxy` service to both; attach `user-service` only to `backend`.
4. Run `docker compose up` and confirm:
   - `curl localhost:8000` from host → connection refused.
   - `curl localhost/api/users/health` through the proxy → 200 OK (assuming the proxy is configured; see the path-based routing file for Day 6).
   - `docker compose exec proxy curl user-service:8000/health` → 200 OK.

## Common Pitfalls

- **Forgetting that `internal: true` blocks outbound traffic too.** A backend service that needs to call a third-party API (Stripe, SendGrid, etc.) cannot do so on an internal network. Either move it to a non-internal backend network or add a side network with egress.
- **Assuming `ports:` is required for inter-service communication.** It is not. `ports:` publishes to the host. Service-to-service traffic uses the container port directly on the shared network. Removing `ports:` does not break Compose-internal calls.
- **Using `host` mode to "fix" connectivity problems on macOS/Windows.** It will not behave as you expect on Docker Desktop. Use named networks and DNS.
- **Multiple networks without explicit attachment.** If you define two networks but only attach a container to one, it cannot reach the other. Re-read the `networks:` block under each service.
- **Network name collisions across projects.** Compose prefixes networks with the project name. If two projects both declare a network called `backend`, they remain isolated. To share a network across projects, declare it as `external: true`.

## Key Takeaways

- Docker networks are the boundary that decides who can talk to whom; the default of "one shared bridge" is a starting point, not a target architecture.
- A two-network topology (edge + internal backend) is the foundation for the reverse-proxy pattern you will build today and rely on through Week 4.
- `internal: true` is the cheap, declarative way to make a service unreachable from the host without writing firewall rules.
- `host` networking is rarely the right tool in development on Docker Desktop; reach for it only when you have measured a need.

---

*Prerequisites: Day 2 (Compose service definitions), Day 5 (multi-service stack composition).*
