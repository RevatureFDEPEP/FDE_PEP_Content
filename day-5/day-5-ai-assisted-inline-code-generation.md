# AI-Assisted Inline Code Generation — Compose Scripts, CLI Invocations, Infra Glue

> *Day 5: Advanced Container Builds & Multi-Service Integration — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*
> *Unit 0: AI Tooling Thread*

## Overview
Through Days 1-4, Claude Code was your **comprehension and debugging partner**. You asked it to explain inherited code, to triage CI logs, to draft PR descriptions and ADRs. You didn't yet ask it to *produce non-trivial new code* you committed.

Today is the first day you do. The use case is deliberately bounded: **infrastructure glue.** Compose snippets, shell scripts, Makefile targets, CLI invocations, environment setup. This is fertile ground for AI generation because:

- The patterns are well-known (Compose, bash, Make have decades of public examples).
- The verification loop is fast (run the script, see if the stack comes up).
- The failure mode is loud (it works or it doesn't).
- The blast radius is local (you're not committing application logic that ships to users).

Day 4's discipline carries forward unchanged: **draft with the agent, own the final content.** Tomorrow you'll be asked in review *why* a script is shaped the way it is. "The agent wrote it" is not an answer.

## What "Glue" Means in Practice
Today's smoke test (login → dashboard) and Week 2's CI both need helper code that isn't quite production application code:

- A bash script that brings the stack up, waits for healthchecks, and runs the smoke test curl commands.
- A Makefile target that builds and pushes all four services with appropriate tags.
- A Compose override file (`compose.test.yaml`) that runs the test-image variants instead of production.
- A `docker buildx` invocation with the right `--platform` and `--target` flags for the CI matrix.
- A `jq` or `yq` one-liner that extracts a value from a JSON/YAML response.

None of this is your *product*. All of it is necessary to ship the product. AI generation shines here.

## The Prompt Pattern for Glue Generation
A repeatable pattern for Claude Code:

```
I need <X — one sentence>.

Context I'm working in:
- <relevant fact 1, e.g., "Compose stack has services: user-service,
  question-management-service, test-management-service, api-gateway,
  frontend, postgres, mongo, minio, registry">
- <relevant fact 2, e.g., "the gateway is exposed on localhost:8080;
  internal calls use service-name DNS">
- <relevant fact 3, e.g., "I'm on Apple Silicon; CI is amd64">

Constraints:
- <e.g., "bash, POSIX-compatible where possible">
- <e.g., "no external tools beyond docker, curl, jq">
- <e.g., "must work both locally and in a GitHub Actions runner">

Produce <the artifact> with inline comments explaining each non-obvious
step. Don't invent context — if you need a fact I haven't given,
ask before assuming.
```

The "don't invent context" line is load-bearing. Without it, the agent will happily produce a script that assumes a `make seed` target you don't have, or a `wait-for-it.sh` script you don't ship. With it, the agent asks or marks the assumption explicitly.

## The Ownership Loop, Specific to Generated Code
Day 4 covered ownership for PR descriptions. For *code* you generated, the loop is stricter:

1. **Read the output line by line.** Not "skim for shape" — *read*.
2. **Run it.** A generated script that doesn't execute is worthless. Find out before you commit.
3. **Stress it.** Run it twice (idempotency). Run it with the stack already up. Run it with the stack already down. Find the silent failure modes.
4. **Strip it.** Generated code is often verbose. Cut anything that doesn't earn its line.
5. **Comment what you couldn't explain.** If a single line still confuses you after step 1, leave a comment marking it for the reviewer — or ask the agent to explain it, then write the comment yourself.

This loop applies to *every* AI-generated script you commit. No exceptions. The trainer will spot the un-stripped, un-stress-tested generated code from across the room.

## Example / Worked Scenario
**The task.** You need a bash script `scripts/smoke.sh` that brings the Compose stack up, waits until the gateway reports healthy, runs the login + dashboard smoke test from the previous topic, and tears down on exit. It needs to work both locally and (eventually) in CI.

**Step 1 — Prompt.** You write to Claude Code:

```
I need a bash script that runs an end-to-end smoke test of our Compose
stack.

Context:
- Compose stack has services: user-service, question-management-service,
  test-management-service, api-gateway, frontend, postgres, mongo, minio.
- The api-gateway is exposed on localhost:8080.
- A seeded trainer account exists with username "trainer@example.com"
  and password "seed-password".
- Healthchecks are defined on every service.
- Login endpoint: POST /api/auth/login (JSON body), returns Set-Cookie.
- Dashboard endpoint: GET /api/dashboard (requires session cookie).

Constraints:
- Bash, runs on macOS and Linux.
- Only requires docker, curl, and jq.
- Must wait for the gateway to be healthy before testing.
- Must tear down the stack on exit, including failures (use trap).
- Exit non-zero if either the login or dashboard request fails.
- Inline comments on any non-obvious step.

Don't invent context — if a fact isn't here, ask first.
```

**Step 2 — Generated draft.** The agent produces (paraphrased for length):

```bash
#!/usr/bin/env bash
set -euo pipefail

# Tear down on any exit (success, failure, signal)
trap 'docker compose down -v --remove-orphans >/dev/null 2>&1 || true' EXIT

echo "[smoke] Bringing up Compose stack..."
docker compose up -d

# Wait for api-gateway to be healthy. We poll `docker compose ps` and
# look for "(healthy)" in the api-gateway row. Timeout after 90s.
echo "[smoke] Waiting for api-gateway to report healthy..."
for i in {1..30}; do
  status=$(docker compose ps --format '{{.Name}} {{.Status}}' \
             | grep '^.*api-gateway' || true)
  if echo "$status" | grep -q '(healthy)'; then
    echo "[smoke] Gateway healthy."
    break
  fi
  if [ "$i" -eq 30 ]; then
    echo "[smoke] FAIL: gateway never became healthy" >&2
    docker compose logs api-gateway >&2
    exit 1
  fi
  sleep 3
done

# Cookie jar for the session cookie
COOKIE_JAR=$(mktemp)
trap 'rm -f "$COOKIE_JAR"' EXIT  # NOTE: this clobbers the earlier trap!

echo "[smoke] POST /api/auth/login..."
http_code=$(curl -sS -o /tmp/login.json -w '%{http_code}' \
  -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -c "$COOKIE_JAR" \
  -d '{"username":"trainer@example.com","password":"seed-password"}')

if [ "$http_code" != "200" ]; then
  echo "[smoke] FAIL: login returned $http_code" >&2
  cat /tmp/login.json >&2
  exit 1
fi

echo "[smoke] GET /api/dashboard..."
http_code=$(curl -sS -o /tmp/dash.json -w '%{http_code}' \
  -b "$COOKIE_JAR" \
  http://localhost:8080/api/dashboard)

if [ "$http_code" != "200" ]; then
  echo "[smoke] FAIL: dashboard returned $http_code" >&2
  cat /tmp/dash.json >&2
  exit 1
fi

echo "[smoke] PASS"
```

**Step 3 — Read line by line.** You catch:

1. **The double `trap` bug.** The second `trap 'rm -f "$COOKIE_JAR"' EXIT` *replaces* the first one — bash traps are single-slotted per signal. The teardown will not run. This is exactly the kind of bug the agent introduces while looking confident.
2. **`grep '^.*api-gateway'`** matches `api-gateway` anywhere in the line, which is correct, but the `^.*` is a red herring; could be just `grep api-gateway`. Style nit.
3. **`/tmp/login.json` and `/tmp/dash.json`** are predictable paths — parallel test runs would collide. Use `mktemp` for both.
4. **`set -euo pipefail` + `|| true` on the teardown** is the right move. Verified.

**Step 4 — Stress it.** You patch the trap bug (combine both cleanups into one), switch to `mktemp` for the response files, and run:

```bash
# Run 1: stack down to start
./scripts/smoke.sh
# [smoke] PASS

# Run 2: stack already up (idempotency check)
docker compose up -d
./scripts/smoke.sh
# [smoke] PASS  (compose up -d is idempotent; healthcheck still applies)

# Run 3: deliberately broken gateway
docker compose stop api-gateway
./scripts/smoke.sh
# [smoke] FAIL: gateway never became healthy
# (followed by gateway logs)
# Exit code 1, stack torn down. Good.
```

**Step 5 — Strip and comment.** Final committed version is the patched draft, ~50 lines, with two comments you added explaining the trap consolidation and the cookie-jar idiom.

**Step 6 — Defend it.** The Day 4 muscle. In review, when asked "why poll `docker compose ps` instead of `curl`-ing a health endpoint?", you have an answer: the gateway *has* a `/healthz` you could hit, but Compose's healthcheck already polls it on a 5-second interval, and `docker compose ps` reflects the result without duplicating the request. You verified by reading the compose file. You own it.

Total time: ~15 minutes. Without the agent: ~45. The 30 minutes you saved go into the Week 1 retrospective.

## Common Pitfalls
- **Committing the first draft.** The trap bug above is representative. Generated bash, in particular, is full of plausible-looking near-misses.
- **Not stress-testing.** "It worked once on my clean machine" is the lowest possible bar. Run it twice. Run it with state already present. Run it offline. Run it with a deliberately broken dependency.
- **Letting the agent invent flags.** `docker compose --wait` exists in newer versions but not all. `jq -e` for exit codes — check. The agent may use flags that work on its mental model of the tool, not your actual installed version.
- **Generated code that you can't explain.** If a reviewer asks "what does `${VAR:-default}` do?" and you don't know, you didn't read it carefully enough. Fix that *before* the review, not during.
- **Over-prompting for trivial things.** `docker compose up -d` does not need an AI. Don't burn the agent on what's faster to type.
- **Under-prompting for non-trivial things.** "Write me a smoke test" without context produces generic code that doesn't fit your stack. Always supply the facts the script needs.

## Key Takeaways
- Infra glue is the right early target for AI generation: well-patterned, fast to verify, low blast radius.
- The ownership loop has six steps: prompt with context, read line by line, run, stress, strip, defend.
- Generated code is a *draft*. The committed code is yours.
- Save your AI budget for the cases where it actually beats hand-typing.

---
*Prerequisites: `day-1-ai-augmented-development-claude-code-agent-tooling-fundamentals.md`, `day-4-defending-ai-generated-changes-in-peer-review.md`, `day-5-end-to-end-integration-testing-patterns-in-compose.md`.*
