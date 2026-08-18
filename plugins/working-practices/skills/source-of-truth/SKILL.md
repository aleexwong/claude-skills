---
name: source-of-truth
description: Keep one canonical definition when the same fact would otherwise live in several places — content that ships in multiple output formats, a rule implemented on both the client and the server, a pipeline reachable from an endpoint and a script, a design doc and the code that implements it. Use when about to copy a definition, when adding a second output format or a second caller, when two implementations of one rule have to agree, when writing or citing a spec, and when duplicated logic has already drifted apart. Covers generating, funnelling, contract tests, and stable spec anchors.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# One source of truth, or a test that catches the drift

Duplication is visible; drift is not. Five hand-rolled copies of the same request handler were easy to see and easy to live with — until three of them had *different* CORS semantics and nobody knew which was intended. The cost of a copy is not the copy, it is that every future fix must be applied N times and eventually will not be.

Before pasting the second copy, count the places this fact will now live. Then take the first of these three moves that is available.

## 1. Generate — one definition, many renderings

Name one canonical, structured definition and render every consumer from it.

Page copy that lived inline as React `createElement` calls meant the only machine-readable version of a page was HTML. Moving it to a data model of typed blocks — route → title, description, content blocks — let the same definition render the prerendered HTML, a `.md` mirror for every route, a whole-site `llms-full.txt`, and a machine-readable page index. Four artifacts, one edit, no possibility of drift.

The tell that you need this: someone asks for the content "but in another format" and the honest answer is "I'd have to rewrite it."

## 2. Funnel — one path, every caller

When several entry points do the same work, make them all call one implementation rather than each re-implementing it.

A race-ingest pipeline is reachable from an HTTP endpoint, a seed script, and a backfill script. All three call one HTTP-agnostic `ingest*` function that reuses the exact same engine as the live analyzer, so the stored corpus and the live path cannot disagree. Each caller supplies only what genuinely differs — its own store, its own transport.

State the rule at the top of the shared module, including the tempting shortcut it exists to forbid: *"Do not reuse `POST /api/analyze-gpx` in a loop — this is the dedicated, idempotent ingest."* A funnel without that note leaks a new bypass every quarter.

## 3. Contract-test — when two implementations must exist

Sometimes one definition is impossible: an edge runtime and a build script cannot share a module, a validation rule genuinely has to run in two languages. Then the agreement itself is the thing to test.

Two implementations of one routing contract — one deciding what the edge serves, one deciding where build-time files are written — get a script that asserts they agree:

```
--- Path mapping agrees with the generator ---
PASS  path /calculator     got="/calculator.md"  want="/calculator.md"
```

Three properties make it worth having:

- **Exhaustive, not sampled.** Iterate every real route from the generator's own list, not four hand-picked examples.
- **Pin the behaviour that must not change**, not just the new behaviour. Half that script's assertions exist to prove ordinary browser navigations still get HTML — Chrome, Firefox, Safari and bare `curl` Accept headers, each asserted `false`. The feature is agents getting Markdown; the risk is everyone else getting it too.
- **Make it a command** (`npm run verify-agent-routing`) that prints PASS/FAIL per case and exits non-zero. A contract test nobody can run in one line is documentation.

## When none of the three applies: cite, don't restate

Prose about a decision will be duplicated somewhere. The fix is to make the copies *pointers*.

Keep one numbered design doc and cite it by section from everywhere else — module headers, constants, test fixtures, other docs, CI notes:

```ts
/**
 * Course identity — geometric dedup (RACE_INTELLIGENCE_ENGINE.md §5, §17, §31, §32).
 * Identity is a NEAREST-NEIGHBOUR problem, not a hash-equality problem. §15.2
 */
```

A citation goes stale loudly — the section is gone, the number is wrong, someone notices. A restatement goes stale silently and then contradicts the doc.

Rules that keep the anchors usable:

- **Number sections and never renumber.** Append new ones. A stable `§17.2` is the entire value; renumbering breaks every citation at once.
- **Lock decisions explicitly**, in a dated section ("Decisions — locked for Phase 1"), so a later reader can tell a settled call from an open one.
- **Cite the evidence sections, not just the spec sections.** Fixtures that say "locks the §15.2 result as a regression test" tell you what breaks if you change the threshold.
- **When the shipped value deviates from the doc, say so at the value**, with the evidence and the condition that would change it:

```ts
// §17.2 defaults MATCH to 150 and flags the RunGo/goandrace pair (185 m) for
// review. §38/§39 tune MATCH to ~200 once the same-course distribution is
// confirmed on the seed set — at which point all five Boston sources auto-merge
// (variantCount=5), which is exactly the §26.1 regression assertion. We ship the
// tuned value; the REVIEW band (200–300 m) stays as the safety valve.
export const MATCH_THRESHOLD_M = 200;
```

That comment survives the author. "Tuned empirically" does not.

## Prove the consolidation

A de-duplication that is not verified is a rename with extra risk. Before claiming it: run the contract test, diff one generated artifact against the pre-refactor version, and confirm every former caller now goes through the funnel (grep for the old symbol and expect zero hits outside it). Report the numbers — files collapsed, callers redirected, cases asserted — not the intention.
