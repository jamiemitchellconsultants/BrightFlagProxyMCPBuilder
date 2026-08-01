---
date: 2026-08-01
slug: replace-runner-deployment-with-local-pull
title: "Replace runner deployment with local pull"
summary: "Retire every GitHub-triggered local deployment and retain only GitHub-hosted CI and verified, digest-pinned GHCR publication. Deploy through a local PowerShell entry point that accepts an exact digest and revision."
kind: product
status: accepted
sequence: 2026-08-01T05:32:09.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/43; merge commit b6e7d19e120b66cb7dc367e76ace6b3a4484c846"
---

## Context

The prior local-deployment stages placed a persistent Docker-capable GitHub Actions runner on
`ai-mcp-server` and kept BrightFlag behind a dedicated nginx listener. The destination host now
has shared LocalStack Secrets Manager, Keycloak, and Caddy infrastructure, and deployment should be
initiated by a trusted operator on the local machine. LocalStack is currently unauthenticated, so
real BrightFlag credentials must not be stored while its endpoint is reachable from another LAN
machine. The replacement must also preserve Jamie's long-lived SSH access and the server's existing
vendor-neutral secret-provider boundary.

## Decision

Retire every GitHub-triggered local deployment and retain only GitHub-hosted CI and verified,
digest-pinned GHCR publication. Deploy through a local PowerShell entry point that accepts an exact
digest and revision. The host retrieves configured LocalStack Secrets Manager values over loopback
and atomically materialises ACL-protected files for the existing File providers; the server does not
gain an AWS SDK or vendor-specific provider. Keycloak is the canonical live token issuer, the server
publishes OAuth Protected Resource Metadata, and Caddy owns TLS termination through one
BrightFlag-owned, LAN-only fragment on the shared edge network. Payment remains disabled.

## Consequences

Fresh implementations skip superseded Stages 14–16 and apply Stage 17 after Stage 11; repositories
that already applied Stages 14 or 15 run Stage 16 before Stage 17. The GitHub runner, environment,
and runner-only host assets require an inventory-first manual retirement, while Jamie's SSH access
and unrelated services are preserved. Real-secret deployment fails closed until LocalStack is
unreachable from the LAN. BrightFlag no longer owns an nginx listener or router-specific isolation;
its Caddy route becomes the service-specific cut-off, and shared Caddy, Keycloak, LocalStack,
PostgreSQL, and `edge_net` remain externally owned infrastructure.
