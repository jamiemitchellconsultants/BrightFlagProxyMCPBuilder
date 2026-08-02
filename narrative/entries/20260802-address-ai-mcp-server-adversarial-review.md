---
date: 2026-08-02
slug: address-ai-mcp-server-adversarial-review
title: "Address ai-mcp-server adversarial review"
summary: "Revise the proposed plan to require flat Keycloak `tid` and `roles` claims, container-to-issuer host-gateway routing, later retirement of the Prompt 17 server-local deployment artifacts, deterministic Git-ref build state, and correctly…"
kind: product
status: accepted
sequence: 2026-08-02T06:00:21.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/48; merge commit c68065794fbe412fd7a492d506ab029a3e5e4e70"
---

## Context

The initial plan correctly captured the intended home-lab authority boundaries, but adversarial review found that the proposed Keycloak option could not satisfy the existing server claim and issuer-resolution contract. The review also exposed an active Prompt 17 deployment owner, a moving-ref implementation gap, and verification claims that the Builder-only work could not prove.

## Decision

Revise the proposed plan to require flat Keycloak `tid` and `roles` claims, container-to-issuer host-gateway routing, later retirement of the Prompt 17 server-local deployment artifacts, deterministic Git-ref build state, and correctly scoped verification. Preserve the intentionally open fixed-token, plaintext LAN, and LAN-accessible LocalStack decisions; describe their consequences plainly rather than adding production controls.

## Consequences

Keycloak is now specified as a functioning initial authentication option. The plan remains proposed rather than implementation approval, and later prompts remain the only route for changes to BrightFlagProxyMCPServer. Source-address proof, shared-client separation, and later secret-lifecycle work remain visible in issues #45, #46, and #47.
