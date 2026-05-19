# End-to-End UI Testing Patterns (Playwright Happy-Path Flow)

> *Day 15: Slice Integration + Week 3 Review — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
v2.3 of the curriculum closes the E2E gap. The smoke script in Topic 2 proves the *services* are alive; it doesn't prove a *user* can drive a real browser through the quiz-taking flow. That requires a browser, a DOM, JavaScript hydration, real `fetch` requests, real autosave timers, real form interactions. Playwright is the tool. Today we write *one* Playwright test — the quiz-taking happy path — and wire it into the CI job alongside the smoke script. It's the highest-confidence test we'll write in PEP, and also the slowest; that asymmetry shapes how we use it.

## Why Playwright, Not Cypress

Both work. Playwright is the curriculum's choice for three reasons:

- **Multi-browser by default.** Chromium, WebKit, Firefox out of the box. Cypress is Chromium-only until you pay.
- **Auto-wait baked in.** `page.click` waits for the element to be actionable. No `cy.wait(1000)` smell.
- **Better isolation between tests.** Each test gets its own browser context (cookies, local storage, etc.); harder to cross-contaminate.
- **First-class Docker support.** The `mcr.microsoft.com/playwright` image is reliable in CI; no display-server gymnastics.

Microsoft maintains it, the API has been stable since 2021, and the docs are excellent. Pick Playwright and move on.

## Setup

Install in the frontend repo (D14 already added Vitest; Playwright lives alongside):

```bash
cd frontend
pnpm add -D @playwright/test
pnpm exec playwright install --with-deps  # downloads browser binaries; do once per machine
```

Minimal `playwright.config.ts`:

```ts
// frontend/playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  timeout: 60_000,                       // generous per-test; slice tests touch many services
  expect: { timeout: 10_000 },           // per-assertion wait
  fullyParallel: false,                  // serialize: we share one Compose stack
  retries: process.env.CI ? 1 : 0,       // one retry in CI; failures locally should be investigated
  reporter: process.env.CI ? "github" : "list",
  use: {
    baseURL: process.env.E2E_BASE_URL || "http://localhost:3000",
    trace: "retain-on-failure",          // gold for debugging CI failures
    screenshot: "only-on-failure",
    video: "retain-on-failure",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  ],
});
```

Scripts in `package.json`:

```json
{
  "scripts": {
    "e2e": "playwright test",
    "e2e:headed": "playwright test --headed",
    "e2e:debug": "PWDEBUG=1 playwright test"
  }
}
```

`pnpm e2e:headed` runs with a real browser window — invaluable when a test fails confusingly.

## The Auth Storage State Pattern

Logging in inside every test wastes seconds and exercises the login flow over and over. Playwright's "storage state" pattern logs in *once*, saves cookies and local storage to a JSON file, then reuses it for every subsequent test.

```ts
// frontend/e2e/auth.setup.ts
import { test as setup, expect } from "@playwright/test";

const STORAGE = "e2e/.auth/candidate.json";

setup("authenticate as candidate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("smoke@example.com");
  await page.getByLabel("Password").fill("smoke-pass");
  await page.getByRole("button", { name: /sign in/i }).click();

  // Wait for the post-login landing (dashboard or tests list)
  await expect(page.getByRole("heading", { name: /available tests/i })).toBeVisible();

  // Persist auth context
  await page.context().storageState({ path: STORAGE });
});
```

Then in the config, declare the setup project and have other projects depend on it:

```ts
// playwright.config.ts (excerpt — replace the projects array)
projects: [
  { name: "setup", testMatch: /auth\.setup\.ts/ },
  {
    name: "chromium",
    use: { ...devices["Desktop Chrome"], storageState: "e2e/.auth/candidate.json" },
    dependencies: ["setup"],
  },
],
```

Every test in the `chromium` project starts already-logged-in. The login flow itself runs once per test session. Add `e2e/.auth/` to `.gitignore` — it contains a token.

## The Happy-Path Test

The slice has one happy path: log in (already done by setup), pick a test from the list, answer all the questions while watching the timer tick and the autosave indicator behave, submit, see the confirmation. Real test file:

```ts
// frontend/e2e/quiz-taking-happy-path.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Quiz taking — happy path", () => {
  test("a candidate can start, answer, autosave, and submit a quiz", async ({ page }) => {
    // Auth state already loaded by setup; start at the available-tests list.
    await page.goto("/tests");

    // Start the seeded smoke test
    await page.getByRole("link", { name: /smoke test 1/i }).click();
    await page.getByRole("button", { name: /start/i }).click();

    // Land on /take/{sessionId}
    await expect(page).toHaveURL(/\/take\/[\w-]+/);
    await expect(page.getByRole("heading", { name: /question 1/i })).toBeVisible();

    // Timer present and counting down
    const timer = page.getByRole("timer");
    await expect(timer).toBeVisible();
    const initial = await timer.textContent();
    await page.waitForTimeout(2_500);
    const later = await timer.textContent();
    expect(initial).not.toBe(later);

    // Answer all questions in the seeded test
    const questionCount = 3; // matches the smoke seed
    for (let i = 1; i <= questionCount; i++) {
      await expect(page.getByRole("heading", { name: new RegExp(`question ${i}`, "i") })).toBeVisible();

      // Pick the first option (single_select) or the first two (multi_select)
      const radios = page.getByRole("radio");
      const checkboxes = page.getByRole("checkbox");
      const radioCount = await radios.count();
      if (radioCount > 0) {
        await radios.first().check();
      } else {
        await checkboxes.first().check();
        if ((await checkboxes.count()) > 1) await checkboxes.nth(1).check();
      }

      // Wait for autosave to settle (D14 indicator: 'idle' → 'saving' → 'saved')
      await expect(page.getByTestId("autosave-status")).toHaveText(/saved/i, { timeout: 5_000 });

      // Next, unless this is the last question
      if (i < questionCount) {
        await page.getByRole("button", { name: /next/i }).click();
      }
    }

    // Submit
    await page.getByRole("button", { name: /submit/i }).click();

    // Confirmation dialog (D14 double-submit defense)
    await page.getByRole("button", { name: /yes, submit/i }).click();

    // Confirmation page / banner
    await expect(page.getByRole("heading", { name: /submitted/i })).toBeVisible({ timeout: 10_000 });
    await expect(page.getByText(/locked at/i)).toBeVisible();

    // Submit button is gone / disabled (idempotent submit guard from D14)
    await expect(page.getByRole("button", { name: /submit/i })).toHaveCount(0);
  });
});
```

