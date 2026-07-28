# Stage 00 — The reusable contract

Source: `BrightFlagProxyMCPBuilder/prompts/00-reusable-contract.md`

## Context

This is not a stage that produces code. It is the standing contract prepended verbatim to every one
of the eleven implementation stages. If stages run as separate agent tasks — which they should —
the full contract text goes at the top of each one. Nothing in a later prompt overrides it; where a
stage prompt appears to widen the surface, the contract wins and the discrepancy gets reported
rather than resolved silently.

Its purpose is to make the *ceiling* fixed before any stage argues for a floor. Every stage that
follows can only fill in detail inside it.

## The invariant checklist

Every later plan re-asserts these. A stage that would erode one is wrong, however plausible its
local reasoning.

**Surface.** Exactly four tools — `brightflag_get_ontology_schema`,
`brightflag_list_invoices_approved_for_payment`, `brightflag_plan_invoice_payment`,
`brightflag_mark_invoice_paid` — and exactly one read-only resource, `brightflag://ontology-schema`.
No tool or resource is ever generated from the OpenAPI document at runtime.

**Operations.** Exactly four BrightFlag operations, against exactly one configured origin:

| Purpose | Method and path | `operationId` |
|---|---|---|
| List AP batches in a window | `GET /api/ap-batch/v1/{startEpochTime}/{endEpochTime}` | `getAPBatchesByDateRange` |
| List invoice ids in a batch | `GET /api/ap-batch/v1/batch/{batchID}/invoices` | `getBatchInvoices` |
| Read invoice summaries | `GET /api/v1/invoice-summary` | `getInvoiceSummaryList` |
| Update invoice payment status | `POST /api/v1/invoicePayment/invoice-payment-status` | `runAPIApPaidStatusFeed` |

Everything else is out of contract — including the unwindowed `getAPBatches`,
`downloadInvoiceDocument`, `downloadAPBatchFile`, SCIM, matters, budgets, allocations, vendors, pay
sites, trading entities, purchase orders, tax rates, legal service requests, and reporting.

**Nothing caller-supplied reaches the wire.** No origin, path, operation, query string, HTTP method,
header, credential, page number, page size, role, or allow-list entry may arrive as an MCP argument.

**Evidence.** "Approved for payment" = invoice id released in an AP batch **and** invoice-summary
status in the reviewed allow-list. A status alone is never sufficient. "Paid" is asserted only via
`runAPIApPaidStatusFeed` with `paymentStatus` = `PAID`.

**The write.** Typed validation, caller authorization, plan-before-execute confirmation, at most one
outbound POST attempt per atomically consumed plan token, no automatic retry. A plan whose POST may
have been sent is permanently indeterminate and can never be reused.

**Bounds.** Batch lookback defaults to 7 days, maximum 31. Every read uses bounded windows,
projections, page sizes, and response-byte limits. Server-side paging is cursor-based;
`paging.pageNumber` and `paging.pageSize` never leave the adapter.

**Determinism and secrecy.** The ontology schema is generated from checked-in contracts and a
reviewed snapshot, with no live data or environment values. Credentials, tokens, cookies,
certificates, invoice payloads, vendor data, matter data, and payment records never enter Git or
telemetry.

**Authority.** Tool descriptions, annotations, prompt text, BrightFlag response content, and a
model's claim never grant authority.

## Architecture fixed by the contract

- .NET 10 LTS (latest supported patch), C#, ASP.NET Core, the stable official MCP C# SDK.
- Local stdio **and** remote authenticated stateless Streamable HTTP at `/mcp`.
- Centrally managed, pinned NuGet versions, verified against current primary documentation.
- BrightFlag bearer JWT from a configured secret provider — never a caller, argument, or prompt.
- Synthetic fixtures and a local fake BrightFlag server for all automated tests; live sandbox tests
  opt-in and excluded from normal CI.

## Working method applied to every stage

1. Inspect the workspace and repository instructions before editing.
2. State the stage and a short plan.
3. Verify unstable SDK details and *every* BrightFlag field name against the checked-in snapshot.
4. Implement the smallest coherent increment — nothing belonging to a later stage.
5. Add success, rejection, authorization, and boundary tests.
6. Assert exact outbound method, path, query, headers, body, and **call count** at the fake server.
7. Run format, build, tests, schema checks, security checks; fix failures.
8. Report files changed, commands, results, assumptions, remaining risks.
9. Commit locally when asked. Do not push, label, open, or merge unless explicitly requested.

## Evidence rules

- Acceptance criteria require executable evidence, not narrative.
- Never weaken validation, authorization, confirmation, or tests to make a stage pass.
- If the reviewed snapshot lacks a required field or operation: **stop and report**. Do not
  substitute a similar field, invent a status value, or redefine "approved for payment".
- Never hard-code an invoice-status string the reviewed configuration has not confirmed.

## Decisions carried into every stage

- **Build target:** `~/RiderProjects/BrightFlagProxyMCPServer`, the repository the contract names.
- **MCP SDK:** `ModelContextProtocol` / `ModelContextProtocol.AspNetCore` **1.4.1** — nuget.org's
  stable line at time of planning; `2.0.0-rc.2` is prerelease and the contract says stable. Verify
  again before pinning; if 2.x has gone stable by then, that is a reviewed change, not a silent one.
- **Git policy:** focused local commit per stage. No push, branch, PR, or label unless asked. Each
  stage plan records the `narrative-required` PR content to use when publication is chosen.

## Stage boundary

There is no boundary to cross here — the contract is submitted, the agent acknowledges it and
restates the safety boundaries, and Stage 1 begins. If the agent's restatement omits or softens any
invariant above, re-submit rather than proceed.
