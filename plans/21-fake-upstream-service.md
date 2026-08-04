# Stage 21 — Build a deployable fake BrightFlag upstream

Source: `BrightFlagProxyMCPBuilder/prompts/21-fake-brightflag-upstream.md`

## Context

**Contingent stage. Not part of version 1.** It runs only when a deployed instance must be exercised
end to end without BrightFlag's integration-test tenant — because that tenant is unavailable,
unstable, or must not be written to. Nothing here is implemented speculatively.

Stage 3 already built a fake, and it is the right fake for the job it has. It is loopback,
in-process, and lives in `test/BrightFlagMcp.Tests/Fakes/` precisely so Stages 9 and 10 can prove it
cannot ship. None of that is a defect to be corrected here. It simply cannot be called by a
container running on `ai-mcp-server`, because a loopback listener inside a test process is not
reachable from a deployed one.

So this stage is a *hosting* change, not a new fake. The behaviour that Stages 5 and 6 assert
against stays one implementation; what changes is that a second host can run it as a real process on
a real network. Writing a second fake instead would be the failure mode this plan exists to prevent:
a green test suite would then be evidence about a fake the deployed server never calls.

The other half of the stage is the two ways a deployment could go wrong. Pointing the server at a
non-BrightFlag origin must be impossible by accident, and a payment confirmation produced against a
fake must be impossible to mistake for a real one. Both are handled by refusing to start rather than
by warning.

## Preconditions

- The need is actual and recorded: which deployed instance is being exercised, and why the
  integration-test tenant is not being used. If it cannot be stated, stop.
- Stages 1–11 committed and green; Stages 17–20 applied, since the target is the deployed topology
  they produce.
- Stage 3's fake is intact and its tests pass unchanged before anything is extracted.

## Scope in

Extracting Stage 3's fake into a shared component; a second self-hosted host for it; checked-in
scenario fixtures; out-of-band scenario selection; the origin acknowledgement and its startup gate;
fake-upstream marking in audit records, logs, and tool response bodies; extending the Stage 9 and 10
image proofs.

## Scope explicitly out

Any MCP surface change — no new tool, resource, argument, or response field that names a scenario.
Any new BrightFlag operation. A deployment entry point in this repository, which Stage 19 retired
and this stage does not reintroduce. Replacing the loopback host the automated tests use. Anything
that makes the fake reachable from outside the home-lab network.

## Work items

### 1. Extract the fake, do not fork it

One component owns the route table, the request recording, and the `ErrorResponse` bodies. Two hosts
consume it: the existing in-process loopback host, whose behaviour must not change, and a new
`tools/BrightFlagMcp.FakeUpstream`.

`tools/` rather than `src/` is the placement decision, and it carries the same weight Stage 3's
`test/` placement did: it is what makes the Stage 9 and 10 absence proofs mechanical rather than
argumentative. The fake is built as its own image; no project under `src/` references it.

### 2. The API listener

Exactly the five Stage 3 routes — the four fixed operations plus the snapshot route — and 404 for
everything else including every admin path. One configured bearer token, compared exactly — no
wildcard, no unauthenticated mode, not even for a health probe. The documented `ErrorResponse` shape
for 400, 403, 404, 409, and 500.

Recording of method, path, query, headers, body, and call count carries over from Stage 3 unchanged.
It is the evidence a deployed run produces, and it is read through the admin listener rather than
through a log file.

### 3. Scenarios as data

A scenario is a named directory of checked-in synthetic fixtures — batches, batch invoice
identifiers, invoice summaries, payment-status responses. Validated at startup against the reviewed
snapshot's schemas; unknown fields, absent required fields, and a status outside the scenario's own
declared vocabulary each fail startup rather than being served. A fixture that is wrong should fail
where it is loaded, not where it is asserted against three stages later.

The eight required scenarios each exist to reach a rule an earlier stage established, and the plan
names which:

| Scenario | The rule it reaches |
|---|---|
| batched and allow-listed | Stage 4's evidence rule, positive case |
| batched, status outside the allow-list | Stage 4, status alone is never sufficient |
| allow-listed, never batched | Stage 4, release evidence is required |
| status-change timestamp outside the batch window | Stage 5's `SummaryWindowMarginDays` and the reconciliation gap |
| summaries spanning more than one page | Stage 5's keyset cursor, no invoice returned twice |
| payment POST answering 409 | Stage 6's permanently indeterminate plan |
| payment POST answering 500 | Stage 6's no-automatic-retry rule |
| empty window | the empty result, distinguished from an error |

The first three carry Stage 3's two mandatory fixtures forward; do not let the extraction drop them.

### 4. Selection, out of band

`BRIGHTFLAG_FAKE_SCENARIO` at startup, with **no default**. Unset or unknown fails startup, on the
same reasoning Stage 4 applied to the empty allow-list: a fake that silently picks a scenario turns
a green run into a statement about nothing.

An admin listener, on its own port, with its own token, bound to a network the MCP server is not
attached to. It lists scenarios, reports and switches the active one, returns recorded requests, and
resets them. Switching is atomic and clears the recording in the same operation, so an assertion
cannot straddle two scenarios.

The separation is the whole point and each part of it is a test:

- no selector reaches the fake through the MCP server — not a tool argument, a propagated header, a
  query parameter, or a path prefix on the four fixed operations;
- admin paths return 404 on the API listener;
- the admin listener shares no port, token, or network with the API listener; and
- no fallback, first-alphabetical, or default scenario exists.

