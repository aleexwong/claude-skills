---
name: shiplog
description: Turn a range of git commits into the artifacts that outlive them — a user-facing CHANGELOG entry, a build-in-public post, and evidence-backed résumé or portfolio bullets. Use this skill when the user wants to write a changelog or release notes, cut or backfill a release, summarize what shipped in a period, draft a "what I built" / build-in-public / launch post, needs portfolio or CV bullets grounded in real work, asks what they've actually shipped lately, or has a long commit history and nothing outside the repo to show for it. This skill reads commits; it does not write commit messages.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Shiplog

Turn a commit range into the three artifacts that compound. The log is the only complete record of what someone built, and it is invisible to every audience that matters — users, peers, people who might hire them. This skill does the translation.

## Core principle: a commit is evidence, a deliverable is a claim

Commits record what changed inside a repo. Every audience outside the repo cares about what became *possible*. Those are different sentences, and getting from one to the other is judgment, not formatting.

This is why the mechanical tools stop where they do. `git-cliff`, `release-please`, and `conventional-changelog` map `feat:` → "Features" and call it a changelog — they can do that without judgment, which is exactly why the output reads like a database dump. The work they skip is the work:

- deciding that 14 commits spread over three weeks were **one shipped thing**
- deciding that 9 of them shipped **nothing anyone outside the repo can perceive**
- deciding what claim the diff **actually supports**, which is nearly always smaller than what it could be stretched to support

Never emit an artifact whose structure mirrors the commit list. If the output has one bullet per commit, no translation happened.

## Three artifacts, three audiences, one pass

| Artifact | Audience | The question it answers |
|---|---|---|
| CHANGELOG entry | users | What can I do now that I couldn't before? |
| Build-in-public post | peers | How was this built, and what was actually hard? |
| Résumé / portfolio bullets | people deciding about you | What does this prove they can do? |

Same evidence, three compressions. Always produce them in one pass — the expensive step is reading diffs, and all three read the same diffs. Producing them separately means paying three times and getting three inconsistent stories.

Offer all three; let the user drop any they don't want. Don't ask which one first and re-harvest later.

## Phase 0 — Bound the range

Look for tags: `git tag --sort=-creatordate | head`. If tags exist, the range is the last tag to `HEAD`, and this phase is done.

**If there are no tags — the common case, and the one worth handling well — do not invent retroactive semver and present it as history.** The product was continuously deployed; there were no releases. Say that plainly in the artifact header and propose boundaries derived from the log itself:

```bash
git log --format='%h %ad %s' --date=short --no-merges | tail -r   # arcs, oldest first
git log --format='%ad' --date=short --reverse                     # for gap detection
```

Cut at **feature arcs** (a cluster of work on one area, then a move to another), not at calendar months. Calendar boundaries split shipped things in half. Dormant stretches are natural boundaries; so is the commit where the file-touch map moves to a new directory.

Propose the boundaries and get confirmation before harvesting. Six to twelve groupings covering a couple of years is right; forty is a diary.

## Phase 1 — Harvest cheaply, then selectively

Never `git show` every commit. A 400-commit range needs roughly a dozen full diffs. Work in this order:

```bash
git log --format='%h|%ad|%an|%s' --date=short --no-merges <range>   # the spine
git log --format='%an' <range> | sort | uniq -c | sort -rn          # authorship (see Phase 5)
git diff --stat <base>..<head> | tail -1                            # magnitude
git log --format='@%h %s' --name-only --no-merges <range>           # file-touch map — the clustering signal
```

The file-touch map is the highest-value command in this skill and the one that gets skipped. Read it before forming any opinion about what shipped.

Then read full diffs **only** for the one or two commits that anchor each cluster: the largest `--stat`, or the one whose subject names the feature. Everything else in the cluster is corroboration and can be read from subjects alone.

## Phase 2 — Cluster into shipped units

Group commits by what they built, using these signals in descending order of reliability:

1. **Shared file paths.** Fourteen commits touching `src/features/plan/` are one thing regardless of their subject lines. This signal is stronger than every other one combined.
2. **Merge commits and PR numbers** (`(#83)`). The author already did the clustering; inherit it.
3. **Time adjacency plus subject nouns.** Weak on its own — bursty solo repos interleave three features in one evening.
4. **Branch names** recovered from merge commit messages.

A cluster is a **shipped unit**: something a user could notice, or a distinct capability the codebase gained. Not a sprint, not a week, not a directory.

Name each cluster with the noun its user would use — "Training plan builder", not `claude/plan-prescriptiveness-user-freedom-suw4ew`, and not "plan module refactor".

## Phase 3 — Triage each cluster

Assign every cluster exactly one label:

- **Shipped** — someone outside the repo can do something new. Appears in all three artifacts.
- **Infrastructure** — no user-visible change, real engineering. **Omitted from the CHANGELOG, and frequently the strongest material for the post and the bullets.** Security hardening, CI, test coverage, build architecture, type safety, agent legibility. Users don't care; peers and interviewers care most.
- **Invisible** — lint fixes, import cleanup, formatting, dependency bumps, typo fixes, reverted work. Counted, never described.

Invisible is usually 40–60% of commits. **Report the count** — "312 of 471 commits are invisible to every audience" is honest, it's the reason the log can't be published raw, and users find it clarifying rather than discouraging.

Reverts cancel. If a feature shipped and was removed within the range, it did not ship — do not list it, and do not list its removal as a feature either.

