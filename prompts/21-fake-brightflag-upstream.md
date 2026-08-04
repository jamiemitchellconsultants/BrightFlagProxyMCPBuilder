# Prompt 21 — Exercise the deployed server against a fake BrightFlag upstream

Using the reusable contract and the artifacts produced by Prompts 1–11 and 17–20, build a separately
deployable fake BrightFlag upstream, serving pre-defined payloads selected out of band, that a
deployed instance of this server can be pointed at instead of BrightFlag's integration-test
environment.

**This stage is contingent.** It is not part of the version 1 sequence and must not be implemented
speculatively. Prompt 3 already provides a loopback fake for automated tests, and that fake remains
the only one those tests use. This stage exists for a different need: exercising a *deployed*
instance end to end when a BrightFlag integration-test tenant is unavailable, unstable, or must not
be written to. Do not begin unless that need is actual.

The fake is a test double for the deployed topology. It is not a BrightFlag simulator, not a
development BrightFlag, and never an upstream any deployment may reach by default.

## What must not change

The MCP surface stays exactly four tools and one resource. The four fixed BrightFlag operations, the
approved-for-payment evidence rule, the plan-before-execute split, the `PAID`-only payload, and the
no-automatic-retry rule are all unchanged. No tool argument, header, or response field learns that a
scenario or a fake exists. This stage adds no capability. If any requirement below appears to need a
wider MCP surface, stop and report the conflict.

Prompt 19 transferred deployment ownership of this server to the LocalAI repository. That is
unchanged. This stage produces the fake, its image, and a documented Compose and configuration shape
for LocalAI to adopt. It does not reintroduce a deployment entry point in this repository.

## One fake, two hosts

Do not write a second fake. Extract Prompt 3's route table, request recording, and `ErrorResponse`
behavior into a single component, and have two hosts consume it:

- the existing in-process loopback host, used by the automated tests, unchanged in behavior; and
- a new self-hosted process in `tools/BrightFlagMcp.FakeUpstream`, packaged as its own container
  image.

A divergence between the two would make a green test suite evidence about a fake the deployed server
never calls. The shared component is what makes that divergence impossible, so prove both hosts
resolve the same routes and produce the same error bodies.

## The fake upstream service

It exposes exactly the Prompt 3 routes on its API listener, and nothing else:

- `GET /v3/api-docs/external`, serving the checked-in snapshot;
- `GET /api/ap-batch/v1/{startEpochTime}/{endEpochTime}`;
- `GET /api/ap-batch/v1/batch/{batchID}/invoices`;
- `GET /api/v1/invoice-summary`, with working `invoiceStatus`, date-window, and paging behavior; and
- `POST /api/v1/invoicePayment/invoice-payment-status`.

Every other path on the API listener returns 404, including the admin paths below. It requires a
bearer token, compares it against one configured value with no wildcard and no unauthenticated mode,
and returns the documented `ErrorResponse` shape — `message`, `errors`, `errorCategory`,
`errorCode`, `status`, `metadata` — for 400, 403, 404, 409, and 500. It records the exact method,
path, query, headers, body, and call count of everything it receives, because that recording is the
evidence a deployed run produces.

## Scenarios are the pre-defined payloads

A scenario is a named directory of synthetic fixtures checked into this repository, covering
batches, batch invoice identifiers, invoice summaries, and payment-status responses. Scenarios are
data, not code, and are validated at startup against the reviewed OpenAPI snapshot's schemas.
Unknown fields, absent required fields, and a status value outside the scenario's own declared
vocabulary each fail startup rather than being served.

Provide at least these, because each one is load-bearing for a rule an earlier stage established:

- an invoice batched and allow-listed, the only path on which payment may proceed;
- an invoice batched whose status is outside the allow-list;
- an invoice whose status is inside the allow-list but which was never batched;
- an invoice whose `invoiceStatusChangeTimestamp` sits outside the batch window, exercising
  `SummaryWindowMarginDays` and the reconciliation gap;
- enough summaries to force the keyset cursor across more than one page;
- a payment POST answering 409, so the permanently indeterminate plan can be reached deliberately;
- a payment POST answering 500; and
- an empty window returning no batches at all.

## Selecting a scenario, out of band only

The active scenario is chosen on the fake, never through the server:

- `BRIGHTFLAG_FAKE_SCENARIO` names it at startup. There is no default. An unset or unknown value
  fails startup, exactly as Prompt 4's empty allow-list does, because a fake that silently picks a
  scenario produces a green run that means nothing.
- An **admin listener on a separate port** lists scenarios, reports and switches the active one,
  returns the recorded requests, and resets them. It carries its own token, distinct from the
  BrightFlag bearer token, and it is bound to a private network the deployed MCP server is not on.
- Switching is atomic and clears the recorded requests in the same operation, so no assertion can
  straddle two scenarios.

Reject, as test cases rather than as comments:

- any scenario selector reaching the fake through the MCP server — a tool argument, a propagated
  header, a query parameter, or a path prefix on the four fixed operations;
- the admin paths answering anything but 404 on the API listener;
- the admin listener sharing a port, a token, or a network with the API listener; and
- a scenario chosen by a fallback, a first-alphabetical rule, or any default.

