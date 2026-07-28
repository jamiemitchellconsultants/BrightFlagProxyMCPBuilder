# Stage 06 — Plan and mark an invoice as paid

Source: `BrightFlagProxyMCPBuilder/prompts/06-plan-and-mark-invoice-paid.md`

## Context

The only mutation this server performs, and the sharpest edge in the project. It writes a payment
assertion into a legal-spend system that finance teams reconcile against real money.

Three verified properties of `POST /api/v1/invoicePayment/invoice-payment-status` drive the whole
design. It accepts **one invoice per call**, so there is no bulk path to reason about. It is **not
documented as idempotent** and carries no client-supplied idempotency key, so a retried submission
may or may not produce a second payment assertion. And it **echoes the submitted
`InvoicePaymentStatus`** on success, which gives the server something concrete to verify rather than
trusting a 200.

Ordinary resilience engineering — retry the 5xx, retry the timeout — is the wrong instinct here. A
timeout after the server received the request is indistinguishable from a timeout before it.
Retrying resolves that ambiguity by guessing, in the direction that risks a duplicate payment
record. Refusing to retry leaves the ambiguity where a human can settle it.

## Preconditions

Stage 5 committed: read capability working end to end, evidence rule reachable for a fresh re-check.

## Scope in

The payload mapping; the plan step; the execute step; the plan store and payment record store; the
audit record.

## Scope explicitly out

`PARTIALLY PAID`, `POSTED`, `VOID`. Partial-payment amount modelling. Transports and JWT identity
(Stage 8) — though authorization is *called* here and re-evaluated at execution.

## Work items

### 1. Payload — version 1 sends exactly these

Mapped to the reviewed `InvoicePaymentStatus` schema, whose only required member is `paymentStatus`:

- `invoiceID`;
- `invoiceNumber` and `vendorRef` as corroborating identity;
- `paymentStatus`, **fixed to `PAID`** — not a caller argument, not a validated enum, a constant;
- `paymentRef`, the accounts-payable system's payment identifier;
- `paymentAmount`, formatted invariantly: decimal point, no grouping, no symbol (the field is a
  string upstream);
