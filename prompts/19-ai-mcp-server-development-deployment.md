# Prompt 19 — `ai-mcp-server` development deployment

Using the reusable contract and the artifacts produced by Prompts 1–11, 17, and 18, hand ownership
of the `ai-mcp-server` deployment to a script in the separately managed LocalAI repository, retire
the server-owned `deploy/local` deployment, and make the server serve the canonical development
endpoint `http://brightflag-mcp.tqaentry.com/mcp`.

This stage is the development deployment for one physically controlled home-lab host. It is not a
production deployment, it prescribes nothing about how this server is deployed or authenticated in
production, and nothing in it may be read as a claim that a plaintext endpoint or a shared static
identity is suitable beyond that host.

Apply this stage after Prompt 18. It does not depend on or authorise contingent Prompts 12 or 13.

## What this stage supersedes

Prompt 17 remains historical and is not rewritten. When the sequence is played in order, this stage
replaces the following implemented home-lab decisions and nothing else:

| Prompt 17 | Prompt 19 |
|---|---|
| Immutable GHCR digest | Resolve a configurable Git ref and build its exact commit |
| Keycloak only | Exclusive fixed-token or Keycloak selection |
| Payment disabled | Read and payment always enabled |
| LocalStack LAN isolation required | Existing LAN-accessible LocalStack accepted |
| `edge_net` | `mcp-public` |
| `C:\mcp\edge` | `C:\mcp-host` |
| HTTPS MCP resource | Plaintext LAN MCP resource |
| HTTPS trusted-proxy transport | Explicit plaintext home-lab transport |
| Dedicated Keycloak realm | Shared `homelab` realm |
| Server `deploy/local` owns deployment | LocalAI script is the sole deployment owner |
| Container hardening and resource limits | Intentionally omitted from this open home-lab Compose |

Every other Prompt 17 requirement survives. Do not treat this table as licence to relax an
unlisted control.

## Retire the server-owned deployment

The BrightFlag server repository stops owning any deployment of itself. Delete:

- `deploy/local/compose.yaml`;
- the `deploy/local` Caddy template;
- `deploy/local/manual-gates.md`;
- every script under `deploy/local/scripts/`; and
- `docs/deploy-local.md`.

Replace them with documentation of the LocalAI-owned deployment: what it produces, what the server
requires from it, and where it lives. Any deployment reference that is retained for history must be
marked superseded and inert, so no reader is left with two apparently supported ways to deploy this
server. Keep GitHub-hosted continuous integration and its verified GHCR publication; that flow is no
longer the source of this development deployment, and the documentation must say so rather than
leaving both paths looking current.

The deployment owner is `LocalAI/docs/setup-brightflag-mcp-windows.ps1`. That file is changed in the
LocalAI repository, under its own instructions and review. Do not copy it into this repository, and
do not reintroduce a deployment entry point here.

`ai-mcp-server` runs PowerShell 7.6.4, so that script targets PowerShell 7.6 or later and declares
the minimum. Windows PowerShell 5.1 is not a target. Anything in this repository that documents,
tests, or asserts against the deployment script assumes the same floor, and must not be written to
the 5.1 subset in the belief that doing so is safer.

## Serve the canonical plaintext endpoint honestly

The canonical development endpoint is `http://brightflag-mcp.tqaentry.com/mcp`. Caddy fronts it with
an explicit `http://` site address so Caddy does not enable automatic HTTPS for the name.

Add an explicit plaintext transport mode to the HTTP transport's existing posture choice, beside
direct TLS termination and the trusted-proxy posture Prompt 8 defined. The plaintext mode:

- is selected explicitly by configuration, exactly as the other postures are, and is never a default
  or a fallback;
- accepts an `http://` external resource URI without claiming an HTTPS trusted-proxy topology, and
  without asserting a fronting proxy terminated TLS; and
- must not be rejected by a deployment-profile guard. Selecting it is the operator's explicit
  decision for this host.

Do not describe this deployment as an HTTPS trusted-proxy deployment, and do not reuse the
trusted-proxy posture to stand in for it. The two make different claims, and only one of them is
true here. The fixed token and Keycloak access token are both bearer credentials. They cross this
LAN in plaintext, which is accepted for this physically controlled network and must be stated
wherever the endpoint is documented.

Add host validation to the HTTP transport. It permits exactly:

- `brightflag-mcp.tqaentry.com`;
- `localhost`; and
- the container alias the Compose healthcheck uses.

Anything else is refused. The healthcheck must send a `Host` value inside that set, so a working
healthcheck is evidence the validation is configured, not a hole in it.

## The deployment the server is built for

Document and validate against this deployment shape. It is produced by the LocalAI script; the
server's job here is to be configured for it, to be provable against it, and to state its limits.

**Caddy.** One BrightFlag-owned fragment under `C:\mcp-host\caddy\conf.d`, added to the shared Caddy
without touching the root Caddyfile or another service's fragment. The site is the explicit
`http://` address above. It carries a private-source matcher intended to restrict the route to LAN
sources, and Streamable HTTP proxy settings that prevent response buffering.

