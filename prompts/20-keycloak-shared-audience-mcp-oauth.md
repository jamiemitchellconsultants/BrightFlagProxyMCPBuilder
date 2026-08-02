# Prompt 20 — One shared home-lab audience, and an OAuth flow a conversational agent can complete

Using the reusable contract and the artifacts produced by Prompts 1–11 and 17, make
`BrightFlagProxyMCPServer` usable from a tier-1 conversational agent authenticating against the
home-lab Keycloak realm. This stage changes caller-identity configuration and OAuth discovery only.
It must not widen the MCP surface by a single tool, resource, or argument, must not touch the
BrightFlag service credential contract, and must not weaken any existing runtime trust boundary.

This stage does not depend on or authorise contingent Prompts 12 or 13.

## What this supersedes

**Prompt 18 is superseded and must not be applied.** It adds a fixed opaque bearer token as a
second caller-authentication mode. It was never applied — the server has no fixed-token mode on any
branch — and it is now the wrong answer to the question it was written for. That question was "how
does a home-lab caller authenticate without an OAuth flow", and the answer this stage gives is that
the caller runs the OAuth flow, because the callers that matter are conversational agents that
insist on running it. A fixed shared token is also the one credential shape those agents cannot use.

If Prompt 18 has already been applied to an implementation, this stage does not require ripping it
out. Leave it in place, unselected, and record it as superseded. Do not extend it.

Prompt 17's Keycloak trust configuration stands and is not revised. This stage changes what the
audience is, adds a discovery path, and states a deployment posture — it does not rework the
validator.

## Why this stage exists

The home lab's users reach this server from Claude, ChatGPT, Copilot, Gemini, and the mainstream
open-source desktop and web agents. Such an agent cannot be pre-registered: it invents its own OAuth
client the first time a user connects it, then discovers where to send them from the server's own
401. Everything below follows from that one fact.

## Read before inspecting

Inspect the repository first and adapt every name below to its current structure:

- the caller-identity configuration record, its validator, and the trust-provider selection;
- the OAuth protected-resource metadata type, the route that serves it, and the bearer challenge
  built from it;
- the hosting options, particularly the transport-security posture and the allowed-host list;
- `appsettings.example.json`, the configuration documentation, and the deployment guide;
- the caller-authentication and configuration tests;
- the repository's agent instructions and Narrative rules.

## One audience for the whole home lab

The deployment configures a single audience — `homelab-mcp` — shared by every MCP server in the lab,
present and future. A token minted for one is accepted by all of them.

That is not production-safe and is not defended as safe. It is a reviewed home-lab decision taken
because the alternative, a dedicated audience per server, makes deploying the next MCP server a task
with identity-provider configuration in it, and the lab has one operator and no adversary between
its services. Production deployments authenticate against the corporate Entra instance and share
none of this.

Requirements:

- The audience remains a single configured value that is never inferred from a token. No code
  change should be needed for this at all — verify that, and if the audience is anywhere assumed to
  be this server's own client identifier, that assumption is the defect this stage fixes.
- Document the shared audience where the audience is documented, as the deployment's choice and its
  consequence: `aud` no longer distinguishes this server from any other in the lab, so it is not
  evidence about which server a caller intended to reach, and nothing may start treating it as such.

## No role differentiation, and no code that assumes otherwise

The lab grants every authenticated user the same capabilities. The realm mints constant `tid` and
`roles` claims for every user, so the existing required-claims check and the existing
role-to-capability configuration keep working unchanged and unweakened — the claims are present,
they are just the same for everyone.

Do not add a "no authorization" mode, do not make the required claims optional, and do not
special-case the home lab in the authorization path. The server's authorization model is sound and
production runs it; the lab simply supplies uniform inputs to it. State this in the documentation so
the next reader does not mistake constant claims for disabled authorization.

## Serve the metadata where clients actually look for it

The server publishes RFC 9728 protected-resource metadata and challenges unauthenticated callers
with a `WWW-Authenticate` header naming it. Two things about that need to change.

