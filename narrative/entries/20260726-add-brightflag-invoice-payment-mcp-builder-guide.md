---
date: 2026-07-26
slug: add-brightflag-invoice-payment-mcp-builder-guide
title: "Add BrightFlag invoice-payment MCP builder guide"
summary: "Teach a capability-limited BrightFlag proxy rather than a generic REST client."
kind: product
status: accepted
---

## Context

The BrightFlag external API published at `https://app.brightflag.com/v3/api-docs/external` exposes well over a hundred operations covering SCIM user provisioning, matters, vendors, allocations, purchase orders, budgets, reporting, and document download. Only three product capabilities were requested: report a BrightFlag schema an ontology service can ingest, retrieve invoices approved for payment, and mark invoices as paid. A prompt sequence that handed a coding agent the whole OpenAPI document would produce a generic authenticated proxy in which a model's assertion, rather than a reviewed contract, decides which financial mutation runs.

## Decision

Adopt the staged builder pattern already used by `SAPProxyMCPServerBuilder` and `OntologyServerBuilder`, and bind the taught server to five fixed BrightFlag operations across three route families: the accounts-payable batch reads, the invoice-summary read, and the single invoice-payment-status write. Define "approved for payment" as release into an accounts-payable batch corroborated by a configured, reviewed invoice-status allow-list, never by a status string the agent guesses. Restrict version one of the payment write to `paymentStatus` `PAID`, behind a caller-bound expiring plan, a single outbound POST, and no automatic retry. Generate the ontology schema deterministically from checked-in contracts and a reviewed OpenAPI snapshot, and keep the ontology service outside the server's outbound surface.

## Consequences

The guide teaches four MCP tools and one read-only resource against a closed BrightFlag surface, so every added operation, status value, or payment state becomes a reviewable contract change rather than a configuration tweak. Requiring accounts-payable batch evidence means an invoice that merely looks approved in a status field is not reported as payable, which is stricter than the API alone requires and will surface tenant configurations where the batch feed is unused. Because the payment-status endpoint is not documented as idempotent, the plan-and-confirm split and the single-POST rule are load-bearing rather than stylistic, and a failed submission must be re-planned and re-authorised instead of retried.

Three of these choices were judgement calls rather than consequences of the API, and each is reversible: the two-signal approval rule, the hard gate and no-retry rule on the payment write, and the read-then-write-then-schema build order. `DESIGN-CALLS.md` records what was chosen, what it costs, which alternatives were rejected, and what a reversal would have to change. Revisiting any of them starts there.
