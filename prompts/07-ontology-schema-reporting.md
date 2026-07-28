# Prompt 7 — Report the BrightFlag ontology schema

Using the reusable contract, implement Stage 7: a deterministic schema document that a separate
ontology service can ingest.

This server never calls an ontology service. It reports a document; registration is somebody else's
job. Do not add an outbound ontology client, a registration tool, a push notification, or a
callback.

## Content

Generate `OntologySchemaDocument` from checked-in contracts and the reviewed OpenAPI snapshot only:

- a schema identifier, contract version, and the snapshot fingerprint it was generated from;
- input and output JSON Schemas for all four MCP tools and the one resource;
- semantic entity definitions for `Invoice`, `InvoiceRevision`, `ApBatch`, `ApBatchRelease`,
  `Vendor`, `Matter`, `InvoiceCurrency`, `PaymentInstruction`, and `PaymentStatusUpdate`;
- field-level definitions carrying name, type, optionality, unit or currency semantics, and the
  BrightFlag source field;
- relationships, including revision-to-logical-invoice, invoice-to-batch release,
  invoice-to-vendor, invoice-to-matter, and payment-to-invoice;
- the vocabulary this server uses for `approved for payment` and `paid`, stated as definitions with
  their evidence requirements rather than as status strings;
- provenance per entity and field, naming the BrightFlag `operationId` and schema component it came
  from; and
- a deterministic fingerprint over the canonical document with the fingerprint property omitted.

State explicitly in the document that the approved-status allow-list is tenant configuration, and
name it as a required input an ingesting ontology service must obtain separately. Do not embed the
configured values.

## Determinism and safety

The document must:

- define one canonical JSON encoding used for fingerprinting and output;
- serialize with ordinal ordering and normalized line endings;
- be byte-identical across runs, machines, locales, and time zones;
- contain no timestamps, hostnames, origins, tenant identifiers, tokens, file paths, environment
  values, or live BrightFlag data; and
- use synthetic examples that contain no real invoice number, vendor, matter, amount, or payment
  reference.

Expose it through `brightflag_get_ontology_schema` and the read-only resource
`brightflag://ontology-schema`, returning identical bytes from both. Check a generated copy into
the repository and fail the build when the generated document and the checked-in copy differ, so a
contract change cannot land without a visible schema diff.

Callers may read the schema. They may not upload, patch, override, or execute mappings.

## Drift-check command

Add an administrative `schema check` command on the server host, invoked as
`dotnet run --project src/BrightFlagMcpServer -- schema check`. It regenerates the document in
memory, compares it byte-for-byte against the checked-in copy, prints a diff on mismatch, and exits
non-zero without starting an MCP listener, opening a transport, or contacting BrightFlag. This is
the same drift gate CI runs in Prompt 9 and the mechanism the independent audit in Prompt 10 invokes
directly.

## Tests

Prove:

- byte-identical output across repeated generation and across at least two locales and time zones;
- tool and resource return identical bytes;
- the document round-trips through JSON Schema validation;
- every declared entity field traces to a field present in the reviewed snapshot;
- the drift check fails when a contract changes without regenerating; and
- no network call and no ontology-service reference exists anywhere in the server.

## Acceptance criteria

- The schema is generated, deterministic, checked in, and drift-gated.
- `schema check` exits zero against the checked-in document and non-zero, with a printed diff, when
  the checked-in copy is stale.
- The server has no outbound ontology dependency.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the decision to publish a schema for ingestion
rather than register with an ontology service directly. Do not push unless requested.
