# Stage 07 — Report the BrightFlag ontology schema

Source: `BrightFlagProxyMCPBuilder/prompts/07-ontology-schema-reporting.md`

## Context

The capability the README lists first is built third-from-last, deliberately (`DESIGN-CALLS.md` §3).
The schema is generated from settled contracts and the reviewed snapshot; generating it earlier
guarantees churn, because every later stage that adds a field or renames an entity invalidates it. A
schema regenerated five times teaches nothing about determinism. Generated once, against finished
contracts, with a build-time drift gate, it does.

The server **never calls an ontology service.** It publishes a document; registration is somebody
else's job. No outbound ontology client, no registration tool, no push notification, no callback.

## Preconditions

Stage 6 committed. All four tools and the resource shape are now settled — this is the first moment
that is true.

## Scope in

The `OntologySchemaDocument` generator; canonical encoding and fingerprint; tool and resource
exposure; the checked-in copy and its drift gate; the `schema check` command.

## Scope explicitly out

Any outbound call. Any caller ability to upload, patch, override, or execute mappings. Live data of
any kind.

## Work items

### 1. Content

Generated from checked-in contracts and the reviewed snapshot only:

- schema identifier, contract version, and the snapshot fingerprint it was generated from;
- input and output JSON Schemas for all four tools and the one resource;
- semantic entity definitions for `Invoice`, `InvoiceRevision`, `ApBatch`, `ApBatchRelease`,
  `Vendor`, `Matter`, `InvoiceCurrency`, `PaymentInstruction`, `PaymentStatusUpdate`;
- field-level definitions carrying name, type, optionality, unit or currency semantics, and the
  BrightFlag source field;
- relationships: revision→logical invoice, invoice→batch release, invoice→vendor, invoice→matter,
  payment→invoice;
- the vocabulary for **`approved for payment`** and **`paid`**, stated as definitions *with their
  evidence requirements* rather than as status strings — that phrasing is the point: the document
  must not become a place where a status vocabulary gets published as though it were universal;
- provenance per entity and field, naming the BrightFlag `operationId` and schema component it came
  from — and only ever the four fixed operations;
- a deterministic fingerprint over the canonical document **with the fingerprint property omitted**
  (the recursive definition was a defect corrected in Narrative entry 3).

State explicitly in the document that the approved-status allow-list is **tenant configuration**,
and name it as a required input an ingesting ontology service must obtain separately. Do not embed
the configured values.

### 2. Determinism and safety

- One canonical JSON encoding used for both fingerprinting and output.
- Ordinal ordering, normalized line endings.
- Byte-identical across runs, machines, locales, and time zones.
- No timestamps, hostnames, origins, tenant identifiers, tokens, file paths, environment values, or
  live BrightFlag data.
- Synthetic examples only — no real invoice number, vendor, matter, amount, or payment reference.

### 3. Exposure

`brightflag_get_ontology_schema` and the read-only resource `brightflag://ontology-schema` return
**identical bytes**. Callers may read; they may not upload, patch, override, or execute mappings.

Check a generated copy into the repository and **fail the build** when the generated document and
the checked-in copy differ, so a contract change cannot land without a visible schema diff.

### 4. Drift-check command

```bash
dotnet run --project src/BrightFlagMcpServer -- schema check
```

Regenerates in memory, compares byte-for-byte against the checked-in copy, prints a diff on
mismatch, exits non-zero — **without** starting an MCP listener, opening a transport, or contacting
BrightFlag. This is the same gate CI runs in Stage 9 and the mechanism Stage 11 invokes directly, so
its exit codes and output format are part of the contract, not an implementation detail.

## Tests

- byte-identical output across repeated generation and across at least two locales and time zones;
- tool and resource return identical bytes;
- the document round-trips through JSON Schema validation;
- **every declared entity field traces to a field present in the reviewed snapshot** — this is the
  test that stops invented vocabulary;
- the drift check fails when a contract changes without regeneration, and writes nothing when it
  fails;
- no network call and no ontology-service reference exists anywhere in the server.

## Acceptance checks

- The schema is generated, deterministic, checked in, and drift-gated.
- `schema check` exits zero against the checked-in document, and non-zero with a printed diff when
  the checked-in copy is stale.
- The server has no outbound ontology dependency.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
dotnet run --project src/BrightFlagMcpServer -- schema check
```

## Stage boundary

Commit locally. Suggested message: `Generate deterministic ontology schema with drift gate`.

`narrative-required` when published. Decision to record: **publishing a schema for ingestion rather
than registering with an ontology service directly.** Context — a registration client would put an
outbound dependency and a second trust relationship inside a server whose whole design is a closed
surface. Decision — publish a deterministic document; registration is a separate system's job.
Consequences — the ingesting service must poll or be handed the document; this server gains no
knowledge of who consumes it.

Do not push unless requested. **Do not begin Stage 8.**

## Risks

- Determinism failures usually come from dictionary iteration order, culture-sensitive number
  formatting, or `\r\n` on a checkout with `core.autocrlf`. Add a `.gitattributes` rule pinning the
  checked-in document to `LF` so a Windows clone — which Stage 10 will produce — does not fail the
  drift gate spuriously.
- JSON Schema generation from records can leak assembly or namespace detail into `$id`. Assert the
  document contains no machine paths.
- The "every field traces to the snapshot" test is the most valuable one here and the easiest to
  write as a tautology. Drive it from the snapshot document, not from the same constants the
  generator uses.
