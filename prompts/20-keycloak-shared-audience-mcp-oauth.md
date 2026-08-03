# Prompt 20 — One shared home-lab audience, and an OAuth flow a conversational agent can complete

Using the reusable contract and the artifacts produced by Prompts 1–11 and 17–19, make
`BrightFlagProxyMCPServer` reachable from a tier-1 conversational agent authenticating against the
shared home-lab Keycloak realm.

Apply this stage after Prompt 19. It does not depend on or authorise contingent Prompts 12 or 13.

This stage changes caller-identity configuration, OAuth discovery, and the transport posture of the
home-lab deployment. It must not widen the MCP surface by a single tool, resource, or argument, must
not touch the BrightFlag service-credential contract, and must not weaken caller authorization.

## Why this stage exists

Prompts 17 to 19 were written for callers that could be handed a credential: a fixed token in a
configuration file, or a device-flow access token pasted into one. Prompt 19 says so explicitly —
the Keycloak path supplies "a token the MCP client sends as a fixed `Authorization: Bearer` header".

The home lab's users do not work that way. They reach this server from Claude, ChatGPT, Copilot,
Gemini, and the mainstream open-source desktop and web agents. Such an agent cannot be handed a
token and cannot be pre-registered: it invents its own OAuth client the first time a user connects
it, discovers where to send them from the server's own `401`, and runs the flow itself. Everything
below follows from that one fact.

