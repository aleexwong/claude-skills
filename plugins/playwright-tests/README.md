# playwright-tests

A skill for writing Playwright tests that don't need a stopwatch.

Almost every flaky end-to-end test fails for one reason: it asserts at a moment the
author guessed at, rather than a moment the app announced. The usual repair — a
`waitForTimeout` in front of the assertion — makes it worse in both directions. Too
short on a loaded CI runner and it still flakes; too long everywhere else and you pay
the full duration on every run forever. And a sleep asserts nothing, so it converts a
loud failure into a silent one: the feature can break outright and the test still passes.

This skill replaces durations with conditions.

## What it covers

- **Why the wait is usually redundant** — the actionability matrix showing which checks
  `click`, `fill`, `hover`, and friends already run, and which methods (`press`,
  `dispatchEvent`) skip them entirely, which is why "fixing" flake with those makes it
  worse.
- **A decision table** — for each thing you might be waiting for, the assertion or API
  that waits for it properly: element state, list counts, navigation, network responses,
  downloads and dialogs, debounces via `page.clock`, eventually-consistent backends via
  `expect.poll` and `toPass`.
- **A timeout budget** — what each of the seven timeouts bounds, how to read the error
  each one produces, and why raising the global expect timeout is the most damaging
  single line you can add to a config.
- **Structural fixes** — `storageState` instead of a UI login in `beforeEach`, API
  seeding instead of clicking through setup, `page.route` for anything you don't own.
- **An honest escape hatch** — the cases where no observable signal exists, and what a
  legitimate wait has to carry to earn its place.

## Install

Claude Code:

```
/plugin marketplace add aleexwong/claude-skills
/plugin install playwright-tests@alex-skills
```

Or copy `skills/playwright-tests/` into your project's `.claude/skills/` directory.

## When it triggers

Authoring a new suite, reviewing an existing one, or debugging a test that's green
locally and red in CI. It also fires on the symptoms — `waitForTimeout`, `networkidle`,
"Test timeout of 30000ms exceeded", strict mode violations — without anyone having to
say "best practices".

## Reference files

`references/waiting-recipes.md` (code for every signal type), `references/anti-patterns.md`
(fifteen before/after rewrites plus a grep audit), and `references/config-and-triage.md`
(annotated config, timeout budget, flake triage playbook).

Examples are TypeScript; the principles and API names carry directly to the Python, Java,
and .NET bindings.

## License

MIT
