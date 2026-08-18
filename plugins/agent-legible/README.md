# agent-legible

Most sites are built for a reader that runs JavaScript. Agents don't. This skill is
the ladder for fixing that, in order, from "an agent can read this page" to "an agent
calls your logic instead of guessing at it."

Extracted from a running-tools site that ships 245 Markdown mirrors, an `llms-full.txt`
of the whole corpus, edge content negotiation, and a free public MCP server exposing
the same math the site's calculators use.

## The two problems, which are different

**Not readable.** A client-rendered calculator returns an empty shell, so an assistant
asked a question your product answers exactly quotes a competitor's static table instead.
Fixed by Markdown mirrors, `Accept` negotiation, and — the rung almost nobody does —
computing real reference tables at build time from the same functions the live widget
uses, so a non-JS reader gets quotable numbers with your name on them.

**Not callable.** Reading gets you cited; being callable gets you used, on every
question, with correct numbers. Fixed by an MCP server over Streamable HTTP that reuses
the exact modules the site uses.

## Also covered

- The rule that must never break: browser navigations keep getting HTML. Half the
  assertions in your negotiation test should protect humans, not agents.
- An explicit AI-crawler policy in `robots.txt` — search fetchers, user-triggered
  fetchers, and training crawlers are three different decisions.
- What actually belongs in `llms.txt`, plus `llms-full.txt` and `llms-index.json`.
- Why your analytics report zero agent traffic by construction, and where to log instead.
- Distribution, because built and unlisted is built and unused.
- `curl` one-liners that verify every rung against production.

## Install

```
/plugin marketplace add aleexwong/claude-skills
/plugin install agent-legible@alex-skills
```

Or copy `skills/agent-legible/` into your project's `.claude/skills/` directory.
