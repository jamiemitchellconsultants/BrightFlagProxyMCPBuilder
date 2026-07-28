---
date: 2026-07-28
slug: add-a-swappable-caller-identity-trust-provider-for-local-development
title: "Add a swappable caller-identity trust provider for local development"
summary: "Keep the JWT validation logic in Prompt 8 completely unchanged, and make only the trust root - issuer, audience, and signing-key source - configuration-driven."
kind: product
status: accepted
sequence: 2026-07-28T07:19:54.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/7; merge commit f19de499f433d00d4ce52c4b20e43824760cd2a4"
---

## Context

Prompt 8 requires full JWT bearer validation for HTTP callers (signature, issuer, audience, expiry,
not-before, required claims), but assumed a live identity provider was always available to issue
tokens against. That's a real obstacle for someone deploying the built server to a container on a
home-lab host for functional investigation, with no interest in standing up or registering with a
corporate IdP just to make test calls.

The tempting shortcut - a shared static secret instead of a JWT - was considered and rejected: it
directly contradicts Prompt 8's own instruction not to weaken its controls while corporate-IdP
alignment is deferred, and it would mean the server's auth code path in dev is not the one that ships.

## Decision

Keep the JWT validation logic in Prompt 8 completely unchanged, and make only the trust root -
issuer, audience, and signing-key source - configuration-driven. This is the same secret-provider
abstraction already used for the BrightFlag bearer token and the cursor-signing key
(environment/file-reference/fake-style providers), applied to caller identity: a live provider
fetches signing keys from a configured JWKS URL; a local provider loads a configuration-supplied
JWKS document, for development and automated tests only. Both feed the identical validation, so
there is exactly one authentication code path, not two.

Added a companion dev token-issuing tool, built the same way as Prompt 3's fake BrightFlag server,
that mints tokens matching the live provider's claim shape (subject, tenant, roles, groups, scope),
including deliberately malformed variants (wrong issuer, wrong audience, missing claim, expired) for
negative testing.

Guarded both additions the same way Prompt 3 already guards the BrightFlag origin: fail startup -
never merely warn - if the local trust provider is selected under a profile not explicitly marked
non-production, and exclude the dev token tool from the delivered container image entirely rather
than just leaving it inactive. Extended Prompt 9's threat model, runbook (how to switch providers
when moving off the home lab), and packaging, and Prompt 10's audit (point 30, plus a new
adversarial-test line) to cover the swap and its production guard.

Rejected alternative: a shared static bearer secret. It would have been less work but is exactly the
control-weakening Prompt 8 already forbids, and it means dev and prod never exercise the same
authentication path.

## Consequences

Anyone building from these prompts can now deploy the server to a container on their own
infrastructure and drive it from a local test harness generating arbitrary dummy caller identities,
without any dependency on a live identity provider, while the server's authentication logic is
identical to what ships in production. The cost is a second, symmetric guard (startup failure plus
image-exclusion) that has to be built and tested alongside the feature itself, and a runbook entry
for the one manual step - pointing the trust provider at a real IdP - that a team must not skip when
moving off local development.
