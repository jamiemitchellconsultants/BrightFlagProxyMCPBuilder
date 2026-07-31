---
date: 2026-07-31
slug: make-prompt-8-rollover-tests-deterministic
title: "Make Prompt 8 rollover tests deterministic"
summary: "Correct Prompt 8 and its Stage 8 plan to require two proof layers. Dedicated live-provider tests exercise actual loopback HTTP retrieval and fail-closed unavailability."
kind: product
status: accepted
sequence: 2026-07-31T19:55:59.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/39; merge commit 19fafcdaf05bce421f1bb5843fda8762c4ecf559"
---

## Context

Prompt 8 required live and local JWKS providers plus bounded signing-key caching, but did not separate
transport integration evidence from rollover and cache semantics. The resulting server used a
loopback HTTP endpoint inside rollover tests. A transient timeout returned
`TrustRootUnavailable`; because the warm-up result was discarded, CI reported only that the
successful fetch count was zero.

## Decision

Correct Prompt 8 and its Stage 8 plan to require two proof layers. Dedicated live-provider tests
exercise actual loopback HTTP retrieval and fail-closed unavailability. Cache lifetime, forced
refresh, rollover, rate-limiting, and refresh-failure tests use a deterministic mutable in-process
key source underneath the production cache and validator. Every cache warm-up must prove
authentication succeeded before fetch counts are asserted.

## Consequences

Future prompt replays retain real transport integration coverage without making cache and rollover
semantics depend on listener scheduling, sockets, DNS, or HTTP timeouts. A genuine warm-up failure
will report the authentication cause directly instead of masquerading as a fetch-count defect. The
server's production trust model and validation behavior do not change.
