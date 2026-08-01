# Prompt 17 — Pull and deploy locally with LocalStack, Keycloak, and shared Caddy

Using the reusable contract and the artifacts produced by Prompts 1–11, implement the replacement
local deployment for `BrightFlagProxyMCPServer`. A PowerShell script running on the destination
machine pulls one reviewed image from GHCR and deploys it beside existing LocalStack, Keycloak, and
Caddy containers.

For a repository where Prompts 14 or 15 were applied, complete Prompt 16 first. For a fresh
implementation, skip Prompts 14–16 and apply this prompt directly after Prompt 11. This stage does
not depend on or authorise contingent Prompts 12 or 13.

## Establish the site contract

Inspect and date the actual host state before encoding it. The intended site currently has:

- Windows 11 and Docker Desktop in Linux-container mode on `ai-mcp-server` at `192.168.50.6`;
- LocalStack in `C:\mcp\localstack`, providing the Secrets Manager API;
- Keycloak 26 with PostgreSQL in `C:\mcp\keycloak`, exposed through the canonical issuer
  `https://auth.tqaentry.com`;
- a shared Caddy stack in `C:\mcp\edge`, owning ports 80 and 443, importing one service-owned file
  from `C:\mcp\edge\conf.d`, and attached to the external Docker network `edge_net`; and
- protected-main GitHub-hosted CI publishing a verified `linux/amd64` image to GHCR.

Treat these as observations to re-check, not timeless facts. Pin or record the exact Keycloak,
LocalStack, Caddy, PostgreSQL, and deployment image versions or digests. If an existing dependency
uses a moving tag, report reproducibility as an open manual gate rather than claiming it.

The BrightFlag repository consumes these services; it does not own, recreate, restart, stop, roll
back, or delete their stacks. Any required change to the separately managed LocalAI configuration
must be named as an external prerequisite and reviewed there, not silently copied into this repo.

## Fail closed on the LocalStack security prerequisite

The current LocalStack endpoint is unauthenticated and accepts arbitrary AWS test credentials. Do
not place a real BrightFlag credential or cursor-signing key in it while port 4566 or its service
range is reachable from another LAN machine. Before real-secret bootstrap, require read-only proof
that the Secrets Manager endpoint is bound to loopback or otherwise denied from the LAN. Test both
host loopback success and second-machine refusal.

If that isolation has not been implemented in the separately managed LocalStack stack, stop before
real-secret creation and leave an exact manual gate. Synthetic integration secrets may still be
used. Do not attempt to compensate with invented AWS credentials, obscurity, Caddy, or application
authorization.

## Preserve the vendor-neutral secret-provider boundary

Do not add the AWS SDK, a LocalStack endpoint, or an AWS-specific secret provider to the MCP server.
The host deployment script uses the local Secrets Manager API over loopback, retrieves only the
configured BrightFlag service-token and cursor-signing-key secret identifiers, and materialises
their values as protected runtime files consumed by the existing `File` secret providers.

- Secret identifiers are non-secret configuration; values never enter Git, `.env`, command
  arguments, process listings, Docker metadata, or logs.
- Write files atomically, apply restrictive NTFS ACLs, validate non-empty values, and remove partial
  files on failure before starting or replacing the server.
- Never print secret values. Redact command output and errors, including AWS CLI diagnostics.
- Rotation re-reads LocalStack, replaces both files atomically, restarts only the BrightFlag server,
  and verifies readiness. Document the rollback and ambiguity boundary.
- TLS keys remain owned by Caddy. Keycloak database and administrator secrets remain owned by the
  Keycloak stack and are outside this deployment.

Use configurable secret identifiers, with examples such as `brightflag/mcp/service-token` and
`brightflag/mcp/cursor-signing-key`; do not create usable real values in repository fixtures.

## Configure Keycloak as the live caller identity provider

Use Stage 8's live trust provider. Do not install a development private signing key, checked-in
JWKS, static long-lived bearer token, or disabled certificate validation. Configure and test:

- issuer `https://auth.tqaentry.com/realms/brightflag-mcp`;
- JWKS URI
  `https://auth.tqaentry.com/realms/brightflag-mcp/protocol/openid-connect/certs`;
