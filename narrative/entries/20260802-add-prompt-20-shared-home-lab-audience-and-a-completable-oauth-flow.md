---
date: 2026-08-02
slug: add-prompt-20-shared-home-lab-audience-and-a-completable-oauth-flow
title: "Add prompt 20: shared home-lab audience and a completable OAuth flow"
summary: "Supersede Prompt 18 rather than complete it, and state that it must not be applied."
kind: product
status: accepted
sequence: 2026-08-02T12:39:37.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/52; merge commit 4ea4d82de3ef8ebc8f531f5bc3a73d213eebae21"
---

## Context

Prompt 18 adds a fixed opaque bearer token as a second caller-authentication mode. It was written to answer "how does a home-lab caller authenticate without an OAuth flow". It was never applied — the server has no fixed-token mode on any branch — and the question turned out to be the wrong one: the home lab's callers are Claude, ChatGPT, Copilot, Gemini and the mainstream open-source agents, which insist on running the flow themselves and for which a fixed shared token is the one credential shape they cannot use.

Reading the server against the deployment that consumes it (jamiemitchellconsultants/LocalAI#29) turned up two concrete defects that no amount of configuration fixes, plus one that was purely a description problem:

The protected-resource metadata is served only at the bare `/.well-known/oauth-protected-resource`. RFC 9728 locates the document for a resource with a path at the path-suffixed form. Clients are split on which they request.

The challenge and metadata name a read scope, and a compliant client copies that scope into its authorization request — where Keycloak rejects any scope not assigned to the client, with `invalid_scope`, before the user reaches a login page. The realm is not this repository's to configure, so the advertised scope cannot be assumed to exist.

The deployment described itself as a plaintext LAN endpoint under an explicit plaintext transport mode. The server has no such mode; `HttpTransportSecurity` is `None`/`DirectTls`/`TrustedProxy`, and the deployment now genuinely sits behind a TLS-terminating Caddy.

## Decision

Supersede Prompt 18 rather than complete it, and state that it must not be applied. Where it has already been applied, leave it in place unselected rather than unpicking it — removing an unused code path is churn with a risk attached and no benefit.

Serve the metadata document at both well-known paths. The alternative, following the RFC exactly and letting non-conforming clients fail, is defensible and costs the user an agent that will not connect with no diagnosable reason. Both paths are a few lines from one document.

Make the advertised scope optional rather than required, so a deployment whose realm has no matching client scope simply advertises none and challenges without one. The alternative — require the scope and document the realm-side prerequisite — puts a failure that surfaces as `invalid_scope` in a browser redirect behind a documentation step someone will miss.

Take one audience shared across the whole home lab, and say plainly that it is not production-safe. The alternative, an audience per server, makes deploying each new MCP server a task with identity-provider configuration in it. The lab has one operator and no adversary between its services; production authenticates against corporate Entra and shares none of this.

Constant `tid` and `roles` claims from the realm rather than any relaxation of the authorization path. Production runs that same path, and a "no authorization" mode carved into it for the lab's convenience is a permanent hazard in exchange for a temporary simplification.

## Consequences

`aud` no longer distinguishes this server from any other in the home lab, so it is not evidence about which server a caller intended to reach and nothing may start treating it as such.

The MCP endpoint becomes reachable from the public Internet over HTTPS, with the server's own token validation as the only thing in front of it. That is why none of the validation may be softened, and it is stated in the deployment documentation rather than left to be discovered.

The client identifier in an accepted token now belongs to a client that registered itself minutes ago and that no reviewed configuration has ever seen. Any code treating `azp`, `appid` or a client id as evidence of anything beyond "this token validated" is wrong under this deployment.

Deliberately left open: the server does not become an authorization server, gains no token or registration endpoint, and Prompt 18's fixed-token mode stays unimplemented.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
