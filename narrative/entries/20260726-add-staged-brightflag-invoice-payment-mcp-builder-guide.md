---
date: 2026-07-26
slug: add-staged-brightflag-invoice-payment-mcp-builder-guide
title: "Add staged BrightFlag invoice-payment MCP builder guide"
summary: "Adopt the staged builder pattern and bind the taught server to five fixed operations: the three accounts-payable batch reads, the invoice-summary read, and the single invoice-payment-status write."
kind: product
status: accepted
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/1; merge commit 19fb28eb5e190496e0c2fae31544ea4d273f7c66"
---

## Context

The BrightFlag external API exposes well over a hundred operations covering SCIM user provisioning, matters, vendors, allocations, purchase orders, budgets, reporting, and document download. Only three product capabilities were requested. A prompt sequence that handed a coding agent the whole OpenAPI document would produce a generic authenticated proxy in which a model's assertion, rather than a reviewed contract, decides which financial mutation runs.

## Decision

Adopt the staged builder pattern and bind the taught server to five fixed operations: the three accounts-payable batch reads, the invoice-summary read, and the single invoice-payment-status write. Define "approved for payment" as release into an accounts-payable batch corroborated by a configured, reviewed invoice-status allow-list, never by a status string the agent guesses. Restrict version one of the payment write to `paymentStatus` `PAID`, behind a caller-bound expiring plan, a single outbound POST, and no automatic retry. Generate the ontology schema deterministically from checked-in contracts and a reviewed OpenAPI snapshot, and keep the ontology service outside the server's outbound surface.

## Consequences

Every added operation, status value, or payment state becomes a reviewable contract change rather than a configuration tweak. Requiring accounts-payable batch evidence is stricter than the API alone requires and will surface tenant configurations where the batch feed is unused. Because the payment-status endpoint is not documented as idempotent, the plan-and-confirm split and the single-POST rule are load-bearing rather than stylistic: a failed submission must be re-planned and re-authorised instead of retried.