- audience `brightflag-mcp`; and
- the existing issuer, audience, expiry, not-before, tenant, and role validation.

Add an idempotent, reviewable realm/client configuration artifact or bootstrap script that contains
no secret. Prefer a public OpenID Connect client for OpenCode using Authorization Code with PKCE
S256, exact operator-supplied redirect URIs, and no wildcard. Disable direct grants and service
accounts initially. Map flat claims matching the server contract: tenant `tid`, capability roles,
and any required `groups` or `scp` claims.

Make the server a conforming OAuth protected resource, not an authorization server. Serve OAuth
Protected Resource Metadata at the standard well-known URI with the canonical MCP HTTPS resource,
the Keycloak realm in `authorization_servers`, and the minimum read scope. Include its absolute
`resource_metadata` URI and required scope in `WWW-Authenticate` on 401 responses. Preserve the
canonical MCP resource as the access-token audience. Verify Keycloak discovery advertises S256 and
that the selected pre-registered public-client flow accepts the client's `resource` parameter and
still issues an access token with the exact MCP audience.

Dynamic client registration is not required for this known local client. Keep it disabled unless a
separate threat review authorises it. Supply the non-secret pre-registered client ID and exact
loopback redirect to the operator; do not create or distribute a public-client secret.

Define `brightflag.read` and `brightflag.payment`, but assign only the read role during first
deployment. Keep the payment overlay disabled. If the installed OpenCode version cannot complete
the selected OAuth flow, record a client-compatibility gate; do not weaken token validation or
replace it with a committed signing key or static bearer token.

The server container must resolve the canonical issuer hostname and validate Caddy's normal TLS
chain. Document and test one exact split-DNS, hosts, or host-gateway mechanism. Never substitute an
internal HTTP issuer or turn off TLS verification.

## Join the existing Caddy edge

Replace the dedicated nginx/8443 topology. The BrightFlag server joins external `edge_net`,
publishes no host port, and is reached by Caddy at its container name and internal HTTP port. Add
one BrightFlag-owned Caddy fragment for `C:\mcp\edge\conf.d`; do not overwrite the root Caddyfile or
another service's fragment.

Require one operator-supplied canonical HTTPS hostname for the MCP resource. Use split DNS or a
narrowly scoped client override so the host, server container, and local clients resolve it to the
shared Caddy edge, while Caddy obtains or reuses a normally trusted certificate through the site's
existing certificate policy. Do not expose the application route to the Internet merely to issue a
certificate, and do not disable hostname or chain validation.

Caddy terminates TLS and reverse-proxies Streamable HTTP to the server. Configure the server's
trusted-proxy boundary for Caddy, preserve streaming flush behaviour, bounded request bodies, and
appropriate transport timeouts, and keep authentication and authorization inside the application.
Caddy is not the OAuth enforcement point.

Make the BrightFlag route LAN-only by default using an explicit private-source policy that fails
closed. Validate the complete Caddy configuration before an atomic fragment update, reload Caddy
without restarting it, and prove unrelated routes remain unchanged. A separate, reviewed exposure
decision is required before Internet access. Because shared 443 also serves Keycloak, router
shut-off is no longer a BrightFlag-only isolation switch; removing or denying the BrightFlag Caddy
route is the service-specific emergency cut-off.

## Local immutable-image deployment

Add an operator-run PowerShell deployment entry point under `deploy/local`. It must:

1. accept an exact lowercase `ghcr.io/...@sha256:<64-hex>` reference and expected 40-character
   source revision, never a tag or `latest`;
2. obtain a least-privilege package-read credential at run time, avoid persisting or logging it, and
   run `docker logout` on success or failure;
3. pull and inspect the exact digest, requiring `linux/amd64` and the expected
   `org.opencontainers.image.revision` before changing the stack;
4. serialise deployments and record the outgoing and incoming digest and revision;
5. check LocalStack isolation, retrieve and materialise secrets, validate Keycloak discovery/JWKS,
   validate the Caddy configuration, and run preflight before replacement;
6. deploy the single BrightFlag instance with payment disabled and no published backend port;
7. update only the BrightFlag Caddy fragment, reload the existing edge, and verify readiness through
   the configured HTTPS MCP hostname without contacting BrightFlag; and
