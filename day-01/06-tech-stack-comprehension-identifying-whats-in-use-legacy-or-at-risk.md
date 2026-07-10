# Tech Stack Comprehension — Identifying What's in Use, Legacy, or at Risk

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

## Overview

By now you have a service map, a brownfield reading discipline, and a navigation pattern for the repository. The last orientation piece is *stack-level awareness*: scanning the libraries and frameworks across the substrate and forming a mental model of what is current, what is legacy, what is risky, and what was deliberately removed. This matters because tomorrow you will start bringing the stack up and Days 3–4 you will debug CI — and being able to say "that's the modern stack, that should work" versus "that library has been deprecated, the error is probably real" speeds every diagnosis.

## The Three Categories

For each library or framework in the substrate, you want to be able to place it in one of three buckets:

- **In use and current.** Actively maintained, modern version, mainstream choice. Treat as working unless evidence says otherwise.
- **Legacy.** Older version of something current, or a once-popular tool that has been superseded. Still works, but be cautious — bugs may be unpatched, examples online may not match, the project may be planning to migrate off it.
- **At risk.** Deprecated, abandoned, vulnerable, or operationally fragile. Avoid building new functionality against it. Be prepared for it to break.

A fourth category sits outside the substrate but is worth naming explicitly:

- **Stripped.** Components that *used to* be part of the parent `rev-eval-ai` repository but were deliberately removed before this cohort saw it. References to them may still appear in old docs or stale code paths. If you see one, do not be confused — they live in the 10-week intensive (Phase 2), not in PEP.

## The Substrate Stack

### Backend (Python services)

- **FastAPI** — modern Python web framework. *In use and current.* This is the HTTP framework for all four services.
- **Pydantic** — data validation library (paired with FastAPI). *In use and current.*
- **SQLAlchemy** — Python ORM, used by services backed by Postgres (user-service, test-management). *In use and current* if SQLAlchemy 2.x; *legacy* if pinned to 1.x. Check `pyproject.toml`.
- **Alembic** — SQLAlchemy's migration tool. *In use and current.*
- **Pymongo / Motor** — Mongo drivers, used by question-management. *In use and current.*
- **uv / pip / Poetry** — dependency tool. The project uses `pyproject.toml`; check the lockfile name to confirm which resolver runs it.
- **pytest** — test runner. *In use and current.*

### Frontend

- **Next.js 16** — React framework. *In use and current* — this is a recent major version, with App Router as the default and React Server Components.
- **React** — UI library, peer dependency of Next.js.
- **TypeScript** — language. *In use and current.*
- **Tailwind CSS** — utility-first styling. *In use and current* (assuming v3.x or v4.x).
- **pnpm** — package manager. *In use and current.*

### Data and Infrastructure

- **PostgreSQL** — relational DB. *In use and current.*
- **MongoDB** — document DB. *In use and current.*
- **MinIO** — S3-compatible object storage for local dev. *In use and current.*
- **Docker / Docker Compose v2** — local container orchestration. *In use and current.*
- **GitHub Actions** — CI. *In use and current.*

### Stripped (do not expect to find working, ignore if you see references)

The original `rev-eval-ai` repository contained an AI surface that was deliberately removed before PEP. References you might see in stale docs but should *not* expect to find functional:

- `ai-quiz-service` and `ai-interview-service` — AI Lambdas.
- LangChain / LangGraph orchestration.
- AWS Bedrock model invocation code.
- ElevenLabs voice integration.
- Kubernetes deployment manifests (PEP uses Docker Compose only).

These belong to Phase 2 (the 10-week intensive). If you see a reference, it's residue.

## How to Read a `pyproject.toml` or `package.json` for Stack Awareness

Open the manifest. Scan the dependency list. For each entry:

1. **Recognize the role.** "FastAPI = web framework." "SQLAlchemy = ORM." "Pytest = test runner." If you don't know a library, that's a Claude Code prompt: *"What is `<library>` and what is it typically used for?"*
2. **Check the version.** Major versions matter. `sqlalchemy = "^2.0"` is modern; `sqlalchemy = "^1.4"` is legacy. `next: "^16"` is current; `next: "^12"` would be at risk.
3. **Flag anything you'd hesitate to build against.** Unmaintained packages, packages with known CVEs, packages that have been superseded (e.g., `requests` is fine but `urllib3` directly is unusual; `moment.js` would be legacy; `aiohttp` is fine but mixing it with `httpx` in the same service is a smell).

Do this once per service, in 5–10 minutes. By the end you have a mental table: "user-service is FastAPI + SQLAlchemy 2 + pytest, all current. Question-management is FastAPI + Motor, current."

## Why This Matters Operationally

When something breaks in the next four weeks — and things will break, by design (the seeded CI bugs, the Compose stack-up friction) — your diagnostic speed depends on stack awareness:

