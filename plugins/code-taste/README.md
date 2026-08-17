# code-taste

A skill that reads how your codebase is organized, then asks you only about the parts it can't tell.

Generated code lands in the wrong directory, invents a naming scheme, or reaches across a
boundary nobody wrote down. Not because the conventions don't exist — because they live in
people's heads and in the diff history. This skill excavates them and produces a
**conventions doc**: rules for boundaries, placement, naming, decomposition, interfaces, and
tests, kept in the repo and pointed at from `CLAUDE.md`.

## How it works

The repo has already answered most of the question. So the session starts by reading — the
directory shape, the newest modules, the lint and formatter configs, the churn hotspots, the
places two directories solve the same problem two different ways — and reports back what your
code says about itself before asking anything.

The expensive minutes go to the three places reading fails:

- **Contested** — the repo forks. You get both real examples side by side and pick which one
  new code should look like.
- **Unwritten** — no evidence yet, so the skill generates two competent variants in your idiom.
- **Unexplained** — the pattern is consistent but the reason is invisible. Rules without a
  rationale can only be pattern-matched, never extended to a case nobody anticipated.

Six layers, coarse to fine: boundaries, placement and colocation, naming, size and
decomposition, interfaces and data flow, tests and fixtures.

## Install

Claude Code:

```
/plugin marketplace add aleexwong/claude-skills
/plugin install code-taste@alex-skills
```

Then start a session with `/code-taste`.

Or copy `skills/code-taste/` into your project's `.claude/skills/` directory.

## Output

A conventions doc where every rule carries three separate axes: **force** (architectural — push
back before violating; or consistency — comply silently and never argue), **confidence**, and
**scope**. Collapsing those is what produces agents that litigate file naming while quietly
inverting a dependency.

Two sections matter more than they look:

- **Enforced by tooling** — anything a linter can check is moved into config rather than written
  as prose. A good session shrinks the doc; what's left is the judgment no tool can make.
- **Migrations in flight** — the old pattern, the new one, and an explicit answer to the question
  that actually comes up: when you touch legacy code, do you leave it, convert the file, or
  convert only the lines you touched?

Before the doc is written, the skill lays out every file a real change would produce under the
derived rules and hands you the tree. Rules that were fine individually collide there, where you
can see it.

The doc records the commit it was derived from, so it can be re-verified instead of trusted
indefinitely — unlike taste, this evidence base moves.

## Related

Pairs with [design-taste](../design-taste), which does the same job for visual design.

## License

MIT
