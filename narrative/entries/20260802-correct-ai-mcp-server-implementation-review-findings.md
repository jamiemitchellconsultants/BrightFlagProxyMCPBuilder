---
date: 2026-08-02
slug: correct-ai-mcp-server-implementation-review-findings
title: "Correct ai-mcp-server implementation review findings"
summary: "Correct Stage 18/19 sequencing and require the reviewed integration-test invoice status explicitly after implementation review; track the remaining Keycloak work in LocalAI issue #28."
kind: correction
status: accepted
sequence: 2026-08-02T10:02:26.000Z
---

## Context

Implementation review compared the frozen deployment plan and its generated Stage 18 and 19
artifacts with the pending LocalAI Windows deployment. It found that Stage 18 had prematurely
adopted Stage 19's shared Keycloak topology and that LocalAI had guessed a customer-specific
approved-invoice status. It also found command-argument credential exposure, optional Caddy
activation, false-success stop and removal paths, and incomplete Keycloak desired-state handling.

The frozen-plan decision required a later correction to receive its own review and decision record.
The operator directed that sequencing and status defects be corrected plan-first, selected three
implementation defects for direct correction, accepted or skipped the home-lab operational risks,
and grouped the remaining Keycloak work into a dedicated follow-up issue.

## Decision

Revise the reviewed plan so Stage 18 preserves Prompt 17's dedicated `brightflag-mcp` realm and
public-client contract, while Stage 19 alone moves deployment to the shared `homelab` realm and
`mcp-client`. Require the reviewed integration-test tenant's approved-invoice status as an explicit
deployment input with no guessed default, and regenerate the affected prompts and plans from those
corrections.

In LocalAI, remove the GitHub token command parameter, make Caddy validation and reload deployment
conditions, and make stop and removal fail rather than report success after native-command errors.
Track complete fail-closed Keycloak setup, exact mapper verification, and auth-independent fixed
token materialisation together in LocalAI issue #28. Preserve all twelve implementation-review
findings and their dispositions in a separate review artifact.

## Consequences

Stage 18 can now be applied directly to the Prompt 17 baseline without depending on a later host
topology, and Stage 19 receives the customer-specific status it needs without inventing vocabulary.
The LocalAI script no longer offers a GitHub credential through process arguments and does not
claim Caddy, stop, or removal success when the underlying operation failed.

The dedicated Keycloak correction remains explicit follow-up work rather than a partially applied
change. Saving the review records the accepted home-lab risks, the deliberately skipped
production-origin check, and the still-open impossible token-claim manual gate without describing
any of them as fixed.
