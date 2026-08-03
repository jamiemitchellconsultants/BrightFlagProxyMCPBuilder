---
date: 2026-08-03
slug: frame-prompt-18-s-secret-store-exposure-as-a-presumption-not-a-fact
title: "Frame prompt 18's secret-store exposure as a presumption, not a fact"
summary: "Hedge the deployment identity, never the severity."
kind: product
status: accepted
sequence: 2026-08-03T07:30:43.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/56; merge commit 7e11b73c595f08c086f6d4ab5b61335ed24465ad"
---

## Context

Prompt 18 required the implementer to write into `docs/`, as fact, that the fixed token sits in an
unauthenticated LocalStack instance readable from the LAN, alongside the cursor-signing key. That
is only true once Prompt 19 is applied. At stage 18 the applied state is Prompt 17, whose
fail-closed rule requires the opposite: no real secret enters LocalStack while it is LAN-reachable,
read-only proof of loopback binding is required first, and absent that proof the implementer stops
at a manual gate. Prompt 19 is what supersedes that rule.

Stage 18 therefore required shipping a documentation statement contradicted by the repository it
was applied to, while forbidding the implementer from reconciling the two ("record them; do not add
a compensating control"). It also gated acceptance on it: nothing in Prompt 18's Prove list covers
the claim, yet acceptance criterion 2 requires it documented — so the stage could not be signed off
without a fact only a later stage could establish. The failure is directional: applying 18 onto a
17-conformant deployment produces docs claiming secrets are LAN-readable when 17's gate has
isolated them.

Four other forward references from 18 to 19 were reviewed and left alone. They are negative
boundary markers — "do not move to the shared realm, Stage 19 owns it" — satisfied by not acting
and checkable against the repository as it stands. That is the sequence's normal shape.

## Decision

Hedge the deployment identity, never the severity. Prompt 18 §4 forbids qualifying a limitation
away, so the security consequence stays flat and unconditional, because it is true of the mode
itself on any store: the server does not control the store, whatever can read the mounted file
holds the credential outright, and the constant-time comparison is not that boundary. Only the
LocalStack specifics become presumptive, with Prompt 19 named as what settles them and the
documentation directed to record what the deployment does rather than what the stage presumed.

Rejected: moving the bullet into Prompt 19 entirely. That resolves the contradiction but costs
stage 18 a real disclosure — a reader who applies 18 alone would meet the shared-identity and
no-revocation consequences with nothing said about the store the credential comes from, which is
the one that bounds the other two in practice.

`DESIGN-CALLS.md` records no judgement on this, so no deliberate rough edge is being overturned.

## Consequences

The tension with Prompt 17's fail-closed LocalStack rule disappears: stage 18 no longer asserts
LAN-readability as present fact while 17's isolation gate is still in force. Prompt 18's `docs/`
requirement becomes provable at its own stage, since it now asserts a property of the mode the
implementer can verify. Prompt 19's supersession becomes the thing that resolves the presumption
rather than something 18 pre-empts.

Prompt 19 is unchanged and still states the LocalStack exposure as fact, which is correct there.

Deliberately left open: the two other forward-reference mismatches found in the same review, both
in the boundary markers and both about naming rather than substance. Prompt 18 twice says Stage 19
moves the deployment to `mcp-client`, a name that appears nowhere in Prompt 19 — 19 names the
audience and resource client `brightflag-mcp` and refers only to "the existing public caller
client". Prompt 18 also attributes to 19 a "shared-client audience limitation that follows from"
the realm move; 19's nearest statement is that the shared caller client's audience mapper is not a
service-separation boundary, tracked by issue #46. Neither is addressed here.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
