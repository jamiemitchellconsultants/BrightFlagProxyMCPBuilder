---
date: 2026-07-29
slug: settle-the-open-mechanisms-in-the-stage-12-and-13-plans
title: "Settle the open mechanisms in the Stage 12 and 13 plans"
summary: "DynamoDB backs both shared stores, chosen for atomic conditional writes rather than throughput; strong reads and TTL-as-cleanup-only are correctness requirements; rate limits split between a shared counter and the edge."
kind: architecture
status: accepted
sequence: 2026-07-29T20:36:23.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/24; merge commit ce5bde1bd4ff7f92dc6df282ba20200fec66c2d1"
---

## Context

The contingent stages added earlier the same day state *what* a multi-instance reclassification must
achieve but leave the mechanism open: which shared store, where rate limiting moves to, and how
AWS-managed secrets reach a server whose secret-provider contract forbids a vendor-specific vault.
`plans/README.md` says an execution plan exists to settle exactly that class of question before
implementation, rather than leaving it to whoever executes the stage.

Two of these are traps rather than preferences, and both share a property that makes them worth
settling in advance: they fail only under conditions a local test does not reproduce, so a plan
written during the reclassification itself would most likely get them wrong and the error would not
surface until production.

## Decision

**DynamoDB backs both shared stores**, selected on conditional-write and consistency semantics rather
than throughput. At a few interactions an hour throughput is irrelevant; the atomic compare-and-set
behind plan consumption is the entire requirement, and it is what makes the losing instance in a race
learn that it lost from the store rather than from a second read.

Two properties are recorded as correctness requirements, not tuning:

- **Reads in the payment path must be explicitly strongly consistent.** The default is eventually
  consistent, which passes every test run against an idle local instance and fails intermittently
  under concurrent load — the worst possible failure signature for a payment path.
- **Time-to-live is for cleanup only, never for expiry.** TTL deletion is asynchronous and can lag by
  as much as two days. A plan whose expiry was delegated to it would stay usable far beyond its
  intended short lifetime, silently widening the window the plan/confirm split exists to narrow.

**Rate limiting splits.** The strict per-caller payment limit moves to a shared counter, because it is
the one limit whose exactness is load-bearing. Coarse request-rate limits move to the edge, and the
in-process limiter is **removed** rather than left in place to multiply by instance count.

**Secrets reconcile with the no-vendor-vault rule** by having ECS inject Secrets Manager values via
`valueFrom`, so the existing environment and file-reference providers read them unchanged and no AWS
SDK enters the core. The rule is honoured rather than excepted.

## Consequences

The stages are executable without re-deriving these positions under the time pressure a
reclassification would bring, and each plan records what would have to change to revisit its
position.

Both stages remain contingent. A reader who implements Stage 12 because it is numbered has added an
availability dependency in front of a financial write for no reason, so the contingency is stated in
the prompts, the plans, and the index rather than in one place.

`plans/README.md` grows from three open decisions to five.

Stage 11's audit is still **not** extended to cover exactly-once payment behaviour under concurrency.
That is recorded as an open gap rather than closed, because adding audit points for a stage that may
never run would assert more confidence in the reclassification than currently exists.

This entry was written by hand. The pull request carried `narrative-required` but the label was
applied while its body still lacked the three required `## Narrative …` headings, and it merged inside
that window — so the maintenance workflow failed rather than proposing a fragment. The body was
corrected afterwards, which changes nothing: the action reads the body from the merge event payload,
so a re-run reads the same incomplete text. The merge-event-only trigger makes hand-authoring the only
available repair, which is precisely the limitation `CLAUDE.md` §2 now records.
