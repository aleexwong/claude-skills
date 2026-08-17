# Waiting recipes

Code for every kind of signal, organized by what you are waiting for. Find your case, take the pattern.

- [1. Element state](#1-element-state)
- [2. Navigation and URL](#2-navigation-and-url)
- [3. Network](#3-network)
- [4. Browser events: downloads, uploads, popups, dialogs](#4-browser-events-downloads-uploads-popups-dialogs)
- [5. Time: debounces, polls, countdowns, expiry](#5-time-debounces-polls-countdowns-expiry)
- [6. Eventual consistency](#6-eventual-consistency)
- [7. Animation and visual stability](#7-animation-and-visual-stability)
- [8. Overlays and interrupters](#8-overlays-and-interrupters)
- [9. Frames, shadow DOM, and canvas](#9-frames-shadow-dom-and-canvas)
- [10. Last-resort escapes](#10-last-resort-escapes)

---

## 1. Element state

Every matcher below retries until it passes or the expect timeout expires. That retry loop is the wait — nothing else is needed around it.

```ts
await expect(page.getByRole('alert')).toBeVisible();
await expect(page.getByTestId('spinner')).toBeHidden();      // also passes if detached
await expect(page.getByRole('button', { name: 'Save' })).toBeEnabled();
await expect(page.getByLabel('Email')).toHaveValue('a@b.com');
await expect(page.getByRole('heading')).toHaveText('Order #1234');
await expect(page.getByTestId('summary')).toContainText('shipped');
await expect(page.getByRole('checkbox', { name: 'Terms' })).toBeChecked();
await expect(page.getByRole('link', { name: 'Docs' })).toHaveAttribute('href', '/docs');
await expect(page.getByTestId('row')).toHaveClass(/selected/);
```

**A list finishing its load** is a count assertion, not a sleep:

```ts
const rows = page.getByRole('row');
await expect(rows).toHaveCount(25);        // waits for exactly 25
await expect(rows).toHaveCount(0);         // waits for the empty state
```

`toHaveText` against a list compares element-by-element, which waits for both the count and the contents:

```ts
await expect(page.getByRole('listitem')).toHaveText(['Alpha', 'Beta', 'Gamma']);
```

**Reading a value you need in JS** — assert first so the read is not a race:

```ts
const total = page.getByTestId('total');
await expect(total).toHaveText(/^\$\d+\.\d{2}$/);   // the wait
const text = await total.textContent();             // now safe to sample
```

**Waiting for something to change** rather than to reach a known value:

```ts
await expect(page.getByTestId('total')).not.toHaveText(previousTotal);
```

Negated matchers retry too — `not.toHaveText` waits until the text differs, it does not check once and give up.

---

## 2. Navigation and URL

Clicking a link that navigates already waits for the navigation to commit. Assert where you landed:

```ts
await page.getByRole('link', { name: 'Dashboard' }).click();
await expect(page).toHaveURL(/\/dashboard/);
await expect(page).toHaveTitle(/Dashboard/);
```

`page.waitForURL` is the imperative equivalent, useful when a redirect chain runs and you want to block until it settles before doing anything else:

```ts
await page.getByRole('button', { name: 'Sign in' }).click();
await page.waitForURL('**/onboarding/step-1');
```

For a client-side router that changes the URL without a document load, both of the above still work — they poll the URL rather than listening for a load event.

Do **not** follow `goto` with a sleep. `page.goto()` resolves on the `load` event by default; if the content you care about arrives later, assert on that content:

```ts
await page.goto('/reports');
await expect(page.getByRole('table')).toBeVisible();   // not waitForTimeout(3000)
```

Avoid `waitForLoadState('networkidle')`. The docs mark it discouraged, and any app with polling, telemetry, websockets, or a chat widget never reaches idle — the call then burns its whole timeout on every run.

---

## 3. Network

### Waiting for a response triggered by an action

Create the promise **before** the action, or the response can arrive before you start listening:

```ts
const responsePromise = page.waitForResponse(
  r => r.url().includes('/api/orders') && r.request().method() === 'POST' && r.ok()
);
await page.getByRole('button', { name: 'Place order' }).click();
const response = await responsePromise;
const { orderId } = await response.json();
```

The equivalent `Promise.all([page.waitForResponse(…), locator.click()])` idiom is also correct; the promise-first form just reads better when several things happen between the two.

Reach for this when the outcome is invisible (an analytics beacon, a background save) or when you need the body. When the outcome is visible, assert the visible thing — it survives an endpoint rename and this does not.

### Matching a GraphQL operation

```ts
const opPromise = page.waitForResponse(async r => {
  if (!r.url().endsWith('/graphql')) return false;
  const body = r.request().postDataJSON();
  return body?.operationName === 'GetInvoices';
});
```

### Mocking so there is nothing to wait for

```ts
await page.route('**/api/invoices*', route =>
  route.fulfill({ status: 200, contentType: 'application/json', body: JSON.stringify(invoices) })
);
```

Mocked responses are instant and identical every run. This is the single highest-leverage way to delete waits from a suite.

### Deliberately slow responses, to test loading states

A timer here is fine — it shapes the fixture rather than sleeping in the test:

```ts
await page.route('**/api/slow', async route => {
  await new Promise(r => setTimeout(r, 1_000));
  await route.fulfill({ status: 200, body: '{}' });
});
await page.getByRole('button', { name: 'Load' }).click();
await expect(page.getByTestId('spinner')).toBeVisible();   // assert the loading state exists
await expect(page.getByTestId('spinner')).toBeHidden();    // then assert it resolves
```

### Failure and offline paths

```ts
await page.route('**/api/**', route => route.abort('failed'));
await expect(page.getByRole('alert')).toContainText('Something went wrong');

await context.setOffline(true);   // whole-context offline
```

### Blocking noise that slows every test

```ts
await page.route(/\.(png|jpg|woff2)$/, route => route.abort());
await page.route('**/analytics/**', route => route.abort());
```

---

## 4. Browser events: downloads, uploads, popups, dialogs

All of these follow the same shape: subscribe first, then trigger.

**Download:**

```ts
const downloadPromise = page.waitForEvent('download');
await page.getByRole('button', { name: 'Export CSV' }).click();
const download = await downloadPromise;
await download.saveAs('/tmp/export.csv');
expect(download.suggestedFilename()).toBe('export.csv');
```

**Native file picker:**

```ts
const chooserPromise = page.waitForEvent('filechooser');
await page.getByText('Upload').click();
const chooser = await chooserPromise;
await chooser.setFiles('fixtures/avatar.png');
```

When the input is a plain `<input type="file">`, skip the event entirely: `await page.getByLabel('Avatar').setInputFiles('fixtures/avatar.png')`.

**New tab or window:**

```ts
const popupPromise = page.waitForEvent('popup');
await page.getByRole('link', { name: 'Open report' }).click();
const popup = await popupPromise;
await expect(popup.getByRole('heading')).toHaveText('Report');
```

**Dialogs** must be handled by a listener registered before the trigger, because Playwright auto-dismisses them otherwise and the action would hang if nothing responded:

```ts
page.once('dialog', dialog => dialog.accept());
await page.getByRole('button', { name: 'Delete' }).click();
await expect(page.getByText('Deleted')).toBeVisible();
```

---

## 5. Time: debounces, polls, countdowns, expiry

`page.clock` replaces the app's clock so the test can move time instead of spending it. Install it before the navigation that starts the timers.

**Pin a date for deterministic rendering:**

```ts
await page.clock.setFixedTime(new Date('2024-02-02T10:00:00'));
await page.goto('/');
await expect(page.getByTestId('current-time')).toHaveText('2/2/2024, 10:00:00 AM');
```

**Fast-forward past an idle timeout, a cache TTL, or a trial period:**

```ts
await page.clock.install();
await page.goto('/dashboard');
await page.clock.fastForward('05:00');            // or fastForward(300_000)
await expect(page.getByText('You have been logged out')).toBeVisible();
```

`fastForward` jumps and fires pending timers immediately. `runFor` ticks through the interval firing timers in order, which is what you want when each tick has an observable effect (a countdown that must render every second, a poll that must fire five times).

**Freeze at an instant, then step:**

```ts
await page.clock.install({ time: new Date('2024-02-02T08:00:00') });
await page.goto('/');
await page.clock.pauseAt(new Date('2024-02-02T10:00:00'));
await expect(page.getByTestId('banner')).toBeVisible();
await page.clock.resume();
```

**Debounced search** — the classic reason people sleep 500ms after typing:

```ts
await page.clock.install();
await page.goto('/search');
await page.getByLabel('Search').fill('lap');
await page.clock.runFor(400);                     // past the 300ms debounce
await expect(page.getByRole('listitem')).toHaveCount(3);
```

If the clock can't be installed (a cross-origin iframe, a widget with its own worker), assert on the debounce's *result* with a per-assertion timeout instead — the result is still an observable condition.

---

## 6. Eventual consistency

When the backend needs a moment — a queue, a search index, a replica — poll the condition rather than guessing a duration. Both forms exit the moment they succeed.

**`expect.poll`** wraps a value-producing function in any normal matcher:

```ts
await expect.poll(async () => {
  const res = await request.get('/api/search?q=widget');
  return (await res.json()).total;
}, {
  message: 'search index should pick up the new item',
  timeout: 30_000,
  intervals: [500, 1_000, 2_000, 5_000],
}).toBeGreaterThan(0);
```

**`toPass`** retries a whole block, including multiple assertions and page interactions:

```ts
await expect(async () => {
  await page.reload();
  await expect(page.getByRole('row')).toHaveCount(3);
}).toPass({ timeout: 30_000, intervals: [1_000, 2_000, 5_000] });
```

Use `toPass` when the retry needs a side effect (a reload, a re-request). Use `expect.poll` when you are just re-reading a value. Give both an `intervals` ramp so the fast path stays fast and the slow path doesn't hammer the service.

Neither is a license to wrap flaky UI code in a retry — retrying a race hides it. These are for genuinely asynchronous *systems*, not for assertions you haven't written correctly yet.

---

## 7. Animation and visual stability

Actionability's *stable* check already waits for the bounding box to hold still across two animation frames, so `click`, `hover`, and `screenshot` handle CSS transitions on their own.

Screenshot assertions retry until two consecutive captures match, and disable CSS animations by default:

```ts
await expect(page).toHaveScreenshot('dashboard.png', {
  mask: [page.getByTestId('timestamp')],   // hide inherently unstable regions
  maxDiffPixelRatio: 0.01,
});
```

Mask or stub anything genuinely nondeterministic — clocks, avatars, ads, randomized IDs — rather than raising the diff tolerance until real regressions fit through it.

For a long entrance animation you don't want to watch, cut it at the source:

```ts
await page.addStyleTag({ content: `*, *::before, *::after {
  animation-duration: 0s !important; transition-duration: 0s !important;
}` });
```

---

## 8. Overlays and interrupters

Cookie banners, "rate this app" modals, and session-refresh prompts appear on their own schedule. A conditional `isVisible()` check races them. `addLocatorHandler` runs whenever the element shows up, before the blocked action's actionability check:

```ts
await page.addLocatorHandler(
  page.getByRole('button', { name: 'Accept all cookies' }),
  async locator => { await locator.click(); },
  { times: 1 },
);
```

Better still, prevent the interrupter: set the cookie or localStorage flag in the context so it never renders.

```ts
test.use({ storageState: 'auth/consented.json' });

// or, per-test
await context.addCookies([{ name: 'cookie_consent', value: 'accepted', url: 'https://example.com' }]);
await context.addInitScript(() => window.localStorage.setItem('tour_seen', '1'));
```

Deleting the interrupter is always better than handling it, because a handler is one more thing that can race.

---

## 9. Frames, shadow DOM, and canvas

**Frames** — `frameLocator` resolves lazily, so it waits like any other locator:

```ts
const frame = page.frameLocator('#payment-iframe');
await expect(frame.getByLabel('Card number')).toBeVisible();
await frame.getByLabel('Card number').fill('4242424242424242');
```

**Shadow DOM** needs nothing special: Playwright's locators pierce open shadow roots by default.

**Canvas and WebGL** expose no DOM to assert on. Prefer a signal the app can give you — a `data-render-complete` attribute, a resolved promise on `window`, a test-only event — and ask the app team for one if it doesn't exist:

```ts
await expect(page.getByTestId('chart')).toHaveAttribute('data-rendered', 'true');
```

Failing that, a masked screenshot assertion is a better wait than a sleep, because it retries until the pixels stop changing.

---

## 10. Last-resort escapes

These are legitimate but rarely the best answer. Prefer an assertion whenever the thing you want is observable to a user.

```ts
await page.getByTestId('drawer').waitFor({ state: 'attached' });   // in DOM, maybe not visible
await page.getByTestId('drawer').waitFor({ state: 'detached' });   // removed entirely
```

`waitFor` states are `attached`, `detached`, `visible`, `hidden`. It is useful when you need the element in the DOM before evaluating something against it, and when the meaningful condition really is attachment rather than visibility.

```ts
await page.waitForFunction(() => window.__APP_READY__ === true);
await page.waitForFunction(el => el.scrollHeight > 1000, await handle.elementHandle());
```

`waitForFunction` polls in the page. Reasonable for app-exposed readiness flags; a smell when it is reaching into internals the user can't see, because it couples the test to implementation and will break on the next refactor.

`page.waitForSelector` still works but is superseded — `expect(locator).toBeVisible()` gives a better failure message, and `locator.waitFor()` covers the imperative case.

And the one that is only ever a debugging tool:

```ts
await page.waitForTimeout(1000);   // delete before committing
await page.pause();                // use this instead while investigating — opens the inspector
```
