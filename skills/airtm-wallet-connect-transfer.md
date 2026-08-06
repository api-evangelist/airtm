---
name: Move funds in an Airtm user's wallet via OAuth (Connect)
description: Run the PKCE authorization-code flow, quote, then execute a scoped wallet:receive or wallet:send transaction with idempotency and per-transaction 2FA.
api: https://api.enterprise.airtm.com/api/connect/v1
operations: [GET /balance, GET /kyc, POST /quotes, POST /transactions, POST /transactions/{transactionId}/2fa, POST /transactions/{transactionId}/2fa/resend]
generated: '2026-08-06'
method: generated
source: https://api.enterprise.airtm.com/openapi.json (Wallet Resource API + OIDC guides) + scopes/airtm-scopes.yml + conventions/airtm-conventions.yml
---

# Move funds in an Airtm user's wallet via OAuth (Connect)

This is a **different API from the Enterprise payouts API**. There are no API keys here — the OAuth
access token and the scopes baked into it *are* the credential. Base URL:
`https://api.enterprise.airtm.com/api/connect/v1`.

> This API is documented in prose only. It has **no OpenAPI document**, so the operations above are
> paths, not `operationId`s, and no code generator can produce a client for it.

## 1. Get a token

Discovery: `https://api.enterprise.airtm.com/oidc/.well-known/openid-configuration`
(saved at `well-known/airtm-openid-configuration.json`).

- Authorization Code flow **requires PKCE with `code_challenge_method=S256`**. Requests without it
  are rejected with `invalid_request`.
- Request `openid` plus exactly the resource scope you need: `wallet:read`, `wallet:receive`,
  `wallet:send`, `kyc:status`.
- To get a refresh token you must request `offline_access` **and** send `prompt=consent`. Without
  `prompt=consent` the scope is silently dropped and you get no refresh token.
- Scopes are pre-approved per client. An unapproved scope returns `invalid_scope` before the user
  ever sees a screen.
- Access tokens are **opaque, not JWTs**. Validate by calling `POST /oidc/token/introspection` —
  do not attempt local signature verification. Only the `id_token` is signed (RS256).
- Access token TTL 1h; refresh token 30d and **rotates on every use**. Replaying a consumed refresh
  token is treated as theft: the entire grant is revoked and the user must re-consent.

## 2. Read state

- `GET /balance` — requires `wallet:read`. Returns USDC balance and deposit/withdrawal limits.
- `GET /kyc` — requires `kyc:status`.

## 3. Quote, then transact

Both directions are **priced by a quote first**, then executed by a transaction referencing the
locked quote.

- `POST /quotes` and `POST /transactions` **require an `Idempotency-Key` header** (a UUID).
  Omitting it is a `400`.
  - Same key + same body → the original transaction with HTTP `200` (not `201`). No duplicate transfer.
  - Same key + different body → `409`.
- Set `scope` in the request body to the action you want. The server verifies your token actually
  carries that exact scope before doing anything else; a mismatch is `403` regardless of the rest of
  the body.
- **`wallet:receive` (deposit).** The response carries `deposit` instructions: a pooled Airtm Stellar
  address, a `memo`, and the amount. You send USDC on Stellar with that memo; Airtm matches it and
  credits the user. The memo must fit the 28-byte `MEMO_TEXT` limit (`415120`).
- **`wallet:send` (withdrawal).** The response carries `destination` and `transactionMemo`. You must
  supply `destination.stellarAddress` or you get `415119`.

## 4. Confirm every send with 2FA

Since July 2026, **a newly created `wallet:send` transaction is returned as
`pending_user_confirmation` and is not dispatched until the user confirms a one-time code.**

- `POST /transactions/{transactionId}/2fa` with the code. Users with 2FA configured receive it on
  SMS/WhatsApp/authenticator; users without get it by email.
- `POST /transactions/{transactionId}/2fa/resend` re-issues it.
- These two endpoints deliberately do **not** take an `Idempotency-Key` — an already-confirmed send
  returns its current state, and resend is naturally repeatable subject to rate limiting.
- `415122` = not awaiting 2FA (wrong state or scope). `415123` = code invalid or expired.
  `415124` = 2FA settings could not be verified — **retryable**.

## 5. Watch the lifecycle

Webhook events (Svix): `oidc.transaction.created`, `oidc.transaction.pendingSettlement`,
`oidc.transaction.completed`, `oidc.transaction.failed`. On a rare terminal event where the quote
can no longer be loaded, `amountToSend`, `amountToReceive` and `fees` arrive as `null` so the status
change is never dropped — handle nulls. The REST read endpoints never return those as null.

## Error handling

`401 invalid_token` (missing/expired/revoked) and `403 insufficient_scope` apply on every endpoint,
both signalled in `WWW-Authenticate`. Quote/transaction reason codes are `415115`–`415124`; see
`errors/airtm-error-codes.yml`. Two conflicts return a `409` with a plain `message` and **no** reason
code: `partnerReference already in use`, and `Idempotency key already associated to another
transaction`.
