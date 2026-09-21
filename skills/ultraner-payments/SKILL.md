---
name: "ultraner-payments"
description: "Charge an African mobile-money wallet, correctly, including the amount unit and the asynchronous result."
version: "1.0.0"
api_version: "v1"
sdks:
  - "@ultraner/node@0.2.0"
  - "@ultraner/mcp@0.3.0"
frameworks:
  - "any"
last_updated: "2026-09-20"
source: "https://ultraner.com/ai/skills/ultraner-payments"
---
# Accepting payments with Ultraner

Charge a mobile money wallet, card, bank account or PayPal. Live in 14 markets: Benin, Cameroon, DR Congo, Gabon, Ivory Coast, Kenya, Mozambique, Republic of Congo, Rwanda, Senegal, Sierra Leone, Tanzania, Uganda and Zambia.

## The two things that are usually got wrong

**1. The amount unit.** Most African currencies have no minor unit. TZS has no minor unit. Send whole numbers: 5000 means 5000 TZS, not 50.00. There are no cents to divide by.

Whole units today: XAF, XOF, RWF, TZS, UGX.
Two decimal places: CDF, KES, MZN, SLE, ZMW.

Never assume cents. Fetch https://ultraner.com/ai/context/currencies.json and read
`decimals` for the currency you are charging.

**2. The payment is not finished when the call returns.** A mobile-money
charge sends a prompt to the payer's handset. The response tells you the
request was accepted, not that the money arrived. Listen for the webhook, or
poll the status endpoint. Do not treat a 200 as a completed payment.

## Making a charge

```bash
curl https://api.ultraner.com/v1/payments/express/mno \
  -H "X-API-Key: $ULTRANER_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 5000,
    "currency": "TZS",
    "provider": "Vodacom",
    "account_number": "255700000000"
  }'
```

`provider` is the network's provider code, not its brand name. Get it from
https://ultraner.com/ai/context/countries.json: each country lists its networks with
the exact `providerCode` to send. For example Tanzania's M-Pesa is
`Vodacom`, and the rebranded Mixx by Yas is still `Tigo` on the wire.

`account_number` is the payer's phone in full international form with no plus.

## Checking the result

```bash
curl https://api.ultraner.com/v1/payments/express/status/{reference} \
  -H "X-API-Key: $ULTRANER_API_KEY"
```

Prefer the webhook. Polling is for reconciliation, not for the happy path.

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

- Do not send `Authorization: Bearer <api key>`. It will fail.
- Do not divide or multiply the amount by 100 for a zero-decimal currency.
- Do not tell the user the payment succeeded before the webhook says so.
- Do not hardcode a market's networks. They change; fetch the context pack.
