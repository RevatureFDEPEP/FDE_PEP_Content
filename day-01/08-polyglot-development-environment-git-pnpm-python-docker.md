# Polyglot Development Environment (Git, pnpm, Python, Docker)

> *Day 1: Onboarding & Project Familiarization — PEP 4-Week Curriculum — Daily Topic Breakdown (v3.0)*
> *Week 1: Inherit & Stabilize*

## Overview

The `rev-eval-ai` PEP substrate is a deliberately polyglot system: Python services on the backend, a Next.js frontend in TypeScript, and Docker Compose tying everything together. You cannot productively read, run, or modify any of it without a working local toolchain for all four — and "works on my machine for Python" is not enough if pnpm is broken or Docker Desktop is not running. Today's job is verification: confirming that each tool is installed, on a compatible version, and actually functional, *before* tomorrow's Docker Compose stack-up exposes any gaps painfully.

## The Four Toolchains

### Git

Source control and the entry point to the codebase. PEP assumes you already know `git clone`, `git status`, `git add`, `git commit`, `git push`, `git pull`, and `git checkout -b`. Verify it is installed and that you can authenticate to GitHub (HTTPS with a Personal Access Token or SSH with a key — either works; pick one and configure it). The codebase will be cloned today.

Minimum version: Git 2.40+ is recommended; anything 2.30+ works.

### pnpm (Node.js package manager)

The frontend (Next.js 16) uses `pnpm` as its package manager, not `npm` or `yarn`. Using the wrong package manager in a pnpm-managed repo produces a lockfile mismatch, a different `node_modules` layout, and confusing "module not found" errors that look like code bugs. **Always use the package manager the repo expects.**

pnpm requires Node.js underneath it. The recommended path is:

1. Install a Node version manager (`fnm` or `nvm`).
2. Install Node.js LTS (currently Node 20.x or 22.x).
3. Enable Corepack (ships with Node 16.10+): `corepack enable`.
4. Corepack will activate the pnpm version pinned by the repo's `package.json` automatically.

### Python

The backend services are Python. Each service has a `pyproject.toml` declaring its dependencies. Python 3.11+ is the target. You do not need to install service dependencies onto your host machine for normal development — they run inside Docker containers — but you *do* need Python locally for scripts, for IDE support (type checking, navigation), and for the occasional ad-hoc command.

Recommended: install Python 3.11 or 3.12 via your OS package manager (`brew install python@3.12` on macOS, `apt install python3.12` on Debian/Ubuntu, or the official installer on Windows). Confirm `python --version` (or `python3 --version`) reports 3.11+.

### Docker

The whole stack runs under Docker Compose. On macOS and Windows, install **Docker Desktop** (which bundles Docker Engine, the Docker CLI, and Compose v2). On Linux, install Docker Engine and the Compose plugin via your distribution's package manager.

Critical Docker Desktop settings for this cohort:
- **Resources -> Memory:** at least 6 GB (8 GB if your laptop can spare it). The full stack runs ~7 containers; the default 2 GB will not be enough.
- **Resources -> CPUs:** at least 4 cores.
- **WSL 2 backend** on Windows (not Hyper-V).
- Docker Desktop must be **running** before you try Compose commands — this catches people daily.

## Verification Sequence

Run these commands in order. Each should produce output without errors. If any fails, stop and fix it before moving on.

```bash
# 1. Git
git --version
# Expected: git version 2.x.y

# 2. Node + pnpm (via Corepack)
node --version
# Expected: v20.x or v22.x
corepack enable
pnpm --version
# Expected: a version number (Corepack will fetch on first use)

# 3. Python
python3 --version
# Expected: Python 3.11.x or 3.12.x

# 4. Docker
docker --version
# Expected: Docker version 24.x or newer
docker compose version
# Expected: Docker Compose version v2.x
docker run --rm hello-world
# Expected: "Hello from Docker!" message
```

The `docker run --rm hello-world` line is the real test — it confirms the Docker daemon is actually running, not just that the CLI is installed.

## Example / Worked Scenario

You sit down on Day 1 and run through the verification sequence. Most commands succeed, but `pnpm --version` reports "command not found" even though you installed Node.js.

The diagnostic walk:

1. Is Corepack enabled? `corepack --version` — yes, it reports a version.
2. Is your shell's PATH picking up the Corepack-managed shims? `which pnpm` — empty.
3. Re-run `corepack enable` and check again. Still empty? Your Node installation may have been done via a method that does not include Corepack (some Homebrew taps, some old `nvm` versions).

The fix in practice: reinstall Node via the official installer or via `fnm install --lts`, then re-run `corepack enable`. This is the kind of small-but-blocking issue Day 1 exists to surface. If you hit it tomorrow during Compose work, you waste cohort time; if you hit it today during verification, you fix it and move on.

## Common Pitfalls

- **Running `npm install` in a pnpm-managed repo.** It will appear to work, but it creates an `npm`-style `node_modules` plus a `package-lock.json` that conflicts with the existing `pnpm-lock.yaml`. Symptoms: phantom missing modules, mysterious dependency resolution errors. Fix: delete `node_modules` and `package-lock.json`, then `pnpm install`.
- **Docker Desktop not running.** Compose commands fail with "Cannot connect to the Docker daemon." Always glance at the Docker Desktop tray icon before running anything Docker-related.
- **Under-allocated Docker memory.** The stack starts, then services crash a few minutes in with OOM kills that look like application bugs. Set memory to 6+ GB *before* you bring the stack up tomorrow.
- **Using the system Python on macOS.** macOS ships an old Python at `/usr/bin/python3` that you should not install packages into. Use Homebrew's `python@3.12` or a version manager like `pyenv`.
- **Mixing Python virtual environments and Docker.** You generally do *not* need a local Python venv for this course — the services run in containers. Creating one and trying to install dependencies into it is wasted effort and a common source of "but it works in my venv" confusion.

## Key Takeaways

- The PEP substrate is polyglot: Git, pnpm (Node), Python, and Docker must all be functional locally before Day 2.
- Always use the package manager the repo expects — `pnpm` here, not `npm` or `yarn`. Corepack makes this automatic.
- Docker Desktop needs 6+ GB of memory and must be running before any `docker` or `docker compose` command.
- Verification is a *sequence*: each tool has a "hello world"-style command that confirms it actually works, not just that it is installed.
- Surfacing toolchain issues on Day 1 is cheap; surfacing them on Day 2 during Compose stack-up costs the whole cohort time.

---
*Prerequisites: none — this is foundational setup. Connects forward to Day 2 (Docker Compose stack-up).*
