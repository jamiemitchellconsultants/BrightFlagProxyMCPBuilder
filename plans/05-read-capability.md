# Stage 05 — Retrieve invoices approved for payment

Source: `BrightFlagProxyMCPBuilder/prompts/05-list-invoices-approved-for-payment.md`

## Context

First stage that performs outbound calls — against the fake server only. It turns Stage 4's rule
into a working capability by adding the adapter, the join, and the first MCP tool.

The read is built before the write deliberately (`DESIGN-CALLS.md` §3): Stage 6's payment plan
re-verifies approval using this exact rule and a fresh read. Building the write first would mean
stubbing that check, and a stub on the authorisation path of a financial mutation is precisely the
temporary code that survives to production.

## Preconditions

Stage 4 committed: evidence rule proven network-free and exhaustively tested. Fake server carries
the batched-not-allow-listed and allow-listed-not-batched fixtures.

## Scope in

The BrightFlag adapter; the three-call sequence and the join; the
`brightflag_list_invoices_approved_for_payment` tool; the MCP SDK's first appearance.

## Scope explicitly out

The payment operation — tested to be unreachable from here. Hosting, transports, JWT identity,
authorization policy (all Stage 8). The ontology schema (Stage 7).

## Why the call sequence is what it is

`getAPBatchesByDateRange` → `getBatchInvoices` (per batch) → `getInvoiceSummaryList` → in-process
join. Each step is forced, not chosen:

1. **The authoritative signal drives the query.** Stage 4 makes batch release authoritative and
   status corroborating. Starting at the batch feed makes release the candidate-set generator;
   everything after decorates it. Starting from a status-filtered summary read inverts that and
   reintroduces the guess the whole design removes.
2. **The window can only be bound on step 1.** `getAPBatchesByDateRange` takes the window as
   epoch-millisecond *path segments* — the only one of the four operations whose bounded window is
   structural. The unwindowed `getAPBatches` was deleted from the contract (Narrative entry 5) so no
   unbounded entry point survives. Step 1 is where the bounded-reads rule gets its anchor.
3. **The batch feed carries no invoice data, so the join is unavoidable.** `getBatchInvoices`
   returns `BatchDTO { batchID, invoices: [{ invoiceID }] }` — identifiers only. Number, date,
   vendor, matter, currency, and amounts exist only on `InvoiceSummaryAPI`. This is why demanding
   both signals costs nothing: a two-source join was mandatory regardless.
4. **Fan-out lives between steps 1 and 2.** Step 1 returns N batches; step 2 is N calls. That is the
   only multiplier in the read, so the configured maximum batch fan-out sits exactly there and
   **fails closed** rather than truncating. Step 1 itself returns a bare *unpaged* array of `Batch`
   — no paging parameters exist — so byte and result ceilings are its only bound.
5. **Status filtering at step 3 is an optimisation, never the decision.** `invoiceStatus` is
   single-valued with no enum, and the allow-list is tenant data — hence the prompt's hedge, "the
   reviewed status filter *where it narrows the request usefully*". The constraint inside that
   hedge: the excluded-count must report batched invoices whose status is **outside** the
   allow-list, with the observed status, so those rows must actually be fetched. Filtering them away
   upstream destroys the signal that makes a misconfigured allow-list visible.
6. **The join runs batch → summary, never the reverse.** Batch-released with no summary row is a
   typed reconciliation gap — reported and counted, never fabricated. A summary row with no batch
   release was never a candidate. The asymmetry is Stage 4's rule expressed as control flow.
7. **Revision collapse happens at the join.** Join on `invoiceID` to preserve which revision was
   released, then collapse per `invoiceGroupId`, raising the typed conflict from Stage 4 when a
   batch released two.

## The specification gap, and how this plan closes it

`getInvoiceSummaryList` has **no `invoiceID` filter**. Its parameters, verified in the reviewed
document, are `invoiceNumber, vendorRef, matterRef, invoiceStatusChangeStartDate,
invoiceStatusChangeEndDate, invoiceStatus, includePreviousDrafts, invoiceDateStartDate,
invoiceDateEndDate, customerMatterId, sorting.ascending, paging.pageSize, paging.pageNumber`. You
cannot ask for "these 40 invoice ids". Step 3 must therefore enumerate over a window and filter
locally — and the prompts never say which window.

Release date and status-change date are different axes: an invoice released in a batch on day 5 may
have last changed status on day 40. Bounding step 3 by the batch window alone manufactures
reconciliation gaps systematically.

