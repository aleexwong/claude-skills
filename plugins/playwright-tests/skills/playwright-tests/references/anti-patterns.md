# Anti-patterns and their fixes

Each entry is a pattern that shows up constantly in real suites, the failure it actually causes, and the rewrite. Read the failure column first — it is the part that convinces a reviewer.

## Quick audit

Run these before reading any code. They find most of it.

```bash
# hard waits and network-idle guessing
rg -n 'waitForTimeout|wait_for_timeout|networkidle|setTimeout\(.*resolve' tests/

# sampled checks masquerading as assertions
rg -n 'expect\(await .*\.(isVisible|isEnabled|isChecked|isEditable|textContent|innerText|inputValue|count)\(\)' tests/

# racy conditionals and silent selector fallbacks
rg -n 'if \(await .*\.(isVisible|count)\(\)|\.first\(\)|\.nth\(0\)' tests/

# forced actions and timeout inflation
rg -n 'force: *true|timeout: *[0-9]{5,}|setTimeout\(0\)|test\.setTimeout\(0\)' tests/

# implementation-coupled selectors
rg -n "page\.locator\(['\"][.#]" tests/
```

---

## 1. The hard wait

```ts
// wrong
await page.getByRole('button', { name: 'Save' }).click();
await page.waitForTimeout(2000);
await expect(page.getByText('Saved')).toBeVisible();
```

**Fails because** 2000ms is either too short on a loaded runner (flake) or too long everywhere else (2s × every run × every occurrence). It also asserts nothing on its own.

```ts
// right
await page.getByRole('button', { name: 'Save' }).click();
await expect(page.getByText('Saved')).toBeVisible();
```

The assertion already retries for up to the expect timeout and returns the instant the text appears. Deleting the sleep makes the test both faster and stricter.

---

## 2. The sampled check

```ts
// wrong
expect(await page.getByText('Welcome').isVisible()).toBe(true);
expect(await page.getByRole('row').count()).toBe(5);
```

**Fails because** `isVisible()` and `count()` resolve against the DOM as it is *right now*. There is no retry, so the assertion races the render. This is the pattern that causes the sleeps in #1 to get added in the first place.

```ts
// right
await expect(page.getByText('Welcome')).toBeVisible();
await expect(page.getByRole('row')).toHaveCount(5);
```

---

## 3. The racy conditional

```ts
// wrong
if (await page.getByTestId('cookie-banner').isVisible()) {
  await page.getByRole('button', { name: 'Accept' }).click();
}
```

**Fails because** on a fast machine the banner hasn't rendered when the check runs, so the branch is skipped — and then the banner appears over the button you click three lines later. Worse, the same shape is often used to guard assertions, producing a test that "passes" by verifying nothing.

```ts
// right — remove the interrupter entirely
await context.addCookies([{ name: 'cookie_consent', value: 'accepted', url: baseURL }]);

// or handle it whenever it appears
await page.addLocatorHandler(
  page.getByRole('button', { name: 'Accept' }),
  async locator => { await locator.click(); },
);
```

---

## 4. `networkidle` as a readiness signal

```ts
// wrong
await page.goto('/dashboard');
await page.waitForLoadState('networkidle');
```

**Fails because** the docs discourage it, and any app with polling, websockets, telemetry, or a support-chat widget never has a 500ms quiet period. The call then burns its full timeout on every single run and still tells you nothing about whether the dashboard rendered.

```ts
// right
await page.goto('/dashboard');
await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
```

Assert the thing you came to check. That is the readiness signal.

---

## 5. Escaping actionability instead of fixing the cause

```ts
// wrong
await page.getByRole('button', { name: 'Submit' }).click({ force: true });
await page.getByRole('button', { name: 'Submit' }).dispatchEvent('click');
```

**Fails because** `force: true` skips the non-essential checks and `dispatchEvent` skips all of them. The test goes green while the real defect stands: if the button is covered, disabled, or unstable for Playwright, it is covered, disabled, or unstable for users. You have converted a caught bug into a shipped one.

```ts
// right — wait for the blocker to clear, then click normally
await expect(page.getByTestId('modal-backdrop')).toBeHidden();
await page.getByRole('button', { name: 'Submit' }).click();
```

If the overlay is legitimate and permanent, that is a product bug worth reporting rather than routing around.

---

## 6. Global timeout inflation

```ts
// wrong
export default defineConfig({
  timeout: 300_000,
  expect: { timeout: 60_000 },
});
```

**Fails because** every broken locator now costs a full minute instead of five seconds, so a red suite takes twenty minutes to tell you what it used to say in two. And a page that quietly got four times slower still passes, so the config change buries the regression it was added to work around.

```ts
// right — narrow scope, stated reason
await expect(page.getByTestId('report')).toBeVisible({ timeout: 30_000 }); // PDF render, p95 ~18s

test('generates the annual export', async ({ page }) => {
  test.slow();   // triples the configured timeout, tracks the baseline
});
```

---

## 7. Hand-rolled retry loops

```ts
// wrong
let ok = false;
for (let i = 0; i < 10; i++) {
  if (await page.getByText('Ready').isVisible()) { ok = true; break; }
  await page.waitForTimeout(1000);
}
expect(ok).toBe(true);
```

**Fails because** it reimplements `toPass` badly: no useful error message on failure (`expected true, received false` tells you nothing), a fixed interval that is both too eager and too slow, and ten seconds of wall clock in the failure case regardless.

```ts
// right
await expect(page.getByText('Ready')).toBeVisible({ timeout: 10_000 });

// or, when the retry needs a side effect
await expect(async () => {
  await page.reload();
  await expect(page.getByText('Ready')).toBeVisible();
}).toPass({ timeout: 30_000, intervals: [1_000, 2_000, 5_000] });
```

