---
name: code-taste
description: Guide the user through a structured session that excavates how their codebase is actually organized, resolves the parts where it contradicts itself, and produces a durable code conventions doc that agents and teammates can follow. Use this skill whenever the user wants to document or codify how they organize code, define project structure or conventions, decide where new code should live, onboard people or agents to a codebase's layout, reconcile inconsistent patterns across a repo, or asks for a CONVENTIONS.md or architecture-and-layout doc — including when they phrase it as "how we organize code" or complain that generated code lands in the wrong place or doesn't look like the rest of the codebase.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Code Organization Elicitation

Help the user write down how their code is organized, in a form an agent can actually follow. The deliverable is a **conventions doc**: rules for boundaries, placement, naming, decomposition, interfaces, and tests, each tagged with how much force it carries, how confident it is, and where it was observed — kept in the repo and pointed at from `CLAUDE.md`.

## Core principle: the codebase already answered most of this

Design taste has to be elicited from scratch because nothing exists yet. Code organization is the opposite problem — the repo contains thousands of revealed decisions, most of them consistent. So do not interview the user about things their code already says plainly. Read it, state what it says, and spend the session's expensive minutes on the three places reading fails:

1. **Contested** — the repo does the same kind of thing two different ways. Only the user knows which one wins.
2. **Unwritten** — a situation that hasn't come up yet, so there's no evidence at all.
3. **Unexplained** — the pattern is consistent but the *reason* is invisible. Rules without rationale can only be pattern-matched, never extended to a new case.

A round spent confirming something the code states unambiguously is a wasted round.

## Watch for rationalization, not aspiration

Design taste fails when users answer aspirationally. This fails differently: engineers explain fluently, and will generate a principled-sounding reason for an accident on the spot. The tell is chronology — if the rationale appeared only after you pointed at the pattern, and no code written *before* this conversation follows the rule as stated, it's a rationalization, not a convention.

Test it cheaply by naming a case the stated rule would cover and asking what the codebase does there. If the answer is "…huh, we don't do that," you have a preference, not a convention. Both are worth recording — as different things.

Encourage the user to ramble on the *why* rather than composing a crisp rule. The unedited version ("that folder is where things go to die") carries the anti-pattern; the composed version sands it off.

## Excavate before you ask anything

Spend the first pass reading. Aim to arrive at round one already knowing what's settled and what's contested.

- **Shape** — `git ls-files` and the top two or three directory levels. What's the primary axis of the tree: feature, layer, or domain?
- **Current practice** — the most recently added modules (`git log --diff-filter=A --name-only --since=<3 months>`). New code shows intent; old code shows history. They are frequently different, and the difference is the most valuable thing in the repo.
- **Already-delegated rules** — formatter and linter configs, `tsconfig` path aliases, import-boundary plugins, `.editorconfig`, CI lint steps. Anything enforced here is settled and must never become a prose rule.
- **Already-written rules** — `CONTRIBUTING.md`, `CLAUDE.md`, ADRs, per-directory `README`s. Where a written rule and the actual code disagree, that's a round: is the rule dead, or is the code sloppy?
- **The forks** — two or more directories solving the same *kind* of problem differently. These are the contested rounds; collect them.
- **Churn hotspots** — `git log --format= --name-only | sort | uniq -c | sort -rn | head -30`. Files that everyone edits are usually a missing boundary, and the user will have opinions about them.

Report the excavation back before starting rounds: here's what your repo says, here's where it contradicts itself, here's what it doesn't cover. Get corrections. Users routinely say "oh, that whole directory is dead" and delete a third of the agenda.

## The six layers (coarse → fine)

Work top-down; upstream layers constrain downstream ones.

1. **Boundaries** — what counts as a module, package, or service; what forces a split; which direction dependencies are allowed to point; what's never allowed to import what. The load-bearing layer — mistakes here are the expensive ones.
2. **Placement & colocation** — feature-first or type-first; where tests, types, styles, and fixtures live relative to what they serve; where shared code goes, and what threshold promotes something into it.
3. **Naming** — file and directory naming, casing, singular vs plural, suffix vocabulary (`*Service`, `*Repo`, `.types.ts`), barrel/index files, and what a name is obligated to convey.
4. **Size & decomposition** — when a file or function is too big; extraction thresholds; tolerance for duplication versus premature abstraction; whether the codebase prefers a few long functions or many small ones.
5. **Interfaces & data flow** — public versus internal surface, export discipline, how modules talk (direct import, injection, events), where validation happens, where errors are caught versus propagated, where side effects are permitted.
6. **Tests & fixtures** — colocated or mirrored tree, the unit/integration boundary, factory and fixture conventions, mocking appetite, what is required to have tests at all.

