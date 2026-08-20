# Changelog

## 0.1.0
- Initial release, extracted from a shipped one-time-purchase upgrade flow (Stripe Checkout + Firestore)
- `paid-access` — one writer for entitlements; scarce names claimed at settlement rather than reserved at intent; uniqueness by key collision in a transaction; settlement vs. session completion; buyer id stamped on the PaymentIntent; event-id claim for idempotency with release-on-failure; refund/dispute/fraud revocation that releases the name; refund-early-and-concede economics; allowlisted public mirror; explicit error reporting from swallowing catches; same-origin return URLs; and how to verify the whole flow without paying
