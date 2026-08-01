# Stage 16 — Retire the GitHub runner local-deployment path

Source: `BrightFlagProxyMCPBuilder/prompts/16-retire-github-runner-local-deployment.md`

## Context

Stages 14 and 15 introduced a Docker-capable self-hosted runner and a dedicated nginx listener for
local deployment. The selected replacement keeps reviewed GHCR publication but moves image pull and
deployment to an operator-run script on the destination host. This stage removes the obsolete
remote-execution path before the replacement is added.

## Preconditions

- Stages 1–11 are merged and green.
- Stage 14 or 15 was applied; otherwise skip this stage and apply Stage 17 after Stage 11.
- The current runner, GitHub environment, host service, account, group membership, installation
  directory, and stable scripts can be inventoried without changing them.
- Jamie's long-lived SSH access can be distinguished from the dedicated runner identity.

## Scope in

Runner-deployment workflow removal; runner-only script, configuration, test, and documentation
removal; preserved GitHub-hosted GHCR publication; manual runner-retirement instructions; explicit
preservation of SSH and unrelated host services.

## Scope explicitly out

Performing destructive GitHub or host retirement automatically; changing repository visibility;
changing Caddy, Keycloak, LocalStack, DNS, TLS, router, firewall, or BrightFlag settings; enabling
payment; adding the replacement deployment path; beginning contingent Stages 12 or 13.

## Work items

### 1. Remove the GitHub deployment entry point

Delete the self-hosted deployment workflow and runner scheduling. Remove runner-only bootstrap,
host-script, environment-dispatch, configuration, and test assets without deleting reusable local
deployment validation.

### 2. Preserve verified publication

Keep protected-main GitHub-hosted build, test, scan, image verification, GHCR publication, immutable
digest, `linux/amd64`, and OCI revision guarantees. Keep the repository private and permissions
least-privileged.

### 3. Replace obsolete documentation

Mark the Prompt 14/15 runner and dedicated-nginx/8443 path as superseded. Add an inventory-first,
evidence-bearing retirement sequence and make every external destructive operation a manual gate.

### 4. Protect unrelated access and services

Name Jamie's Windows account, long-lived SSH access, and unrelated containers as preserved. Require
operator confirmation when ownership is ambiguous. Keep payment disabled and avoid live tenant
traffic.

## Tests

Map one-to-one to the prompt's Prove list:

1. search workflows, scripts, configuration, documentation, and tests for a supported self-hosted
   deployment path;
2. assert no self-hosted label, runner label, deployment environment, or workflow dispatch can
   deploy locally;
3. run or inspect protected-main CI and verify digest-pinned `linux/amd64` GHCR publication with the
   commit revision;
4. inventory retained Compose, preflight, validation, rollback, and image-verification assets;
5. review retirement instructions for inventory and evidence before every destructive operation;
6. assert the runbook protects Jamie's SSH access and unrelated services explicitly;
7. run the secret scan, inspect privacy and payment defaults, and confirm no live tenant call; and
8. run all existing checks that are not specific to the removed runner path.

## Acceptance checks

```bash
rg -n "runs-on:.*self-hosted|ai-mcp-server.*runner|deploy-ai-mcp-server" .github deploy docs
```

The command must find only clearly labelled historical or retirement text, never an executable or
supported deployment path.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

```bash
npx --yes --package=github:jamiemitchellconsultants/Narrative narrative check
```

Run the documented GitHub and Windows retirement checks on the intended systems. Until an operator
records their results, they are manual gates rather than passing acceptance checks.

## Stage boundary

Commit locally. Suggested message: `Retire the self-hosted deployment runner`.

Use `narrative-required` when published. Do not push unless requested. Apply Stage 17 next; do not
enable payment or begin contingent Stages 12 or 13.

## Risks

- Removing a shared account, key, directory, or group membership could break administrator access;
  inventory ownership and preserve Jamie's SSH path explicitly.
- Deleting reusable validation with runner-only code would weaken the replacement deployment.
- Leaving a dispatch or misleading runbook behind creates two apparent deployment authorities.
- Removing image publication would force unreviewed local builds and break the selected trust split.
