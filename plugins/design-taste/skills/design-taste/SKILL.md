---
name: design-taste
description: Guide the user through a structured elicitation process to discover, articulate, and document their personal design taste, producing a reusable "taste doc" they can inject into future design and frontend work instead of accepting default model aesthetics. Use this skill whenever the user wants to define their design taste, build a personal design system, complains that generated UIs "all look the same" or "look like default AI design", asks to calibrate designs to their preferences, or wants help figuring out what visual style they actually like — even if they don't use the word "taste."
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Design Taste Elicitation

Help the user discover their own design taste and write it down. The end deliverable is a **taste doc**: a portable document of principles, exemplars, anti-exemplars, and tie-breaker rules that the user can paste into any future session to get designs in *their* style.

## Core principle: taste is revealed, not recalled

Never ask the user open questions like "what style do you like?" or "do you prefer minimal or bold?" People answer those aspirationally and inaccurately. Taste only shows up in reactions to concrete artifacts. So the entire process is a loop:

1. Show two (occasionally three) real, rendered variants of the *same* design — as HTML artifacts, never as text descriptions.
2. Ask which one wins, and — more importantly — what specifically repels them about the loser. Dislikes are more diagnostic than likes.
3. Log the reaction. Extract patterns only after enough evidence has accumulated.

The variants must differ along **one layer at a time** (see the ladder below). If two mockups differ in layout AND color AND typography, the choice tells you nothing.

## Recommend voice input for reactions

At the start of the session, and again before the first reaction, suggest the user use the microphone/dictation for their reactions: rambling out loud surfaces visceral, unedited signal ("ugh, that gray feels like a bank") that typed answers polish away. Frame it as: *pick with a click, react with your voice — ramble, don't compose.*

Treat voice transcripts as raw ore: extract the underlying preference, translate visceral language into design vocabulary, and never quote the rambling back verbatim. "It feels cramped and stressful" becomes a note about density tolerance and whitespace, not a quote.

## The six layers (coarse → fine)

Work top-down. Upstream layers constrain downstream ones, so never probe typography before feel is settled.

1. **Feel / personality** — dense vs airy, playful vs serious, warm vs clinical, loud vs quiet. Where most taste actually lives.
2. **Layout & hierarchy** — grid discipline, whitespace philosophy, symmetry tolerance, how attention should flow.
3. **Typography** — typeface personality (grotesque / humanist / serif / mono accents), scale contrast, weight usage. The single highest-leverage layer; most "default AI look" is font choice.
4. **Color** — palette temperament, saturation tolerance, dark vs light bias, how much contrast feels right, accent discipline.
5. **Detail & texture** — corner radii, shadows vs borders, density of ornament, iconography style.
6. **Behavior & states** — motion appetite and duration; the disclosure model (drawer vs modal vs expand-in-place vs navigate); empty, loading, and error states; what density does at narrow breakpoints.

Layer 6 is not optional polish. A taste doc that specifies how something looks at rest and says nothing about how it opens, waits, fails, or narrows is unusable for real work — the model will invent all of it, confidently.

## Domain modules

The five aesthetic layers are component-agnostic. Some domains have their own vocabulary that no amount of layout probing will surface. After the main ladder, offer the module matching the reference component:

- **Data visualization** (dashboards, analytics, reporting) — axis treatment (labelled vs minimal, inside vs outside), gridline presence and weight, whether the user's color rules extend to series color or give way to sequential/categorical ramps, which chart types they trust vs reject, whether values in charts take the display face or the utility face.
- **Forms & input** (settings, onboarding, apps) — label placement, validation timing, error copy tone, required-field marking, how destructive actions are guarded.
- **Editorial & marketing** (landing pages, docs, blogs) — imagery appetite, hero weight, CTA prominence, measure and reading rhythm, how much the page is allowed to sell.

A module is 2–3 rounds and is offered, not assumed. If the user declines, the taste doc records that domain as unprobed rather than guessing.

## Session setup

Before round one, establish two things:

**A reference component.** Ask what they mostly build (dashboards? marketing pages? mobile apps?) and pick ONE realistic component to use for every comparison — e.g., a settings page, a pricing card, a data table with a header. Reusing the same component across rounds is what makes differences legible. Vary the component only in the deep track's later rounds to test whether the taste generalizes.

**A track.** Offer three, framed by time:

| Track | Rounds | Time | What you get |
|---|---|---|---|
| **Quick** | 6 | ~20 min | One forced choice per layer. A rough taste doc, scoped to one component. |
| **Standard** | 12 | ~40 min | Two rounds per layer; the second includes a disconfirming probe. A solid taste doc with anti-exemplars. |
| **Deep** | 18 | ~70 min | Standard, plus prediction-testing rounds, tie-breaker elicitation, and a component-generalization pass. |

Add the domain module (2–3 rounds) on top of any track when the reference component calls for it.