**Resolution:** query `invoiceStatusChangeStartDate/EndDate` as the validated batch window widened
by the configured `SummaryWindowMarginDays` from Stage 3 (its own default and maximum, mirroring the
lookback). Anything still absent is a typed reconciliation gap — counted, reported, never
fabricated. One window to reason about, still bounded, and the miss stays visible.

## Work items

### 1. Adapter (`src/BrightFlagMcp.Api`)

- Attaches the bearer token from the configured provider and nothing caller-supplied.
- Always calls `getAPBatchesByDateRange` with epoch-millisecond path segments for the validated
  explicit or default window.
- Calls `getBatchInvoices` per released batch, bounded by the configured maximum fan-out.
- Calls `getInvoiceSummaryList` with `includePreviousDrafts=false`, the reviewed status filter where
  it narrows usefully, the margin-widened status-change window, and internal `paging.pageSize` /
  `paging.pageNumber` — which never leave the adapter.
- Enforces timeout, total-byte, page-count, and result-count ceilings, **failing closed** at each.
- Retries only idempotent GETs, only on transport failures and 429/503, with bounded jittered
  backoff and a total attempt cap. Safe here precisely because none of the three mutates — the
  deliberate contrast with Stage 6.
- Maps `ErrorResponse` and every non-2xx to the normalized error, preserving `errorCode` and
  `errorCategory` while discarding anything token-bearing.
- Parses `paymentAmount`-style string/number boundaries to `decimal` directly, never via `double`.

### 2. Tool

`brightflag_list_invoices_approved_for_payment`, arguments limited to:

- optional window `startedAfter` / `startedBefore`, subject to Stage 4's rules;
- optional opaque `cursor`;
- optional `maxResults`, bounded by configuration.

Nothing else. The result carries approved invoices, the excluded-count with observed statuses, the
reconciliation-gap count, the effective window, and an optional `nextCursor`. It never carries a
page number, a BrightFlag URL, a raw response body, or a token.

Annotate the tool read-only — and prove the annotation is *not* what enforces it, by asserting the
POST path is unreachable regardless of annotation.

### 3. MCP SDK first appearance

`ModelContextProtocol` 2.0.0, centrally pinned, with a minimal static tool registration in
`src/BrightFlagMcpServer`. Note the sequencing: Stage 1 forbids MCP packages and Stage 8 owns
hosting, but this stage's acceptance requires the capability to work "through the MCP tool
boundary". Introduce the SDK here; Stage 8 hardens rather than introduces.

## Tests — against the fake server

- exact outbound method, path, query string, headers, and **call count** for a single-page result;
- the same for a multi-page result driven by a returned keyset cursor, including that bounded
  upstream reads may repeat, rows at or before the last emitted key are discarded, and no invoice
  already returned in that cursor sequence is returned again;
- cursor rejection after a change of caller, window, contract, configuration, or snapshot, and on an
  unrecognized signing-key identifier; plus documented behaviour when upstream data inside the fixed
  window changes between pages;
- fan-out, page-count, byte, and result ceilings each fail closed with a typed error;
- 401 and 403 surface as authorization failures **without leaking the token**;
- 429 retries within the cap and then fails;
- the POST payment operation is **never** called by this tool;
- a batch-released invoice missing from the summary read is reported as a reconciliation gap, not
  imputed — use the out-of-window `invoiceStatusChangeTimestamp` fixture from Stage 3;
- responses carrying unexpected extra fields are ignored safely rather than trusted.

## Acceptance checks

- The read works end to end against the fake server through the MCP tool boundary.
- No caller argument can change origin, path, operation, headers, paging strategy, or the allow-list.
- Cursor round-trips are stable and tamper-evident.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

## Stage boundary

Commit locally. Suggested message: `Add approved-invoice read capability over cursor paging`.

`narrative-required` when published. Decision to record: **exposing cursor-based paging over
BrightFlag's page-number paging.** Context — page numbers leak upstream structure and break under
concurrent mutation. Decision — stateless signed keyset cursor; BrightFlag paging confined to the
adapter. Consequences — bounded upstream reads may repeat on resume; snapshot isolation is not
claimed and is documented as not claimed.

Worth recording alongside it, since it is a real decision this stage makes: the summary-window margin
resolving the `invoiceID`-filter gap.

Do not push unless requested. **Do not begin Stage 6** — no plan store, no payment payload, no POST.

## Risks

- Fan-out is the cost centre. A 31-day window on a busy tenant could return many batches; the
  ceiling must fail closed loudly rather than silently returning a partial set that looks complete.
- Over-eager status filtering at step 3 is the subtle failure: it makes tests pass and destroys the
  excluded-count. Test the excluded-count against a *live* fake-server round trip, not just the
  in-memory rule.
- The margin widens step 3's result set. Verify the byte and page ceilings account for it.