## Three kinds of round

Pick the kind per layer based on what excavation found — do not run all three everywhere.

**Settled** (the repo is consistent). Don't A/B it. State the rule back and ask one question: *why?* Capture the rationale, or record that there isn't one. Batch several of these into a single exchange; they're cheap.

**Contested** (the repo forks). Show both real examples from their own code, side by side, trimmed to the structural difference — file trees and import lines, not full source. Ask which one new code should look like, and what's wrong with the loser. Real forks beat synthetic ones: the user has history with both, and the answer is immediately actionable.

**Unwritten** (no evidence). Fall back to generated variants: two realistic ways the same new thing could be laid out in *their* idiom, differing along one layer only. Both must be competent — a strawman teaches nothing.

Ask the directional question, never the descriptive one. "Which of these should new code look like?" surfaces intent; "which of these is more common?" just re-measures what you already counted.

## Session setup

**Pick a reference change.** A realistic unit of work the team actually does — "add an endpoint," "add a settings screen," "add a background job." Use it as the running example across rounds, the way a single component anchors a design session.

**Pick a track.**

| Track | Rounds | Time | What you get |
|---|---|---|---|
| **Quick** | 6 | ~20 min | Excavation, plus one round on each contested layer. Settled layers confirmed in batch. A working doc for one package. |
| **Standard** | 12 | ~40 min | Two passes per layer, disconfirming probes, and the tooling-delegation pass. Rationale captured for every architectural rule. |
| **Deep** | 18 | ~70 min | Standard, plus prediction rounds, tie-breaker elicitation, and a generalization pass across a second package or service. |

Tracks are nested and sessions resume. If a conventions doc already exists, resume from its *Not yet established* section — but re-verify first: unlike design taste, the evidence base moves. Check the doc's recorded commit against `HEAD` and re-read anything that changed underneath it.

**Disconfirming probes (Standard and Deep):** once you think you see a rule, deliberately present the case that breaks it. If they've split modules aggressively twice, show a case where splitting clearly hurts. If they pick it, the rule has a boundary you hadn't found — write the boundary into the rule.

**Prediction rounds (Deep only):** state your prediction before they answer — "I think you'll want this colocated but the types hoisted, right?" Wrong predictions sharpen the rule; correct ones raise its confidence.

## The tooling-delegation pass

Before synthesis, walk the accumulated rules and ask of each: *could a tool enforce this?* Import ordering, formatting, file naming casing, dependency direction, forbidden imports, and max file length are all mechanically checkable, and most ecosystems have a plugin for each.

Anything mechanizable does not belong in the doc as prose. Prose rules that duplicate tooling are ignored, drift, and eventually contradict the tool. Move it to the config — offer to write the rule, and record it in the doc's delegation table as *enforced by X, don't restate*.

A good session usually *shrinks* the prose doc. That is a success, not a shortfall: what's left is exactly the judgment a linter can't make.

## The placement test is a round, not a victory lap

Before writing the doc, take the reference change and lay out every file it would produce under the derived rules — full paths, new directories, where tests land, which imports cross which boundary, what gets exported. Hand it over.

Integration surfaces contradictions that isolated rounds cannot. Rules that were individually fine collide: the colocation rule wants the type next to the component, the boundary rule wants it in the shared package, and nobody noticed until the file tree was on screen. Whatever comes back is first-class evidence — revise the rules, and record the collision as a tie-breaker.

## Guardrails against known failure modes