The deployment side is already built and reviewed in the separately managed LocalAI repository
([LocalAI#29][localai-29]), which now owns the realm, the shared audience, the claim mappers and the
dynamic-registration policy. This stage brings the server into line with it.

## What this stage supersedes

Prompts 17 to 19 remain historical and are not rewritten. When the sequence is played in order, this
stage replaces the following **implemented** decisions and nothing else. Every other requirement of
those stages survives; do not read this table as licence to relax an unlisted control.

| Prompt 18 or 19 | Prompt 20 |
|---|---|
| Audience and resource client `brightflag-mcp` | Audience `homelab-mcp`, shared by every MCP server in the lab. No BrightFlag resource client exists |
| Refuse a token whose audience lacks `brightflag-mcp` | Refuse a token whose audience lacks `homelab-mcp` |
| `tid` mapper and a mapper projecting `brightflag-mcp` client roles into flat `roles` | Constant `tid` and `roles` claims from a realm-default client scope. No BrightFlag client roles exist |
| A designated development user holding both `brightflag.read` and `brightflag.payment` | No designated user and no per-user assignment. Every user receives the same claims |
| Device authorization through the public caller client, its token sent as a fixed header | The agent runs authorization-code + PKCE with a client it registered itself. Device flow is retained as the fallback for tooling that runs no OAuth flow |
| Canonical endpoint `http://brightflag-mcp.tqaentry.com/mcp` | `https://brightflag-mcp.tqaentry.com/mcp` |
| Deployment selects the explicit plaintext transport mode | Deployment selects the trusted-proxy posture and names Caddy. The plaintext mode stays implemented and selectable; this deployment stops selecting it |
| "Do not describe this deployment as an HTTPS trusted-proxy deployment" | It is one. Caddy holds a Let's Encrypt certificate for the name and the container speaks plain HTTP only on the internal Docker network |
| Plaintext `http://` external resource URI | The protected-resource URI is HTTPS, as the validator has always required outside plaintext mode |
| Caddy fragment with an explicit `http://` site address | A bare hostname, so Caddy enables automatic HTTPS |
| Caddy fragment carries a private-source matcher, documented as unproven and tracked by issue #45 | The matcher is gone. Issue #45 is closed: the LAN-only policy was reversed, not verified |
| Container carries `extra_hosts: auth.tqaentry.com:host-gateway` | No such entry. The ingress Caddy carries a network alias for that name, so the issuer resolves over the internal network |
| Bearer credentials cross the LAN in plaintext | They cross TLS to Caddy. What replaces that exposure is public reachability — see below |
| Client separation tracked by issue #46 | Closed and settled: one shared audience, deliberately. Verification is [LocalAI#31][localai-31] |

Prompt 18's fixed-token mode is **retained unchanged and still selectable**. Nothing in this stage
deprecates it, and its bare `Bearer` challenge with no `resource_metadata` and no protected-resource
metadata endpoint remains exactly right for the clients it exists for. Every requirement below that
concerns discovery applies to Keycloak mode only, and the exclusive two-mode selection with no
fallback in either direction is untouched.

## One audience for the whole home lab

The realm issues tokens whose audience is `homelab-mcp`, shared by every MCP server in the lab,
present and future. A token minted for one is accepted by all of them.

That is not production-safe and is not defended as safe. It is a reviewed home-lab decision, taken
because a dedicated audience per server makes deploying each new MCP server a task with
identity-provider configuration in it, on a network with one operator and no adversary between its
services. Production authenticates against the corporate Entra instance and shares none of this.

Requirements:

- The audience remains one configured value, never inferred from a token. This should need no code
  change; verify that. If the audience is anywhere assumed to be this server's own client
  identifier, that assumption is a defect this stage fixes.
- Document the consequence where the audience is documented: `aud` no longer distinguishes this
  server from any other in the lab, so it is not evidence about which server a caller intended to
  reach, and nothing may begin treating it as such.
- Prompt 19 already stated that the shared caller client's audience mapper is not a
  service-separation boundary and that the required roles are. That is now more true, not less.
  Keep the statement and update what it points at.

## Constant claims, and no relaxation of authorization

The lab grants every authenticated user the same capabilities. The realm mints constant `tid` and
`roles` claims for every user, so the required-claims check, the tenant-boundary corroboration and
the role-to-capability configuration from Prompts 8, 17 and 18 all keep working unchanged and
unweakened. The claims are present; they are simply the same for everyone.

Do not add a "no authorization" mode, do not make the required claims optional, and do not
special-case the home lab in the authorization path. Production runs that same path. State this in
the documentation so a later reader does not mistake constant claims for disabled authorization.

Prompt 19's requirement that payment is granted by a named role value —
`BrightFlag__Authorization__MarkInvoicePaidRoles__0` alongside the read role — is unchanged, and
both capabilities remain available in both authentication modes.

## Serve the metadata where clients actually look for it

Prompt 17 established the protected-resource metadata document and its `WWW-Authenticate` challenge,
and Prompt 18 confined them to Keycloak mode. Both decisions stand. Two things about them change.

**Serve both metadata paths.** The document is served at the bare
`/.well-known/oauth-protected-resource`. RFC 9728 locates the metadata for a resource with a path at
the path-suffixed form — for a resource URI ending `/mcp`, that is
`/.well-known/oauth-protected-resource/mcp`. Clients are split on which they request, and one that
asks for the form this server does not serve receives a 404 and abandons the flow. That presents to
the user as an agent silently refusing to connect, with nothing useful in any log. Serve both, from
one document, both unauthenticated, in Keycloak mode only.

**Do not advertise a scope the authorization server will reject.** The challenge and the metadata
name a read scope, and the configuration validator requires it to be one non-blank token. A
compliant client copies that scope into its authorization request, and Keycloak refuses an
authorization request naming a scope that is not among the client's assigned client scopes — with
`invalid_scope`, before the user ever sees a login page.

The realm is not this repository's to configure, so this stage must not assume its contents. Make
the advertised scope optional:

- when configured, advertise and challenge with it exactly as now;
- when absent, serve metadata with no `scopes_supported`, challenge with no `scope` parameter, and
  let configuration validation accept that as a complete configuration rather than a missing one.

Name the realm-side requirement — an advertised scope must exist as a client scope on the realm —
in the deployment documentation as an external prerequisite, in the way Prompt 17 names the
infrastructure it consumes without owning. The LocalAI deployment creates the capability names as
realm-default client scopes for exactly this reason.

## The transport posture, corrected

Prompt 19 had this server serve a plaintext endpoint and forbade describing it as a trusted-proxy
deployment, because at that time nothing terminated TLS in front of it. That has changed: Caddy
holds a Let's Encrypt certificate for `brightflag-mcp.tqaentry.com`, and the container listens on
plain HTTP reachable only over the internal Docker network.

- The deployment selects the trusted-proxy posture and names the proxy. `TrustedProxyName` is a
  declaration about the deployment, not a header the server reads — no caller can nominate itself as
  the proxy, and this stage must not introduce any forwarded-protocol or forwarded-address header
  handling.
- The canonical endpoint and the protected-resource URI are `https://brightflag-mcp.tqaentry.com/mcp`.
- The explicit plaintext transport mode added by Prompt 19 **remains implemented, tested, and
  selectable**. It is honest about what it is and another host may want it. What changes is that
  this deployment no longer selects it, and every document and test asserting that it does must be
  updated to assert the trusted-proxy posture instead.
- Host validation still permits exactly three names, unchanged.

## What the openness now is

Prompt 19 recorded plaintext bearer credentials on the LAN as its accepted exposure. That exposure
is gone and a different one replaces it, which must be recorded just as plainly, in the same places:

**The MCP endpoint is reachable from the public Internet by anyone who resolves the name.** The only
thing in front of it is this server's own token validation. Caddy authenticates nobody — that is
unchanged and is corporate policy, mirrored here so the lab and production behave the same way.

This is why nothing in the validation path may be softened. It is also why the private-source
matcher is removed rather than kept: it was never demonstrated to distinguish a LAN client from an
Internet one through Windows, Docker Desktop's networking and router forwarding — issue #45 recorded
exactly that uncertainty — and an unproven control that also blocks every intended caller is worse
than no control at all.

Prompt 19's other recorded openings are unchanged and must remain stated: the omitted container
hardening and resource limits, the unauthenticated LAN-reachable LocalStack holding the deployment's
secrets, and the fact that a resolved commit makes a build reproducible without making a branch
immutable.

## Dynamic client registration

There is nothing to implement. This server is a resource server and must not become an authorization
server, gain a token endpoint, a registration endpoint, a session store, or a token-introspection
call. Keycloak is the authorization server and the client talks to it directly.

It is stated because it constrains what may be assumed. The client identifier in an accepted token
now belongs to a client that registered itself minutes ago and that no reviewed configuration has
ever seen. Any code that treats `azp`, `appid`, or a client id as evidence of anything beyond "this
token validated" is wrong under this deployment. Verify that nothing does, and say so where caller
identity is documented.

## The cross-repository configuration contract

Prompt 19 made the server's option names a contract with the LocalAI deployment script, and that
rule stands: a rename on either side is a breaking change to the other repository and must be made
in both. The **names** below are unchanged; the **values** the deployment supplies are not, and the
documentation of the contract must be updated to match what LocalAI now generates:

| Setting | Was | Is |
|---|---|---|
| `BrightFlag__CallerIdentity__Audience` | `brightflag-mcp` | `homelab-mcp` |
| `BrightFlag__CallerIdentity__ProtectedResource__ResourceUri` | `http://brightflag-mcp.tqaentry.com/mcp` | `https://brightflag-mcp.tqaentry.com/mcp` |
| `BrightFlag__CallerIdentity__ProtectedResource__ReadScope` | required | optional |
| `BrightFlag__Hosting__HttpTransportSecurity` | the plaintext value | the trusted-proxy value |
| `BrightFlag__Hosting__TrustedProxyName` | unset | names the Caddy ingress |

`BrightFlag__CallerIdentity__Mode`, the `__FixedToken__*` settings, `__AllowedHosts__0..2`,
`__AllowedInvoiceStatuses__0`, and both authorization role settings are unchanged.

Do not edit the LocalAI repository from this stage, and do not copy its script here.

## Prove

- both metadata paths return the same document, unauthenticated, in Keycloak mode, and neither is
  served in fixed-token mode;
- with a read scope configured, the metadata advertises it and the challenge carries it; with none
  configured, the metadata omits `scopes_supported`, the challenge carries no `scope` parameter, and
  configuration validation passes;
- an unauthenticated MCP request in Keycloak mode is refused with a challenge naming a metadata URL
  derived from the configured resource URI, and in fixed-token mode with a bare `Bearer` challenge
  carrying no `resource_metadata` and no scope, exactly as Prompt 18 requires;
- a token carrying `homelab-mcp` in its audience validates, and one carrying only the former
  `brightflag-mcp` audience is refused;
- required-claim, tenant, expiry, not-before, issuer, signature, algorithm and role rejections all
  behave exactly as Prompts 17 and 18 left them, and a token whose flat `roles` claim lacks the role
  required for the capability being exercised is still refused;
- the trusted-proxy posture is selected, an HTTPS resource URI is required and accepted, and the
  plaintext mode is still implemented, still explicitly selectable, still refused as a default or
  fallback, and still not rejected by any deployment-profile guard;
- the documented deployment contract describes the HTTPS Caddy site with no private-source matcher
  and no `auth.tqaentry.com:host-gateway` entry, and every test or document asserting the former
  shapes has been updated rather than deleted wholesale;
- no code treats a client identifier as evidence beyond token validity;
- both capabilities remain reachable in both authentication modes, and neither mode falls back to
  the other; and
- if any existing test needed editing to accommodate this stage beyond the supersession table above,
  that is a signal the change was not additive — reconsider it rather than adjusting the test.

## Acceptance criteria

- A conversational agent given only `https://brightflag-mcp.tqaentry.com/mcp` completes the flow end
  to end: refused, discovers the authorization server, registers itself, sends the user to sign in,
  and calls a tool successfully.
- Prompt 18's fixed-token mode still works, still serves no discovery, and is still selected only
  explicitly.
- No authorization behaviour is weakened and no new authentication mode is introduced.
- The MCP surface and the BrightFlag service-credential contract are unchanged, and BrightFlag
  remains integration-test only.
- Documentation states the shared audience, the public reachability, the constant claims, and the
  realm-side prerequisite for any advertised scope — each as a decision with a consequence, not as a
  feature — and no longer presents the plaintext endpoint or the private-source matcher as current.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record why Prompt 18's fixed token is retained while
Prompt 19's plaintext endpoint is not, why one audience is shared across the lab and what that
costs, why an advertised scope is a claim about the authorization server's configuration, and that
the endpoint is now publicly reachable with only token validation in front of it. Do not push unless
requested.

[localai-29]: https://github.com/jamiemitchellconsultants/LocalAI/pull/29
[localai-31]: https://github.com/jamiemitchellconsultants/LocalAI/issues/31
