# Prompt 1 — Solution scaffold and BrightFlag contracts

Using the reusable contract, implement Stage 1: the .NET solution and stable contracts for only the
three requested capabilities. Do not contact BrightFlag or expose MCP yet.

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

Cursors are opaque to the caller. Define the cursor as a signed, version-tagged, tamper-evident
token that encodes only the query window and the adapter's position within it. A cursor must not be
accepted for a different query window, a different caller, or a different contract version.

Add deterministic JSON serialization with ordinal property/order rules, normalized line endings,
and no timestamps, random values, machine paths, hostnames, or environment data in generated schema
artifacts.

Do not add BrightFlag calls, OpenAPI acquisition, MCP packages, payment logic, Docker, or workflows
in this stage.

## Acceptance criteria

- `dotnet restore --locked-mode` succeeds.
- `dotnet build --no-restore` succeeds with zero warnings.
- `dotnet test --no-build` proves stable identities, decimal/currency validation, closed JSON
  contracts, unknown-field rejection, cursor opacity and tamper rejection, and byte-identical
  deterministic serialization.
- `dotnet format --verify-no-changes` succeeds.
- Tests prove no contract can carry an arbitrary origin, raw HTTP detail, credential, page number,
  or role.

Commit locally with a focused message. Do not push.
