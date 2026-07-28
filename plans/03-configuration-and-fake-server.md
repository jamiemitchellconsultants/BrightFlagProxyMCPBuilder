# Stage 03 — Configure the fixed BrightFlag operations, and build the fake server

Source: `BrightFlagProxyMCPBuilder/prompts/03-fixed-brightflag-endpoint-configuration.md`

## Context

This stage draws the boundary the rest of the build lives inside: one origin, four operations, two
independent secrets, and a set of ceilings. It also builds the loopback fake BrightFlag server that
every later stage tests against — nothing after this point ever needs a live tenant.

The reviewed OpenAPI document declares `servers: [{"url": "/"}]`, so the origin genuinely cannot be
inferred and must come from configuration. That is a fact about the document, not a design
preference.

## Preconditions

Stage 2's Narrative installation **published and merged to the default branch**. Stage 1 committed.

## Scope in

Strongly-typed configuration and its validation; secret-provider contracts for two independent
secrets; the checked-in snapshot's runtime path; the fake BrightFlag server and its fixtures.

## Scope explicitly out

The evidence rule (Stage 4). The adapter that calls BrightFlag (Stage 5). MCP hosting (Stage 8).
Any real outbound call.

## Work items

### 1. Configuration

Bound to a closed options tree in `src/BrightFlagMcp.Api`, validated at startup, rejecting unknown
properties:

- **Origin** — exactly one per deployment, outside MCP arguments, e.g. `https://app.brightflag.com`
  or a tenant regional/sandbox host.
- **Bearer-token provider reference** and token lifetime handling.
- **Cursor-signing key provider reference**, current key identifier, and the set of prior key
  identifiers still valid for verification — consumed by Stage 1's cursor.
- **Tenant/customer boundary** the caller may act within.
- **Approved-status allow-list** — ships empty; Stage 4 makes empty a startup failure.
- **Ceilings** — request timeout, concurrency, response bytes, page size, result count.
- **Lookback** — default 7 days, maximum 31.
- **Summary-window margin** — see "Decisions carried in".
- **Snapshot path** — a repository-relative path only.
- **The four fixed relative paths**, as literals:
  - `/api/ap-batch/v1/{startEpochTime}/{endEpochTime}`
  - `/api/ap-batch/v1/batch/{batchID}/invoices`
  - `/api/v1/invoice-summary`
  - `/api/v1/invoicePayment/invoice-payment-status`

### 2. What configuration must reject

Each is a test case, not a comment:

- any fifth operation — explicitly including `getAPBatches`, `downloadInvoiceDocument`,
  `downloadAPBatchFile`;
- caller-controlled origins, paths, or query strings;
- URL user-info, fragments, wildcards, traversal, or embedded credentials;
- plain HTTP, except loopback for the fake server;
- redirects to another origin;
- the bearer token or cursor-signing key appearing in configuration files, URLs, or logs;
- a production profile with no authentication;
- an unbounded or absent tenant boundary;
- a missing cursor-signing key or current key identifier;
- unknown configuration properties.

### 3. Secret providers — two, independent

Separate contracts for the BrightFlag bearer token and the cursor-signing key. Each gets an
environment provider, a file-reference provider, and a fake provider for tests. No vendor-specific
vault.

Both are redacted in every log, error, trace, and audit record, and redaction is *asserted*, not
assumed.

The cursor-signing key must always come from configuration and must **never** be generated
in-process at startup. It has to be identical across every instance and stable across restarts,
because Stage 1's cursor is stateless and Stage 8's transport is session-affinity-free. A key
generated per process silently breaks pagination the moment there is more than one instance — which
is exactly why Stage 11 point 12 audits it.

### 4. OpenAPI snapshot at runtime

Configuration may select the checked-in snapshot's repository path but may not replace its contents
or point outside the repository. Startup reads it and only it. The Stage 1 administrative refresh
command remains the only network path that can update it. If a later tenant-release review finds a
required operation or field missing, **fail configuration** — do not substitute a similar field or
fall back to an operation outside the four.

### 5. Fake BrightFlag server

Loopback only. It lives in `test/BrightFlagMcp.Tests` under `Fakes/`, deliberately **not** in a
`src/` project — that placement is what guarantees it cannot ship in the Stage 9 container image, a
property Stage 9 and Stage 10 both have to prove.

Routes, and nothing else (everything else returns 404):

- `GET /v3/api-docs/external` — serves the checked-in snapshot;
- `GET /api/ap-batch/v1/{startEpochTime}/{endEpochTime}`;
- `GET /api/ap-batch/v1/batch/{batchID}/invoices`;
- `GET /api/v1/invoice-summary` — with *working* `invoiceStatus` filtering, date-window filtering,
  and paging behaviour;
- `POST /api/v1/invoicePayment/invoice-payment-status`.

It requires a bearer token and returns the documented `ErrorResponse` shape — `message`, `errors`,
`errorCategory`, `errorCode`, `status`, `metadata` — for 400, 403, 404, 409, and 500.

Fixtures are synthetic invoices, vendors, matters, batches, and payments. Two are mandatory and
load-bearing for Stage 4:

- an invoice that **appears in a batch but whose status is outside the allow-list**;
- an invoice whose **status is inside the allow-list but which was never batched**.

Add, while the fixtures are being built, a case whose `invoiceStatusChangeTimestamp` sits *outside*
the batch window — it is the fixture that will exercise Stage 5's summary-window margin and the
reconciliation-gap path.

The fake must record the exact method, path, query, headers, body, and **call count** of everything
it receives, because from Stage 5 onward that recording is the evidence.

## Decisions carried in

- **Summary-window margin.** `getInvoiceSummaryList` has no `invoiceID` filter, so Stage 5's join
  must enumerate summaries over a *window* and filter locally. Release date and status-change date
  are different axes. Configure `SummaryWindowMarginDays` with its own default and maximum,
  mirroring the lookback, applied to widen the batch window when querying
  `invoiceStatusChangeStartDate/EndDate`. Introducing it here keeps Stage 5 free of new
  configuration surface.
- **Fake server placement** in the test project, per above.

## Tests

Configuration resolves exactly four operations against one origin. Every rejection case above fails
— unsafe URLs, a fifth operation, secrets in configuration, unknown properties, an absent tenant
boundary, a missing signing key or key identifier. Stage 1's snapshot validation still proves every
required operation and field. Startup performs no OpenAPI network call. Bearer-token *and*
cursor-signing-key redaction asserted in logs, errors, and audit records. The signing key is proven
to come from configuration rather than in-process generation, and **two independently configured
instances sharing the same configured key verify each other's cursors**. Fake-server tests assert
exact paths and reject every other route.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

## Stage boundary

Commit locally. Suggested message: `Configure four fixed BrightFlag operations and add fake server`.

This is the first `narrative-required` stage. When published, the PR records **why the server binds
four fixed operations instead of generating tools from the BrightFlag OpenAPI document** — Context:
114 operations in the reviewed document, three requested capabilities; Decision: bind four, fail
configuration on a fifth; Consequences: every widening becomes a reviewed contract change.

Do not push unless requested. **Do not begin Stage 4** — the evidence rule is a separate stage even
though the allow-list is configured here.

## Risks

- The allow-list ships empty and Stage 4 makes empty fatal at startup. Between Stage 3 and Stage 4
  the tests must supply a value explicitly; do not let a convenience default creep in.
- Redaction is easy to assert shallowly. Test it on structured log state and exception `ToString()`,
  not only on the formatted message.