## Phase 4 — Evidence and counting rules

**Every claim carries a SHA. If you cannot cite one, cut the claim.** Keep the SHAs in your working notes even when the final artifact won't display them; they are what lets the user defend a bullet in an interview.

Numbers may come from exactly three places:

1. **Countable in the repo right now** — routes in the router, entries in a config array, `<loc>` elements in `sitemap.xml`, files in a feature directory, specs in the test dir. Run the command and cite it in your notes.
2. **Stated by the author** in a commit body, PR description, or repo doc.
3. **Computed from the diff** — commits in the cluster, files changed, insertions.

Everything else is fabrication. Specifically forbidden without a source in the repo:

- performance percentages with no benchmark committed ("40% faster")
- users, traffic, signups, revenue, conversion, retention — **none of this is in git**
- "reduced X by Y%" where Y was inferred from the nature of the change
- team size, timelines, or scope the log doesn't show

If the user wants a traffic or revenue number in the post, ask for it, and mark it in your notes as user-supplied rather than derived. Never reverse-engineer a plausible-sounding metric.

**Word discipline.** *Architected, spearheaded, leveraged, robust, seamless, comprehensive, best-in-class, end-to-end* are inflation markers — they appear when a claim is being stretched past its evidence. Prefer the verb that's actually in the diff: added, replaced, moved, deleted, typed, cached, indexed, generated, validated. A precise small claim outperforms a vague large one with every audience, and it survives follow-up questions.

## Phase 5 — Attribution honesty

Run the authorship count from Phase 1. Modern repos have agent-authored commits — Claude, Copilot, bots, co-authored trailers — sometimes a large share.

Do not silently present that work as hand-written in a résumé bullet, and do not hide it either. Both are bad advice. The defensible framing is the one that's true: the user chose the problem, specified the change, reviewed it, integrated it, and shipped it under their name. That is real engineering work and it's what the bullet should describe — the system and the decisions, not the keystrokes.

Bullets about *what was built and why* survive an interview. Bullets implying *who typed it* do not, and the user is the one sitting in that room.

Surface the authorship split to the user, state the framing you're using, and let them adjust. Never quietly choose for them.

## Phase 6 — Emit

### CHANGELOG.md

Newest first. One entry per shipped unit, phrased as a capability. Infrastructure omitted or compressed into a single trailing line. No commit SHAs, no author names — this file is for users.

```markdown
# Changelog

[One-line note on how versions were assigned, if they were assigned retroactively.]

## [version] — YYYY-MM-DD
### Added
- [Capability, in the user's vocabulary, present tense.]
### Changed
- [What is different now, not the journey to get there.]
### Fixed
- [The user-visible symptom that is gone.]
```

Collapse the journey. Fifteen commits of "fix X", "refactor X", "fix X again" is one line: X works. Users want the end state; the process belongs in the post, if anywhere.

### The build-in-public post

Pick the **one** cluster in the range with genuine tension — a real constraint, a wrong first approach, a surprising limit. Write about that one. Structure:

1. What you set out to do, in one paragraph.
2. The constraint you hit — specific and technical, the part a peer would recognize.
3. What you tried that didn't work. **This is the part people read.** A post with no failed approach is a press release.
4. What you shipped, with the concrete detail that proves it's real.
5. What you'd do differently, or what's still unsolved.

**Match the user's voice.** Before drafting, sample their existing writing — blog posts in the repo, the README, docs, commit bodies. If there's nothing to sample, ask them to paste one paragraph they wrote. Never default to LinkedIn cadence: no rhetorical-question openers, no one-sentence-per-line, no "Here's what I learned 🧵".

### Résumé / portfolio bullets

Format: **what was built → the technical decision that made it work → the number, if a real one exists.**

Rank by what's hardest to fake — architecture and systems above features, features above fixes. Six to ten bullets covering the full range, not per release.

Bullets must not be recognizable paraphrases of commit subjects. If a bullet reads like its commit, translation failed — go back to the diff and describe what the code does.

## Guardrails against known failure modes

- **`feat:` does not mean user-visible.** In most repos `feat:` includes internal refactors, and some of the most user-visible work is filed under `fix:`. Type prefixes are a weak hint about intent, never a triage decision. Read the diff.
- **Don't narrate the journey in the changelog.** Users want the end state.
- **A post per release is a quota, and quotas produce filler.** If no cluster in the range has real tension, say so and skip the post rather than manufacturing a struggle.
- **Don't inflate a fix into an initiative.** A null check is a null check. The credibility cost of one stretched claim is paid by every other claim in the document.
- **Don't imply a cadence the dates don't support.** If there are two-month dormant stretches, the artifacts must not read as steady weekly shipping. Solo projects have gaps; pretending otherwise is the most easily checked lie in the document.
- **Don't dump SHAs into user-facing artifacts.** Keep them in working notes, and in the changelog only if the repo's convention already does that.
- **Ask before writing files.** Confirm paths before creating `CHANGELOG.md` or overwriting anything. If a changelog already exists, prepend — never rewrite entries the user already published.

## Tone

Write like an engineer telling a colleague what they did over a beer: specific, a bit deadpan, willing to say what didn't work. Not a launch announcement, not a LinkedIn post, not a recruiter's summary of someone else's work.

The value here is entirely in the compression being *honest*. A changelog that overstates gets caught by users the first time they open the app; a bullet that overstates gets caught in the interview it was written for. Undersell slightly and be exactly right — that's the version that holds up.
