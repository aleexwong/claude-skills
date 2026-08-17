# Config and flake triage

## Baseline config

Every line has a reason, and the comments say what it is, so the file stays reviewable.

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  forbidOnly: !!process.env.CI,               // catches a committed test.only
  retries: process.env.CI ? 2 : 0,            // absorbs infra noise; zero locally so you see races
  fullyParallel: true,                        // surfaces ordering dependencies instead of hiding them
  workers: process.env.CI ? 4 : undefined,

  // Generous enough for a real journey, tight enough to fail in one coffee break.
  // Keep it above the sum of the waits inside a test so failures surface as
  // assertion errors, which name the condition that never came true.
  timeout: 60_000,

  // The most consequential number in the file. 5s is the default and is right for
  // most apps. Raise it per assertion, never here.
  expect: { timeout: 5_000 },

  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    actionTimeout: 15_000,                    // guardrail: a wedged action fails targeted
    navigationTimeout: 30_000,
    trace: 'on-first-retry',                  // diagnostics only when something went wrong
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    { name: 'chromium', dependencies: ['setup'],
      use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' } },
  ],

  // Wait on a real readiness signal, not a sleep.
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000/health',
    reuseExistingServer: !process.env.CI,
  },
});
```

## Which timeout blew?

The message tells you, and they mean different things:

- `Test timeout of 30000ms exceeded` — out of overall budget. Usually the *symptom*; check the last trace step for what it was really waiting on.
- `expect(locator).toBeVisible() failed … Timed out 5000ms` — a condition never came true. The good failure: it names the locator and shows the DOM.
- `locator.click: Timeout 15000ms exceeded … waiting for element to be visible, enabled and stable` — actionability never passed, and the message says which check failed. That's usually the whole diagnosis.

Scoped overrides, in order of preference: a single assertion with the reason in a comment → `test.slow()` on the test → `test.slow()` on the describe block → `expect.configure({ timeout })` for a known-slow area. The global config comes last, and only when the whole app changed.

## Triage playbook

**1. Reproduce deliberately.**

```bash
npx playwright test tests/checkout.spec.ts --repeat-each=20 --workers=1
npx playwright test tests/checkout.spec.ts --repeat-each=20          # then in parallel
```

Fails only in parallel → shared state. Fails serially too → a race in the test or the app.

**2. Read the trace before changing anything.** `--trace on`, then `npx playwright show-trace`. It shows each action, what it was waiting for, and the DOM at that moment; most diagnoses take two minutes here and an hour without. `--ui` gives the same live.

**3. Classify, then fix.**

| Trace shows | Cause | Fix |
|---|---|---|
| element existed but was covered | overlay, toast, sticky header | wait for it to clear, or seed it away |
| element detached mid-action | framework re-render | assert the settled state first (`toHaveCount`/`toHaveText`) |
| assertion ran before data arrived | sampled check | use an `expect(locator)` matcher |
| passes alone, fails in a suite | shared state | per-test data via the API |
| fails only in CI | slower machine, viewport, timezone, locale | set these explicitly in config; run `--workers=4` locally |
| fails only on retry #1 | leftover state from attempt 0 | unconditional teardown, fresh data per attempt |
| off by ~one frame | animation or focus race | rely on actionability's stable check; stop forcing |

**4. Prove the fix.** Re-run with `--repeat-each=20` and confirm the suite got *faster* as well as greener. A fix that adds wall-clock time is usually a sleep in disguise.

**5. Quarantine visibly, if you must.** `test.fixme` with a date and a ticket. `test.fail()` is better where it applies — it asserts the test *does* fail, so it turns red again the moment someone fixes the bug and forgets to re-enable. A silently deleted test is how coverage evaporates.

## CI

```bash
npx playwright install --with-deps chromium    # only the browsers you run
npx playwright test --shard=1/4
```

Run on Linux. Cache the browser download keyed on the Playwright version. Upload `playwright-report/` and `test-results/` on failure — a CI failure without a trace costs someone a full reproduction cycle. Watch the `flaky` count over time: a rising number means the suite is drifting back toward sleeps and retries, and ten flaky tests are far easier to fix than a hundred.
