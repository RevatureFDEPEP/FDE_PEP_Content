# Container Registries — Local Registries, Docker Hub, Image Tags

> *Day 5: Advanced Container Builds & Multi-Service Integration — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*

## Overview
Through Day 4 you've been building images and running them straight from your local Docker daemon. That works when one developer is producing and consuming an image on one machine. The moment CI builds an image that a different machine has to run — or that you want to ship to a teammate without re-building — you need a **registry**: a network-addressable store of named, tagged images.

The PEP v3.0 substrate uses a **local registry** spun up as a Compose service (you'll see it as `registry:2` on port `5000`). It's the same protocol Docker Hub speaks, the same protocol an enterprise registry like ECR speaks, but it costs nothing and runs entirely on your laptop. Day 7's CI will push tagged images here so that test stages and deploy-like stages can pull them by digest rather than rebuilding from scratch.

This topic is the conceptual scaffolding. Tagging *strategy* is the next topic; multi-stage *production* image authoring is the topic after that. Here we cover: what a registry is, what tags are, how push/pull works, and how the local registry fits the curriculum.

## What a Registry Actually Is
A registry is a content-addressable HTTP server that stores image layers and the manifests that point at them. When you `docker push myimage:v1`:

1. The daemon computes the SHA256 digest of each layer.
2. It checks with the registry whether each digest is already present. Existing layers are skipped — registries are deduplicated.
3. Missing layers are uploaded.
4. Finally, a manifest is uploaded that names the layers, sets architecture/OS metadata, and is associated with the tag `v1` under the repository `myimage`.

When you `docker pull myimage:v1`, the reverse happens. The daemon fetches the manifest, then the layers it doesn't already have locally.

The key mental model: **a tag is a mutable pointer; a digest is an immutable identifier**. `myimage:v1` can be re-pushed and now mean a different set of layers; `myimage@sha256:abc123...` cannot. This distinction underpins the next topic on tagging strategy.

## Registries You'll Encounter
- **Docker Hub** (`docker.io`) — the public default. When you `docker pull postgres:16`, you're hitting Docker Hub. Rate-limited for unauthenticated pulls; in CI this surfaces as flaky pulls on Monday morning. Day 3 already touched on caching upstream images for this reason.
- **GitHub Container Registry** (`ghcr.io`) — tied to GitHub auth, common pairing with GitHub Actions.
- **AWS ECR** — what production rev-eval-ai used. Stripped from the PEP variant; mentioned here so you recognize the name when you graduate to the 10-week intensive.
- **Local registry** (`localhost:5000`) — what this cohort uses. Same API surface as the others, no auth required by default, no network egress, ephemeral by design.

For this week, treat the local registry as a **stand-in for any of the above** — the verbs and the mental model transfer cleanly.

## Image References: The Anatomy
A fully-qualified image reference has four parts:

```
[registry-host[:port]/][namespace/]repository[:tag | @digest]
```

Concrete examples from the PEP substrate:

| Reference | Registry | Namespace | Repository | Tag |
|---|---|---|---|---|
| `postgres:16` | Docker Hub (implicit) | `library` (implicit) | `postgres` | `16` |
| `localhost:5000/user-service:dev` | local registry | (none) | `user-service` | `dev` |
| `localhost:5000/api-gateway:sha-a1b2c3d` | local registry | (none) | `api-gateway` | `sha-a1b2c3d` |

When the registry portion is omitted, the daemon defaults to Docker Hub. This is why `docker pull ubuntu` works without ceremony — and also why a typo in the registry portion silently sends you to the wrong place.

## Push / Pull Against the Local Registry
The local registry runs as a Compose service. Pushing one of the PEP services looks like:

```bash
# Build, tagging the image with the local-registry-qualified name
docker build -t localhost:5000/user-service:dev ./services/user-service

# Push it
docker push localhost:5000/user-service:dev

# Verify the registry knows about it
curl -s http://localhost:5000/v2/_catalog
# {"repositories":["user-service"]}

curl -s http://localhost:5000/v2/user-service/tags/list
# {"name":"user-service","tags":["dev"]}
```

That `v2/_catalog` endpoint is the same Distribution API every registry speaks. Knowing it exists is occasionally rescue-grade information when CLI tools fail you.

To pull from a teammate's hypothetical published image, the reverse:

```bash
docker pull localhost:5000/user-service:dev
```

In CI on Day 7, the test stage will pull these exact references rather than rebuild — saving minutes per run.

## Where Registries Fit in the Week 2 Picture
Week 2's CI will:

1. **Build** each of the four in-scope services (user-service, question-management-service, test-management-service, api-gateway) into a tagged image.
2. **Push** each image to the local registry under a tag derived from the commit SHA (covered in the tagging-strategy topic).
3. **Pull** those images in subsequent test jobs and an integration job that runs them under Compose.

If you've never used a registry before, today's hands-on is the moment that pipeline stops looking like magic.

## Example / Worked Scenario
The trainer asks: "Push the `api-gateway` image to the local registry under two tags: a SHA-style tag and a `dev` convenience tag. Verify both are queryable."

```bash
# One build, two tags
SHA=$(git rev-parse --short HEAD)
docker build \
  -t localhost:5000/api-gateway:sha-$SHA \
  -t localhost:5000/api-gateway:dev \
  ./services/api-gateway

# Push both tags
docker push localhost:5000/api-gateway:sha-$SHA
docker push localhost:5000/api-gateway:dev

# Verify
curl -s http://localhost:5000/v2/api-gateway/tags/list | jq
# {"name":"api-gateway","tags":["dev","sha-a1b2c3d"]}
```

Note that **only one upload happens** for the layers themselves. The second push is metadata-only because the layers are deduplicated by digest. You can confirm by watching the second `docker push` — it'll report "Layer already exists" for every layer.

## Common Pitfalls
- **Forgetting the registry prefix.** `docker push user-service:dev` (no `localhost:5000/` prefix) tries to push to Docker Hub, which rejects you with an auth error. Always include the registry in the tag at *build* time, not just at push time.
- **Trusting `:latest` to mean what you think.** Pulling `:latest` from any registry gives you whatever was last tagged `latest`, which may not be the newest *built* image. Covered in depth in the tagging-strategy topic.
- **Pushing to a stopped local registry.** The Compose `registry` service has to be up. `docker compose up -d registry` if pushes hang.
- **Confusing the local registry's port with the registry-host port in tags.** `localhost:5000` is the host:port the *daemon* connects to; in CI the Compose-internal hostname might be `registry:5000` instead. The tag has to match the network path the *pusher* uses.

## Key Takeaways
- A registry is a content-addressable HTTP store; tags are mutable pointers, digests are not.
- The local registry behaves like Docker Hub or ECR — the verbs transfer.
- Always include the registry prefix at build time so the tag is push-ready.
- Layers are deduplicated by digest, so re-tagging is essentially free.

---
*Prerequisites: `day-2-docker-fundamentals-images-containers-layers-registries.md`.*
