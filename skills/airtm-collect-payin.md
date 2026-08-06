---
name: Collect a payment with an Airtm payin
description: Create a hosted-checkout payin, redirect the payer, and confirm the outcome via webhooks.
api: openapi/airtm-enterprise-v2-openapi.json
operations: [CreatePayin, GetPayinById, GetPayinByCode, ListPayins, CancelPayin]
generated: '2026-08-06'
method: generated
source: openapi/airtm-enterprise-v2-openapi.json + components/airtm-components.yml + errors/airtm-error-codes.yml
---

# Collect a payment with an Airtm payin

A payin collects money **from** a user into your enterprise balance, through an Airtm-hosted
checkout page.

## Steps

1. **Create the payin** with `CreatePayin`. Supply:
   - a unique `code` — reusing one returns `415029` (payin code already exists)
   - `items[]` with description/amount/quantity
   - `cancelUri`, `failureUri` and `callbackUri` so you control where the payer lands on each outcome

2. **Redirect the payer to `confirmationUri`** from the response. This is Airtm's hosted checkout;
   there is no JavaScript element library to embed and no card fields for you to render.

3. **Wait for the outcome on a webhook**, not on the redirect. Subscribe to `payin.created`,
   `payin.confirmed`, `payin.failed` and `payin.canceled`. Payin statuses:
   - `CREATED` — awaiting user action (non-terminal)
   - `PROCESSING` — user initiated, in flight (non-terminal)
   - `CONFIRMED` — succeeded (terminal)
   - `CANCELED` — user cancelled (terminal)
   - `FAILED` — processing failed (terminal)
   Confirmation is immediate once the user confirms.

4. **Read back** with `GetPayinById` or `GetPayinByCode` (your own code) if you need to reconcile
   without waiting for a webhook. `ListPayins` is cursor-paginated (`before`/`after`/`perPage`).

5. **Cancel** with `CancelPayin` only while the payin is still open — otherwise `415030` (cannot
   confirm or cancel) or `415032` (already completed).

## Constraints

- **Payins expire after 15 days.** Do not hold a `confirmationUri` longer than that.
- **US users cannot make payins** — `415053`. Route US payers to an alternative method.

## Do not

- Do not confirm the sale on the browser redirect alone; the redirect is not proof of settlement.
- Do not poll aggressively — you have 10 requests/second per key across the whole API.
