# Stage 11 — Independent reconstruction audit

Source: `BrightFlagProxyMCPBuilder/prompts/11-independent-reconstruction-audit.md`

## Context

The final stage verifies the finished server against the reusable contract and every staged prompt,
independently. Run it as a **fresh task and, ideally, with a different coding agent** — the point is
an auditor without the builder's assumptions. Narrative entry 3 records why: the prompts themselves
were corrected only because a different model cross-examined them, and "fluency is not evidence of
consistency."

Add no capabilities. Fix only **confirmed** defects, with the smallest correction plus a regression
test.

## Preconditions

Stages 1–10 committed. A clean clone is the starting point — audit what is checked in, not a warm
working tree.

## Scope in

The 36 verification points; the adversarial test suite; the command run; the findings report and
requirements-to-evidence table.

## Scope explicitly out

New features. Refactoring for taste. Weakening any control to make a check pass. Contacting a live
BrightFlag tenant to compensate for missing fake-server evidence.

## The 36 verification points

Grouped as the prompt groups them. Each needs *evidence*, not assertion — a test name, a file and
line, or a command's output.

**Capability ceiling (1–6).** Exactly four tools and one resource, with startup failing otherwise.
Configuration resolves exactly the four operations against one origin. No caller can supply an
origin, path, operation, method, header, query string, page number, allow-list entry, role, or
credential. The snapshot cannot create a tool, field, or operation at runtime, and no runtime path
fetches `/v3/api-docs/external`. `getAPBatches`, `downloadInvoiceDocument`, `downloadAPBatchFile`,
SCIM, matters, vendors, allocations, purchase orders, pay sites, tax rates, budgets, legal service
requests, and reporting are all absent. No outbound ontology call, registration hook, or callback.

**Approved-for-payment evidence (7–15).** Both signals required. Allow-list is tenant configuration,
no shipped default, empty fails startup, matched exactly and case-sensitively. Both boundary
fixtures excluded, with the excluded count and observed statuses reported. Lookback defaults to 7,
never exceeds 31, converts to epoch milliseconds correctly across time zones and DST boundaries.
`includePreviousDrafts` false; at most one revision per `invoiceGroupId`; typed conflict on two.
Cursors opaque, signed, expiring, caller- and window-bound, rejecting tampering, replay with a
changed window, and cross-caller use; verification accepts a current or documented prior key
identifier and rejects an unrecognized one; a rotation runbook exists with no undocumented hard
cutover; the key comes from a configured secret provider rather than in-process generation, is
redacted everywhere the bearer token is, and is identical across instances. Fan-out, page, byte,
result, timeout, and concurrency ceilings fail closed. Amounts and currencies carried unconverted,
never summed across currencies, `exposurePercentage` surfaced not applied. A batch-released invoice
missing from the summary read is a reported reconciliation gap.

**Payment mutation (16–24).** Planning re-verifies approval with a fresh read and validates amount
tolerance, currency, dates, duplicate `paymentRef`, and already-paid state before any write. Plan
tokens cryptographically random, expiring, single-use, race-safe, bound to caller, tenant, invoice
identity, amount, currency, `paymentRef`, contract version. Execution accepts only token plus
explicit confirmation; no planned field overridable. Authorization re-evaluated at execution, denial
**before** the service credential is used. `paymentStatus` only `PAID`; the other three unreachable.
`paymentAmount` invariantly formatted, `paymentDate` `YYYY/MM/DD`. One POST for a confirmed plan and
never more than one attempt on any path — replay, concurrent confirmation, timeout, 409, 5xx.
Ambiguity reported as indeterminate, naming the `paymentRef`, instructing reconciliation. Echoed
response verified against the submitted invoice and status.

**Ontology reporting (25–29).** Tool and resource return byte-identical bytes with no BrightFlag
call. Every entity, field, and relationship traces to the snapshot or a checked-in contract — no
invented facts. Provenance names only the four operations. Byte-identical across runs, locales, time
zones; drift check fails on an unregenerated contract change **without writing**. No live data,
tenant identifier, allow-list value, origin, hostname, token, path, or timestamp in the artifacts.

