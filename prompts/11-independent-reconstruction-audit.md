# Prompt 11 — Independent reconstruction audit

Use this prompt as a fresh task or, ideally, with a different coding agent.

Audit the completed BrightFlag proxy MCP server against the reusable contract and all staged
prompts. Do not add new capabilities. Fix only confirmed defects.

## Verify the capability ceiling

Confirm independently:

1. The server advertises exactly four tools and one ontology resource, and startup fails if that
   changes.
2. Runtime configuration resolves exactly the four fixed BrightFlag operations against one origin.
3. No caller can supply an origin, path, operation, method, header, query string, page number,
   status allow-list entry, role, or credential.
4. The OpenAPI snapshot cannot create a tool, field, or operation at runtime, and no runtime code
   path fetches `/v3/api-docs/external`.
5. The unwindowed `getAPBatches` batch listing, `downloadInvoiceDocument`, `downloadAPBatchFile`,
   SCIM, matters, vendors, allocations, purchase orders, pay sites, tax rates, budgets, legal
   service requests, and reporting are all absent.
6. No outbound ontology-service call, registration hook, or callback exists.

## Verify approved-for-payment evidence

7. An invoice is reported only with both accounts-payable batch release and an allow-listed status.
8. The allow-list is tenant configuration, has no shipped default, fails startup when empty, and is
   matched exactly and case-sensitively.
9. Batched-but-not-allow-listed and allow-listed-but-not-batched fixtures are both excluded, and
   the excluded count with observed statuses is reported.
10. Lookback defaults to 7 days, never exceeds 31, and converts to epoch milliseconds correctly
    across time zones and daylight-saving boundaries.
11. `includePreviousDrafts` is false and at most one revision per `invoiceGroupId` is returned, with
    a typed conflict when a batch releases two revisions.
12. Cursors are opaque, signed, expiring, bound to caller and window, and reject tampering, replay
    with a changed window, and cross-caller use. Verification accepts a current or documented prior
    signing-key identifier and rejects an unrecognized one, and a runbook procedure exists for
    rotating the key without an undocumented hard cutover. The signing key is sourced from a
    configured secret provider rather than generated in-process, is redacted everywhere the
    BrightFlag bearer token is redacted, and is identical across every instance in a multi-instance
    deployment.
13. Fan-out, page, byte, result, timeout, and concurrency ceilings fail closed.
14. Amounts and currencies are carried unconverted, never summed across currencies, and
    `exposurePercentage` is surfaced rather than applied.
15. A batch-released invoice missing from the summary read is reported as a reconciliation gap, not
    fabricated or dropped.

## Verify the payment mutation

16. Planning re-verifies approval with a fresh read and validates amount tolerance, currency, dates,
    duplicate `paymentRef`, and already-paid state before any write.
17. Plan tokens are cryptographically random, expiring, single-use, race-safe, and bound to caller,
    tenant, invoice identity, amount, currency, `paymentRef`, and contract version.
18. Execution accepts only a plan token plus explicit confirmation, and no planned field can be
    overridden at execution.
19. Authorization is evaluated again at execution and denial occurs before the service credential is
    used.
20. `paymentStatus` can only be `PAID`; `PARTIALLY PAID`, `POSTED`, and `VOID` are unreachable.
21. `paymentAmount` is invariantly formatted and `paymentDate` is `YYYY/MM/DD`.
22. A successfully confirmed plan reaches the fake server once, and no plan causes more than one
    POST attempt on any path including replay, concurrent confirmation, timeout, 409, and 5xx.
23. An ambiguous outcome is reported as indeterminate, names the `paymentRef`, and instructs
    reconciliation before re-planning.
24. The echoed response is verified against the submitted invoice and status.

## Verify ontology reporting

25. Tool and resource return byte-identical schema bytes without any BrightFlag call.
26. Every declared entity, field, and relationship traces to the reviewed snapshot or a checked-in
    contract, with no invented facts.
