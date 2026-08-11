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
```

**Any agent (via [skills.sh](https://skills.sh) / `npx skills`):**

```
npx skills add aleexwong/claude-skills --skill design-taste
```

## Skills

| Plugin | What it does |
|---|---|
| [design-taste](plugins/design-taste) | Elicits your personal design taste through structured A/B rounds and produces a reusable taste doc. |

## License

MIT
