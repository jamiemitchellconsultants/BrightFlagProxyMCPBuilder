# Stage 15 — Prepare the local deployment runbook

Source: `BrightFlagProxyMCPBuilder/prompts/15-deploy-local-runbook.md`

## Context

Prompt 14 established reviewed image publication and a private, main-only runner path, but GitHub
Actions intentionally cannot manufacture the host's trust material or tenant decisions. The
stable root on `ai-mcp-server` therefore still needs a one-time bootstrap before the first digest
can run and before an authenticated OpenCode client can connect.

The host already runs an unrelated Caddy service on ports 80 and 443. The BrightFlag stack retains
its reviewed nginx proxy on port 8443 rather than folding a financial capability into that existing
edge configuration without a separate design review.

## Preconditions

- Stages 1–11 and 14 are merged and green.
- The server repository is private and protected `main` publishes a verified GHCR image.
- The repository-scoped `ai-mcp-server` runner and stable host deployment script are installed.
- An operator can supply the BrightFlag tenant facts, service credential, public DNS, TLS material,
  and local caller trust material without putting them in Git or a prompt.
- Existing host services, port ownership, Docker mode, addresses, and deployment files are inspected
  read-only before being described as current.

## Scope in

One site-specific `docs/deploy-local.md`; a README link; observed-state commands; the stable host
file layout; router-forwarded DNS and TLS preparation; local caller identity; preflight; immutable
image selection and workflow dispatch; validation; OpenCode handoff values; recovery and repeat
deployment.

## Scope explicitly out

Creating or storing real secrets; changing router state through automation; modifying Caddy;
changing Compose, deployment workflows, application code, the MCP surface, BrightFlag operations,
identity validation, or payment controls; enabling payment; implementing a live identity provider;
claiming that any unrun Windows, DNS, TLS, BrightFlag, client, or external-router check passed.

## Work items

### 1. Re-observe the starting state

Inspect repository visibility, protected-main CI, the runner, stable script, Docker mode, address
and port ownership, Caddy, and required bootstrap files. Record commands, date, and results
separately from prerequisites and expected results.

### 2. Inventory operator inputs and host files

Name the BrightFlag origin, tenant, exact approved status, service token, public hostname,
certificate, private key, and caller roles without inventing them. Map `.env` and each secret,
JWKS, TLS, and proxy file to its exact stable-host path.

### 3. Document the selected reachability path

Use nginx on `192.168.50.6:8443`, preserve Caddy on 80/443, and document external 8443 forwarding
to the same internal port. Require split DNS or a host-local override so readiness reaches the
configured hostname with normal certificate validation. Keep router changes manual and port 8080
unreachable.

### 4. Bootstrap local identity and secrets

Document keypair creation, public-JWKS installation, tenant-bound read-token minting, negative
tokens, cursor-key generation, direct service-token placement, TLS files, ACLs, proxy image pull,
firewall confirmation, and a clean preflight. Keep every private value outside the runbook and
repository.

### 5. Select and dispatch the first artifact

Extract the final manifest digest and revision from the latest successful main publication, verify
the pair, record a dated candidate, and provide the exact main-only deployment dispatch. Explain
how to refresh the candidate instead of presenting it as a moving latest reference.

### 6. Validate and hand off

Run the existing validation and manual gates without making a payment. End with only the MCP URL,
bearer-header requirement, and certificate trust needed by an OpenCode client. State token expiry,
recovery, repeat-deploy, rollback, rotation, runner-removal, and router-isolation procedures.

## Tests

Map one-to-one to the prompt's Prove list:

1. review the runbook tables and wording for the three evidence classes;
2. run the repository secret scanner and inspect the diff for private or tenant material;
3. assert the runbook preserves Caddy and reserves 80/443;
4. assert every router example targets TLS port 8443 and no example forwards 8080;
5. assert the host-side readiness prerequisites name DNS and normal certificate validation;
6. assert the pinned proxy-image pull precedes first workflow deployment;
7. match the candidate image against the final publish manifest digest and OCI revision;
8. inspect token commands for key id, subject, tenant, read role, and off-host private key;
9. inspect preflight wording for the exact 0, 3, and 1 meanings;
10. assert no command selects `compose.payment.yaml`, `-EnablePaymentCapability`, or the payment
    tool;
11. inspect the final handoff for URL, bearer header, and certificate trust only; and
12. retain the external enabled/disabled router gate with connection failure as the disabled result.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

```bash
./scripts/build-image.sh && ./scripts/verify-image.sh && ./scripts/validate-homelab.sh
```

Run the Windows preflight, validation, second-machine, and external-router commands exactly as the
runbook names them. Until recorded on the intended host and networks, they are manual gates, not
passing acceptance checks.

## Stage boundary

Commit locally. Suggested message: `Document first deployment to ai-mcp-server`.

Use `narrative-required` when published. Do not push unless requested. Do not enable payment or
begin the contingent Stages 12 or 13.

## Risks

- A stale first-deploy digest can look authoritative. Date it, pair it with its revision, and teach
  how to refresh it from the final publish result.
- Public DNS that does not resolve back to the host can make the service healthy while the
  workflow's validated-TLS readiness check times out.
- Reusing ports 80/443 or modifying Caddy couples two deployments and exceeds this
  documentation-only stage.
- A long-lived development bearer reduces token-renewal friction by extending the theft window. Keep
  the local-provider limit visible and migrate rather than hiding expiry.
- Router shut-off removes reachability, not credentials. Authentication and authorization remain
  load-bearing whenever the forward is enabled.
