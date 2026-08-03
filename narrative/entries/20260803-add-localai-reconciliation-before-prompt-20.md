---
date: 2026-08-03
slug: add-localai-reconciliation-before-prompt-20
title: "Add LocalAI reconciliation before Prompt 20"
summary: "Introduce Prompt 19L, played separately in LocalAI after Prompt 19 and before Prompt 20."
kind: product
status: accepted
sequence: 2026-08-03T10:03:14.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/58; merge commit 1ce608d2337b3137702667eb51d47789b79175aa"
---

## Context

Prompt 20 deliberately leaves LocalAI untouched because the shared `homelab-mcp` audience, public HTTPS ingress, constant claims, dynamic registration, and issuer alias were independently prepared there. Replaying Prompt 19 correctly restores its earlier BrightFlag deployment boundary, but also rewinds BrightFlag-specific LocalAI values that Prompt 20 assumes. A whole-file or historical-tree restore would risk deleting later shared-host and other-MCP work.

## Decision

Introduce Prompt 19L, played separately in LocalAI after Prompt 19 and before Prompt 20. It restores BrightFlag's Prompt 20 values semantically, treats shared infrastructure as a verified external prerequisite, and forbids merge reversion or wholesale file restoration. It retains Prompt 18 fixed-token authentication and Prompt 19 ownership, build, rollback, secrets, host validation, payment, and open-posture controls. It also requires explicit `CallerIdentity__Mode=Keycloak` and forbids restoring the obsolete claim that fixed-token support is absent.

## Consequences

The sequence gains one cross-repository review boundary, but Prompt 20 remains truthfully server-only and independently prepared LocalAI work is protected. Missing shared-host prerequisites now stop reconciliation instead of causing a duplicate realm or ingress implementation. Prompt 19 remains valid and historical at its own boundary; Prompt 19L selectively prepares its deployment owner for the next server stage.
