# Prompt 10 — Generate a homelab and local-network deployment guide

Using the reusable contract and the artifacts produced by Prompts 1–9, implement Stage 10:
an operator-ready deployment path for one MCP server instance on a trusted home-lab or local-network
host. This stage documents and automates deployment; it must not widen the MCP or BrightFlag surface.

## Supported baseline

Choose and state one primary, reproducible baseline:

- a Windows 11 host with Docker Desktop, its WSL 2 Linux-container backend, and the bundled Docker
  Compose plugin;
- one server container using the image produced by Prompt 9;
- exactly one dev instance, with no horizontal scaling or session affinity — a choice this stage
  makes for its own homelab/local-network purposes, independently of whatever live topology Prompt 8
  currently declares. The two happen to match today because Prompt 8 currently declares a
  single-instance live topology, but this deployment does not derive its instance count from that
  decision: if Prompt 8 is later revised to a multi-instance live topology, this homelab deployment
  still runs exactly one instance; and
- Streamable HTTP at `/mcp`, reachable from explicitly allowed LAN clients only.

Use PowerShell 7 for host-side commands and identify every command that instead runs inside WSL or a
container. The guide may note equivalent Windows Server, Hyper-V, Podman, or NAS steps, but do not
claim them supported unless they are tested. Record the tested Windows edition and build, CPU
architecture, WSL version, Docker Desktop and engine versions, Compose version, container mode, and
client environment. Use placeholders for hostnames, addresses, tenant values, and credentials.
Detect and fail preflight if Docker is using Windows containers rather than Linux containers.

State Docker Desktop's licensing, interactive-logon, automatic-start, update, and availability
assumptions explicitly. Do not describe a container restart policy as proof that the stack starts
after a Windows reboot. Provide and test a least-privilege Windows Task Scheduler startup procedure
if unattended reboot recovery is claimed; otherwise document operator sign-in and Docker Desktop
startup as a manual availability gate.

## Deployment artifacts

Add a production-quality example Compose file and checked-in PowerShell preflight, validation,
start, stop, upgrade, and rollback scripts. Set strict mode, stop on errors, reject unresolved
placeholders, and avoid changing machine-wide policy. Pin the image by immutable digest or require
the operator to supply a digest; never deploy `latest`. Configure:

- a non-root user, read-only root filesystem, dropped Linux capabilities, `no-new-privileges`,
  bounded CPU and memory, a small writable temporary filesystem, and a restart policy;
- a health check against the Prompt 9 readiness endpoint;
- an explicit port binding, preferably to a dedicated LAN address rather than every interface;
- a persistent, access-controlled plan-store volume only if the chosen implementation requires it;
- log rotation and a documented location or driver that does not capture secrets; and
- all required configuration with fail-closed placeholder detection.

Do not mount the container engine socket. Do not use host networking, privileged mode, embedded
credentials, plaintext secret values in Compose, or an automatically downloaded deployment script.
Keep the Prompt 8 dev token-issuing executable and private signing key outside the server image.
Use Windows paths deliberately: keep deployment files outside OneDrive and other synchronized
folders, quote paths containing spaces, avoid relying on drive sharing that pre-dates the WSL 2
backend, and document Docker Desktop file-sharing behavior for every bind mount.

## Network boundary and TLS

Document a concrete network layout and request path from an allowed MCP client to `/mcp`. Account
for the Windows host, Docker Desktop's WSL 2 virtual network, published container ports, and any
reverse-proxy container. Bind only the TLS entry point to the intended Windows LAN address. Bind the
unencrypted application port to loopback or an internal Compose network so it is never a LAN
listener. Give idempotent PowerShell commands using Windows Defender Firewall to create narrowly
named inbound rules for the selected TCP port, LocalAddress, RemoteAddress clients or VLAN, profile,
program or service where meaningful, and default-deny behavior. Include equally exact inspection
and removal commands that affect only those rules.

Use HTTPS for traffic beyond loopback. Choose one documented TLS pattern:

- a reverse proxy on the same host, with the server port reachable only from that proxy; or
- a trusted private overlay that provides authenticated encryption and restricts membership.

For the reverse-proxy pattern, include a minimal pinned configuration, certificate provisioning or
private-CA trust steps, correct forwarding behavior for Streamable HTTP, request and idle timeouts,
body-size limits aligned with Prompt 8, and a test proving the unencrypted backend port is not
reachable from another LAN machine. Do not publish the service directly to the public internet,
configure automatic router port forwarding, or describe a self-signed certificate as trusted
without installing its CA certificate on each client.

Use Windows certificate-store terminology and PowerShell certificate inspection commands. If a
private CA is used, install only its public root certificate into the appropriate Trusted Root store
on each authorized client, never the CA private key. Explain hostname/SAN matching and prefer a
stable local DNS name over a raw address. Do not disable certificate validation in example clients.

## Local caller identity bootstrap

For a deployment explicitly marked non-production, document how an operator:

1. generates the local signing key and public JWKS on an administrator-controlled machine;
2. supplies only the public JWKS and expected issuer, audience, and required-claim configuration to
   the server;
