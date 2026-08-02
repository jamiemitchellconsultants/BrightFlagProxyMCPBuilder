# Execution plans

One execution plan per prompt, mapping file-for-file onto `prompts/`. These describe *how* a prompt
is applied to a concrete implementation repository; the prompts themselves remain authoritative.
Where a plan and its prompt disagree, the prompt wins and the divergence is a defect in the plan.

They were written against builder commit `b4cb267` and validated against the BrightFlag external
OpenAPI document read on 2026-07-28 (114 operations, 70 schemas, byte-identical in size to the
2026-07-15 copy).

| Plan | Prompt |
|---|---|
| [00-reusable-contract.md](00-reusable-contract.md) | [00](../prompts/00-reusable-contract.md) |
| [01-scaffold-and-contracts.md](01-scaffold-and-contracts.md) | [01](../prompts/01-solution-scaffold-and-invoice-contracts.md) |
| [02-project-narrative.md](02-project-narrative.md) | [02](../prompts/02-adopt-project-narrative.md) |
| [03-configuration-and-fake-server.md](03-configuration-and-fake-server.md) | [03](../prompts/03-fixed-brightflag-endpoint-configuration.md) |
| [04-approved-for-payment-rule.md](04-approved-for-payment-rule.md) | [04](../prompts/04-approved-for-payment-evidence.md) |
| [05-read-capability.md](05-read-capability.md) | [05](../prompts/05-list-invoices-approved-for-payment.md) |
| [06-payment-plan-and-execute.md](06-payment-plan-and-execute.md) | [06](../prompts/06-plan-and-mark-invoice-paid.md) |
| [07-ontology-schema.md](07-ontology-schema.md) | [07](../prompts/07-ontology-schema-reporting.md) |
| [08-hosting-identity-authorization.md](08-hosting-identity-authorization.md) | [08](../prompts/08-mcp-identity-and-authorization.md) |
| [09-delivery-and-governance.md](09-delivery-and-governance.md) | [09](../prompts/09-delivery-documentation-and-audit.md) |
| [10-homelab-deployment.md](10-homelab-deployment.md) | [10](../prompts/10-homelab-local-network-deployment.md) |
| [11-reconstruction-audit.md](11-reconstruction-audit.md) | [11](../prompts/11-independent-reconstruction-audit.md) |
| [12-multi-instance-reclassification.md](12-multi-instance-reclassification.md) † | [12](../prompts/12-multi-instance-reclassification.md) |
| [13-aws-production-deployment.md](13-aws-production-deployment.md) † | [13](../prompts/13-aws-production-deployment.md) |
| [14-github-runner-homelab-delivery.md](14-github-runner-homelab-delivery.md) | [14](../prompts/14-github-runner-homelab-delivery.md) |
| [15-deploy-local-runbook.md](15-deploy-local-runbook.md) | [15](../prompts/15-deploy-local-runbook.md) |
| [16 — retire runner local deployment][plan-16] | [16][prompt-16] |
| [17 — shared local deployment][plan-17] | [17][prompt-17] |
| [18 — home-lab fixed-token authentication][plan-18] | [18][prompt-18] |
| [19 — `ai-mcp-server` development deployment][plan-19] | [19][prompt-19] |

[plan-16]: 16-retire-github-runner-local-deployment.md
[prompt-16]: ../prompts/16-retire-github-runner-local-deployment.md
[plan-17]: 17-local-pull-localstack-keycloak-caddy-deployment.md
[prompt-17]: ../prompts/17-local-pull-localstack-keycloak-caddy-deployment.md
[plan-18]: 18-home-lab-fixed-token-authentication.md
[prompt-18]: ../prompts/18-home-lab-fixed-token-authentication.md
[plan-19]: 19-ai-mcp-server-development-deployment.md
[prompt-19]: ../prompts/19-ai-mcp-server-development-deployment.md

† **Contingent stages, not part of version 1.** Stage 12 runs only if corporate governance
reclassifies this service off the low-use / low-criticality classification under which Stage 8's
single-instance topology is permitted. Stage 13 depends on Stage 12 — deploying more than one instance
before Stage 12 lands is the specific defect Stage 12 exists to prevent. Both are written in advance
so the transformation is a prepared sequence rather than an improvisation, and neither is to be
implemented speculatively.

Stages 14 and 15 are superseded. Stage 16 is a corrective stage only for implementations where
either was applied; it removes the self-hosted runner path while preserving verified GHCR
publication. Stage 17 then adds the replacement host-pull deployment. A fresh implementation skips
Stages 14–16 and applies Stage 17 directly after Stage 11. None of these stages enables payment or
makes a contingent stage apply.

