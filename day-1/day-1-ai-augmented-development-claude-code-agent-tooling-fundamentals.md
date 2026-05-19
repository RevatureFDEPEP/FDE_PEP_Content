# AI-Augmented Development: Claude Code Agent Tooling Fundamentals

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

> Note: This topic is part of the cross-cutting **AI Tooling Thread** running through the course. Every day of PEP includes an AI-tooling beat; today is orientation, and comprehension/debugging/drafting depth follow on Days 2–4.

## Overview

Claude Code is a first-class tool in this cohort. You will use it every day — to read code you have never seen before, to draft commits, to debug failing tests, to author content, and to navigate the inherited `rev-eval-ai` repository. But "AI tool" covers a wide range of things, and Claude Code specifically is not a chat window with autocomplete bolted on — it is an *agentic* tool that takes actions on your machine. Today's job is to install it, authenticate it, run a first task, and — most importantly — understand the trust boundary: what it can do, what it asks permission for, and what the limits of its judgement are.

## What "Agentic" Means

A traditional AI coding assistant (Copilot-style autocomplete, an editor-side chat panel) suggests text. You accept or reject; nothing happens until you do.

An *agentic* tool like Claude Code can:

- **Read** files in your project.
- **Run** shell commands (with your permission).
- **Edit** files (with your permission).
- **Search** the codebase (grep, glob).
- **Loop** — observe the result of one action, decide what to do next, act again, until the task is done or it needs your input.

This is qualitatively different. You are no longer reviewing text suggestions — you are reviewing *actions on your system*. That changes both the power of the tool and your responsibility when using it. Claude Code is built around explicit permission prompts for write/execute actions for exactly this reason.

## The Basic Invocation Loop

The mental model is simple:

1. **You give Claude a task** in natural language. Example: *"Run the test suite for the user-service and tell me which tests are failing."*
2. **Claude plans an approach** — what files to read, what commands to run.
3. **Claude executes**, asking permission for write/execute actions (read-only actions like file reads and searches generally proceed without prompting).
4. **Claude observes the result** of each action and decides the next step.
5. **Claude reports back** — either with the answer, a completed change, or a question for you.

You stay in the loop. Big-picture decisions are yours; mechanical execution and exploration are Claude's.

## Authentication

To use Claude Code in this cohort:

1. Install Claude Code via the install command for your OS (see the official docs — the canonical install method).
2. Run `claude` from the directory you want to work in.
3. On first run, you will be prompted to authenticate. Follow the OAuth flow — this links Claude Code to your Anthropic account.
4. Verify with a trivial prompt: *"What files are in this directory?"* Claude should list them.

Your authentication is per-user, not per-project. Once authenticated on a machine, Claude Code works from any directory.

## The Trust Boundary — What to Know on Day 1

This is the part to internalize before you ever use the tool on real work:

- **Read-only is safe.** Letting Claude read files, search the codebase, and list directories is low-risk. You should be liberal with this.
- **Execute and edit need scrutiny.** When Claude proposes running `rm -rf something` or rewriting a config file, *read the proposal*. Do not reflexively approve. The permission prompts are there to give you a checkpoint, not as a speed bump to dismiss.
- **It can be wrong.** Claude is an LLM — confident, articulate, and capable of generating plausible-looking but incorrect code or commands. Verify, especially for: destructive operations, security-sensitive changes, anything that touches credentials, anything that modifies CI configuration, anything in a `Dockerfile` or `docker-compose.yml`.
- **It does not know what it has not been told.** If a critical detail about your project lives only in someone's head or a Slack message, Claude cannot use it. Provide context.
- **It is not a substitute for understanding.** Using Claude to read code for you does not substitute for being able to read the code yourself when you need to. Use it as an accelerator, not a crutch — especially this week, when building your own mental model of the substrate is the actual learning objective.

## Effective Prompting — A Concrete Example

A weak prompt:

