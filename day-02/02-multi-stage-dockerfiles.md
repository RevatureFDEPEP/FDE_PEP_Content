# Multi-Stage Dockerfiles

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Every backend service in the `rev-eval-ai` PEP substrate uses a multi-stage Dockerfile, and the frontend does too. The point is to keep production images small (no build toolchain, no source-only artifacts) without sacrificing build-time flexibility. You will read these Dockerfiles confidently today, debug them on Day 3 when CI hits them, and extend them on Day 5.

## Why Multi-Stage

Recall from the previous topic that layers are additive. If you `pip install`, then `pip uninstall`, the bytes are still in the earlier layer. If you compile a Next.js app with `node_modules` and dev dependencies, those 400 MB stick in the final image unless you actively shed them.

A **multi-stage build** uses multiple `FROM` directives in one Dockerfile. Each `FROM` starts a fresh image. You build in one stage, then `COPY --from=<stage>` only the artifacts you want into a final, lean stage. The intermediate stages are discarded — they never ship.

The two universal benefits:

1. **Smaller final images.** A Python service can drop from ~900 MB to ~150 MB; a Next.js image from ~1.2 GB to ~200 MB.
2. **Smaller attack surface.** No compilers, no package managers, no source files an attacker can pivot from in the production image.

## Anatomy of a Multi-Stage Dockerfile

The pattern across the substrate's Python services looks like this:

```dockerfile
# ---------- Stage 1: builder ----------
FROM python:3.11-slim AS builder

WORKDIR /build

# System build deps (compilers for native wheels)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
 && rm -rf /var/lib/apt/lists/*

# Install uv, then install project deps into a venv we control
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv venv /opt/venv \
 && uv pip install --python /opt/venv/bin/python .

# ---------- Stage 2: runtime ----------
FROM python:3.11-slim AS runtime

WORKDIR /app

# Copy ONLY the prebuilt venv and the source code
COPY --from=builder /opt/venv /opt/venv
COPY app/ ./app/

ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Key reading points:

- `AS builder` names the stage so later stages can reference it.
- `build-essential` (compilers) lives only in the builder; the runtime image never sees it.
- `COPY --from=builder /opt/venv /opt/venv` lifts just the prepared virtualenv into runtime.
- The final stage has no `pip`, no `uv`, no source manifest — just the venv and the application code.

The frontend uses the same pattern with three stages: `deps` (install `node_modules`), `build` (run `next build`), `runner` (copy the `.next/standalone` output plus the trimmed `node_modules`).

## Example / Worked Scenario

Open `services/question-management-service/Dockerfile`. You will find roughly:

```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /build
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv venv /opt/venv && uv pip install --python /opt/venv/bin/python .

FROM python:3.11-slim AS runtime
WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY app/ ./app/
ENV PATH="/opt/venv/bin:$PATH"
EXPOSE 8001
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001"]
```

Build and compare:

```bash
docker build -t qms:full --target builder services/question-management-service
docker build -t qms:runtime services/question-management-service
docker image ls | grep qms
```

Expect the `builder` image to be ~2x the size of the `runtime` image. That delta is what multi-stage saves you on every pull, every push, and every cold start.

To debug a build that fails inside the builder stage, target it directly:

```bash
docker build --target builder -t qms:debug services/question-management-service
docker run --rm -it qms:debug bash
# now poke at /build, run uv pip install manually, etc.
```

## Common Pitfalls

- **Copying `node_modules` or `.venv` from the host.** Your `.dockerignore` should exclude these. If it doesn't, the host's platform-specific binaries get baked into the image and break inside the container (especially on Windows hosts targeting Linux containers).
- **Forgetting `PATH` in the runtime stage.** If you copy a venv but don't put it on `PATH`, the container runs system Python without your deps and dies with `ModuleNotFoundError`.
- **Naming stages but never referencing them.** `AS builder` is documentation when nothing copies `--from=builder`. Make sure the runtime stage actually pulls from named earlier stages.
- **One giant `RUN` chain in the builder.** It works but kills cache granularity. Keep dependency install separate from source copy so a code edit doesn't reinstall everything.

## Key Takeaways

- Multi-stage Dockerfiles use multiple `FROM` directives to build in one image and ship a different, smaller one.
- The runtime stage should contain only what you need to run: interpreter, dependencies, source. No compilers, no package managers.
- `COPY --from=<stage>` is the seam that selectively lifts artifacts forward.
- Use `--target <stage>` to debug intermediate stages in isolation.

---
*Prerequisites: [01-docker-fundamentals-images-containers-layers-registries.md](01-docker-fundamentals-images-containers-layers-registries.md)*
