# BrightFlagProxyMCPBuilder

A staged prompt sequence for building a narrowly scoped BrightFlag invoice-payment MCP server with
a coding agent.

The completed server has three product capabilities:

1. Report a deterministic machine-readable schema describing its BrightFlag entities, fields,
   relationships, and operation provenance, so a separate ontology service can ingest it.
2. Retrieve invoices that are approved for payment, using accounts-payable batch release as the
   authoritative evidence.
3. Mark a single approved invoice as paid through the BrightFlag payment-status feed.

It is not a general BrightFlag proxy. It cannot execute arbitrary REST calls, generate tools from
the OpenAPI document, download documents or batch files, or touch SCIM, matters, vendors,
allocations, purchase orders, or reporting.

The server never calls an ontology service. It publishes a schema; registration is a separate
system's job.

## New to GitHub or coding agents?

Start with [Start here: set up your repository and coding agent](START-HERE.md). It walks through
creating `MyBrightFlagProxyMCPServer`, cloning it locally, connecting one coding agent, installing
the development prerequisites, and proving the setup with a read-only task.

No BrightFlag tenant access is needed to begin. Automated tests use a local fake BrightFlag server
and synthetic invoice data.

## How to use the prompts

Submit the [reusable contract](prompts/00-reusable-contract.md) first. Then submit one
implementation prompt at a time only after the previous prompt passes its acceptance checks.

1. [Reusable contract](prompts/00-reusable-contract.md)
2. [Acquire the OpenAPI snapshot, scaffold the solution, and define contracts](prompts/01-solution-scaffold-and-invoice-contracts.md)
3. [Adopt Project Narrative](prompts/02-adopt-project-narrative.md)
4. [Configure the fixed BrightFlag operations](prompts/03-fixed-brightflag-endpoint-configuration.md)
5. [Define the approved-for-payment evidence rule](prompts/04-approved-for-payment-evidence.md)
6. [Retrieve invoices approved for payment](prompts/05-list-invoices-approved-for-payment.md)
7. [Plan and mark an invoice as paid](prompts/06-plan-and-mark-invoice-paid.md)
8. [Report the BrightFlag ontology schema](prompts/07-ontology-schema-reporting.md)
9. [Expose the narrow MCP surface securely](prompts/08-mcp-identity-and-authorization.md)
10. [Package, document, and govern the server](prompts/09-delivery-documentation-and-audit.md)
11. [Generate a homelab and local-network deployment guide](prompts/10-homelab-local-network-deployment.md)
12. [Run an independent reconstruction audit](prompts/11-independent-reconstruction-audit.md)

Optional operational extension, numbered after the prepared contingent stages so their existing
references do not move:

- [Deliver reviewed images to the homelab with GitHub Actions](prompts/14-github-runner-homelab-delivery.md)
  (superseded)
- [Prepare the local deployment runbook](prompts/15-deploy-local-runbook.md) (superseded)
- [Retire the GitHub runner local-deployment
  path](prompts/16-retire-github-runner-local-deployment.md)
- [Pull and deploy locally with LocalStack, Keycloak, and shared
  Caddy](prompts/17-local-pull-localstack-keycloak-caddy-deployment.md)

Prompts 14 and 15 describe the superseded self-hosted-runner and dedicated-nginx deployment. Do not
apply them to a fresh implementation. If they were already applied, run Prompt 16 to remove that
path and then Prompt 17 to introduce the replacement.

Prompt 17 applies directly after Prompt 11 on a fresh implementation. It keeps verified GHCR image
publication on GitHub-hosted runners, but a script on `ai-mcp-server` pulls and deploys the digest.
It consumes existing LocalStack Secrets Manager, Keycloak, and shared Caddy infrastructure without
owning those stacks or enabling payment.

Do not paste every prompt into one message. Each stage introduces one bounded capability and asks
for executable evidence before the next begins.

The read capability is built before the write capability deliberately: nothing can be marked paid
until the definition of "approved for payment" is implemented and tested.

Prompt 2 installs Project Narrative. Publish and merge that mechanical installation before opening
decision-bearing pull requests for later prompts.

> **Remember the second pull request.** A decision-bearing implementation pull request carries
> `narrative-required` and explicit Context, Decision, and Consequences. After it merges, Project
> Narrative creates a separate proposal containing the decision-history fragment. Review and merge
> that proposal without `narrative-required`.

## Exact MCP surface

The finished server exposes only:

