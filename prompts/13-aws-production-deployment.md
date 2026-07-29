# Prompt 13 — Generate an AWS production deployment

Using the reusable contract and the artifacts produced by Prompts 1–12, implement Stage 13: an
operator-ready AWS deployment path for the live service. This stage documents and automates
deployment; it must not widen the MCP or BrightFlag surface by a single tool, operation, or argument.

**This stage is contingent and depends on Prompt 12.** Do not deploy more than one instance to AWS
before Stage 12's concurrency work is implemented and its tests are green. Running two replicas of a
build whose payment path was only ever proven single-instance is the exact defect Stage 12 exists to
prevent, and no amount of infrastructure correctness compensates for it.

Two framings govern everything below:

- **This is the live deployment.** Unlike Prompt 10's homelab guide, claims made here are claims
  about production. A control that is not tested is documented as untested, never as supported.
- **Infrastructure is the small part.** The instance count is a consequence of Stage 12's guarantee,
  not a substitute for it. Do not present multi-AZ redundancy as if it addressed payment correctness.

## Supported baseline

Choose and state one primary, reproducible baseline:

- ECS on Fargate running the image produced by Prompt 9, pinned by immutable digest from ECR, built
  for `linux/amd64`;
- an Application Load Balancer terminating TLS with an ACM certificate, as the trusted fronting proxy
  Prompt 8's TLS-posture configuration expects — the task itself serves plain HTTP on the internal
  VPC network and is never a public listener;
- the topology Prompt 8 declares at the time of deployment, with its matching store selection: a
  single task with in-process stores, or multiple tasks with Stage 12's shared stores. State which,
  and make the deployment fail rather than silently run the wrong pairing;
- Streamable HTTP at `/mcp`, reachable only through the load balancer.

Record the tested region, account boundary, Fargate platform version, image digest and source
revision, and the exact ALB, ACM, and store configuration. Use placeholders for account identifiers,
hostnames, tenant values, and credentials. Everything is defined as checked-in infrastructure as
code; nothing is created by console click and then documented after the fact.

## Network boundary

Document the concrete layout and the full request path from an allowed MCP client to `/mcp`,
accounting for DNS, the load balancer, target group health checks, and the task's placement.

- tasks run in private subnets with no public IP;
- the task security group accepts inbound traffic **only** from the load balancer's security group,
  by security group reference rather than by CIDR;
- egress is restricted to the destinations the service actually needs — the configured BrightFlag
  origin, the identity provider's JWKS endpoint, the shared store when Stage 12 is in play, and the
  AWS service endpoints required for secrets and logging. Prefer VPC endpoints where they remove a
  NAT path entirely. A default-open `0.0.0.0/0` egress rule is a finding, not a baseline;
- the load balancer is internal unless external reachability is an explicit, recorded requirement.

Prove, as a test rather than an assertion, that the task's port is unreachable from anywhere except
the load balancer, and that the service is unreachable over plain HTTP from outside the VPC.

## TLS and forwarded headers

TLS terminates at the load balancer. Configure the server for its fronted-by-proxy posture per
Prompt 8, and configure the trusted proxy hop to be the load balancer specifically — not a wildcard.
Prove that a request arriving with a spoofed forwarded-protocol header from an untrusted source does
not change the server's view of the connection. Enforce a modern TLS policy on the listener, redirect
or refuse plain HTTP at the listener, and document certificate renewal and the failure mode if
renewal lapses.

## Identity

The live trust provider fetches signing keys from the organisation's identity provider over HTTPS.
Preserve Prompt 8's startup failure when the local trust provider is selected outside an explicitly
non-production profile, and confirm the dev token-issuing tool is absent from the deployed image
rather than merely disabled. Document issuer, audience, and required-claim configuration, key-cache
lifetime, and what the service does when the JWKS endpoint is unreachable at startup and at renewal.

This stage does not close Prompt 8's deferred corporate alignment gap. Record explicitly which parts
of the organisation's MCP authorization profile, gateway controls, and production trust policy remain
outstanding, and do not represent a VPC boundary or a load balancer as a substitute for them.

## Secrets

Provide the BrightFlag bearer token, the cursor-signing key, and any store credential through Prompt
3's secret-provider interface, backed by Secrets Manager or SSM Parameter Store SecureString and read
via the task role at startup. Requirements:

- the task role grants access to exactly those secret ARNs and nothing broader;
- no secret value appears in the task definition, environment variables, infrastructure code, logs,
  or state files;
- the cursor-signing key is identical across tasks and is never generated inside the server;
- rotation procedures for each secret, and the observable effect of rotating each one while the
  service is running.

## Observability and operations

- structured logs to CloudWatch Logs, carrying the correlation identifier and the instance identity
  Stage 12 adds, and carrying no credential, token, or full payload. Document the retention period;
- alarms on task health, restart count, load balancer 5xx rate, and — because it is the signal that
  matters most here — any occurrence of the typed indeterminate-payment outcome, which requires
  operator reconciliation in BrightFlag and must never sit unnoticed in a log;
- a documented deployment procedure using immutable digests, with rollback, and with Stage 12's
  version-skew behaviour stated: during a rolling deploy a plan issued by the outgoing version is
  rejected with a re-plan error rather than misinterpreted;
- a runbook entry for the reconciliation path when a payment outcome is indeterminate.

## Automated validation

Add a validation target that, **without contacting a live BrightFlag tenant and without making a
payment**:

- validates the infrastructure definitions and rejects unresolved placeholders;
- proves the effective task definition carries every hardening control Prompt 9 and Prompt 10
  already require — non-root user, read-only root filesystem, dropped capabilities, no privileged
  mode, bounded CPU and memory;
- proves the task definition contains no plaintext secret and references only the intended ARNs;
- proves the task security group admits only the load balancer, and that egress is scoped;
- proves the declared instance count and the configured store implementation are a valid pairing per
  Stage 12's startup gate, and that an invalid pairing fails rather than deploys;
- verifies readiness without triggering a BrightFlag call;
- verifies unauthenticated, expired, and wrong-audience requests are refused through the load
  balancer;
- verifies a read-only caller cannot execute a payment;
- verifies the registered surface remains exactly four tools and one resource;
- tears down only resources carrying the validation run's unique name.

Anything that cannot be validated safely against a non-production account is labelled a manual gate
with an exact expected result and a place to record the observed result. Skipped checks are never
reported as passing.

## Acceptance criteria

- The service is reachable only over TLS, only through the load balancer, and only by authenticated
  callers.
- The deployed topology and the configured store implementation are a valid pairing, enforced at
  startup rather than assumed by the infrastructure.
- No credential or private signing material in the image, repository, task definition, infrastructure
  code, state files, or logs.
- Task role permissions are scoped to named resources; egress is scoped to named destinations.
- Deployment and rollback use immutable digests and retain auditable source revisions.
- The indeterminate-payment outcome is alarmed and has a documented reconciliation procedure.
- Deferred corporate alignment is recorded as still outstanding, not quietly treated as closed.
- Validation runs with no live BrightFlag call and no payment.
- Formatting, build, tests, infrastructure validation, and the deployment validation target succeed.

Commit locally. Use `narrative-required` and record the supported baseline, the network boundary and
its proof, the TLS termination point and trusted proxy hop, the secret placement and rotation
procedures, the alarm on indeterminate payments, and which corporate-alignment items remain open. Do
not push unless requested.
