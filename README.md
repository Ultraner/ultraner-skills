# Ultraner Agent Skills

Instruction packs for AI coding agents building payments across Africa with
[Ultraner](https://ultraner.com). Written for Claude Code, Cursor, Copilot, Windsurf and
anything else that can read a Markdown file.

Live in 14 markets: Benin, Cameroon, DR Congo, Gabon, Ivory Coast, Kenya, Mozambique, Republic of Congo, Rwanda, Senegal, Sierra Leone, Tanzania, Uganda and Zambia.

| Skill | What it teaches |
|---|---|
| [`ultraner-payments`](./skills/ultraner-payments/SKILL.md) | Charge an African mobile-money wallet, correctly, including the amount unit and the asynchronous result. |
| [`ultraner-payouts`](./skills/ultraner-payouts/SKILL.md) | Get money out of Ultraner, and the difference between settlement and disbursement. |
| [`ultraner-webhooks`](./skills/ultraner-webhooks/SKILL.md) | Receive and verify Ultraner webhooks, including the raw-body trap. |

## Using them

Clone the repository, or point your agent at a single file:

```bash
git clone https://github.com/Ultraner/ultraner-skills.git
```

Each skill is a self-contained `SKILL.md`. Paste one into a prompt, drop the
directory into your agent's skills folder, or fetch it over HTTP:

```
https://ultraner.com/ai/skills/ultraner-payments
```

The frontmatter declares the API and SDK versions the skill was written
against. **Check them before following it.** A skill that has drifted from
the API is worse than no skill, because an agent will follow it confidently.

## What these are for

Not marketing. Each one exists because there is a specific mistake that
costs an afternoon, and prose on a docs page was not preventing it:

- **Amount units.** Most African currencies have no minor unit. 5000 TZS is
  five thousand shillings, not fifty. Assuming cents charges a payer a
  hundred times the intended amount.
- **The payment is not finished when the call returns.** A mobile-money
  charge sends a prompt to a handset. A 200 means the request was accepted.
  Poll the status endpoint or listen for the webhook.
- **Provider codes are not brand names.** Mixx by Yas is still `Tigo` on
  the wire.
- **Two endpoints, two field names.** A charge takes `provider`; a payout
  takes `network`. Both take `account_number`, never `accountNumber`.
  Unknown keys are stripped rather than rejected, so a wrong name does not
  error: the recipient silently disappears and the call fails as a missing
  required field.
- **Money out needs two extra headers.** `X-Signature-Key` names the person
  authorising it and `Idempotency-Key` stops a retry paying twice.

## Generated, not written

These files are built from
[`lib/agent-skills.ts`](https://ultraner.com) in the main Ultraner repository, which
derives from the same data that drives the website, `llms.txt`, the AI
context packs and the OpenAPI. That is deliberate: a hand-maintained copy
would claim a market or a field name the API does not have, which is exactly
the failure these skills exist to prevent.

**Do not edit `SKILL.md` files here.** Change the source and re-run the
build. Edits made here are overwritten on the next generation.

## Also useful

- API reference: https://ultraner.com/docs
- OpenAPI: https://ultraner.com/openapi.json
- `llms.txt`: https://ultraner.com/llms.txt
- AI context packs: https://ultraner.com/ai
- Your framework: https://ultraner.com/frameworks
- SDKs: https://github.com/Ultraner
- MCP server: `npx @ultraner/mcp`

MIT licensed.
