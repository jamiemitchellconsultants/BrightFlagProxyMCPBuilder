# Stage 20 — Shared home-lab audience and a completable OAuth flow

Source: `BrightFlagProxyMCPBuilder/prompts/20-keycloak-shared-audience-mcp-oauth.md`

## Context

Stages 17 to 19 built a home-lab deployment for callers that could be handed a credential. Stage 18
added a fixed opaque token read from a mounted file. Stage 19 moved deployment ownership to LocalAI,
served the canonical endpoint over plaintext HTTP on the LAN, and described the Keycloak path as
supplying "a token the MCP client sends as a fixed `Authorization: Bearer` header".

The users turned out to be tier-1 conversational agents — Claude, ChatGPT, Copilot, Gemini, and the
mainstream open-source desktop and web agents. None of them can be handed a token, and none can be
pre-registered: each invents its own OAuth client the first time a user connects it, reads the
server's `401` to find the authorization server, and runs authorization-code + PKCE itself.

Stage 20 is what that costs. The audience becomes one value shared by every MCP server in the lab,
the endpoint moves to public HTTPS behind the shared Caddy, the metadata document has to be served
where clients actually look for it, and the advertised scope becomes a claim about the authorization
server rather than an assumption. The fixed-token mode survives untouched — the agents are a new
population of callers, not a replacement for the old one.