8. verify the running image and disabled payment state. On failure, provide an explicit manual
   rollback using the recorded artifact; never automatically roll back an ambiguous deployment.

A same-digest rerun is an idempotent readiness and configuration reconciliation. GitHub Actions
must have no local deployment job, and the local script must not require a self-hosted runner.

## Validate and document operations

Update `docs/deploy-local.md` and the repository README. Separate observed state, operator inputs,
automated evidence, and unrun manual gates. Document secret bootstrap and rotation, Keycloak realm
bootstrap and token acquisition, Caddy fragment validation/reload/removal, immutable deployment,
OpenCode handoff values, health and diagnostics, manual rollback, certificate renewal, and
emergency isolation. Do not include a secret, token, private key, private JWKS member, or invented
tenant value.

`Stop.ps1`, rollback, and uninstall operations affect only BrightFlag-owned containers, files, and
Caddy fragment. They must never stop or remove Caddy, Keycloak, LocalStack, PostgreSQL, `edge_net`,
Jamie's SSH access, or another service's configuration.

## Prove

- no self-hosted GitHub runner or GitHub-triggered local deployment path remains, while verified
  protected-main GHCR publication still succeeds;
- the Compose model has one BrightFlag instance, no nginx, no host ports, an external `edge_net`,
  and payment disabled;
- a second LAN machine cannot reach LocalStack before any real BrightFlag secret is stored, while
  host loopback can reach the required Secrets Manager API;
- synthetic LocalStack secrets are materialised into protected files without appearing in Git,
  `.env`, Docker metadata, process arguments, test output, or logs; missing or malformed secrets
  fail before the running stack changes, and rotation is demonstrated;
- protected-resource metadata and the 401 challenge identify the canonical MCP resource, Keycloak
  issuer, and minimum read scope; Keycloak discovery and JWKS work over validated TLS, the
  pre-registered OpenCode PKCE flow produces a valid read token, and anonymous, expired,
  wrong-issuer, wrong-audience, wrong-tenant, and unassigned-role tokens are refused;
- the read-only caller can list tools, read the ontology resource, and invoke read operations but
  cannot plan or execute payment;
- Caddy validation and reload preserve every unrelated fragment and route, serve the MCP endpoint
  at its canonical split-DNS hostname over trusted TLS, preserve Streamable HTTP behaviour, and
  keep the backend unreachable directly;
- the local script refuses tags, wrong architecture, digest/revision mismatch, reachable-LAN
  LocalStack, unresolved placeholders, unavailable issuer trust, or invalid Caddy configuration
  before replacing the running server;
- the deployed container uses the requested digest, exposes exactly four tools and one resource,
  does not contact a live BrightFlag tenant during validation, and keeps payment disabled;
- a same-digest rerun is idempotent, and the recorded previous digest supports a manual rollback;
  and
- stop, rollback, fragment removal, and emergency-isolation procedures affect only BrightFlag-owned
  resources and preserve Jamie's SSH access plus the shared infrastructure stacks.

## Acceptance criteria

- A trusted operator on `ai-mcp-server` can pull a reviewed immutable image, consume protected
  runtime copies of LocalStack-managed secrets, and expose an authenticated read-only MCP endpoint
  through the existing Caddy instance without a GitHub self-hosted runner.
- Keycloak is the only live caller-token issuer, the server publishes standard OAuth protected
  resource discovery, Caddy is the only TLS listener, and LocalStack is the source of truth for
  BrightFlag runtime secrets without adding vendor coupling to server code.
- Real-secret deployment fails closed until LocalStack is unreachable from the LAN and all normal
  TLS, DNS, issuer, digest, revision, Caddy, and payment-disabled checks pass.
- All automated repository checks pass. Windows, LAN, shared-Caddy, Keycloak, and real-secret checks
  remain labelled manual gates until their commands and dated results are recorded.

Commit locally. Use `narrative-required` and record the host-pull trust boundary, LocalStack
materialisation decision, canonical Keycloak issuer, shared-Caddy ownership boundary, LAN-only
default, and disabled payment state. Do not push unless requested.
