---
name: playwright-tests
description: Write, review, and repair Playwright tests so they wait on observable conditions instead of fixed durations. Use this skill whenever Playwright or Playwright Test is in play — authoring a new e2e or browser suite, reviewing existing specs, or debugging tests that are flaky, slow, or green locally and red in CI. Triggers include any mention of waitForTimeout, sleeps or hard waits, networkidle, waitForSelector, "Test timeout of 30000ms exceeded", strict mode violations, retries, trace viewer, or tuning timeouts in playwright.config. Reach for it even when the user only says "this e2e test is flaky", "make the browser test pass", or "add a test for this page" and never mentions best practices.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Playwright Tests That Wait on Conditions, Not Clocks

Almost every flaky Playwright test fails for the same reason: it asserts at a moment the *author guessed at* rather than a moment the *app announced*. Every technique here is one move applied over and over — **replace a duration with a condition**.

Use this when writing new specs, reviewing a suite, or chasing a test that passes on a laptop and fails on a CI runner.

## The rule

`page.waitForTimeout()` — and its cousins `sleep()`, `time.sleep()`, `Thread.sleep()`, a bare `await new Promise(r => setTimeout(r, 500))` — belongs in a debugging session, never in a committed test.

A fixed wait is wrong in both directions at once:

- **Too short** on a loaded CI runner, a cold cache, or a throttled container, and the test flakes. Flake is expensive out of proportion to its frequency, because it teaches a team to re-run red builds without reading them — which is exactly how a real regression ships.
- **Too long** everywhere else, and you pay the full duration on every run forever. Twenty `waitForTimeout(2000)` calls is forty seconds per file that no amount of parallelism gives back.

The deeper problem is that a sleep **asserts nothing**. Adding one converts a loud failure into a silent one: the feature can break outright and the test still passes, because the only proposition it ever verified was that time passes.

So the question is never "how long should I wait?" It is **"what is the observable thing I am waiting for?"** That thing is nearly always assertable, and asserting it is simultaneously faster (it returns the instant the condition holds) and stricter (it fails if the condition never holds).

## Playwright already waits — don't wrap it

Before every action, Playwright runs actionability checks and retries them until they pass or the timeout expires. Adding a wait before a click duplicates work the library already does.

| Action | Checks performed |
|---|---|
| `click`, `dblclick`, `tap`, `check`, `uncheck`, `setChecked` | visible, stable, enabled, receives events |
| `hover`, `dragTo` | visible, stable, receives events |
| `fill`, `clear` | visible, enabled, editable |
| `selectOption` | visible, enabled |
| `screenshot` | visible, stable |
| `focus`, `blur`, `press`, `pressSequentially`, `dispatchEvent`, `setInputFiles` | **none** |

Two consequences worth internalizing:

- **"Stable" means the bounding box is unchanged for two consecutive animation frames.** Sleeping for an animation to finish is redundant — `click()` already does this.
- **The last row is a trap.** People "fix" a flaky `click()` by switching to `dispatchEvent('click')` or `press('Enter')`, and it appears to work because those skip actionability entirely. The race is still there; it now fails somewhere less obvious. Same with `force: true`, which disables the non-essential checks — if force-clicking is the only way past an overlay, the overlay is covering the button for real users too. That is a bug report, not a test fix.

## Assert the condition, don't sample it

```ts
await expect(page.getByText('Welcome')).toBeVisible();   // retries until true or times out
expect(await page.getByText('Welcome').isVisible()).toBe(true);  // samples once, right now
```

The second line is the single most common flake source in review. `isVisible()`, `textContent()`, `innerText()`, `count()`, and `inputValue()` all resolve immediately against whatever the DOM happens to look like. Only the `expect(locator)` matchers retry.

The same trap wears a second costume:

```ts
if (await page.getByText('Cookie notice').isVisible()) {   // racy: false if it hasn't rendered yet
  await page.getByRole('button', { name: 'Accept' }).click();
}
```

A conditional built on an instantaneous check silently skips its own assertions on fast machines and then fails downstream when the banner appears over the element you wanted. Handle interrupters deterministically instead — seed the dismissal cookie in `storageState`, or register `page.addLocatorHandler()` so Playwright dismisses it whenever it shows up.

## What to reach for instead

Find the row that matches what you are actually waiting for.

