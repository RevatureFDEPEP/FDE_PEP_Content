# Docker Fundamentals — Images, Containers, Layers, Registries

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Every service in the `rev-eval-ai` PEP substrate is shipped as a container, and today's deliverable — the full stack via `docker-compose up` — only succeeds when you have a mental model of what Docker is actually doing. This topic establishes that mental model: the difference between an image and a container, why layer caching dominates build performance, and how registries fit between your laptop, CI, and (eventually) ECR. You will reuse this model on Day 3 when CI builds the same images, on Day 5 when you tune multi-stage builds, and every time a build feels mysteriously slow.

## The Four Concepts

### Image
An **image** is an immutable, content-addressed filesystem snapshot plus metadata (entrypoint, env, exposed ports). It is a *template*, not a running thing. Built once from a `Dockerfile`, identified by a SHA digest, and tagged with human-readable names like `user-service:latest` or `postgres:16-alpine`.

```bash
docker image ls                 # list local images
docker image inspect user-service:latest
docker image rm user-service:latest
```

### Container
A **container** is a running (or stopped) instance of an image — the image's filesystem plus a thin writable layer on top, an isolated process tree, network namespace, and cgroups. You can run many containers from one image. Containers are cheap; treat them as disposable.

```bash
docker ps                       # running containers
docker ps -a                    # include stopped
docker run --rm -it user-service:latest bash
docker rm <container-id>
```

If you `docker exec` into a running container and create a file, it lives in that container's writable layer and dies with the container. This is why **persistent data lives in named volumes**, not in containers (Postgres, Mongo, and MinIO topics expand on this).

### Layers
A Dockerfile produces an image as a stack of **layers** — one per significant instruction (`FROM`, `RUN`, `COPY`, `ADD`). Each layer is content-addressed; Docker caches them aggressively. When you rebuild, Docker walks the Dockerfile from the top and reuses cached layers until something changes, then rebuilds from that point down.

Two practical consequences you will feel today:

1. **Order matters for cache hits.** Copy your manifest (`pyproject.toml`, `package.json`) and install dependencies *before* copying source code. Otherwise every code edit invalidates the dependency-install layer and reinstalls everything.
2. **Layers are additive only.** `RUN rm -rf /tmp/junk` after `RUN apt-get install ...` does not shrink the image — the junk lives in the earlier layer forever. To remove, do it in the same `RUN`. Multi-stage builds (next topic) sidestep this entirely.

### Registry
A **registry** is the remote store for images. Docker Hub is the default; the cohort uses pre-provisioned AWS ECR for the deployed pipeline (Day 7+). The flow is always `build -> tag -> push -> pull`.

```bash
docker tag user-service:latest 123.dkr.ecr.us-east-1.amazonaws.com/user-service:abc123
docker push 123.dkr.ecr.us-east-1.amazonaws.com/user-service:abc123
docker pull postgres:16-alpine   # also a registry pull, just from Docker Hub
```

Locally for Day 2 you do not push anywhere — Compose builds images and runs them directly. Registries become relevant on Day 3.

## Example / Worked Scenario

Walk through what happens the first time you build `user-service` for the local stack:

```bash
cd services/user-service
docker build -t user-service:dev .
```

Docker reads the Dockerfile top-down:

1. `FROM python:3.11-slim` — pulls the base image from Docker Hub (one network round-trip, cached forever after).
2. `WORKDIR /app` — cheap layer, sets working directory.
3. `COPY pyproject.toml uv.lock ./` — copies just the manifest; this layer is cached unless the manifest changes.
4. `RUN uv pip install --system .` — installs dependencies; the **expensive** layer, cached as long as step 3 is cached.
5. `COPY . .` — copies source; invalidated by every code edit (but cheap).
6. `CMD ["uvicorn", "app.main:app", ...]` — metadata, no filesystem change.

Now edit a single line in `app/routes.py` and rebuild. Steps 1–4 hit cache; only steps 5–6 re-execute. Build time drops from ~90s to ~3s. This is the layer cache earning its keep.

Inspect the result:

```bash
docker image ls user-service
docker image history user-service:dev   # shows each layer + its size
docker image inspect user-service:dev | jq '.[0].Config.Cmd'
```

## Common Pitfalls

- **Copying everything before installing deps.** `COPY . .` followed by `RUN pip install` invalidates the install layer on every source edit. Always copy the manifest first.
- **Confusing image tags with image identity.** `latest` is just a mutable label — two different builds can both be tagged `latest`. The SHA digest is the truth. When debugging "why is CI running old code," check digests, not tags.
- **Leaving dangling images.** Repeated rebuilds leave untagged image layers behind that quietly eat disk. `docker image prune` reclaims them; `docker system prune -a` is the nuclear option (do not run mid-cohort without warning).
- **Treating containers as persistent.** Anything written inside a container's filesystem disappears when the container is removed. State belongs in a volume.

## Key Takeaways

- Images are immutable templates; containers are running instances with a thin writable layer on top.
- Layers are cached top-down by Dockerfile instruction — order manifest copies before source copies to keep dependency installs cached.
- Registries store images for sharing between machines; for Day 2 you build and run locally only.
- Persistent data never lives in a container — it lives in a named volume.

---
*Prerequisites: [08-polyglot-development-environment-git-pnpm-python-docker.md](../day-01/08-polyglot-development-environment-git-pnpm-python-docker.md), [05-codebase-navigation-conventions.md](../day-01/05-codebase-navigation-conventions.md)*
