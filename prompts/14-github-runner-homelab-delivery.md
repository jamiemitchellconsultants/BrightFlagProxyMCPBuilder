# Prompt 14 — Deliver reviewed images to the homelab with GitHub Actions

Using the reusable contract and the artifacts produced by Prompts 1–11, implement Stage 14: build
and publish the reviewed `linux/amd64` image in GitHub Actions, then deploy that exact image
digest to the existing single-instance homelab stack on `ai-mcp-server` with a dedicated
self-hosted runner.

This is an optional operational extension. It does not depend on the contingent Stages 12 or 13 and
does not make either one applicable. It replaces neither Stage 10's single-instance choice nor Stage
11's audit; it changes how an already-reviewed image reaches that one host.

## Security prerequisite

The server repository is public at the time this prompt was written. Do not attach a persistent
self-hosted runner with Docker access to a public repository. GitHub warns that pull-request
workflow code can persistently compromise such a runner. On this host, Docker control can reach
containers that mount the BrightFlag credential, cursor-signing key, and TLS private key, so the
impact is not limited to a disposable build directory.

Require the server repository to be private before registering or enabling the runner. Make the
deployment job fail closed or remain unscheduled while `github.event.repository.private` is false.
Record the visibility change separately; this prompt does not authorise changing repository
visibility through an API. Review repository access and Actions approval policy: every person able
to run workflow code must be trusted to execute code on the host. If that cannot be guaranteed, use
a separate private deployment-control repository with only trusted workflow authors or do not
install the runner.

## Build and publication boundary

- Pull-request jobs continue to use GitHub-hosted runners and read-only permissions.
- Only a successful push to protected `main`, after formatting, build, tests, schema drift,
  dependency, secret-scan, and image verification gates, may publish an image.
- Publish to GHCR with a commit-addressed tag, then derive and pass the registry digest. The deploy
  job consumes `ghcr.io/...@sha256:...`, never the tag.
- Grant `packages: write` only to the publishing job and `packages: read` only to deployment. Keep
  `contents: read`. Log out of GHCR even when deployment fails.
- Preserve the image revision label and prove it equals the GitHub commit being deployed.

## Runner boundary

Register a repository-scoped Windows x64 runner on `ai-mcp-server` with the additional label
`ai-mcp-server`. It runs under a dedicated local account, outside every synchronised directory, and
has no repository, organisation, or personal access token stored on disk. The short-lived
registration token is never checked in or copied into a prompt.

Docker access is privileged access. State that membership of `docker-users` lets the runner control
containers and can expose mounted host secrets. Keep the runner repository-scoped, schedule only the
one main-branch deployment job to its labels, and use an `ai-mcp-server` GitHub environment. Prefer
a required reviewer before the job starts. If the repository's billing plan does not support
required reviewers for a private repository, use a separate `workflow_dispatch` deployment whose
inputs require the published digest and expected revision. Restrict it to `main`, and record that
the manual dispatch is the approval. Never silently fall back to automatic deployment. A GitHub
environment or manual dispatch is an approval gate, not a sandbox.

The self-hosted job must not check out or execute scripts from the workflow revision. It invokes a
reviewed deployment script already installed in the stable deployment directory during the manual
bootstrap. The job supplies only the immutable image digest and expected source revision. Host
secrets, `.env`, certificates, private keys, and caller tokens remain outside the Actions workspace
and outside GitHub secrets.

## Deployment behaviour

- Serialise deployments; never cancel one halfway through to start a newer one.
- Pull and inspect the exact digest before changing the running stack. Require `linux/amd64` and the
  expected `org.opencontainers.image.revision` label.
- Reuse Stage 10's upgrade path so the outgoing digest and both source revisions remain in the
  deployment log. A rerun of the same digest is an idempotent readiness check, not a failure.
- Start with the payment overlay disabled. GitHub Actions must not enable the payment capability;
  that remains Stage 10's separate, deliberate operator action.
- Wait for readiness without contacting BrightFlag. Verify the running container uses the requested
  digest. On failure, leave exact rollback instructions but do not automatically roll back an
  ambiguous or partially completed deployment.

## Optional router-forwarded exposure

Stage 10 forbade router forwarding. This stage deliberately permits one opt-in homelab exposure
mode: an external TCP port forwards to the proxy's TLS port on `ai-mcp-server`. It never forwards
the server's plain HTTP port. Public DNS and the certificate must name the public hostname.

The router rule is controlled manually and is never created, enabled, disabled, or inspected by the
GitHub runner. Turning it off is an isolation control and an emergency cut-off; it is not caller
authentication or authorization. JWT validation, issuer and audience checks, capability roles, the
payment overlay, TLS, container hardening, and the unpublished backend remain unchanged when the
forward is on.

Keep LAN allow-list mode supported. If router-forwarded mode accepts changing client source
addresses, require an explicit exposure-mode setting plus a separate confirmation switch before
creating an `Any`-remote Defender Firewall rule or an allow-all proxy list. Never obtain public
exposure from an empty variable or a missing allow-list.

## Prove

- no self-hosted label appears in any pull-request job;
- publication and deployment are impossible until all existing gates pass;
- deployment is unscheduled while the repository is public and restricted to protected `main` when
  private;
- the published reference is digest-pinned and its revision label equals the triggering commit;
- the self-hosted job performs no checkout and invokes only the stable host-installed script;
- package permissions are job-scoped and registry credentials are removed after the job;
- a same-digest rerun succeeds without changing the recorded deployment;
- a wrong architecture, tag-only reference, digest/revision mismatch, unresolved placeholder, or
  missing stable deployment directory fails before the running stack changes;
- the deployed server still publishes no host port and exposes exactly four tools and one resource;
- the deployment path neither enables payment nor contacts a live BrightFlag tenant;
- LAN allow-list mode remains fail-closed;
- router-forwarded mode requires explicit confirmation, forwards only to TLS, succeeds from an
  external client when enabled, and refuses the same client when the router rule is disabled; and
- runner removal, GHCR logout, rollback, router isolation, and credential rotation are documented.

## Acceptance criteria

- A reviewed merge to private `main` can publish one verified image and, after environment approval,
  deploy that exact digest to the existing container on `ai-mcp-server`.
- Under the recorded repository-access and Actions-approval policy, untrusted pull-request code
  cannot be approved to run on the homelab runner. Where environment reviewers are unavailable, a
  merge publishes only and a separate main-only manual dispatch deploys.
- No host secret or private key enters GitHub, the Actions workspace, an image layer, or a log.
- Router forwarding changes reachability only; it does not change the MCP or BrightFlag capability
  surface and is never described as an authorization boundary.
- Formatting, build, tests, schema drift, image verification, Compose validation, and the deployment
  workflow's static tests succeed. Windows runner and router checks are manual gates until recorded.

Commit locally. Use `narrative-required` and record the public-repository runner risk, the private
repository prerequisite, the build-versus-deploy trust split, Docker's privilege boundary, the
router's limited role, and the fact that GitHub Actions cannot enable payment. Do not push unless
requested.
