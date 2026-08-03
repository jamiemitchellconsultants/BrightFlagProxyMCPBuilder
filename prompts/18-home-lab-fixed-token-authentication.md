# Prompt 18 — Home-lab fixed-token authentication

Using the reusable contract and the artifacts produced by Prompts 1–11 and 17, add a second
caller-authentication mode to `BrightFlagProxyMCPServer`: one fixed opaque bearer token, selected
explicitly at startup, for a physically controlled home-lab development network.

Prompt 17 is merged and applied before this stage. This prompt does not remove, revise, or weaken
the Keycloak path Prompt 17 established. It adds an alternative beside it and makes the choice
between the two exclusive. It changes only the server; Prompt 19 deploys the result.

This stage does not depend on or authorise contingent Prompts 12 or 13, and it does not change the
BrightFlag service credential, which is a separate contract that no caller ever supplies.

## Select exactly one mode, and fail closed on everything else

Extend the caller-identity configuration with an authentication mode chosen once, at startup, from
reviewed configuration. Exactly two modes exist:

- **Keycloak** — the JWT trust provider, issuer, audience, and claim contract Prompt 17
  configured, unchanged; and
- **fixed token** — one opaque bearer token compared against a value read from a mounted file.

Startup must fail, never warn and continue, when the configured mode is:

- **absent** — no mode selected;
- **unknown** — a value that names neither mode;
- **ambiguous** — configuration supplying both modes' required values, or otherwise selecting more
  than one; or
- **incomplete** — the selected mode missing a value it requires.

Name those four cases separately in the failure, because they are four different operator mistakes
and a single "misconfigured" message sends an operator to the wrong place. The failure must name
which mode was selected and which value was missing, and must never echo a secret value.

There is no fallback in either direction, under any condition:

- in Keycloak mode, an unreachable issuer, a failed discovery or JWKS retrieval, an expired key
  cache, or a rejected token refuses the request — the server must never try the fixed token; and
- in fixed-token mode, a Keycloak-issued JWT is not a credential. It is an opaque string that fails
  the fixed comparison, and it must never reach JWT parsing, signature verification, or
  `CallerTokenValidator`.

Exactly one authentication branch is reachable after the startup selection. Make that structural
— one branch is constructed, the other is not — rather than a runtime condition evaluated per
request that a later refactor can invert.

## Implement the fixed-token provider

- Read the token from a file mounted into the container, using the existing `File` secret source.
  Do not accept it from an environment variable, a command argument, a repository fixture, or a
  configuration literal.
- Fail startup when the file is absent, unreadable, empty, or whitespace-only, before the transport
  accepts a request.
- Compare the presented credential to the configured token in constant time. Reject a malformed or
  absent `Authorization` header before comparison, and never let comparison time, error text, or
  response timing distinguish a near-miss from any other rejection.
- Redact the token completely. It must never appear in a log line, audit record, error response,
  health payload, diagnostic bundle, metric label, or exception message, at any log level.
- Answer an unauthenticated request with a plain `WWW-Authenticate: Bearer` challenge. In
  fixed-token mode the server serves no OAuth protected-resource metadata, advertises no
  authorization server, and puts no `resource_metadata` parameter or scope in the challenge, because
  the clients this mode exists for send a fixed header from a configuration file and run no OAuth
  flow. Do not offer Keycloak discovery to a caller the server will not accept a Keycloak token
  from.

## Establish the shared caller identity from configuration alone

In fixed-token mode a successful comparison establishes one deterministic caller identity, built
entirely from reviewed configuration:

- a configured subject;
- the configured tenant boundary; and
- the full read and payment roles.

Grant that identity both capabilities: it may list tools, read the ontology resource and schema,
list invoices approved for payment, plan a payment, and execute that plan. Nothing in the identity
may be derived from, or influenced by, the presented credential.

State these consequences plainly in the code documentation and in `docs/`, without qualifying them
away:

- **Every token holder is the same caller.** Audit attribution, the per-caller rate-limit bucket,
  and the caller-bound plan scope all collapse onto one identity. A payment plan issued to that
  identity can be executed by any holder of the token, so caller-scoping no longer distinguishes
  callers.
- **The fixed-token path does not traverse `CallerTokenValidator`.** It has no token expiry, no
  revocation, and no tenant-claim corroboration; subject, tenant, and roles come from static
  reviewed configuration and from nothing else. Rotation means replacing the file and restarting.
- **The token is only as private as the store it comes from.** The server does not control that
  store. Whatever can read the mounted file, or the store the file is materialised from, holds the
  credential outright, and the constant-time comparison is not that boundary. This stage presumes
  the home-lab deployment will keep the token in the same secret store as the cursor-signing key,
  with readers bounded by the network rather than by a credential. Prompt 19 establishes that
  deployment and states its actual exposure; the documentation records what the deployment does,
  not what this stage presumed.

