---
name: Send a single payout with Airtm
description: Verify a recipient, create a payout, commit it, and track it to a terminal state using the Airtm Enterprise API V2.
api: openapi/airtm-enterprise-v2-openapi.json
operations: [GetMe, GeBalance, GetUserByEmail, CreatePayout, CommitPayout, GetPayout, GetPayoutEvents, CancelPayout]
generated: '2026-08-06'
method: generated
source: openapi/airtm-enterprise-v2-openapi.json + conventions/airtm-conventions.yml + errors/airtm-error-codes.yml
---

# Send a single payout with Airtm

Airtm payouts are **two-step**: `CreatePayout` reserves the payout in `CREATED`, and `CommitPayout`
is what actually moves money. A payout left uncommitted never pays anyone.

## Before you start

- Base URL: `https://api.enterprise.airtm.com/v2` (sandbox: `https://api.stg.enterprise.airtm.com/v2`).
- Auth: HTTP Basic, username = API key, password = secret key. There is **no** bearer token here.
- Rate limit: 10 requests/second per API key. On `429`, back off exponentially.
- Never mix sandbox and production keys.

## Steps

1. **Confirm the account is live and funded.** Call `GetMe` for enterprise details and `GeBalance`
   for the available balance. If the balance will not cover the payout, stop — `CreatePayout` will
   fail with reason code `415016` (insufficient balance for payout).

2. **Verify the recipient first.** Call `GetUserByEmail` with the recipient's email. Check the user
   status is `ACTIVE`. This is the single highest-value pre-flight check: it turns the most common
   downstream failures (`415059` recipient not found, `415060` cannot receive payouts, `415061`
   restricted territory, `415010` deleted or banned, `415103` missing first/last name) into a
   decision you make before any money is reserved.
   - A recipient with no Airtm account is not an error. The payout will sit in `PENDING` and reason
     code `415912` ("waiting for recipient to create an Airtm account") applies. Unregistered-user
     payouts expire after 90 days by default.

3. **Create the payout.** Call `CreatePayout`. Supply your own unique `code` — it is the natural key
   and your de-duplication mechanism. Reusing one returns `415056` (payout code already exists), so
   generate it deterministically from your own ledger entry and treat `415056` as "already
   submitted", not as a failure.
   - Amount bounds: minimum $1 USDC for US and Puerto Rico recipients, $0.01 elsewhere. Below/above
     the window returns `415058` / `415057`, and `415035` for an amount that is too small.
   - Use `notes` for anything you will later want to search payouts by.

4. **Commit it.** Call `CommitPayout` with the payout id. Until this succeeds the recipient gets
   nothing. Committing a payout that already moved returns `415027` (already processed) — treat that
   as success, not as an error to retry.

5. **Track to a terminal state.** Prefer webhooks over polling: subscribe to `payout.created`,
   `payout.completed`, `payout.failed` and `payout.canceled` (see `asyncapi/airtm-webhooks.yml`).
   If you must poll, use `GetPayout` (or `GetPayoutByCode` with your own code) and
   `GetPayoutEvents` for the transition history.
   - Terminal states: `COMPLETED`, `CANCELED`, `FAILED`. Non-terminal: `CREATED`, `COMMITTED`,
     `PENDING`. Compare status strings case-insensitively.
   - Existing verified recipients typically complete in under 2 minutes.

6. **Cancel only while pending.** `CancelPayout` works on a pending payout; anything else returns
   `415055` (cannot cancel non-pending payout).

## Error handling

Every error is `application/json` shaped `{code, message, data}` where `code` is an Airtm reason
code — **not** RFC 9457. Read `errors/airtm-error-codes.yml` for the full registry. Do not retry
`415xxx` validation or recipient errors; they are deterministic. Retry only `429` and `415003`
(error reaching external service).

## Do not

- Do not treat `CreatePayout` as payment. It is a reservation.
- Do not generate a fresh `code` on retry — that creates a duplicate payout.
- Do not put credentials in client-side code; Basic auth here is a server-side credential.
