# playwright-tests

A skill for writing Playwright tests that don't need a stopwatch.

Flaky end-to-end tests nearly always assert at a moment the author guessed at rather than
one the app announced. The usual repair — a `waitForTimeout` in front of the assertion —
is wrong in both directions: too short on a loaded CI runner and it still flakes, too long
everywhere else and you pay it on every run forever. And a sleep asserts nothing, so it
converts a loud failure into a silent one.

This skill replaces durations with conditions.

## What it covers

- **Why the wait is usually redundant** — the actionability matrix, including which methods
  (`press`, `dispatchEvent`) skip the checks entirely, which is why "fixing" flake with
  those makes it worse.
- **A decision table** — for each thing you might be waiting for, the API that waits for it:
  element state, list counts, navigation, network, downloads and dialogs, `page.clock` for
  debounces and countdowns, `expect.poll` / `toPass` for eventually-consistent backends.
- **A timeout budget** — what each timeout bounds, how to read the error it produces, and why
  raising the global expect timeout is the most damaging line you can add to a config.
- **Structural fixes** — `storageState` over a UI login in `beforeEach`, API seeding over
  clicking through setup, `page.route` for anything you don't own.
- **An honest escape hatch** — the cases where no signal exists, and what a legitimate wait
  has to carry to earn its place.

## Install

```
/plugin marketplace add aleexwong/claude-skills
/plugin install playwright-tests@alex-skills
```

Or copy `skills/playwright-tests/` into your project's `.claude/skills/` directory.

It triggers on authoring, reviewing, and debugging alike — including the symptoms
(`waitForTimeout`, `networkidle`, "Test timeout of 30000ms exceeded", strict mode
violations) without anyone having to say "best practices".

Examples are TypeScript; the principles and API names carry over to the Python, Java, and
.NET bindings.

## License

MIT
