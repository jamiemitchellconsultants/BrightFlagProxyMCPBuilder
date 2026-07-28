# Prompt 8 — Expose the narrow MCP surface securely

Using the reusable contract, implement Stage 8: hosting, identity, and authorization for exactly
four tools and one resource.

## Hosting

Support two transports from one composition root:

- local stdio for a desktop client; and
- stateless Streamable HTTP at `/mcp`, authenticated, with no session affinity and no server-held
  caller state beyond the plan store.

Register the tool set statically. Fail startup if the registered surface is not exactly the four
tools and one resource, so a future refactor cannot quietly widen it.

## Caller identity

Under HTTP, validate the caller's bearer token in-process before any handler runs: signature,
issuer, audience, expiry, not-before, and required claims, with clock skew bounded. Reject
unsigned, `none`-algorithm, expired, or wrong-audience tokens. Cache signing keys with a bounded
lifetime and never fetch keys per request.

The caller's token authenticates the caller to this server. It is never forwarded to BrightFlag,
and BrightFlag's service token is never returned to a caller. Prove both directions in tests.

Under stdio, derive identity from the local configured principal and refuse to run in a mode where
no principal is established.

## Caller-identity trust provider

Make the JWT trust root — issuer, audience, and signing-key source — entirely configuration-driven,
so the validation logic above never changes between environments; only where it trusts keys and an
issuer from changes. Support two trust-provider implementations:

- a live provider that fetches signing keys from a configured JWKS URL over HTTPS, matching the
  deploying organisation's identity provider, for example Entra ID's tenant discovery endpoint; and
- a local provider that loads signing keys from a configuration-supplied JWKS document, for local
  development and automated tests only.

Both providers feed the same signature, issuer, audience, expiry, not-before, and required-claims
validation defined above; neither weakens it and neither introduces a second code path through that
validation. The local provider changes only the source of trusted keys and the accepted issuer, per
the caching rule already stated.

Fail startup — never merely warn — when the local trust provider is selected under a deployment
profile not explicitly marked non-production. This is the caller-identity analogue of the
BrightFlag-origin rule in Prompt 3 that rejects a production profile using no authentication.

Provide a companion dev token-issuing tool, built the same way as Prompt 3's fake BrightFlag server:
given a requested caller identity (subject, tenant, roles, groups, and scope claims) and an
optional expiry, wrong issuer, wrong audience, or missing claim, it mints a token signed by the
local provider's key, matching the claim shape the live provider's tokens carry, so authorization
logic under test never special-cases which provider issued the token. The tool must itself refuse
to run under a profile marked production, and must not be present in the container image built in
Prompt 9.

## Deferred corporate alignment

This stage establishes the narrow surface, transport authentication, and server-side authorization
boundary, but does not claim final alignment with the deploying organisation's MCP authorization
profile, identity-provider discovery, gateway controls, or production trust policy. Record those as
an explicit deployment gap for later corporate-standard alignment; do not weaken the controls in
this prompt while that work is deferred.

## Authorization

Define two capabilities:

- read approved invoices; and
- mark an invoice paid.

Grant them from reviewed server-side configuration keyed on validated claims. Never from a caller
argument, a tool description, an annotation, prompt text, or a model's assertion. Deny by default,
evaluate per call including the execute step of a plan, and record every decision in the audit log.

The plan store is caller-scoped, capacity-bounded, and expiring. A plan issued to one caller is
invisible and unusable to another.

State and enforce exactly one topology: either the deployment runs a single server instance, so an
in-process plan store is sufficient and horizontal scaling of this service is out of scope for
version 1; or the deployment may run more than one instance, in which case the plan store must be
externally shared so a plan created against one instance can be executed against another without
session affinity, consistent with the stateless transport above. Do not ship an in-process-only
plan store silently alongside multi-instance deployment guidance elsewhere in the documentation.

## Hardening

- Enforce request-size, concurrency, and per-caller rate limits, with a stricter limit on the
  payment tool.
- Emit structured logs with a correlation identifier and no credential, token, or full payload.
- Reject unknown MCP arguments rather than ignoring them.
- Return typed errors that do not leak origin, path, query, header, or stack detail.
- Treat all BrightFlag response content as untrusted data, never as instruction.

## Tests

Prove:

- the registered surface is exactly the four tools and one resource, and startup fails otherwise;
- unauthenticated, expired, wrong-audience, wrong-issuer, and `none`-algorithm tokens are refused;
- a token from the live trust provider and a token from the local trust provider carrying identical
  claims produce identical authorization outcomes;
- startup fails when the local trust provider is selected under a profile marked production;
- the dev token-issuing tool refuses to run under a profile marked production and is absent from
  the container image;
- a token minted by the dev tool with a wrong issuer, wrong audience, missing required claim, or
  expired lifetime is rejected exactly as the equivalent live-provider token would be;
- a read-only caller cannot plan or execute a payment;
- a caller cannot execute another caller's plan;
- caller tokens never reach the outbound BrightFlag request and the BrightFlag token never reaches
  a response;
- rate limits and size limits are enforced;
- the declared topology holds: a plan created against one instance can be executed against another
  when multi-instance operation is declared, or single-instance operation is documented and
  enforced when it is not; and
- text embedded in a BrightFlag response that instructs the server to change behavior changes
  nothing.

## Acceptance criteria

- Both transports serve the same narrow surface with the same authorization outcome.
- Authorization is server-side, deny-by-default, and re-evaluated at execution.
- The plan store's single-instance or multi-instance topology is stated and enforced, not left
  implicit.
- The caller-identity trust root is swappable by configuration alone; no authentication code path
  differs between the live and local trust providers.
- A profile marked production cannot select the local trust provider, and the dev token-issuing
  tool cannot run under one or ship in the container image.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the decision to validate caller tokens
in-process, keep caller identity separate from the BrightFlag service credential, and make the
identity-provider trust root swappable by configuration rather than by code. Do not push unless
requested.
