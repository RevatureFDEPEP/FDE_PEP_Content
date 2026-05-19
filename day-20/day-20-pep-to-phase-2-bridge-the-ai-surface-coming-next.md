# PEP to Phase 2 Bridge — the AI Surface Coming Next

> *Day 20: Capstone Demo Day — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

PEP's four weeks worked on a deliberately stripped substrate. On Day 1, candidates were told the AI surface — services, libraries, infrastructure — was removed for PEP and would return in the 10-week intensive. By Day 20 most of the cohort has forgotten the strip-list; the substrate they've been working in just feels like "the codebase." This topic re-contextualizes that. The cohort built four slices on top of a foundation, and **the foundation is about to grow** — significantly. What was stripped comes back in Phase 2 and the candidate needs to know what's coming so they can prepare.

This topic is briefing, not skill-building. The discipline is in *understanding the map* — what got removed, why, what's coming back, and where the architectural seams are that Phase 2 will reopen.

## Objective

Explain what was stripped from the substrate for PEP and what returns in Phase 2.

## The Original `rev-eval-ai` Substrate

Before PEP existed, the codebase was called `rev-eval-ai`. It was a quiz + interview platform where:

- Candidates took quizzes (the surface PEP has built on).
- Candidates also did **AI-graded free-response questions** — type an answer, an LLM scores it against a rubric.
- Candidates also did **AI-driven interviews** — a voice-and-text agent conducted live technical interviews with branching follow-ups based on prior answers.
- Trainers configured rubrics, interview templates, and reviewed AI scoring with human override.

That was the original product surface. It included substantial AI machinery — language models, voice synthesis, agentic flows, vector embeddings, prompt orchestration. For a 4-week brownfield onboarding program, that surface area was too much; the PEP version was built by **stripping the AI machinery out** so the cohort could focus on full-stack web fundamentals before adding AI fundamentals.

## What Was Stripped

The Day 1 strip-list, recapped:

### Services removed

- **`ai-quiz-service`** — the FastAPI service that handled free-response question grading. It called an LLM with the question, the candidate's answer, and a rubric, and returned a numeric score with explanation. Removed because LLM-graded questions are out of scope for PEP — multiple-choice and code-completion only.

- **`ai-interview-service`** — the FastAPI service that orchestrated AI-driven live interviews. Held conversation state, generated follow-up questions, integrated voice in and out. Removed because conversational AI is a Phase 2 topic.

### Lambdas removed

- **Quiz-grading Lambda** — async job that picked up free-response submissions from a queue and called `ai-quiz-service` for scoring. Removed alongside the service.

- **Interview-summary Lambda** — async job that took the transcript of a completed interview and produced a structured summary + score. Removed alongside the service.

### Libraries removed from dependencies

- **LangChain** — the LLM orchestration library used in `ai-quiz-service` for prompt templates, output parsing, and the rubric-grading chain. Removed because no service in PEP uses LLMs.

- **LangGraph** — the agentic flow library used in `ai-interview-service` for the interview state machine (intro → question → follow-up → next-question → wrap-up). Removed alongside the service.

- **Bedrock client** (AWS SDK for Anthropic / other foundation models) — the underlying LLM provider client. Removed because no inference happens in PEP.

- **ElevenLabs client** — voice synthesis (text-to-speech for interview agent). Removed because no voice in PEP.

- **Vector store integration** (Pinecone / similar) — used for retrieving relevant grading examples to ground rubric scoring. Removed alongside `ai-quiz-service`.

### Infrastructure removed

- **Kubernetes deployment manifests** — the original substrate ran on EKS. PEP runs on ECS (and locally on Docker Compose) because K8s adds operational complexity that competes with the cohort's attention.

- **GPU-backed inference nodes** — irrelevant once the AI services are removed.

- **Queue infrastructure** (SQS topics for async grading and interview-summary jobs) — removed alongside the Lambdas they served.

### Surface kept (the PEP substrate)

- **user-service** — auth, JWT, RBAC.
- **question-management** — question CRUD on MongoDB.
- **test-management** — sessions, attempts, scoring engine.
- **api-gateway** — path-routing.
- **reporting-and-analytics** — empty scaffold candidates built into on D10 onward.
- **Next.js frontend** — full UI.
- **Postgres + Mongo** — via docker-compose locally, RDS + Atlas in pre-provisioned AWS.
- **GitHub Actions CI** with 5 seeded bugs (D3–D4 entry point).
- **ECR, ECS, RDS, Mongo Atlas, S3** — the deployment target.

