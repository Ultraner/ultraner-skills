---
name: "ultraner-webhooks"
description: "Receive and verify Ultraner webhooks, including the raw-body trap."
version: "1.0.0"
api_version: "v1"
sdks:
  - "@ultraner/node@0.2.0"
  - "@ultraner/mcp@0.3.0"
frameworks:
  - "express"
  - "next.js"
last_updated: "2026-09-20"
source: "https://ultraner.com/ai/skills/ultraner-webhooks"
---
# Receiving Ultraner webhooks

Mobile-money payments finish asynchronously, so the webhook is not optional:
it is how you learn a payment succeeded.

## Verify before you trust

Every delivery is signed with HMAC over the **raw request body**. Verify it
before doing anything with the payload.

**The trap:** most frameworks parse JSON before your handler runs, and
re-serialising the parsed object does not reproduce the bytes that were
signed. Key order and whitespace change, and the signature will never match.
Capture the raw body.

```js
// Express: raw body for this route only, JSON everywhere else.
app.post('/webhooks/ultraner',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const signature = req.header('X-Ultraner-Signature');
    const expected = crypto
      .createHmac('sha256', process.env.ULTRANER_WEBHOOK_SECRET)
      .update(req.body)              // the Buffer, not JSON.stringify(...)
      .digest('hex');

    // Constant-time: a plain === leaks the signature one byte at a time.
    const valid = crypto.timingSafeEqual(
      Buffer.from(signature, 'hex'),
      Buffer.from(expected, 'hex'),
    );
    if (!valid) return res.status(400).end();

    const event = JSON.parse(req.body.toString());
    // Acknowledge fast, work afterwards. A slow handler gets retried.
    res.status(200).end();
    void handle(event);
  });
```

In Next.js App Router, `await req.text()` gives you the raw body; do not use
`await req.json()` for the verification step.

## Handling

- **Acknowledge with 2xx quickly.** Anything else is retried, so slow work
  belongs after the response, not before it.
- **Expect duplicates.** Retries and races mean the same event can arrive
  twice. Key your processing on the event id and make it idempotent.
- **Do not depend on ordering.** A `success` can arrive before the event that
  logically precedes it.
- **Test and live are separate.** A test-mode secret can never validate a
  live payload, which is deliberate: it stops a sandbox response being paired
  with a real webhook.

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

- Do not verify against a re-serialised body.
- Do not compare signatures with `===`.
- Do not do the work before responding.
- Do not treat a webhook as a one-time delivery.
