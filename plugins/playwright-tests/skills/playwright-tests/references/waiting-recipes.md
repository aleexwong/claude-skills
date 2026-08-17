# Waiting recipes

Code per signal type. Find your case, take the pattern.

## Element state

Every matcher retries until it passes or the expect timeout expires. That retry loop *is* the wait.

```ts
await expect(page.getByRole('alert')).toBeVisible();
await expect(page.getByTestId('spinner')).toBeHidden();          // also passes if detached
await expect(page.getByRole('button', { name: 'Save' })).toBeEnabled();
await expect(page.getByLabel('Email')).toHaveValue('a@b.com');
await expect(page.getByRole('heading')).toHaveText('Order #1234');
await expect(page.getByRole('checkbox')).toBeChecked();
await expect(page.getByRole('link')).toHaveAttribute('href', '/docs');
await expect(page.getByTestId('row')).toHaveClass(/selected/);

await expect(page.getByRole('row')).toHaveCount(25);             // list finished loading
await expect(page.getByRole('listitem')).toHaveText(['Alpha', 'Beta']);  // count + contents
await expect(page.getByTestId('total')).not.toHaveText(previous); // waits for a change
```

Need the value in JS? Assert first so the read isn't a race:

```ts
await expect(total).toHaveText(/^\$\d+\.\d{2}$/);
const text = await total.textContent();
```

## Navigation

```ts
await page.getByRole('link', { name: 'Dashboard' }).click();
await expect(page).toHaveURL(/\/dashboard/);
await page.waitForURL('**/onboarding/step-1');   // imperative: block until redirects settle
```

Both poll the URL, so client-side routing works too. Never follow `goto` with a sleep — assert the content you came for instead.

## Network

Create the promise **before** the action, or the response can land before you listen:

```ts
const responsePromise = page.waitForResponse(
  r => r.url().includes('/api/orders') && r.request().method() === 'POST' && r.ok()
);
await page.getByRole('button', { name: 'Place order' }).click();
const { orderId } = await (await responsePromise).json();
```

Matching a GraphQL operation:

```ts
page.waitForResponse(r => r.url().endsWith('/graphql')
  && r.request().postDataJSON()?.operationName === 'GetInvoices');
```

Mocking removes the wait entirely — the highest-leverage change in most suites:

```ts
await page.route('**/api/invoices*', route => route.fulfill({ status: 200, body: JSON.stringify(data) }));
await page.route('**/api/**', route => route.abort('failed'));      // error paths
await page.route(/\.(png|woff2)$/, route => route.abort());         // drop noise
await context.setOffline(true);
```

A timer inside a route handler is fine — it shapes the fixture rather than sleeping in the test:

```ts
await page.route('**/api/slow', async route => {
  await new Promise(r => setTimeout(r, 1_000));
  await route.fulfill({ status: 200, body: '{}' });
});
await expect(page.getByTestId('spinner')).toBeVisible();   // the loading state exists
await expect(page.getByTestId('spinner')).toBeHidden();    // and it resolves
```

## Events: downloads, uploads, popups, dialogs

Same shape throughout — subscribe, then trigger.

```ts
const downloadPromise = page.waitForEvent('download');
await page.getByRole('button', { name: 'Export CSV' }).click();
const download = await downloadPromise;
expect(download.suggestedFilename()).toBe('export.csv');

const chooserPromise = page.waitForEvent('filechooser');
await page.getByText('Upload').click();
await (await chooserPromise).setFiles('fixtures/avatar.png');

const popupPromise = page.waitForEvent('popup');
await page.getByRole('link', { name: 'Open report' }).click();
await expect((await popupPromise).getByRole('heading')).toHaveText('Report');

page.once('dialog', dialog => dialog.accept());   // required before the trigger
await page.getByRole('button', { name: 'Delete' }).click();
```

For a plain `<input type="file">`, skip the event: `await page.getByLabel('Avatar').setInputFiles(…)`.