The substrate the cohort built on was *intentional*: complete enough to be a real four-slice application, restricted enough that no part of it required AI fluency.

## Why The Strip

The pedagogical reasoning, named so the cohort understands the design:

1. **Surface area matters for onboarding.** A brownfield onboarding is harder than a greenfield course because the cohort has to read code they didn't write. Less surface means more depth on what's left.
2. **AI fluency is itself a topic.** Treating LangChain, LangGraph, Bedrock, and embeddings as "background infrastructure" assumes a fluency the cohort doesn't yet have. Phase 2 *teaches* that fluency rather than assuming it.
3. **Full-stack fundamentals come first.** Server-anchored timers, idempotency, transactions, role gates, server components — these are the foundations Phase 2's AI features will sit on top of. Building the foundations first makes the AI layer comprehensible.
4. **Deployment complexity is a learning gate.** K8s, GPU nodes, queue infrastructure — each one is a learning curve. The PEP substrate uses simpler equivalents (ECS, no GPUs, no queues) so the cohort can focus on the application layer.

## What Returns In Phase 2

The 10-week intensive re-introduces the stripped surface, but not all at once and not all in the same form.

### Returning in the first 2 weeks of Phase 2

- **LLM-graded free-response questions.** The `ai-quiz-service` (or a successor) is reintroduced. Candidates learn prompt engineering, rubric-grounded scoring, output parsing, and the discipline of validating LLM output before trusting it.
- **Bedrock client (or equivalent)** as the inference layer. Candidates learn the provider abstraction, retry semantics, cost monitoring, and the difference between Sonnet/Haiku-tier model selection.
- **Prompt caching** for repeated rubric / context tokens. The cohort will hit cost surprises if they don't learn this early.

### Returning in the middle weeks of Phase 2

- **LangChain** for prompt orchestration, output parsing, and multi-step chains.
- **Vector embeddings + retrieval** for grounding LLM responses in known-correct examples (RAG patterns).
- **Async job patterns** — Lambdas, queues — return because LLM inference is latency-bound and shouldn't sit in the request path.

### Returning in the later weeks of Phase 2

- **LangGraph** for stateful agentic flows. The AI interview surface comes back here.
- **ElevenLabs (or equivalent voice provider)** for voice synthesis. Voice-in via Web Speech API on the frontend.
- **Multi-turn conversational state** — managing context windows, summarization, branch-on-prior-answer logic.
- **Kubernetes** as the deployment target for production-grade AI workloads with autoscaling. ECS skills carry over conceptually, but K8s is the destination.

### Returning across all of Phase 2 as cross-cutting concerns

- **AI cost monitoring** — every LLM call is a billable event. Phase 2 emphasizes this constantly because cost discipline is a senior skill.
- **AI evaluation and regression testing** — how do you write a test for "the model gave a reasonable answer"? Phase 2 introduces evaluation frameworks and golden-set patterns.
- **AI safety surface** — prompt injection, refusal handling, output validation. Real concerns when an LLM is in a user-facing path.

## The Architectural Seams That Phase 2 Will Reopen

The PEP substrate has specific seams where the AI machinery will re-enter. The cohort should know where they are:

| Seam | Where it lives in PEP | What Phase 2 does to it |
|---|---|---|
| Question type discriminator | `question-management` service | Adds `free_response` and `interview_prompt` types. |
| Submission grading | `test-management` scoring engine | Adds a branch: deterministic scoring for MC/code-completion, LLM-call for free-response. |
| Submission latency budget | Synchronous response from scoring engine | Free-response goes async — Lambda queue, candidate sees "pending" then "graded". |
| Results page | Shows MC / code-completion results | Adds LLM-generated explanations per free-response answer. |
| Trainer dashboard | Shows scores | Adds rubric-override UI; trainer reviews LLM scoring, can adjust. |
| User-service | JWT auth | Largely unchanged — auth is foundational and stays. |
| API gateway | Path routing | Largely unchanged — gateway pattern stays. |

The cohort built the foundation. Phase 2 adds layers on top. Almost nothing of what they built gets discarded; it gets *extended*.

## How To Prepare Between PEP And Phase 2

If there's a gap between the cohorts (typical), candidates can prepare:

