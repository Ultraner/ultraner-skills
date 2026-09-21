---
name: "ultraner-payouts"
description: "Get money out of Ultraner, and the difference between settlement and disbursement."
version: "1.0.0"
api_version: "v1"
sdks:
  - "@ultraner/node@0.2.0"
  - "@ultraner/mcp@0.3.0"
frameworks:
  - "any"
last_updated: "2026-09-20"
source: "https://ultraner.com/ai/skills/ultraner-payouts"
---
# Paying money out with Ultraner

**Two different things are called a payout, and only one of them is limited.
Getting this wrong is the single most common mistake in this area.**

| | What it is | Where it works |
|---|---|---|
| **Settlement** | your balance to your own bank account, by wire | **Every country in the world** |
| **Disbursement** | your balance to a payer's mobile-money wallet | Tanzania today |

If a business is not in a disbursement market, it is **not** stuck. It settles
to its bank account like anyone else. Never tell a user they cannot get paid
because their market has no wallet payout rail yet.

## Disbursement

Live in Tanzania today; the other markets are being brought up one verified rail at a time. Not to be confused with settlement, which works everywhere.

Two headers beyond the usual:

```bash
curl https://api.ultraner.com/v1/disbursements \
  -H "X-API-Key: $ULTRANER_API_KEY" \
  -H "X-Signature-Key: $ULTRANER_SIGNATURE_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 5000,
    "currency": "TZS",
    "network": "Vodacom",
    "account_number": "255700000000"
  }'
```

A payout selects the rail with `network`, not `provider`: `provider`
belongs to the charge endpoint, and on a payout it is silently dropped and
`network` comes back as missing.

`X-Signature-Key` names the **person** authorising the withdrawal. An API key
identifies a business, not a person, and money leaving should always have a
name against it. It is created in the Developer console under API Keys >
Signatures, confirmed with that person's PIN, and re-checked against their
current role on every call: access removed is signing removed.

Quote first with `POST /v1/disbursements/quote` to see the fee before
committing.

## Settlement

```bash
curl https://api.ultraner.com/v1/settlements/quote \
  -H "X-API-Key: $ULTRANER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "amount": 1000000, "currency": "TZS" }'
```

The quote tells you what leaves the balance, the FX rate, the wire fee, what
lands and when. Statuses follow Stripe's: `pending`, `in_transit`, `paid`,
`failed`, `canceled`, with a webhook on each. A person reviews the wire
before it leaves, because a wire cannot be recalled.

## Authenticating

Send the API key in the `X-API-Key` header. **Not** `Authorization: Bearer`:
that header is for a dashboard session token and a `uk_` key is rejected there.

```
X-API-Key: uk_test_...
```

Keys are `uk_test_` or `uk_live_`. Build against a test key: the sandbox
simulates every rail and touches no real money. A test key on a live endpoint
is refused, and so is the reverse, so the two cannot be mixed up by accident.

## Idempotency

Every call that moves money requires an `Idempotency-Key` header when you
authenticate with an API key. Generate one unique value per payment you intend
to make, and reuse it when retrying.

```
Idempotency-Key: <uuid per intended payment>
```

Retrying with the same key returns the original response rather than making a
second payment. Reusing a key with a different body is refused. A failed
attempt frees the key, so a genuine retry works.

## Errors

Errors come back as `{ success: false, code, message }`. Branch on `code`,
not on the message text. The ones worth handling explicitly:

| Code | Meaning |
|---|---|
| `VALIDATION` | the request shape is wrong |
| `UNAUTHORIZED` | missing or invalid key, or a test key on a live route |
| `INSUFFICIENT_BALANCE` | not enough in the wallet |
| `DOMAIN_NOT_AUTHORIZED` | a live key used from a domain it is not approved for |
| `FEATURE_DISABLED` | the capability is switched off platform-wide, 503, retry later |


## Do not

- Do not say "payouts are only available in Tanzania" without saying which kind. Settlement is worldwide.
- Do not attempt a disbursement without a signature key. It returns 403.
- Do not retry a payout without the original `Idempotency-Key`.
- Do not assume a market that collects can also disburse. Check `disbursement` in https://ultraner.com/ai/context/countries.json.