The realm side is already built and reviewed in LocalAI ([LocalAI#29][localai-29]). This stage does
not configure Keycloak; it stops assuming things about it that are no longer true.

## Preconditions

- Stages 1–11 and 17–19 are merged and green. Stage 18's exclusive two-mode selection and Stage 19's
  LocalAI-owned deployment are both in place.
- LocalAI's `setup-mcp-host-windows.ps1` has created the `homelab` realm, the `homelab-mcp` audience
  client, the `mcp-audience` realm-default client scope minting constant `tid` and `roles`, the
  capability client scopes, the `mcp-client` fallback caller, and the dynamic-registration Trusted
  Hosts policy.
- LocalAI's `setup-brightflag-mcp-windows.ps1` writes the trusted-proxy posture, the `homelab-mcp`
  audience and the HTTPS resource URI, and its Caddy fragment is a bare hostname with automatic
  HTTPS.
- A public DNS A record for `brightflag-mcp.tqaentry.com` resolves to the network's public address,
  and the router forwards inbound TCP 80 **and** 443 to the host. Port 80 is required for Let's
  Encrypt issuance and renewal, not only for redirects — the name had never been issued a
  certificate before LocalAI#29.
- Issues #45 and #46 are **closed**; #47 is transferred to LocalAI. Open follow-ups are
  [LocalAI#30][localai-30], [LocalAI#31][localai-31], and [LocalAI#32][localai-32].

## Scope in

The shared `homelab-mcp` audience; documenting what a shared audience means for `aud` as evidence;
serving the protected-resource metadata at both the bare and path-suffixed well-known paths in
Keycloak mode; making the advertised read scope optional in configuration, metadata and challenge;
selecting and naming the trusted-proxy transport posture with an HTTPS resource URI; updating the
documented deployment contract and every test that asserted the plaintext endpoint, the
private-source matcher, or the `host-gateway` entry; recording public reachability in place of
plaintext-on-LAN as the accepted exposure; and confirming no code treats a client identifier as
evidence.

## Scope explicitly out

Editing LocalAI from this repository, or copying its script here; configuring Keycloak; becoming an
authorization server, or adding a token, registration, introspection or session facility; removing
or deprecating Stage 18's fixed-token mode; removing the plaintext transport mode from the posture
enumeration; relaxing any claim, tenant, role, expiry, issuer or signature check; widening the MCP
surface; introducing a BrightFlag production URL or credential; contingent Stages 12 or 13.

## Work items

### 1. Move to the shared audience

Change the configured audience value to `homelab-mcp` and confirm this needs no code change — the
audience is one configured value, never inferred. If anything assumes the audience is this server's
own client identifier, fix that; it is the defect this item exists to catch.

Document the consequence where the audience is documented: `aud` no longer distinguishes this server
from any other MCP server in the lab, so it is not evidence about which server a caller intended to
reach. Stage 19 already recorded that the audience mapper is not a service-separation boundary and
that the required roles are; keep that statement and repoint it.

### 2. Serve both metadata paths

The document is served at the bare `/.well-known/oauth-protected-resource`. RFC 9728 locates it at
`/.well-known/oauth-protected-resource/mcp` for a resource URI ending `/mcp`. Serve both, from one
document, both unauthenticated, Keycloak mode only. Stage 18's rule that fixed-token mode serves
neither is unchanged.

### 3. Make the advertised scope optional

The validator requires the read scope to be one non-blank token. Make it optional: when configured,
advertise and challenge with it as now; when absent, omit `scopes_supported` from the metadata, omit
the `scope` parameter from the challenge, and pass validation.

The reason is not tidiness. A compliant client copies the advertised scope into its authorization
request, and Keycloak refuses a request naming a scope the client does not have — `invalid_scope`,
before a login page appears. The realm is not this repository's to configure, so the scope's
existence cannot be assumed. Name it in the deployment documentation as an external prerequisite.

### 4. Select the trusted-proxy posture

Caddy now holds a Let's Encrypt certificate for the name and the container listens on plain HTTP
only on the internal Docker network, so the trusted-proxy posture is the true one. Select it, name
the proxy, and set the canonical endpoint and protected-resource URI to
`https://brightflag-mcp.tqaentry.com/mcp`.

`TrustedProxyName` is a deployment declaration, not a header the server reads. Introduce no
forwarded-protocol or forwarded-address header handling with it.

Keep the plaintext mode implemented, tested, explicitly selectable, refused as a default or
fallback, and unrestricted by any profile guard. Only the selection changes.

### 5. Update the documented deployment contract and its tests

Stage 19 had the server document and assert the deployment's shape. Three of those shapes are now
wrong:

- the Caddy site address is a bare hostname with automatic HTTPS, not an explicit `http://`;
- there is no private-source matcher; and
- there is no `extra_hosts: auth.tqaentry.com:host-gateway` entry — the ingress Caddy carries a
  network alias for that name, so the issuer resolves over the internal network.

Update the assertions rather than deleting them. A test that asserted `http://` should assert the
HTTPS site; a test that asserted the matcher's presence should assert its absence, so the removal is
deliberate and provable rather than merely untested.

The option **names** are unchanged and remain a cross-repository contract; the values are not. Update
the documented contract to the values LocalAI now generates.

### 6. Record the exposure that replaced the old one

Stage 19's accepted opening was plaintext bearer credentials on the LAN. That is gone. What replaces
it: the endpoint is reachable from the public Internet by anyone who resolves the name, with only
this server's own token validation in front of it. Caddy authenticates nobody.

State it once, plainly, wherever a reader meets the endpoint. Stage 19's other recorded openings —
omitted container hardening, unauthenticated LAN-reachable LocalStack, resolved-commit-is-not-
immutability — are unchanged and stay stated.

Remove the private-source matcher's "unproven, tracked by #45" framing: #45 is closed because the
LAN-only policy was reversed rather than verified, and a reader must not be left looking for an open
follow-up that no longer exists.

### 7. Confirm client identifiers are not treated as evidence

The client id in an accepted token now belongs to a client that registered itself minutes ago and
that no reviewed configuration has seen. Audit for any use of `azp`, `appid` or a client id as
evidence beyond token validity, and record the finding where caller identity is documented.

## Tests

Map one-to-one to the prompt's Prove list:

1. request both well-known paths in Keycloak mode and assert one identical unauthenticated document
   naming the configured resource and issuer; assert neither path is served in fixed-token mode;
2. with a read scope configured, assert `scopes_supported` and the challenge's `scope` parameter;
   with none, assert both absent and configuration validation passing;
3. assert the Keycloak-mode 401 challenge names a metadata URL derived from the configured resource
   URI, and that the fixed-token-mode challenge is a bare `Bearer` with no `resource_metadata` and
   no scope;
4. accept a token carrying `homelab-mcp`; refuse one carrying only `brightflag-mcp`;
5. re-run the Stage 17 and 18 rejection suites unchanged — required claims, tenant, expiry,
   not-before, issuer, signature, algorithm — and assert a token whose flat `roles` lacks the role
   for the capability being exercised is still refused;
6. select the trusted-proxy posture and assert an HTTPS resource URI is required and accepted;
   separately assert the plaintext mode still selects, still accepts an `http://` resource URI,
   still fails as an unset default, and is still not refused under any profile;
7. assert the documented deployment contract and its fixtures describe the HTTPS Caddy site, no
   private-source matcher, and no `host-gateway` entry;
8. grep for `azp`, `appid` and client-id usage in authorization decisions and assert none; and
9. exercise the complete surface under both authentication modes and assert neither falls back to
   the other.

Tests 1–6, 8 and 9 are automatable in this repository. Test 7 asserts against LocalAI's generated
output: automate what can be asserted against a checked-in expected shape, and label the rest a
manual gate.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

```bash
rg -n "brightflag-mcp\.tqaentry\.com" docs/ README.md | rg -v "https://"
```

The third command must find nothing: no document may still present the endpoint as `http://`.

Manual gates on `ai-mcp-server`, each recorded with its exact command and dated result: Let's
Encrypt issues a certificate for `brightflag-mcp.tqaentry.com` and `/health` answers over HTTPS from
outside the LAN; both well-known paths return the document; a conversational agent given only the
MCP URL registers itself, signs the user in, and calls a tool; a device-flow token still grants read
and payment; fixed-token mode still grants read and payment and serves no discovery; discovery and
JWKS succeed from inside the container without a `host-gateway` entry; and Streamable HTTP responses
are still not buffered.

End-to-end verification of the token contract against the deployed realm is [LocalAI#31][localai-31]
and is not an acceptance gate here.

## Stage boundary

Commit locally. Suggested message: `Adopt the shared home-lab audience and the MCP OAuth flow`.

Use `narrative-required` when published. Do not push unless requested. Do not edit LocalAI from this
stage, configure Keycloak, deprecate the fixed-token mode, remove the plaintext transport mode,
soften any validation, or begin contingent Stages 12 or 13.

## Risks

- Removing the private-source matcher and enabling automatic HTTPS makes the endpoint genuinely
  Internet-reachable. Stage 19 treated that as a risk to avoid; this stage accepts it deliberately
  and relies entirely on token validation. Any later change that weakens validation is now a
  materially different decision from what it would have been before.
- Making the advertised scope optional is one refactor away from silently advertising nothing.
  Test both configurations, not just the empty one.
- The plaintext transport mode survives with no deployment selecting it, which is how a mode rots
  untested. Keep its tests, and keep the unset-posture failure alongside them.
- Two well-known paths serving one document invites them drifting apart if one is later rewritten.
  Assert equality of the two responses, not merely that each is well-formed.
- One audience for the whole lab means a compromised or careless MCP server elsewhere in it accepts
  the same tokens this one does. The boundary is the realm, not the service. Adding an MCP server to
  the lab is now a decision about this server's exposure too.
- Stage 19's cross-repository option contract now has changed values on both sides. A partial
  rollout — one repository updated and not the other — produces a container that fails startup on
  an HTTPS resource-URI check or refuses every token on audience. Roll them together.

[localai-29]: https://github.com/jamiemitchellconsultants/LocalAI/pull/29
[localai-30]: https://github.com/jamiemitchellconsultants/LocalAI/issues/30
[localai-31]: https://github.com/jamiemitchellconsultants/LocalAI/issues/31
[localai-32]: https://github.com/jamiemitchellconsultants/LocalAI/issues/32