27. Provenance names only the four fixed operations.
28. Output is byte-identical across runs, locales, and time zones, and the drift check fails on an
    unregenerated contract change without writing.
29. Schema artifacts contain no live data, tenant identifier, allow-list value, origin, hostname,
    token, path, or timestamp.

## Verify platform controls

30. HTTP bearer validation checks signature, issuer, audience, lifetime, not-before, and required
    claims, and rejects `none`-algorithm tokens. The trust root is swappable by configuration
    between a live JWKS provider and a local development one with no change to the validation code
    path; the local provider and its dev token-issuing tool cannot be selected under a profile
    marked production and are absent from the built container image.
31. Caller tokens never reach BrightFlag and the BrightFlag service token never reaches a caller,
    a log, an error, an audit record, or a trace.
32. Permissions are deny-by-default and the plan store is caller-scoped, bounded, and expiring, and
    its declared single-instance or multi-instance topology is actually enforced.
33. Rate, concurrency, message-size, response-size, cancellation, and shutdown behavior is bounded,
    with a stricter limit on the payment tool.
34. Automated tests contact only the local fake BrightFlag server.
35. Container, CI, documentation, threat model, runbook, and Narrative governance match the
    implementation.
36. The homelab deployment artifacts pin an immutable image, enforce the documented single-instance
    topology and container hardening, keep secrets and private signing material outside the image
    and Compose model, provide HTTPS only to explicitly allowed LAN clients, preserve the
    non-production guard on local identity, and have validation evidence that makes no live
    BrightFlag call or payment. The documented Windows baseline uses Linux containers, PowerShell
    scripts, narrow Defender Firewall rules, NTFS ACLs, valid certificate trust, and an honest,
    tested or explicitly manual Docker Desktop reboot-start procedure.

## Adversarial tests

Exercise:

- URL, path, query, header, JSON, decimal, date, and log injection;
- unknown MCP arguments and unknown fields in BrightFlag responses;
- a BrightFlag response body containing text instructing the server to pay an invoice;
- plan replay, expiry, argument swapping, cross-caller use, and concurrent confirmation races;
- a payment timeout after the fake server has received the request;
- a second confirmation of an already-executed plan;
- a `paymentRef` reused across two invoices, and two payments for one invoice;
- an amount just inside and just outside the configured tolerance;
- a cross-currency payment and a zero or negative amount;
- allow-list entries differing by case, whitespace, or substring;
- a caller or deployment configured for a different tenant boundary being refused before any
  BrightFlag call; do not require an object-level cross-tenant fixture when the reviewed BrightFlag
  payload has no authoritative tenant discriminator;
- window and epoch-boundary manipulation, including a 32-day request;
- oversized batches, pages, error bodies, and continuation data;
- an attempt to add a fifth tool or a fifth operation;
- an attempt to select the local identity trust provider, or to run the dev token-issuing tool,
  under a profile marked production; and
- seeded secrets and business identifiers at every telemetry and generation boundary.

## Commands

From a clean clone, run at minimum:

```bash
dotnet restore --locked-mode
dotnet format --verify-no-changes
dotnet build --no-restore
dotnet test --no-build
dotnet list package --vulnerable --include-transitive
dotnet run --project src/BrightFlagMcpServer -- schema check
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
git diff --check
docker build .
docker compose -f deploy/homelab/compose.yaml config
```

If Docker is unavailable, report the gap. Do not contact a live BrightFlag tenant to compensate for
missing fake-server evidence. If Prompt 10 chose different checked-in paths or validation commands,
run those exact equivalents and record them.

## Report

Report findings by severity with exact file and line references. For each confirmed defect, explain
the violated requirement, implement the smallest correction, add a regression test, and rerun the
affected and complete checks.

Finish with a requirements-to-evidence table for all 36 verification points. List residual risks,
unrun checks, manual BrightFlag controls, and any tenant-configuration incompatibility.

Commit audit fixes locally if needed. Do not push, label, open, or merge a pull request unless
explicitly requested.
