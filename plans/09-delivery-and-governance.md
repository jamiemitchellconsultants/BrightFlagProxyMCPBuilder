# Stage 09 — Package, document, and govern the server

Source: `BrightFlagProxyMCPBuilder/prompts/09-delivery-documentation-and-audit.md`

## Context

Repeatable delivery for the completed surface. Everything here is about making the finished server
reproducible, inspectable, and operable by someone who was not present while it was built — and
about writing down honestly what it does *not* do.

Stage 10 consumes this stage's image; Stage 11 audits this stage's documentation against the
implementation. So aspirational documentation here becomes an audit finding there.

## Preconditions

Stage 8 committed: both transports serving the narrow surface, topology declared and enforced.

## Scope in

Container image; CI; configuration and secret documentation; product documentation; security policy;
threat model; runbook.

## Scope explicitly out

Deployment artifacts for a specific host (Stage 10). New capabilities of any kind.

## Work items

### 1. Container image

- Built from a **pinned, digest-referenced** base.
- Runs as a **non-root** user with a **read-only root filesystem**, no build toolchain in the final
  layer.
- Exposes only the HTTP transport port.
- **Excludes** Stage 8's dev token-issuing tool and local JWKS trust-provider fixtures — and
  **proves their absence**, not merely their inactivity. Stage 3 put the fake BrightFlag server in
  the test project and Stage 8 put the dev token tool outside the server project so that absence is
  structural; the proof here is an image-content assertion, e.g. listing the published output and
  asserting no matching assembly or fixture file exists.
- Reproducible build, recording the source revision.
- Emits an SBOM and **fails the build on known-critical vulnerabilities**.
- Health endpoint reporting readiness **without contacting BrightFlag** and without revealing the
  origin, tenant, or allow-list.

### 2. Continuous integration

Workflows running: `restore --locked-mode`; format verification; build with warnings as errors;
tests; the **`schema check` drift gate from Stage 7**; dependency and secret scanning; container
build. Live BrightFlag sandbox tests stay behind an explicit opt-in that is **off** in normal CI.
Actions pinned by commit SHA, least-privilege permissions.

### 3. Configuration and secrets

Document every configuration key: default, bound, and **whether changing it is decision-bearing**.
Ship an example configuration using placeholders only. Add a startup validation pass that fails on
an unknown key, a missing allow-list, a non-HTTPS non-loopback origin, or a payment tolerance
outside its permitted range.

### 4. Documentation

State:

- the three capabilities and the four BrightFlag operations, **with the excluded operations named**;
- the definitions of *approved for payment* and *paid*, with their evidence requirements;
- the plan-and-confirm payment lifecycle and what an ambiguous outcome means;
- what an operator must do when a payment result is unknown;
- how the ontology schema is consumed by a separate ontology service;
- the tenant-configuration inputs an integrator must supply — the allow-list first among them;
- operational limits: lookback, summary-window margin, fan-out, page, byte, result, and rate
  ceilings;
- how to point the caller-identity trust provider at a real identity provider when moving from local
  development to a live deployment, and what happens under either profile if that step is skipped.

Record honestly, because Stage 11 will look for them: the **retention bound** on the payment record
store (the already-paid check is only as strong as its window), the deferred corporate MCP
authorization alignment from Stage 8, and that snapshot isolation is not claimed for pagination.

### 5. Security policy and threat model

Security policy: credential handling, the untrusted-content boundary, reporting.

Threat model naming **at least**: a prompt-injected payment instruction; a replayed plan token; a
duplicated payment; a widened allow-list; a leaked service token; a compromised cursor-signing key;
unauthorized read access to the plan store; a local development identity-trust provider or dev-token
tool reachable in a production deployment; an ontology schema carrying live data. Specific to this
server — generic advice is an audit finding.

### 6. Runbook

Procedures for: rotating the BrightFlag token; rotating the cursor-signing key; refreshing the
OpenAPI snapshot and reviewing its diff; changing the approved-status allow-list; responding to a
duplicate payment; switching the trust provider from local to live; rolling back a release. Each
names **who approves it** and **what evidence is retained**.

The cursor-signing key rotation entry must cover, specifically:

- generating a new key under a **new key identifier**;
- deploying it so verification accepts **both** the new and the immediately prior identifier, while
  new cursors are only ever signed under the new one;
- retiring the prior identifier from verification **no sooner than the maximum cursor lifetime**
  after cutover, so no cursor issued under it is still outstanding;
- the alternative **suspected-compromise** procedure, where the prior identifier is retired
  immediately and every caller with a cursor in flight receives a clean rejection and restarts
  pagination from a fresh window — expected, tested behaviour, not an incident.

This is what Stage 1's key-identifier indirection was built for.

## Acceptance checks

- The container builds, starts, serves `/mcp`, and passes health checks **with no BrightFlag call**.
- The dev token tool and local JWKS fixtures are verifiably absent from the image.
- CI runs every gate above and is green.
- Documentation matches the implemented surface exactly, with no aspirational capability.
- Threat model and runbook are specific to this server.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
docker build .
```

```bash
dotnet list package --vulnerable --include-transitive
```

## Stage boundary

Commit locally. Suggested message: `Package, document, and govern the server`.

`narrative-required` when published, recording the delivery and governance decisions — pinned
digest base, absence-proof for dev tooling, the CI gate set, and the rotation procedures.

Do not push unless requested. **Do not begin Stage 10.**

## Risks

- Docker 29.4.3 is present locally, so the image builds here. It will be *deployed* only to Windows
  (Stage 10) — build for `linux/amd64` explicitly rather than inheriting the build host's
  architecture, or Stage 10's Windows/WSL 2 host will refuse the image.
- "Prove absence" is easy to fake with a grep over the Dockerfile. Assert against the built image's
  actual contents.
- A read-only root filesystem breaks anything that writes temp files; provision a small writable
  `tmpfs` now rather than discovering it in Stage 10.