Stages 18 and 19 both run after Stage 17, in that order, and are the development deployment for the
`ai-mcp-server` home-lab host. They override only the Stage 17 home-lab decisions named in the
supersession table in Prompt 19, and every unlisted Stage 17 requirement survives. Stage 17's prompt
text and the historical narrative are not rewritten. Stage 19 is the one stage in the sequence that
enables payment, against BrightFlag's integration-test environment only, and neither stage makes a
contingent stage apply.

Each plan follows one structure: Context, Preconditions, Scope in, Scope explicitly out, Work items,
Tests mapped one-to-one onto the prompt's own "Prove" list, Acceptance checks with runnable
commands, Stage boundary, and Risks. Every plan ends by naming what must **not** begin next, because
the sequence's value comes from stopping at each boundary.

## Open decisions the plans surface

Eleven points where the prompt sequence leaves a choice, or leaves a gap, and a plan had to take a
position:

- **The invoice-summary join window (Plan 05).** `getInvoiceSummaryList` exposes no `invoiceID`
  filter, so joining batch release evidence to invoice detail requires enumerating summaries over a
  window and filtering locally — and release date and status-change date are different axes. The
  plans introduce a configured `SummaryWindowMarginDays` in Stage 3, widening the batch window for
  the status-change query, with anything still absent reported as a typed reconciliation gap.
- **The live plan-store topology (Plan 08).** Prompt 8 requires exactly one topology to be stated
  and enforced for the live deployment, and Prompt 10's dev deployment does not narrow it. The plans
  put both stores behind interfaces in Stage 6 so the choice stays cheap, and flag it as needing an
  explicit answer before Stage 8 is implemented.
- **The shared store, if reclassification happens (Plan 12).** Prompt 12 requires a store with atomic
  conditional writes and strongly consistent reads but does not name one. Plan 12 picks DynamoDB, on
  its conditional-write and consistency semantics rather than throughput — at a few interactions an
  hour throughput decides nothing — and records that this deliberately overrides the contract's
  exclusion of a database dependency.
- **Secrets Manager versus Stage 3's no-vendor-vault rule (Plan 13).** Prompt 13 wants AWS-managed
  secrets; Stage 3 forbids a vendor-specific vault in the secret providers. Plan 13 resolves it by
  having ECS *inject* rather than the server *fetch* — `valueFrom` resolves the reference into the
  container and the existing environment or file-reference provider reads it unchanged, so no AWS SDK
  call and no vendor coupling reaches `BrightFlagMcp.Core`.
- **Stage 10's Windows-host gates (Plan 10).** The deployment target is Windows unconditionally.
  Anything not executable while authoring on another host is written as a labelled manual gate with
  an exact command and expected result, per the prompt's own escape clause — never reported as
  passing.
- **The public-repository runner boundary (Plan 14).** GitHub warns against persistent self-hosted
  runners on public repositories. The plan requires the server repository to become private before
  the Docker-capable `ai-mcp-server` runner can accept work, and keeps pull-request execution on
  GitHub-hosted runners.
- **The existing edge-port boundary (Plan 15).** Caddy already owns host ports 80 and 443 for another
  deployment. The plan keeps BrightFlag on its reviewed nginx TLS listener at 8443 and uses a
  separate router mapping, rather than silently coupling the two services through Caddy.
- **The runner-retirement boundary (Plan 16).** Removing repository automation is safe to automate;
  deleting a Windows account, runner service, directory, GitHub environment, or credential is not.
  The plan makes those inventory-first manual gates and explicitly preserves Jamie's long-lived SSH
  access.
- **The replacement shared-infrastructure boundary (Plan 17).** The server remains vendor-neutral:
  the host materialises LocalStack Secrets Manager values into protected files, Keycloak supplies
  tokens from one canonical HTTPS issuer, and BrightFlag owns one LAN-only Caddy fragment rather
  than the shared stacks. Real-secret deployment is blocked while unauthenticated LocalStack is
  reachable from the LAN.
- **What a shared caller identity costs (Plan 18).** Prompt 18 requires a fixed-token mode whose
  identity is one static configured caller, and Prompt 8 requires caller-bound, caller-scoped
  payment plans. Both hold at once only if the plan boundary collapses onto that single identity.
  Plan 18 takes the position that this is disclosed and tested — a test asserts the shared audit
  subject — rather than compensated for, because a compensating control here would invent a
  distinction the mode does not have.
- **Who owns the deployment (Plan 19).** Stage 17 put a deployment entry point in the server
  repository; Stage 19 moves it to a script in the separately managed LocalAI repository, which this
  repository's prompts cannot edit. Plan 19 resolves the split by deleting the server-owned
  artifacts outright, keeping only the server-side transport, host-validation, and capability
  configuration the external deployment requires, and checking the boundary with a grep that fails
  if any retired reference still reads as current.
