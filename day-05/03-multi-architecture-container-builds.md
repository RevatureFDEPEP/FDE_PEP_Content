# Multi-Architecture Container Builds

> *Day 5: Advanced Container Builds & Multi-Service Integration — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Half the cohort is on Apple Silicon (`arm64`). The CI runners are `linux/amd64`. The prod targets in the 10-week intensive are also `linux/amd64`. This mismatch is invisible until it isn't — typically when an `arm64`-built image is pulled by an `amd64` runner and fails to start with the cryptic `exec format error`.

PEP v3.0 does **not** ship multi-arch images. The substrate is single-architecture and that is fine. But you will hit this seam in the wild within weeks of graduating, and trainers in the 10-week intensive expect you to recognize it on sight. So today's objective is conceptual: **understand when multi-architecture builds matter and how `docker buildx` produces them.** No required hands-on multi-arch build, but the trainer may demo one.

## What "Architecture" Means in a Container Context
A container image is **not** OS-independent in the way the marketing implies. Each layer contains compiled binaries (or interpreted-language runtimes that are themselves compiled binaries). Those binaries are built for a specific instruction set: `amd64` (Intel/AMD x86-64), `arm64` (Apple Silicon, AWS Graviton, Raspberry Pi 4+), `arm/v7` (older Pi, embedded), etc.

When you `docker build` on your Apple Silicon laptop without flags, you get an `arm64` image. When that same image lands on an `amd64` runner, the kernel tries to execute `arm64` instructions natively, fails, and reports `exec format error`. The image is *technically* a valid container image — it's just compiled for the wrong CPU.

A **multi-architecture image** is really a *manifest list*: one tag (`api-gateway:dev`) that points at *multiple* per-architecture manifests under the hood. When a node pulls it, the daemon picks the manifest matching the node's architecture.

```
api-gateway:dev   (manifest list)
   |-- linux/amd64 --> manifest A --> layers compiled for x86_64
   |-- linux/arm64 --> manifest B --> layers compiled for arm64
```

## When It Actually Matters
- **Mixed-arch fleets.** Apple Silicon developers + `amd64` CI + `amd64` (or Graviton `arm64`) production. Most modern engineering orgs.
- **Public open-source images.** Postgres, Redis, Node, Python — these ship multi-arch precisely because they don't know what architecture you'll pull from.
- **Edge deployments.** ARM-based industrial gear, Pi clusters.

When it **doesn't** matter:
- **Single-arch, controlled environment.** PEP v3.0. Everyone targets `linux/amd64` because that's what CI and (future) prod use.
- **Pure-interpreted-language images where every layer is from a multi-arch base.** Even here, native modules (`bcrypt`, `argon2`, anything via `node-gyp`) can sneak in arch-specific compiled bits.

Rule of thumb: **if any consumer of your image is on a different CPU architecture than the machine that built it, you need multi-arch — or you need to be explicit about the target arch and let consumers know.**

## Producing an `amd64`-Targeted Build From Apple Silicon
This is the *common* case for the PEP cohort. Apple Silicon developer; image needs to run on `amd64` CI. The cheap solution is to build *for* `amd64` even though you're *on* `arm64`:

```bash
docker build --platform linux/amd64 -t localhost:5000/user-service:dev ./services/user-service
```

Under the hood this uses QEMU emulation to run `amd64` instructions during the build. It's slower than native — sometimes painfully so for compile-heavy steps — but it produces an image that will run on the target arch without surprise.

Most cohort members on Apple Silicon will end up with `--platform linux/amd64` baked into their muscle memory by Day 5.

## Producing a True Multi-Arch Image with `buildx`
For a real multi-arch image, the tool is `docker buildx` (built into Docker Desktop, available as a plugin elsewhere):

```bash
# One-time: create a builder with QEMU support
docker buildx create --name pep-builder --use
docker buildx inspect --bootstrap

# Build and push (multi-arch images must go straight to a registry —
# the local Docker image store can only hold one arch at a time)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag localhost:5000/api-gateway:dev \
  --push \
  ./services/api-gateway
```

