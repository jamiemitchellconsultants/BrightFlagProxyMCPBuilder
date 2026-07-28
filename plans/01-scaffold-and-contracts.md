# Stage 01 — Acquire the OpenAPI snapshot, scaffold the solution, define contracts

Source: `BrightFlagProxyMCPBuilder/prompts/01-solution-scaffold-and-invoice-contracts.md`

## Context

Contracts are defined *after* the authoritative API document is in hand, not before — this ordering
was a deliberate correction recorded in Narrative entry 4. Everything downstream keys off the
snapshot: the evidence rule cites its field names, the cursor carries its fingerprint, the ontology
document derives provenance from it. Getting a contract wrong here is expensive in every later
stage, so this stage does exactly two things: prove the API is what the prompts claim, and freeze
the shapes.

No BrightFlag tenant call, no MCP package, no payment logic.

## Preconditions

Reusable contract submitted and acknowledged. Repository is the current state: one commit, MIT
`LICENSE`, two-line `README.md`, no `.gitignore`, no solution. .NET SDK 10.0.201 present.

## Scope in

Snapshot acquisition and validation; solution scaffold; immutable contracts; deterministic JSON;
the signed keyset cursor.

## Scope explicitly out

MCP packages. BrightFlag runtime calls. Payment logic. Docker. CI workflows. Configuration binding
(Stage 3). The evidence rule itself (Stage 4) — only the *shapes* it will operate on.

## Work items

### 1. The administrative snapshot command

Add a CLI verb on the server host, mirroring the shape Prompt 7 later mandates for `schema check`:

```bash
dotnet run --project src/BrightFlagMcpServer -- snapshot refresh
```

Requirements, all testable:

- HTTPS only, to `https://app.brightflag.com` for this initial acquisition. No credential is sent
  and none is requested.
- Bounded: request timeout, and a hard response-size cap that aborts rather than truncates.
- Redirects to a different origin are rejected outright.
- Writes atomically (temp file + move), never a partial document.
- Prints a review diff against the existing checked-in copy and exits non-zero when it differs, so
  a refresh is always a reviewed act.
- Does not start an MCP listener or open a transport.

Snapshot lands at `openapi/brightflag-external.json`, checked in. Its fingerprint is a SHA-256 over
the stored bytes, **computed at load**, not kept in a sidecar — a stored fingerprint is one more
thing that can drift.

**Store it canonicalized, not as served.** BrightFlag returns the document minified onto a single
line: 171,651 bytes, zero newlines. A line diff over that carries no information, so the review diff
the command prints would be unreadable and every later snapshot change would be unreviewable in a
pull request. Run the fetched bytes through the same canonical encoding the contracts use — ordinal
property ordering, two-space indent, LF — before writing, which yields 12,715 diffable lines. Do not
write a second encoding for this purpose.

Fingerprint the canonical bytes. That has a second effect worth having: the fingerprint moves when
the document's content moves, rather than when BrightFlag merely reorders the properties it emits.
It gates cursor validity in Stage 5 and schema generation in Stage 7, so churn there is not free.

State plainly, in the code and in the stage report, that the stored document is deliberately not
byte-identical to the served one.

Add `.gitattributes` pinning the snapshot to LF. The fingerprint is over exact stored bytes, so a
CRLF checkout fails validation — and Stage 10 produces a Windows clone. The same file covers the
generated schema document Stage 7 checks in.

### 2. Snapshot validation (runs at startup and in tests)

Fail loudly if any of these is absent. Verified present in the live document at planning time, so a
failure here means the tenant release moved, not that the plan was wrong:

- the four `operationId`s at exactly the expected method and path;
- `Batch`: `batchID`, `batchCreated`, `customerBatchID`;
- `BatchDTO`: `batchID`, `invoices[]` carrying `invoiceID`;
- `InvoiceSummaryAPI`: `invoiceID`, `invoiceGroupId`, `invoiceNumber`, `invoiceDate`,
  `invoiceStatus`, `invoiceStatusChangeTimestamp`, `approvedGrossTotal`, `originalGrossTotal`,
  `taxTotal`, `exposurePercentage`, `currencyIsoCode`, `invoiceCurrencyDetails`, `vendorLink`,
  `matterLink`;
- `getInvoiceSummaryList` parameters: `invoiceStatus`, `invoiceStatusChangeStartDate`,
  `invoiceStatusChangeEndDate`, `includePreviousDrafts`, `paging.pageSize`, `paging.pageNumber`;
- `InvoicePaymentStatus`: `invoiceID`, `invoiceNumber`, `vendorRef`, `paymentRef`, `paymentStatus`,
  `paymentAmount`, `paymentDate`, `paymentComment`, with `paymentStatus` required.

Runtime startup reads the checked-in file only and must never fetch. Prove that with a test that
asserts no `HttpClient` is constructed on the startup path.

### 3. Solution scaffold

`global.json` (10.0.x feature band), `Directory.Build.props` (nullable, deterministic builds,
analyzers, `TreatWarningsAsErrors`), `Directory.Packages.props` (central pinning,
`RestorePackagesWithLockFile`), checked-in `packages.lock.json`, `.editorconfig`, `.gitignore`.
`LICENSE` already exists and is MIT — verify, do not duplicate.

