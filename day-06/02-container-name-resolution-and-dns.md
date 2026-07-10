# Container Name Resolution and DNS

> *Day 6 — Week 2, Monday*

## Overview

On Day 2 you used service names (`postgres`, `user-service`) as hostnames in connection strings and they "just resolved." Today's reverse-proxy work requires that resolution to be reliable across multiple networks, aliases, and replicas — so it is worth unpacking what Docker is actually doing when one container looks up another by name.

Inside every Docker user-defined network, the Docker daemon runs an embedded DNS server at `127.0.0.11`. Every container on that network gets `127.0.0.11` as its sole nameserver, and the resolver answers queries for any container name, service name, or alias attached to networks the querying container is also on.

## How the embedded resolver works

When `proxy` looks up `user-service`:

1. Resolver in the `proxy` container consults `/etc/resolv.conf` → finds `nameserver 127.0.0.11`.
2. Query goes to the Docker daemon's embedded DNS process.
3. Daemon checks its in-memory map of (network → name → container IP).
4. If `user-service` is on a network `proxy` is also attached to, it returns the IP. Otherwise it returns NXDOMAIN.
5. If the name is not a Docker name at all (e.g., `api.stripe.com`), the daemon forwards to the host's upstream DNS servers.

That last step matters: the embedded resolver is not just a Docker name lookup; it is the *only* resolver the container sees. If the daemon cannot reach the upstream DNS (e.g., on an `internal: true` network), even public lookups fail.

## What names resolve

For a Compose service `user-service` with two running replicas (`scale: 2`), the resolver knows about:

- **`user-service`** — the service name. Resolves to the IPs of *all* replicas, returned in round-robin order. This is how DNS-based load balancing works in Compose.
- **`<project>-user-service-1`, `<project>-user-service-2`** — the per-container names. Each resolves to one specific replica's IP.
- **Any aliases** declared under the service's `networks:` block.

```yaml
services:
  user-service:
    build: ./services/user-service
    networks:
      backend:
        aliases:
          - users.internal
          - user-api
```

Now `user-service`, `users.internal`, and `user-api` all resolve to the same container(s) on the `backend` network. Aliases are useful when a piece of code expects a specific hostname you cannot change — they let you make the Docker name match the expected name without renaming the service.

## What name resolution does *not* do

- **It does not cross networks the container is not on.** If `proxy` is only on `edge` but `postgres` is only on `backend`, `proxy` cannot resolve `postgres` — and a query returns NXDOMAIN, not "permission denied." This is the most common cause of "DNS is broken" reports that turn out to be "I forgot to attach the network."
- **It does not survive container restart with the same IP.** Names are stable; IPs are not. Always connect by name, never by IP. The Day 4 review etiquette around "no magic strings" applies here — a hardcoded `172.18.0.4` in a config will break the next time the stack comes up.
- **It does not work outside user-defined networks.** The legacy default bridge (the one created by `docker run` without `--network`) does not have embedded DNS. Compose always creates a user-defined network so this is mostly a footnote, but it explains why `docker run` containers cannot find each other by name without `--link` or `--network`.

## Diagnosing resolution failures

When a service cannot reach another by name, work through this checklist:

```bash
# 1. Are both containers on the same network?
docker network inspect <project>_backend | grep -A2 Containers

# 2. Can the source container query the embedded resolver at all?
docker compose exec proxy cat /etc/resolv.conf
# expect: nameserver 127.0.0.11

# 3. Does the name resolve?
docker compose exec proxy getent hosts user-service
docker compose exec proxy nslookup user-service 127.0.0.11

# 4. If resolution succeeds but connection fails, the problem is at L4/L7, not DNS:
docker compose exec proxy curl -v http://user-service:8000/health
```

If `getent hosts` returns nothing, the two services are not on a shared network. If it returns an IP but `curl` fails, the target service is not listening on the expected port — and you would investigate the container's logs and `EXPOSE` declaration, not DNS.

## Worked scenario — multi-network alias for the proxy

The reverse proxy you build today sits on both `edge` and `backend`. Suppose backend services expect to call the proxy back (for callbacks or webhooks). You want them to use `proxy.internal` rather than `proxy`:

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    networks:
      edge:
      backend:
        aliases:
          - proxy.internal
          - gateway
```

From a backend container:

```bash
docker compose exec user-service getent hosts proxy.internal
# 172.20.0.2     proxy.internal
docker compose exec user-service getent hosts gateway
# 172.20.0.2     gateway
```

Both resolve to the same proxy container, but only on the `backend` network. From the host or from a third network, neither name would resolve.

## Common Pitfalls

- **Putting full URLs in `.env` files that bake in a host.** A backend `DATABASE_URL=postgres://postgres:5432/app` works inside Compose; `postgres://localhost:5432/app` does not, because `localhost` from inside a container is the container itself.
- **Using underscores in hostnames.** Docker allows them, but some HTTP clients and TLS libraries (Java, older curl) reject hostnames with underscores. Stick to hyphens — `user-service`, not `user_service`.
- **Assuming DNS caches a failed lookup.** It does not, by default — every lookup hits the embedded resolver fresh. But application-level resolvers (Node's `dns` module, Java's `InetAddress`) *do* cache. If a service starts before its dependency exists, the failure can stick. Healthchecks and retry-on-startup logic (Day 2) fix this.
- **Forgetting that `scale: N` changes the resolver's answer.** With one replica, `user-service` returns one IP. With three, it returns three in round-robin. If a client caches the first answer, two replicas go unused — relevant when you scale services later today.

## Key Takeaways

- Embedded DNS at `127.0.0.11` is the only resolver inside Compose containers; it answers for container names, service names, and aliases on networks the querying container shares.
- Service name resolution returns *all* replicas in round-robin order — a poor-man's load balancer that the reverse proxy will replace properly.
- Resolution failures are almost always a network-attachment problem, not a DNS problem. Inspect the network before blaming the resolver.
- Aliases let you decouple "what the service is called in Compose" from "what other code expects to call it" — useful when porting brownfield code that hardcoded a hostname.

---

*Prerequisites: Day 2 (service-to-service connection strings), Day 6 — Docker networks (network topology underlies resolution).*