> "Fix the bug."

A useful prompt:

> "The `user-service` container is failing to start. I get a connection error to Postgres in the logs. Read the service's `database.py` and `docker-compose.yml`, identify the most likely cause, and explain what you find before changing anything."

Why the second is better:
- **It scopes the work** — specific service, specific files.
- **It surfaces the symptom** — connection error to Postgres in the logs.
- **It asks for diagnosis before action** — "explain what you find before changing anything." This keeps you in the loop on the *reasoning*, not just the result.
- **It does not prejudge the fix.** "Fix the bug" implies you know what bug. The better prompt invites Claude to investigate, which is what it is good at.

A useful workflow pattern for this cohort:

1. **Explore phase.** "Read the codebase and tell me where X lives." Read-only, low-risk.
2. **Diagnose phase.** "Here is the symptom. What is the most likely cause? Explain your reasoning." Still read-only.
3. **Plan phase.** "Propose the smallest change that would fix this." Claude proposes; you read and approve.
4. **Execute phase.** "Make that change." Now Claude edits with permission. You verify after.

## Example / Worked Scenario

You have just cloned `rev-eval-ai` and have no idea what it does. A good Day 1 Claude Code session:

```
You: I just cloned this repo. Read the top-level README.md and the
     docker-compose.yml, then give me a summary: what services exist,
     what they do, and how they talk to each other.

Claude: [reads files] Here's what I found...
        - 4 backend services: user, question-management, test-management,
          api-gateway
        - 1 frontend (Next.js)
        - Postgres, Mongo, MinIO as data stores
        - api-gateway is the public entry point...

You: Now read the user-service's pyproject.toml and tell me what
     framework it uses and what the main dependencies are.

Claude: [reads] FastAPI-based, using SQLAlchemy for Postgres, ...
```

Notice this is *all reads*. You are building your own mental model of the substrate, accelerated by Claude doing the file-opening for you. No edits, no permissions to grant, no risk. This is the right shape for Day 1.

## Common Pitfalls

- **Auto-approving every permission prompt.** The prompts exist for a reason. Read what is being proposed before approving — especially anything destructive, anything in `.env*`, anything touching CI files, and anything running with sudo or root.
- **Asking Claude to "just figure it out" on a vague task.** It will try, and it may succeed, but the result will reflect the vagueness of the input. Spend the 30 seconds to write a scoped prompt — you get vastly better output.
- **Using it as an answer machine when you should be learning.** This week's job is to build *your* mental model of the substrate. If you let Claude do all the reading, you will be lost when something goes wrong and you are away from the tool. Read alongside it, especially this week.
- **Forgetting that it cannot see what is not in the repo.** Slack threads, internal wikis, conversations with a trainer — none of those are visible to Claude unless you paste them in. Don't be surprised when it doesn't know about Revature-specific context.
- **Confusing Claude Code with a chat tool.** Chat tools generate text in a window. Claude Code takes actions in your filesystem and shell. The mental model and the responsibilities are different.

## Key Takeaways

- Claude Code is an *agentic* tool — it reads, runs, edits, and loops, not just suggests text — which makes it more powerful and makes your review responsibility larger.
- The invocation loop is: you give a task, Claude plans/executes (with permissions for write/execute actions), Claude reports back. You stay in the loop on direction; Claude does the mechanical exploration.
- Authentication is OAuth-based and per-user; once set up, it works from any project directory.
- Trust boundary: read-only operations are safe and you should be liberal with them; execute and edit operations need scrutiny — read each permission prompt before approving.
- Good prompts scope the work, surface the symptom, ask for diagnosis before action, and do not prejudge the fix. The explore -> diagnose -> plan -> execute workflow keeps you in control.

---
*Prerequisites: [day-1-fde-role-and-pep-positioning-in-the-three-phase-programme.md](day-1-fde-role-and-pep-positioning-in-the-three-phase-programme.md) (course framing).*