Keeping selection entirely off the server is what preserves the reusable contract's rule that no
base URL, path, operation, query string, method, header, or credential is ever accepted from an MCP
argument. A header propagated end to end would have been simpler and would have broken it.

### 5. The origin acknowledgement

The address stays in Stage 3's single configured origin. One value joins it,
`BrightFlag__Upstream__Acknowledgement`, accepting only a fixed literal stating the origin is not
BrightFlag. The gate is symmetric, and both halves fail startup:

| Origin | Acknowledgement | Result |
|---|---|---|
| BrightFlag host | absent | start |
| not a BrightFlag host | present | start |
| not a BrightFlag host | absent | **fail startup** |
| BrightFlag host | present | **fail startup** |

The fourth row matters as much as the third. Without it the acknowledgement becomes a value someone
leaves set, and the gate stops meaning anything the next time the origin moves back.

The acknowledgement extends Stage 3's plain-HTTP exception from loopback to a private or
container-network address and to nothing else. A public origin is still HTTPS, and user-information,
fragments, wildcards, traversal, embedded credentials, and cross-origin redirects are all still
rejected exactly as before.

### 6. Marking, where a reader will actually meet it

While the acknowledgement is in force, the marker appears in every audit record, every log scope,
**and the body of every tool response** — asserted specifically on a successful
`brightflag_mark_invoice_paid` result.

The response body is the load-bearing one. A payment confirmation is the single artifact in this
system that must never read as real when it is not, and a log line does not travel with the response
to whoever is reading it.

### 7. Image proofs

Extend the Stage 9 and Stage 10 proofs to assert the fake assembly, the scenarios directory, and the
admin token are absent from the server image, in the same way those stages already prove the Stage 8
development token-issuing tool is absent. Add the grep that fails if any project under `src/`
references `tools/BrightFlagMcp.FakeUpstream`.

### 8. Documentation

State what a run against the fake proves — call construction, paging, the evidence rule,
authorization, plan consumption, refusal behaviour, transport under the deployed topology — and what
it does not: BrightFlag's actual response bodies, its tenant's status vocabulary, the payment feed's
real semantics, its behaviour under load. Do not report a BrightFlag integration-test gate as passed
on the strength of a fake run.

Deployment shape for LocalAI to adopt is documented here, not executed here. Stage 19 made LocalAI
the sole deployment owner and this stage does not take that back.

## Tests

One-to-one with Prompt 21's "Prove" list:

- both hosts resolve the same route table and produce byte-identical `ErrorResponse` bodies for 400,
  403, 404, 409, and 500 — a parameterised test over both hosts, not a second copy of the suite;
- the API listener serves exactly the five routes, requires the bearer token, and returns 404 for
  every other path including every admin path;
- startup fails when `BRIGHTFLAG_FAKE_SCENARIO` is unset, names an unknown scenario, or names one
  whose fixtures do not validate against the reviewed snapshot;
- each of the eight scenarios drives its intended outcome end to end through the server, per the
  table in work item 3, with the 409 and 500 payment responses each leaving the plan permanently
  indeterminate and no automatic retry observed at the fake's call count;
- the admin listener switches atomically, clears recorded requests in the same operation, refuses a
  request bearing the BrightFlag bearer token, and is unreachable from the MCP server's network;
- a scenario selector smuggled through the server as a tool argument, header, query parameter, or
  path prefix changes nothing about which payload is returned;
- every row of work item 5's table, in both configuration orders, and every Stage 3 URL rejection
  still rejecting while the acknowledgement is in force;
- the fake marker is present in audit records, log scopes, and tool response bodies, asserted on a
  successful `brightflag_mark_invoice_paid` result;
- the ontology schema and the resource are byte-identical against the fake and against a BrightFlag
  origin; and
- the server image contains no fake assembly, no scenarios directory, and no admin token, and no
  `src/` project references the fake.

Stage 3's existing fake-server tests must pass unchanged after the extraction. If they need editing,
the extraction changed behaviour and that is the finding to report, not a test to update.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

The end-to-end scenario runs against a deployed instance are **manual gates**. Record each with its
exact command, the scenario selected, and the expected result. They exercise Windows, Docker
Desktop, and the home-lab network, none of which run here, and none of them may be reported as
passing.

## Stage boundary

Commit locally. Suggested message: `Add a deployable fake BrightFlag upstream for deployed runs`.

`narrative-required` when published. Decisions to record: **why this is contingent** rather than a
development BrightFlag every deployment may reach; **why the scenario selector is kept entirely off
the MCP server**, and what the rejected header-propagation design would have cost; and **why the
origin acknowledgement fails in both directions** rather than warning.

Do not push unless requested. **Do not begin any stage that treats the fake as an environment** —
there is no "fake tenant", no seeded data service, and no path by which the fake becomes something
the sequence depends on.

## Risks

- The extraction is where behaviour drifts silently. Stage 3's suite passing unchanged is the guard,
  and editing that suite to accommodate the extraction defeats it.
- A fake that is easier to reach than the integration-test tenant becomes the thing that is always
  exercised. Naming what the fake does not prove is the mitigation, and it is a weak one; expect to
  re-state it.
- The admin listener is a control surface with a token, and it will be tempting to put it on the
  same network as the API listener because that is one fewer thing to configure. That is the whole
  separation.
- The acknowledgement's symmetric gate will look like pedantry the first time someone hits row four
  while moving an origin back to BrightFlag. It is the row that keeps the value from going stale.
- Marking only the logs would pass a shallow review. Assert the marker on the response body of a
  successful payment specifically.
