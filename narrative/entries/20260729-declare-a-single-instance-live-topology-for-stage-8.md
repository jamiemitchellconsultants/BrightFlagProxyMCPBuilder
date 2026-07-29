---
date: 2026-07-29
slug: declare-a-single-instance-live-topology-for-stage-8
title: "Declare a single-instance live topology for Stage 8"
summary: "**The live deployment runs a single server instance.** The in-process plan store and payment record store are therefore sufficient, and horizontal scaling of this service is out of scope for version 1."
kind: product
status: accepted
sequence: 2026-07-29T11:43:20.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/17; merge commit 1896995b03c9008b834584c8601ce371c5d2fd18"
---

## Context

Stage 8 was written with the plan-store topology as an explicit open question, marked as needing an
answer before implementation began and with neither option allowed as a default. That was the right
call at the time: Stage 6 had just put both stores behind `IPlanStore` and `IPaymentRecordStore`
precisely so the choice would be cheap rather than a rewrite, and the sequence deliberately declines
to guess about anything that decides how real money moves.

The question could not stay open, because the two options are not two configurations of one design.
The transport Stage 8 specifies is stateless and free of session affinity, which invites horizontal
scaling and does nothing to prevent it. The stores are in-process. Those facts are compatible only
while exactly one instance runs, and an unenforced assumption of that shape does not fail loudly — it
fails as a plan issued by one instance being invisible to another, and an already-paid check silently
narrowing to whichever instance took the call.

Stage 10 also bears on it. Its homelab deployment runs one dev instance for functional investigation.
A multi-instance live topology would mean the shared store shipped exercised by nothing, while the dev
deployment sat there implying an assurance it had not produced.

## Decision

**The live deployment runs a single server instance.** The in-process plan store and payment record
store are therefore sufficient, and horizontal scaling of this service is out of scope for version 1.

Enforce it at startup rather than documenting it: a configuration declaring more than one instance
must fail. Single instance is only safe while something refuses the alternative, and the failure mode
it prevents is a deployment scaling to two replicas because the transport is stateless and nothing
stopped it.

Keep both stores behind their interfaces, so revisiting this is a substitution rather than a rewrite
of the payment path.

Rejected: multi-instance operation with an externally shared plan store. It would add the one database
dependency the contract otherwise excludes, and Stage 10's single dev instance would not exercise it —
so it would ship unproven. Nothing about the choice is hard to revisit: the transport is stateless
regardless, and Stage 3 already requires the cursor-signing key to be identical across instances.

`plans/10` is corrected in the same change. Its dev deployment now happens to agree with the live
topology, and the documentation must say the agreement is a consequence of Stage 8's decision rather
than something the dev deployment established — a dev deployment proves nothing about a live topology
even when the two match.

## Consequences

Stage 8 gains a startup gate it did not have, and Stage 9's documentation gains an obligation: state
the single-instance limit plainly, and state that an outstanding plan does not survive a restart. The
second is acceptable for a five-minute capability — the caller re-plans, and re-planning re-runs the
fresh approval check — but an operator should read it in a runbook rather than discover it during a
deployment.

Stage 11's audit item on topology is unchanged and remains correctly phrased: it asks whether the
declared topology is actually enforced, which is now a question with a checkable answer.

`plans/06` is left as written. Its remark that the interfaces are what let Stage 8 declare a
multi-instance topology was true when written and describes why the seam exists; rewriting it would
be tidying history rather than recording a decision.

Left deliberately open: nothing here forecloses multi-instance operation. It becomes available by
supplying shared implementations of two interfaces and raising the declared instance count — a
reviewed contract change with a store behind it, not a configuration edit.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
