# Stage 13 — Generate an AWS production deployment

Source: `BrightFlagProxyMCPBuilder/prompts/13-aws-production-deployment.md`

## Context

**Contingent stage, and dependent on Stage 12.** Do not deploy more than one instance to AWS before
Stage 12's concurrency work is implemented and its tests are green against a real store. Two replicas
of a build whose payment path was only ever proven single-instance is the exact defect Stage 12 exists
to prevent, and no amount of infrastructure correctness compensates for it.

Two framings govern everything below:

- **This is the live deployment.** Stage 10 could label an untested control a manual gate and move
  on, because a homelab claim is a claim about one operator's house. A claim here is a claim about
  production. An untested control is documented as untested — never as supported.
- **Infrastructure is the small part.** The instance count is a *consequence* of Stage 12's guarantee,
  not a substitute for it. Multi-AZ redundancy is an availability property and says nothing about
  whether two tasks can both pay the same invoice. Do not let the runbook imply otherwise.

Stage 13 also does not close Stage 8's deferred corporate alignment. A VPC and a load balancer are not
an MCP authorization profile.

## Preconditions

- Stage 12 committed and green, including the concurrency suite, **or** the deployment is declared
  single-task with in-process stores and the startup gate enforces exactly that.
- Stage 9 committed: image built from a pinned digest base, `linux/amd64`, dev tooling provably
  absent, readiness endpoint working without a BrightFlag call.
- The Stage 8 TLS posture is configurable, since this deployment uses the fronted-by-proxy mode.

## Scope in

The ECS/Fargate service and task definition; ECR and digest pinning; the ALB and its TLS listener;
VPC placement, security groups, and scoped egress; the fronted-by-proxy TLS configuration and trusted
hop; secret placement via the Stage 3 providers; observability and alarms; deployment, rollback, and
the reconciliation runbook entry; automated validation.

## Scope explicitly out

Any change to the MCP surface, the four operations, the evidence rule, or the payment gate. Stage 12's
store semantics — consumed here, not redesigned. Public internet exposure unless explicitly required
and recorded. Autoscaling policy: the instance count follows the declared topology, and a scaling rule
that can move it outside the declared value is a defect, not a feature. Closing the Stage 8 corporate
alignment gap.

## Work items

### 1. Supported baseline — state one, precisely

- **ECS on Fargate**, the Stage 9 image pulled from ECR **by immutable digest**, `linux/amd64`, never
  a tag;
- **ALB terminating TLS** with an ACM certificate, as the trusted fronting proxy Stage 8's TLS posture
  expects. The task serves plain HTTP on the internal VPC network and is never itself a public
  listener;
- **the topology Stage 8 declares at deployment time, with its matching store selection** — one task
  with in-process stores, or multiple tasks with Stage 12's shared stores. The deployment must fail
  rather than run a mismatched pairing, and the pairing is checked by the same startup gate rather
  than by a second implementation of the rule in infrastructure code;
- Streamable HTTP at `/mcp`, reachable only through the load balancer.

Record: region, account boundary, Fargate platform version, image digest and source revision, and the
exact ALB, ACM, and store configuration. Placeholders for account identifiers, hostnames, tenant
values, credentials.

Everything is checked-in infrastructure as code. Nothing is created by console click and documented
afterwards — that pattern is what makes an environment undescribable six months later.

### 2. Network boundary

Document the concrete layout and the full request path from an allowed MCP client to `/mcp`: DNS, the
listener, the target group and its health check, and the task's placement.

- tasks in **private subnets, no public IP**;
- the task security group accepts inbound **only from the ALB's security group, by security group
  reference rather than CIDR**. A CIDR that happens to be the ALB's subnet is a different and weaker
  statement;
- **egress scoped to named destinations** — the configured BrightFlag origin, the identity provider's
  JWKS endpoint, the store, and the AWS endpoints needed for secrets and logs. Prefer VPC endpoints
  where they remove a NAT path entirely. A default `0.0.0.0/0` egress rule is a finding, not a
  baseline;
- the ALB is **internal** unless external reachability is an explicit recorded requirement. "The
  clients are internal" is the common case and should not quietly get a public listener.

Prove as tests, not assertions: the task port is unreachable from anywhere but the ALB, and the
service is unreachable over plain HTTP from outside the VPC.

### 3. TLS and forwarded headers

TLS terminates at the ALB. Configure the server's fronted-by-proxy posture per Stage 8, with the
trusted hop set to **the load balancer specifically, not a wildcard**.

Prove a request carrying a spoofed forwarded-protocol header from an untrusted source does not change
the server's view of the connection. This is the test that distinguishes a configured trust boundary
from a decorative one.

Enforce a modern TLS policy on the listener; redirect or refuse plain HTTP at the listener; document
certificate renewal and the failure mode if renewal lapses.

### 4. Identity

The live trust provider fetches signing keys from the organisation's identity provider over HTTPS.
Preserve Stage 8's startup failure when the local provider is selected outside a non-production
profile, and confirm the dev token-issuing tool is **absent from the deployed image** rather than
merely inactive — Stage 9 made absence structural, and this stage verifies it on the artifact actually
deployed.

Document issuer, audience, required claims, key-cache lifetime, and behaviour when the JWKS endpoint
is unreachable at startup and at renewal. Unreachable-at-startup must fail closed.

Record explicitly which parts of the organisation's MCP authorization profile, gateway controls, and
production trust policy remain outstanding. Do not represent a VPC boundary or an ALB as a substitute.

### 5. Secrets — reconciling with Stage 3