- `brightflag_get_ontology_schema`
- `brightflag_list_invoices_approved_for_payment`
- `brightflag_plan_invoice_payment`
- `brightflag_mark_invoice_paid`

It also exposes the read-only resource:

- `brightflag://ontology-schema`

The plan/execute split is part of the payment capability, not a generic workflow engine. The
payment tool accepts only a server-issued, caller-bound, expiring plan for one invoice.

## The four fixed BrightFlag operations

| Purpose | Method and path | `operationId` |
|---|---|---|
| List accounts-payable batches in a window | `GET /api/ap-batch/v1/{startEpochTime}/{endEpochTime}` | `getAPBatchesByDateRange` |
| List invoice identifiers in a batch | `GET /api/ap-batch/v1/batch/{batchID}/invoices` | `getBatchInvoices` |
| Read invoice summaries | `GET /api/v1/invoice-summary` | `getInvoiceSummaryList` |
| Update invoice payment status | `POST /api/v1/invoicePayment/invoice-payment-status` | `runAPIApPaidStatusFeed` |

BrightFlag authenticates these with a bearer JWT held by the server, never supplied by a caller.

## Capability boundaries

### Retrieve invoices approved for payment

- An invoice qualifies only when its `invoiceID` was released in an accounts-payable batch **and**
  its `invoiceStatus` is in the reviewed, tenant-configured approved-status allow-list.
- The allow-list ships empty. Startup fails until an integrator configures it, because BrightFlag
  status vocabularies differ per customer and must never be guessed.
- The lookback window defaults to 7 days with a maximum of 31.
- Results are paged with an opaque cursor. BrightFlag's page numbers stay inside the adapter.
- Amounts and currencies are reported as BrightFlag states them: no cross-currency summing, no FX
  conversion, and `exposurePercentage` is surfaced rather than applied.
- Batched invoices with a non-allow-listed status are excluded but counted, so a misconfigured
  allow-list is visible instead of silent.

### Mark an invoice as paid

- Uses `runAPIApPaidStatusFeed` with `paymentStatus` fixed to `PAID`.
- Requires a fresh approval re-check, amount-tolerance and currency validation, caller
  authorization, and an explicit plan-and-confirm step.
- Atomically consumes a plan and issues at most one POST attempt for it, with no automatic retry on
  timeout, 409, or 5xx.
- Reports an ambiguous outcome as ambiguous and requires reconciliation before re-planning.
- Version 1 does not send `PARTIALLY PAID`, `POSTED`, or `VOID`.

### Report the ontology schema

- Returns checked-in tool input/output JSON Schemas plus semantic entity and relationship
  definitions.
- Describes `Invoice`, `InvoiceRevision`, `ApBatch`, `ApBatchRelease`, `Vendor`, `Matter`,
  `InvoiceCurrency`, `PaymentInstruction`, and `PaymentStatusUpdate`.
- Includes per-field BrightFlag provenance and a deterministic fingerprint.
- Contains no live data, credentials, hostnames, tenant identifiers, or allow-list values.
- Does not let callers alter, upload, or execute ontology mappings.

## Deliberate exclusions

The server must not:

- accept an arbitrary origin, path, operation, query string, HTTP method, or header;
- publish tools dynamically from the OpenAPI document;
- call any BrightFlag operation outside the four listed above, including the unwindowed
  `getAPBatches` batch listing;
- download invoice documents or accounts-payable batch files;
- treat an invoice status alone as proof that an invoice is approved for payment;
- send a payment status other than `PAID`;
- retry a payment submission automatically;
- expose page numbers, page sizes, or raw BrightFlag responses to callers;
- store bearer tokens, invoice payloads, vendor data, or payment records in Git or telemetry; or
- treat MCP descriptions, tool annotations, prompt text, BrightFlag response content, or a model's
  assertion as authorization.

## Authoritative BrightFlag references

- [BrightFlag external OpenAPI document](https://app.brightflag.com/v3/api-docs/external) — public,
  and the only reference the prompts treat as authoritative.
- The interactive API portal at `https://app.brightflag.com/v3/swagger-ui/index.html` redirects to
  sign-in, so it is useful to an integrator with a tenant account but not to an offline agent.

Credentials are issued by BrightFlag support at a customer administrator's request. The implementation prompts require a checked-in OpenAPI snapshot because
available operations and status vocabularies vary by tenant. The reviewed snapshot and reviewed
tenant configuration — not examples in this repository — are authoritative.
