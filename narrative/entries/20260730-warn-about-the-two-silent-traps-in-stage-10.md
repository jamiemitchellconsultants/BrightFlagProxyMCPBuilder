---
date: 2026-07-30
slug: warn-about-the-two-silent-traps-in-stage-10
title: "Warn about the two silent traps in Stage 10"
summary: "**State the constraint and the failure; never the technique.** Neither addition names bash's TCP support, a proxy-side probe, or Compose's `${VARIABLE:?}` form."
kind: product
status: accepted
sequence: 2026-07-30T18:41:57.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/35; merge commit f19453f1ce85c112d6db08e6051372add14b48b4"
---

## Context

Stage 10 was executed against the implementation repository. Two of its requirements turned out to
have no obvious correct resolution, and in both cases the obvious *incorrect* one is silent.

The first: "a health check against the Prompt 9 readiness endpoint". The Prompt 9 image is built on
the .NET runtime base, which carries no `curl` and no `wget`. The natural resolution is to install
one — which changes the artefact whose contents Prompt 9 spends a whole stage proving, and which
`scripts/verify-image.sh` will not object to, because it asserts that a named list of development
affordances is absent rather than that only expected things are present. The trainee's own Stage 9
check passing would confirm they were fine.

The second: "an explicit port binding, preferably to a dedicated LAN address" alongside "fail-closed
placeholder detection". Written the obvious way, an unset Compose variable resolves to an empty
string rather than failing, and a port mapping with an empty host part publishes on every interface.
That is the exact exposure the stage's network boundary exists to prevent, produced by following the
stage, with no error and no warning at any point. It was found by rendering the model with the value
deliberately removed — a test nothing in the stage asks for.

Both were judged too complex for a trainee to resolve unaided, which is the standing criterion for
back-porting into this sequence. `DESIGN-CALLS.md` §4 governs the ones that are not.

## Decision

**State the constraint and the failure; never the technique.** Neither addition names bash's TCP
support, a proxy-side probe, or Compose's `${VARIABLE:?}` form. A trainee is told that the image has
no HTTP client and that installing one is a change to a proved artefact, and that an unset variable
becomes an empty string rather than an error — and is left to work out what to do about either.

**Say why the existing safety net will not catch the first one.** That `verify-image.sh` is a
deny-list rather than an allow-list is the part a trainee cannot derive: without it, "do not add a
client" reads as fussiness, and their own passing Stage 9 check actively reassures them. Naming the
mechanism's shape is not the same as naming the answer.

**Include one method instruction for the second one:** render the model with nothing supplied. That
says how to see the failure, not how to fix it. Leaving it out was rejected — the failure is
invisible by construction, so a warning that does not say how to look at it can be read carefully
and still missed, which is what happened here.

**Rejected: fixing the prompts so the obvious resolution is correct** — naming a probe technique and
the mandatory-variable form. That converts two decisions into dictation, and the second one is worth
making: a learner who has watched an empty variable publish a port on every interface understands
fail-closed configuration in a way that being handed the syntax does not produce.

## Consequences

An agent running Stage 10 now meets both traps as stated constraints rather than as a crash or a
silent exposure, and the learner still has to decide what to do about each.

The cost is a slightly easier Stage 10. Two of its sharper edges are now signposted, and a trainee
who would have discovered the port-binding exposure themselves — by testing an empty value, which
nothing else prompts — no longer gets that discovery. Judged worth it: the failure is silent and
security-relevant, and an undiscovered instance ships a LAN-exposed deployment that looks correct.

The health-check warning tells a trainee that `scripts/verify-image.sh` cannot see an added tool.
That is also a true statement about Stage 9's proof, and a reader may take it as licence to treat
that check as weaker than it is. It is not weaker than it was; it is now merely described
accurately.

Stage 11's audit reads the prompts against the implementation. Both warnings describe properties the
implementation already has, so they narrow what that audit can find rather than widening it.
