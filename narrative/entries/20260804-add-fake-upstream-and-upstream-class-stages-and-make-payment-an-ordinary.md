---
date: 2026-08-04
slug: add-fake-upstream-and-upstream-class-stages-and-make-payment-an-ordinary
title: "Add fake upstream and upstream-class stages, and make payment an ordinary capability"
summary: "**Stage 21** hosts Stage 3's fake twice from one shared component — the existing in-process loopback host, unchanged, and a new `tools/BrightFlagMcp.FakeUpstream` packaged as its own image."
kind: product
status: accepted
sequence: 2026-08-04T06:00:36.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/60; merge commit 6f256090a8c5e876f40e2529efd139c1a31825a9"
---

## Context

Two gaps surfaced while working out how to exercise the server without calling BrightFlag.

The first is that the sequence never named the BrightFlag origin in its cross-repository
configuration contract. `prompts/19` lists eight environment-variable names, calls a rename on
either side a breaking change to the other repository, and omits the origin — while the same prompt
requires that origin to be an integration-test one. `prompts/20` restates the contract and does not
add it. The deployment must therefore be setting it under a name no document states, which is why
"point the server at a different BrightFlag" had no answer.

The second is that the origin was never the only thing that varied by environment. Prompt 3 wrote
one rule for every deployment — HTTPS everywhere, plain HTTP only for a loopback fake — and Prompt
19 attached the environment to the *capability* instead, enabling payment against the
integration-test tenant only and naming a separate `MarkInvoicePaidRoles` grant to do it. So "which
BrightFlag" ended up half in the origin and half in the payment grant, and neither half could
express production.

Separately, Stage 3's fake is deliberately unshippable — `plans/03` places it in
`test/BrightFlagMcp.Tests/Fakes/` precisely so Stages 9 and 10 can prove it cannot reach the
container image. That is correct for automated tests and leaves a deployed container with nothing to
call.

The requirement driving both stages is to be able to name three upstreams: BrightFlag production for
production deployments, BrightFlag integration-test for the home lab, and a fake for the home lab.
The decision to withdraw payment's special status was taken deliberately: an agent with an
authenticated user reaches all of this server's tools.

## Decision

**Stage 21** hosts Stage 3's fake twice from one shared component — the existing in-process loopback
host, unchanged, and a new `tools/BrightFlagMcp.FakeUpstream` packaged as its own image. Writing a
second fake was rejected: a divergence would make a green test suite evidence about a fake the
deployed server never calls.

Payload selection is out of band and never touches the server: a startup scenario with no default,
plus an admin listener on a separate port, token, and network that the tester drives directly. A
scenario header propagated from the MCP client through the server was rejected outright — it
breaches the reusable contract's rule that no base URL, path, operation, query string, method,
header, or credential is accepted from an MCP argument, and it would mean the deployed server under
test is not the one that would ship.

**Stage 22** replaces the unnamed origin with a closed upstream class — `Production`,
`IntegrationTest`, `Fake` — with no default. The origin stays a single configured value supplied by
the deployment, so no BrightFlag hostname enters this repository, but it is now validated against
the declared class: scheme, address category, admissible secret provider, and permitted deployment
profile. Result marking is derived from the class rather than configured beside it, which supersedes
Stage 21's `BrightFlag__Upstream__Acknowledgement` — a second setting governing the same label is a
second thing that can be wrong.

The server cannot verify what is at the far end of an origin, and Stage 3 forbids it from fetching
the OpenAPI document at startup to find out. Both documents state that plainly rather than letting
the class read as validation of the upstream's identity.

Payment's special status is withdrawn in the same change, and the supersession table names exactly
where it lived: Prompt 8's two-capability split, Prompt 8's stricter payment rate limit, and Prompt
19's separately named payment role and integration-test-only restriction. One grant now governs the
whole surface. Every integrity property of the write survives — the plan token, `PAID` only, atomic
consumption, at most one POST, no automatic retry, ambiguous returned as ambiguous. `DESIGN-CALLS.md`
§2 is deliberately not edited: its subject is the write's mechanics and its refusal to retry, not
who may call it.

The configuration contract becomes a generated manifest derived from the options tree, with a drift
test and a no-default-on-required test. Completing the hand-written list was rejected: it would fix
today's omission and keep the mechanism that produced it, and that list has already been silently
wrong once across two stages and a reviewed implementation.

## Consequences

Three upstreams become specifiable, and a deployment must now say which one it is talking to —
there is no default and startup fails without it.

Selecting `Production` makes a production upstream reachable, and because payment is now ordinary it
makes a production payment write reachable with it. This is the largest single increase in blast
radius in the sequence, and it lands in the same change that removes the payment grant. Prompt 8's
single-instance gate still holds and still fails startup above one instance, so a production
deployment under Stage 22 is a single instance; contingent Stages 12 and 13 are not discharged by
this and remain the prerequisites for anything more.

Retiring `BrightFlag__Authorization__MarkInvoicePaidRoles__*` breaks the LocalAI deployment script
the moment this ships, against a server that rejects unknown configuration properties. That is a
loud startup failure rather than a silent one, but the two repositories need a known landing order.
The LocalAI change is out of scope here — Prompt 19 made that repository the sole deployment owner
and this repository does not edit its script.

Stage 21 stays contingent; Stage 22 does not, because every deployment has to name its upstream.
Stage 22 defines the `Fake` class whether or not Stage 21 is played, and selecting it without the
service Stage 21 builds is a configuration error the operator meets as a connection failure.

Deliberately left open: origins are supplied by the deployment rather than checked in, so a typo can
still redirect traffic within what the class permits — the class constrains scheme, address
category, and marking, not identity. The manual gates remain manual: nothing in CI reaches a real
BrightFlag origin, Windows, Docker Desktop, or the home-lab network, and none of those checks may be
reported as passing.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
