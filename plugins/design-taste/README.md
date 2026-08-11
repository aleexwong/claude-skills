# design-taste

A skill that finds out what you actually like, instead of asking you.

Generated UIs tend to converge on the same defaults. This skill runs a structured
elicitation session — rendered A/B comparisons, one design layer at a time — and
turns your reactions into a **taste doc**: a portable markdown file of principles,
anti-exemplars, and tie-breaker rules you paste into future sessions to get designs
in your style.

## How it works

Taste is revealed, not recalled. Nobody accurately answers "what style do you like?",
so the skill never asks. It shows two competently-built variants of the same component,
differing along exactly one layer, and asks which one wins and what repels you about
the loser. Dislikes turn out to be more diagnostic than likes.

Six layers, coarse to fine: feel, layout and hierarchy, typography, color, detail and
texture, behavior and states. Optional domain modules for data visualization, forms,
and editorial work.

## Install

Claude Code:

```
/plugin marketplace add aleexwong/claude-skills
/plugin install design-taste@alex-skills
```

Then start a session with `/design-taste`.

Or copy `skills/design-taste/` into your project's `.claude/skills/` directory.

## Output

A versioned taste doc. Every principle carries both a confidence level and a scope —
where it was actually tested — so a preference formed on a dashboard doesn't silently
get applied to a landing page. Gaps are listed explicitly as questions to ask rather
than assumptions to make.

Sessions are resumable: paste an existing doc back in and the skill picks up from the
least-evidenced layer.

## License

MIT
