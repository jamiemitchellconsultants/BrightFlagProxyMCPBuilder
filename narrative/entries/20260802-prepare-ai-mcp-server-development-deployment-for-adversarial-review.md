---
date: 2026-08-02
slug: prepare-ai-mcp-server-development-deployment-for-adversarial-review
title: "Prepare the ai-mcp-server development deployment for adversarial review"
summary: "Save and revise the proposed implementation plan through adversarial review."
kind: architecture
status: proposed
---

## Context

The requested home-lab development path differs from Stage 17 in authentication, image sourcing,
payment enablement, network names, LocalStack exposure, Keycloak realm, and transport security. The
existing prompts must remain historical, BrightFlag server changes must still arrive through later
builder prompts, and LocalAI must own the Windows host deployment.

Implementing those decisions directly would make it difficult to distinguish a defect in the plan
from a defect introduced while applying it. A standalone artifact was needed so another reviewer
can challenge the cross-repository boundaries, secret lifecycle, full-access payment behavior,
plaintext LAN routing, Keycloak flow, and production isolation before any numbered prompt or host
script is written.

The review found an inaccurate HTTPS trusted-proxy model for the plaintext endpoint, a moving-ref
build race, and several areas where a production-oriented design would add restrictions outside
the deliberately open home-lab requirement.

## Decision

Record a proposed implementation plan under `reviews/` rather than adding a numbered stage before
review. The plan proposes two later stages: one for exclusive fixed-token or Keycloak
authentication with full home-lab capabilities, and one for the `ai-mcp-server` Git-ref deployment.
It also assigns the Windows deployment script and shared-infrastructure integration to LocalAI.

The review brief asks an adversarial reviewer to test assumptions and contradictions without
editing the plan or implementation repositories. Existing prompts and the BrightFlag server remain
unchanged at this point.

Revise the plan to add an explicit plaintext application transport and to resolve and build an
exact commit with retained-image rollback. Do not add production-profile restrictions to the
fixed-token shortcut or rewrite Prompt 17 history. Preserve the Prompt 17 LAN-only Caddy source
policy, but defer its public-hostname verification, the final Keycloak caller contract, and further
LocalStack secret-lifecycle work to issues [45][issue-45], [46][issue-46], and [47][issue-47]
respectively.

## Consequences

The agreed direction is now reviewable as one bounded artifact, including the places where it
overrides Stage 17. No numbered prompt or deployment script exists yet, so an adverse finding can
change the proposed plan without rewriting prompt history or unwinding implementation work.

The plan now makes the moving-ref build reproducible while leaving the fixed-token and plaintext
home-lab paths intentionally open. The three deferred issues provide visibility but do not block
the initial deployment plan.

The plan itself is not approval to implement. A later decision must accept or correct it before the
builder stages and LocalAI script are created.

[issue-45]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/45
[issue-46]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/46
[issue-47]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/47
