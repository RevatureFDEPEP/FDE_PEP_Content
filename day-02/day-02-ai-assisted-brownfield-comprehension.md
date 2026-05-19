# AI-Assisted Brownfield Comprehension — Reading Inherited Services with Claude Code

> *Day 2: Local Dev Stack with Docker Compose — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*
> ⚡ *AI Tooling Thread (Unit 0)*

## Overview
Day 1 introduced Claude Code as a first-class agentic tool and established the trust boundary. Today is the first hands-on application: using Claude Code to comprehend the inherited `rev-eval-ai` substrate faster than you could by reading it cold. The goal is **comprehension**, not generation — you are asking the agent to explain code that already exists so you can act on it confidently. The deliverable depends on the agent helping you read a Compose file or a service entrypoint and verifying what you learn against the source.

## Why This Matters on Day 2

The substrate is roughly 40 files of Compose + Dockerfiles + service-bootstrap code that you have ten minutes to make sense of. Cold reading is slow; cold reading while the rest of the cohort moves on is worse. Claude Code condenses that initial read into a guided tour — *if* you ask it the right questions and *if* you verify what it tells you against the file.

The cohort's Day 1 brownfield mindset ("read 5–10x more than you write, find the seams, don't refactor on sight") still applies. Claude Code accelerates the reading; it does not replace your judgment.

## The Comprehension Workflow

A repeatable four-step loop:

1. **Frame the question with scope.** Tell Claude Code which files to look at. Vague prompts ("explain the project") produce vague answers; scoped prompts ("read `docker-compose.yml` and list each service, its image, and what it depends on") produce verifiable ones.
2. **Ask for explanation tied to file paths.** A good answer cites the file and line. If the agent says something happens but cannot point at where, treat the claim as a hypothesis to check.
3. **Verify against the source.** Open the file. Confirm. If the agent's claim disagrees with the file, the file wins — every time.
4. **Iterate with follow-up questions.** Use the agent's answer as a map and ask deeper questions about specific seams ("show me where `user-service` reads `JWT_SECRET` and what happens if it's missing").

The verification step is the trust boundary in practice. The agent will sometimes be wrong, especially about details. Catching that on Day 2 against a Compose file is much cheaper than catching it on Day 14 against production code.

## Worked Example — A Claude Code Session

A concrete session for today's deliverable. After cloning the substrate, ask Claude Code:

> **Prompt:**
> Read `docker-compose.yml` and produce a table with one row per service. Columns: service name, image (or build context), exposed host port, key environment variables (just names, not values), and which other services it `depends_on` (with the condition if any). Cite the line numbers for each row so I can verify.

Claude Code will run `Read` on `docker-compose.yml`, parse it, and respond with something like:

| Service | Image / Build | Host Port | Key Env | Depends On |
|---|---|---|---|---|
| postgres | `postgres:16-alpine` | 5432 | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | — |
| user-service | build `./services/user-service` | 8000 | `DATABASE_URL`, `JWT_SECRET` | postgres (healthy) |
| ... | ... | ... | ... | ... |

**Then verify.** Open `docker-compose.yml`, find the `user-service` block, and confirm those env vars exist and `depends_on: postgres: condition: service_healthy` is present on the cited lines. If it matches, you have a structured mental map of the stack in a fraction of the time. If it does not, you have found a discrepancy worth investigating — sometimes a Compose file is more or less complete than the agent assumes.

A useful follow-up after the table:

> **Prompt:**
> Trace what happens when `api-gateway` receives a login request. Identify the relevant files in `services/api-gateway/` and explain the call path to `user-service`. Cite each file and the line where the call is made.

The agent will read the gateway's route handlers and HTTP client and produce a flow. **Open each cited file and confirm.** This is how you build a verified mental model: agent-proposed path → human-verified.

## Useful Prompts for Day 2

A few that map directly to today's deliverable:

- "Read `services/user-service/Dockerfile` and explain why it has multiple stages. What is in the runtime stage that wasn't in the builder, and vice versa?"
- "List every named volume declared in `docker-compose.yml`, the service that mounts it, and the mount path. Tell me which would survive `docker compose down` and which would not."
- "I ran `docker compose up` and `question-management-service` is stuck at `health: starting`. Read the service's Dockerfile and its compose entry. What healthcheck is configured and what command does it run? Is the command actually available in the image?"
- "Compare the healthcheck configurations across `postgres`, `mongo`, and `minio`. Are the `start_period` values reasonable for each? Cite the lines."

Notice the pattern: each prompt names the file(s), asks a specific question, and asks for citations. That is the format that produces verifiable answers.

## What the Agent Is Good At vs Not

**Good at:**
- Summarizing structure across many files (Compose, multi-service Dockerfiles).
- Surfacing seams: "where is X configured," "what calls Y," "what reads env var Z."
- Explaining unfamiliar syntax (a healthcheck `test:` form, a multi-stage `COPY --from=`).
- Drafting a diagnostic plan when something is broken.

**Not as good at (verify especially carefully):**
- Exact version numbers and minor syntax details (Compose v2 vs v3 differences, deprecated fields).
- Behavior of third-party images at runtime — the agent reads source files, it does not always know what `postgres:16-alpine`'s exact entrypoint does.
- Anything that depends on *your* local environment (which port is free, what's in your `.env`).

The trust boundary from Day 1 applies: writes and executions require your approval. For pure comprehension queries, the agent is reading files — low-risk — but its conclusions still need to be checked against the source.

## Common Pitfalls

- **Asking without scope.** "Explain the project" gets a vague summary. "Read these three files and answer this specific question" gets useful, verifiable output.
- **Trusting without verifying.** A confident answer is not a correct answer. Every claim with a citation is a verifiable claim; treat uncited claims as hypotheses.
- **Using the agent for things you should learn directly.** "What does `depends_on` mean in Compose?" — read the Compose orchestration topic above. The agent is for navigating *this* codebase, not for replacing fundamentals.
- **Ignoring the agent's "I don't know."** When Claude Code declines to make a claim or flags uncertainty, that signal is more valuable than a guess. Treat hedges as evidence, not weakness.

## Key Takeaways

- Use Claude Code to comprehend the inherited substrate, not to generate new code on Day 2.
- Scope prompts to specific files and ask for citations; structured, citable answers are verifiable answers.
- The agent proposes; the source code disposes. Always verify against the file.
- Best on structural overviews and seam-finding; weakest on third-party runtime behavior and local environment specifics.

---
*Prerequisites: day-1-ai-augmented-development-claude-code-agent-tooling-fundamentals.md, day-1-brownfield-mindset-reading-inherited-code-finding-the-seams.md, day-2-local-orchestration-with-docker-compose.md*
