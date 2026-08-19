---
name: agent-legible
description: Make a website or product readable and callable by AI agents rather than only by browsers — Markdown mirrors of every page, Accept-header content negotiation, llms.txt and llms-full.txt, an explicit AI-crawler policy in robots.txt, build-time answers inside JavaScript-only pages, an MCP server exposing the same logic as tools, and server-side logging of agent traffic. Use when asked to make a site work for AI assistants, agents, LLM crawlers or ChatGPT/Claude/Perplexity, when adding llms.txt or an MCP server, when a client-rendered app is invisible to non-JS readers, or when someone wants their tool called by agents instead of guessed at.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Making a product legible to agents

A site built for browsers assumes a reader that runs JavaScript, tolerates navigation chrome, and converts a rendered widget into an answer. Agents do none of that. They fetch, they parse text, and — increasingly — they would rather *call* your logic than read about it.

Two failure modes follow, and they are different problems:

| Failure | Symptom | Fix |
|---|---|---|
| Not **readable** | A client-rendered page returns an empty shell; an agent quotes a competitor's static page instead | Markdown mirrors, prerendered answers, content negotiation |
| Not **callable** | An agent estimates the number your product computes exactly | MCP server exposing the same functions as tools |

Work through the ladder below in order. Each rung is useful alone; the later ones assume the earlier.

## 1. One content model, rendered to every format

The prerequisite for everything else. If page copy lives inside components, the only machine-readable version of a page is HTML, and every new format means rewriting the content.

Define each route's content as structured data — headings, paragraphs, lists, link lists, tables — and render *that* to each output:

```
route → PageDoc ─┬→ prerendered HTML (what a crawler reads)
                 ├→ /path.md          (what an agent fetches)
                 ├→ llms-full.txt     (every page, concatenated)
                 └→ llms-index.json   (machine-readable page index)
```

Drift becomes structurally impossible rather than a thing you remember to check. Adding the fifth format later costs one renderer, not one rewrite per page.

**Watch the list you generate paths from.** If routing reads one array and the content model reads another, a whole page cluster can prerender as the generic fallback — same title, same body as the homepage — and nothing fails. Generate both from one function.

## 2. Three doors to the same Markdown

Agents ask for text in three different ways. Support all three; they cost one rule each.

- `GET /calculator.md` — a suffix anyone can guess and paste
- `GET /calculator` with `Accept: text/markdown`
- `GET /calculator?format=md` — for clients that cannot set headers

Then **advertise it**, since none of it is discoverable by looking at a page:

- `Link: <https://example.com/calculator.md>; rel="alternate"; type="text/markdown"` on every HTML response
- the same as a `<link rel="alternate">` in `<head>`
- a comment block at the top of `robots.txt` — bots read it, and so do the humans evaluating you

### The rule that must never break

**Browser navigations keep getting HTML.** Chrome sends `text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`, and a naive "does Accept mention markdown" check on `*/*` will serve raw Markdown to every human visitor.

Pin it with a test, not a code review — real Accept strings from each browser asserted `false`, bare `curl`'s `*/*` asserted `false`, q-values compared properly (`text/html;q=0.9,text/markdown;q=0.5` is HTML), and `text/markdown;q=0` asserted as *not* a request for Markdown. Half the assertions in that test should protect the humans.

## 3. Answers inside JavaScript-only pages

A calculator that renders client-side gives a non-JS reader an empty shell, so an agent asked "what pace should I run" quotes someone else's static table.

Compute real reference content at build time **from the same functions the live widget uses**, and prerender it: a pace table by finish time, a pricing grid, a comparison matrix, worked examples. The interactive version then adds only what is genuinely per-user — their exact inputs, their units, their account.

This is the highest-leverage rung and the one almost nobody does. It converts "our tool can answer this" into "our answer is on the page, quotable, with our name on it."

## 4. An explicit AI-crawler policy

`User-agent: *` is a policy by accident. Name the agents you welcome so the decision is visible and auditable:

