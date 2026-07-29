---
date: 2026-07-29
slug: prepare-contingent-stages-for-a-multi-instance-reclassification
title: "Prepare contingent stages for a multi-instance reclassification"
summary: "Add Stages 12 and 13 as explicitly contingent, to run only if governance reclassifies this service off the low-use basis for single instance. Stage 12 restores the one-payment guarantee under concurrency; Stage 13 deploys it on AWS."
kind: product
status: accepted
sequence: 2026-07-29T20:22:36.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/22; merge commit a5668baafec7d98a95a08d958f32731fff7f180f"
---

## Context

Stage 8's single-instance topology is permitted by corporate governance because this service is
classified low-use and low-criticality — predicted load is a few interactions an hour. That
classification is a fact about today, not a property of the design, and it can be revisited by people
who are not looking at this repository when they do it.

The risk in that is specific. The single-instance decision was never a scaling opinion; it was the
*justification* for `IPlanStore` and `IPaymentRecordStore` being in-process at all. Withdrawing the
justification does not break anything visibly. A second replica starts, the transport is stateless and
session-affinity-free, health checks pass, and reads work. What fails is that a plan issued by one
instance is invisible to the other, and the already-paid check narrows to whichever instance took the
call — surfacing not as an error but as a duplicate payment assertion in a system finance teams
reconcile against real money.

Left to be improvised under time pressure, the reclassification would most likely be handled as an
infrastructure change: raise the replica count, add a shared store, ship. The store substitution is the
easy part and the interfaces were built for it. Restoring the guarantee is not, and is the part a hurried
change would skip.

## Decision

**Add Stages 12 and 13, both explicitly contingent**, so the transformation is a prepared sequence
rather than an improvisation. Neither is part of version 1 and neither is to be implemented
speculatively; Stage 12 requires the reclassification to be recorded and cited before it begins.

Stage 12 carries the substance and is deliberately framed as a correctness stage, not a scaling one:

- shared implementations of both stores, selected and evidenced against atomic conditional writes and
  strongly consistent reads, with the in-process pair retained because Stage 10 still uses it;
- plan consumption as a **single atomic conditional transition committed before the outbound POST**, so
  the losing instance is told it lost by the store rather than by a second read. Every check-then-act in
  the payment path becomes a conditional write whose failure *is* the rejection;
- Stage 8's startup gate **inverted rather than deleted** — multi-instance with in-process stores becomes
  a startup failure, preserving the principle that a wrong topology is a refusal rather than an
  intermittent payment defect;
- per-caller rate limits, which silently multiply by instance count and turn a documented limit of N into
  an enforced limit of N × instances;
- contract-version rejection across a rolling deploy, which a single-instance deployment never faced.

Stage 13 deploys it on AWS — ECS Fargate, an ALB as the trusted fronting proxy the new TLS posture
expects, scoped ingress and egress, secrets through the existing providers — and depends on Stage 12
rather than standing alone.

Recorded explicitly: this **overrides the contract's exclusion of a database dependency**, by authority
of the reclassification and nothing else. The preceding topology entry rejected a shared plan store
partly on that exclusion, and the override is filed as an override rather than as a discovery that the
exclusion was mistaken.

## Consequences

The sequence now contains two stages that are not meant to run, which is a new shape for it and needs
the contingency stated in the prompts, the plans, and the plans index — a reader who finds Stage 12 and
implements it because it is numbered has done real harm, since a shared store is a new availability
dependency in front of a financial write.

Writing the stages before they are needed produced findings that would have been expensive to discover
during the change itself, and which are the actual value of doing this early: that plan expiry cannot be
delegated to a store's time-to-live sweep without silently widening every plan's lifetime, that a store
defaulting to eventually consistent reads will pass every test run against an idle local instance and
fail intermittently in production, and that the rate-limit multiplication makes a documented control
false rather than merely loose.

Stage 6's note that keeping both stores behind interfaces would make this a substitution is now testable
rather than aspirational, and Stage 12 discharges it explicitly.

Stage 11's audit is **not** yet updated. Its topology points already read "single-instance or
multi-instance", which was written to accommodate either, but nothing in the audit yet examines
exactly-once payment behaviour under concurrency. That gap is recorded here rather than closed, because
adding audit points for a contingent stage that may never run would assert more confidence in the
reclassification happening than currently exists.
