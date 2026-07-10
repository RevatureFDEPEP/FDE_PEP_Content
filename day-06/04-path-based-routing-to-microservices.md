# Path-Based Routing to Microservices

> *Day 6 — Week 2, Monday*

## Overview

The reverse-proxy fundamentals file established the *why*. This file is the *how*: a complete Nginx config that fronts the four PEP substrate services (`user-service`, `question-management-service`, `test-management-service`, `api-gateway`) under one `localhost` entry point, with each service owning a distinct path prefix.

This is the core of today's deliverable. Get this working and the rest of the day (TLS, scaling, lifecycle) is incremental on top.

## The routing table

The convention this cohort uses:

| External path                | Backend                            | Container port |
|------------------------------|------------------------------------|----------------|
| `/api/users/*`               | `user-service`                     | `8000`         |
| `/api/questions/*`           | `question-management-service`      | `8000`         |
| `/api/tests/*`               | `test-management-service`          | `8000`         |
| `/api/*` (catch-all)         | `api-gateway`                      | `8000`         |
| `/*` (everything else)       | frontend (Next.js, later in week)  | `3000`         |

The order matters. Nginx's `location` matching has specific rules (covered below), and you want `/api/users/` to match before the more general `/api/` catch-all.

## A complete `nginx.conf`

```nginx
# nginx.conf — PEP substrate front door
worker_processes auto;
events {
    worker_connections 1024;
}

http {
    # Logging — useful when debugging routing
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'upstream=$upstream_addr request_time=$request_time';
    access_log /var/log/nginx/access.log main;

    # Upstreams — one per backend service. Names match Docker service names.
    upstream user_service {
        server user-service:8000;
    }
    upstream question_service {
        server question-management-service:8000;
    }
    upstream test_service {
        server test-management-service:8000;
    }
    upstream api_gateway {
        server api-gateway:8000;
    }

    # Shared proxy headers — applied via include
    # (define once at server-block level to avoid repetition)

    server {
        listen 80;
        server_name localhost;

        # ----- Backend API routes -----
        location /api/users/ {
            proxy_pass http://user_service/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Prefix /api/users;
        }

        location /api/questions/ {
            proxy_pass http://question_service/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Prefix /api/questions;
        }

        location /api/tests/ {
            proxy_pass http://test_service/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Prefix /api/tests;
        }

        # Catch-all for anything else under /api/
        location /api/ {
            proxy_pass http://api_gateway/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # ----- Healthcheck for the proxy itself -----
        location = /healthz {
            access_log off;
            return 200 "ok\n";
        }

        # ----- Default — eventually proxies to frontend -----
        location / {
            return 404;   # placeholder until Day 11 frontend integration
        }
    }
}
```

## Wiring it into Compose

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    volumes:
      - ./infra/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - edge
      - backend
    depends_on:
      - user-service
      - question-management-service
      - test-management-service
      - api-gateway
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/healthz"]
      interval: 10s
      timeout: 3s
      retries: 3

  user-service:
    build: ./services/user-service
    networks: [backend]
    # No ports: section — only reachable via proxy

  question-management-service:
    build: ./services/question-management-service
    networks: [backend]

  test-management-service:
    build: ./services/test-management-service
    networks: [backend]

  api-gateway:
    build: ./services/api-gateway
    networks: [backend]

networks:
  edge:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

## How Nginx picks a location

This is where most routing bugs originate, so internalize it:

1. **Exact match** — `location = /healthz` wins over everything else if the path matches exactly.
2. **Longest prefix with `^~`** — `location ^~ /static/` wins before regex matching, regardless of order.
3. **Regex (`~` case-sensitive, `~*` case-insensitive)** — first match in file order wins.
4. **Longest plain prefix** — the default kind shown in the config above. Longer prefix beats shorter.

For our config, all routes use plain prefixes, so the longest-match rule applies:

- `GET /api/users/123` → `/api/users/` (15 chars) beats `/api/` (5 chars). Hits user-service. Correct.
- `GET /api/auth/login` → `/api/` is the longest plain prefix that matches. Hits api-gateway. Correct.
- `GET /healthz` → exact match wins. Returns "ok". Correct.
- `GET /` → only `/` matches. Returns 404 placeholder.

