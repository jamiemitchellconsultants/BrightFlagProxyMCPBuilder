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

† **Contingent stages, not part of version 1.** Stage 12 runs only if corporate governance
reclassifies this service off the low-use / low-criticality classification under which Stage 8's
single-instance topology is permitted. Stage 13 depends on Stage 12 — deploying more than one instance
before Stage 12 lands is the specific defect Stage 12 exists to prevent. Both are written in advance
so the transformation is a prepared sequence rather than an improvisation, and neither is to be
implemented speculatively.

Stage 14 is an optional operational extension that applies directly after Stage 11. Its number
preserves the already-published identifiers of the two contingent stages; it does not depend on or
authorise either one.

Stage 15 is an optional operational extension after Stage 14. It documents the site-specific first
deployment and OpenCode handoff; it does not enable payment or make either contingent stage apply.

Each plan follows one structure: Context, Preconditions, Scope in, Scope explicitly out, Work items,
Tests mapped one-to-one onto the prompt's own "Prove" list, Acceptance checks with runnable
commands, Stage boundary, and Risks. Every plan ends by naming what must **not** begin next, because
the sequence's value comes from stopping at each boundary.

## Open decisions the plans surface

Seven points where the prompt sequence leaves a choice, or leaves a gap, and a plan had to take a
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