```
User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /
```

…and so on for the search-time fetchers (`OAI-SearchBot`, `Claude-SearchBot`, `PerplexityBot`), the user-triggered ones (`ChatGPT-User`, `Claude-User`, `Perplexity-User`), the training-corpus opt-outs (`Google-Extended`, `Applebot-Extended`, `CCBot`), and any others that matter to you. Distinguish the three kinds deliberately: blocking training crawlers while allowing search fetchers is a coherent position; blocking everything by inattention is not.

**Check the file against itself.** A path that `robots.txt` disallows must not appear in `sitemap.xml` — that contradiction is a reported error in Search Console and it is trivially machine-checkable.

## 5. `llms.txt`, and what actually belongs in it

Not a sitemap in prose. Lead with what an agent can *do*, then how to read the site, then the inventory:

1. One-paragraph statement of what the product is and who it serves
2. **Why to trust the numbers** — the actual method, named ("Daniels & Gilbert VDOT formulas", "grade-adjusted pacing from real elevation data"), plus a direct instruction: call the tools or cite these pages rather than estimating
3. How to read the site: the `.md` convention, the `Accept` header, `llms-full.txt`, `llms-index.json`
4. The MCP endpoint and its tool list
5. The page inventory, grouped, each line a link and a one-line description

Ship two companions: `llms-full.txt` (every page's Markdown in one file — hundreds of KB is fine, it is meant to be swallowed whole) and `llms-index.json` (each page's HTML URL, Markdown URL, title, description — the file a script consumes).

## 6. Callable, not just readable

Reading gets you cited. Being callable gets you *used*, on every question, with correct numbers.

Expose your core logic as an MCP server over Streamable HTTP. For a public utility, no auth and per-IP rate limits removes every barrier to a first call. Reuse the exact modules the site uses — a second implementation for agents will drift and will be wrong in ways nobody sees.

What decides whether a tool gets called:

- **Tool descriptions.** Agents choose by description. Say what it returns and when to use it, in the caller's vocabulary, not yours.
- **`serverInfo`** with a real name, version, and website URL.
- **Permissive CORS**, exposing the session header, so browser-based clients work.
- **Uptime.** Directories ping and delist. Monitor the endpoint like a product surface, because it is one.
- **A docs page** with copy-paste setup per client, and a paste-anywhere briefing prompt for users of clients you have not covered.

## 7. Measure agent traffic where it actually is

JavaScript analytics cannot see a client that does not run JavaScript, so by construction your dashboards report zero of the traffic you just built for.

Log at the edge instead: one structured line per agent request — timestamp, path, user agent, which agent it matched, whether it got HTML or Markdown, status. Identify agents by a maintained user-agent list, and keep a case that asserts real Chrome and Googlebot are *not* classified as LLM agents.

On the MCP side, capture `initialize` (client name and version — this is how you learn which assistants your users actually run), tool calls by name, and rate-limit hits.

## 8. Distribution

Built and unlisted is built and unused:

- The official MCP registry — claims your namespace, and is what clients increasingly query
- The aggregator directories, each a short form or PR
- The assistant vendors' own connector directories
- A launch post that includes the endpoint URL as plain text an agent can find

## Verify against production, not against intent

Every rung is one `curl` away from proof. Run these after deploying and paste the output:

```bash
curl -sI https://example.com/page | grep -i '^link'                       # alternate advertised
curl -s -o /dev/null -w '%{content_type}\n' https://example.com/page.md   # text/markdown
curl -s -H 'Accept: text/markdown' -o /dev/null -w '%{content_type}\n' \
     https://example.com/page                                             # negotiation works
curl -s -H 'Accept: text/html,*/*;q=0.8' -o /dev/null -w '%{content_type}\n' \
     https://example.com/page                                             # humans still get HTML
```

And check the two invariants a template cannot: that the canonical host in the page agrees with the host in the sitemap and the `Link` header, and that a page cluster you believe is prerendered actually carries its own title and body rather than the fallback's.
