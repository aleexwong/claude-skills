> **This is a worked example, not a template.** It is one person's session output,
> published so you can see the shape of what `design-taste` produces. Do not paste
> this into your own sessions — the whole point of the skill is that your doc will
> disagree with this one. Live version with the nine rendered rounds:
> https://aleexwong.github.io/claude-skills/

# A Design Taste — v1.1

Generated 2026-08-07 · Quick track, 5 rounds + 1 bonus round on motion
**Entire evidence base: one component type — a data dashboard (stat cards + table).**

---

## Scope & calibration — read this before the principles

Every principle below was extracted from six A/B comparisons of the same dashboard. That is a small, narrow sample, and the confidence markers mean two different things that should not be collapsed:

- **Confidence** = how sure I am this preference is real and stable.
- **Scope** = where it has actually been tested.

Nothing here has been tested on a marketing page, a form, a settings screen, a document view, or a mobile-first layout. A principle marked *high confidence · dashboard only* means "he clearly meant this, about dashboards." It does not license applying it to a landing page.

Two principles are exceptions worth calling out, because he stated them as general convictions rather than picking them off a comparison: **color must carry meaning**, and **alignment is non-negotiable**. Those are craft rules and probably do travel. Everything else should be treated as dashboard-local until tested.

---

## Identity

Build interfaces that feel like an instrument panel over the user's own data, not a product being sold to them. Pack information tightly and trust the reader to handle it. Rank everything — one number owns the screen, the rest visibly defer, and anything that can't earn a rank goes behind an interaction rather than into a weaker tier. Use serif for the numbers and headings that carry consequence, so the data reads as a statement of fact rather than a status update. Color is a working part, never a finish: every hue must encode a category, a state, or a direction. Surfaces are rounded and sit on soft elevation, but softness buys depth only — never extra space. Motion is choreographed on arrival and instant on repeat. Align every label to a single vertical line; nothing here should look unpositioned.

---

## Principles

### Feel
- Dense over airy. Tight spacing, small type, high information-per-pixel. *(medium · dashboard only)*
- The surface should read as *the user's own information*, not a marketed product. Generosity of whitespace reads as sales copy. *(medium · dashboard only)*

### Layout & hierarchy
- One dominant metric, secondaries in a visibly different and smaller treatment. Never a row of equal-weight cards. *(medium · dashboard only)*
- Supporting breakdowns live one interaction deep, not on the primary surface. The headline states the fact; the reader asks for the justification. *(medium-high · dashboard only — see open question on interaction model)*
- Density means no wasted space and no unranked elements — not maximum information on screen. If something can't earn a rank, it goes behind an interaction rather than into a weaker tier. *(medium-high · dashboard only)*
- Cap nesting at two levels: container → card. Cards inside cards inside cards is the failure mode. *(medium · dashboard only)*
- Alignment is a hard requirement, not a polish step. Page title, card eyebrows, section labels, and the first table column all start on one vertical line. *(high · likely general — raised unprompted)*
- Disclosure must not reflow its neighbours. An expanding element that stretches sibling cards is a defect, not a style choice. *(high · dashboard only — caught unprompted)*

### Typography
- Serif for headings and primary numerals; sans, uppercase, letterspaced for utility labels and column heads. *(low-medium · dashboard only, financial content only)*
- Serif is chosen for weight of consequence, not for elegance. It should feel like a ledger entry. *(low-medium · financial content only)*
- Tabular numerals everywhere numbers stack. *(medium · likely general)*

### Color
- Color must have a job: category, status, or direction of change. Decorative color is rejected outright. *(high · likely general — stated as a rule, not picked from a comparison)*
- Multi-hue category coding is welcome — this is not a one-accent palette. *(medium · dashboard only)*
- Keep a print/no-color fallback legible; meaning should survive in grayscale via weight and position. *(medium · dashboard only)*

### Detail & texture
- Rounded corners (~12px) and soft, low shadows over hard hairline borders. Flat all-border surfaces read as one-dimensional. *(medium · dashboard only)*
- Elevation is flavor, applied sparingly, on real containers only. *(medium · dashboard only)*

### Motion
- Expressive over functional. Staggered entrances, a spring-like curve (~`cubic-bezier(.16,1,.3,1)`), hover lift on cards, sequenced reveals. Bare 140ms fades read as under-invested. *(medium · dashboard only)*
- Value counters stay under ~450ms. A number that takes a beat too long to settle becomes friction. *(medium-high · dashboard only)*
- Motion budget scales inversely with frequency. First paint and on-demand reveals can be choreographed; hover, filter, and re-render must be near-instant. *(inferred, untested)*
- Always honour `prefers-reduced-motion`. *(assumed, not his stated preference)*