State the matcher's status exactly: whether it reliably distinguishes a private source from an
Internet source through Windows, Docker Desktop, and router forwarding is **unproven**, and is
tracked by issue [#45][issue-45]. It is configured because it is the right shape, not because it has
been demonstrated. Do not describe it as a completed rejection property, do not present it as an
Internet-exposure control, and do not make its untested behaviour an acceptance gate.

**Container.** One BrightFlag container on the external Docker network `mcp-public`, publishing no
host port, reachable only through the shared Caddy at its internal port. It carries a self-contained
`extra_hosts` entry mapping `auth.tqaentry.com` to `host-gateway`, so the container resolves and
fetches the HTTPS Keycloak issuer's discovery and JWKS documents without any change to the shared
Caddy or Keycloak deployment. The plaintext MCP endpoint does not weaken the Keycloak issuer,
discovery, token, or JWKS endpoints, which stay HTTPS.

**Capabilities.** Both authentication modes permit the complete surface. The generated
configuration names the payment grant explicitly —
`BrightFlag__Authorization__MarkInvoicePaidRoles__0` alongside the corresponding
`BrightFlag__Authorization__ReadApprovedInvoicesRoles__0` value — so payment is enabled by a named
role value rather than by an overlay file's absence or presence.

**Configuration contract.** The deployment supplies configuration as environment variables, and the
server's option names are therefore a cross-repository contract rather than a private detail. These
names must match what the LocalAI script generates:

- `BrightFlag__CallerIdentity__Mode` — `FixedToken` or `Keycloak`, the Prompt 18 selection;
- `BrightFlag__CallerIdentity__FixedToken__Subject`, `__Tenant`, `__Roles__0`, `__Roles__1`, and
  `__Source__Kind` / `__Source__Path` for the mounted token file;
- `BrightFlag__Hosting__HttpTransportSecurity` — the explicit plaintext value added by this stage;
- `BrightFlag__Hosting__AllowedHosts__0..2` — the three permitted host values; and
- `BrightFlag__Approval__AllowedInvoiceStatuses__0` — the reviewed integration-test tenant status,
  supplied explicitly to the deployment script with no default; and
- `BrightFlag__Authorization__ReadApprovedInvoicesRoles__0` and
  `BrightFlag__Authorization__MarkInvoicePaidRoles__0`.

A rename on either side is a breaking change to the other repository and must be made in both.

**Deliberate omissions.** This Compose intentionally omits Prompt 17's container user, read-only
root filesystem, capability drop, `no-new-privileges`, `tmpfs`, CPU and memory limits, and log
rotation. That is an accepted open posture for this host, not an oversight. Record it as omitted on
purpose; do not reintroduce those settings here, and do not describe the container as hardened.

## Build the exact commit, and be able to go back to it

The deployment builds from a configurable private repository URL and a configurable Git ref. It
resolves that ref **once** to a full 40-character commit before building, then builds and tags that
exact commit and passes it as `SOURCE_REVISION` so the image label reports the source it was built
from.

- The requested ref, the resolved commit, the image tag, the image ID, and the previous deployable
  image are recorded as deployment state.
- The previous image is retained, and rollback recreates only BrightFlag from it — without
  resolving or rebuilding a moving ref, which would defeat the point of having recorded one.
- The GitHub credential used to fetch a private repository is transient. It never reaches
  LocalStack, Compose, a generated file, a log, or a command argument.

Say plainly what this is and is not. Resolving a ref once and recording the commit makes a
deployment reproducible and reversible; it does not make a mutable branch immutable. Do not describe
a branch build as digest-pinned or immutable.

## Secrets stay outside the application

Three configurable LocalStack Secrets Manager identifiers supply the fixed MCP token, the
cursor-signing key, and the BrightFlag integration-test service token. The deployment retrieves them
without printing values, materialises them atomically into ACL-protected runtime files, and the
server reads those files through its existing `File` secret sources.

No AWS SDK, LocalStack endpoint, or vendor-specific secret provider enters application code. That
boundary is unchanged from Prompt 17 and is not negotiable here.

This deployment adds no isolation gate to LocalStack and changes none of its network, port, or
firewall configuration. LocalStack remains unauthenticated and reachable from other LAN machines,
which means any LAN caller able to read it can obtain the fixed MCP token, the cursor-signing key,
and the BrightFlag integration-test credential. This is accepted for this network and is recorded,
not mitigated. Prompt 17's fail-closed rule against real-secret bootstrap while LocalStack is
LAN-reachable is superseded for this development deployment only.

## Keycloak stays HTTPS and shared

- Issuer `https://auth.tqaentry.com/realms/homelab`, in the shared `homelab` realm.
- Audience and resource client `brightflag-mcp`.
- Device authorization through the existing public caller client, which supplies a token the MCP
  client sends as a fixed `Authorization: Bearer` header.
- A hardcoded `tid` mapper carrying the exact configured tenant value, and a mapper projecting the
  `brightflag-mcp` client roles into a flat top-level `roles` array — the claim shape the server
  requires.
- The designated development user holds both `brightflag.read` and `brightflag.payment`.
- No public-client secret, and no password, access token, or refresh token in generated
  configuration.

The shared caller client's audience mapper is not a service-separation boundary; the required
BrightFlag roles are. Further client separation and hardening are tracked by issue [#46][issue-46].

## BrightFlag remains integration-test only

The BrightFlag endpoint and credential configured here refer only to BrightFlag's integration-test
environment. No production BrightFlag API URL or production credential exists for this deployment,
and none may be introduced by it. Payment is enabled against the integration-test tenant only. The
operator must supply that tenant's reviewed approved-invoice status explicitly; the deployment
script has no default and must not guess one from a plausible label.

## Prove

- no deployment entry point, Compose file, Caddy template, manual-gates file, or `deploy-local`
  document remains in this repository, every retained deployment reference is marked superseded and
  inert, and the documentation names the LocalAI script as the sole deployment owner while GHCR
  publication is described as CI evidence rather than a deployment path;
- the explicit plaintext transport mode is selected by configuration, accepts the `http://` resource
  URI, makes no HTTPS or trusted-proxy claim, is rejected as an accidental default, and is not
  refused under any deployment profile;
- host validation accepts `brightflag-mcp.tqaentry.com`, `localhost`, and the healthcheck's
  container alias, refuses every other `Host` value, and the configured healthcheck's `Host` value
  is inside the accepted set;
- the generated Compose declares one BrightFlag container on external `mcp-public`, publishes no
  host port, and carries the `auth.tqaentry.com:host-gateway` mapping and the expected healthcheck;
- the generated Compose contains no unresolved placeholder and no secret value, and deliberately
  omits the container user, read-only root filesystem, capability drop, `no-new-privileges`,
  `tmpfs`, CPU and memory limits, and log rotation, each recorded as an intentional omission; it
  contains the explicitly supplied reviewed approved-invoice status, and the deployment script has
  no default for that value;
- the generated Caddy fragment uses the explicit `http://` site address, carries the private-source
  matcher and non-buffering Streamable HTTP settings, leaves every unrelated fragment and the root
  Caddyfile unchanged, and documents the matcher as unproven and tracked by [#45][issue-45];
- the deployment state record contains the requested ref, the full resolved commit, the image tag,
  the image ID, and the previous deployable image; a build from a full commit passes
  `SOURCE_REVISION` and the resulting image label reports that commit; and rollback recreates only
  BrightFlag from the retained image without resolving or rebuilding the ref;
- the three configured LocalStack identifiers are materialised atomically into ACL-protected files
  consumed by the existing `File` secret sources, values appear in no log, error, Compose file,
  process argument, or Docker metadata, and no AWS SDK, LocalStack endpoint, or vendor secret
  provider is reachable from application code;
- both authentication modes reach the complete surface, with the payment grant named by
  `BrightFlag__Authorization__MarkInvoicePaidRoles__0` alongside the read role, and neither mode
  falls back to the other;
- a Keycloak device-flow token carries the configured flat `tid` and `roles` claims, the
  `brightflag-mcp` audience, and the HTTPS issuer, and discovery and JWKS retrieval succeed from
  inside the container through `auth.tqaentry.com`; and
- no configuration, fixture, document, or generated file references a BrightFlag production URL or
  credential, and nothing in this stage changes LocalStack's network, ports, or firewall rules.

## Acceptance criteria

- This repository no longer deploys itself. One LocalAI-owned script builds a named commit,
  materialises the three secrets, generates the Compose project and the Caddy fragment, and can roll
  back to a retained image.
- The server serves `http://brightflag-mcp.tqaentry.com/mcp` under an explicitly selected plaintext
  transport mode that states what it is, with host validation permitting exactly three names.
- Both read and payment capabilities are available in both authentication modes, against
  BrightFlag's integration-test environment only, using an explicitly supplied reviewed
  approved-invoice status rather than a deployment default.
- The open posture is recorded, not disguised: the omitted container hardening, the unproven
  private-source matcher, the LAN-readable unauthenticated LocalStack, and the plaintext bearer
  credentials are each stated where a reader will meet them.
- All automated repository checks pass. Every Windows, Docker Desktop, LAN, Caddy, Keycloak, and
  LocalStack check that cannot run in this environment is a labelled manual gate with its exact
  command and expected result, and is never reported as passing.

Commit locally. Use `narrative-required` and record the transfer of deployment ownership to LocalAI,
the retirement of the `deploy/local` artifacts, the explicit plaintext transport mode and canonical
HTTP endpoint, the resolved-commit build and retained-image rollback model, the enabled payment
capability in both authentication modes, and the accepted open posture of this development host. Do
not push unless requested.

[issue-45]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/45
[issue-46]: https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/issues/46