---

## 8. Listening after the fact

```ts
// wrong
await page.getByRole('button', { name: 'Export' }).click();
const download = await page.waitForEvent('download');
```

**Fails because** a fast download fires before the listener attaches, and the test then hangs until timeout. It passes locally on a slow dev server and fails on CI, which is the most confusing failure mode there is.

```ts
// right
const downloadPromise = page.waitForEvent('download');
await page.getByRole('button', { name: 'Export' }).click();
const download = await downloadPromise;
```

Same rule for `waitForResponse`, `waitForRequest`, `waitForEvent('popup')`, and `page.on('dialog')`.

---

## 9. `.first()` to silence strict mode

```ts
// wrong
await page.getByRole('button', { name: 'Delete' }).first().click();
```

**Fails because** strict mode caught something real — the page has several matching buttons — and `.first()` answers it by picking whichever one the DOM happens to order first. The test then deletes the wrong row the day someone reorders the list, and it does so silently.

```ts
// right — say which one you mean
await page.getByRole('row', { name: 'Invoice 1024' })
  .getByRole('button', { name: 'Delete' })
  .click();

await page.getByRole('listitem').filter({ hasText: 'Widget' })
  .getByRole('button', { name: 'Delete' }).click();
```

`.first()` is fine when "any of them" is genuinely the intent and the set is homogeneous — say, asserting that at least one result card renders.

---

## 10. Implementation-coupled locators

```ts
// wrong
await page.locator('.css-1x2y3z > div:nth-child(2) button').click();
await page.locator('#root > div > div.MuiBox-root').click();
```

**Fails because** the selector describes the render tree rather than the interface. It breaks on any refactor, on a CSS-in-JS hash change, on a component-library upgrade — and when it breaks it says "element not found" rather than anything about the feature.

```ts
// right
await page.getByRole('button', { name: 'Add to cart' }).click();
await page.getByLabel('Delivery address').fill('…');
await page.getByTestId('checkout-summary');   // when no accessible handle exists
```

Role and label locators double as an accessibility check: if you cannot address the control by its accessible name, neither can a screen-reader user.

---

## 11. UI login in `beforeEach`

```ts
// wrong
test.beforeEach(async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('hunter2');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.waitForTimeout(2000);
});
```

**Fails because** it pays a full login — network, redirects, session setup — before every one of your two hundred tests, and it makes an auth hiccup fail tests that have nothing to do with auth. It is usually the largest single block of waiting in a suite.

```ts
// right — log in once in a setup project, reuse the state
// auth.setup.ts
setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible();
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});

// playwright.config.ts
projects: [
  { name: 'setup', testMatch: /auth\.setup\.ts/ },
  { name: 'chromium', dependencies: ['setup'], use: { storageState: 'playwright/.auth/user.json' } },
]
```

Keep one test that logs in through the UI — the login form deserves coverage. Just not two hundred of them.

---

## 12. Building fixture data through the UI

```ts
// wrong
test('archives a project', async ({ page }) => {
  await page.getByRole('button', { name: 'New project' }).click();
  await page.getByLabel('Name').fill('Test project');
  await page.getByRole('button', { name: 'Create' }).click();
  await page.waitForTimeout(1000);
  // …the actual test finally starts here
});
```

**Fails because** the setup is slower and more fragile than the assertion it exists to enable, and a break in project creation now fails the archive test — a misleading signal that costs someone an hour.

```ts
// right
test('archives a project', async ({ page, request }) => {
  const res = await request.post('/api/projects', { data: { name: 'Test project' } });
  const { id } = await res.json();
  await page.goto(`/projects/${id}`);
  await page.getByRole('button', { name: 'Archive' }).click();
  await expect(page.getByText('Archived')).toBeVisible();
});
```

---

## 13. Tests that depend on each other

```ts
// wrong
let projectId: string;
test('creates a project', async ({ page }) => { /* sets projectId */ });
test('edits the project', async ({ page }) => { /* reads projectId */ });
```

**Fails because** it forbids parallelism, breaks under `--shard`, and makes a single early failure cascade into a wall of red that hides the one real cause. Running a single test with `-g` stops working, which is exactly when you most need it.

```ts
// right — each test creates what it needs, via API, and cleans up after itself
test('edits a project', async ({ page, request }) => {
  const { id } = await createProject(request);
  // …
});
```

If steps genuinely must share a journey, put them in one test with `test.step()` blocks — that is the honest way to express "these are sequential".

---

## 14. Sleeping for an app timer

```ts
// wrong
await page.getByLabel('Search').fill('lap');
await page.waitForTimeout(500);              // the debounce
await expect(page.getByRole('listitem')).toHaveCount(3);
```

**Fails because** the wait tracks a constant in the app's source that nobody will update here when it changes, and the test spends half a second per search either way.

```ts
// right
await page.clock.install();
await page.goto('/search');
await page.getByLabel('Search').fill('lap');
await page.clock.runFor(600);
await expect(page.getByRole('listitem')).toHaveCount(3);
```

If the clock isn't installable, just assert the outcome — `toHaveCount` will retry across the debounce on its own. The sleep buys nothing even then.

---

## 15. Retries as the fix

```ts
// wrong
export default defineConfig({ retries: 5 });   // "it passes eventually"
```

**Fails because** retries are a *detector*, not a cure. A test that only passes on attempt three is failing, and the flake it is papering over is frequently a real race in the application — the exact class of bug that reaches users as "it works if I refresh".

```ts
// right
export default defineConfig({ retries: process.env.CI ? 2 : 0 });
```

Two retries on CI absorbs infrastructure noise while keeping the report's `flaky` bucket meaningful. Treat that bucket as the to-do list, not as a passing column. Locally, zero retries — you want to see the flake while you have the context to fix it.
