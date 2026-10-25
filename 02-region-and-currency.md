# Currency is not a redemption region

A card says “USD 25.” That tells us the denomination currency. It does not tell us where the card can be redeemed, which account region it supports, or whether the issuer has changed its rules. Treating currency as a shortcut for all three creates avoidable support cases.

I would keep at least these facts separate:

| Field | Meaning |
| --- | --- |
| `currency` | The card's denomination currency |
| `accountRegion` | Region configured on the recipient's platform account |
| `redemptionRegion` | Where the recipient expects to redeem it |
| `issuerRule` | The issuer or platform's current, product-specific restrictions |

The rule belongs to a specific product and should carry a source and a review date. A country with a matching currency is not proof of compatibility. Rules that are missing or old need review, not an optimistic “yes.”

## A three-outcome check

This small JavaScript example uses **fictional** rules. It is a model for validation, not a claim about any real issuer.

```js
function assessCard(card, recipient, rule) {
  if (!rule || !rule.reviewedAt || !rule.allowedAccountRegions ||
      !rule.allowedRedemptionRegions) {
    return { status: "unknown", reason: "rule_missing" };
  }

  if (!card.currency || !recipient.accountRegion || !recipient.redemptionRegion) {
    return { status: "unknown", reason: "input_missing" };
  }

  if (card.currency !== rule.currency) {
    return { status: "ineligible", reason: "currency_mismatch" };
  }

  if (!rule.allowedAccountRegions.includes(recipient.accountRegion)) {
    return { status: "ineligible", reason: "account_region_mismatch" };
  }

  if (!rule.allowedRedemptionRegions.includes(recipient.redemptionRegion)) {
    return { status: "ineligible", reason: "redemption_region_mismatch" };
  }

  return { status: "eligible", reason: "rules_matched" };
}

const fictionalRule = {
  currency: "USD",
  allowedAccountRegions: ["US"],
  allowedRedemptionRegions: ["US"],
  reviewedAt: "2026-10-25"
};

console.log(assessCard(
  { currency: "USD" },
  { accountRegion: "CA", redemptionRegion: "US" },
  fictionalRule
));
// { status: 'ineligible', reason: 'account_region_mismatch' }
```

The `unknown` result is intentional. It prevents a missing rule from silently becoming an approval. A production system should also decide how long a reviewed rule stays valid, store its source, and route unknown cases to a person or a current issuer check.

## The product consequence

Show the buyer the restriction **before** payment or transfer. Show the recipient the same restriction before they try to redeem. If a rule changes, retain the rule version used for the original decision so support can explain what happened.

The implementation is simple. Keeping the underlying rules current is the hard part. That is why I would build the data model around explicit evidence and an honest “unknown,” rather than trying to infer eligibility from the currency printed on a card.
