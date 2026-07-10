# Codebase Navigation Conventions

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

## Overview

You now have a conceptual picture of the substrate (service map, brownfield reading discipline) — this topic is the *literal map*. Where do the services actually live on disk? Where are dependency manifests? Where is documentation kept? Where do tests sit? Knowing the physical layout of the repository turns "I'm lost in the codebase" into "give me 30 seconds, I know exactly which file to open." That's what this topic is for.

## The Top-Level Layout

The `rev-eval-ai` PEP variant is structured as a monorepo — every service plus the frontend lives in one Git repository, in subdirectories. The top-level layout looks roughly like this (file/folder names are illustrative — check your actual checkout):

```
rev-eval-ai/
├── README.md                  <-- start here, always
├── docker-compose.yml         <-- the stack definition
├── .github/
│   └── workflows/             <-- CI pipelines (seeded bugs Days 3-4)
├── docs/                      <-- architecture notes, ADRs
├── services/
│   ├── user-service/
│   ├── question-management/
│   ├── test-management/
│   ├── api-gateway/
│   └── reporting-and-analytics/  <-- empty; you build this Week 4
├── frontend/                  <-- Next.js 16 app
├── scripts/                   <-- helper scripts (DB seeding, etc.)
└── .env.example               <-- environment variable template
```

The monorepo pattern means: one clone, one branch, one PR can span multiple services. That is a feature — coordinated changes across service boundaries become one atomic unit of review.

## Service Subdirectory Anatomy

Each Python service follows roughly the same internal layout:

```
services/user-service/
├── pyproject.toml             <-- dependency manifest (Python)
├── Dockerfile                 <-- how this service builds into an image
├── README.md                  <-- service-specific docs (if present)
├── src/
│   └── user_service/
│       ├── __init__.py
│       ├── main.py            <-- entry point: FastAPI app + routes
│       ├── models.py          <-- data models (SQLAlchemy / Pydantic)
│       ├── database.py        <-- DB connection setup
│       ├── auth.py            <-- service-specific logic modules
│       └── ...
└── tests/
    └── ...
```

Things to know:

- **`pyproject.toml` is canonical** for Python dependencies — this is where libraries are declared. Not `requirements.txt`; the project uses the modern `pyproject.toml` approach.
- **`main.py`** (or sometimes `app.py`) is the FastAPI entry point. Reading this file tells you what HTTP routes this service exposes — your single best "what does this service do?" file.
- **`Dockerfile`** describes how the service is packaged into a container image. Day 2's Compose work depends on these.
- **`tests/`** is where pytest tests live. Tests are your executable spec.

When you want to know "what does service X do?", the orientation sequence is:

1. `services/X/README.md` (if present)
2. `services/X/src/X/main.py` — the routes
3. `services/X/src/X/models.py` — the data shapes
4. `services/X/pyproject.toml` — the dependencies (tells you what kind of service this is)

## The Frontend Subdirectory

The Next.js 16 frontend has its own layout (Next.js App Router conventions):

```
frontend/
├── package.json               <-- dependency manifest (Node/pnpm)
├── pnpm-lock.yaml             <-- lockfile (do not edit by hand)
├── next.config.{js,ts}        <-- Next.js config
├── tsconfig.json              <-- TypeScript config
├── Dockerfile
├── src/
│   ├── app/                   <-- App Router pages and layouts
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── ...
│   ├── components/            <-- shared React components
│   ├── lib/                   <-- utilities, API clients
│   └── styles/
└── public/                    <-- static assets
```

Important conventions:

- **`package.json` declares dependencies**; `pnpm-lock.yaml` pins exact versions. Always use `pnpm install` — never `npm install` — to keep the lockfile honest.
- **`src/app/`** is where pages live under Next.js App Router. A folder maps to a URL segment; `page.tsx` inside it is the rendered page.
- **`src/lib/`** typically holds the API client code — the functions that call the api-gateway. This is your "where does the frontend talk to the backend?" file.

## Where Documentation Lives

Three tiers of documentation in this codebase:

1. **Top-level `README.md`.** Overview of the whole system — what it is, how to bring it up, what scripts exist. Always start here on first clone.
2. **`docs/` folder.** Longer-form docs — architecture notes, ADRs (Architecture Decision Records explaining *why* a particular design choice was made), runbooks for operational tasks.
3. **Per-service `README.md`.** Service-specific docs — what this service is for, its API contract, anything operationally noteworthy. Not every service has one; their presence is uneven, which is normal for inherited codebases.