| You want to wait for… | Use |
|---|---|
| an element to appear or disappear | `expect(loc).toBeVisible()` / `toBeHidden()` |
| a list to finish loading | `expect(rows).toHaveCount(n)` — `toHaveCount(0)` for empty |
| text to update after a fetch | `expect(loc).toHaveText()` / `toContainText()` |
| an input to be populated | `expect(loc).toHaveValue()` |
| a button to become usable | `expect(loc).toBeEnabled()` |
| a spinner to clear | `expect(spinner).toBeHidden()` — assert the *absence*, don't sleep past it |
| navigation | `expect(page).toHaveURL()` or `page.waitForURL()` |
| a network call to land | `page.waitForResponse()` — **started before the action** |
| a download, popup, or dialog | the matching `waitForEvent` / `page.on()`, **registered before the trigger** |
| a debounce, poll, countdown, or auto-dismissing toast | `page.clock` — fast-forward time instead of living through it |
| a backend that is eventually consistent | `expect.poll()` or `expect(async () => { … }).toPass()` |
| a screenshot to settle | `expect(page).toHaveScreenshot()` — retries until two consecutive shots match |
| an element in the DOM but not yet shown | `loc.waitFor({ state: 'attached' })` — a last resort, not a default |
| arbitrary JS state | `page.waitForFunction()` — last resort; prefer asserting the rendered result |
| "the page to be done loading" | assert the thing you care about. `waitForLoadState('networkidle')` is **discouraged by the Playwright docs**: *"Don't use this method for testing, rely on web assertions to assess readiness instead."* An app with polling or analytics beacons never goes idle |

Full code for each of these is in `references/waiting-recipes.md` — read it when the one-liner above isn't enough, particularly for network, events, and clock work.

### Register the listener before the action

Events and responses that fire during an action must be awaited on a promise created *first*, or you race the thing you're trying to observe:

```ts
const responsePromise = page.waitForResponse(
  r => r.url().includes('/api/cart') && r.request().method() === 'POST' && r.ok()
);
await page.getByRole('button', { name: 'Add to cart' }).click();
await responsePromise;
```

Use this when the outcome isn't visible in the UI, or when you need the response body. When the outcome *is* visible, assert the visible thing instead — `expect(cartBadge).toHaveText('1')` survives a refactor of the endpoint, and `waitForResponse` doesn't.

### Control time rather than spending it

A five-minute idle-logout has no business taking five minutes:

```ts
await page.clock.install();
await page.goto('/dashboard');
await page.clock.fastForward('05:00');
await expect(page.getByText('You have been logged out')).toBeVisible();
```

`setFixedTime` pins `Date.now()` for deterministic date rendering; `pauseAt` freezes at an instant; `runFor` ticks timers forward step by step. This is the correct answer for debounces, polling intervals, countdowns, cache expiry, and anything else where the test is otherwise waiting on the app's own timer.

## Budget the timeouts you do keep

Timeouts are a safety net for *stuck*, not a schedule for *slow*.

| Timeout | Default | Bounds | Change it via |
|---|---|---|---|
| test | 30s | the test body plus its fixtures and `beforeEach` | `timeout` in config · `test.setTimeout()` · `test.slow()` |
| expect | 5s | one auto-retrying assertion | `expect.timeout` in config · `{ timeout }` per assertion · `expect.configure()` |
| action | none | one `click`, `fill`, … (falls back to the test budget) | `use.actionTimeout` · per-call `{ timeout }` |
| navigation | none | `goto`, `waitForURL`, click-triggered nav | `use.navigationTimeout` · per-call `{ timeout }` |
| `beforeAll` / `afterAll` | 30s | the hook | `test.setTimeout()` inside the hook |
| fixture | none | one fixture | `{ timeout }` in the fixture's option tuple |
| global | none | the entire run | `globalTimeout` in config |

Three rules keep this honest:

**Raise at the narrowest scope that fixes the problem, and say why.** `await expect(report).toBeVisible({ timeout: 30_000 }) // PDF render, p95 ~18s` is reviewable: a reader can check the claim and delete it when the report gets faster. Bumping the global `expect.timeout` to 30s is the most damaging single line you can add to a config — every genuinely broken locator now costs 30 seconds instead of 5, a suite of 200 assertions turns a fast red into a twenty-minute red, and a page that got four times slower still passes silently.

**Keep the test timeout comfortably above the sum of the waits inside it.** When the test budget blows first you get "Test timeout of 30000ms exceeded", which points at the test. When the assertion budget blows first you get "expected visible, received hidden" with the locator, the actual DOM, and a trace step. The second message is the one that tells you what broke.

**Prefer `test.slow()` to a hardcoded number.** It triples whatever the config says, so it keeps tracking the project's baseline instead of freezing one machine's guess into the file.

Setting a modest global `actionTimeout` (say 10–15s) is a reasonable guardrail: it turns a wedged action into a targeted error instead of letting it eat the entire test budget. Never set any timeout to `0` — that means *unlimited*, and a hung test then burns a CI slot until the job itself is killed.

