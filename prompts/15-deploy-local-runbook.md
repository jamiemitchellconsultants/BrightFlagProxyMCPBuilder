# Prompt 15 — Prepare the local deployment runbook

Using the reusable contract and the artifacts produced by Prompts 1–11 and 14, implement Stage 15:
write the site-specific first-deployment and client-handoff runbook for `ai-mcp-server`.

This is an optional operational extension after Prompt 14. It documents how to bootstrap the stable
host directory, deploy one already-verified image through GitHub Actions, validate the running
service, and hand its authenticated Streamable HTTP endpoint to one OpenCode client. It does not
change the MCP surface, build pipeline, deployment workflow, container model, identity checks,
BrightFlag operations, payment controls, or router state.

## Establish facts before writing

Inspect the current server repository, the latest successful protected-`main` image-publication run,
and the host through an existing administrator-approved connection when one is available. Separate
the runbook into:

- **observed state**, with the command and observation date;
- **operator inputs**, which cannot be inferred safely; and
- **expected results**, which remain unverified until the operator runs them.

Do not convert an assumption into an observed fact. If the host, GitHub Actions logs, GHCR package,
DNS, certificate, router, or BrightFlag tenant cannot be inspected, leave the corresponding item as
an explicit prerequisite or manual gate. Never describe an unrun gate as passing.

The intended site currently uses Windows 11, Docker Desktop in Linux-container mode, PowerShell 7,
the stable root `C:\BrightFlagMcp`, and the LAN address `192.168.50.6`. Port 8443 is the intended
BrightFlag proxy listener. An existing Caddy deployment owns host ports 80 and 443 and is outside
this stage: do not edit, replace, restart, or route BrightFlag through it. Re-observe these facts
when the prompt is replayed; document a difference instead of overwriting another service.

## Runbook artifact

Add `docs/deploy-local.md` and link it from the repository README documentation table. It must be
operator-ready and contain no credential, bearer token, private key, private JWKS member, `.env`
contents, or invented tenant value.

### 1. State what is already prepared

Record the repository privacy and protected-main boundary, runner labels and availability, stable
deployment script, Docker container mode, intended listener address and port, existing edge-service
port ownership, and whether `.env`, the proxy client policy, secret files, and TLS files exist.
Include read-only commands an operator can rerun to refresh each observation.

### 2. Name the inputs only an operator can supply

Require the exact BrightFlag HTTPS origin, tenant name, tenant-specific approved-for-payment status,
and a least-privilege BrightFlag service token. Require a public hostname, a publicly trusted
certificate and private key for that hostname, and the caller role names. Explain which items are
secret and must never be pasted into a prompt, command argument, shell history, Actions secret, log,
or repository.

Keep payment disabled for the first deployment. A read role may be named in the runbook; a payment
role remains a separate deliberate operator decision under Stage 10.

### 3. Give the exact stable-host file layout

List `.env`, the BrightFlag service-token file, cursor-signing-key file, public caller JWKS, TLS
certificate and key, nginx configuration, and proxy client policy under `C:\BrightFlagMcp`. Start
from the checked-in `.env.example`; identify every non-secret setting the operator must decide and
every file reference it retains. Never duplicate secret material into `.env`.

For the router-forwarded mode selected for this site, require an explicit
`MCP_EXPOSURE_MODE=RouterForwarded`, an `allow all;` followed by `deny all;` proxy policy, and the
matching explicit Defender Firewall confirmation. State why allow-all changes reachability only:
caller authentication, tenant checks, capability roles, and the disabled payment overlay remain.

### 4. Preserve edge-port ownership and define DNS

Use the BrightFlag proxy's dedicated host port 8443 so the existing Caddy service on 80/443 is
untouched. Recommend forwarding an external 8443 to `192.168.50.6:8443`; never forward 8080. If an
operator chooses a different external port, distinguish it from the internal proxy port explicitly.

Require the public hostname to resolve to the public router address externally and to the host's LAN
address from the host and local clients, using split DNS or a narrowly scoped hosts-file entry where
needed. Explain that deployment readiness calls the configured hostname over validated TLS, so DNS
and trust must work on the host before GitHub deployment can report success. Do not disable
certificate validation. Prefer an ACME DNS challenge when obtaining a certificate without disturbing
the existing 80/443 service.

The runner must never create, enable, disable, or inspect the router rule. The runbook may tell the
operator when to change it and how to prove public reachability disappears after it is disabled.

### 5. Bootstrap local caller identity without moving the private key

