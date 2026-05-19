# FDE Role and PEP Positioning in the Three-Phase Programme

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

## Overview

Before you touch a single line of code in the inherited `rev-eval-ai` repository, you need to understand what role you are stepping into and where this four-week cohort sits in Revature's larger talent-development pipeline. The "Front-End Developer" (FDE) title at Revature carries a broader scope than the same title elsewhere in the industry, and the Pre-Employment Programme (PEP) is deliberately shaped to expose you to that scope before you are placed on a client engagement. This topic frames the *why* behind everything else you will do this week — and explains why a course called "Front-End Developer" spends its first week debugging Docker Compose and CI pipelines instead of writing React components.

## The FDE Role at Revature

At Revature, "Front-End Developer" does not mean *only* front-end. It means a delivery-oriented engineer whose primary discipline is the UI tier, but who is expected to:

- **Own a vertical slice end-to-end.** When a feature lands on your plate, you are accountable for the React/Next.js component, the API call that feeds it, the contract with the backend service, and the operational story (does it work in staging? are the logs useful?). You are not handed a Figma file and a stub endpoint and told to "wire it up."
- **Work fluently across the stack to deliver.** You are not expected to be a database expert or a platform engineer, but you *are* expected to read a Python service, change a small thing safely, debug a Docker Compose stack-up, and reason about a CI pipeline. The bar is "competent collaborator across the stack," not "specialist in every tier."
- **Operate in inherited codebases.** Client work at Revature is overwhelmingly brownfield. You will join a team with an existing system, existing conventions, and existing technical debt. The FDE who can productively land changes in someone else's codebase on day one is worth far more than the FDE who can architect a greenfield application from scratch.
- **Take ownership of the operational surface.** When something breaks in your slice — local environment, CI, deployment — you investigate before escalating. You are not expected to fix every operational issue alone, but you are expected to be able to *describe* it accurately to whoever can.

## The Three-Phase Programme

Revature's FDE pipeline runs in three sequential phases. PEP is the middle phase:

```
Phase 0: Hiring & Selection
   |
   v
Phase 1: PEP (4 weeks)  <-- YOU ARE HERE
   |
   v
Phase 2: 10-Week Intensive
   |
   v
Phase 3: Client Placement
```

**Phase 1 — PEP (this cohort, 4 weeks).** Brownfield onboarding. You inherit a stripped-down version of an internal evaluation app, stabilize it, then build a vertical product slice on top of it. The goal is not to teach you React from scratch — it is to take an intermediate developer and make them effective in a real, inherited, polyglot, containerized codebase. By the end of PEP you should be able to land changes in a brownfield system without breaking it, and you should have a portfolio artefact (the capstone) that demonstrates that.

**Phase 2 — 10-Week Intensive.** Deeper specialization. AI-augmented features (the very features stripped out of your PEP substrate — the AI quiz service, the AI interview service, the LangChain/Bedrock layer, Kubernetes deployment) get reintroduced and built on. Phase 2 assumes Phase 1's outputs as its starting point.

**Phase 3 — Client Placement.** Billable engagement. You ship for an enterprise client. Phase 3's success rate depends heavily on Phase 1 and 2 having produced an engineer who can join an inherited team and contribute quickly.

## Why PEP Looks the Way It Does

PEP's four weeks are organized as four vertical slices:

- **Week 1: Inherit & Stabilize.** Operational surface — environment, Docker, CI. No new product features.
- **Week 2: Question Authoring.** First product slice.
- **Week 3: Quiz Taking.** Second product slice.
- **Week 4: Results, Dashboard, Capstone.** Third slice plus capstone presentation.

Week 1 deliberately front-loads the *unfun* parts — local environment setup, container orchestration, CI debugging — because these are historically the cohort's weakest areas, and because no amount of React skill matters if you cannot get the application running locally to develop against. Phase 2 assumes you can stand the stack up on day one; PEP's job is to make that assumption true.

## Example / Worked Scenario

Imagine you finish PEP and land on a client engagement six months from now. The client uses a Django backend, a Vue frontend, and Kubernetes — none of which appeared in your PEP curriculum. Has PEP failed you?

No — and here is why. On day one of that engagement, the tech lead points you at a repository, hands you a Confluence onboarding doc, and says "see if you can get this running locally and then look at ticket CLIENT-1247." The skills PEP builds are:

1. Cloning an unfamiliar repo and orienting yourself in it (Day 1 here).
2. Reading inherited code before changing it (Day 1 here).
3. Bringing up a containerized polyglot stack you did not write (Day 2 here).
4. Debugging a CI pipeline you did not author (Days 3–4 here).
5. Landing a small change safely in someone else's system (Week 2 onwards).

The specific stack (Python+Next.js+Docker vs. Django+Vue+K8s) is incidental. The *pattern* is what transfers.

## Common Pitfalls

- **Treating "Front-End Developer" as React-only.** Candidates who arrive expecting a four-week React bootcamp are surprised by Week 1's operational focus. Reset that expectation now — your job here is to become a delivery-capable engineer whose primary tier is the UI, not a component specialist.
- **Assuming PEP teaches the 10-week intensive's content.** The AI features (LangChain, Bedrock, ElevenLabs, the AI quiz/interview services) are deliberately *not* in PEP — they belong to Phase 2. If you see references to them in the original `rev-eval-ai` README, they have been stripped from your substrate.
- **Skipping the "why" of the programme structure.** Engineers who understand *why* Week 1 is operational tend to engage with it; engineers who don't, treat it as a chore to get through before "the real work" starts. The real work is Week 1.
- **Treating PEP as evaluation-only rather than learning.** PEP is observed, yes, but its primary purpose is to make you effective for Phase 2 and client work. Asking for help, surfacing confusion, and showing your reasoning are all rewarded behaviours, not weaknesses.

## Key Takeaways

- "Front-End Developer" at Revature means a delivery-oriented engineer whose primary discipline is the UI but who owns a vertical slice end-to-end and is fluent across the stack.
- PEP is Phase 1 of a three-phase pipeline (PEP -> 10-week intensive -> client placement); its job is to make you effective in inherited, containerized, polyglot codebases.
- Week 1 prioritizes the operational surface (environment, Docker, CI) because the cohort's weakest area is operations, and Phase 2 assumes operational competence as a starting point.
- The specific tech stack in PEP is incidental; the transferable skill is "land changes safely in an inherited system."

---
*Prerequisites: none — this is the orientation topic for the cohort.*