3. runs Prompt 8's separate dev token-issuing tool outside the server container;
4. issues a short-lived token for a dummy caller with the minimum read-only capability first;
5. configures the LAN MCP client to send that bearer token; and
6. proves expired, wrong-audience, and unauthorized tokens are rejected.

Private signing material must never enter the server image, Compose file, repository, logs,
PowerShell history, or client configuration. Explain that anyone holding the local signing key can
impersonate any configured caller. Require restrictive NTFS ACLs, backup and rotation guidance, and
a separate deliberate enablement step for the payment capability. Do not rely on Unix mode bits on
a Windows bind mount as the host-side access-control boundary.

State prominently that the local trust provider is for non-production functional investigation
only. Preserve Prompt 8's startup failure outside an explicitly non-production profile. Link the
operator to Prompt 9's runbook for switching to the organisation's live identity provider; do not
represent a LAN boundary, reverse proxy, VPN, or private CA as a substitute for that migration.

## BrightFlag and application secrets

Document how to provide the BrightFlag bearer token, cursor-signing key, and any plan-store
credential through the secret-provider interface created in Prompt 3. Prefer file references backed
by files protected with explicit NTFS ACLs for only the deploying Windows account and required
administrators. Include idempotent PowerShell commands using `Get-Acl` and `icacls` to set and verify
ownership, inheritance, and permissions without printing secret contents. Explain the effective
access seen inside the Linux container and avoid claiming that a container UID owns the host file.

The example must not contain usable credentials, generate a cursor-signing key inside the server,
or pass secrets on a command line. Explain outbound DNS and HTTPS requirements for the configured
BrightFlag origin and how to verify the fixed origin without logging its token.

## Operator procedure

Write a copy-and-pasteable guide that covers:

1. Windows, WSL 2, virtualization, Docker Desktop, Linux-container, PowerShell, and architecture
   prerequisites;
2. creating deployment directories outside synchronized folders and protecting them with NTFS ACLs;
3. obtaining and verifying the image digest and source revision;
4. preparing configuration, public JWKS, secrets, certificates, and permissions;
5. running preflight checks before the first start;
6. starting the stack and waiting for readiness;
7. testing TLS, network isolation, authentication, exact MCP surface, and a read-only request;
8. deliberately enabling and separately testing payment authorization without making a live
   payment;
9. connecting one LAN MCP client, using placeholders where client-specific syntax is unknown;
10. collecting redacted diagnostics;
11. upgrading by digest, running compatibility checks, and rolling back; and
12. stopping and removing the deployment, naming which state and secrets remain.

Every command must state whether it runs in non-elevated PowerShell, elevated PowerShell, WSL, or a
container. Use elevation only for narrowly scoped prerequisites, certificate-store changes,
firewall rules, ACL ownership when required, and Task Scheduler registration. Commands must be safe
to paste after placeholder substitution, avoid destructive wildcards, fail on unset values, and
preserve PowerShell's execution policy rather than weakening it. Separate server-health verification
from BrightFlag connectivity so readiness never causes a BrightFlag call.

## Automated validation

Add a test or validation target that, without contacting a live BrightFlag tenant:

- validates the Compose model and rejects unresolved placeholders;
- proves the effective container settings include every hardening control above;
- starts the stack with fake BrightFlag and local identity fixtures kept outside the delivered
  server image;
- verifies readiness, HTTPS access, and `/mcp` authentication from an allowed client;
- verifies direct backend access and a simulated disallowed client are refused;
- inspects Windows port bindings, firewall rules, certificate hostname and trust, Docker's
  Linux-container mode, and the documented reboot-start behavior;
- verifies an unauthenticated, expired, wrong-audience, and read-only payment request are refused;
- verifies the registered surface remains exactly four tools and one resource; and
- tears down only resources carrying the test project's unique name.

If a Windows firewall, router, DNS, certificate-trust, reboot, Task Scheduler, or remote-client step
cannot be automated safely, label it as a manual gate with an exact expected result. Do not silently
report skipped checks as passing.

## Acceptance criteria

- A learner can deploy one instance from a clean supported Windows host using only the checked-in
  guide, example files, and locally supplied secrets.
- The service is encrypted and accessible only to the explicitly allowed local-network clients.
- No private signing key or BrightFlag credential is present in the image, repository, Compose
  model, logs, or generated documentation.
- The local trust provider works only under an explicitly non-production profile, and the guide
  clearly distinguishes homelab investigation from a live organisational deployment.
- Restart preserves only the state this stage's own single dev instance requires.
- The guide honestly proves or qualifies recovery after Windows reboot and Docker Desktop restart.
- Upgrade and rollback use immutable image digests and retain auditable source revisions.
- Validation exercises the deployment without making a live BrightFlag call or payment.
- Formatting, build, tests, Compose validation, and the deployment validation target succeed.

Commit locally. Use `narrative-required` and record the supported baseline, network boundary,
identity-bootstrap risk, persistence choice, and deliberately unsupported variants. Do not push
unless requested.
