---
date: 2026-08-02
slug: add-stages-18-and-19-for-the-ai-mcp-server-development-deployment
title: "Add Stages 18 and 19 for the ai-mcp-server development deployment"
summary: "Add two stages after Stage 17. Stage 18 makes caller authentication exactly one explicitly selected mode—Keycloak or one fixed opaque bearer token—with separately named startup failures and no fallback."
kind: product
status: accepted
sequence: 2026-08-02T10:10:54.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/50; merge commit 407deb1fe531484211c13cfaf832d38aa4cd76be"
---

## Context

Stage 17 deployed the server from an entry point inside the server repository: an immutable GHCR
digest, a dedicated Keycloak contract, HTTPS through shared Caddy, payment disabled, and
real-secret bootstrap blocked until LocalStack was proved unreachable from the LAN. The
`ai-mcp-server` host now uses LocalAI scripts as the deployment owners for its MCP services, and
the clients this deployment serves need a fixed-header option as well as Keycloak.

The deployment plan was approved and frozen after adversarial review. Implementation review then
showed that the generated Stage 18 wording crossed its stage boundary by assuming Stage 19's shared
realm/client, while the LocalAI implementation guessed a tenant-specific BrightFlag status. The
same review found credential, Caddy, Keycloak, secret-lifecycle, rollback, state, and removal
behaviour that needed an explicit disposition rather than silent implementation drift.

## Decision

Add two stages after Stage 17. Stage 18 makes caller authentication exactly one explicitly selected
mode—Keycloak or one fixed opaque bearer token—with separately named startup failures and no
fallback. It preserves Prompt 17's dedicated `brightflag-mcp` realm and pre-registered public
client.

Stage 19 retires the server-owned deployment, names LocalAI as sole deployment owner, moves the host
deployment to the shared `homelab` realm and `mcp-client`, adds an explicit plaintext transport
mode and exact host validation, and enables payment in both authentication modes against
BrightFlag's integration-test environment. The reviewed tenant's approved-invoice status is an
operator-supplied value with no default.

Save the implementation review as a separate artifact. Correct the GitHub credential,
Caddy-activation, and false-success stop/removal defects in LocalAI. Track complete Keycloak setup,
exact mapper read-back, and auth-independent fixed-token materialisation in LocalAI issue #28.
Accept or skip the explicitly directed home-lab findings without presenting them as fixed.

## Consequences

Stage 18 can be applied to the Prompt 17 baseline without depending on a topology introduced later,
and Stage 19 receives customer-specific vocabulary without inventing it. The server repository
stops owning deployment only when Stage 19 is applied, and both authentication modes retain the
complete read and payment surface.

The open home-lab posture remains visible: bearer credentials cross the LAN in plaintext,
LocalStack is unauthenticated and LAN-reachable, generated Compose omits hardening and resource
limits, deployment state is written before health is proven, and rollback uses the retained tag
under operator observation. Fixed-token materialisation is currently conditional on fixed mode.
The dedicated Keycloak correction remains follow-up work in LocalAI #28 rather than being partially
claimed here.

End-to-end proof of the private-source matcher remains in Builder #45. The saved implementation
review records every finding, including the skipped production-origin check and the still-open
device-token claim-set gate.
