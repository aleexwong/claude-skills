---
name: project-memory
description: Set up and maintain the memory a repository gives every future session — CLAUDE.md conventions and gotchas, an append-only LESSONS ledger of mistakes and the rules they produced, skills for recipes that cost more than one failed attempt, a subagent roster scoped by blast radius, and a permissions allowlist. Use when starting or onboarding a project for agent work, when a session ends and something was learned, when a mistake is caught, when the same environment problem is being solved from scratch a second time, or when asked to record, remember, or document how this repo works.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Project memory

Every session starts from zero. The repo does not have to. Five artifacts carry what the session cannot, and each answers a different question — putting a fact in the wrong one is why it gets rediscovered.

| Artifact | Answers | Shape |
|---|---|---|
| `CLAUDE.md` | How does this repo work? Commands, structure, conventions, gotchas. | Evergreen, edited in place |
| `.claude/LESSONS.md` | What has already gone wrong here, and what rule follows? | Append-only ledger |
| `.claude/skills/*/SKILL.md` | How do I do the thing that took three tries to get right? | Evergreen recipe, versioned |
| `.claude/agents/*.md` | Who does what, with which tools, on which model? | Evergreen roster |
| `.claude/settings.json` | Which commands should never prompt? | Read-only + build allowlist |

## Read before you write

- **Check `.claude/skills/` before solving any environment or workflow problem from scratch.** A session burned two failed runs rediscovering a browser-launch workaround that was already written down — and the half-remembered version it reconstructed was *worse*, pinning a versioned path that breaks on the next upgrade.
- **If the branch has been open a while, run `git log origin/main -- .claude/`.** The recipe may have landed after your base commit, so it is absent from your working tree and invisible to a grep.
- **Verify the premise before restating it.** "Picking up where I left off" needs one `git diff` against the base branch. Sometimes there is nothing to continue.

## Ledger entries: cost, then rule

Vague advice ("be careful with fonts") is dead weight. Every entry says what was done, what it cost, and the rule that follows:

```markdown
## Session: <what was being built> (PR #NN)

### <Category: Verification / Code quality / Design / Process>

**<One-line description of the mistake, in bold.>**
What actually happened, concretely — the exact call, the exact wrong output,
and what it nearly caused.
→ **Rule:** the thing to do instead, specific enough to follow without context.
```

Keep a **"What worked, and is worth repeating"** section at the end of each entry. It is the half everyone skips, and it is where the next project's skills come from — verifying math against published external values, driving the real page at two breakpoints, counting lint warnings before and after.

## Promotion: where a lesson actually belongs

A lesson filed in the wrong place gets paid for twice.

| The lesson is… | It goes in… |
|---|---|
| A one-off slip in judgement | `LESSONS.md`, and nowhere else |
| A durable fact about this repo (a global reset, a legacy route, a length limit) | `CLAUDE.md` **Gotchas**, with a one-line pointer from the ledger |
| A recipe that cost more than one failed attempt to get right | A skill in `.claude/skills/` |
| A pattern that has now bitten in a second project | A portable skill or plugin, outside the repo |

The ledger is the intake queue, not the destination. When an entry graduates, leave the entry and point at where it went.

## Document consequences, not settings

A settings note is only useful if it names what breaks. `font-synthesis: none` is a line of CSS; the useful memory is what it *does*:

> Any face missing from the font URL fails silently rather than being faked. `font-extrabold` on a heading renders as 700 with no error anywhere. Italics need the `1,...` axis in the request or every `italic` element renders upright.

Write the trap, the symptom, and the silent part. "Maps render blank without a valid token" beats "requires `VITE_MAPBOX_TOKEN`".

## Roster agents by blast radius

Pick the model by the cost of being wrong, not the size of the diff, and narrow the tools to the job:

| Model | For |
|---|---|
| Cheapest | Read-only exploration, file search, summarizing, simple renames — fan several out in parallel |
| Mid | Scoped implementation, component work, tests, code review |
| Most capable | Anything expensive to get wrong: data model changes, security rules, cross-cutting refactors |

Two rules that make the roster work:

- **An advisor gets read-only tools and is told so in its prompt.** "You advise and plan only — do not edit files" next to `tools: Read, Grep, Glob` means the boundary is enforced, not requested.
- **Ground each agent in repo specifics.** A reviewer that knows this repo's real invariants — legacy routes that must keep working, the length limits, where business logic is allowed to live — finds real defects. A generic "review this code" agent finds style opinions.

Prefer model aliases over pinned IDs so the roster tracks releases without edits.

## Keep it honest

Prune what has become false. A gotcha describing a bug that was fixed last month is worse than no gotcha — it sends the next session chasing a ghost, and it teaches them the file is unreliable. When you fix the underlying cause, delete the warning in the same commit.
