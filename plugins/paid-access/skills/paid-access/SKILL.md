---
name: paid-access
description: Turn a payment into access the buyer cannot grant themselves and you can take back — entitlement flags writable only by the code path that saw the money, scarce names (usernames, handles, slugs, seats) claimed at settlement instead of reserved at intent, uniqueness by key collision inside a transaction, webhook idempotency under retries, and refund/dispute/fraud revocation. Use when adding Stripe Checkout or any paid upgrade, writing or reviewing a payment webhook, gating a feature behind isPaid/isPro/plan, letting a user claim a unique handle or slug, exposing a public profile derived from private data, or handling chargebacks, refunds and delayed-settlement payments.
metadata:
  author: aleexwong
  version: "0.1.0"
---

# Paid access

Every paid feature has two halves, built months apart: the half that grants access when money arrives, and the half that takes it back when the money leaves. Ship one without the other and you get a namespace full of handles owned by charged-back cards.

One rule holds the whole thing together:

> **Exactly one code path may write an entitlement — the one that has seen a settled payment. Every other actor reads it: the client, the profile endpoint, the UI, you in the dashboard at 2 a.m.**

Everything below is that rule applied to a specific place where it usually leaks.

## 1. The client never writes the flag — check the stored value, not the sent one

Two layers, because either alone fails. Storage rules deny the write outright:

```
match /publicProfiles/{userId} {
  allow read: if resource.data.isPublic == true && resource.data.isPaid == true;
  allow write: if false;              // Admin SDK only
}
match /usernames/{username} {
  allow read: if true;                // public resolution
  allow write: if false;
}
```

And the API re-reads the entitlement from storage at the moment of the write:

```ts
// Claiming a username and going public are both paid features. Don't let an
// unpaid user set either — enforce server-side regardless of which UI calls in.
if (wantsUsername || wantsPublic) {
  const profile = await db.users.findById(user.id)
  if (profile?.isPaid !== true) return errorResponse('That requires an upgrade.', 403)
}
```

`findById`, not `body.isPaid`, and not a claim baked into a token minted before the purchase. The UI hides the paid control, so this endpoint is never exercised by an unpaid user during testing — the first caller to find it is someone with `curl`.

## 2. Claim scarce things at settlement, never at intent

If the desired handle is written when the user clicks *Buy*, the namespace is free to squat: start checkout, abandon it, keep the name. So the desire travels **through** checkout as metadata and is claimed on the other side.

| Moment | What it checks | Why there |
|---|---|---|
| Checkout creation | *format* (`/^[a-z0-9_-]{3,20}$/`) | Cheap, rejects garbage before taking money |
| Webhook, after settlement | *uniqueness* | The only moment that decides anything — minutes have passed |

And the collision case has exactly one correct answer: the payment wins.

```ts
if (username) {
  try {
    await db.users.update(userId, { username, isPaid: true, isPublic: true })
    return
  } catch (err) {
    if (!(err instanceof Error && err.message === 'USERNAME_TAKEN')) throw err
    // Taken while they were paying. Grant paid access anyway; the
    // "paid but no username yet" prompt lets them pick another.
  }
}
await grantPaidFlags(userId)
```

**Never let the optional half fail the grant.** A settled payment that produced no entitlement is the worst state in the system: the buyer has no product, no error, and no reason to think anything went wrong until they email you.

## 3. Uniqueness is a key collision, not a query

`SELECT … WHERE username = ?` followed by a write is check-then-act: two claims that read "free" a millisecond apart both write. Make the name itself the key, and resolve it in the same transaction as the profile update:

```ts
await adb.runTransaction(async (tx) => {
  const newMappingRef = adb.collection('usernames').doc(newUsername)   // name IS the id
  if (prevUsername !== newUsername) {
    const mappingSnap = await tx.get(newMappingRef)
    if (mappingSnap.exists && mappingSnap.data()?.uid !== id) throw new Error('USERNAME_TAKEN')
  }
  tx.update(userRef, userUpdate)
  tx.set(publicRef, { ...publicUpdate, username: newUsername }, { merge: true })
  tx.set(newMappingRef, { uid: id, createdAt: now })
  if (prevUsername && prevUsername !== newUsername) {
    tx.delete(adb.collection('usernames').doc(prevUsername))   // release, or the namespace only shrinks
  }
})
```

In SQL this is a unique index on `lower(name)` and letting the constraint violation be the answer. Same idea: the database decides, not your `if`.

Two things that quietly break it:

- **Normalize once and identically in every path.** Lowercase before the value becomes a key — in the availability check, the claim, *and* the lookup. One path that skips it makes `Alex` and `alex` two names for one handle, and the availability endpoint starts lying.
- **Delete the old mapping in the same transaction as the rename.** Otherwise every rename leaks a name that nobody can claim and nobody owns.

## 4. "Session completed" is not "money arrived"

Card payments settle instantly; bank debits, Klarna and friends complete the session as `unpaid` and settle days later — or never.

```ts
case 'checkout.session.completed': {
  if (session.payment_status !== 'paid') break   // wait for settlement
  await grantAccess(session)
  break
}
case 'checkout.session.async_payment_succeeded':  // it cleared, days later
  await grantAccess(event.data.object)
  break
```

Handling only `checkout.session.completed`, unconditionally, hands out product for payments that may never arrive. It works perfectly in testing, because test cards settle instantly.

## 5. Stamp the buyer onto the payment, not just the session

