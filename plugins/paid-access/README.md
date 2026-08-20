# paid-access

Granting paid access is easy to write and easy to write wrong. This skill is the
other 90 %: the client that must never write the flag, the handle that must not be
reservable for free, the webhook that will be delivered twice, and the refund that
has to give the handle back.

Extracted from a shipped one-time-purchase upgrade — Stripe Checkout, a Firestore
mirror, a username as the paid product — but the rules are about payments and
scarcity, not about Stripe or Firestore.

## The rule

> Exactly one code path may write an entitlement — the one that has seen a settled
> payment. Every other actor reads it.

## What's in it

- **One writer.** Storage rules deny client writes; the API re-reads the stored flag
  at the moment of the write, never a value from the request or a token minted
  before the purchase.
- **Scarce names claimed at settlement.** Format validated at checkout, uniqueness
  resolved in the webhook — so an abandoned checkout can't squat the namespace.
  A name collision never fails the grant: the payment wins.
- **Uniqueness as a key collision**, resolved in the same transaction as the profile
  write, with the old mapping released on rename. Check-then-write loses races.
- **Settlement ≠ session completion.** Delayed-settlement methods complete `unpaid`
  and clear later; test cards hide this perfectly.
- **The buyer stamped on the PaymentIntent**, because dispute and refund events
  reference a charge and never the session — the one line you cannot backfill.
- **Idempotency as a claim on the event id**, created (not `set`) before processing
  and released if the handler throws, so genuine failures stay retryable.
- **Revocation as half the feature** — refund, dispute and fraud warning all revoke
  and release the name, idempotently, because they overlap.
- **Refund early, concede disputes** on a cheap product: dispute *rate* is what gets
  an account blocked.
- Plus: allowlisted public mirrors, explicit error reporting from catches that
  return a response, same-origin return URLs, and a way to verify the whole flow
  without ever paying.

## Install

Claude Code:

```
/plugin marketplace add aleexwong/claude-skills
/plugin install paid-access@alex-skills
```

Or copy `skills/paid-access/` into your project's `.claude/skills/` directory.

Any agent, via [skills.sh](https://skills.sh):

```
npx skills add aleexwong/claude-skills --skill paid-access
```

## Using it

It triggers on the work itself — adding checkout, writing or reviewing a webhook,
gating a feature behind a plan flag, letting users claim a handle. The checklist at
the end is meant to be run against a diff before it merges.
