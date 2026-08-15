# claude-skills

Agent skills for Claude Code, Claude.ai, and other agent harnesses.

## Install

**Claude Code:**

```
/plugin marketplace add aleexwong/claude-skills
```

Then install individual plugins:

```
/plugin install design-taste@alex-skills
/plugin install shiplog@alex-skills
```

**Any agent (via [skills.sh](https://skills.sh) / `npx skills`):**

```
npx skills add aleexwong/claude-skills --skill design-taste
npx skills add aleexwong/claude-skills --skill shiplog
```

## Skills

| Plugin | What it does |
|---|---|
| [design-taste](plugins/design-taste) | Elicits your personal design taste through structured A/B rounds and produces a reusable taste doc. |
| [shiplog](plugins/shiplog) | Turns a range of git commits into a changelog, a build-in-public post, and evidence-backed résumé bullets. |

## See it run

**[aleexwong.github.io/claude-skills](https://aleexwong.github.io/claude-skills/)** — a full
session record: the taste doc it produced, plus all nine rendered A/B rounds embedded live.
Rounds 6–8 are interactive, so motion, disclosure and error states can actually be judged.

It's a worked example, not a template — a different session lands somewhere else, and should.
Source: [`docs/`](docs).

## License

MIT
