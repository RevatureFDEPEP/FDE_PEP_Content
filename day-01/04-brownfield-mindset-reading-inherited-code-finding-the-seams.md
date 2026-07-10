# Brownfield Mindset — Reading Inherited Code, Finding the Seams

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

## Overview

PEP is a brownfield course. You are not building from a blank repo — you are inheriting a working (or mostly-working) system that someone else designed, someone else wrote, and someone else has already deployed and operated. This is deliberate, because that is what client work at Revature looks like. The skill to develop is not "write code from scratch" — you already have some version of that — but "land changes safely in code you did not write." This topic is the *mindset* piece of that skill: how to approach inherited code so you do not break it, embarrass yourself, or waste the trust of the team that brought you in.

## The Two Modes: Greenfield vs Brownfield

| | Greenfield | Brownfield |
|---|---|---|
| Starting point | Empty repo | Existing system |
| Primary skill | Design and synthesis | Reading and inference |
| Speed comes from | Decisions made and never revisited | Understanding before changing |
| Biggest risk | Over-engineering | Breaking something invisible to you |
| Bias should be toward | Building | Reading |
| Refactor instinct | Sometimes appropriate | Almost always wrong this early |

Most engineering training is greenfield-shaped — bootcamps, tutorials, and side projects all start from nothing. So when you arrive in a brownfield codebase, your default instincts are often miscalibrated. You see something that looks "wrong" and want to fix it. You see code you don't understand and want to rewrite it in a way you do. You see structure that seems suboptimal and want to refactor it. All of these instincts, applied early, are *destructive*.

## The Core Principle: Read Before You Write

The single most important brownfield discipline is to **read more than you write, especially early**. A rough ratio for your first week in any inherited codebase: read 5–10x more than you write.

This is not about being slow. It is about being *correct*. Inherited code encodes:

- **Decisions that look strange but have reasons.** A timeout set to 47 seconds. A weird-looking workaround. A piece of duplication that resists deduplication. These are usually scar tissue from real bugs — and removing them silently re-introduces those bugs.
- **Conventions you haven't seen yet.** The codebase has a way of naming things, structuring files, handling errors, and writing tests. Following the existing convention — even when you'd personally do it differently — is what makes your change look like it belongs.
- **Operational constraints you can't see.** A function that retries three times exists because in production it fails twice. Removing the retry "to clean up" breaks the system in a way no test catches.

Reading first is not deferring action — it is *enabling* action that won't be reverted in code review or, worse, hot-fixed on a Friday night.

## Finding the Seams

A *seam* is a place in inherited code where you can make a change without rippling effects everywhere else. Seams are where you want to land changes. The opposite of a seam is a "load-bearing wall" — code that is touched by many things, where any change cascades.

How to recognize seams:

- **Well-defined interfaces.** A function with a clear single responsibility, called in a small number of places, with a stable input/output shape. Easy to modify behind, easy to extend at the call site.
- **Configuration extension points.** Adding a new entry to a registry/dictionary/config file rather than altering the dispatch logic.
- **New code in a new file.** Adding a new module that imports existing things but is not yet imported by anything (until you wire it up at one specific point) is almost always safer than weaving changes into an existing module.
- **Tests that exercise the surface you are about to change.** A change to code that has tests is dramatically safer than a change to code that does not — both because you'll know when you've broken something, and because the test itself documents the intended contract.

How to recognize load-bearing walls:

- Functions imported by many other modules.
- Classes inherited from in many places.
- Configuration values consumed by half the codebase.
- Any module whose name shows up frequently in `git log`.

Your first changes in a brownfield codebase should land at seams, not on load-bearing walls. Save the walls for later, after you've built credibility (and understanding).

## The Reading Strategy

A useful order for getting oriented in inherited code:

1. **Top-level documentation.** README, `docs/` folder, ADRs (Architecture Decision Records) if they exist. This is the intended story of the system.
2. **The entry points.** Where does control start? For a service, that's the main HTTP handler / route registration. For a frontend, it's the app entry. Trace control flow from there.
3. **The data shapes.** Models / schemas / TypeScript types. Data structure tells you more about a system's intent than any other single artefact.
4. **The dependency manifests.** `pyproject.toml`, `package.json`. What does this system rely on? That tells you which patterns to expect (ORM-style if SQLAlchemy is there, framework-driven if FastAPI is there, etc.).
5. **The tests.** What is the system claiming to do correctly? Tests are executable specifications.
6. **The deployment / CI config.** How does this thing get to production? `docker-compose.yml`, GitHub Actions workflows, Dockerfiles. This tells you the operational contract.

