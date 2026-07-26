# Prompt 6 — Plan and mark an invoice as paid

Using the reusable contract, implement Stage 6: the only mutation this server performs. Treat it as
a financial action, not a status update.

`POST /api/v1/invoicePayment/invoice-payment-status` is not documented as idempotent, accepts one
invoice per call, and BrightFlag echoes the submitted payment status on success. The plan split,
the at-most-one-attempt rule, and the audit record exist because of those three facts.

## Payload

Map a domain payment confirmation to the reviewed `InvoicePaymentStatus` schema. Version 1 sends
only:

- `invoiceID`;
- `invoiceNumber` and `vendorRef` as corroborating identity;
- `paymentStatus`, fixed to `PAID`;
- `paymentRef`, the accounts-payable system's payment identifier;
- `paymentAmount`, formatted invariantly with a decimal point, no grouping, and no symbol;
- `paymentDate`, formatted `YYYY/MM/DD`; and
- `paymentComment` when supplied and within a configured length.

Do not send `postedAmount`, `postedCurrency`, `postedDate`, `postedFXRate`, `vendorOfficeId`, or
`invoiceDate` in version 1. Reject `PARTIALLY PAID`, `POSTED`, and `VOID`: partial payment,
posting, and voiding are separate reviewed capabilities, and `VOID` in particular is not a payment.

## Plan step

`brightflag_plan_invoice_payment` accepts the invoice identity, `paymentRef`, `paymentAmount`,
`paymentDate`, and optional comment. It:

- re-verifies the invoice is currently approved for payment using the Prompt 4 rule and a fresh
  read, not a cached listing or a value the caller asserted;
- validates `paymentAmount` against `approvedGrossTotal` within a configured absolute or
  proportional tolerance, and rejects a mismatch;
- validates the currency is the invoice's `currencyIsoCode` and rejects a cross-currency payment;
- validates `paymentDate` is not in the future beyond configured skew and not older than a
  configured window;
- rejects a `paymentRef` already recorded against a different invoice, and a second plan for an
  invoice already marked paid by this server;
- returns the exact outbound body that would be sent, with the token absent; and
- issues a plan token bound to caller, tenant, invoice identity, amount, currency, `paymentRef`,
  and contract version, expiring after a short configured lifetime.

The plan performs no write.

## Execute step

`brightflag_mark_invoice_paid` accepts only a plan token and an explicit confirmation flag. It:

- rejects an expired, replayed, tampered, unknown, or cross-caller token;
- rejects any attempt to override a planned field at execution time;
- performs authorization independently of the plan step;
- atomically consumes the plan before attempting the write and issues at most one POST attempt for
  that plan, with no automatic retry on timeout, transport failure, 409, or 5xx;
- treats an ambiguous outcome as ambiguous, returning a typed unknown-result error that names the
  `paymentRef` and instructs the operator to reconcile in BrightFlag before re-planning;
- verifies the echoed response matches the submitted invoice and status, and fails loudly when it
  does not;
- writes an audit record containing caller, tenant, invoice identity, `paymentRef`, amount,
  currency, decision, outcome, and timing, and no token or bearer credential; and
- returns a `PaidInvoiceReceipt`.

Mark the tool as destructive and non-idempotent in its annotations, and prove the annotation is not
what enforces confirmation.

## Tests

Prove:

- a valid plan produces the exact expected JSON body, field for field, including invariant decimal
  and `YYYY/MM/DD` date formatting;
- successful execution issues one POST, while every plan produces at most one POST attempt even
  under replay, concurrent confirmation, cancellation, or process-shutdown paths;
- amount mismatch, currency mismatch, stale or future payment date, non-approved invoice, duplicate
  `paymentRef`, and already-paid invoice are all rejected before any POST;
- expired, tampered, replayed, and cross-caller tokens are rejected;
- a timeout, a 409, and a 500 each produce no second POST;
- an echoed response for a different invoice or status fails;
- unauthorized callers are refused at both steps; and
- audit records contain no credential and no full payload dump.

## Acceptance criteria

- Only `PAID` can be sent, and only through a valid plan.
- At most one outbound POST attempt per atomically consumed plan token. A plan is never made
  reusable after execution begins.
- Ambiguity is reported as ambiguity, never as success or as a silent retry.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the decision to gate the only mutation behind a
caller-bound expiring plan with no automatic retry. Do not push unless requested.
