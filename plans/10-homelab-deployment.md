# Stage 10 — Generate a homelab and local-network deployment guide

Source: `BrightFlagProxyMCPBuilder/prompts/10-homelab-local-network-deployment.md`

## Context

An operator-ready deployment path for **one** MCP server instance on a trusted home-lab or
local-network Windows host. This stage documents and automates deployment; it must not widen the MCP
or BrightFlag surface by a single tool, operation, or argument.

Two framings that govern everything below:

- **This is a dev deployment.** It runs one instance for functional investigation. It does **not**
  narrow the live topology Stage 8 declared. If Stage 8 declared multi-instance for live, this
  deployment simply exercises one of them, and the documentation must say so rather than implying
  the live topology is proven here.
- **The container only ever runs on Windows.** The Windows 11 / Docker Desktop / WSL 2 /
  PowerShell 7 baseline is the real target, not a portability compromise. The authoring machine
  being a Mac changes what can be *executed during the build*, not what gets written.

## Preconditions

Stage 9 committed: image built from a pinned digest base, `linux/amd64`, dev tooling provably
absent, health endpoint working without a BrightFlag call.

## Scope in

Compose model; PowerShell preflight/validate/start/stop/upgrade/rollback scripts; network boundary
and TLS; local caller-identity bootstrap; secret placement with NTFS ACLs; the operator procedure;
automated validation.

## Scope explicitly out

Any change to the MCP surface, the four operations, the evidence rule, or the payment gate. Public
internet exposure. Router port forwarding. Any claim that a LAN boundary substitutes for the live
identity-provider migration.

## Work items

### 1. Supported baseline — state one, precisely

Windows 11 host, Docker Desktop with the **WSL 2 Linux-container backend** and bundled Compose
plugin; one server container from the Stage 9 image; the single dev instance; Streamable HTTP at
`/mcp` reachable only from explicitly allowed LAN clients.

PowerShell 7 for host commands; every command labelled as to whether it runs in **non-elevated
PowerShell, elevated PowerShell, WSL, or the container**. Equivalent Windows Server, Hyper-V,
Podman, or NAS steps may be *noted* but never claimed supported unless tested.

Record: tested Windows edition and build, CPU architecture, WSL version, Docker Desktop and engine
versions, Compose version, container mode, client environment. Placeholders for hostnames,
addresses, tenant values, credentials.

**Preflight must detect and fail if Docker is in Windows-container mode** rather than Linux.

State Docker Desktop's licensing, interactive-logon, automatic-start, update, and availability
assumptions explicitly. Do **not** describe a container restart policy as proof the stack starts
after a Windows reboot — provide and test a least-privilege Task Scheduler startup procedure if
unattended reboot recovery is claimed, otherwise document operator sign-in plus Docker Desktop
startup as a manual availability gate.

### 2. Deployment artifacts — `deploy/homelab/`

Compose file plus checked-in PowerShell scripts: preflight, validate, start, stop, upgrade,
rollback. Strict mode, stop on errors, reject unresolved placeholders, no machine-wide policy
changes.

**Pin the image by immutable digest** or require the operator to supply one. Never `latest`.

Container settings: non-root user; read-only root filesystem; dropped Linux capabilities;
`no-new-privileges`; bounded CPU and memory; small writable `tmpfs`; restart policy; health check
against Stage 9's readiness endpoint; explicit port binding **preferably to a dedicated LAN address
rather than every interface**; a persistent access-controlled plan-store volume **only if the chosen
implementation requires one**; log rotation to a documented location or driver that captures no
secrets; fail-closed placeholder detection on all required configuration.

Forbidden, each a validation assertion: mounting the container engine socket; host networking;
privileged mode; embedded credentials; plaintext secrets in Compose; an auto-downloaded deployment
script; the Stage 8 dev token executable or private signing key inside the image.

Windows path handling deliberately: keep deployment files **outside OneDrive and other synchronised
folders**; quote paths containing spaces; avoid drive sharing that predates the WSL 2 backend;
document Docker Desktop file-sharing behaviour for every bind mount.