## Pointing a deployed server at it

The address lives where it already lives: the single configured BrightFlag origin from Prompt 3. Add
one value beside it, `BrightFlag__Upstream__Acknowledgement`, whose only accepted content is a fixed
literal stating the origin is not BrightFlag. Startup then enforces, in both directions:

- an origin that is not a BrightFlag host, with the acknowledgement absent, fails; and
- an origin that is a BrightFlag host, with the acknowledgement present, fails.

Neither direction is a warning. The acknowledgement narrowly extends Prompt 3's plain-HTTP exception
from loopback to a private or container-network address, and to nothing else: a public origin is
still HTTPS, and the acknowledgement never relaxes URL user-information, fragment, wildcard,
traversal, embedded-credential, or cross-origin-redirect rejection.

When the acknowledgement is in force, every audit record, every log scope, and the body of every
tool response — including a successful `brightflag_mark_invoice_paid` result — carries an
unmistakable marker that the upstream was not BrightFlag. A payment confirmation is the one artifact
in this system that must never be mistakable for a real one, and a log line alone does not travel
with the response.

The ontology schema is unaffected. It is generated from checked-in contracts and the reviewed
snapshot, so it is identical whichever upstream is configured, and proving that is cheaper than
assuming it.

## The fake never ships inside the server

`tools/BrightFlagMcp.FakeUpstream` is referenced by no project under `src/`. It is built as its own
image, published separately if at all, and never appears as a layer, file, assembly, or environment
variable of the server image. Extend the Prompt 9 and Prompt 10 image proofs to assert the absence
of the fake assembly, the scenarios directory, and the admin token from the server image, in the
same way those stages already prove the Prompt 8 development token-issuing tool is absent.

## What a run against the fake does and does not prove

A green run proves this server's outbound call construction, paging, evidence rule, authorization,
plan-consumption, refusal behavior, and transport under the deployed topology. It proves nothing
about BrightFlag's actual response bodies, its tenant's status vocabulary, the payment feed's real
semantics, or its error behavior under load. Say so wherever the fake is documented. Do not report a
BrightFlag integration-test gate as passed on the strength of a fake run, and do not let the fake
become the reason the integration-test tenant stops being exercised.

## Prove

- the in-process test host and the self-hosted process resolve the same route table and produce
  byte-identical `ErrorResponse` bodies for 400, 403, 404, 409, and 500;
- the API listener serves exactly the five routes listed above — the four fixed operations and the
  snapshot route — requires the bearer token, and returns 404 for every other path including every
  admin path;
- startup fails when `BRIGHTFLAG_FAKE_SCENARIO` is unset, names an unknown scenario, or names one
  whose fixtures do not validate against the reviewed snapshot;
- each required scenario drives its intended outcome end to end through the deployed server: the
  allow-listed batched invoice is the only one returned as approved for payment, the batched
  non-allow-listed and allow-listed non-batched invoices are both withheld, the out-of-window
  timestamp exercises the summary-window margin, the multi-page scenario is traversed by cursor with
  no invoice returned twice, and the 409 and 500 payment responses each leave the plan permanently
  indeterminate with no automatic retry;
- the admin listener switches scenario atomically, clears recorded requests in the same operation,
  refuses a request bearing the BrightFlag bearer token, and is unreachable from the network the MCP
  server is attached to;
- no tool argument, response field, or outbound header carries a scenario selector, and an attempt
  to smuggle one through the server changes nothing about which payload is returned;
- a non-BrightFlag origin without the acknowledgement fails startup, a BrightFlag origin with the
  acknowledgement fails startup, and the acknowledgement permits plain HTTP only to a private or
  container-network address while every other Prompt 3 URL rejection still applies;
- every audit record, log scope, and tool response body produced while the acknowledgement is in
  force is marked as a fake upstream, asserted specifically on a successful
  `brightflag_mark_invoice_paid` result;
- the ontology schema and the resource are byte-identical against the fake and against a BrightFlag
  origin; and
- the server image contains no fake assembly, no scenarios directory, and no admin token, and no
  project under `src/` references `tools/BrightFlagMcp.FakeUpstream`.

## Acceptance criteria

- One shared fake implementation is hosted two ways, and the automated tests continue to use the
  loopback host unchanged.
- A deployed server can be pointed at the fake only by configuring both the origin and the explicit
  acknowledgement, and every artifact a caller or auditor sees says the upstream was fake.
- The pre-defined payload is selected on the fake, out of band, with no default and no path through
  the MCP server.
- The fake is provably absent from the shipped server image.
- The documentation states plainly what a run against the fake does not prove, and no BrightFlag
  integration-test gate is reported as passed on its strength.
- Every check that cannot run in this environment is a labelled manual gate with its exact command
  and expected result, and is never reported as passing.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record why the deployable fake upstream is a contingent
stage rather than a development BrightFlag, why the scenario selector is kept entirely off the MCP
server, and why pointing a deployment at a non-BrightFlag origin requires an explicit
acknowledgement in both directions. Do not push unless requested.
