---
date: 2026-07-31
slug: add-local-deployment-runbook-stage
title: "Add local deployment runbook stage"
summary: "Add optional Stage 15 after Stage 14."
kind: product
status: accepted
sequence: 2026-07-31T20:27:41.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/41; merge commit 08b5557fc684631fb4436abe9e03ff4ead7db2b7"
---

## Context

Prompt 14 installed the private, main-only GitHub delivery path, but intentionally left `.env`, host
secrets, public JWKS, TLS material, DNS, router state, and tenant-specific configuration outside
GitHub. The intended host also already runs Caddy on ports 80 and 443, while the BrightFlag nginx
proxy has a reviewed dedicated listener on 8443. A replayable prompt was needed so a future agent
does not infer secrets, treat stale host observations as current, choose an arbitrary image digest,
or silently couple the two edge deployments.

## Decision

Add optional Stage 15 after Stage 14. It requires `docs/deploy-local.md` to distinguish observed
state, operator inputs, and expected but unrun results; preserve Caddy; use the BrightFlag proxy's
8443 listener; select the final published manifest digest with its exact source revision; keep local
signing material off the host; and validate a read-only endpoint before handing its URL, bearer
header requirement, and certificate trust to OpenCode.

The stage remains documentation-only. Router changes are manual, GitHub deployment cannot enable
payment, and mutable observations must be refreshed whenever the prompt is replayed.

## Consequences

The first deployment now has a bounded, reviewable bridge from successful image publication to a
callable local MCP endpoint. Operators still have to supply the BrightFlag tenant facts, service
credential, DNS, certificate, and private caller key through channels outside Git and prompts.
Keeping a separate 8443 listener avoids changing Caddy but means the public URL includes a port and
the router needs a distinct mapping. Short-lived local tokens also require renewal until the
deployment moves to the organisation's live identity provider.