### 3. Network boundary and TLS

Document the concrete layout and the full request path from an allowed MCP client to `/mcp`,
accounting for the Windows host, Docker Desktop's WSL 2 virtual network, published container ports,
and any reverse-proxy container.

Bind **only the TLS entry point** to the intended Windows LAN address. Bind the unencrypted
application port to loopback or an internal Compose network so it is **never** a LAN listener.

Idempotent PowerShell using Windows Defender Firewall to create narrowly named inbound rules: TCP
port, `LocalAddress`, `RemoteAddress` clients or VLAN, profile, program/service where meaningful,
default-deny. Include equally exact inspection and removal commands affecting **only** those rules.

HTTPS beyond loopback. Choose one documented pattern: a reverse proxy on the same host with the
server port reachable only from it; or a trusted private overlay providing authenticated encryption
and restricted membership. For the reverse proxy: minimal pinned configuration, certificate
provisioning or private-CA trust steps, correct forwarding for Streamable HTTP, request and idle
timeouts, body-size limits aligned with Stage 8, and a test proving the unencrypted backend port is
unreachable from another LAN machine.

Windows certificate-store terminology and PowerShell certificate inspection. If a private CA is
used, install **only its public root** into the appropriate Trusted Root store on each authorised
client — never the CA private key. Explain hostname/SAN matching; prefer a stable local DNS name
over a raw address. Never disable certificate validation in example clients.

### 4. Local caller-identity bootstrap

For a deployment explicitly marked **non-production**, document how an operator:

1. generates the local signing key and public JWKS on an administrator-controlled machine;
2. supplies **only the public JWKS** and the expected issuer, audience, and required-claim
   configuration to the server;
3. runs Stage 8's dev token-issuing tool **outside** the server container;
4. issues a short-lived token for a dummy caller with the minimum **read-only** capability first;
5. configures the LAN MCP client to send that bearer token;
6. proves expired, wrong-audience, and unauthorized tokens are rejected.

Private signing material never enters the server image, Compose file, repository, logs, PowerShell
history, or client configuration. State plainly that **anyone holding the local signing key can
impersonate any configured caller**. Require restrictive NTFS ACLs, backup and rotation guidance,
and a **separate deliberate enablement step** for the payment capability.

Do not rely on Unix mode bits on a Windows bind mount as the host-side access-control boundary.

State prominently that the local trust provider is for non-production functional investigation only,
preserve Stage 8's startup failure outside a non-production profile, and link to Stage 9's runbook
for migrating to the organisation's live identity provider. A LAN boundary, reverse proxy, VPN, or
private CA is **not** a substitute for that migration.

### 5. BrightFlag and application secrets

Provide the BrightFlag bearer token, cursor-signing key, and any plan-store credential through Stage
3's secret-provider interface. Prefer **file references** backed by files with explicit NTFS ACLs
for only the deploying Windows account and required administrators. Idempotent PowerShell using
`Get-Acl` and `icacls` to set and verify ownership, inheritance, and permissions **without printing
secret contents**. Explain the effective access seen inside the Linux container; do not claim a
container UID owns the host file.

No usable credentials in examples. Never generate the cursor-signing key inside the server. Never
pass secrets on a command line. Explain outbound DNS and HTTPS requirements for the configured
BrightFlag origin and how to verify the fixed origin without logging its token.

### 6. Operator procedure

Copy-and-pasteable, covering the twelve steps the prompt enumerates: prerequisites; deployment
directories outside synchronised folders with NTFS ACLs; obtaining and verifying the image digest
and source revision; preparing configuration, public JWKS, secrets, certificates, permissions;
preflight; start and wait for readiness; testing TLS, network isolation, authentication, the exact
MCP surface, and a read-only request; **deliberately enabling and separately testing payment
authorization without making a live payment**; connecting one LAN MCP client; collecting redacted
diagnostics; upgrading by digest with compatibility checks and rollback; stopping and removing,
naming which state and secrets remain.