You do not need to order `location` blocks for plain prefixes — Nginx sorts internally by length. Order does matter for regex blocks. Keep the config plain-prefix-only unless you have a reason otherwise.

## The trailing-slash trap

This is the single most-common Nginx routing bug. It comes down to how `proxy_pass` interacts with the `location` prefix.

| `location` | `proxy_pass` | Incoming request | Forwarded to backend |
|------------|--------------|------------------|----------------------|
| `/api/users/` | `http://user_service/` | `/api/users/123` | `/123` (prefix stripped) |
| `/api/users/` | `http://user_service` | `/api/users/123` | `/api/users/123` (preserved) |
| `/api/users` (no slash) | `http://user_service/` | `/api/users/123` | `/123` |
| `/api/users` (no slash) | `http://user_service` | `/api/users/123` | `/api/users/123` |

**Rule of thumb:** trailing slash on `proxy_pass` (with a path component, even just `/`) tells Nginx "this is a URI rewrite — strip the `location` match prefix and append the rest." No trailing slash (no path at all) tells Nginx "this is just an upstream — forward the full URI as-is."

For our microservices, **strip the prefix**. The backend `user-service` exposes routes like `/123`, `/health`, `/login` — it should not have to know that the external world calls them under `/api/users/...`. Keep the trailing slash on `proxy_pass` and the backend code stays clean.

If a backend *does* need to know its external prefix (for generating absolute URLs in responses), pass it via `X-Forwarded-Prefix` — which is why that header is in the config above. The backend reads the header when constructing URLs in its responses.

## Verifying the routing

```bash
docker compose up -d
docker compose exec proxy nginx -t          # validate config
curl -i http://localhost/healthz            # 200 ok
curl -i http://localhost/api/users/health   # whatever user-service /health returns
curl -i http://localhost/api/questions/     # whatever question-management /  returns
docker compose logs proxy | grep upstream   # see which upstream served each request
```

The `upstream=$upstream_addr` token in the log format is the diagnostic you will reach for first: it tells you *which* backend served a given request. If `/api/users/123` shows `upstream=172.x.x.x:8000` matching `api-gateway`, your routing is wrong.

## Worked scenario — adding `reporting-and-analytics`

Week 4 introduces a `reporting-and-analytics` service. When it lands, the routing update is:

1. Add an upstream block:
   ```nginx
   upstream reporting_service {
       server reporting-and-analytics:8000;
   }
   ```
2. Add a location block:
   ```nginx
   location /api/reports/ {
       proxy_pass http://reporting_service/;
       # ... same proxy_set_header lines
   }
   ```
3. Reload Nginx — no restart needed:
   ```bash
   docker compose exec proxy nginx -s reload
   ```

That is the entire change to introduce a fifth backend. The other services and the frontend are unaffected.

## Common Pitfalls

- **Trailing-slash mismatch silently working in dev, breaking in prod.** Test with curl, look at the access log, confirm the path the backend actually receives.
- **Hard-coding upstream IPs.** Use Docker service names. IPs change on restart.
- **Backend services bound to `127.0.0.1` instead of `0.0.0.0`.** A backend listening only on loopback inside the container is unreachable from the proxy container. Bind to `0.0.0.0`.
- **Forgetting to remove `ports:` from backend services.** If `user-service` still has `ports: ["8000:8000"]`, the proxy is a recommendation, not an enforcement. Bypass routes appear on `localhost:8000`.
- **CORS still configured on backends.** With one origin (the proxy), CORS becomes the proxy's job, not each service's. Strip per-service CORS middleware now to avoid double headers later.
- **No proxy healthcheck.** The proxy is now SPOF for the entire stack. A healthcheck is mandatory.

## Key Takeaways

- One `nginx.conf`, one `location` block per service, one consistent set of `proxy_set_header` directives. Boring on purpose.
- Longest-plain-prefix matching means specific routes (`/api/users/`) win over general ones (`/api/`) regardless of order in the file.
- The trailing-slash rule on `proxy_pass` decides whether the location prefix is stripped before forwarding — the most common bug to watch for.
- Adding a new service is a two-block, one-reload change. The proxy is the dial that controls topology evolution.

---

*Prerequisites: Day 6 — Reverse proxy fundamentals, Day 6 — Docker networks.*
