---
name: grill-me
description: Interrogate the user about a fuzzy idea or request — one pointed question at a time — until it becomes a brief that can actually be acted on, then write that brief down. Use this skill whenever the user says "grill me", "ask me questions until this is clear", "poke holes in this", "interview me about it", "help me figure out what I actually want", or hands over a vague-but-expensive request and would rather be questioned than have you guess. Also use it when the user wants a spec, brief, or requirements doc written from a half-formed idea, or says they know they want *something* but can't articulate it yet. Prefer this over silently making assumptions any time a large or hard-to-reverse piece of work rests on an under-specified ask.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Grill Me

The user wants something, and its shape is fuzzy. Interrogate them until it is a **brief**: one page that someone could act on without asking anything else — including you, ten minutes from now, with no memory of this conversation.

The brief separates three things that get fatally blurred when clarity lives only in a conversation: what was **decided**, what you are **assuming**, and what is still **open**. That separation is the actual deliverable. A conversation that produced clarity but wrote nothing down has produced clarity for exactly one person, for about a day.

## Core principle: ask only the questions whose answers change the work

Most clarifying questions are theater — they perform diligence without moving anything. A question earns its place only if you can name what you would do differently depending on the answer. If both answers lead to the same next move, you already have your answer; asking anyway spends the user's patience on a formality.

Two categories spend that patience for nothing:

- **Questions you could answer yourself.** The repo, the files, the earlier messages, the existing design all hold answers. Asking anyway signals you weren't paying attention, and every such question makes the next real one land softer.
- **Questions about mechanism.** How to structure it, what to name it, which library — that is your job. Handing it back as a question reads as either flattery or abdication.

Rank what survives by *how much the answer changes* × *how expensive it is to discover you were wrong later*. Ask the top one. Then re-rank, because the answer usually reshuffles the list.

## Before the first question, do your homework

Read what's already there. Come in with findings so questions carry evidence: "there's already a `reports/` module doing half of this — is the new thing replacing it or sitting beside it?" is worth five generic ones. It proves you looked, which is what buys you the right to keep asking.

## Where the fog hides

This is a map of places to search, not a checklist to walk. Walking it in order turns a grill into an intake form, and intake forms get skimmed answers. Go where the leverage is.

- **Outcome** — what is different in the world when this is done, and who notices? Requests arrive described as mechanism. The outcome underneath is usually unstated, and occasionally reachable a cheaper way.
- **Why now** — what happened that put this on the list *today*? This surfaces the real deadline and the real pain, both of which are routinely different from the stated ones.
- **One user, one moment** — a specific person doing a specific thing at a specific time. "Users" and "the team" are abstractions, and abstractions hide every interesting requirement.
- **Edges** — what is the nearest thing that is deliberately *not* in scope? The center of a request is always agreed on; disagreement lives at the boundary, so that's where to push.
- **Immovables** — stack, data, deadline, budget, people, rules, and the systems this has to fit inside. Ask what already exists that this must not break.
- **Failure appetite** — what is allowed to go wrong, and what is not survivable? Reversibility governs how much grilling is even warranted; a cheap, undoable thing deserves three questions, not fifteen.
- **Done** — the artifact, its form, who receives it, and who gets to say it's finished.

When a territory is already settled by what the user told you, note it as settled and move on. Asking someone to reconfirm what they just said is the fastest way to lose them.

## Question craft

- **Forced choice over open field.** People specify badly and react well. "Closer to A or B?" — with both options real and competently described — beats "what would you like?" every time.
- **Attach the consequence.** "If it's shareable links, we need an auth model — call it a day of work. If it's export-a-file, it's an afternoon." This converts trivia into a decision the user can actually feel, and they'll often answer instantly once they can see what it costs.
- **Scenario, not category.** "What should happen on errors?" gets a shrug. "It's 4pm, Maria uploads a 400MB CSV with a broken header row over hotel wifi — what does she see?" gets a specification.
- **Episodes, not averages.** "When did this last bite you, and what did you do about it?" beats "how often does this happen?" People recall incidents accurately and estimate frequencies terribly.
- **Numbers where adjectives appear.** Fast, scalable, soon, a lot — ask for a bound, and ask for the one at the edge: not the average day, the worst day they'd still call it working.
- **State a default instead of asking.** For anything you can reasonably decide: "I'll assume weekly, unless you say otherwise," then keep going. This is where your question budget comes from — spend it only on what nobody but the user knows.
- **Name a contradiction once.** When an answer collides with an earlier one, or with the stated goal, say so plainly and let the user resolve it. Once. Then take their resolution and move on; a grill that argues stops being a grill.

## When they don't know

"I don't know" is common and often the most honest thing said in the whole session. Escalate in this order:

1. **Offer candidates.** Two or three concrete options with what each implies. Recognition is far easier than recall, and a wrong option often triggers the sharpest correction of the session.
2. **Ask what would make it knowable.** Is this a decision waiting to be made, a fact someone else holds, or something only the work itself can reveal? Each of those has a different fix, and naming which one it is *is* progress.
3. **Convert it.** Choose a default, write down what would invalidate it, and move on.