A few important constraints:
- **Multi-arch images cannot live in your local image store.** They must be pushed to a registry. The `--push` flag is mandatory; `--load` is not supported for multi-platform builds.
- **Build time roughly doubles** (or more, with emulation).
- **Layer cache is per-architecture.** A change that invalidates the cache on one arch invalidates it on the other.

## Inspecting the Result
You can confirm a manifest list with `docker buildx imagetools inspect`:

```bash
docker buildx imagetools inspect localhost:5000/api-gateway:dev
# Name:      localhost:5000/api-gateway:dev
# MediaType: application/vnd.docker.distribution.manifest.list.v2+json
# Manifests:
#   - linux/amd64
#   - linux/arm64
```

If you ever see one consumer report `exec format error` while others succeed, this is the command you run.

## Example / Worked Scenario
The trainer asks: "Identify whether the team needs multi-arch builds for PEP v3.0, and if not, what single command Apple Silicon developers should use so their pushed images run on the `amd64` CI runners."

**Analysis.** PEP v3.0 ships a single architecture target (`linux/amd64`) for both CI and the future 10-week intensive. Multi-arch is not required. However, half the cohort is on `arm64` laptops. Without intervention, those developers will produce `arm64` images that fail on CI.

**Decision.** Single-arch builds, but every Apple Silicon developer adds `--platform linux/amd64` to their build commands.

**Concrete fix.** In each service's Makefile (or developer cheatsheet):

```bash
# services/user-service/Makefile
build:
	docker build --platform linux/amd64 -t localhost:5000/user-service:dev .
```

Specifying `--platform` even when the host arch already matches is a no-op, so this is safe for both `amd64` and `arm64` developers. It eliminates an entire class of "works on my machine" bug.

**When the team would revisit.** If the 10-week intensive's prod target moved to Graviton (`arm64`) while CI stayed `amd64`, or if part of the team deployed locally on Apple Silicon while pushing to `amd64` prod — at that point, true multi-arch via `buildx` becomes worth its build-time cost.

## Common Pitfalls
- **Building on Apple Silicon, pushing, and discovering CI breaks.** Default `docker build` on `arm64` produces `arm64` images; the registry happily stores them; the `amd64` CI runner fails on pull. Use `--platform linux/amd64` or `buildx` multi-arch.
- **Assuming `latest` means "for my arch".** Once an image is in a registry, its manifest determines what gets pulled. A single-arch `arm64` image tagged `latest` will fail on `amd64` pulls regardless of how confidently it's named.
- **Forgetting `--push` with `buildx` multi-arch.** Without it, the build runs and then errors because the result has nowhere to land. Use `--push` or `--output type=registry,...`.
- **Native modules in Node/Python.** A pure-JS Node app may build cleanly multi-arch; the moment `bcrypt` or `sharp` is in dependencies, the build needs platform-correct native bindings, and emulation cost rises.
- **Treating `--platform` as a runtime flag.** `docker run --platform linux/amd64 ...` exists, but it's for forcing emulated *execution* of a non-native image. Use it for debugging, not as a substitute for building correctly.

## Key Takeaways
- A container image is compiled for a specific CPU; `exec format error` is the canonical symptom of an arch mismatch.
- PEP v3.0 is single-arch; Apple Silicon developers should `--platform linux/amd64` to match CI.
- Multi-arch images are *manifest lists*; they must be pushed to a registry, not stored locally.
- `docker buildx imagetools inspect` is the diagnostic command when arch is in question.

---
*Prerequisites: [01-docker-fundamentals-images-containers-layers-registries.md](../day-02/01-docker-fundamentals-images-containers-layers-registries.md), [01-container-registries-local-registries-docker-hub-image-tags.md](01-container-registries-local-registries-docker-hub-image-tags.md).*
