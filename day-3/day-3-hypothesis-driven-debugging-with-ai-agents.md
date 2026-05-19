# Hypothesis-Driven Debugging with AI Agents — Read Before You Change

> *Day 3: Diagnose the CI Pipeline — PEP 4-Week Curriculum*
> *Week 1: Inherit & Stabilize*
> *AI Tooling Thread*

## Overview
Claude Code is exceptionally good at generating root-cause hypotheses for CI failures — give it a workflow file and a failing log, and you'll get five plausible explanations in seconds. It is also exceptionally good at confidently generating wrong hypotheses dressed in convincing prose. Today's discipline, carried forward from Day 1, is **read before you change**: use Claude Code as a hypothesis generator, but validate every hypothesis against the actual source before you treat it as fact and before you propose any fix.

## The hypothesis-generator mode

For today's deliverable — five bugs in `ci-pipeline.yml`, each with a root-cause hypothesis, no fixes — Claude Code is well-suited to the *generation* half of the job. It can read your workflow file and your failing log in one prompt and produce candidate explanations faster than you can.

The trap is that it produces those candidates whether or not they're right. Some will be correct. Some will be plausible but wrong (e.g., it will diagnose a real-looking YAML issue that actually parses fine). A few will be confidently fabricated (the model "remembers" that `actions/cache@v3` deprecated some flag when it didn't).

Your job is to use the agent for what it's good at (volume of plausible hypotheses, fast recall of common patterns) and not for what it's bad at (ground truth about your specific repo). The structure: **generate broadly, then validate narrowly**.

## The four-step workflow for today

1. **Run the pipeline.** Get a real failure log.
2. **Prompt Claude Code for hypotheses.** Give it the workflow file and the failure log. Ask for ranked hypotheses, not fixes.
3. **Validate each hypothesis against the source.** For each one, open the relevant file or setting and confirm the claim. Reject hypotheses you can't substantiate.
4. **Write the deliverable from validated hypotheses only.** The final list goes to your trainer; it must be defensible.

The agent participates in step 2. Steps 3 and 4 are yours.

## Concrete prompts for hypothesis generation

These are prompt templates you can use in Claude Code today, adapted to whichever bug you're investigating.

### Prompt 1: Bulk hypothesis generation

> Read `.github/workflows/ci-pipeline.yml` in full. The pipeline has been seeded with five deliberate bugs that produce CI failures or silent misbehavior. Without fixing anything, generate ten candidate hypotheses for things that might be wrong, ranked by how likely each is to be a real bug. For each, cite the specific line range in the file. Do not propose fixes.

What this prompt does well:
- Asks for **more hypotheses than you need** (ten, not five). Over-generation is cheap; you'll filter.
- Asks for **line-range citations**. Lets you validate each claim against the actual file.
- Explicitly forbids fixes — keeps the agent in diagnosis mode.

What you do with the output: open each cited line range, confirm the claim is real, mark each as "confirmed bug," "no actual bug, looks fine," or "ambiguous, needs further checking."

### Prompt 2: Log-driven hypothesis

> Here is a failing log from the `test-services (user-service)` job [paste log]. The pipeline file is `.github/workflows/ci-pipeline.yml`. Read both. Generate three ranked hypotheses for why this job failed. For each, point to the specific line in the pipeline file and the specific line in the log that supports the hypothesis. Do not propose fixes.

What this prompt does well:
- Pairs **log evidence** with **YAML evidence**. The agent can't fabricate a connection if you require both citations.
- Limits to three hypotheses — keeps the output focused for a single failing job.

### Prompt 3: Validation challenge (the second prompt)

> [Earlier in the session you got a hypothesis from Claude.] Now act as a skeptical reviewer. Reread `.github/workflows/ci-pipeline.yml` and find the strongest argument that the previous hypothesis is *wrong*. What would I expect to see in the file or in the log if the hypothesis were correct, and is that evidence actually present?

What this prompt does well:
- Asks the agent to **argue against its own output**. This is one of the most effective ways to catch confident fabrications.
- Forces explicit "what would falsify this" reasoning — closer to a real diagnostic discipline.

You won't always get a useful adversarial response, but when you do, it's high signal.

## Validating a hypothesis against the source

The validation pass for each hypothesis follows a fixed checklist:

1. **Does the line range cited exist, and does it say what the agent claimed?** Open the file. Read the lines. Sometimes the agent hallucinates a line that isn't there.
2. **Is the claimed semantics correct?** If the agent says "this YAML indentation puts `env:` at job scope," does it actually? Re-derive the parse from indentation rules.
3. **Does the proposed root cause match the failure mode in the log?** A hypothesis that says "missing secret" should produce a log signature consistent with missing secrets (empty-string substitution, downstream auth failure). If the log doesn't match, the hypothesis is wrong even if the YAML claim is correct.
4. **Could you reproduce or test it cheaply?** "If I temporarily added `- run: echo $SECRET_NAME` would I see an empty value?" If the test is feasible, run it. If the test is expensive, weight the hypothesis lower.

The discipline: **no hypothesis enters your deliverable unless you've gone through this checklist.** It takes longer than copy-pasting the agent's output. That's the point.

## What Claude Code is reliably good at

Use the agent freely for these:

