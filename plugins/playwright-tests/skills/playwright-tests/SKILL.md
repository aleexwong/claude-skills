---
name: playwright-tests
description: Write, review, and repair Playwright tests so they wait on observable conditions instead of fixed durations. Use whenever Playwright or Playwright Test is in play — writing an e2e suite, reviewing specs, or debugging tests that are flaky, slow, or green locally and red in CI. Triggers on waitForTimeout, sleeps or hard waits, networkidle, waitForSelector, "Test timeout of 30000ms exceeded", strict mode violations, retries, or tuning timeouts in playwright.config. Reach for it even when the user only says "this e2e test is flaky" or "add a test for this page".
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Playwright tests that wait on conditions, not clocks

Flaky tests nearly always assert at a moment the author guessed at rather than one the app announced. Every technique here is the same move: **replace a duration with a condition**.

## The rule

`waitForTimeout` — and `sleep()`, `time.sleep()`, a bare `setTimeout` — is a debugging tool, never a committed one. A fixed wait is wrong in both directions: too short on a loaded CI runner, too long everywhere else and paid on every run forever. And it asserts nothing, which turns a loud failure into a silent one — the feature can break outright and the test still passes.

So never ask "how long should I wait?" Ask **"what observable thing am I waiting for?"** That thing is nearly always assertable, and asserting it is both faster (it returns the instant the condition holds) and stricter (it fails if the condition never holds).

## Playwright already waits

Actionability checks run and retry before every action, so a wait in front of a click duplicates them.

| Action | Checks |
|---|---|
| `click`, `dblclick`, `tap`, `check`, `uncheck`, `setChecked` | visible, stable, enabled, receives events |
| `hover`, `dragTo` | visible, stable, receives events |
| `fill`, `clear` | visible, enabled, editable |
| `selectOption` | visible, enabled |
| `screenshot` | visible, stable |
| `focus`, `blur`, `press`, `dispatchEvent`, `setInputFiles` | **none** |

*Stable* means the bounding box held still for two animation frames — so sleeping through an animation is redundant.

The last row is a trap. "Fixing" a flaky `click()` with `dispatchEvent('click')` or `press('Enter')` appears to work because those skip actionability entirely; `force: true` skips the non-essential checks for the same false win. If force-clicking is the only way past an overlay, that overlay covers the button for real users too — a bug report, not a test fix.

## Assert, don't sample

```ts
await expect(page.getByText('Welcome')).toBeVisible();            // retries until true
expect(await page.getByText('Welcome').isVisible()).toBe(true);   // samples once, right now
```

`isVisible()`, `textContent()`, `count()`, and `inputValue()` resolve against the DOM as it is at that instant. Only the `expect(locator)` matchers retry — and this is usually the reason someone added the sleeps in the first place. The same trap in disguise:

```ts
if (await page.getByText('Cookie notice').isVisible()) { /* dismiss */ }
```

On a fast machine the banner hasn't rendered, so the branch is skipped — and then it covers the element you click three lines later. Seed the dismissal in `storageState`, or register `page.addLocatorHandler()`.

## What to use instead

| Waiting for | Use |
|---|---|
| an element to appear / disappear | `expect(loc).toBeVisible()` / `toBeHidden()` |
| a list to finish loading | `expect(rows).toHaveCount(n)` — `0` for the empty state |
| text or a value to update | `toHaveText()` / `toContainText()` / `toHaveValue()` |
| a control to become usable | `toBeEnabled()` |
| a spinner to clear | `expect(spinner).toBeHidden()` — assert the absence |
| navigation | `expect(page).toHaveURL()` or `page.waitForURL()` |
| a network call to land | `page.waitForResponse()`, **started before the action** |
| a download, popup, or dialog | matching `waitForEvent` / `page.on()`, **before the trigger** |
| a debounce, poll, countdown, or auto-dismissing toast | `page.clock` |
| an eventually-consistent backend | `expect.poll()` or `expect(async () => {…}).toPass()` |
| a screenshot to settle | `toHaveScreenshot()` — retries until two shots match |
| attachment, not visibility | `loc.waitFor({ state: 'attached' })` — last resort |
| "the page to be done loading" | assert what you came to check. `waitForLoadState('networkidle')` is discouraged by the docs and never fires in an app that polls |

Code for each case: `references/waiting-recipes.md`.

**Subscribe before you trigger.** Anything that fires *during* an action needs its promise created first, or you race it:

```ts
const responsePromise = page.waitForResponse(r => r.url().includes('/api/cart') && r.ok());
await page.getByRole('button', { name: 'Add to cart' }).click();
await responsePromise;
```