Notice: code (`.py`/`.ts` files) shows up *third or later* in this list, not first. Reading code line-by-line without first understanding the structure is the slowest possible way to get oriented.

## When You Spot Something That Looks Wrong

You will spot things that look wrong. Some genuinely will be — there are seeded bugs in this substrate (the CI bugs you debug Days 3–4 are *intentional* defects). But many things that look wrong will simply be unfamiliar.

A useful internal script when you spot something:

1. **Note it down.** Keep a "questions" list as you read.
2. **Resist fixing it immediately.** Especially this week.
3. **Ask, "what would have to be true for this to be correct?"** Often, by the time you've answered that, you understand why it is the way it is.
4. **If it still looks wrong after that exercise**, raise it — in cohort discussion, with your trainer, or as a PR comment if you find it during review. Don't silently rewrite.

## Example / Worked Scenario

You are reading the inherited `user-service` and find this:

```python
# In auth.py
def verify_token(token: str) -> User | None:
    try:
        payload = jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        return None
    except jwt.InvalidTokenError:
        return None
    except Exception:  # noqa: BLE001
        logger.warning("Unexpected token error", exc_info=True)
        return None
    return User.from_payload(payload)
```

Your greenfield instinct: "That bare `except Exception` is bad practice; I should remove it."

Your brownfield reaction:

1. *Note it down* — "broad exception in verify_token."
2. *Look at `git log` and `git blame` on that line.* You find a commit message: "fix: don't 500 when jwt library raises non-standard exception in edge case (issue #847)."
3. *Now you understand.* The broad except was added because the JWT library, in a specific edge case, raises an exception that does not inherit from `InvalidTokenError`, and an uncaught exception there crashed the auth flow in production. The `noqa: BLE001` is acknowledging the linter complaint deliberately.
4. *Decision:* leave it alone. It is the way it is for a reason. If you really want to improve it, narrow the exception type *after* you understand the edge case it was catching — but that's a Week 3 task, not a Day 2 one.

This is the brownfield mindset working correctly. Five minutes of reading saved a regression.

## Common Pitfalls

- **Refactoring on sight.** The strongest brownfield anti-pattern. You see code you don't like, you rewrite it. The rewrite removes scar tissue, the bug returns, and you cannot reproduce it locally. Don't.
- **Trusting your sense of "weird" too early.** Until you've read enough of the codebase to know what its conventions are, "this looks weird" really means "I haven't seen this pattern in this codebase yet." Read more before acting.
- **Ignoring `git blame` and `git log`.** These are the project's memory. A two-minute look at the history of a confusing line will often answer the question "why is this here?" Get into the habit.
- **Rewriting tests instead of understanding what they assert.** If a test is in your way, the answer is almost never to delete it. Understand what behaviour it is locking in first.
- **Conflating "I don't understand this" with "this is wrong."** They are not the same. Most of the time, code you don't understand is correct code you haven't yet figured out.
- **Trying to read the entire codebase before doing anything.** The opposite failure mode — analysis paralysis. You don't need to understand everything; you need to understand enough to land your specific change safely. Read with a purpose.

## Key Takeaways

- Brownfield work is qualitatively different from greenfield: the primary skill is reading and inference, not design and synthesis.
- Read 5–10x more than you write in your first week in an inherited codebase. Reading first is not slowness — it is correctness.
- Seek "seams" (low-blast-radius extension points) for your changes; avoid "load-bearing walls" (code touched by many things) until you have credibility and understanding.
- A useful reading order: docs -> entry points -> data shapes -> dependencies -> tests -> deployment config. Code line-by-line comes later.
- When something looks wrong, note it down, ask "what would have to be true for this to be correct?", and check `git log`/`git blame` before changing it.
- "I don't understand this" is not the same as "this is wrong." Calibrate your sense of "weird" only after you've read enough to know what the codebase's normal looks like.

---
*Prerequisites: [03-microservices-architecture-and-service-boundaries.md](03-microservices-architecture-and-service-boundaries.md). Sets up [05-codebase-navigation-conventions.md](05-codebase-navigation-conventions.md) (the literal map of the substrate) and [06-tech-stack-comprehension-identifying-whats-in-use-legacy-or-at-risk.md](06-tech-stack-comprehension-identifying-whats-in-use-legacy-or-at-risk.md).*
