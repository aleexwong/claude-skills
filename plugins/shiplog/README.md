# shiplog

Your commit log is the only complete record of what you built, and nobody will ever read it.

This skill reads a range of commits and produces the three things that outlive them: a
user-facing **changelog entry**, a **build-in-public post**, and evidence-backed
**résumé bullets**. One pass over the diffs, three audiences.

It reads commit messages. It does not write them.

## Why not `git-cliff`

The mechanical changelog tools map `feat:` → "Features" and stop. They can do that
without judgment, which is why the output reads like a database dump.

The judgment is the job: deciding that fourteen commits over three weeks were *one
shipped thing*, that nine of them shipped nothing anyone outside the repo can perceive,
and that the claim the diff supports is smaller than the claim it could be stretched to
support.

## How it works

Six phases. Bound the range (feature arcs, not calendar months — and if there are no
tags, say so rather than inventing retroactive semver). Harvest cheaply, reading full
diffs only for the commits that anchor each cluster. Cluster by **file-touch map**,
which beats commit type and timestamps combined. Triage every cluster as *shipped*,
*infrastructure*, or *invisible*. Attach evidence. Emit.

## Install

Claude Code:

```
/plugin marketplace add aleexwong/claude-skills
/plugin install shiplog@alex-skills
```

Then run `/shiplog`, optionally with a range.

Or copy `skills/shiplog/` into your project's `.claude/skills/` directory.

## The counting rules

The failure mode for this kind of skill is inflation — a null check becoming "engineered
robust error handling", a refactor acquiring a performance percentage that was never
measured. So numbers may come from exactly three places: countable in the repo right now
(routes, config entries, `<loc>` tags, files), stated by the author in a commit or PR, or
computed from the diff.

Traffic, users, revenue, and conversion are not in git and are never derived. Every claim
carries a SHA in the working notes — the thing that lets you defend a bullet when someone
asks about it.

There's a word list too. *Architected, spearheaded, leveraged, robust, seamless,
comprehensive* — these show up when a claim is being stretched, so the skill prefers the
verb that's actually in the diff.

## Attribution

Most repos now contain agent-authored commits. The skill counts them and surfaces the
split, then writes bullets about the system and the decisions — which are yours — rather
than implying who typed what. Bullets that survive the interview they were written for.

## Worked example

Run against [TrainPace](https://github.com/aleexwong/trainpace): 471 commits, 627 days,
zero tags, no changelog. Output lives in that repo as
[`CHANGELOG.md`](https://github.com/aleexwong/trainpace/blob/main/CHANGELOG.md) and
[`docs/shiplog/`](https://github.com/aleexwong/trainpace/tree/main/docs/shiplog).

160 of the 471 commits describe cleanup, formatting, merges, or reverted work — invisible
to every audience. That number is the reason a commit log can't just be published.

## License

MIT
