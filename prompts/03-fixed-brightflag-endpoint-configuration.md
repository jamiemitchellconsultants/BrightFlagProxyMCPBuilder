# Prompt 3 — Configure the fixed BrightFlag operations

Using the reusable contract, implement Stage 3: secret-free configuration against the reviewed
OpenAPI snapshot acquired in Prompt 1, and a local fake BrightFlag server covering exactly five
operations.

Configure:

- one BrightFlag origin per deployment, outside MCP arguments, for example
  `https://app.brightflag.com` or the tenant's regional or sandbox host;
- the bearer-token provider reference and token lifetime handling;
- the customer or tenant boundary the caller may act within;
- the reviewed approved-status allow-list, described in Prompt 4;
- request timeout, concurrency, response-byte, page-size, and result-count limits;
- the default and maximum accounts-payable batch lookback, 7 and 31 days;
- the OpenAPI snapshot path; and
- the fixed relative operation paths:
  - `/api/ap-batch/v1`
  - `/api/ap-batch/v1/{startEpochTime}/{endEpochTime}`
  - `/api/ap-batch/v1/batch/{batchID}/invoices`
  - `/api/v1/invoice-summary`
  - `/api/v1/invoicePayment/invoice-payment-status`

Reject:

- any sixth operation, including `downloadInvoiceDocument` and `downloadAPBatchFile`;
- caller-controlled origins, paths, or query strings;
- URL user information, fragments, wildcards, traversal, or embedded credentials;
- plain HTTP except for the loopback fake server;
- redirects to another origin;
- bearer tokens in configuration files, URLs, or logs;
- production profiles using no authentication;
- an unbounded or absent tenant boundary; and
- unknown configuration properties.

Support secret-provider contracts for the BrightFlag bearer token, including an environment
provider, a file-reference provider, and a fake provider for tests. Redact the token in every log,
error, trace, and audit record; assert redaction in tests. Do not implement a vendor-specific
vault.

## OpenAPI snapshot

Use the checked-in, reviewed snapshot acquired and validated in Prompt 1. Configuration may select
its repository path but may not replace its contents or point outside the repository. Runtime
startup reads that snapshot only: it never fetches the OpenAPI document and never grows the MCP
surface.

The administrative refresh command created in Prompt 1 remains the only network path allowed to
update the snapshot. If a later tenant-release review finds a required operation or field absent,
fail configuration; do not substitute a similar field or fall back to an operation outside the
five.

## Fake BrightFlag server

Implement a loopback fake server exposing only:

- `GET /v3/api-docs/external` serving the checked-in snapshot;
- the three accounts-payable batch reads;
- `GET /api/v1/invoice-summary` with working `invoiceStatus`, date-window, and paging behavior; and
- `POST /api/v1/invoicePayment/invoice-payment-status`.

It must require a bearer token, return the documented `ErrorResponse` shape for 400, 403, 404, 409,
and 500, and reject every other route with 404. Use synthetic invoices, vendors, matters, batches,
and payments. Include a fixture whose invoice appears in a batch but whose status is outside the
allow-list, and a fixture whose status is inside the allow-list but which was never batched.

## Acceptance criteria

- Configuration resolves exactly five fixed operations against one origin.
- Unsafe URLs, a sixth operation, secrets in configuration, unknown properties, and an absent
  tenant boundary all fail.
- Prompt 1's snapshot validation still proves every required operation and field.
- Runtime startup performs no OpenAPI network call.
- Bearer-token redaction is asserted in logs, errors, and audit records.
- Fake-server tests assert exact paths and reject every other route.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` on the implementation pull request and record why the
server binds five fixed operations instead of generating tools from the BrightFlag OpenAPI
document. Do not push unless requested.