## Design tests that have nothing to wait for

Most waiting is a symptom of setup done the slow way.

- **One context per test.** Playwright already gives each test a fresh context — fresh cookies, storage, cache. Don't reintroduce shared state with module-level variables or tests that depend on running in order. `test.describe.configure({ mode: 'parallel' })` will find the ones that do.
- **Authenticate once, reuse `storageState`.** A setup project that logs in and saves storage state removes a UI login — and its four waits — from every test in the suite. Logging in through the form in `beforeEach` is both the slowest and the flakiest thing in most suites.
- **Seed through the API, assert through the UI.** Creating fixtures with the `request` fixture is instant and deterministic; creating them by clicking through five screens is neither, and a failure there fails the test you meant to run for a reason unrelated to it.
- **Mock what you don't own.** `page.route()` makes third-party responses instant and identical every run, and it eliminates the network waits those calls implied. It also lets you *choose* latency to test loading and error states — a `setTimeout` inside a route handler is fine, because that is shaping the fixture, not sleeping in the test.
- **Prefer user-facing locators.** `getByRole`, `getByLabel`, `getByText` describe what a user perceives, so they survive refactors and fail meaningfully. Reach for `getByTestId` when no accessible handle exists; reach for CSS/XPath chains such as `.btn > div:nth-child(2)` essentially never. Chain and filter (`.filter({ hasText: … })`, `.and()`, `.or()`) to resolve strict-mode violations instead of falling back to `.first()`, which silently makes the test pass against the wrong element.
- **Wrap phases in `test.step()`.** Steps make the trace readable, which is what turns a CI failure into a two-minute diagnosis.

Annotated config, CI settings, and retry policy are in `references/config-and-triage.md`.

## When a wait is genuinely unavoidable

Be honest that this case exists: a third-party widget with no DOM signal, a canvas or WebGL surface, a cross-origin iframe you can't reach into with `page.clock`. The escape hatch has conditions.

Prefer `toPass()` over a sleep even here, because it exits the moment the condition holds and a fixed wait never can:

```ts
// The payment iframe exposes no mounted event and repaints for ~300ms while its
// fonts load. Polling rather than sleeping so the common case stays fast.
await expect(async () => {
  await expect(frame.getByLabel('Card number')).toBeEnabled();
}).toPass({ timeout: 5_000, intervals: [100, 250, 500] });
```

If it truly must be a fixed duration, it needs a comment naming the missing signal, the tightest possible scope, and a real assertion immediately after so the wait is never the last word in the test. A sleep with no assertion behind it is not a test.

## Reviewing an existing suite

Start with a census, because the cheap signals are grep-able:

```bash
rg -n 'waitForTimeout|wait_for_timeout|networkidle|force: *true|timeout: *[0-9]{5,}' tests/
rg -n 'expect\(await .*\.(isVisible|isEnabled|isChecked|textContent|innerText|count)\(\)' tests/
rg -n 'if \(await .*\.isVisible\(\)\)|\.first\(\)|waitForSelector' tests/
```

Then work in this order, since each step removes work from the next:

1. **Delete the sleeps**, replacing each with the condition it was standing in for. Most map straight onto a row of the table above.
2. **Convert sampled checks to retrying assertions.** This usually removes the reason the sleeps were added.
3. **Pull global timeout inflation back to per-assertion overrides** with reasons attached.
4. **Move setup off the UI** — auth into `storageState`, data into API calls, third parties behind `page.route`.
5. **Re-run to confirm**, don't assume: `npx playwright test --repeat-each=20` on the suspect files, and check the run got faster as well as greener.

When a test is red because the application is genuinely slow or genuinely broken, say so plainly rather than tuning a timeout until it passes. A timeout bump that hides a real regression is a worse outcome than the red build, and it is the outcome the person asking almost never wants once it's named. The playbook for reproducing and classifying flake is in `references/config-and-triage.md`.

## Reference files

- `references/waiting-recipes.md` — code for every signal type: element state, navigation, network, downloads and dialogs, the clock API, eventual consistency, animations, overlays, frames, and the last-resort escapes.
- `references/anti-patterns.md` — before/after rewrites of the recurring mistakes, each with the failure it actually causes.
- `references/config-and-triage.md` — annotated `playwright.config.ts`, the timeout budget in practice, CI and retry policy, and a triage playbook for reproducing and fixing flake.

## A note on languages

Examples here are TypeScript, which is where most Playwright suites live. The principles and the API names carry over directly to Python (`page.wait_for_timeout`, `expect(locator).to_be_visible()`), Java, and .NET — the auto-waiting locators, the retrying assertions, and the timeout layers are the same engine underneath. Match the target project's language and test-runner conventions when writing.
