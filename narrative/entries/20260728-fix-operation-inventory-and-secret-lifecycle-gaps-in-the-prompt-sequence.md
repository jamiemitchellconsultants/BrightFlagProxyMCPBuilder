---
date: 2026-07-28
slug: fix-operation-inventory-and-secret-lifecycle-gaps-in-the-prompt-sequence
title: "Fix operation inventory and secret-lifecycle gaps in the prompt sequence"
summary: "Removed `getAPBatches` from the fixed operation set entirely, updating the contract, README, and every affected prompt (1, 3, 5, 9, 11) in lockstep rather than documenting it as intentionally unused dead configuration."
kind: product
status: accepted
sequence: 2026-07-28T06:32:51.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/5; merge commit c3dcfff3c3906a5c322cecaba94ec329ffe6d572"
---

## Context

The unwindowed `getAPBatches` listing is residue from an earlier correction (recorded in
`DESIGN-CALLS.md` entry 4) that removed the *runtime allowance* to call it but never removed it from
the fixed-operation inventory, so it stayed validated, configured, and faked while genuinely unused
by any of the four MCP tools.

Separately, the sequence requires a signed, stateless keyset cursor (Prompt 1) and a caller-bound
plan token gating the project's only irreversible write (Prompt 6), but no prompt assigned where
either secret comes from, how the plan token is generated, whether the plan store persists across
instances, how the cursor-signing key is rotated, or named either as a threat — despite every other
credential in the project (the BrightFlag bearer token, caller JWTs) having that lifecycle fully
specified.

## Decision

Removed `getAPBatches` from the fixed operation set entirely, updating the contract, README, and
every affected prompt (1, 3, 5, 9, 11) in lockstep rather than documenting it as intentionally unused
dead configuration.

Added a `schema check` requirement to Prompt 7 (exact invocation, exit-code contract, no MCP listener
or BrightFlag call) so Prompt 11's audit command is backed by something the sequence actually builds,
and cross-referenced it from Prompt 9's CI step.

For the payment plan token: required CSPRNG generation with at least 128 bits of entropy and no
encoded payload (Prompt 6), so it's an opaque capability into the server-side plan store rather than
a signed, decodable structure.

For the plan store: required an explicit single-instance (in-process store) or multi-instance
(externally shared store) topology decision, since the plan store was the one stated exception to
otherwise-stateless HTTP transport and needed its own consistency story (Prompt 8).

For the cursor-signing key: added a key identifier to the cursor so verification can accept a
current-or-prior key at once (Prompt 1), wrote the corresponding rotation runbook entry covering both
scheduled and suspected-compromise rotation (Prompt 9), and gave the key its own secret-provider
contract independent of the BrightFlag bearer token — with the same redaction requirement, a
required-at-startup check, and an explicit prohibition on generating it in-process, since the cursor
being stateless and callable from any instance depends on every instance verifying against the same
key material (Prompt 3).

Extended the threat model (compromised cursor-signing key, unauthorized plan-store read access) and
the independent-reconstruction audit (points 5, 12, 27, 32) to cover all of the above without
renumbering the existing 35 verification points.

Rejected alternative for the plan token: a signed self-describing token like the cursor. The plan
store already needs server-side state to support atomic single-use consumption, so a signed token
would duplicate a security property the store already provides while adding a second thing to key
and rotate.

## Consequences

The prompts' stated operation inventory now matches what any correct implementation actually calls,
so an integrator reading the fixed-operation table won't wire up, test, and expose an endpoint with
no caller. An agent following Prompt 11 literally can complete its audit without hitting a command
the earlier stages never specified.

The cursor-signing key and plan token now carry the same lifecycle rigor as the BrightFlag bearer
token: provisioning, redaction, rotation, topology, and threat coverage. Left open: a real deployment
still has to choose an actual secret store behind the new provider contracts, and the single- vs
multi-instance topology decision is now mandatory but still deferred to Prompt 8's implementer to
make explicitly rather than default into.
