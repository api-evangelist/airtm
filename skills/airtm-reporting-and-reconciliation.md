---
name: Generate and download an Airtm activity report
description: Request a report, poll it to completion, and fetch the time-limited download URL.
api: openapi/airtm-enterprise-v2-openapi.json
operations: [GenerateReport, ListReports, GetReport, GetReportFile, ListDeposits, GetDeposit, ListWithdrawals]
generated: '2026-08-06'
method: generated
source: openapi/airtm-enterprise-v2-openapi.json + lifecycle/airtm-lifecycle.yml + errors/airtm-error-codes.yml
---

# Generate and download an Airtm activity report

Reports are **asynchronous**. You request one, wait, then fetch a signed file URL.

## Steps

1. **Request it** with `GenerateReport`. Supply the date range and the timezone — timezone support
   was added across all reporting in May 2025, and omitting it is the usual cause of off-by-one-day
   reconciliation disputes.

2. **Poll with `GetReport`** (or `ListReports`, cursor-paginated via `before`/`after`/`perPage`).
   Generation takes 5–15 minutes depending on data volume.
   - `415042` statement not found — verify the id
   - `415043` statement not pending — it is already past that stage
   - `415044` statement not completed — keep waiting

3. **Download with `GetReportFile`.** The URL it returns **expires after 1 hour**. Fetch and persist
   the file immediately; do not store the URL for later.

4. **Cross-check money movement** with `ListDeposits` / `GetDeposit` and `ListWithdrawals` for the
   same window, and `ListPayouts` / `ListPayins` filtered by status, rather than reconciling from a
   single report alone.

## Do not

- Do not hand a report URL to a downstream system that may consume it more than an hour later.
- Do not generate reports in a tight loop — 10 requests/second per key covers the entire API.
