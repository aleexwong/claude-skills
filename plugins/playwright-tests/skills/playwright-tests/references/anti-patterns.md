# Anti-patterns

## Audit first

```bash
rg -n 'waitForTimeout|wait_for_timeout|networkidle|setTimeout\(.*resolve' tests/
rg -n 'expect\(await .*\.(isVisible|isEnabled|isChecked|textContent|inputValue|count)\(\)' tests/
rg -n 'if \(await .*\.(isVisible|count)\(\)|\.first\(\)|\.nth\(0\)|waitForSelector' tests/
rg -n 'force: *true|timeout: *[0-9]{5,}|setTimeout\(0\)' tests/
rg -n "page\.locator\(['\"][.#]" tests/
```

## The catalogue

| Pattern | What it actually breaks | Rewrite |
|---|---|---|
| `waitForTimeout(2000)` before an assertion | Too short on a loaded runner, too long everywhere else, and asserts nothing | Delete it; the assertion already retries |
| `expect(await loc.isVisible()).toBe(true)` | No retry — races the render. This is why the sleeps got added | `await expect(loc).toBeVisible()` |
| `expect(await loc.count()).toBe(5)` | Same, plus a useless message on failure | `await expect(loc).toHaveCount(5)` |
| `if (await loc.isVisible())` | False on a fast machine, so the branch — and its assertions — silently skip | Seed the state away, or `addLocatorHandler` |
| `waitForLoadState('networkidle')` | Discouraged by the docs; never fires in an app that polls, so it burns its full timeout every run | Assert the thing you came to check |
| `click({ force: true })` | Skips the checks that caught a real defect: if it's covered for Playwright, it's covered for users | Wait for the blocker to clear, then click |
| `dispatchEvent('click')` / `press('Enter')` to dodge flake | Skips actionability entirely; the race just fails somewhere less obvious | Fix the cause and use `click()` |
| Global `expect.timeout: 60_000` | Every broken locator costs a minute; a 4× slower page still passes | Per-assertion `{ timeout }` with the reason stated |
| `test.setTimeout(0)` / any `timeout: 0` | Unlimited — a hung test holds a CI slot until the job dies | A real number, or `test.slow()` |
| `while` + sleep retry loop | Reimplements `toPass` with no error message and a fixed interval | `toPass({ intervals })`, or a longer assertion timeout |
| `waitForEvent('download')` *after* the click | Fast downloads fire before the listener attaches, then hang to timeout | Create the promise before the trigger |
| `.first()` to silence strict mode | Strict mode caught a real ambiguity; this picks whichever the DOM ordered first and fails silently later | Chain or `.filter()` to say which one you mean |
| `page.locator('.css-1x2y3z > div:nth-child(2)')` | Describes the render tree, breaks on any refactor, and says "not found" instead of naming the feature | `getByRole` / `getByLabel` / `getByTestId` |
| `retries: 5` | Retries detect flake, they don't cure it — and the flake is often a real race that reaches users as "works if I refresh" | `retries: process.env.CI ? 2 : 0`, and treat the `flaky` bucket as the to-do list |
| Sleeping for an app timer (debounce, poll) | Tracks a constant in the app's source that nobody will update here | `page.clock.runFor()`, or assert the outcome |

## The three structural ones

These are worth code, because the fix isn't a one-liner.

**UI login in `beforeEach`** pays a full login before all 200 tests and fails them all when auth hiccups. Log in once:

```ts
// auth.setup.ts
setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
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

Keep one test that logs in through the form — it deserves coverage. Just not 200 of them.

**Fixture data built through the UI** is slower than the assertion it enables, and a break in creation fails the test you meant to run for an unrelated reason:

```ts
test('archives a project', async ({ page, request }) => {
  const { id } = await (await request.post('/api/projects', { data: { name: 'Test' } })).json();
  await page.goto(`/projects/${id}`);
  await page.getByRole('button', { name: 'Archive' }).click();
  await expect(page.getByText('Archived')).toBeVisible();
});
```

**Tests that share state** forbid parallelism, break under `--shard`, and turn one failure into a wall of red that hides the cause. Give each test its own data via the API. If steps genuinely form one journey, put them in a single test with `test.step()` blocks — that's the honest way to say "these are sequential".