These consequences are accepted for the physically controlled home-lab development network without
additional mitigation. Record them; do not add a compensating control that was not asked for.

## Leave the Keycloak path exactly as Prompt 17 left it

Keycloak remains a functioning option, not a deprecated one. Preserve unchanged:

- issuer `https://auth.tqaentry.com/realms/brightflag-mcp`;
- audience `brightflag-mcp`;
- signature, expiry, not-before, clock-skew, tenant, role, and scope validation;
- the flat top-level `tid` and `roles` claims the server requires; and
- OAuth protected-resource metadata and its `WWW-Authenticate` challenge, served in this mode only.

Grant the designated Keycloak development user both `brightflag.read` and `brightflag.payment`.
Refuse a token whose audience lacks `brightflag-mcp`, and refuse a token whose flat `roles` claim
lacks the required BrightFlag role for the capability being exercised.

Preserve Prompt 17's dedicated realm and pre-registered public-client contract. Do not move this
stage to the shared `homelab` realm or `mcp-client`; Prompt 19 owns that later deployment change and
the shared-client audience limitation that follows from it.

## What this stage deliberately does not add

- **No environment or profile restriction on fixed-token mode.** A deployment marked `Production`
  may select it. Selecting the mode is the operator's explicit decision, and the server does not
  second-guess it. This does not relax Prompt 8's existing rule that the `Local` JWKS trust provider
  fails startup under a profile not explicitly marked non-production; that rule stands unchanged and
  applies to a different thing.
- **No automatic hardening and no production-suitability check.** Do not synthesise an expiry, add a
  revocation list, escalate a warning into a refusal, restrict the surface, or disable payment
  because fixed-token mode is selected. Both supported modes permit the complete BrightFlag MCP
  surface, including payment planning and execution.
- **No Keycloak realm or caller-client migration.** Keep Prompt 17's dedicated realm and public
  client intact. Prompt 19 alone moves the deployment to the shared `homelab` realm and
  `mcp-client`.

## Prove

- startup selects exactly one authentication branch, and an absent, unknown, ambiguous, or
  incomplete mode fails startup with a typed error naming which of the four defects occurred and
  which value was missing, without echoing a secret;
- the fixed-token branch constructs the configured static caller identity — subject, tenant, and
  both roles — from configuration alone, and no field of it varies with the presented credential;
- a Keycloak-issued JWT presented in fixed-token mode is refused and never reaches
  `CallerTokenValidator`, proven against the constructed composition root rather than by inspecting
  the rejection's shape;
- an absent, empty, whitespace-only, or unreadable token file fails startup before the transport
  accepts a request; comparison is constant time; and the token appears in no log, audit record,
  error, health payload, diagnostic output, or exception at any level;
- the fixed identity can list tools, read the ontology resource, list invoices approved for payment,
  plan a payment, and execute that plan, and every one of those decisions is recorded in the audit
  log against the single configured subject;
- in fixed-token mode the 401 challenge is a bare `Bearer` challenge with no `resource_metadata`
  parameter and no scope, and the protected-resource metadata endpoint is not served;
- in Keycloak mode, issuer, audience, signature, expiry, not-before, tenant, role, and scope
  validation behave exactly as Prompt 17 left them, protected-resource metadata and its challenge
  are served, and the designated development user's token grants both read and payment;
- a Keycloak token whose audience lacks `brightflag-mcp` is refused, and so is a token that carries
  `brightflag-mcp` in its audience but lacks the required flat `roles` value;
- neither mode falls back to the other: an unreachable Keycloak, a failed JWKS retrieval, and a
  rejected JWT each refuse the request without attempting the fixed token, and a valid fixed token
  presented in Keycloak mode is refused; and
- selecting fixed-token mode under a `Production` profile starts successfully and grants the
  complete surface, while Prompt 8's refusal of the `Local` JWKS trust provider under that same
  profile still holds.

## Acceptance criteria

- Caller authentication is exactly one explicitly selected mode, with four separately named startup
  failures and no fallback path in either direction.
- Fixed-token mode authenticates one shared configured identity holding both capabilities, and its
  consequences for audit attribution, rate limiting, plan scoping, expiry, revocation, and the
  exposure of the store the token is read from are documented rather than mitigated.
- The dedicated Prompt 17 Keycloak realm, public-client contract, claim requirements, and
  protected-resource discovery are unchanged; the shared realm and caller client remain Stage 19's
  work.
- No environment, profile, or hardening check restricts an operator's explicit selection of
  fixed-token mode.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the exclusive two-mode authentication choice,
the shared static home-lab identity and the audit, rate-limit, plan-scope, expiry, and revocation
consequences it accepts, the deliberate absence of a production-profile guard, and the decision that
Prompt 17's dedicated Keycloak realm and public-client contract remain unchanged until Stage 19. Do
not push unless requested.
