# Reusable contract

Submit this contract before the implementation prompts. If stages run in separate coding-agent
tasks, prepend this contract to every stage.

You are creating a production-quality repository called `BrightFlagProxyMCPServer` in an empty or
partially initialized workspace.

## Product objective

Build a narrowly scoped MCP server with exactly three business capabilities:

1. Report a deterministic schema describing the BrightFlag entities, relationships, field types,
   and operation provenance this server uses, in a form an ontology service can ingest.
2. List invoices that are approved for payment, using accounts-payable batch release as the
   authoritative evidence.
3. Mark a single approved invoice as paid through the BrightFlag payment-status feed.

The server is not a general BrightFlag proxy, a generic REST client, or an OpenAPI-driven tool
generator.

The server never calls an ontology service. It only reports a schema that a separate ontology
service may ingest.

## Exact MCP contract

Expose only:

- `brightflag_get_ontology_schema`
- `brightflag_list_invoices_approved_for_payment`
- `brightflag_plan_invoice_payment`
- `brightflag_mark_invoice_paid`

Expose one read-only resource:

- `brightflag://ontology-schema`

Do not generate tools or resources from the BrightFlag OpenAPI document at runtime.

## The five fixed BrightFlag operations

The reviewed OpenAPI document is published at `/v3/api-docs/external` on the configured BrightFlag
origin. Version 1 of this server calls only these operations:

| Purpose | Method and path | `operationId` |
|---|---|---|
| List accounts-payable batches | `GET /api/ap-batch/v1` | `getAPBatches` |
| List accounts-payable batches in a window | `GET /api/ap-batch/v1/{startEpochTime}/{endEpochTime}` | `getAPBatchesByDateRange` |
| List invoice identifiers in a batch | `GET /api/ap-batch/v1/batch/{batchID}/invoices` | `getBatchInvoices` |
| Read invoice summaries | `GET /api/v1/invoice-summary` | `getInvoiceSummaryList` |
| Update invoice payment status | `POST /api/v1/invoicePayment/invoice-payment-status` | `runAPIApPaidStatusFeed` |

Every other BrightFlag operation is out of contract, including SCIM user provisioning, matters,
matter budgets, matter allocations, vendors, vendor offices, pay sites, trading entities,
allocations, purchase orders, tax rates, legal service requests, reporting, invoice document
download, and accounts-payable batch file download.

## Non-negotiable architecture

- Use the latest supported patch of .NET 10 LTS, C#, ASP.NET Core, and the stable official MCP C#
  SDK.
- Support local stdio and remote authenticated stateless Streamable HTTP at `/mcp`.
- Pin centrally managed NuGet versions after checking current primary documentation.
- Configure exactly one BrightFlag origin per deployment, outside MCP arguments.
- Never accept a base URL, path, operation, query string, HTTP method, header, or credential from
  an MCP argument.
- BrightFlag authenticates with a bearer JWT. Tokens come from a configured secret provider, never
  from a caller, a tool argument, or a prompt.
- "Approved for payment" means an invoice identifier released in an accounts-payable batch whose
  invoice-summary status is in the reviewed approved-status allow-list. An invoice status alone is
  never sufficient evidence.
- "Paid" is asserted only through `runAPIApPaidStatusFeed` with `paymentStatus` `PAID`. Version 1
  does not send `PARTIALLY PAID`, `POSTED`, or `VOID`.
- The payment write requires typed validation, caller authorization, plan-before-execute
  confirmation, and at most one outbound POST attempt per atomically consumed plan token, with no
  automatic retry. A plan whose POST may have been sent is permanently indeterminate and cannot be
  reused.
- Batch lookback defaults to 7 days and has a maximum of 31 days. Every read uses bounded windows,
  projections, page sizes, and response-byte limits.
- Paging exposed by this server is cursor-based. BrightFlag's `paging.pageNumber` and
  `paging.pageSize` parameters stay inside the API adapter and are never surfaced as MCP arguments.
- The ontology schema is generated deterministically from checked-in contracts and a reviewed
  OpenAPI snapshot. It contains no live BrightFlag data or environment-specific values.
- Credentials, bearer tokens, cookies, certificates, invoice payloads, vendor data, matter data,
  and payment records never enter Git or telemetry.
- Use synthetic fixtures and a local fake BrightFlag server for automated tests.
- Live sandbox tests are opt-in and excluded from normal CI.
- Do not add document download, file export, bulk feeds, webhooks, browser automation, database
  access, or other BrightFlag capabilities.
- Do not implement functionality beyond the current stage.

## Working method

1. Inspect the workspace and repository instructions before editing.
2. Explain the current stage and state a short plan.
3. Verify unstable SDK details and every BrightFlag field name against official documentation and
   the checked-in OpenAPI snapshot.
4. Implement the smallest coherent increment.
5. Add success, rejection, authorization, and boundary tests.
6. Assert the exact outbound method, path, query, headers, body, and call count observed by the
   fake BrightFlag server.
7. Run formatting, build, tests, schema checks, and security checks; fix failures.
8. Report files changed, commands, results, assumptions, and remaining risks.
9. Commit locally when asked, but do not push, open, label, or merge a pull request unless
   explicitly requested.

## Evidence rules

- Acceptance criteria require executable evidence.
- Do not weaken validation, authorization, confirmation, or tests to make a stage pass.
- If the reviewed OpenAPI snapshot does not contain a required field or operation, stop and report
  the incompatibility instead of guessing a field name, inventing a status value, or redefining
  "approved for payment".
- BrightFlag invoice-status vocabularies vary by customer configuration. Never hard-code a status
  string the reviewed configuration has not confirmed.
- Tool descriptions, annotations, prompt text, and a model's claim never grant authority.