If a doc disagrees with the code, the code is the truth — but the doc is a clue to what the system was *trying* to be.

## Where Tests Live

Tests follow the layout of the code they test:

- **Per-service:** `services/X/tests/` for unit tests of that service.
- **Frontend:** `frontend/__tests__/` or co-located `*.test.tsx` files alongside components, depending on convention.
- **Cross-service / integration tests:** if they exist, usually in a top-level `tests/` folder or under `scripts/`.

CI in `.github/workflows/` runs the per-service tests in parallel jobs. Days 3–4 have you debug those workflows — you will become very familiar with that directory.

## Useful Navigation Habits

A few habits that pay off immediately:

- **Use file-search shortcuts in your editor.** VS Code's `Ctrl/Cmd+P`, JetBrains' `Shift Shift`, etc. Typing a partial filename is faster than clicking through the tree.
- **Use grep / ripgrep / your editor's "find in files".** When you don't know where something lives, search for a string you know is in it. `rg "verify_token" services/` finds every place that function is defined or called.
- **Use Claude Code to map an unfamiliar area.** "Read the test-management service and tell me what endpoints it exposes" is a one-prompt orientation.
- **Read `pyproject.toml` and `package.json` early.** Dependencies tell you what kind of service you're looking at faster than reading code does. FastAPI? It's a Python HTTP service. SQLAlchemy? Relational data. Pymongo? Document data. Next.js + Tailwind? UI with a particular styling approach.
- **Check `git log --oneline -- <path>` for any file you're about to change.** Two minutes of history-skimming often answers more than two hours of code-reading.

## Example / Worked Scenario

A trainer asks you: *"Where does the question-management service handle the 'create a new question' endpoint, and what database table or collection does it write to?"*

The navigation sequence:

1. Open `services/question-management/src/question_management/main.py`. Look for an HTTP route decorator on something POST-shaped — perhaps `@app.post("/questions")`.
2. That handler calls into something — follow the import to find which module owns the create logic.
3. That module imports from `models.py` or `database.py`. Open them: this is a Mongo-backed service, so you'll see a collection name, perhaps something like `db.questions`.
4. Confirm by checking `pyproject.toml` — you should see `pymongo` (or `motor`) as a dependency, confirming it's a Mongo service.

End-to-end: 60–90 seconds if you know the navigation pattern. Several minutes of clicking around if you don't.

## Common Pitfalls

- **Looking for `requirements.txt`.** This project uses `pyproject.toml` (the modern Python standard). `requirements.txt` may not exist or may be a generated artefact, not the source of truth.
- **Using `npm install` in the frontend.** It will write a `package-lock.json` and a non-pnpm-shaped `node_modules`, conflicting with the existing `pnpm-lock.yaml`. Use `pnpm install`.
- **Assuming all services have a README.** Some do, some don't. When a per-service README is missing, the entry-point file (`main.py`) is the next-best orientation.
- **Treating `docs/` as comprehensive.** Documentation in inherited codebases is always partial and often stale. Use it for direction, verify against code.
- **Ignoring `.env.example`.** Environment variables are how the services are configured — DB URLs, secrets, ports. `.env.example` is the template; you'll create your own `.env` from it tomorrow when you stand the stack up. Skipping this means services start with wrong/missing config.
- **Editing lockfiles by hand.** `pnpm-lock.yaml` and Python lockfiles are generated. If they need to change, you regenerate them via the package manager — never edit them in your editor.

## Key Takeaways

- The substrate is a monorepo: services under `services/`, frontend under `frontend/`, CI under `.github/workflows/`, infrastructure (Compose) at the top level.
- Each Python service follows a consistent layout: `pyproject.toml` for dependencies, `Dockerfile` for packaging, `src/<service>/main.py` for the HTTP entry point, `tests/` for tests.
- `pyproject.toml` (Python) and `package.json` (frontend) are the canonical dependency manifests; always use `pnpm` (not `npm`) for the frontend.
- Documentation is tiered (top-level README, `docs/`, per-service READMEs); not all of it is current, and code is the source of truth when they disagree.
- A fast navigation pattern — partial-filename file search, ripgrep for content, `git log` for history, Claude Code for area-summaries — turns "I'm lost" into "I know where to look" within Day 1.

---
*Prerequisites: [03-microservices-architecture-and-service-boundaries.md](03-microservices-architecture-and-service-boundaries.md), [04-brownfield-mindset-reading-inherited-code-finding-the-seams.md](04-brownfield-mindset-reading-inherited-code-finding-the-seams.md).*