- **A `from sqlalchemy.ext.declarative import declarative_base` import error** in a SQLAlchemy 2.x project is immediately suspicious — that import path moved in 2.x. If you didn't know the version was 2.x, the error looks generic.
- **A Next.js page that doesn't render** behaves very differently between Pages Router (older Next.js) and App Router (Next.js 13+). Knowing this is Next.js 16 / App Router tells you which mental model applies.
- **A Mongo query that returns `None` instead of a document** behaves differently between sync `pymongo` and async `motor`. Knowing which is in use tells you whether you need to `await`.

Stack awareness is force-multiplier knowledge: a small upfront cost that pays back many times across debugging sessions.

## Example / Worked Scenario

You are asked to scan the substrate and report what's in use, what's legacy, and what's at risk. The 10-minute version:

1. Open `services/user-service/pyproject.toml`. List dependencies. Note: FastAPI 0.110+, SQLAlchemy 2.x, Pydantic v2, pytest, alembic. *All current.*
2. Open `services/question-management/pyproject.toml`. Note: FastAPI, Motor (async Mongo driver), Pydantic, pytest. *All current.*
3. Open `services/test-management/pyproject.toml`. Note: FastAPI, SQLAlchemy 2.x, Pydantic, httpx (for calling question-management), pytest. *All current.*
4. Open `services/api-gateway/pyproject.toml`. Note: FastAPI, httpx, python-jose or PyJWT for token verification. *All current.*
5. Open `frontend/package.json`. Note: next 16.x, react 19.x, typescript 5.x, tailwindcss 3.x or 4.x. *All current.*
6. Skim `docs/` for any references to `bedrock`, `langchain`, `elevenlabs`, `k8s`. *Any hits are stripped components — note as residue.*

Output of the exercise: a one-paragraph summary like:

> *"The PEP substrate runs on a modern Python stack (FastAPI + Pydantic v2 + SQLAlchemy 2 / Motor) and a modern frontend stack (Next.js 16 + React 19 + TypeScript 5 + Tailwind). Postgres, Mongo, MinIO via Docker Compose v2; CI via GitHub Actions. No legacy or at-risk components observed in the in-tree code. Residual references to AI integration (Bedrock, LangChain, ElevenLabs) and Kubernetes appear in older docs — these are the stripped Phase-2 components and can be ignored in PEP."*

That paragraph is the artefact you want to be able to produce by end of Day 1.

## Common Pitfalls

- **Assuming "modern stack" means "no bugs."** The substrate is intentionally seeded with CI bugs (Days 3–4) and may have other latent issues. A current stack reduces *categories* of risk; it does not eliminate them.
- **Confusing stripped components with broken components.** If you find a stray reference to LangChain in a `docs/` markdown file but no `langchain` in any `pyproject.toml`, you have found residue, not a broken integration. Don't try to "fix" stripped components.
- **Flagging unfamiliar libraries as "at risk" by default.** "I haven't heard of it" is not the same as "it's risky." Look it up before flagging.
- **Mixing async and sync HTTP clients in the same service.** If a service has both `requests` and `httpx`, that's a small smell — usually a sign of incomplete migration. Note it; don't fix it this week.
- **Treating version numbers as decoration.** `FastAPI 0.95` and `FastAPI 0.110` have meaningfully different APIs in places. Major+minor matters. Read the lockfile if the manifest uses caret ranges.

## Key Takeaways

- Sort every library/framework in the substrate into one of three buckets: *in use and current*, *legacy*, or *at risk*. (Plus a fourth, *stripped*, for components removed before the cohort saw them.)
- The PEP substrate stack is intentionally modern: FastAPI + Pydantic v2 + SQLAlchemy 2 / Motor on the backend; Next.js 16 + React + TypeScript + Tailwind + pnpm on the frontend; Postgres + Mongo + MinIO + Docker Compose v2 + GitHub Actions for infra.
- AI integration (LangChain, Bedrock, ElevenLabs), the AI services, and Kubernetes have been *stripped* — references in old docs are residue and live in Phase 2, not PEP.
- Read `pyproject.toml` and `package.json` per service in 5–10 minutes total — this small upfront investment accelerates every debugging session in Weeks 2–4.
- Stack awareness is force-multiplier knowledge: knowing "this is SQLAlchemy 2" or "this is App Router Next.js 16" tells you which mental model to apply when something goes wrong.

---
*Prerequisites: [05-codebase-navigation-conventions.md](05-codebase-navigation-conventions.md), [03-microservices-architecture-and-service-boundaries.md](03-microservices-architecture-and-service-boundaries.md). Sets up Day 2 (Docker Compose stack-up) and Days 3–4 (CI debugging).*
