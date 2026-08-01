# Stage 17 — Pull and deploy locally with LocalStack, Keycloak, and shared Caddy

Source: `BrightFlagProxyMCPBuilder/prompts/17-local-pull-localstack-keycloak-caddy-deployment.md`

## Context

The destination machine now has shared LocalStack, Keycloak, and Caddy infrastructure. The selected
deployment trust split keeps GitHub responsible for reviewed immutable image publication, while a
trusted operator runs the pull and deployment script locally. Secrets Manager remains the source of
truth, Keycloak becomes the live caller issuer, and Caddy becomes the sole TLS edge.

## Preconditions

- Stages 1–11 are merged and green. Stage 16 is complete if Stage 14 or 15 was applied.
- Protected `main` publishes a verified `linux/amd64` GHCR image with an immutable digest and OCI
  source revision, but no GitHub workflow deploys locally.
- The actual LocalStack, Keycloak, PostgreSQL, Caddy, Docker, network, DNS, and host state is
  inspected and dated; exact dependency versions or unresolved moving tags are recorded.
- LocalStack Secrets Manager is reachable over host loopback and demonstrably refused from a second
  LAN machine before real credentials are created.
- The canonical Keycloak issuer and Caddy certificate resolve and validate from the host, container,
  and intended client network.

## Scope in

Local PowerShell pull/deploy entry point; LocalStack secret materialisation into existing file
providers; Keycloak live-provider configuration and idempotent realm/client artifact; shared-Caddy
fragment and `edge_net`; LAN-only reachability; immutable deployment and rollback records; operator
documentation and automated, host, LAN, identity, and edge validation.

## Scope explicitly out

Adding an AWS SDK or vendor-specific server secret provider; owning shared infrastructure stacks;
changing the separately managed LocalAI repository; committing real secrets or tokens; publishing a
backend port; using local JWKS or disabled TLS validation; enabling payment; Internet exposure;
automatic ambiguous rollback; contingent Stages 12 or 13.

## Work items

### 1. Capture and gate the shared-infrastructure contract

Record versions, paths, ownership, network, ports, canonical names, and commands for LocalStack,
Keycloak, PostgreSQL, Caddy, and Docker. Make loopback-only LocalStack access, normal TLS, DNS, and
dependency pinning explicit prerequisites, with separately reviewed LocalAI changes where needed.

### 2. Materialise LocalStack secrets safely

Have the host script fetch only configured secret identifiers over loopback and atomically create
ACL-protected service-token and cursor-key files for the existing File providers. Redact all
output, fail before replacement on any error, and implement safe rotation without AWS coupling in
the server.

### 3. Configure live Keycloak trust

Set the canonical realm issuer, JWKS URI, audience, claims, tenant, and roles. Add an idempotent,
secret-free public-client configuration using Authorization Code and PKCE with exact redirects.
Serve OAuth Protected Resource Metadata and its 401 challenge with the canonical resource, issuer,
and read scope. Assign read only, keep payment disabled, and verify canonical DNS, TLS, S256,
`resource` handling, and audience issuance end to end from OpenCode.

### 4. Move the service behind shared Caddy

Remove nginx and published ports, join `edge_net`, trust only Caddy as the proxy, and install one
BrightFlag-owned LAN-only fragment after whole-configuration validation. Reload without
restarting Caddy and preserve other services. Document fragment denial/removal as
BrightFlag-specific isolation. Require a canonical MCP hostname, split-DNS path to Caddy, and
normal certificate trust without Internet application exposure.

### 5. Implement immutable local pull and deployment

Accept only an exact digest and revision, use an ephemeral package-read credential, inspect
architecture and labels, serialise changes, record both artifacts, preflight dependencies, deploy
one payment-disabled instance, validate through Caddy, and provide manual rollback.

### 6. Update validation and operations documentation

Cover synthetic integration, live host gates, OpenCode handoff, rotation, recovery, Caddy reload,
certificate renewal, and emergency isolation. Bound stop and rollback to BrightFlag-owned
resources.

## Tests

Map one-to-one to the prompt's Prove list:

1. search for executable self-hosted deployment paths and run or inspect verified GHCR publication;
2. render Compose and assert one server, no nginx, no host ports, external `edge_net`, and no
   payment overlay;
3. test LocalStack loopback success and second-machine LAN refusal before real-secret bootstrap;
4. use synthetic secrets to test protected materialisation, redaction, missing/malformed failure,
   Docker metadata and process arguments, and rotation;
5. exercise protected-resource metadata, its 401 challenge, canonical Keycloak discovery/JWKS, the
   pre-registered OpenCode PKCE flow, `resource` and audience binding, plus anonymous, expiry,
   issuer, audience, tenant, and role negative-token cases;
6. invoke discovery, ontology, read, payment-plan, and payment-execute paths as the read-only
   caller;
7. validate and reload the complete Caddy configuration, diff unrelated fragments and routes, test
   canonical split-DNS TLS and streaming, and prove direct backend refusal;
8. exercise every local-script pre-change refusal for mutable references, architecture, revision,
   LocalStack exposure, placeholders, issuer trust, and Caddy validity;
9. inspect the running digest and MCP surface, capture outbound traffic during synthetic validation,
   and assert payment remains disabled;
10. rerun the same digest and exercise the documented manual rollback with synthetic configuration;
    and
11. inspect and, in a disposable fixture, exercise stop, rollback, fragment removal, and isolation
    boundaries without touching SSH or shared infrastructure.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

```bash
docker compose -f deploy/local/compose.yaml config
```

Run the prompt's exact Windows, second-machine, Keycloak, DNS/TLS, Caddy, synthetic-secret,
same-digest, and rollback checks. Real secrets may be introduced only after the dated LocalStack LAN
refusal passes. Every check not run on the intended system remains a manual gate.

## Stage boundary

Commit locally. Suggested message: `Add shared-infrastructure local deployment`.

Use `narrative-required` when published. Do not push unless requested. Do not enable payment, expose
the service publicly, or begin contingent Stages 12 or 13.

## Risks

- LAN-reachable unauthenticated LocalStack would expose every stored BrightFlag secret; fail before
  real-secret creation rather than treating test AWS credentials as authentication.
- An internal Keycloak hostname would produce tokens with the wrong issuer or invite disabled TLS;
  preserve one canonical HTTPS issuer end to end.
- Updating the root Caddyfile or restarting shared Caddy could disrupt unrelated services; own one
  fragment and validate the whole configuration before reload.
- Host materialisation creates protected runtime copies of secrets. ACL, redaction, atomic rotation,
  and deletion boundaries remain security-critical even though LocalStack is the source of truth.
- Shared router port 443 cannot isolate BrightFlag without also affecting Keycloak; retain a
  service-specific Caddy deny or fragment-removal procedure.
