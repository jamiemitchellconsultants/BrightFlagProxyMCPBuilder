# Stage 04 — Define the approved-for-payment evidence rule

Source: `BrightFlagProxyMCPBuilder/prompts/04-approved-for-payment-evidence.md`

## Context

This is the definition the whole server exists to protect. It is implemented as a **pure,
network-free, configuration-driven** rule so that it can be tested exhaustively without a transport,
and so that Stage 6's payment plan can re-run it against a fresh read without duplicating logic.

`DESIGN-CALLS.md` §1 records why it is two signals rather than one: `getInvoiceSummaryList` accepts
an `invoiceStatus` filter, so the obvious implementation is a single filtered call — but the
reviewed document defines **no enumeration** for that field (confirmed: `invoiceStatus` has a
description and no `enum`), and status vocabularies are configured per customer. The obvious version
requires inventing a string, and a wrong guess silently reports the wrong set of invoices to whoever
is about to pay them.

## Preconditions

Stage 3 committed: configuration validated, allow-list configurable, fake server with the two
mandatory boundary fixtures.

## Scope in

The evidence rule; query validation; revision handling; the excluded-count contract; currency and
amount carriage rules.

## Scope explicitly out

Any outbound call — this stage is provably network-free. The adapter (Stage 5). The MCP tool
(Stage 5).

## Work items

### 1. The rule

An invoice is approved for payment only when **both** hold:

1. **Release evidence** — its `invoiceID` appears in an AP batch returned by
   `getAPBatchesByDateRange` within the requested bounded window. The batch is BrightFlag's record
   that the invoice was released to accounts payable: a released-to-finance event, not an opinion
   about a status label.
2. **Status corroboration** — its `InvoiceSummaryAPI.invoiceStatus` is in the reviewed allow-list
   configured in Stage 3.

Neither is sufficient alone. Record *which signals were observed* on every returned invoice, so a
reconciliation reviewer can see the basis of the decision rather than trusting the verdict.

### 2. Allow-list handling

The allow-list is **data, not code**. Ship no default. Startup fails when it is empty. Each entry is
compared as an **exact, case-sensitive, trimmed** string — never a substring, never a prefix, never
a case-insensitive match against words like "approv". Those three shortcuts are named as rejected
alternatives in `DESIGN-CALLS.md` precisely because each trades a visible configuration step for an
invisible wrong answer.

### 3. Excluded count

A batched invoice whose status is outside the allow-list is **not returned**, but is reported in a
bounded, separately named excluded-count carrying the **observed status**. This is what makes a
misconfigured allow-list show up as a non-zero excluded count instead of a short list nobody
questions. It also constrains Stage 5: the adapter cannot filter those rows away upstream, or the
signal disappears.

### 4. Draft revisions

`invoiceID` identifies a revision; `invoiceGroupId` identifies the logical invoice. The batch
releases a *revision*. So:

- send `includePreviousDrafts=false`;
- return at most one row per `invoiceGroupId`, preferring the revision the batch released;
- if a batch releases two revisions of the same logical invoice, **fail that row with a typed
  conflict** rather than choosing one silently.

The collapse happens after the join on `invoiceID`, never before — collapsing first destroys the
record of which revision was actually released.

### 5. Query validation

Validate `ApprovedInvoiceQuery` before any outbound call is even contemplated:

- lookback defaults to 7 days, may not exceed 31;
- explicit start and end instants must be ordered, must not be in the future beyond the configured
  clock skew, and must convert to the **epoch-millisecond strings** the batch operation's path
  segments require (the reviewed document types both segments as `string`);
- a supplied cursor must be well-formed, unexpired, and bound to the same caller and window —
  rejected otherwise;
- a cursor and a changed window may not be combined;
- unknown properties are rejected.

### 6. Currency and amounts

Carry `approvedGrossTotal`, `originalGrossTotal`, `taxTotal`, and `currencyIsoCode` **unchanged**.
Do not sum across currencies. Do not convert using `invoiceCurrencyDetails` FX rates. Do not compute
a net payable from `exposurePercentage` — surface it as reported and let the accounts-payable system
apply it. All four are JSON `number` upstream and `decimal` in the domain; the conversion must be
lossless and culture-independent.

## Tests

One-to-one with the prompt's list:

- batched **and** allow-listed → approved;
- batched with a non-allow-listed status → excluded **and counted**, with the observed status;
- allow-listed status without batch release → excluded;
- empty allow-list → startup fails;
- allow-list matching is exact and case-sensitive — include entries differing only by case,
  leading/trailing whitespace, and substring;
- duplicate revisions of one `invoiceGroupId` → typed conflict;
- window defaulting, ordering, the 31-day ceiling, and epoch-millisecond conversion — correct across
  time zones and **around daylight-saving transitions** (run under at least two `TZ` values);
- cursor tampering, expiry, caller mismatch, window mismatch → rejected;
- **no evaluation path performs a network call** — assert structurally, not by observation;
- decimal handling is culture-independent and lossless (run under at least two cultures).

## Acceptance checks

- The rule is deterministic, network-free, and configuration-driven.
- Excluded invoices are counted and explained, never dropped silently.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

## Stage boundary

Commit locally. Suggested message: `Define approved-for-payment evidence rule`.

`narrative-required` when published. The decision to record: **requiring accounts-payable batch
release rather than trusting an invoice status string.** Context — the reviewed document defines no
status enumeration and vocabularies vary per customer. Decision — two signals, allow-list ships
empty, startup fails until configured. Consequences — stricter than the API requires; a tenant not
using the batch feed gets an empty result from a correctly working server, loudly rather than
silently; reversing it means changing this stage, Stage 7's vocabulary, and Stage 9's documentation
together.

Do not push unless requested. **Do not begin Stage 5** — no adapter, no outbound call.

## Risks

- The temptation at this stage is to "just check the status" for a simpler test. That is the exact
  failure mode the stage exists to prevent.
- DST and epoch conversion bugs are easy to write and easy to miss. Pin the tests to specific
  instants either side of a transition in a zone that has one, not to `DateTime.Now`.
