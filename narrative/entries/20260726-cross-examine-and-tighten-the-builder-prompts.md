---
date: 2026-07-26
slug: cross-examine-and-tighten-the-builder-prompts
title: "Cross-examine and tighten the builder prompts"
summary: "Correct overclaims and sequencing gaps found when a different AI cross-examined the AI-generated prompt set."
kind: product
status: accepted
---

## Context

The staged prompts were generated with substantial AI assistance and then cross-examined by a
different AI from the one that generated them. That independent review found requirements that were
individually plausible but inconsistent when read across stages: an impossible exactly-once claim
around a non-idempotent external write, contracts requested before their authoritative OpenAPI
source was acquired, a cursor contract that promised both statelessness and no repeated upstream
reads, an unwindowed read allowed despite a global bounded-window rule, a recursively defined schema
fingerprint, and a cross-tenant fixture unsupported by any authoritative payload field.

The review also identified MCP corporate-alignment work and detailed payment outcome classification
that remain valid follow-up concerns, but which are deliberately deferred rather than silently
presented as complete.

## Decision

State the payment invariant as at most one outbound POST attempt per atomically consumed plan token,
never exactly-once delivery. Move acquisition and validation of the public BrightFlag OpenAPI
snapshot into Prompt 1, before API-derived contracts are defined. Use a stateless signed keyset
cursor: bounded upstream reads may repeat, but previously emitted keys are filtered out and snapshot
isolation is not claimed. Remove the unwindowed runtime batch-read allowance. Define the ontology
fingerprint over canonical JSON with the fingerprint property omitted. Relax the cross-tenant audit
case when the reviewed payload supplies no authoritative tenant discriminator.

Keep full corporate MCP authorization alignment and the detailed payment-response outcome matrix as
explicit later work. Do not broaden the current implementation while those items are deferred.

## Consequences

The prompts now ask agents to enforce properties the server can actually guarantee and give them
authoritative API evidence before they stabilize contracts. Pagination remains stateless at the cost
of repeated bounded reads and without pretending that a mutable upstream API provides snapshot
isolation. The payment flow still cannot prove exactly-once delivery across the external HTTP
boundary; it instead makes one attempt per consumed plan and quarantines ambiguity.

More broadly, this correction records a process requirement: AI-generated designs and prompts must
be cross-examined, preferably by an independent reviewer with different context or a different
model. Fluency is not evidence of consistency. Review must trace requirements across stages,
challenge absolute guarantees, and test whether every acceptance criterion has an implementable
state and evidence model.