## Time: debounces, polls, countdowns, expiry

Install the clock before the navigation that starts the timers.

```ts
await page.clock.setFixedTime(new Date('2024-02-02T10:00:00'));   // deterministic dates
await expect(page.getByTestId('current-time')).toHaveText('2/2/2024, 10:00:00 AM');

await page.clock.install();
await page.goto('/dashboard');
await page.clock.fastForward('05:00');            // jump, firing pending timers
await expect(page.getByText('You have been logged out')).toBeVisible();
```

`runFor(ms)` ticks through firing timers in order — use it when each tick must render (a countdown, a poll that fires five times). `pauseAt` freezes at an instant; `resume` restarts natural time.

```ts
await page.getByLabel('Search').fill('lap');
await page.clock.runFor(400);                     // past a 300ms debounce
await expect(page.getByRole('listitem')).toHaveCount(3);
```

If the clock can't be installed (cross-origin iframe, its own worker), assert the debounce's *result* — that's still a condition, and the sleep buys nothing.

## Eventual consistency

Both forms exit the moment they succeed. Give them an `intervals` ramp so the fast path stays fast.

```ts
await expect.poll(async () => {
  const res = await request.get('/api/search?q=widget');
  return (await res.json()).total;
}, { message: 'search index picks up the new item', timeout: 30_000, intervals: [500, 1_000, 5_000] })
  .toBeGreaterThan(0);

await expect(async () => {
  await page.reload();                            // toPass, for retries with side effects
  await expect(page.getByRole('row')).toHaveCount(3);
}).toPass({ timeout: 30_000, intervals: [1_000, 2_000, 5_000] });
```

These are for asynchronous *systems*. Wrapping a UI race in a retry hides it rather than fixing it.

## Animation and visual stability

Actionability's *stable* check already waits out CSS transitions. Screenshot assertions retry until two consecutive captures match, and disable animations by default:

```ts
await expect(page).toHaveScreenshot('dashboard.png', { mask: [page.getByTestId('timestamp')] });
```

Mask what's nondeterministic rather than raising the diff tolerance until real regressions fit through. To kill a long entrance animation outright:

```ts
await page.addStyleTag({ content: '*,*::before,*::after{animation-duration:0s!important;transition-duration:0s!important}' });
```

## Overlays and interrupters

A conditional `isVisible()` races them. Handle it whenever it appears:

```ts
await page.addLocatorHandler(
  page.getByRole('button', { name: 'Accept all cookies' }),
  async locator => { await locator.click(); },
  { times: 1 },
);
```

Better, prevent it — deleting an interrupter beats handling one, because a handler is one more thing that can race:

```ts
await context.addCookies([{ name: 'cookie_consent', value: 'accepted', url: baseURL }]);
await context.addInitScript(() => window.localStorage.setItem('tour_seen', '1'));
```

## Frames, shadow DOM, canvas

`frameLocator` resolves lazily, so it waits like any locator. Open shadow roots are pierced automatically.

```ts
const frame = page.frameLocator('#payment-iframe');
await expect(frame.getByLabel('Card number')).toBeVisible();
```

Canvas and WebGL expose no DOM. Ask the app for a signal — `data-rendered="true"`, a resolved promise on `window` — and assert that. Failing that, a masked screenshot assertion still beats a sleep, because it retries until the pixels stop changing.

## Last-resort escapes

```ts
await page.getByTestId('drawer').waitFor({ state: 'attached' });   // attached|detached|visible|hidden
await page.waitForFunction(() => window.__APP_READY__ === true);
```

`waitForFunction` is reasonable for an app-exposed readiness flag, a smell when it reaches into internals a user can't see. `page.waitForSelector` still works but is superseded by `toBeVisible()` (better failure message) and `locator.waitFor()`.

And the debugging-only pair — use `page.pause()` to open the inspector rather than sleeping blind:

```ts
await page.waitForTimeout(1000);   // delete before committing
await page.pause();
```
