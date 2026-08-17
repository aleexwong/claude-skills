# grill-me

A skill that asks you the questions you'd rather skip, until the thing you want is
something someone could actually build.

You know roughly what you want. Roughly is where projects go to die — the gap gets
filled with guesses, and the guesses only surface as rework. This skill interrogates
the idea one pointed question at a time and turns your answers into a **brief**: one
page you can paste into any future session as the working contract.

## How it works

A question earns its place only if a different answer would change what gets built.
That single filter kills most clarifying questions, which is the point — the ones left
are the ones worth your time. Anything the repo, the files, or your earlier messages
already answer gets looked up instead of asked.

It searches where fog actually hides: the outcome underneath the request, why this
landed today, one real user in one real moment, the edges of scope, the immovable
constraints, what's allowed to break, and who decides it's done. Questions come with
their consequences attached, so you're choosing between outcomes rather than
answering trivia.

Three tracks — a ~3-question sanity check, a ~8-question standard grill, and a
~15-question deep pass that adds a pre-mortem and kill criteria for work that's
expensive to undo.

## Install

Claude Code:

```
/plugin marketplace add aleexwong/claude-skills
/plugin install grill-me@alex-skills
```

Then start with `/grill-me`, or just say "grill me on this."

Or copy `skills/grill-me/` into your project's `.claude/skills/` directory.

## Output

A brief that keeps three things apart, because conversations blur them and rework
comes from the blur:

- **Decided** — settled, with who made the call (including the ones you deferred to it)
- **Assumed** — defaults it's proceeding under, each with what would invalidate it
- **Open** — only what actually blocks, each with the cost of guessing wrong

Plus what's explicitly *not* in scope, which prevents more rework than the rest
combined.

Briefs are versioned and resumable. When the work contradicts an assumption, that's
the brief doing its job — bring it back and the next grill starts from `Open`.

## License

MIT
