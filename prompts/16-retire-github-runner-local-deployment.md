# Prompt 16 — Retire the GitHub runner local-deployment path

Using the reusable contract and the artifacts produced by Prompts 1–11, reverse the local
deployment changes introduced by Prompts 14 and 15. Remove the self-hosted GitHub Actions runner
path from the BrightFlag server repository and document the corresponding operator-controlled
retirement on `ai-mcp-server`.

This is a corrective operational stage for repositories where Prompts 14 or 15 were applied. A fresh
implementation must skip Prompts 14–16 and apply Prompt 17 directly after Prompt 11. Keep the
GitHub-hosted continuous-integration jobs and verified GHCR image publication on protected `main`;
only deployment from GitHub into the local network is being retired.

## Repository boundary

- Delete `deploy-ai-mcp-server.yml` and every job scheduled with `runs-on: self-hosted` or the
  `ai-mcp-server` runner label.
- Remove the stable host-installed GitHub deployment script, its bootstrap/install instructions,
  runner-specific configuration, environment-dispatch examples, and tests whose only purpose was
  that path.
- Preserve image build, test, scan, verification, and digest-pinned publication on GitHub-hosted
  runners. Do not weaken branch protection, package permissions, repository privacy, or the
  immutable-image contract because deployment moved elsewhere.
- Do not remove generally useful Compose, preflight, validation, rollback, or image-verification
  code merely because Prompt 14 first referenced it. Prompt 17 will replace the deployment entry
  point.
- Mark the old runner and dedicated-nginx/8443 instructions as superseded. Do not leave two
  apparently supported local-deployment methods.

## Host and GitHub retirement runbook

Add an exact, manual retirement section to the deployment documentation. It must tell an operator
to inventory the runner name, labels, Windows service, installation directory, service account,
GitHub environment, and stable deployment files before changing anything. Then require this order:

1. stop the GitHub Actions runner service or process and prove that it accepts no new work;
2. remove the runner in the repository settings and run GitHub's generated removal command on the
   host;
3. prove the runner no longer appears in GitHub and no runner process or service remains;
4. remove the dedicated runner account from `docker-users`, and delete its installation directory
   only after the preceding evidence is recorded; and
5. after the workflow is gone, revoke or delete the `ai-mcp-server` GitHub environment and
   `BRIGHTFLAG_DEPLOYMENT_ROOT` configuration if nothing else owns them.

These are destructive external operations and remain operator-confirmed manual gates. Repository
automation must not remove accounts, services, directories, environments, or credentials.

Preserve Jamie's long-lived SSH access to `ai-mcp-server`. Do not remove or rotate Jamie's SSH key,
Windows account, administrator access, host SSH service, or any unrelated Docker access. Inventory
ambiguous resources and stop for operator confirmation rather than treating a shared resource as
runner-owned.

## Security and capability boundary

Removing the runner removes one Docker-capable remote execution path; it does not make the host,
GHCR package, LocalStack, Caddy, Keycloak, or BrightFlag credentials public. Keep the server
repository private. Keep payment disabled and do not contact a live BrightFlag tenant during the
change. Do not modify router, firewall, DNS, TLS, Caddy, Keycloak, or LocalStack state in this
stage.

## Prove

- no workflow, script, configuration, documentation, or test presents self-hosted GitHub Actions as
  a supported local-deployment path;
- no `runs-on: self-hosted`, `ai-mcp-server` runner label, deployment environment, or workflow
  dispatch can start a local deployment;
- protected-main CI still builds, tests, scans, verifies, and publishes the `linux/amd64` image to
  GHCR with its commit revision and immutable digest;
- generally useful local Compose, preflight, validation, rollback, and image-verification assets
  remain available for Prompt 17;
- the retirement sequence requires inventory and evidence before account, group-membership,
  directory, environment, or credential removal;
- Jamie's long-lived SSH access and every unrelated host service are explicitly preserved;
- the repository remains private, no secret enters Git or a log, and payment remains disabled; and
- all existing non-runner automated checks pass.

## Acceptance criteria

- Merging this stage makes it impossible for GitHub Actions in this repository to execute a local
  deployment or obtain Docker control on `ai-mcp-server`.
- A trusted operator can remove the retired runner without guessing which host or GitHub resources
  are dedicated to it, while shared access and services remain untouched.
- The verified GHCR publication flow remains suitable for the local pull deployment introduced by
  Prompt 17.
- Unrun host and GitHub retirement steps are labelled manual gates and never reported as complete.

Commit locally. Use `narrative-required` and record the removal of the Docker-capable GitHub trust
path, the preserved GitHub-hosted publication flow, and the explicit protection of Jamie's SSH
access. Do not push unless requested.