Elevation only for narrowly scoped prerequisites, certificate-store changes, firewall rules, ACL
ownership where required, and Task Scheduler registration. Commands must be safe to paste after
placeholder substitution, avoid destructive wildcards, fail on unset values, and **preserve
PowerShell's execution policy rather than weakening it**. Keep server-health verification separate
from BrightFlag connectivity so readiness never causes a BrightFlag call.

### 7. Automated validation

A validation target that, **without contacting a live BrightFlag tenant**: validates the Compose
model and rejects unresolved placeholders; proves the effective container settings include every
hardening control; starts the stack with fake BrightFlag and local identity fixtures kept outside
the delivered server image; verifies readiness, HTTPS access, and `/mcp` authentication from an
allowed client; verifies direct backend access and a simulated disallowed client are refused;
inspects Windows port bindings, firewall rules, certificate hostname and trust, Docker's
Linux-container mode, and documented reboot-start behaviour; verifies unauthenticated, expired,
wrong-audience, and read-only-payment requests are refused; verifies the surface is still exactly
four tools and one resource; tears down **only** resources carrying the test project's unique name.

## Executing this stage from a Mac

The prompt's own escape clause governs: anything that cannot be automated safely is labelled a
**manual gate with an exact expected result**, and skipped checks are never reported as passing.

Runnable while authoring, on any host:

```bash
docker compose -f deploy/homelab/compose.yaml config
```

plus the image build and hardening inspection, the fake-BrightFlag stack, readiness, `/mcp`
authentication, token rejection cases, and the exact-surface check.

**Manual gates, to be executed on the Windows host with recorded evidence:** Defender Firewall rule
creation, inspection, and removal; NTFS ACL application and verification via `Get-Acl` / `icacls`;
certificate-store installation and trust validation; Windows port-binding inspection; Docker Desktop
Linux-container-mode detection; reboot and Docker Desktop restart recovery; Task Scheduler
registration; the disallowed-LAN-client refusal test, which needs a second machine on the network.

Each gate is written with the exact command, the exact expected output, and a place to record the
observed result — so the Windows run is a verification pass, not a re-derivation.

## Acceptance checks

- A learner can deploy one instance from a clean supported Windows host using only the checked-in
  guide, example files, and locally supplied secrets.
- The service is encrypted and reachable only by explicitly allowed local-network clients.
- No private signing key or BrightFlag credential in the image, repository, Compose model, logs, or
  generated documentation.
- The local trust provider works only under an explicitly non-production profile, and the guide
  clearly distinguishes homelab investigation from a live organisational deployment.
- Restart preserves only the state this single dev instance requires.
- Reboot and Docker Desktop restart recovery is honestly proven **or** qualified.
- Upgrade and rollback use immutable digests and retain auditable source revisions.
- Validation exercises the deployment with **no live BrightFlag call and no payment**.

## Stage boundary

Commit locally. Suggested message: `Add Windows homelab deployment guide and validation`.

`narrative-required` when published, recording: the supported baseline; the network boundary; the
identity-bootstrap risk (local signing key = impersonation of any caller); the persistence choice;
and the deliberately unsupported variants. Record explicitly that this is a **dev** deployment that
does not narrow Stage 8's live topology.

Do not push unless requested. **Do not begin Stage 11.**

## Risks

- Building the Stage 9 image on an Apple-silicon Mac produces `linux/arm64` unless told otherwise;
  the Windows/WSL 2 target needs `linux/amd64`. Pin the platform in Stage 9 and verify the digest's
  architecture here.
- The unencrypted backend port leaking onto the LAN is the most likely real defect. Test it from a
  second machine — that test is a manual gate and must not be quietly dropped.
- The reboot-recovery claim is the easiest thing to overstate. If Task Scheduler is not tested, the
  guide says operator sign-in is required, full stop.
- Line endings: the Stage 7 drift gate will fail on a Windows clone without the `.gitattributes`
  rule pinning the checked-in schema document to `LF`.
