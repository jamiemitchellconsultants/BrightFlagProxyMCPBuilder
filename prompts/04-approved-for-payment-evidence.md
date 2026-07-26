# Prompt 4 — Define the approved-for-payment evidence rule

Using the reusable contract, implement Stage 4: the deterministic, network-free rule that decides
whether an invoice is approved for payment, plus validation of the query that selects candidates.

## The evidence rule

An invoice is approved for payment only when both hold:

1. **Release evidence.** Its `invoiceID` appears in an accounts-payable batch returned by
   `getAPBatches` or `getAPBatchesByDateRange` within the requested window. The batch is the record
   that BrightFlag released the invoice to accounts payable.
2. **Status corroboration.** Its `InvoiceSummaryAPI.invoiceStatus` is in the reviewed
   approved-status allow-list configured in Prompt 3.

Neither signal is sufficient alone. Record which signals were observed on every returned invoice so
a reconciliation reviewer can see the basis of the decision.

Because BrightFlag invoice-status vocabularies are configured per customer, the allow-list is data,
not code. Ship no default allow-list. Startup must fail when the allow-list is empty, and each
entry must be compared as an exact, case-sensitive, trimmed string — never a substring, prefix, or
case-insensitive guess at words like "approved".

An invoice that is batched but whose status falls outside the allow-list is not returned. Report it
in a bounded, separately named excluded-count with the observed status so a misconfigured
allow-list is visible rather than silent.

## Draft revisions

`invoiceID` identifies an invoice revision; `invoiceGroupId` identifies the logical invoice. Send
`includePreviousDrafts=false` and return at most one row per `invoiceGroupId`, preferring the
revision the batch released. If a batch releases two revisions of the same logical invoice, fail
that row with a typed conflict rather than choosing one silently.

## Query validation

Validate `ApprovedInvoiceQuery` before any outbound call:

- the lookback window defaults to 7 days and may not exceed 31 days;
- explicit start and end instants must be ordered, must not be in the future beyond a small
  configured clock skew, and must convert to the epoch-millisecond strings the batch operation
  requires;
- a supplied cursor must be well-formed, unexpired, bound to the same caller and window, and
  rejected otherwise;
- a cursor and a changed window may not be combined; and
- unknown properties are rejected.

## Currency and amounts

Carry `approvedGrossTotal`, `originalGrossTotal`, `taxTotal`, and `currencyIsoCode` unchanged. Do
not sum across currencies, do not convert using `invoiceCurrencyDetails` FX rates, and do not
compute a net payable from `exposurePercentage`. Surface `exposurePercentage` as reported and let
the accounts-payable system apply it.

## Tests

Prove:

- batched plus allow-listed status is approved;
- batched with a non-allow-listed status is excluded and counted;
- allow-listed status without batch release is excluded;
- an empty allow-list fails startup;
- allow-list matching is exact and case-sensitive;
- duplicate revisions of one `invoiceGroupId` raise a typed conflict;
- window defaulting, ordering, the 31-day ceiling, and epoch-millisecond conversion are correct
  across time zones and around daylight-saving transitions;
- cursor tampering, expiry, caller mismatch, and window mismatch are rejected;
- no evaluation path performs a network call; and
- decimal handling is culture-independent and lossless.

## Acceptance criteria

- The evidence rule is deterministic, network-free, and configuration-driven.
- Excluded invoices are counted and explained rather than dropped silently.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the decision to require accounts-payable batch
release rather than trusting an invoice status string. Do not push unless requested.
