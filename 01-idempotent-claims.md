# A gift-card claim endpoint that survives retries

A user taps **Claim**, the request times out, and the app tries again. At about the same moment, another session tries to claim the same card. Those are two different problems: the first is a retry of one operation; the second is competition for one resource. The endpoint has to handle both.

This example covers claiming a card in **our own database**. Calling an external issuer to redeem a code is a separate workflow; that call cannot safely be folded into the transaction below.

## Give the request an identity

The client generates one random idempotency key for a logical claim and reuses it on retries. A new claim gets a new key. Scope the key to the authenticated user, and store which card the request meant to claim. Otherwise, someone could reuse a key with different inputs and receive a misleading old result.

```sql
CREATE TABLE gift_cards (
  id uuid PRIMARY KEY,
  owner_id uuid NOT NULL,
  state text NOT NULL CHECK (state IN ('available', 'claimed'))
);

CREATE TABLE claim_requests (
  user_id uuid NOT NULL,
  request_key uuid NOT NULL,
  card_id uuid NOT NULL,
  state text NOT NULL CHECK (state IN ('pending', 'claimed', 'rejected')),
  result_code text,
  PRIMARY KEY (user_id, request_key)
);
```

Here `result_code` is an application outcome such as `already_claimed`, **not** the gift-card PIN or redemption code. Never put secret card material in this table, routine logs, or an error response.

## Make the database arbitrate

The following is transaction pseudocode. The `UPDATE ... WHERE state = 'available'` is the decisive step: for a given card, only one transaction can change that row from available to claimed. The request row remembers the outcome so a retry can return it.

```text
BEGIN

INSERT INTO claim_requests(user_id, request_key, card_id, state)
VALUES (:user_id, :request_key, :card_id, 'pending')
ON CONFLICT DO NOTHING

SELECT card_id, state, result_code
FROM claim_requests
WHERE user_id = :user_id AND request_key = :request_key
FOR UPDATE

if stored card_id != :card_id:
    ROLLBACK; return 409 key_reused_with_different_card

if state != 'pending':
    COMMIT; return stored outcome

UPDATE gift_cards
SET state = 'claimed'
WHERE id = :card_id
  AND owner_id = :user_id
  AND state = 'available'
RETURNING id

if one row returned:
    UPDATE claim_requests SET state = 'claimed', result_code = 'claimed'
    WHERE user_id = :user_id AND request_key = :request_key
else:
    UPDATE claim_requests SET state = 'rejected', result_code = 'unavailable'
    WHERE user_id = :user_id AND request_key = :request_key

COMMIT; return stored outcome
```

In PostgreSQL, the unique primary key settles two inserts using the same request key. The row lock serializes reads of that request; the conditional card update settles competing keys for the same card. Keep all of these steps in one short transaction. A crash before commit rolls the request and card change back together. A crash after commit leaves a result the client can retrieve with the same key.

## What I would test

| Case | Expected result |
| --- | --- |
| Same key, same card, repeated after timeout | Same stored outcome |
| Same key, different card | Conflict; no second claim |
| Different keys, same available card, concurrent requests | One claim; one unavailable result |
| Database transaction fails before commit | No claim or cached result survives |
| Unauthorized user names someone else's card | No claim; response does not reveal card secrets |

If an issuer API must be called, I would use a durable operation state and an outbox or job, then reconcile uncertain issuer responses using that issuer's documented idempotency or lookup behavior. A database transaction cannot make a remote HTTP call atomic. That boundary is where many apparently tidy diagrams stop being useful.

**Reading behind this design:** PostgreSQL documentation on row-level locks and transaction isolation; Stripe's documentation on idempotent requests; OWASP's logging guidance.