- **"What does this YAML stanza mean?"** Reading specifications and explaining them in plain English is squarely in the model's strength.
- **"What are the typical failure modes of `actions/cache@v4`?"** Pattern recall about well-documented tools.
- **"Where in this log is the actual error vs the noise?"** Triaging a long log is exhausting for humans and fast for the model.
- **"What's the difference between v3 and v4 of `actions/checkout`?"** Often correct; verify on the release notes for anything you'll commit.
- **"Translate this stack trace into a one-sentence hypothesis."** Compressing logs is a strength.

## What Claude Code is reliably bad at

Don't trust the agent for these without verification:

- **Specific values from your repo's settings.** The agent cannot see your configured secrets. It will guess names that "sound right" — `AWS_SECRET_ACCESS_KEY` vs `AWS_SECRET_KEY` — and be wrong half the time.
- **The current state of action releases.** "This was deprecated in v4.2" claims should be verified. The model's training cutoff doesn't include the last few months of upstream changes.
- **Exact line numbers after edits.** If you've edited the file mid-session, the agent's earlier line citations are stale. Always re-cite after edits.
- **Whether a YAML mistake actually affects runtime.** The agent will sometimes flag "this is wrong" for stylistic choices that don't affect behavior. Verify the semantic impact.
- **Confidence calibration.** The agent says "definitely" about both correct and incorrect claims. Treat confidence statements as zero signal.

## A worked example

You run the CI. Job `build-and-push` fails on the `Configure AWS credentials` step with `Error: Could not load credentials from any providers`.

You prompt Claude Code:

> Read `.github/workflows/ci-pipeline.yml`. The job `build-and-push` is failing at the `Configure AWS credentials` step with "Could not load credentials from any providers." Generate three ranked hypotheses, each with a line-range citation, and the supporting evidence in the log. Do not propose fixes.

The agent returns:

> **Hypothesis 1 (most likely):** Lines 47–50 reference `${{ secrets.AWS_SECRET_KEY }}`. The canonical secret name in most AWS setups is `AWS_SECRET_ACCESS_KEY`. If the repository's configured secret is named `AWS_SECRET_ACCESS_KEY` and the YAML asks for `AWS_SECRET_KEY`, the reference resolves to empty string, producing the "Could not load credentials" failure.
>
> **Hypothesis 2:** Lines 47–50 use `aws-actions/configure-aws-credentials@v4`. Older v1/v2 of that action accepted slightly different input names. If the action was bumped without updating the inputs, the configuration would silently fail. (Check the action's input names against the v4 README.)
>
> **Hypothesis 3:** The job lacks `permissions: id-token: write`. For OIDC-based AWS auth (no static keys), this permission is required. If the workflow was migrated from static keys to OIDC and the permissions block was missed, configure-aws-credentials cannot mint a token.

Now you validate:

1. **Open lines 47–50.** Confirmed: the YAML references `secrets.AWS_SECRET_KEY`. Hypothesis 1 is structurally consistent.
2. **Check repo settings** (Settings → Secrets and variables → Actions). Find the configured secret name. If it's `AWS_SECRET_ACCESS_KEY`, hypothesis 1 is **validated**. If it's `AWS_SECRET_KEY`, hypothesis 1 is **falsified**.
3. **Check the action version.** Lines 47–50 use `@v4`. Open the v4 README. Confirm the input is `aws-secret-access-key`. The YAML passes `aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}`. Input name is right. Hypothesis 2 (input rename) is **falsified**.
4. **Check whether the workflow uses OIDC.** If `aws-access-key-id` and `aws-secret-access-key` are both passed (as static keys), it's not OIDC. Hypothesis 3 is **not applicable**.

Verdict: hypothesis 1 is the real bug, validated against repo settings. Write it up.

The agent did the heavy lifting of *generating* three candidates. You did the lifting of *validating* them. Neither half works alone.

## Common Pitfalls

- **Treating agent confidence as accuracy.** The agent says "definitely" about both correct and incorrect claims. Verify every hypothesis against the source.
- **Skipping the validation step because "it sounded right."** A plausible-sounding hypothesis is exactly the failure mode of unverified agent output.
- **Asking for fixes in the same prompt as hypotheses.** The agent will helpfully fix the wrong thing. Keep diagnosis and fix in separate prompts (and Day 4 is for fixes anyway).
- **Letting the agent's first answer anchor your investigation.** Ask for more hypotheses than you need, so you have alternatives to compare.
- **Forgetting to re-read after the agent has cited a line range.** Always open the file and read the lines yourself.

## Key Takeaways

- Claude Code is a hypothesis generator, not a fix generator. Use it to expand your candidate list; validate each candidate yourself.
- Ask for ranked hypotheses with explicit citations to file lines and log lines. Citations make the validation step tractable.
- Use a second adversarial prompt ("argue against your own hypothesis") to flush out confident fabrications.
- The agent is reliable for explaining well-documented patterns; unreliable for specific values from your repo (secrets, configured settings) and for exact action release dates.
- For today's deliverable, every bug on your list must be validated against the actual file or repo settings before it goes on the page. The agent's output is input to your investigation, not output from it.

---
*Prerequisites: `day-1-ai-augmented-development-claude-code-agent-tooling-fundamentals.md`, `day-2-ai-assisted-brownfield-comprehension.md`, `day-3-workflow-log-triage-to-localize-failures.md`.*
