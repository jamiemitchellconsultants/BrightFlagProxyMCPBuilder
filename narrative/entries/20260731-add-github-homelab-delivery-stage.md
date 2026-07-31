---
date: 2026-07-31
slug: add-github-homelab-delivery-stage
title: "Add GitHub homelab delivery stage"
summary: "Add optional Stage 14. GitHub-hosted CI builds and verifies an immutable image on protected `main`; a repository-scoped Windows runner may deploy it only after the server repository is private."
kind: product
status: accepted
sequence: 2026-07-31T19:12:04.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/37; merge commit d2e6116ff29e428e4170c60bc10922996f8807e9"
---

## Context

The existing homelab stage required manual image delivery and prohibited router forwarding. A
reviewed image now needs to reach `ai-mcp-server` through GitHub without allowing public-repository
pull-request code to control a persistent runner with Docker and secret-bearing container access.

## Decision

Add optional Stage 14. GitHub-hosted CI builds and verifies an immutable image on protected
`main`; a repository-scoped Windows runner may deploy it only after the server repository is
private. Where private-repository environment reviewers are unavailable, deployment is a separate
main-only manual dispatch. The runner executes only a reviewed host-installed script. Router
forwarding is an explicit TLS-only reachability mode and is never treated as authorization.

## Consequences

Docker access remains privileged even under a dedicated runner account. Host secrets stay outside
GitHub and the Actions workspace, payment cannot be enabled by the workflow, and turning off the
router forward provides isolation without replacing JWT or capability checks. Stages 12 and 13
remain contingent and are neither required nor authorized by this stage.