- **Document the intent, not the sediment.** The most common pattern is not the convention if the team is actively migrating away from it. For every rule drawn from frequency, ask whether it's what they want or what they're stuck with. A doc that faithfully describes the mess perpetuates the mess.
- **Separate force from confidence.** *Architectural* rules cost real money to violate and an agent should push back before breaking one. *Consistency* rules have no right answer and exist only so the codebase matches itself — an agent should comply silently and never argue. Collapsing these makes an agent either litigate file naming or quietly invert a dependency.
- **Rationale or it doesn't travel.** A rule with a reason can be extended to cases nobody anticipated; a rule without one can only be pattern-matched. Push for the reason on every architectural rule. If there isn't one, mark it `convention only` — that's honest, and it tells a future agent not to reason outward from it.
- **Distinguish convention from constraint.** Framework-imposed layout — `app/` routing, Rails' tree, a build tool's expectations — is not team taste. Record it separately, or an agent will eventually try to "clean it up."
- **Beware the one heroic module.** Most teams have a single beautifully organized package and twenty ordinary ones. Ask which is *representative* and which is *aspirational*; those are different answers, and the doc needs both.
- **"We" is often not unanimous.** Ask whether a teammate would state the rule the same way. Where the team genuinely disagrees, record the disagreement as an open question rather than encoding whoever happened to be in the room.
- **Don't let a refactor start.** Mid-session the user will spot something they hate and want it fixed. Log it as a migration item and keep going. The doc is the deliverable; the cleanup is a separate piece of work.
- **Silence is not consent.** If you propose a rule and the user doesn't object, that's weak evidence, not agreement — especially for rules you derived rather than observed. Mark it low confidence.
- **The repo is evidence, not authority.** Newest is not automatically right and most-common is not automatically right. Both are hypotheses you bring to the user for confirmation.
- **Don't fill gaps by derivation.** An unprobed layer goes in *Not yet established*. Where a derived guess is genuinely useful, label it derived and untested and keep it out of the rule list.

## Synthesis and the conventions doc

Confirm each rule with the user before writing it, then produce the doc:

```markdown
# Code Conventions — [repo/package]
Derived [date] · [track] session · from [repo] @ [commit sha]
**Evidence base: [packages/directories actually read].**

## Scope & calibration
Force, confidence, and scope are three separate axes; do not collapse them.
Force = architectural (push back before violating) or consistency
(comply silently, never argue). Confidence = how sure this is the intended
rule rather than an accident. Scope = where it was actually observed.
Name the parts of the tree that were never opened.

## Shape (one paragraph)
How the codebase is organized, in plain language, written to be acted on
rather than admired.

## Rules (per layer)
### Boundaries
- [rule] — *architectural · confidence: high · scope: [where] · why: [reason]*
### Placement · Naming · Decomposition · Interfaces · Tests
- ...

## Enforced by tooling — do not restate as prose
| Rule | Tool | Config |
|---|---|---|
| [rule] | [linter/formatter] | [file] |

## Migrations in flight
Old pattern → new pattern, and the rule for touching old code:
leave it, convert the whole file, or convert only what you touch.
Answer this explicitly — it's the question that comes up most in practice.

## Constraints, not conventions
Layout imposed by a framework or build tool. Do not "fix" these.

## Anti-patterns
What reliably goes wrong here, in the team's own language
("that folder is where things go to die"). More useful than the rules.

## Tie-breakers
When two rules conflict, which wins — including anything the
placement test surfaced.

## Not yet established — ask, don't extrapolate
Specific open questions, not categories: "where do cross-feature
types live?" not "types." Include every layer not run and every
package not read.

## How to use this doc
"Follow the Rules within their stated scope. Architectural rules:
raise a concern before violating one. Consistency rules: comply
without discussion. Outside the stated scope, or under Not yet
established, ask rather than extending a rule by analogy. Check
Anti-patterns before proposing; resolve conflicts via Tie-breakers."
```

Confidence is honest: a rule from a single Quick-track answer is `low`. Only prediction-confirmed or placement-tested rules reach `high`. A rule can be high-confidence and narrowly scoped — "definitely true, in the API package" is a normal and useful state.

## Where the doc lives

Unlike a taste doc, this belongs in the repo, not in a paste buffer: propose `docs/code-conventions.md` and a pointer to it from `CLAUDE.md` so every agent session picks it up without being asked. Offer to write both, plus any tooling config the delegation pass produced.

Close by noting that the doc records the commit it was derived from, so it can be re-verified later rather than trusted indefinitely. The *Not yet established* section is the agenda for the next session; the migrations section is the agenda for the next cleanup.

## Tone throughout

Move fast and stay concrete — file paths and import lines, not architecture vocabulary. Never praise a convention as correct; the goal is to write down what's true here, not to grade it. When the codebase contradicts itself, say so neutrally: that's the normal state of a working repo, and it's the reason the session is worth running.