- **Read the Anthropic Claude API docs.** Specifically the messages API and prompt caching. Even a 30-minute skim gives a head start.
- **Build a one-prompt toy.** A script that calls Claude with a fixed prompt and parses output. The smallest possible LLM integration; teaches the basics of provider clients, error handling, and parsing.
- **Read about RAG patterns.** Vector embeddings, similarity search, the basic shape of retrieval-augmented generation.
- **Don't try to learn LangChain in advance.** LangChain's surface is large and Phase 2 teaches it in the right order. Pre-learning often produces wrong mental models.
- **Don't try to learn LangGraph in advance.** Same reason, doubled.

The pre-learning that *helps* is foundational (provider clients, prompt caching, basic output parsing). The pre-learning that *hurts* is framework-specific (LangChain idioms learned out of order).

## What The Capstone Demonstrates About Phase 2 Readiness

The capstone is partly an assessment of "is this candidate ready for Phase 2?" The readiness signals:

- **Full-stack ownership.** Candidate can walk through frontend, backend, and database changes coherently. Phase 2's AI features require this baseline.
- **Architectural reasoning.** Candidate names alternatives, justifies choices, owns trade-offs (see the architectural-walkthrough topic). Phase 2 will demand this for AI design choices.
- **Debt awareness.** Candidate sees where shortcuts were taken (see the tech-debt topic). Phase 2's AI surface generates debt fast; awareness is survival.
- **AI tooling literacy.** Candidate has used Claude Code across PEP (the Unit 0 thread), owns the output, defends it. Phase 2 ramps this up.

A candidate who lands the capstone cleanly is *ready* for Phase 2. A candidate who lands "pass with notes" has specific gaps named for them to close before Phase 2 starts. A candidate who's "not yet" needs the gaps closed before they bridge.

## The Continuity Message

The PEP cohort sometimes feels like "the appetizer" before the AI work. That's the wrong frame. The right frame:

> "PEP built the application. Phase 2 adds the intelligence layer to that application. Without the application, there'd be nothing for the intelligence to attach to."

The four slices the cohort built — question authoring, quiz taking, results, trainer dashboard — are the surface area Phase 2's AI features modify. The discipline learned during PEP — server-anchored truth, idempotency, transactional integrity, role-gated UI, full-slice integration — is exactly the discipline Phase 2 will demand at a higher scale.

What's coming is harder. The foundation is what makes it learnable.

## Anti-Patterns In Framing The Bridge

- **"Now the real work starts."** Disrespects the foundation. The full-stack work was real.
- **"PEP was just preamble."** Same problem. The application *is* the application.
- **"AI is the hard part."** AI is *a* hard part. Distributed transactions, race-free state, and role-gated UI are also hard. Don't rank.
- **"Phase 2 is greenfield."** It's not. The PEP substrate is what Phase 2 extends. The candidate's PEP code is the foundation they'll iterate on for the next 10 weeks.
- **"You can skip PEP-level discipline once we have AI."** AI generates surface area; that surface area still needs to be transactionally correct, idempotent, role-gated, and observable. PEP-level discipline becomes *more* important with AI in the mix, not less.

## Connecting Back

Day 1 named the strip. The four weeks built the substrate. Day 20 closes the loop by re-naming the strip and pointing at what comes next. The cohort that walks out of capstone with a clear mental model of "here's what I built, here's what got stripped, here's what's coming back" is positioned to start Phase 2 with the right expectations.

## Key Takeaways

- The PEP substrate is a *stripped* version of `rev-eval-ai`: AI services (ai-quiz-service, ai-interview-service), Lambdas, libraries (LangChain, LangGraph, Bedrock, ElevenLabs, vector stores), and K8s were removed.
- The strip was deliberate: surface area, AI fluency as its own topic, full-stack fundamentals first, deployment complexity as a learning gate.
- Phase 2 reintroduces the stripped surface in stages: LLM-graded answers and prompt caching first, then LangChain and RAG, then LangGraph and voice agents, with K8s and cost discipline as cross-cutting concerns.
- The architectural seams where AI will re-enter are predictable: question type discriminator, scoring engine, results page, trainer dashboard.
- Pre-learn foundationals (API clients, prompt caching, RAG concept). Don't pre-learn LangChain or LangGraph — order matters.
- PEP is not preamble. PEP is the foundation Phase 2 builds on. The four slices the cohort shipped are the canvas the AI layer attaches to.

---
*Prerequisites: day-01-claude-code-cli-fundamentals-and-config, day-20-technical-debt-identification-in-own-work.*
