---
date: 2026-08-02
slug: approve-and-freeze-ai-mcp-server-development-deployment-plan
title: "Approve and freeze the ai-mcp-server development deployment plan"
summary: "Approve the adversarially reviewed plan as the frozen implementation baseline for Builder Stages 18 and 19 and the LocalAI Windows deployment owner."
kind: architecture
status: accepted
sequence: 2026-08-02T06:30:48.000Z
---

## Context

The proposed `ai-mcp-server` development deployment plan was revised after two adversarial reviews.
It now states the cross-repository ownership boundaries, the exclusive fixed-token or Keycloak
authentication contract, the plaintext home-lab transport, the exact-commit build and rollback
model, the LocalStack secret consequences, and the separate verification classes explicitly.

Implementation remained blocked because the reviewed artifact still said that it was proposed and
not approved. The approval was requested as a one-off direct commit to `main`, so no pull request
merge event will exist for Project Narrative to turn into a follow-up fragment.

## Decision

Approve the reviewed plan and freeze it as the implementation baseline. Builder Stages 18 and 19
and the LocalAI Windows deployment owner may now be created within the plan's stated repository
boundaries. Implementation must not revise the frozen plan to accommodate a discovered problem; a
required revision must stop implementation and be made through a new reviewed decision.

For this approval only, commit directly to `main`. Record the decision with this hand-written
fragment and compile the generated `Narrative.md`, because the normal labelled pull-request
workflow will not run.

## Consequences

Implementation may begin against the frozen requirements and verification plan. Existing prompts
remain historical, and changes to `BrightFlagProxyMCPServer` still arrive only through the future
Builder prompts rather than direct edits during this work. The deferred issues remain non-blocking
and do not weaken the plan's stated limitations.

Any later change to the approved plan requires its own review and decision record. The direct-main
exception applies only to this approval and does not replace the repository's normal branch,
pull-request, label, or narrative workflow for implementation changes.
