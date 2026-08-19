# working-practices

Three habits that showed up over and over in shipped work, written down so the next
project starts with them instead of rediscovering them.

They came out of a running app and its API — but nothing in them is about running,
React, or Firebase. Each skill is the general rule plus the concrete incident that
earned it.

## The skills

### `verify-before-claiming`

Almost no bad claim comes from a check that failed. It comes from a check that
*could not have failed*, and passed. This skill is a taxonomy of those: probes that
silently fall back (an availability API returning `true` for a font weight that does
not exist), assertions that never matched anything (`innerText` applying CSS
`text-transform`, so every assertion missed on a perfectly rendered page), and
guards that run too late to guard (a timeout that throws on a later tick, outside
the `try/catch`).

Plus the positive half: fingerprint every measurement with the content it came from,
sweep instead of glancing, anchor to values the system did not produce, and put a
negative control in every fixture corpus — one that matches everything scores 100 %
against fixtures that all match.

### `project-memory`

Every session starts from zero; the repo does not have to. Five artifacts —
`CLAUDE.md`, an append-only `LESSONS.md` ledger, skills for recipes, an agent roster,
a permissions allowlist — each answering a different question, plus a routing table
for deciding which one a new lesson belongs in. A lesson filed in the wrong place
gets paid for twice.

### `source-of-truth`

Duplication is visible; drift is not. Three moves in order — generate every consumer
from one definition, funnel every caller through one path, or contract-test two
implementations that must agree — and when none of them applies, cite a numbered
spec instead of restating it. A citation goes stale loudly. A restatement goes stale
silently and then contradicts the doc.

## Install

Claude Code:

```
/plugin marketplace add aleexwong/claude-skills
/plugin install working-practices@alex-skills
```

Or copy any of `skills/*/` into your project's `.claude/skills/` directory.

Any agent, via [skills.sh](https://skills.sh):

```
npx skills add aleexwong/claude-skills --skill verify-before-claiming
```

## Using them

`verify-before-claiming` and `source-of-truth` trigger during work — before a "this
is done" claim, and before a second copy of a definition. `project-memory` is worth
invoking deliberately at the end of a session that taught you something, and again
when setting up a new repo.
