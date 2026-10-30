# Gift Card Systems Notes

Small engineering notes on the parts of digital gifting that are easy to underestimate: retries, concurrent requests, regional rules, and incomplete data.

These are design examples, not issuer policy or production-ready payment code. The sample gift cards and rules are fictional. I work on these questions while exploring digital gifting at CardNaro.

## Articles

1. [A gift-card claim endpoint that survives retries](01-idempotent-claims.md) — a database transaction that prevents a retry or a race from claiming one card twice.
2. [Currency is not a redemption region](02-region-and-currency.md) — a small rule evaluator that keeps currency and account region separate and treats missing rules as unknown.
3. [A balance should have a history](03-balance-history.md) — an event record and balance projection that make retries, concurrent redemptions, and reversals auditable.