**Serve both metadata paths.** The document is currently served at the bare
`/.well-known/oauth-protected-resource`. RFC 9728 locates the metadata for a resource with a path at
the path-suffixed form — for a resource URI ending `/mcp`, that is
`/.well-known/oauth-protected-resource/mcp`. Clients are split on which they request, and one that
asks for the form this server does not serve gets a 404 and abandons the flow, which presents to the
user as the agent simply refusing to connect with no useful error anywhere. Serve both, from one
document. Both stay unauthenticated.

**Do not advertise a scope the authorization server will reject.** The challenge and the metadata
name a read scope. A compliant client copies that scope into its authorization request, and Keycloak
rejects an authorization request naming a scope that is not among the client's assigned client
scopes — with `invalid_scope`, before the user ever sees a login page. So either the advertised
scope exists as a client scope in the realm, or it must not be advertised.

The realm is not this repository's to configure, so this stage must not assume it. Make the
advertised scope explicitly optional: when it is configured, advertise and challenge with it exactly
as now; when it is absent, serve metadata with no `scopes_supported` and challenge without a `scope`
parameter, and let the validator accept that as a complete configuration rather than a missing one.
Name the realm-side requirement — that an advertised scope must exist as a client scope — in the
deployment documentation as an external prerequisite, in the same way Prompt 17 names the
infrastructure it consumes without owning.

## The transport posture is now a TLS-terminating proxy

The deployment that consumes this stage sits behind a Caddy reverse proxy holding a Let's Encrypt
certificate for the public hostname, and the MCP endpoint is reachable from the public Internet over
HTTPS. The server keeps listening on plain HTTP inside the container, reachable only over the
internal Docker network.

That is the existing `TrustedProxy` transport posture and needs no code change — it needs the
deployment to stop describing itself as a plaintext home-lab endpoint. The protected-resource URI is
consequently `https://<host>/mcp`, which the existing validator already requires, and the
documentation must stop presenting an `http://` MCP endpoint as a supported shape.

Say plainly in the deployment documentation what this costs: the endpoint is reachable by anyone on
the Internet who resolves the name, and the only thing in front of it is this server's own token
validation. That is the accepted posture, and it is the reason none of the validation above may be
softened.

## Dynamic client registration

Nothing to implement — this server is a resource server and must not become an authorization server,
gain a token endpoint, a registration endpoint, a session store, or a token-introspection call.

It is stated because it constrains what may be assumed: the client identifier in an accepted token
belongs to a client that registered itself minutes ago and that no reviewed configuration has ever
seen. Any code that treats `azp`, `appid`, or a client id as evidence of anything beyond "this token
validated" is wrong under this deployment. Check that nothing does, and say so in the documentation.

## Tests

- Metadata is served identically at the bare and path-suffixed well-known paths, unauthenticated.
- With a read scope configured, the metadata advertises it and the challenge carries it.
- With no read scope configured, the metadata omits `scopes_supported`, the challenge carries no
  `scope` parameter, and configuration validation passes.
- An unauthenticated MCP request is still refused with a challenge naming a metadata URL derived
  from the configured resource URI.
- Tokens carrying the shared audience validate; a token for another audience is still refused.
- Required-claim, tenant, expiry, issuer, signature and algorithm rejections all behave exactly as
  before. If any existing test needed editing to accommodate this stage, treat that as a signal the
  change was not additive and reconsider it rather than adjusting the test.

## Acceptance criteria

- A conversational agent given only the public MCP URL completes the flow end to end: refused,
  discovers the authorization server, registers itself, sends the user to sign in, and calls the
  server successfully.
- No authorization behaviour is weakened, and no new authentication mode is introduced.
- The MCP surface and the BrightFlag service credential contract are unchanged.
- Documentation states the shared audience, the public reachability, the constant claims, and the
  realm-side prerequisite for any advertised scope — each as a decision with a consequence, not as a
  feature.
- The repository's full check suite passes.

This is a meaningful security decision. If opening a pull request, apply `narrative-required` and
include substantive `## Narrative Context`, `## Narrative Decision`, and `## Narrative Consequences`
sections recording why Prompt 18's fixed token is superseded rather than completed, why one audience
is shared across the lab and what that costs, why an advertised scope is a claim about the
authorization server's configuration, and that the endpoint is now publicly reachable. Never
hand-edit generated `Narrative.md`.

Commit locally with a focused message. Do not push.
