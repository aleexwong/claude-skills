---
name: verify-before-claiming
description: Build a check that can actually fail before claiming a change works, is fixed, or is safe. Use when about to report something as done or verified, when a check passes suspiciously easily, when an assertion fails and you are about to act on the failure, when auditing whether an existing guard or protection actually fires, and when assembling test fixtures or reporting results back to a user. Covers silent-fallback probes, assertions that never ran, guards that execute too late to protect anything, negative controls, and how to scope a claim to what was measured.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Verify before claiming

A claim is worth exactly the check behind it. Almost no bad claim comes from a check that failed — it comes from a check that *could not have failed*, and passed. So before running anything, answer one question:

> **What would this check print if the thing were completely broken?**

If the answer is "the same thing", it is not a check. Fix the check first, then run it.

## The three species of fake check

**1. Probes that fall back silently.** An availability API that answers "can this be rendered / resolved / served?" is not answering "is this thing present?", because a fallback counts as yes. A real case: `document.fonts.check()` returned `true` for a font weight that does not exist, for an italic face that was never loaded, and kept returning `true` on a page that had failed to load entirely.

The fix is calibration against a deliberately bogus input:

```js
// A family that is not applied measures identically to a family that does not exist.
const isFallback = Math.abs(width("400 40px 'DM Sans'") - width("400 40px 'NoSuchFontXYZ'")) < 0.01;
```

Whatever the domain, run the nonsense input through the same probe. If nonsense and truth score the same, the probe is blind — replace it with a measurement that has to differ.

**2. Assertions that never matched anything.** A smoke test compared substrings against `document.body.innerText`. `innerText` applies CSS `text-transform`, and several headings were `uppercase`, so `"Realistic"` never matched `"REALISTIC HALF MARATHON FINISH"`. Every assertion reported MISS while the page rendered perfectly — a working feature was nearly reported as broken.

- Assert against the untransformed source (`textContent`), or case-insensitively.
- Log `count()` and the matched text for every selector. A selector matching zero elements and a selector matching the wrong element both look like a result.
- If two runs are supposed to differ, diff them. Two "before/after" screenshots of the same element prove nothing and look like proof.

**3. Guards that cannot fire.** Reviewing a codebase's protections, three were theater:

| Guard | Why it never fired |
|---|---|
| "processing timeout" that `throw`s inside `setTimeout` | The throw lands on a later tick, outside the `try/catch`. Nothing is aborted. |
| "8MB request size" check calling `JSON.stringify(req.body)` | The body was already parsed into memory. The cost was paid before the check ran. |
| Regex denylist against prompt injection | Trivially reworded around. Denylists do not hold against an adversary. |

For every guard ask: **on what tick does it run, what has already been spent by then, and against whom.** A guard that answers those badly is worse than no guard, because it stops anyone from looking.

## Fingerprint every measurement

Print the thing you measured next to the number you measured. Dump the element's text alongside its computed styles; dump the row alongside the aggregate; dump the request alongside the latency.

This is what caught a whole run of plausible-looking font metrics: the "page body" being measured had the text `"agent-proxy relay: this proxy only accepts…"` in Times New Roman. Without the content next to the numbers, those measurements read as a real finding about the app.

## Sweep, do not glance

A screenshot tells you *that* something is wrong. A sweep tells you *where* and *how much*, and leaves you a pass condition:

```js
for (let i = 0; i <= STEPS; i++) {
  await moveCursorTo(i / STEPS);
  const m = await measure();   // distance from each edge of the container
  // > 0 on any edge means clipped
}
```

Stepping a cursor across an elevation chart and recording the tooltip against its stage on all four edges found a second clipped edge that no static screenshot showed, and proved vertical placement needed no change at all. Then **re-run the identical sweep after the fix and report both columns.** Vary one axis while you are there — two viewports exercised different clamp behaviour.

## Anchor to something outside the system

Compiling is not correctness, and a test that only agrees with the implementation is a mirror. Check the output against a value the system did not produce:

- Published reference values (Magnus dew point: 30 °C / 50 % → 18.4 °C; published altitude VO₂max figures).
- A golden snapshot with the numbers written into the test, not regenerated from current behaviour.
- Independently sourced fixtures, where the disagreement between sources *is* the finding: five GPX recordings of the same marathon course reported elevation gain from 90 m to 263 m. That 2.9× spread is why gain was removed from the identity key.

## A corpus with no negative control cannot fail

If every fixture is supposed to match, a matcher that returns "yes" to everything scores 100 %. Ship the case that must **not** match, at a known distance from the ones that must, and assert both directions: same-course pairs measured 12–185 m apart, control-vs-target > 1 km, so the thresholds have evidence under them rather than vibes.

Document each fixture's provenance, its rights, and the measured number it exists to lock. A fixtures README that says why each file is there is the difference between a corpus and a folder.

## Counts, not impressions

"No new lint warnings" is a number. Record it before and after — a 96-warning baseline eyeballed twice is not a comparison. Same for bundle size, query count, test count, error rate.

## A failing check is a claim too

Before acting on a failure — reverting, rewriting, escalating — confirm the failure is real. Species 2 above cost most of a session and nearly got a working feature declared broken. Reproduce the failure by a second, independent route before you treat it as fact. The same applies to premises: verify "picking up where I left off" against `git log`/`git diff` before restating it as fact; the branch may be at exact parity with main.

## Scope the claim to what you measured

- Say what you checked and how. If a screenshot did not cover a case, say the case is unchecked rather than letting the shot stand in for it.
- **Confidence and scope are separate axes** and collapsing them is how a finding becomes a lie six weeks later. "Certain, on one component type" is a normal and honest state.
- When you ship a value that deviates from the spec or the default, say so at the value, with the evidence that would change it.
- Silence is not agreement. If you overrode someone's suggested approach and they did not push back, record their original instinct as an open question, not as consent.
- Distinguish a defect from a preference. When someone reacts badly to output, check whether they are rejecting a choice or catching a bug. Fix the bug, say so plainly, and do not encode it as a preference.

## Close with a handoff ledger

End the work with three lists, not a summary:

```
Verified   — claim, and the check that backs it (with numbers)
Unverified — what was not checked, and why
Blocked    — exact step, who has to do it, roughly how long
```

`Owner action needed: DNS access or deploy access for the well-known file. ~30 minutes.` is a handoff. "Mostly done, some follow-ups remain" is not.