Things this test does that matter:

1. **Uses role-based locators** (`getByRole`, `getByLabel`) rather than CSS selectors. They survive markup refactors and exercise accessibility at the same time.
2. **Asserts on the timer ticking** — proves the D14 server-anchored timer is live, not just rendered. The 2.5-second wait is the only sleep in the test; everything else uses Playwright's auto-wait.
3. **Asserts on the autosave indicator** — proves D14's `idle → saving → saved` state machine is wired through the UI.
4. **Walks the loop, not just question 1.** Catches navigation bugs the smoke script can't see (e.g., the index reducer from D13 misbehaving on the last question).
5. **Asserts on the post-submit absence** of the submit button — proves the lock-out UI from D14 actually engaged.

## Selectors: Why Role-Based, Not `data-testid` Or CSS

Three reasons:

- **`getByRole` matches what assistive tech sees.** A test that passes verifies the app is at least nominally accessible.
- **Role + accessible name is stable.** Refactor from `<button>` to `<a role="button">` and the test still passes. CSS classes change every time you touch Tailwind.
- **`data-testid` is a maintenance treadmill.** Every component needs one; teams forget; the test breaks for production-irrelevant reasons.

`data-testid` has its place — for state machines exposed in the UI (the `autosave-status` element above is one), where there's no semantic role for "internal state indicator." Use it sparingly.

## Running Against The Compose Stack

The test assumes `http://localhost:3000` is the Next.js frontend in Compose. The full flow before running:

```bash
docker compose down -v
docker compose up -d --build
# wait for healthchecks
docker compose ps  # all (healthy)

# Run smoke first (Topic 2) — fail fast
bash scripts/smoke-quiz-taking.sh

# Then Playwright
cd frontend && pnpm e2e
```

In CI, the same sequence in a workflow step. The smoke-first ordering is deliberate: if the stack isn't routing, Playwright will burn 60 seconds proving it before giving up. The smoke script proves it in 10.

## Trace Files: The Best CI Debugging Tool You're Not Using

`trace: "retain-on-failure"` in the config records a full timeline of every action, network request, console message, and DOM snapshot for failed tests. The artifact is a zip; view it with:

```bash
pnpm exec playwright show-trace test-results/.../trace.zip
```

This is what makes flaky CI failures debuggable. Instead of staring at "expected `submitted`, got nothing" and guessing, you see the request that failed, the DOM at the moment of failure, the network response, and the timeline leading up to it. Tell the cohort this exists; they will use it constantly.

## What This Test Does NOT Cover

Just like with smoke (Topic 2), it's important to be explicit about scope:

- **Edge cases.** No retries, no network failures, no double-clicks. That's component tests (D14) and Topic 6 integration tests.
- **Multiple users in parallel.** Single-user happy path only.
- **The "abandoned then resumed" flow.** Not in scope today; Day 17 may add it.
- **Accessibility audits.** Use `@axe-core/playwright` later if needed; not in Week 3.
- **Visual regression.** Playwright supports it; we don't use it in PEP.

One test, one happy path. It's the highest-value Playwright test we'll write and it's enough for the slice.

## Anti-Patterns

- **Logging in inside every test.** Wastes minutes per suite; exercises the same code paths repeatedly. Use the storage-state pattern.
- **`page.waitForTimeout` everywhere.** Auto-wait covers 95% of cases. The only fixed wait in the happy-path test above is the 2.5s to verify the timer is *moving* — that's a deliberate use, not a smell.
- **CSS selectors (`.btn-primary`, `#submit-button`).** Refactor-fragile, accessibility-blind. Use role-based locators.
- **One test that does everything.** "Log in, browse, author, take, submit, review" as one giant test. When it fails, you don't know which step broke. Keep tests focused; one happy path per concern.
- **Running Playwright against deployed environments.** It's a Compose-stack test. Hitting staging means competing with other developers and getting non-reproducible failures. Local Compose only (until Phase 2 figures out staging E2E).
- **Ignoring traces on CI failures.** Open the trace; the bug is in there. Re-running tests until they pass is debt accumulation.

## Key Takeaways
- Playwright is the curriculum's E2E tool — multi-browser, auto-wait, first-class Docker support.
- The auth storage-state pattern means the login flow runs once per session; every other test starts already logged in.
- One happy-path test today: log in (via setup) → pick test → answer all questions → watch autosave → submit → see confirmation.
- Role-based locators (`getByRole`, `getByLabel`) survive refactors and exercise accessibility; `data-testid` is for state-machine exposure only.
- Trace files (`retain-on-failure`) are the CI debugging tool — open them; don't re-run blindly.
- Always run smoke (Topic 2) before Playwright — fail fast.

---
*Prerequisites: day-13-client-components-for-stateful-interactivity, day-14-client-side-temporal-state-timers-and-autosave, day-14-submit-and-lock-ux-patterns, day-15-end-to-end-smoke-testing-patterns.*