Use it when the outcome is invisible or you need the body. When the outcome is visible, assert the visible thing — it survives an endpoint rename and this doesn't.

**Move time, don't spend it.** A five-minute idle logout takes five minutes only if you let it:

```ts
await page.clock.install();
await page.goto('/dashboard');
await page.clock.fastForward('05:00');
await expect(page.getByText('You have been logged out')).toBeVisible();
```

## Budget the timeouts you keep

Timeouts are a safety net for *stuck*, not a schedule for *slow*.

| Timeout | Default | Bounds | Set via |
|---|---|---|---|
| test | 30s | body + fixtures + `beforeEach` | `timeout` · `test.setTimeout()` · `test.slow()` |
| expect | 5s | one retrying assertion | `expect.timeout` · `{ timeout }` per assertion |
| action | none | one `click`/`fill` (else capped by the test) | `use.actionTimeout` · per-call |
| navigation | none | `goto`, `waitForURL` | `use.navigationTimeout` · per-call |
| `beforeAll`/`afterAll` | 30s | the hook | `test.setTimeout()` inside it |
| global | none | the whole run | `globalTimeout` |

**Raise at the narrowest scope, with the reason stated.** `toBeVisible({ timeout: 30_000 }) // PDF render, p95 ~18s` is reviewable and deletable. Raising the global expect timeout to 30s is the most damaging line you can add to a config: every broken locator now costs 30s instead of 5, and a page that got 4× slower still passes.

**Keep the test timeout above the sum of the waits inside it.** Blow the test budget and you get "Test timeout of 30000ms exceeded", which points at the test. Blow the assertion budget and you get the locator, the actual DOM, and a trace step — the message that says what broke. Prefer `test.slow()` (triples the configured value) over a hardcoded number, and never set any timeout to `0`, which means unlimited.

## Design out the waiting

- **Authenticate once.** A setup project that saves `storageState` removes a UI login — and its waits — from every test. A login form in `beforeEach` is usually the slowest and flakiest thing in a suite.
- **Seed through the API, assert through the UI.** Building fixtures with the `request` fixture is instant; clicking through five screens is slow and fails the wrong test when it breaks.
- **Mock what you don't own.** `page.route()` makes third parties instant and identical, and deletes the network waits they implied. A `setTimeout` inside a route handler is fine — that shapes the fixture rather than sleeping in the test.
- **Keep tests independent.** Playwright gives each test a fresh context; don't undo that with module-level state or ordering dependencies. `fullyParallel: true` finds them.
- **Prefer user-facing locators.** `getByRole`/`getByLabel`/`getByText` survive refactors and fail meaningfully; `getByTestId` when no accessible handle exists. Resolve strict-mode violations by chaining and filtering, not with `.first()`, which silently picks the wrong element.

Config baseline and CI settings: `references/config-and-triage.md`.

## When a wait is genuinely unavoidable

It happens — a third-party widget with no DOM signal, a canvas surface, a cross-origin iframe. Prefer `toPass()` even then, because it exits the moment the condition holds and a fixed wait never can:

```ts
// The payment iframe exposes no mounted event and repaints while its fonts load.
await expect(async () => {
  await expect(frame.getByLabel('Card number')).toBeEnabled();
}).toPass({ timeout: 5_000, intervals: [100, 250, 500] });
```

A fixed duration needs a comment naming the missing signal and a real assertion after it. A sleep with nothing behind it is not a test.

## Auditing a suite

```bash
rg -n 'waitForTimeout|wait_for_timeout|networkidle|force: *true|timeout: *[0-9]{5,}' tests/
rg -n 'expect\(await .*\.(isVisible|isEnabled|textContent|count)\(\)' tests/
rg -n 'if \(await .*\.isVisible\(\)\)|\.first\(\)|waitForSelector' tests/
```

Work in this order — each step removes work from the next: delete the sleeps, replacing each with the condition it stood in for; convert sampled checks to retrying assertions; pull global timeout inflation back to per-assertion overrides; move setup off the UI. Then confirm with `--repeat-each=20` that the suite got faster as well as greener.

When a test is red because the app is genuinely slow or broken, say so rather than tuning a timeout until it passes. A bump that hides a regression is worse than the red build, and it's rarely what the person asking wants once it's named.

## References

- `references/waiting-recipes.md` — code per signal type: element state, network, events, clock, eventual consistency, overlays, frames.
- `references/anti-patterns.md` — the recurring mistakes, what each actually breaks, and the rewrite.
- `references/config-and-triage.md` — annotated config, CI and retry policy, flake triage playbook.

Examples are TypeScript; the principles and API names carry over to the Python, Java, and .NET bindings.
