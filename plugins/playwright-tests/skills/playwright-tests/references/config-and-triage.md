# Config, timeouts, and flake triage

## A baseline `playwright.config.ts`

Every line here exists for a reason; the comments say what it is, so the file stays reviewable when someone wants to change one.

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',

  // Fail the run if someone commits test.only. Cheap, catches a real mistake.
  forbidOnly: !!process.env.CI,

  // Retries detect flake; they do not fix it. Zero locally so you see races
  // while you still have the context to fix them.
  retries: process.env.CI ? 2 : 0,

  // Files run in parallel by default; this also makes tests inside a file parallel,
  // which surfaces hidden ordering dependencies instead of hiding them.
  fullyParallel: true,
  workers: process.env.CI ? 4 : undefined,   // undefined = half the cores locally

  // Generous enough for a real user journey, tight enough that a wedged test
  // fails in the same coffee break. Keep it above the sum of the waits inside a
  // test so failures surface as assertion errors, which name the condition.
  timeout: 60_000,

  expect: {
    // The single most consequential number in the file. 5s is the default and is
    // right for most apps: long enough for a normal render, short enough that a
    // broken locator fails fast. Raise it per assertion, not here.
    timeout: 5_000,
  },

  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',

    // A guardrail, not a schedule: a wedged action fails with a targeted error
    // instead of consuming the whole test budget. Never 0 — that means unlimited.
    actionTimeout: 15_000,
    navigationTimeout: 30_000,

    // Diagnostics only when something went wrong, so the happy path stays fast.
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      dependencies: ['setup'],
      use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
    },
    // Add firefox and webkit once the suite is stable in one browser; engine
    // differences in timing are a genuine source of findings.
  ],

  // Start the app if it isn't already running, and wait on a real readiness
  // signal rather than a sleep.
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000/health',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },

  reporter: process.env.CI
    ? [['html', { open: 'never' }], ['github'], ['json', { outputFile: 'results.json' }]]
    : [['html', { open: 'on-failure' }], ['list']],
});
```

## The timeout budget in practice

| Timeout | Default | Bounds | Set it |
|---|---|---|---|
| test | 30s | test body + fixtures + `beforeEach`/`afterEach` | `timeout` · `test.setTimeout(ms)` · `test.slow()` |
| expect | 5s | one retrying assertion | `expect.timeout` · `{ timeout }` per assertion · `expect.configure({ timeout })` |
| action | none | one `click`/`fill`/… , otherwise capped by the test budget | `use.actionTimeout` · per-call `{ timeout }` |
| navigation | none | `goto`, `waitForURL`, click-triggered navigation | `use.navigationTimeout` · per-call `{ timeout }` |
| `beforeAll`/`afterAll` | 30s | the hook, shared across the tests it serves | `test.setTimeout()` inside the hook |
| fixture | none | one fixture's setup and teardown | `{ timeout }` in the fixture option tuple |
| global | none | the whole run | `globalTimeout` |

**Which one blew?** The error message tells you, and they mean different things:

- `Test timeout of 30000ms exceeded` — the test ran out of overall budget. Often the *symptom* of an assertion that was going to fail anyway; look at the last trace step to find what it was actually waiting on.
- `expect(locator).toBeVisible() failed … Timed out 5000ms waiting` — a specific condition never came true. This is the good failure: it names the locator and shows the DOM.
- `locator.click: Timeout 15000ms exceeded … waiting for element to be visible, enabled and stable` — actionability never passed. The message says which check failed, which is usually the whole diagnosis.

**Scoped overrides, in order of preference:**

```ts
// 1. One assertion, with the reason stated
await expect(page.getByTestId('report')).toBeVisible({ timeout: 30_000 }); // PDF gen, p95 ~18s

// 2. One test
test('generates the annual export', async ({ page }) => {
  test.slow();          // triples the configured timeout — tracks the baseline
});

// 3. A file or describe block that is uniformly slow
test.describe('report generation', () => {
  test.slow();
});

// 4. A pre-configured expect for a known-slow area
const reportExpect = expect.configure({ timeout: 30_000 });
await reportExpect(page.getByTestId('report')).toBeVisible();
```

Reach for the global config only when the *whole app* changed, and say so in the commit message.

## Flake triage playbook

Work in this order. Each step is cheaper than the next and frequently ends the investigation.

**1. Reproduce it deliberately.**

```bash
npx playwright test tests/checkout.spec.ts --repeat-each=20 --workers=1
npx playwright test tests/checkout.spec.ts --repeat-each=20            # then, in parallel
```

If it only fails in parallel, it is shared state — a database row, a fixture user, a file on disk, a module-level variable. If it fails serially too, it is a race in the test or the app.

**2. Read the trace before changing anything.**

```bash
npx playwright test --trace on
npx playwright show-trace test-results/…/trace.zip
```

The trace shows each action, exactly what it was waiting for, and the DOM at that moment. Most flake diagnoses take two minutes here and would take an hour of guessing without it. `npx playwright test --ui` gives the same information live while you iterate.

**3. Classify what you found.** The fix follows from the class:

| What the trace shows | Cause | Fix |
|---|---|---|
| element existed but was covered | overlay, toast, sticky header | wait for it to clear, `addLocatorHandler`, or seed the dismissal |
| element found, then detached mid-action | React/Vue re-render replacing the node | assert the settled state first (`toHaveCount`, `toHaveText`) so the re-render is done |
| assertion checked before data arrived | sampled check instead of a retrying one | use an `expect(locator)` matcher |
| passes alone, fails in a suite | shared state or ordering dependency | isolate; create per-test data via the API |
| fails only in CI | slower machine, different viewport, missing env, timezone or locale | match CI locally with `--workers=4`, set explicit `timezoneId`/`locale` in config |
| fails only on retry #1 | leftover state from the first attempt | make teardown unconditional; prefer fresh data per attempt |
| intermittent by ~1 frame | animation or focus race | rely on actionability's stable check; stop forcing |

**4. Fix the cause, then prove it.** Re-run with `--repeat-each=20` and confirm the suite got *faster* as well as greener. A fix that adds wall-clock time is usually a sleep wearing a costume.

**5. If you must quarantine, make it visible and dated.**

```ts
test.fixme('checkout applies the promo code', async ({ page }) => {
  // Flaky since 2026-08-10, tracked in ENG-4821. Race between the cart poll and
  // the promo write; needs a fix in the app, not the test.
});
```

`test.fixme` skips and marks the intent; `test.fail()` asserts the test *does* fail, so it turns red again the moment the bug is fixed and someone forgets to re-enable it. A silently deleted or `test.skip`-ed test is how coverage quietly evaporates.

## CI notes

```bash
npx playwright install --with-deps chromium     # only the browsers you run
npx playwright test --shard=1/4                 # split across machines
```

- Run on Linux — it is the cheapest runner and the one the browser images are built for.
- Cache the browser download keyed on the Playwright version, not on the lockfile hash alone.
- Upload `playwright-report/` and `test-results/` as artifacts on failure. A CI failure without a trace costs someone a reproduction cycle.
- Watch the `flaky` count in the HTML report over time. A rising number is the signal that the suite is drifting back toward sleeps and retries, and it is much easier to fix ten flaky tests than a hundred.