Give exact `BrightFlagMcp.DevTokens keypair` and `mint` commands. Keep the private signing key on an
administrator-controlled development machine and copy only the public JWKS to the host. Mint the
first token for one named subject, the configured tenant, and the minimum read role. Also mint
expired and wrong-audience tokens for validation.

State the token lifetime and its operational consequence for a static OpenCode header. A short-lived
development token proves the path; routine use needs renewal or migration to the organisation's
live identity provider. Never solve expiry by copying the private signing key to the server,
container, or client configuration.

### 6. Bootstrap and preflight before the workflow runs

Provide commands to generate the cursor-signing key without displaying it, install the public JWKS,
service credential, TLS material, proxy policy, and non-secret `.env`, pull the pinned proxy image,
apply and show NTFS ACLs, apply the router-forwarded Defender rule with its confirmation switch, and
run `Preflight.ps1`.

Explain exit codes exactly: 0 means every preflight check ran and passed; 3 means runnable checks
passed but at least one gate did not run; 1 means failure. Require 0 on the deployment host before
the first workflow dispatch.

### 7. Select and deploy an immutable server image

Do not hard-code an image tag or tell an operator to copy the first `sha256` string found in a log.
Give a GitHub CLI or UI procedure that starts from a successful protected-`main` CI run, selects the
`Publish verified image` job, records its commit revision, and extracts the final manifest digest
printed for the commit-addressed tag. Form the complete lowercase
`ghcr.io/...@sha256:<64-hex>` reference and verify its image revision label.

The runbook may contain a clearly dated **first-deploy candidate** block with the currently observed
digest and 40-character revision. Label it as an observation, not a moving alias, and require the
operator to refresh it after a later merge. Give the exact `gh workflow run` command for
`deploy-ai-mcp-server.yml` with `image_digest` and `source_revision`. The workflow, not an
interactive host login, performs the server-image pull and start.

State that the first deployment still needs the pinned proxy image pulled locally because Start uses
`--pull never`. State that the workflow deploys with payment disabled and verifies that fact.

### 8. Validate before registering a client

Run `Validate.ps1` with valid read, expired, and wrong-audience tokens. Require validated TLS,
anonymous 401, invalid-token 401s, exactly four tools and one resource, a successful
ontology-schema call, no LAN listener on 8080, and refusal of the read-only caller's payment plan.
Retain the second-machine and external-router checks from the manual gates, including successful
external access while forwarding is enabled and connection failure after it is disabled.

End with the client handoff values only: `https://<hostname>:8443/mcp`, an Authorization bearer
header, and normal certificate trust. Do not prescribe OpenCode syntax when the operator already
owns that configuration, and never write a live token into the runbook.

### 9. Include recovery and repeat-deployment notes

Name Docker Desktop's signed-in availability requirement, the health and diagnostics commands, the
upgrade log, manual rollback, token renewal, certificate renewal, BrightFlag credential rotation,
runner disable/removal, and router shut-off. A deployment failure must not automatically roll back
or enable payment.

## Prove

- the runbook distinguishes observed state, operator input, and expected but unrun results;
- no secret, live token, private key, private JWKS member, or usable tenant value is present;
- the existing Caddy deployment and host ports 80/443 are explicitly preserved;
- only the TLS proxy port is forwarded and port 8080 remains unpublished and unforwarded;
- DNS and certificate validation work from the host before deployment readiness is claimed;
- the first deployment pulls the pinned proxy image before a `--pull never` start;
- the image reference comes from the final published manifest digest and is paired with its exact
  source revision;
- caller tokens include the configured tenant and read role, while private signing material remains
  off the host;
- preflight exit 3 is never reported as a pass;
- GitHub deployment cannot enable payment and validation makes no payment;
- the final handoff names the Streamable HTTP URL, bearer header, and certificate trust; and
- external reachability succeeds with the router forward and fails after the operator disables it.

## Acceptance criteria

- A trusted operator can go from the observed partially bootstrapped host to a validated,
  authenticated read-only MCP endpoint using the checked-in runbook without inventing a value.
- Replaying the prompt refreshes mutable observations instead of treating an old digest, address,
  runner state, or file state as current.
- No action in the runbook alters the existing Caddy deployment or enables the payment capability.
- The repository's Markdown, narrative, and existing automated checks remain green.

Commit locally. Use `narrative-required` and record the separation between observed state and
operator-supplied trust material, the dedicated-port choice that preserves Caddy, the immutable
first-deploy artifact, and the short-lived local identity limit. Do not push unless requested.
