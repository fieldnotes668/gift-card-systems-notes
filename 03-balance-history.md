# A balance should have a history

A stored-value card often appears in a UI as one number. That number is useful, but it is a poor explanation of *why* the balance changed. When an activation, redemption, reversal, or correction arrives late—or arrives twice—a single mutable field leaves little to investigate.

The design I prefer keeps an append-only event record and a balance projection in the same database transaction. The projection makes reads cheap; the event record makes every accepted change traceable. This example uses fictional cards and integer minor units, not real payment credentials.

## Two tables, two jobs

```sql
CREATE TABLE card_balances (
  card_id text PRIMARY KEY,
  balance_minor bigint NOT NULL CHECK (balance_minor >= 0)
);

CREATE TABLE card_events (
  card_id text NOT NULL,
  operation_id text NOT NULL,
  event_type text NOT NULL,
  delta_minor bigint NOT NULL,
  balance_after_minor bigint NOT NULL CHECK (balance_after_minor >= 0),
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (card_id, operation_id)
);
```

`operation_id` identifies the business action, not the HTTP request. A client retry must reuse it. A genuinely new redemption needs a new one. The event table's unique key makes the retry visible as the same action rather than another debit.

The balance is a projection, not an independent source of truth. Do not edit it through a separate admin path that leaves no event. If a correction is needed, record a correction event with its reason and operator reference.

## Make the debit and the evidence atomic

For a redemption of `amount_minor`, begin a transaction and lock the card's balance row:

```sql
SELECT balance_minor
FROM card_balances
WHERE card_id = :card_id
FOR UPDATE;
```

While holding that lock, look up `(card_id, operation_id)` in `card_events`. If it exists, return its stored `balance_after_minor`; do not subtract again. Otherwise, check that the amount is positive and run:

```sql
UPDATE card_balances
SET balance_minor = balance_minor - :amount_minor
WHERE card_id = :card_id
  AND balance_minor >= :amount_minor
RETURNING balance_minor;
```

If no row is returned, roll the transaction back. If a row is returned, insert a `redemption` event with the negative delta and the returned balance, then commit. The lock serializes concurrent redemptions for one card. Two different operations cannot both spend the same remaining funds.

Two workers trying the *same* `operation_id` also pass through the same lock. The second worker sees the committed event and returns the prior result. If the first transaction rolls back, the second may proceed normally. Never convert a uniqueness error into an unexplained success.

## A ledger is only useful when it reconciles

Periodically compute the sum of events for each card and compare it with the projection. A mismatch is an incident, not a number to silently overwrite. Also record the source of every event, the currency and unit convention, and a reference to the external order where relevant. Keep sensitive redemption codes out of this log.

The event type matters too. A reversal is not a deletion of the original debit; it is a separate positive event linked to the debit it reverses. That preserves the sequence a support agent or auditor needs to understand.

## Tests that earn trust

- Retry the same operation before and after a simulated timeout. The balance must move once.
- Run two different redemptions concurrently against a balance that covers only one. One succeeds; the other fails without an event.
- Force a database error between event insert and balance update. Both writes roll back.
- Reverse an operation once, then retry the reversal. The credit happens once.
- Compare the event sum with the balance projection after each scenario.

The visible balance can remain one number. Underneath, it should be a number with a defensible history.
