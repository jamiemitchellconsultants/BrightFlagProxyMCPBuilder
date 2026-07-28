---
date: 2026-07-28
slug: add-staged-execution-plans-for-the-builder-prompt-sequence
title: "Add staged execution plans for the builder prompt sequence"
summary: "Record the application of each prompt as a checked-in plan rather than leaving it to the agent executing the stage, so that the reasoning behind a call sequence, and any position taken on a gap, is reviewable before implementation rather…"
kind: product
status: accepted
sequence: 2026-07-28T17:07:35.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/10; merge commit ebd676d28b4cf50e7b2cf2e833f868aadd8bd5a2"
---

## Context

The prompt sequence was cross-examined twice for internal consistency, but never for whether a
coding agent applying it stage by stage would find the instructions sufficient at the point of
execution. Reading all twelve prompts against the live BrightFlag external OpenAPI document
surfaced one genuine specification gap and two decisions the sequence deliberately leaves open but
which bind earlier than the stage that states them.

The gap: Prompt 5 instructs the adapter to "join batch release evidence to invoice summaries by
`invoiceID`", but `getInvoiceSummaryList` exposes no `invoiceID` filter. Its parameters are
`invoiceNumber, vendorRef, matterRef, invoiceStatusChangeStartDate, invoiceStatusChangeEndDate,
invoiceStatus, includePreviousDrafts, invoiceDateStartDate, invoiceDateEndDate, customerMatterId,
sorting.ascending, paging.*`. The join must therefore enumerate summaries over a window and filter
locally, and no prompt says which window. Release date and status-change date are different axes: an
invoice released in a batch on day 5 may have last changed status on day 40, so bounding the summary
read by the batch window alone manufactures reconciliation gaps systematically.

The early-binding decisions: Prompt 8 requires exactly one plan-store topology to be stated and
enforced for the live deployment, but Prompt 6 is where the plan store is first written; and Prompt
6 requires rejecting a `paymentRef` already recorded against a different invoice and a second plan
for an already-paid invoice, which implies durable state Prompt 1 never contracts.

Separately, verification against the OpenAPI document confirmed the sequence's factual claims hold:
all four required `operationId`s exist at the documented method and path; `Batch`, `BatchDTO`,
`InvoiceSummaryAPI`, `InvoicePaymentStatus`, and `ErrorResponse` carry every field Prompt 1
requires; and neither `invoiceStatus` nor `paymentStatus` declares an enum, which is the premise
both governing design calls rest on.

## Decision

Record the application of each prompt as a checked-in plan rather than leaving it to the agent
executing the stage, so that the reasoning behind a call sequence, and any position taken on a gap,
is reviewable before implementation rather than inferred from a diff afterwards.

Take three positions explicitly:

- **Summary-window margin.** Query `invoiceStatusChangeStartDate/EndDate` as the validated batch
  window widened by a configured `SummaryWindowMarginDays`, introduced in Stage 3 alongside the
  lookback so Stage 5 adds no configuration surface. Anything still absent is reported as a typed
  reconciliation gap. Rejected: an independent status-change window, which doubles the ways to
  misconfigure recall; and an unwindowed status-filtered enumeration, which reintroduces the
  unbounded read removed in the operation-inventory correction.
- **Store abstraction ahead of the topology choice.** Put the plan store and a payment record store
  behind interfaces in Stage 6, so Prompt 8's live-topology declaration — which Prompt 10's dev
  deployment does not narrow — remains a configuration-shaped choice rather than a rewrite. State
  the payment record store's retention bound as a limit rather than presenting the already-paid
  check as permanent.
- **Manual gates over silent skips.** Stage 10's Windows-host checks are written with exact commands
  and expected results and labelled manual, per that prompt's own escape clause.

Also record two sequencing observations the plans act on: the MCP SDK is first required at Stage 5,
where the read capability must work through the MCP tool boundary, not at Stage 8; and the container
must be built for `linux/amd64` explicitly, or Stage 10's Windows and WSL 2 host will refuse the
image.

## Consequences

An agent executing a stage now has the reasoning for its call sequence and the position taken on any
gap available before it writes code, and a reviewer can disagree with a plan without reading an
implementation. The cost is a second artefact to keep in step: a prompt change that is not reflected
in its plan makes the plan wrong, and the plans are explicitly subordinate for that reason.

The summary-window margin is the consequential one. It widens the invoice-summary read beyond the
batch window, which affects recall, byte and page ceilings, and the reconciliation-gap count — so it
is configuration with a default and a maximum rather than a constant, and changing it is a reviewed
act. Should a tenant's status-change timestamps prove unrelated to release timing, the margin will
be visible as a persistent non-zero gap count rather than as silently missing invoices.

The live topology remains deliberately open. The plans make it cheap to decide, and say plainly that
neither option is a safe default, but a wrong declaration that is then enforced is worse than either
choice — so Stage 8 cannot begin without an explicit answer.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