An unknown with a default and a tripwire is not a hole — it's a managed risk. Holding the entire brief hostage to it is almost always the worse trade.

## Play it back before you write it

State the whole thing back concretely enough to be wrong: what you'll build, in what order, and what happens in the specific scenario you discussed. People correct much better than they specify, so give them something worth correcting.

The failure mode here is hedging. A vague playback earns "yeah, sounds right," which teaches you nothing and feels like agreement. If part of the playback is vague, that vagueness is yours, not theirs — resolve it before offering it back. Treat whatever comes back as first-class evidence; a correction at playback is worth more than three answers mid-grill, because now the user is reacting to the whole shape rather than one slice of it.

## Depth

| Track | Questions | When |
|---|---|---|
| **Sanity check** | ~3 | Small and reversible. Find the one ambiguity that would waste the whole effort, kill it, start. |
| **Standard** | ~8 | A real chunk of work. Sweep the territories that matter, play it back, write the brief. |
| **Deep** | ~15 | Expensive or hard to undo. Standard, plus a pre-mortem and kill criteria. |

Deep adds two moves that no direct question reaches:

- **Pre-mortem** — "It's three months on and this was a mistake. What happened?" People can criticize a future they could never have designed, and the answer is usually a constraint they'd never have volunteered.
- **Kill criteria** — "What would you have to see to stop doing this?" An idea nobody can imagine abandoning hasn't been thought about yet.

Say which track you're running and let the user shorten it. Running fifteen questions at someone who wanted three doesn't produce clarity; it produces short answers, which are worse than no answers because they look like data.

## Guardrails

- **One question at a time.** Batching feels efficient and isn't. People answer the easy ones and quietly drop the hard one — which was, reliably, the only one that mattered.
- **Don't ask the user to do your job.** Intent, priorities, constraints, and taste are theirs. Architecture, structure, mechanism, and naming are yours: state the call, invite the veto.
- **"Whatever you think" is not an answer.** It's fatigue or deference. Re-ask once, concretely, with the consequence attached. If it comes back the same, record it in the brief as *your* decision rather than their preference — so that in a month nobody misremembers who chose it.
- **Don't re-litigate what's decided.** If the user has already made a call, grill its edges, not the call itself. If you believe the call is a real mistake, say so once in a sentence or two, then work inside it.
- **Notice when grilling grows the project.** Every question is an invitation to add something. If the scope is larger coming out than going in, say so out loud. That may be exactly right, but it should be a decision, not a side effect of being asked a lot of questions.
- **Fuzzy is sometimes correct.** Some things genuinely should stay open until the work reveals them. Forcing a decision early manufactures precision, and manufactured precision reads later as though someone actually chose it. Park it as an assumption instead of inventing an answer.
- **Don't hide in the grill.** Questions feel productive, which makes them a comfortable place to avoid starting. Once you can begin the first chunk, begin — real work surfaces sharper questions than any interrogation will.

## The brief

Write it as a markdown artifact the user can keep.

```markdown
# Brief: [thing] — v1
Grilled [date] · [N] questions · [track]

## The ask, in one line
What is being built or done, in the user's own framing.

## Why now
The trigger and the real deadline, not the stated one.

## Done looks like
Observable and checkable — a state of the world, not a feeling.
Someone else should be able to hold up the result against this
line and say yes or no.

## Explicitly not this
The nearest excluded neighbours — the things a reasonable person
would assume were included. This section prevents more rework
than every other section combined.

## Constraints
Each with its source: "Postgres — already in prod", "ship by the
14th — board demo". A constraint with no source is a guess in a
costume, and gets treated as one later.

## Decided
Settled calls, with who made each. Include the ones you made when
the user deferred — attribution is what makes a decision
revisitable instead of load-bearing folklore.

## Assumed
The defaults you're proceeding under, each with what would
invalidate it: "assuming single-tenant; if a second customer
appears before launch, this changes the data model."

## Open
Only what actually blocks, each with the cost of guessing wrong.
If it doesn't block and it's cheap to change, it belongs in
Assumed, not here.

## First move
The smallest piece that gets built or checked first, and what it
would teach us.

## How to use this brief
Paste this in before asking for the work. Instruction to the
model: build against Done looks like, inside Constraints. Treat
Assumed as provisional — if you hit an invalidating condition,
stop and say so rather than working around it. Anything under
Open is not yours to decide quietly; ask.
```

Then say plainly what changed: the two or three things that were fuzzy at the start and are now nailed down, plus what's still open on purpose. That closing summary is what tells the user whether the time they just spent bought anything.

The brief is versioned and resumable. Work reveals what conversation can't, so when reality contradicts an assumption, that's the brief working correctly — the user can bring it back, and a second grill starts from `Open` rather than from nothing.

## Tone

Sharp, fast, and genuinely curious — a good editor, not an intake form. "Grill" describes the questions, not the user: they aren't being tested, the idea is. Never make someone defend a preference; ask them to describe consequences instead, which is easier to answer and more useful to hear. React with real interest when an answer surprises you, because a surprising answer means you found a place where you would have guessed wrong — which is the entire point.

If the answers are boring, the questions were.