**Platform controls (30–36).** Bearer validation checks signature, issuer, audience, lifetime,
not-before, required claims, and rejects `none`; the trust root is swappable by configuration with
no change to the validation path; the local provider and dev token tool cannot be selected under a
production profile and are absent from the image. Caller tokens never reach BrightFlag; the
BrightFlag token never reaches a caller, log, error, audit record, or trace. Deny-by-default
permissions; caller-scoped, bounded, expiring plan store with its **declared topology actually
enforced**. Rate, concurrency, message-size, response-size, cancellation, and shutdown behaviour
bounded, stricter on the payment tool. Automated tests contact only the local fake server. Container,
CI, documentation, threat model, runbook, and Narrative governance match the implementation.

**Point 36 (new).** The homelab deployment artifacts pin an immutable image, enforce the documented
single-instance dev topology and container hardening, keep secrets and private signing material
outside the image and Compose model, provide HTTPS only to explicitly allowed LAN clients, preserve
the non-production guard on local identity, and carry validation evidence making no live BrightFlag
call or payment. The Windows baseline uses Linux containers, PowerShell scripts, narrow Defender
Firewall rules, NTFS ACLs, valid certificate trust, and an honest — tested or explicitly manual —
Docker Desktop reboot-start procedure.

## Adversarial tests

URL, path, query, header, JSON, decimal, date, and log injection. Unknown MCP arguments and unknown
fields in BrightFlag responses. **A BrightFlag response body containing text instructing the server
to pay an invoice.** Plan replay, expiry, argument swapping, cross-caller use, concurrent
confirmation races. A payment timeout *after* the fake server received the request. A second
confirmation of an already-executed plan. A `paymentRef` reused across two invoices, and two
payments for one invoice. An amount just inside and just outside tolerance. A cross-currency
payment; a zero or negative amount. Allow-list entries differing by case, whitespace, or substring.
A caller or deployment configured for a different tenant boundary refused before any BrightFlag call
— **without** requiring an object-level cross-tenant fixture, since the reviewed payload has no
authoritative tenant discriminator (`Batch.customerID` is the closest, and the batch read is the
only place it appears). Window and epoch-boundary manipulation, including a 32-day request.
Oversized batches, pages, error bodies, continuation data. An attempt to add a fifth tool or
operation. An attempt to select the local trust provider or run the dev token tool under a
production profile. Seeded secrets and business identifiers at every telemetry and generation
boundary.

## Commands — from a clean clone

```bash
dotnet restore --locked-mode
```

```bash
dotnet format --verify-no-changes
```

```bash
dotnet build --no-restore
```

```bash
dotnet test --no-build
```

```bash
dotnet list package --vulnerable --include-transitive
```

```bash
dotnet run --project src/BrightFlagMcpServer -- schema check
```

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

```bash
git diff --check
```

```bash
docker build .
```

```bash
docker compose -f deploy/homelab/compose.yaml config
```

If Stage 10 chose different checked-in paths or validation commands, run those exact equivalents and
record them. If Docker is unavailable, report the gap.

## Report

Findings by severity with exact file and line references. For each confirmed defect: the violated
requirement, the smallest correction, a regression test, and a rerun of the affected **and** complete
checks.

Finish with a **requirements-to-evidence table for all 36 points**. Then list residual risks, unrun
checks, manual BrightFlag controls, and any tenant-configuration incompatibility.

Expect these to appear as residual items rather than defects, since each is a deliberate limitation
recorded earlier: the payment record store's retention bound on the already-paid check; deferred
corporate MCP authorization alignment; pagination not claiming snapshot isolation; the summary-window
margin's effect on recall; and Stage 10's Windows-host manual gates if they have not yet been
executed on a Windows machine.

## Stage boundary

Commit audit fixes locally if any were needed. **Do not push, label, open, or merge** unless
explicitly requested.

## Risks

- The strongest temptation in an audit is to accept a passing test as evidence of the property the
  test is named after. Read the test bodies for points 12, 17, 22, and 32 in particular.
- An auditing agent that also implemented the build will confirm its own assumptions. Use a
  different agent if at all possible.
- Point 36's Windows-specific items cannot be discharged from a Mac. Record them as unrun with the
  exact expected result, never as passing.
