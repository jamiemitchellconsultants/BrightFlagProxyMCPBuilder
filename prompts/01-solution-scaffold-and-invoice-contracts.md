# Prompt 1 — Acquire the OpenAPI snapshot, scaffold the solution, and define contracts

Using the reusable contract, implement Stage 1: acquire and review the public BrightFlag OpenAPI
document, then create the .NET solution and stable contracts for only the three requested
capabilities. Do not call a tenant API operation or expose MCP yet.

Create:

- `global.json` for the supported .NET 10 SDK feature band;
- `Directory.Build.props` enabling nullable types, deterministic builds, analyzers, and warnings as
  errors;
- `Directory.Packages.props` with centrally pinned packages;
- a checked-in NuGet lock file;
- `.editorconfig`, `.gitignore`, and an MIT licence;
- `src/BrightFlagMcp.Core`;
- `src/BrightFlagMcp.Api`;
- `src/BrightFlagMcpServer`; and
- `test/BrightFlagMcp.Tests`.

## Acquire and review the OpenAPI snapshot first

Before defining an API-derived contract, add and run an explicit administrative command that fetches
`/v3/api-docs/external` from the configured BrightFlag origin only. It must use bounded HTTPS, cap
the response size, reject cross-origin redirects, write atomically, and print a review diff. Use the
public `https://app.brightflag.com` origin for the initial snapshot; do not request or use a tenant
credential.

Check the reviewed snapshot into the repository. Validate that:

- the five required `operationId` values exist at the expected method and path;
- `Batch` provides `batchID`, `batchCreated`, and `customerBatchID`;
- `BatchDTO` provides `batchID` and an `invoices` array of objects carrying `invoiceID`;
- `InvoiceSummaryAPI` provides `invoiceID`, `invoiceGroupId`, `invoiceNumber`, `invoiceDate`,
  `invoiceStatus`, `invoiceStatusChangeTimestamp`, `approvedGrossTotal`, `originalGrossTotal`,
  `taxTotal`, `exposurePercentage`, `currencyIsoCode`, `invoiceCurrencyDetails`, `vendorLink`, and
  `matterLink`;
- `getInvoiceSummaryList` accepts `invoiceStatus`, `invoiceStatusChangeStartDate`,
  `invoiceStatusChangeEndDate`, `includePreviousDrafts`, `paging.pageSize`, and
  `paging.pageNumber`; and
- `InvoicePaymentStatus` provides `invoiceID`, `invoiceNumber`, `vendorRef`, `paymentRef`,
  `paymentStatus`, `paymentAmount`, `paymentDate`, and `paymentComment`, and requires
  `paymentStatus`.

If a required operation or field is absent, stop and report the incompatibility. Do not substitute a
similar field or continue by guessing. Runtime startup will read this checked-in snapshot only; it
must never fetch the OpenAPI document.

Define immutable JSON-compatible contracts for:

- `ApprovedInvoiceQuery`, carrying only a bounded lookback window and an optional opaque cursor;
- `ApprovedInvoicePage`, carrying a bounded result list and an optional next cursor;
- `ApprovedInvoice`, joining accounts-payable batch release evidence to invoice summary detail;
- `ApBatchRelease`, carrying batch identity, creation instant, and the released invoice identifier;
- `InvoiceSummary`, carrying the reviewed projection of invoice detail;
- `InvoiceCurrency`, carrying ISO currency code plus the reviewed FX detail;
- `MoneyAmount`, using decimal plus ISO 4217 currency code;
- `VendorReference` and `MatterReference`;
- `InvoicePaymentPlan`;
- `InvoicePaymentConfirmation`;
- `PaidInvoiceReceipt`;
- normalized BrightFlag error;
- caller and tenant identity;
- authorization decision;
- audit outcome; and
- `OntologySchemaDocument`.

Use stable invoice identity `InvoiceId` for the BrightFlag revision identifier and
`InvoiceGroupId` for the logical invoice, and keep both. Use `BatchId` for accounts-payable batch
identity. Model vendor identity as `VendorRef`, not a display name.

Amounts are decimal in the domain. BrightFlag transports `paymentAmount` as a string; that
conversion belongs in the adapter added in a later stage, not in the domain contract.

The contracts must not include:

- an arbitrary BrightFlag origin, path, or operation field;
- raw query-string, HTTP-method, or header fields;
- generic entity or operation abstractions;
- page numbers or page sizes on any caller-facing type;
- caller-supplied roles;
- credential or bearer-token values; or
- an extensible property bag that bypasses validation.

Cursors are opaque to the caller. Use a stateless, signed, version-tagged keyset cursor containing
the fixed query window, the last emitted stable sort key, caller binding, contract version,
configuration fingerprint, snapshot fingerprint, and expiry. It must not contain credentials or
full response payloads. Reject it for a different caller, window, contract, configuration, or
snapshot, or after expiry.

Order results by a documented immutable key with an explicit tie-breaker. Resuming may repeat
bounded upstream reads; discard every row at or before the last emitted key so the server does not
return a duplicate. Do not claim snapshot isolation: document that upstream changes inside the
fixed window can appear on a later page, while the keyset prevents already emitted rows from being
returned again.

Add deterministic JSON serialization with ordinal property/order rules, normalized line endings,
and no timestamps, random values, machine paths, hostnames, or environment data in generated schema
artifacts.

Other than the explicit administrative OpenAPI snapshot fetch, do not add BrightFlag calls, MCP
packages, payment logic, Docker, or workflows in this stage.

## Acceptance criteria

- `dotnet restore --locked-mode` succeeds.
- `dotnet build --no-restore` succeeds with zero warnings.
- `dotnet test --no-build` proves stable identities, decimal/currency validation, closed JSON
  contracts, unknown-field rejection, cursor opacity and tamper rejection, and byte-identical
  deterministic serialization.
- The checked-in snapshot passes all required operation-and-field validation, and runtime startup
  performs no OpenAPI network call.
- `dotnet format --verify-no-changes` succeeds.
- Tests prove no contract can carry an arbitrary origin, raw HTTP detail, credential, page number,
  or role.

Commit locally with a focused message. Do not push.