Refund, dispute and fraud-warning events reference a **charge or PaymentIntent — never the checkout session**. Without an identifier on the payment object, a chargeback cannot be mapped back to a user at all, and no later code change fixes charges already made.

```ts
payment_intent_data: { metadata: { userId: user.id } },
metadata: username ? { userId: user.id, username } : { userId: user.id },
```

Write that line on day one, before revocation exists. It is the cheapest line in the file and the only one you cannot backfill.

## 6. Idempotency is a claim on the event id

Providers retry deliveries, and retries overlap: a handler that creates a refund or closes a dispute is not safe to run twice. Claim the event before processing, with a create that *fails* on collision — `set()` would happily overwrite and defeat the whole thing:

```ts
const eventClaim = adminDb().collection('stripeEvents').doc(event.id)
try {
  await eventClaim.create({ type: event.type, receivedAt: new Date().toISOString() })
} catch (err) {
  if ((err as { code?: number }).code === 6 /* ALREADY_EXISTS */) {
    return successResponse({ received: true, duplicate: true })   // ack, don't reprocess
  }
  throw err
}
try {
  await handleEvent(event)
} catch (err) {
  await eventClaim.delete().catch(() => {})   // a genuine failure must stay retryable
  throw err
}
```

The release-on-failure half matters as much as the claim: without it, one transient error turns a retryable event into a permanently skipped one, and someone paid.

## 7. Revocation is half the feature, and it releases the name

Refunds, disputes and fraud warnings all end in the same place — and a revoked user must give the scarce thing back, or a charged-back handle is squatted forever:

```ts
await adb.runTransaction(async (tx) => {
  tx.set(userRef,   { isPaid: false, isPublic: false, username: '', updatedAt: now }, { merge: true })
  tx.set(publicRef, { isPaid: false, isPublic: false, username: '', updatedAt: now }, { merge: true })
  if (prevUsername) tx.delete(adb.collection('usernames').doc(prevUsername))
})
```

Idempotent on purpose: a refund and the dispute it spawns both fire, and the fraud-warning handler's own refund re-enters through `charge.refunded`. Overlapping revokes must be free.

## 8. On a cheap product, refund early and concede

| Event | Do | Why |
|---|---|---|
| `radar.early_fraud_warning.created` | Refund the charge immediately, then revoke | A refund issued *before* the dispute is filed usually stops it being filed — so it never counts toward your dispute rate |
| `charge.dispute.created` | Revoke, then close the dispute | Funds are already gone; contesting a single-digit-dollar charge costs more than it recovers, and *lost* disputes count worse than conceded ones |
| `charge.refunded` | Revoke | Catches dashboard and support refunds too |

The number that gets a payment account blocked is the dispute *rate*, not the disputed amount. Compare the product price against the chargeback fee plus the time to assemble evidence; below that line, conceding is the correct business answer and should be automatic in code rather than a decision someone makes while annoyed.

## 9. Public reads never touch the private document

Serve public pages from a mirror document, written only by the granting path, containing an **allowlisted tuple** of fields:

```ts
const PUBLIC_PROFILE_FIELDS = ['username', 'displayName', 'isPaid', 'isPublic', 'createdAt'] as const
```

Allowlist, not denylist: the private doc will grow an `email`, a `stripeCustomerId`, an internal note — and with a copy-everything-except mirror, each of those leaks on the day it is added, silently, to a public URL. With a fixed tuple, a new private field is invisible by default.

## 10. A swallowed error in payment code is silent lost revenue

A `catch` that returns an HTTP response means the framework's automatic error reporting never sees the exception. Report explicitly, in both the webhook and the checkout route:

```ts
// This catch swallows the error (returns a response), so onRequestError never
// fires — report explicitly. A silent webhook failure means someone paid and
// didn't get their username.
Sentry.captureException(error)
```

## 11. Client-supplied return URLs are an open redirect

`success_url` / `cancel_url` taken from the request body let a real, legitimate checkout page bounce the buyer to an attacker's site — mid-purchase, at maximum trust. Accept only your own origin, fall back to a safe default, and fail loudly if the origin is unconfigured rather than creating a session with a broken URL.

## 12. Verify the whole flow without paying

The grant path runs at most once per real user and never in CI. Drive it deliberately:

- **Mint a token server-side** (Admin SDK custom token → ID token) and call the real HTTP endpoints. No browser, no card.
- **Simulate settlement** by flipping the entitlement with admin credentials, then testing everything downstream of it.
- **Replay webhooks** with the provider's CLI (`stripe listen --forward-to …`, `stripe trigger …`) — including delivering the *same event id twice* and asserting the second delivery is acked and skipped.
- **Assert the negatives**, which is where the value is: unpaid `PUT` → 403, second user claiming a taken name → 400, no `Authorization` header → 401, revoked user's public URL → 404, and the mirror document containing no field outside the allowlist.

Before merging anything that touches money:

- [ ] Entitlement writable only by the settled-payment path — storage rules *and* the API check the stored value
- [ ] Scarce name claimed after settlement, uniqueness enforced by key collision in a transaction, old mapping released
- [ ] A name collision cannot fail the grant
- [ ] `payment_status` gated, delayed settlement handled
- [ ] Buyer id stamped on the PaymentIntent, not only the session
- [ ] Event id claimed before processing, released on handler failure
- [ ] Refund / dispute / fraud paths revoke *and* release, idempotently
- [ ] Public mirror carries an allowlisted field tuple
- [ ] Exceptions reported explicitly from every catch that returns a response
- [ ] Return URLs restricted to your own origin
