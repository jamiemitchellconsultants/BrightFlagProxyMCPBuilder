# Prompt 5 — Retrieve invoices approved for payment

Using the reusable contract, implement Stage 5: the read capability, end to end, against the fake
BrightFlag server. This is the first stage that performs outbound calls.

## Adapter

Implement a BrightFlag API adapter that:

- attaches the bearer token from the configured provider and nothing caller-supplied;
- calls `getAPBatchesByDateRange` with epoch-millisecond path segments when a window is given, and
  `getAPBatches` only when the configuration permits an unwindowed default;
- calls `getBatchInvoices` per released batch, bounded by a configured maximum batch fan-out;
- calls `getInvoiceSummaryList` with `includePreviousDrafts=false`, the reviewed status filter
  where it narrows the request usefully, and internal `paging.pageSize` and `paging.pageNumber`;
- enforces timeout, total-byte, page-count, and result-count ceilings, failing closed when a
  ceiling is reached;
- retries only idempotent GETs, only on transport failures and 429/503, with bounded jittered
  backoff and a total attempt cap; and
- maps `ErrorResponse` and every non-2xx status to the normalized BrightFlag error, preserving
  `errorCode` and `errorCategory` while discarding anything token-bearing.

Join batch release evidence to invoice summaries by `invoiceID`. An invoice released in a batch but
absent from the summary read is a typed reconciliation gap, reported and counted, never fabricated.

## Tool

Expose `brightflag_list_invoices_approved_for_payment` with:

- an optional window (`startedAfter`, `startedBefore`) subject to the Prompt 4 rules;
- an optional opaque `cursor`;
- an optional `maxResults` bounded by configuration; and
- no other arguments.

The result carries the approved invoices, the excluded-count with observed statuses, the
reconciliation-gap count, the effective window, and an optional `nextCursor`. It never carries a
page number, a BrightFlag URL, a raw response body, or a token.

The tool is read-only. Annotate it accordingly and prove the annotation cannot be the thing that
enforces it.

## Tests

Prove against the fake server:

- the exact outbound method, path, query string, headers, and call count for a single-page result;
- the same for a multi-page result driven by a returned cursor, including that resuming with the
  cursor issues no duplicate batch reads and returns no duplicate invoices;
- fan-out, page-count, byte, and result ceilings fail closed with a typed error;
- a 401 or 403 surfaces as an authorization failure without leaking the token;
- a 429 retries within the cap and then fails;
- the POST payment operation is never called by this tool;
- a reconciliation gap is reported rather than imputed; and
- responses containing unexpected extra fields are ignored safely rather than trusted.

## Acceptance criteria

- The read capability works end to end against the fake server through the MCP tool boundary.
- No caller argument can change origin, path, operation, headers, paging strategy, or the
  allow-list.
- Cursor round-trips are stable and tamper-evident.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the decision to expose cursor-based paging over
BrightFlag's page-number paging. Do not push unless requested.
