# Prompt 3 — Configure the fixed BrightFlag operations

Using the reusable contract, implement Stage 3: secret-free configuration, a reviewed OpenAPI
snapshot, and a local fake BrightFlag server covering exactly five operations.

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

Add an explicit administrative command that fetches `/v3/api-docs/external` from the configured
origin only. It must use bounded HTTPS, cap the response size, reject cross-origin redirects, write
atomically, and print a review diff.

Validate from the reviewed snapshot that:

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

If a required operation or field is absent for the tenant's BrightFlag release, fail configuration.
Do not substitute a similar field, and do not fall back to an operation outside the five.

Runtime startup reads the checked-in snapshot only. It never fetches the OpenAPI document and never
grows the MCP surface.

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
- Snapshot validation proves every required operation and field named above.
- Runtime startup performs no OpenAPI network call.
- Bearer-token redaction is asserted in logs, errors, and audit records.
- Fake-server tests assert exact paths and reject every other route.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` on the implementation pull request and record why the
server binds five fixed operations instead of generating tools from the BrightFlag OpenAPI
document. Do not push unless requested.
