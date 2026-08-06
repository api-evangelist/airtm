---
name: Run a bulk payout with Airtm
description: Submit a batch of payouts, watch it through validation and processing, and reconcile per-payout results.
api: openapi/airtm-enterprise-v2-openapi.json
operations: [GeBalance, CreateBulkPayout, GetBulkPayout, ListBulkPayouts, GetPayout, ListPayouts]
generated: '2026-08-06'
method: generated
source: openapi/airtm-enterprise-v2-openapi.json + errors/airtm-error-codes.yml
---

# Run a bulk payout with Airtm

Bulk payouts are how you pay many recipients in one call. The batch has its own lifecycle,
independent of the individual payouts inside it.

## Before you start

- A bulk batch can be **rejected wholesale** on validation, so pre-validate your rows.
- The daily bulk ceiling is **$250,000**; exceeding it returns `415015`.
- Check `GeBalance` first — `415014` (insufficient balance) fails the whole batch, not one row.

## Steps

1. **De-duplicate your rows before submitting.** Airtm returns `415012` (duplicate payouts detected)
   and rejects the batch. Every row still needs a unique `code`.

2. **Submit with `CreateBulkPayout`.** A malformed file/payload returns `415011` (invalid file
   upload). The batch starts in `PENDING` while validation runs.

3. **Poll the batch with `GetBulkPayout`.** Batch statuses:
   - `PENDING` — received, validation in progress
   - `IN_PROGRESS` — individual payouts being processed
   - `COMPLETED` — all payouts processed. **Some may still have failed** — `COMPLETED` is a batch
     statement, not a per-payout guarantee.
   - `REJECTED` — the batch failed validation and nothing was paid
   Typical batches finish in 15–30 minutes depending on size.

4. **Reconcile per payout.** Once the batch is `COMPLETED`, list the payouts and check each one
   individually — `ListPayouts` accepts a `bulkPayoutId` query parameter, and `GetPayout` gives you
   the reason code on any that failed. Never report a batch as fully paid on batch status alone.

5. **Cancelling.** `415017` means the batch is already in progress and cannot be rejected; `415013`
   means it is not in `PENDING`; `415019` means it has not finished yet.

## Pagination

`ListBulkPayouts` and `ListPayouts` are cursor-paginated with `before`, `after` and `perPage`. Walk
the cursor; do not assume a single page.

## Do not

- Do not treat batch `COMPLETED` as "everyone got paid".
- Do not resubmit a whole batch to fix a few rows — send the failed rows as their own batch with
  their original codes.