Projects, exactly as the prompt names them:

- `src/BrightFlagMcp.Core` — contracts, cursor, deterministic JSON, later the evidence rule.
- `src/BrightFlagMcp.Api` — later the BrightFlag adapter and configuration binding.
- `src/BrightFlagMcpServer` — composition root and CLI verbs.
- `test/BrightFlagMcp.Tests`.

### 4. Contracts

Immutable `record` types, JSON-compatible, closed to unknown members
(`JsonUnmappedMemberHandling.Disallow`), no property bags:

`ApprovedInvoiceQuery`, `ApprovedInvoicePage`, `ApprovedInvoice`, `ApBatchRelease`,
`InvoiceSummary`, `InvoiceCurrency`, `MoneyAmount`, `VendorReference`, `MatterReference`,
`InvoicePaymentPlan`, `InvoicePaymentConfirmation`, `PaidInvoiceReceipt`, normalized BrightFlag
error, caller and tenant identity, authorization decision, audit outcome, `OntologySchemaDocument`.

Identity rules: `InvoiceId` = BrightFlag revision id, `InvoiceGroupId` = logical invoice — keep
both, never collapse. `BatchId` for batch identity. Vendor identity is `VendorRef`, never a display
name.

`MoneyAmount` is `decimal` + ISO 4217 code. **Deserialize upstream JSON `number` straight to
`decimal`, never via `double`** — `approvedGrossTotal`, `originalGrossTotal`, `taxTotal`, and
`exposurePercentage` are all JSON numbers in the reviewed document. BrightFlag transports
`paymentAmount` as a *string*; that conversion belongs in the Stage 5/6 adapter, not here.

Contracts must not carry: an origin, path, or operation field; raw query/method/header fields;
generic entity or operation abstractions; page numbers or sizes on any caller-facing type;
caller-supplied roles; credential values; or an extensible bag that bypasses validation.

### 5. The keyset cursor

Opaque to callers, stateless, signed, version-tagged. Payload: contract version tag; the fixed query
window (start/end epoch ms); the last emitted stable sort key; caller binding; configuration
fingerprint; snapshot fingerprint; signing-key identifier; expiry. It carries no credential and no
response payload.

Signing: HMAC-SHA256, base64url, key selected **by the identifier inside the cursor** so more than
one key verifies at once. That indirection is not decoration — it is the mechanism Stage 9's
rotation runbook depends on, and Stage 11 point 12 audits it.

Rejection cases: different caller, window, contract version, configuration fingerprint, or snapshot
fingerprint; expired; unknown key identifier; any tampering.

Sort key: `(batchCreated, batchID, invoiceID)` ascending, with `invoiceID` as the explicit
tie-breaker. All three are immutable upstream. Document honestly that resuming may repeat bounded
upstream reads, that every row at or before the last emitted key is discarded so nothing is returned
twice, and that **snapshot isolation is not claimed** — upstream changes inside the fixed window can
surface on a later page.

Stage 1 may sign with a fixed test key, but the implementation must read the key through the
abstraction Stage 3 will supply — not hard-code it as the only possible source.

### 6. Deterministic JSON

One canonical writer: ordinal property ordering, `\n` line endings, invariant number and date
formatting, no timestamps, random values, machine paths, hostnames, or environment data. Stage 7's
fingerprint depends on this being exactly right.

## Tests

Stable identities; decimal and currency validation; closed contracts rejecting unknown fields;
cursor opacity and tamper rejection; cursor verification against a named current *or* previous
signing key and rejection of an unknown key id; byte-identical serialization across repeated runs;
snapshot validation passing for every required operation and field; no OpenAPI network call on the
startup path; and — explicitly — that no contract can carry an arbitrary origin, raw HTTP detail,
credential, page number, or role.

## Acceptance checks

```bash
dotnet restore --locked-mode
```

```bash
dotnet build --no-restore
```

```bash
dotnet test --no-build
```

```bash
dotnet format --verify-no-changes
```

Build must produce zero warnings.

The checked-in snapshot must additionally be reviewable as a line diff, and a test must prove its
fingerprint is unchanged when the served document differs only in property order.

## Stage boundary

Report: files changed, commands run and their output, the snapshot's byte size and fingerprint, and
any field whose presence was marginal. Commit locally — suggested message
`Scaffold solution, acquire reviewed OpenAPI snapshot, define invoice contracts`. Do not push.

**Do not begin Stage 2.** Do not add MCP packages, configuration binding, the fake server, or the
evidence rule.

## Risks

- `dotnet restore --locked-mode` fails on the first run because no lock file exists yet; generate it
  with a normal restore, check it in, then re-run locked. Expected, not a defect.
- The served document is 171,651 bytes and the canonical form is roughly twice that, since
  canonicalizing adds indentation and newlines. The response cap applies to the fetched bytes, but
  set it well clear of both (4 MiB) so modest tenant-side growth does not fail the fetch spuriously.
- `Batch.customerID` exists and is the only tenant-ish discriminator in the reviewed payload. Model
  it, but claim nothing about cross-tenant enforcement here — Stage 11 point 32 explicitly relaxes
  that requirement given the payload.