### Disclosure & states
- Disclosure is a flip: the card turns over to a detail view in the same footprint. Drawers and expand-in-place bands both lost to it. *(medium · dashboard only)*
- The headline number must survive the transition untouched. Only the region beneath it turns over — flipping the number away defeats the point. *(high · stated as a condition)*
- Degrade in place while any of the data is still trustworthy. Mark what failed where it failed, keep the rest working, retry as a small inline link. *(medium-high · dashboard only)*
- Take over the screen the moment none of it is trustworthy. Total failure gets a stated headline, a plain-language cause, and one primary action — not a panel full of stale numbers. *(medium · dashboard only)*
- Skeletons over spinners for loading. Preserve the layout, don't announce the wait. *(medium · dashboard only)*
- Empty states stay in the panel: real zeros in the cards, one quiet line and an inline action where the data would be. No centered illustration-and-CTA block. *(medium · dashboard only)*
- Stale data is acceptable as a labelled patch over a failed region, and not acceptable as a whole-panel substitute. *(inferred from the split between partial and total failure)*

---

## Exemplars

- **The dense hierarchical dashboard.** A lead metric at 40px serif beside two demoted stats, over a table with 7px cell padding. Attention lands immediately and nothing is wasted.
- **Category-coded tags and status dots.** Four hues, each mapped to a real category, plus red/green for urgency. The color does actual work.
- **A headline number with its breakdown hidden behind a click.** The surface asserts; the reader interrogates.
- **Choreographed entrance, instant hover.** Arrival is worth staging. Repeated interaction is not.

## Anti-exemplars

- **"Feels like a SaaS."** Generous padding, big quiet numbers, lots of air. Something being sold rather than something owned. Strongest repulsion in the set.
- **"Boom, boom, boom" — three equal stat cards in a row.** Uniform weight makes the strip skippable; the eye actively discards it.
- **"Too vibed."** Labels whose left edges don't line up. Signals that nobody positioned the components.
- **The always-visible breakdown bar.** Explaining the headline before anyone asked. Over-determines the reading.
- **Monochrome-with-one-accent.** Flattens distinct things into a single undifferentiated statement.
- **Disclosure that stretches its siblings.** Opening one card should not resize the cards next to it.

## Tie-breakers

- **Density beats softness.** *(inferred)* Rounded corners and shadows apply to surfaces; they never justify loosening spacing.
- **Meaning beats decoration.** *(stated)* If a color, box, or shadow can't name its job, remove it.
- **Alignment beats everything.** *(inferred, strongly signalled)* When a layout idea would break the shared vertical line, change the idea.
- **Consequence beats friendliness.** *(inferred)* Where a choice makes data feel softer or more casual, take the harder-hitting option.
- **Assert, don't justify.** *(stated)* When a supporting element exists only to explain a primary one, move it behind an interaction.

---

## Not yet established — ask, don't extrapolate

These are holes, not omissions. There is no evidence either way, and guessing from the principles above will produce confident-sounding fabrication.

**Data visualization** — the largest gap, given this is a dashboard taste doc.
- Axis treatment: labelled or minimal, inside or outside the plot area?
- Gridlines: present, ghosted, or absent?
- Does "color must mean something" hold for series color, or do multi-series charts get a sequential/categorical ramp?
- Chart types he trusts vs. rejects. One data point only: he implicitly preferred a segmented bar over a donut, but that was my choice, not his.
- Do charts get the serif treatment for values, or does the utility-sans rule take over?

**The interaction model for "one level deep."** Settled — flip, with the number held fixed. What remains open: whether flip still holds for detail that doesn't fit the card footprint, and what the back face does on narrow screens.

**States.** Probed across loading, partial failure, total failure, and empty. What remains open: error copy tone (the takeover copy was mine), and whether retry is ever automatic.

**Typographic scale.** The sizes in the composite are mine, not chosen. The serif/sans split is his; the ratios are not.

**Breakpoint behavior.** The most likely place the whole system breaks. Density is the first casualty on a narrow viewport, and there is no rule for what gets dropped. *Derived prediction, untested:* since density means "nothing unranked," the narrow layout should shed rank-3 content entirely rather than shrink everything proportionally.

**Dark vs. light.** Never probed. All rounds were light-mode.

**Non-dashboard components.** Marketing pages, forms, settings, document views, mobile-first layouts. Zero evidence.

---

## How to use this doc

Paste this document into any session before asking for UI work.

Instruction to the model: follow the Principles within their stated scope. When the work falls outside that scope — a different component type, or anything under *Not yet established* — say so and ask, rather than extending a dashboard preference by analogy. Check Anti-exemplars before proposing; resolve conflicts via Tie-breakers. Treat low-confidence principles as defaults that explicit direction overrides.