- `paymentDate`, formatted `YYYY/MM/DD` (matching the document's own example, `2019/10/30`);
- `paymentComment` when supplied and within the configured length.

Do **not** send `postedAmount`, `postedCurrency`, `postedDate`, `postedFXRate`, `vendorOfficeId`, or
`invoiceDate`, all of which exist on the schema and are deliberately unused.

`DESIGN-CALLS.md` §2 names exposing `paymentStatus` as a caller argument "the most tempting and the
worst" rejected alternative: it turns one reviewed capability into four unreviewed ones.

### 2. `brightflag_plan_invoice_payment`

Accepts invoice identity, `paymentRef`, `paymentAmount`, `paymentDate`, optional comment. It:

- **re-verifies approval** using Stage 4's rule and a *fresh read* — not a cached listing, not a
  value the caller asserted;
- validates `paymentAmount` against `approvedGrossTotal` within the configured absolute or
  proportional tolerance, rejecting a mismatch;
- validates the currency equals the invoice's `currencyIsoCode`, rejecting cross-currency payment;
- validates `paymentDate` is not future beyond configured skew, and not older than a configured
  window;
- rejects a `paymentRef` already recorded against a **different** invoice, and a second plan for an
  invoice **already marked paid by this server**;
- returns the exact outbound body that would be sent, with the token absent;
- issues a plan token from a cryptographically secure random source, ≥128 bits of entropy, **opaque
  and encoding none of the bound fields**, expiring after a short configured lifetime.

The plan performs no write.

The binding — caller, tenant, invoice identity, amount, currency, `paymentRef`, contract version —
lives **only in the server-side plan store record** the token references. The token is a capability,
not a signed self-describing structure. That is the opposite of the cursor design in Stage 1, and
deliberately so: a cursor is a resumption hint, a plan token authorises a financial write.

### 3. `brightflag_mark_invoice_paid`

Accepts **only** a plan token and an explicit confirmation flag. It:

- rejects expired, replayed, tampered, unknown, or cross-caller tokens;
- rejects any attempt to override a planned field at execution;
- performs authorization **independently** of the plan step;
- **atomically consumes** the plan before attempting the write, and issues at most one POST attempt
  for that plan — no automatic retry on timeout, transport failure, 409, or 5xx;
- treats an ambiguous outcome as ambiguous: a typed unknown-result error naming the `paymentRef`,
  instructing the operator to reconcile in BrightFlag before re-planning;
- verifies the echoed response matches the submitted invoice and status, failing loudly otherwise;
- writes an audit record — caller, tenant, invoice identity, `paymentRef`, amount, currency,
  decision, outcome, timing — and **no token or bearer credential**;
- returns a `PaidInvoiceReceipt`.

Annotate destructive and non-idempotent — and prove the annotation is not what enforces
confirmation.

### 4. Stores

Two abstractions in `src/BrightFlagMcp.Core`, with in-process implementations here:

- `IPlanStore` — caller-scoped, capacity-bounded, expiring. Consumption must be atomic
  (compare-and-remove), because the one-attempt guarantee reduces to that single operation being
  race-free.
- `IPaymentRecordStore` — the ledger Stage 6 implies but Stage 1 never contracted. It backs the
  duplicate-`paymentRef` and already-paid checks. Bounded and expiring like the plan store.

**Retention is a real limit, not a guarantee.** The already-paid check is only as strong as the
record store's retention window; say so plainly, and carry it into Stage 9's runbook rather than
overclaiming permanence.

Keeping both behind interfaces is what lets Stage 8 declare a multi-instance **live** topology and
add a shared implementation without rewriting this stage. Stage 10's dev deployment runs one
instance and does not constrain that choice.

## Tests

- a valid plan produces the exact expected JSON body, field for field, including invariant decimal
  and `YYYY/MM/DD` formatting;
- successful execution issues **one** POST; every plan produces at most one POST attempt under
  replay, concurrent confirmation, cancellation, and process-shutdown paths;
- amount mismatch, currency mismatch, stale or future payment date, non-approved invoice, duplicate
  `paymentRef`, and already-paid invoice are each rejected **before any POST**;
- expired, tampered, replayed, and cross-caller tokens rejected;
- plan tokens come from a CSPRNG, with no observable correlation between issued tokens and no
  plausible path to guess or enumerate one;
- a timeout, a 409, and a 500 each produce **no second POST**;
- an echoed response for a different invoice or status fails;
- unauthorized callers refused at **both** steps;
- audit records contain no credential and no full payload dump.

## Acceptance checks

- Only `PAID` can be sent, and only through a valid plan.
- At most one outbound POST attempt per atomically consumed plan token; a plan is never made
  reusable after execution begins.
- Ambiguity is reported as ambiguity — never as success, never as a silent retry.
- The plan token is an opaque random capability; bound fields live only in the plan store.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

## Stage boundary

Commit locally. Suggested message: `Gate the payment write behind a caller-bound expiring plan`.

`narrative-required` when published. Decision to record: **gating the only mutation behind a
caller-bound expiring plan with no automatic retry.** Context — the endpoint is non-idempotent,
single-invoice, and echoes its input. Decision — plan/confirm split, atomic consumption, one
attempt, ambiguity quarantined. Consequences — an operator must reconcile manually after an
ambiguous submission, and a caller cannot express a partial payment at all; both are real
limitations taken deliberately in version 1.

Do not push unless requested. **Do not begin Stage 7.**

## Risks

- The concurrency tests are the ones most likely to be written weakly. Drive genuine parallel
  confirmations of one token and assert the fake server's recorded call count is exactly one.
- "No automatic retry" must survive the HTTP stack's own defaults — check that no handler in the
  pipeline retries POSTs, and assert it, since Stage 5 legitimately adds retry for GETs.
- The already-paid check depends on the record store surviving restart in whatever topology Stage 8
  declares. Note the dependency here; do not resolve it here.
