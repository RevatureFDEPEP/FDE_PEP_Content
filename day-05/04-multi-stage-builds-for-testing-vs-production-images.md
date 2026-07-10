# Multi-Stage Builds for Testing vs Production Images

> *Day 5: Advanced Container Builds & Multi-Service Integration — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Day 2 introduced multi-stage Dockerfiles as a way to produce a slim production image — the `builder` stage compiles or installs, the `runner` stage copies only the runtime artifacts and discards the toolchain. That pattern solves *one* problem: image size.

Today we extend it to solve a second problem: **producing distinct test and production images from the same Dockerfile**. The test image carries dev dependencies, test runners, and (often) coverage tooling. The production image carries none of that. Both come out of one `docker build` invocation by selecting a different target stage. Day 7's CI will use this directly: the `test` job pulls the test image to run pytest/jest; the `package` job emits the production image for the registry.

If you've ever shipped a production container that bundled pytest, `node_modules/.bin/eslint`, and a 200MB toolchain because "we needed it for the tests in CI" — that's the anti-pattern this topic eliminates.

## The Shape of a Test-vs-Prod Multi-Stage Dockerfile
The Dockerfile gains a third (sometimes fourth) stage. For `user-service` (Node/TypeScript):

```dockerfile
# syntax=docker/dockerfile:1.7

# --- Stage 1: deps (cacheable, shared) ---
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# --- Stage 2: build (compile TS, produce dist/) ---
FROM deps AS build
COPY . .
RUN pnpm run build

# --- Stage 3: test (full toolchain + test deps + source) ---
FROM build AS test
ENV NODE_ENV=test
# Keep dev deps; they're already in node_modules from `deps`
CMD ["pnpm", "run", "test"]

# --- Stage 4: prod (slim runtime, dist + prod deps only) ---
FROM node:20-alpine AS prod
WORKDIR /app
ENV NODE_ENV=production
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile --prod
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

Two key moves:

1. **`FROM build AS test`** — the test stage *inherits* from `build`, which means it already has source, compiled output, and (critically) dev dependencies. No re-install.
2. **`FROM node:20-alpine AS prod`** — the prod stage starts from a fresh slim base and copies *only* `dist/` plus a clean `--prod` install. The dev deps from earlier stages do not survive into this image.

## Building a Specific Stage
Docker builds the final stage by default. To target an earlier one:

```bash
# Test image (default tag, won't be pushed to a prod registry)
docker build --target test -t user-service:test .

# Production image (the one CI will push)
docker build --target prod -t localhost:5000/user-service:dev .
```

Layers from `deps` and `build` are cached and shared between the two builds. The first time you build both targets you pay the install cost once; subsequent builds of either reuse the cached layers as long as `package.json` and `pnpm-lock.yaml` haven't changed.

## Why This Matters Beyond Image Size
- **Attack surface.** A production image with `pytest` installed has more package surface for a CVE scan to flag. Day 7's CI introduces image scanning; minimizing surface there pays off immediately.
- **Determinism in tests.** The test image runs the *same* compiled artifact as production, plus the test harness. Tests don't accidentally run against differently-compiled code.
- **Cache locality in CI.** One `docker build --target test` and one `--target prod` share the heavy `deps` layer. Two separate Dockerfiles would not.
- **One source of truth.** The prod-vs-test difference is *expressed* in the Dockerfile, not hidden in a CI script that copies files around.

## The Python Variant
`question-management-service` is Python. Same pattern, different verbs:

```dockerfile
# --- Stage 1: deps ---
FROM python:3.12-slim AS deps
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install --no-cache-dir poetry==1.8.* \
 && poetry config virtualenvs.create false \
 && poetry install --no-root --with dev

# --- Stage 2: test ---
FROM deps AS test
COPY . .
ENV PYTHONDONTWRITEBYTECODE=1
CMD ["pytest", "-x", "--cov=app"]

# --- Stage 3: prod ---
FROM python:3.12-slim AS prod
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY pyproject.toml poetry.lock ./
RUN pip install --no-cache-dir poetry==1.8.* \
 && poetry config virtualenvs.create false \
 && poetry install --no-root --only main
COPY app/ ./app/
USER 1000
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

The trick — installing dev groups in `deps`, then doing a *separate* `--only main` install in `prod` — keeps the prod image free of pytest and friends while the test stage still has them. Worth it.

## Example / Worked Scenario
The trainer asks: "Build the `test` and `prod` images for `user-service`. Show that the test image can run the suite, and that the prod image is materially smaller."

```bash
docker build --target test -t user-service:test ./services/user-service
docker build --target prod -t localhost:5000/user-service:dev ./services/user-service

# Run the suite from the test image
docker run --rm user-service:test
# > pnpm run test
# > jest
# ... all tests passing

# Compare sizes
docker images user-service:test localhost:5000/user-service:dev
# REPOSITORY                       TAG    SIZE
# user-service                     test   612MB
# localhost:5000/user-service      dev    178MB
```

Roughly a 3-4x size delta is typical. The 600MB+ test image is fine — it never leaves CI. The 180MB prod image is what gets pushed, scanned, pulled by the integration job, and (in the 10-week intensive) deployed.

## Common Pitfalls
- **Copying everything in `deps`.** Keep `COPY package.json pnpm-lock.yaml ./` (or equivalent) tightly scoped so the install layer caches well. Don't `COPY . .` in `deps`.
- **Forgetting `--target` and getting the wrong image.** Without `--target`, you get the last stage. If you reorder stages, default behavior changes silently. Be explicit.
- **Re-installing dev deps in prod by accident.** If your prod stage `FROM build` (instead of `FROM <base>`), you carry all the dev cruft. Production stages should generally start from a clean base.
- **Running tests against the *prod* image.** They'll fail because pytest/jest aren't there. The test image exists precisely so this doesn't happen.
- **Burying environment differences in CI scripts.** If the test container needs `NODE_ENV=test` and the prod container needs `NODE_ENV=production`, set it in the Dockerfile stages, not in shell scripts that wrap `docker run`.

## Key Takeaways
- One Dockerfile, multiple targets: `test` and `prod` are different stages, not different files.
- Test images inherit the build's toolchain; prod images start clean and copy only runtime artifacts.
- Use `--target` explicitly in both local builds and CI.
- The size and surface-area savings on the prod image compound across scans, pulls, and deploys.

---
*Prerequisites: [02-multi-stage-dockerfiles.md](../day-02/02-multi-stage-dockerfiles.md), [04-ci-quality-gates-build-test.md](../day-04/04-ci-quality-gates-build-test.md), [01-container-registries-local-registries-docker-hub-image-tags.md](01-container-registries-local-registries-docker-hub-image-tags.md).*
