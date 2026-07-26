---
date: 2026-07-26
slug: tighten-prompt-guarantees-after-independent-ai-cross-examination
title: "Tighten prompt guarantees after independent AI cross-examination"
summary: "Express the payment invariant as at most one external attempt per consumed plan, move authoritative OpenAPI acquisition before contract definition, adopt stateless keyset pagination with honest consistency semantics, remove unbounded…"
kind: product
status: accepted
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/3; merge commit 661685a6156292eb288ae18380756a588247e877"
---

## Context

The prompts were generated with substantial AI assistance. A different AI independently cross-examined them and found requirements that sounded plausible in isolation but conflicted across stages or claimed guarantees the proposed state model could not provide.

## Decision

Express the payment invariant as at most one external attempt per consumed plan, move authoritative OpenAPI acquisition before contract definition, adopt stateless keyset pagination with honest consistency semantics, remove unbounded reads, make schema fingerprinting non-recursive, and avoid requiring tenant evidence absent from the reviewed payload.

Keep full corporate MCP authorization alignment and detailed payment-response classification as explicit later work.

## Consequences

The guide now asks implementation agents to enforce properties that can be evidenced. Pagination may repeat bounded upstream reads and does not claim snapshot isolation. The payment flow cannot claim exactly-once delivery across a non-idempotent external boundary.

The decision history also establishes a review principle: fluent AI-generated output is not evidence of consistency and should be cross-examined, preferably by an independent reviewer or model.
