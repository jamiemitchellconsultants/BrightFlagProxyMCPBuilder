# Stage 18 — Home-lab fixed-token authentication

Source: `BrightFlagProxyMCPBuilder/prompts/18-home-lab-fixed-token-authentication.md`

## Context

Stage 17 made Keycloak the live caller-token issuer for the shared home-lab infrastructure. The MCP
clients this deployment actually serves send a fixed `Authorization` header from a configuration
file and run no OAuth flow, so a Keycloak-only server is unusable by them without a token-refresh
helper. Stage 18 adds a fixed opaque bearer token as an explicitly selected second mode, keeps
Keycloak working beside it, and makes the two mutually exclusive so no request can be authenticated
by whichever mode happens to succeed.

The shortcut is real and is taken deliberately. The plan's job is to make its consequences visible
and testable, not to soften them.

## Preconditions

- Stages 1–11 are merged and green, and Stage 17 is merged and its result is the server baseline.
- Prompt 17's dedicated Keycloak realm `brightflag-mcp`, its pre-registered public client, resource
  audience, and flat `tid` and `roles` claims are configured and working; the development user holds
  the read role, and this stage grants the payment role without changing that realm or client.
- The reviewed home-lab tenant value, subject, and role names for the fixed identity exist as
  configuration, not as literals in code.
- The deployment path that mounts the fixed token file is Stage 19's; this stage needs only the
  existing `File` secret source.

## Scope in

An explicit two-value authentication-mode selection with four named startup failures; a fixed-token
caller-identity provider reading a mounted file with constant-time comparison and complete
redaction; a static configured caller identity holding both capabilities; a bare `Bearer` challenge
and suppressed protected-resource metadata in that mode; unchanged Keycloak validation, claim
requirements, and discovery; documentation of the accepted consequences.

## Scope explicitly out

Removing, deprecating, or revising the Keycloak path; any fallback between modes; a production or
environment guard on fixed-token mode; synthesised expiry, revocation, or surface restriction for
the fixed token; changing the BrightFlag service credential or cursor-signing contract; dedicated
Keycloak client separation; the `ai-mcp-server` deployment, which is Stage 19; contingent Stages 12
or 13.

## Work items

### 1. Make the mode an explicit, exclusive startup selection

Add the mode to the caller-identity options and validate it before the host is built. Refuse absent,
unknown, ambiguous, and incomplete configuration as four distinct typed failures naming the selected
mode and the missing value, with no secret in the message. Construct one authentication branch in
the composition root and leave the other unconstructed, so exclusivity is structural rather than a
per-request condition.

### 2. Implement the fixed-token provider

Read the token through the existing `File` secret source; fail startup on absent, unreadable, empty,
or whitespace-only content. Reject a malformed or absent `Authorization` header before comparison,
compare in constant time, and route every rejection through one indistinguishable outcome. Add the
token to the redaction boundary that already covers the BrightFlag service credential and the
cursor-signing key.

### 3. Build the static caller identity

Construct subject, tenant, and both capability roles from reviewed configuration alone. Grant read
and payment. Assert in construction that no field is populated from the presented credential.

### 4. Split the unauthenticated challenge by mode

Serve the existing OAuth protected-resource metadata and its `resource_metadata` challenge in
Keycloak mode only. In fixed-token mode answer with a bare `WWW-Authenticate: Bearer` challenge and
do not route the metadata endpoint at all.

### 5. Preserve the Prompt 17 Keycloak contract

Leave the dedicated realm, pre-registered public client, issuer, audience, signature, lifetime,
tenant, role, scope, metadata, and challenge contract untouched. Grant the designated development
user the payment role, but leave the move to the shared `homelab` realm and `mcp-client` to
Stage 19.

### 6. Document the accepted consequences

Record in `docs/` the single shared identity's effect on audit attribution, rate-limit bucketing,
and caller-bound plan scope; the absence of expiry, revocation, and tenant-claim corroboration on
the fixed path; and the exposure of the store the token and cursor key are read from. State them as
accepted for this network, with no compensating control implied.

## Tests

Map one-to-one to the prompt's Prove list:

1. assert one authentication branch is constructed after selection, and drive absent, unknown,
   ambiguous, and incomplete configuration into four distinct typed startup failures whose messages
   name the mode and missing value and contain no secret;
2. construct the fixed-token branch and assert subject, tenant, and both roles equal the configured
   values, including under varied presented credentials that must not change any field;
3. present a Keycloak-issued JWT in fixed-token mode against the real composition root and assert
   refusal plus zero `CallerTokenValidator` invocations, using an instrumented or absent validator
   rather than the rejection's shape;
4. drive absent, empty, whitespace-only, and unreadable token files into startup failure before the
   transport listens; assert constant-time comparison; and scan log, audit, error, health,
   diagnostic, and exception output at every level for the token value;
5. as the fixed identity, list tools, read the ontology resource, list approved invoices, plan a
   payment, and execute that plan, asserting each authorization decision is audited against the one
   configured subject;
6. assert the fixed-token 401 carries a bare `Bearer` challenge with no `resource_metadata` and no
   scope, and that the protected-resource metadata route is absent in that mode;
7. re-run the Stage 17 Keycloak validation suite unchanged — issuer, audience, signature, expiry,
   not-before, tenant, role, scope, metadata, and challenge — and assert the development user's
   token grants read and payment;
8. refuse a token whose audience lacks `brightflag-mcp`, and refuse a token carrying that audience
   but lacking the required flat `roles` value;
9. assert no fallback in either direction: unreachable Keycloak, failed JWKS retrieval, and rejected
   JWT each refuse without a fixed-token attempt, and a valid fixed token in Keycloak mode is
   refused; and
10. start under a `Production` profile with fixed-token mode selected, assert success and the
    complete surface, and assert Stage 8's refusal of the `Local` JWKS trust provider under that
    profile still fails startup.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

```bash
rg -n "MCP_FIXED_TOKEN|FixedToken__Value|fixed-token" --glob '!test/**' src/ docs/
```

The third command is a review aid, not a pass/fail gate: it exists so a reviewer can see every place
the fixed token is named outside tests and confirm no literal value was committed. Any check that
cannot be executed on the authoring host is a labelled manual gate with its exact command and
expected result.

## Stage boundary

Commit locally. Suggested message: `Add exclusive fixed-token home-lab authentication`.

Use `narrative-required` when published. Do not push unless requested. Do not begin Stage 19's
deployment change, retire any `deploy/local` artifact, move to the shared `homelab` realm or
`mcp-client`, or start contingent Stages 12 or 13.

## Risks

- A per-request mode check invites a later refactor to try both modes; construct one branch and
  leave the other unconstructed so a fallback cannot be reintroduced by an ordinary edit.
- One shared identity silently weakens the caller-bound plan guarantee that the payment path depends
  on. It remains a real limit of this mode, and the plan's answer is disclosure and a test that
  records the shared subject, not a compensating control.
- The token has no expiry or revocation, so rotation is a file replacement plus a restart. Treat a
  suspected disclosure as requiring rotation of both the token and the cursor-signing key.
- The practical strength of this mode is whatever protects the store the token is read from, which
  this stage presumes is the network rather than a credential; Stage 19 establishes the deployment
  that settles it. Do not let the constant-time comparison read as though it were the boundary.
- Suppressing protected-resource metadata in fixed-token mode changes what an OAuth-capable client
  discovers. That is intended here; do not let it leak into Keycloak mode, where the metadata is
  still the contract.
