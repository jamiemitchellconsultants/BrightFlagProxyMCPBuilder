---
date: 2026-08-03
slug: correct-prompt-20-to-retain-the-fixed-token-mode
title: "Correct Prompt 20 to retain the fixed-token mode"
summary: "Prompt 20 was written as though Prompt 18 had never been applied. Rewritten to assume its predecessors were applied and to say what it overrides: almost all of the conflict is with Prompt 19, not 18."
kind: correction
status: accepted
sequence: 2026-08-03T05:20:00.000Z
evidence: "Direct push to main, commit b08f30564c8c3ab7c8137975f8867a3a78182eea; no pull request"
---

## Context

The decision recorded in `#entry-add-prompt-20-shared-home-lab-audience-and-a-completable-oauth-flow`
rested on an observation about the implementation repository: no branch of
`BrightFlagProxyMCPServer` contains a fixed-token mode, so Prompt 18 had evidently never been
applied. From that, Prompt 20 concluded the fixed token should be retired rather than completed, and
said it must not be applied at all.

The observation was accurate and the inference from it was wrong. A builder prompt is not written
against whatever state an implementation repository happens to be in. It is written for a repository
where its predecessors *were* applied, and its job when it conflicts with them is to say exactly
what it overrides in what they built — which is what Prompt 19 itself does, in a supersession table,
for the Stage 17 decisions it replaces.

Reasoning instead from the current state of one implementation produced a prompt that would have
been wrong for any repository that had played the sequence in order, and that quietly discarded a
working mode on the strength of a fact about a repository rather than a fact about the design.

## Decision

Rewrite Prompt 20 on the assumption that Prompts 1 to 19 were all applied, and give it a supersession
table in the same form Prompt 19 uses.

Prompt 18's fixed-token mode is retained unchanged. The conversational agents are an additional
population of callers, not a replacement for the clients that send a fixed header and run no OAuth
flow, and Prompt 18's bare `Bearer` challenge with no discovery remains exactly right for those
clients. Every discovery requirement in Prompt 20 is scoped to Keycloak mode only.

Almost all the genuine conflict is with Prompt 19: the `brightflag-mcp` audience and resource
client, the designated user holding both roles, the device-flow token sent as a fixed header, the
plaintext canonical endpoint and its transport selection, the explicit `http://` Caddy site, the
private-source matcher, the `auth.tqaentry.com:host-gateway` entry, and Prompt 19's instruction not
to describe the deployment as a trusted-proxy deployment — which it now is. Thirteen decisions,
named individually, with everything unlisted surviving.

The cross-repository configuration contract gets its own table. Prompt 19 made the option names a
contract with the LocalAI deployment script; the names are unchanged and five values are not.

## Consequences

The fixed-token mode has a future again. Work that had been treated as contingent on a mode being
retired — generating and rotating the fixed MCP token regardless of the selected authentication mode
— is ordinary work blocked only on Prompts 18 and 19 being applied.

Three statements in the LocalAI repository asserted the superseded framing and were corrected there.
Its `-AuthMode fixed` warning stays, because it remains true of the server today, but it now means
"Prompts 18 and 19 have not been applied" rather than "this mode is going away".

This entry is hand-written because the rewrite was pushed directly to `main` rather than through a
pull request. The Project Narrative action triggers on merge events only, so no entry was produced
and none could be produced afterwards by labelling anything. The content reached `main` without
review, which is the second cost of that mistake and the reason it is recorded here rather than left
to be inferred from a commit that has no pull request attached to it.