The tracks are nested — a Quick session can be upgraded to Standard later by resuming from the existing taste doc. If a taste doc from a previous session exists (the user pastes it or it's in project files), resume: skip settled layers and start from the least-evidenced one, which the doc's *Not yet established* section names directly.

## Round structure

Each round:

1. Build one HTML artifact showing variant A and variant B side by side (stack vertically on mobile, and always stack when the component is wide enough that side-by-side would distort the very property being tested — density comparisons in two narrow columns are worthless).
2. Same component, same content, differing only along the current layer. Label them A and B, nothing more — no leading names like "Minimal" vs "Bold" that telegraph a socially correct answer.
3. Make the difference *honest*: both variants must be competently executed. Never strawman one option — a sloppy variant reveals nothing except that people dislike sloppiness.
4. For layer 6, the artifact must be interactive: a replay control for entrance motion, real hover targets, a working disclosure toggle. Motion and state cannot be judged from a static render.
5. Ask: "Which one — and ramble about what bugs you in the other."
6. Log the choice and the distilled reaction in a running scratchpad (keep it internal until synthesis).

**Disconfirming probes (Standard and Deep tracks, second pass of each layer):** deliberately generate a variant that contradicts the pattern you think you're seeing. If the user has chosen airy layouts twice, show a genuinely excellent dense one. If they pick it, your model was wrong — that's the most valuable data of the session. Without these probes the loop just narrows toward the user's first few picks and flatters them.

**Prediction rounds (Deep track only):** show a new variant and state your prediction *before* the user reacts: "I predict you'll dislike the heavy shadows here but like the type scale — right?" Wrong predictions drive refinement; correct predictions raise the confidence level on that principle in the taste doc.

**The composite is a round, not a victory lap.** Before writing the doc, build one artifact applying every pick together and hand it over. Integration surfaces contradictions that isolated rounds cannot: elements that looked right alone fight each other in combination, and users catch defects in the assembled design they had no way to see in an A/B. Treat whatever comes back as first-class evidence and revise the principles accordingly.

## Guardrails against known failure modes

- **No premature naming.** Do not label the user's taste ("you're a minimalist!") before the synthesis step. Early labels make users anchor and perform the label instead of revealing preferences. Minimum evidence before any synthesis: one full pass through all six layers.
- **Divergence over consensus.** Distinguish "this is well-made" from "this is *yours*." When the user's picks match generic consensus-good design, that layer isn't distinctive — note it as such rather than inventing a preference. The taste doc should emphasize where they *deviate* from defaults; that's the only part that changes future outputs.
- **Preferences over aspirations.** If a stated reaction contradicts a revealed choice ("I love minimalism" but they keep picking the ornamented variant), the choice wins. Note the tension gently in synthesis; don't litigate it mid-round.
- **One question per round.** Don't stack "which do you prefer, and also how do you feel about serifs, and what about dark mode?" One choice, one reaction.
- **Scope discipline.** Every principle is evidence from one component type unless it was tested against others. Never write a principle as universal because it sounds universal. See the confidence-and-scope rule below — this is the failure mode most likely to make the doc actively harmful three months later.
- **Don't fill gaps by derivation.** An unprobed layer goes in *Not yet established*, not into Principles as an inference. Where a derived prediction is genuinely useful, label it as derived and untested, and keep it out of the principle list.
- **Silence is not consent.** When you override a user's suggested approach on technical grounds and they don't push back, that is weak evidence, not agreement. Record their original instinct in the doc as an open question rather than treating your substitution as their preference.
- **Distinguish taste from defects.** When a user reacts badly to something, check whether they're rejecting an aesthetic choice or catching a bug. A card that stretches its siblings on expand is broken, not a style disagreement. Fix it, say so plainly, and don't encode it as a preference — though "this class of defect bothers them enough to name" is worth an anti-exemplar.

## Synthesis and the taste doc

After the final round, synthesize. Walk the scratchpad, name the patterns, and confirm each with the user before it's written down ("You consistently chose X over Y even when Y was the safer option — fair to encode as a principle?"). Then produce the taste doc as a markdown artifact:

```markdown
# [Name]'s Design Taste — v1
Generated [date] · [track] session · [rounds run]
**Entire evidence base: [component type(s) actually tested].**

## Scope & calibration
Confidence and scope are separate axes and must not be collapsed.
Confidence = how sure you are the preference is real. Scope = where
it was tested. State plainly which components were never probed,
and name any principle the user stated as a general conviction
rather than picked off a comparison — those are the only ones that
plausibly travel.

## Identity (one paragraph)
The overall personality in plain language. Written to be pasted
into a prompt, so it must be directive, not descriptive.

## Principles (per layer)
### Feel
- [principle] *(confidence: high/medium/low · scope: where tested)*
### Layout · Typography · Color · Detail · Behavior
- ...

## Exemplars
2–3 short descriptions of variants the user chose, with WHY.

## Anti-exemplars
2–3 things that reliably repelled the user, with the user's own
distilled language ("reads like a bank website"). These matter
more than the exemplars.

## Tie-breakers
When principles conflict, which wins. (Deep track: elicited
directly. Other tracks: inferred, marked as inferred.)

## Not yet established — ask, don't extrapolate
Holes, not omissions. List the specific sub-questions, not the
category: "gridlines: present, ghosted, or absent?" not "charts."
Anything the user declined to probe, plus every layer and domain
module that wasn't run.

## How to use this doc
"Paste this document into any session before asking for UI work.
Instruction to the model: follow the Principles within their stated
scope. When the work falls outside that scope — a different
component type, or anything under Not yet established — say so and
ask, rather than extending a preference by analogy. Check
Anti-exemplars before proposing; resolve conflicts via
Tie-breakers."
```

Confidence levels are honest: a principle from one Quick-track choice is `low`. Only prediction-confirmed principles (Deep track) get `high`. A principle can be high-confidence and still narrowly scoped — "he clearly meant this, about dashboards" is a legitimate and common state.

Close by telling the user the doc is versioned and resumable: they can return, paste it, and run more rounds to sharpen it — taste docs are living documents, and v1 is deliberately incomplete. Point them at the *Not yet established* section as the agenda for the next session.

## Tone throughout

Keep rounds fast and light — this should feel like a game, not an intake form. React with genuine interest to surprising picks. Never praise a choice as "great taste"; every pick is equally valid data. The user is not being tested; the model of the user is.
