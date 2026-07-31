# Stage 14 — Deliver reviewed images to the homelab with GitHub Actions

Source: `BrightFlagProxyMCPBuilder/prompts/14-github-runner-homelab-delivery.md`

## Context

An optional delivery extension for the existing Stage 10 deployment. GitHub-hosted CI builds and
publishes a verified image; a dedicated Windows runner on `ai-mcp-server` deploys its immutable
digest after approval. Stages 12 and 13 remain contingent and are not preconditions.

The server repository is currently public. A persistent runner with Docker access is therefore
refused until the repository is private. Docker access is equivalent to control over containers
that mount the deployment's secrets, so an environment approval alone is not sufficient isolation.

## Preconditions

- Stages 1–11 are merged and green.
- An operator has changed the server repository to private and protected `main`.
- Repository access and Actions approvals trust every person permitted to run workflow code on the
  runner; otherwise a separate private deployment-control repository is used.
- Stage 10 is manually bootstrapped in a stable, non-synchronised directory on `ai-mcp-server`.
- The `ai-mcp-server` GitHub environment permits protected `main` only. It has a required reviewer
  where the billing plan supports one; otherwise deployment is a separate main-only manual dispatch.

## Scope in

GHCR publication; a main-only deployment job; the Windows runner bootstrap and removal guide; a
stable host-installed deployment entry point; digest and revision validation; idempotent redeploy;
router-forwarded TLS exposure as an explicit optional mode; tests and operator documentation.

## Scope explicitly out

Changing repository visibility through automation; executing pull-request code on the host; storing
host secrets in GitHub; automatic payment enablement; automatic rollback; forwarding the server's
plain HTTP port; changing the MCP surface, BrightFlag operations, evidence rule, or payment gate;
implementing either contingent stage.

## Work items

### 1. Publish only a fully gated image

Extend CI so the container job depends on every existing gate. On a protected-main push only, log in
to GHCR with the job's `GITHUB_TOKEN`, push a commit-addressed tag, capture the registry digest, and
expose that digest to deployment. Keep PR permissions read-only and log out after publication.

### 2. Add the private, main-only deployment job

Target `[self-hosted, Windows, X64, ai-mcp-server]`, use the `ai-mcp-server` environment, serialise
deployments, and require `github.event.repository.private`. Prefer an environment reviewer. Where
the plan cannot provide one, put deployment in a separate `workflow_dispatch` workflow restricted to
`main`, requiring the published digest and expected revision as inputs. Do not check out the
repository. Invoke only the deployment script already present under the configured stable root.

### 3. Add the stable deployment entry point

Accept a digest and expected revision. Validate both, pull the digest, inspect architecture and OCI
revision, call the existing Stage 10 upgrade path with payment disabled, wait for readiness, and
verify the running container's image. Treat the already-current digest as a successful readiness
check. Never read or print secret contents.

### 4. Document runner bootstrap and removal

Use GitHub's current, repository-generated Windows commands so no expiring registration token is
checked in. Install outside synchronised folders under a dedicated account. State Docker privilege,
Docker Desktop sign-in/restart assumptions, outbound GitHub/GHCR requirements, update behaviour,
service inspection, disabling, and removal.

### 5. Add the opt-in router-forwarded mode

Preserve LAN allow-list mode. Add an explicit exposure setting and confirmation for any-source
firewall/proxy configuration. Document public DNS and certificate requirements and an exact router
mapping from an external TLS port to the proxy TLS port only. The runner never changes the router.

### 6. Correct the delivery and homelab documentation

Replace the unconditional no-internet claim with two supported modes and state which is active.
Document the protected environment, immutable digest flow, first-deploy bootstrap, deployment log,
failure and rollback procedure, router isolation, and unchanged application authorization.

## Tests

Map one-to-one to the prompt's Prove list:

1. parse workflows and assert no PR-reachable job uses `self-hosted`;
2. assert publish and deploy dependency edges include every existing gate;
3. assert private-repository, protected-main, and environment conditions, plus manual dispatch when
   environment reviewers are unavailable;
4. publish-script tests reject tags, wrong architecture, and revision mismatch;
5. assert the deployment job contains no checkout and calls the stable path;
6. assert permissions are job-scoped and logout uses an always condition;
7. test the same-digest path as a no-change readiness check;
8. test missing root, unresolved values, and mismatched artifact failures before Compose mutation;
9. retain the no-backend-port and exact-surface suites;
10. assert deployment never selects the payment overlay or invokes a BrightFlag operation;
11. retain the LAN fail-closed allow-list tests; and
12. make router enable/disable and an external client exact Windows/manual gates.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
./scripts/build-image.sh && ./scripts/verify-image.sh
```

Record the Windows runner, GHCR pull, protected-environment approval, public external-client, and
router-disable checks as manual gates until they have run.

## Stage boundary

Commit locally. Suggested message: `Deploy reviewed images to ai-mcp-server from GitHub`.

Use `narrative-required` when published. Do not push unless requested. Do not begin Stages 12 or 13.

## Risks

- A public repository runner turns a pull request into host code execution. The private-repository
  prerequisite is load-bearing.
- Docker access lets the runner control secret-mounting containers. A dedicated account limits
  ordinary file access, not this Docker capability.
- Executing the checked-out deployment script would undo the trust split; the host-installed copy is
  deliberate.
- A router forward can silently persist after testing. The external refusal test after disabling it
  is part of the deployment record.
- Automatic rollback can hide an indeterminate transition. Failure stops and names the reviewed
  rollback command instead.