Stage 3 forbids a vendor-specific vault in the secret providers, and this stage wants Secrets Manager.
The reconciliation is that **ECS injects, the server does not fetch**: the task definition's
`valueFrom` resolves Secrets Manager or SSM SecureString references into the container, and the
existing environment or file-reference provider reads them unchanged. No AWS SDK call, no new
provider, no vendor coupling in `BrightFlagMcp.Core`.

Requirements:

- the task role grants access to **exactly those secret ARNs** and nothing broader;
- no secret value in the task definition, environment blocks, infrastructure code, state files, or
  logs. Note that infrastructure state files are a real leak path and are frequently forgotten;
- the cursor-signing key is **identical across tasks** and never generated in-process — Stage 3's rule,
  now load-bearing per Stage 12;
- rotation procedure per secret, and the observable effect of rotating each one while running.

### 6. Observability and operations

- structured logs to CloudWatch Logs with the correlation identifier and Stage 12's instance identity,
  and no credential, token, or full payload. Document retention;
- alarms on task health, restart count, and ALB 5xx rate; and — **the signal that matters most here** —
  on any occurrence of the typed indeterminate-payment outcome. That outcome means a human must
  reconcile in BrightFlag, and it must never sit unnoticed in a log stream;
- deployment by immutable digest with rollback, and Stage 12's version-skew behaviour stated: during a
  rolling deploy a plan issued by the outgoing version is rejected with a re-plan error rather than
  reinterpreted;
- a runbook entry for the indeterminate-payment reconciliation path, written for whoever is on call
  rather than for whoever wrote it.

### 7. Automated validation

Without contacting a live BrightFlag tenant and without making a payment:

- validate the infrastructure definitions; reject unresolved placeholders;
- prove the effective task definition carries every Stage 9 and Stage 10 hardening control — non-root
  user, read-only root filesystem, dropped capabilities, no privileged mode, bounded CPU and memory;
- prove the task definition contains no plaintext secret and references only the intended ARNs;
- prove the task security group admits only the ALB, and that egress is scoped;
- prove the declared instance count and configured store implementation are a valid Stage 12 pairing,
  and that an invalid pairing fails rather than deploys;
- verify readiness without triggering a BrightFlag call;
- verify unauthenticated, expired, and wrong-audience requests are refused **through the ALB**;
- verify a read-only caller cannot execute a payment;
- verify the surface is still exactly four tools and one resource;
- tear down **only** resources carrying the validation run's unique name.

Anything not safely validatable against a non-production account is a labelled manual gate with an
exact expected result and a place to record what was observed. Skipped checks are never reported as
passing — the Stage 10 rule, and it does not relax because this environment is more important.

## Tests

- the task port is unreachable except from the ALB;
- the service is unreachable over plain HTTP from outside the VPC;
- a spoofed forwarded-protocol header from an untrusted source does not alter the server's view of the
  connection;
- startup fails when the local trust provider is selected under this profile;
- the dev token tool is absent from the deployed image;
- an unreachable JWKS endpoint at startup fails closed;
- the task definition carries every hardening control, and no plaintext secret;
- the task role resolves exactly the intended secret ARNs and no others;
- declared instance count and store selection are a valid pairing, and an invalid pairing fails;
- readiness succeeds with no BrightFlag call;
- unauthenticated, expired, and wrong-audience requests are refused through the ALB;
- a read-only caller cannot execute a payment;
- the registered surface is exactly four tools and one resource;
- the indeterminate-payment alarm fires on a simulated indeterminate outcome;
- validation teardown removes only uniquely named resources.

## Acceptance checks

- Reachable only over TLS, only through the load balancer, only by authenticated callers.
- Deployed topology and store implementation are a valid pairing, enforced at startup rather than
  assumed by infrastructure.
- No credential or private signing material in the image, repository, task definition, infrastructure
  code, state files, or logs.
- Task role scoped to named resources; egress scoped to named destinations.
- Deployment and rollback use immutable digests and retain auditable source revisions.
- The indeterminate-payment outcome is alarmed and has a documented reconciliation procedure.
- Deferred corporate alignment recorded as still open, not quietly treated as closed.
- Validation runs with no live BrightFlag call and no payment.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build && \
  make -C deploy/aws validate
```

## Stage boundary

Commit locally. Suggested message: `Add the AWS production deployment and its validation`.

`narrative-required` when published, recording: the supported baseline; the network boundary and how
it is proven; the TLS termination point and the trusted proxy hop; **the ECS-injects-rather-than-server-fetches
resolution** that keeps Secrets Manager out of the Stage 3 providers; the rotation procedures; the
alarm on indeterminate payments; and which corporate-alignment items remain open.

Do not push unless requested.

## Risks

- **Deploying multiple tasks before Stage 12 lands.** The service will appear healthy. The defect
  surfaces as a duplicate payment assertion in a finance system, which is the worst available failure
  signature. The startup gate is what makes this a refusal rather than a silent risk, which is why it
  is enforced by the server and not re-implemented in infrastructure code.
- A CIDR-based task security group rule that looks equivalent to a security-group reference and is
  not. It admits anything that later occupies that address range.
- Infrastructure state files as a secret-leak path. Frequently forgotten, and not covered by scanning
  aimed at the repository and the image.
- The indeterminate-payment alarm is easy to configure and easy to leave unrouted. An alarm nobody
  receives is the same as no alarm, and this is the one outcome that requires a human.
- Autoscaling added later by someone reasonably assuming a stateless HTTP service can scale freely.
  The declared topology must be the enforced ceiling, and the reason must be written where whoever
  adds the scaling policy will read it.
- Presenting multi-AZ availability as if it addressed payment correctness. It is the most natural
  sentence to write in a production runbook and it is false.
