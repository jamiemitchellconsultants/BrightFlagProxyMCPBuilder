---
date: 2026-07-30
slug: adopt-the-pre-merge-narrative-gate
title: "Adopt the pre-merge narrative gate"
summary: "Adopt `OntologyService`'s workflow **verbatim** rather than writing a variant, so the two cannot drift and the file can be compared by checksum across repositories."
kind: product
status: accepted
sequence: 2026-07-30T05:32:08.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/30; merge commit 3f7d255d12b3077520104a1b5bbcc33199fcbddc"
---

## Context

[PR #24](https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/24) carried `narrative-required` but was labelled before its body carried the three `## Narrative …` sections, and it merged inside that window. The `maintain` job failed with `A narrative-required PR must contain Narrative Context, Narrative Decision and Narrative Consequences headings`.

That failure is **permanent, not retryable**. The action reads `pr.body` from the merge event payload, so correcting the body afterwards changes nothing and `gh run rerun` reads the same incomplete text. The entry had to be hand-written as a fragment in [#27](https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/27), which the merge-event-only trigger makes the only available repair.

Instructions alone cannot close this. `CLAUDE.md` §2 already said both the label and the sections are required — it was added in #26, before #24 merged — and the mistake still happened, because nothing checked. A rule that is only enforced by an agent remembering it is not enforced.

`OntologyService` already carries the fix and has for some time: a `require-narrative-sections` job that runs on `pull_request` including `labeled` and `unlabeled`, and fails a labelled pull request whose body is missing any of the three non-empty sections. Its own inline comment states the intent exactly — a pre-merge mirror of the post-merge gate, "so a missing heading fails review instead of silently failing after merge." Four repositories in this family, including this one, had the older workflow without it.

## Decision

Adopt `OntologyService`'s workflow **verbatim** rather than writing a variant, so the two cannot drift and the file can be compared by checksum across repositories.

The job reads the pull-request body from an environment variable rather than interpolating it into a shell command, because pull-request prose is untrusted input. It triggers on `labeled` and `unlabeled` so it re-evaluates when classification changes rather than only when commits are pushed.

It deliberately does **not** check for a missing label. Only a human can decide whether a change is decision-bearing; the job catches the narrower, mechanically checkable case where classification was declared and the evidence was not supplied. That is the case that failed here.

Prompt 2 gains the same requirement, so repositories built from this sequence get the gate rather than inheriting the older workflow.

Also repairs two inline code spans that an earlier reflow had split across line breaks. They render correctly, but an agent grepping for the exact heading it must emit would not have found `## Narrative Context`.

## Consequences

The label-without-sections failure becomes a red check on an open pull request — visible and fixable — instead of a permanent loss discovered after merge.

The gate does not address the other failure mode. An unlabelled decision-bearing pull request still merges silently and produces no entry, which is how four entries were lost earlier in this sequence. Closing that requires judging whether a change is decision-bearing, and no CI check can do it.

Enforcement now lives in two places, the workflow and `CLAUDE.md` §2, which can disagree. The workflow is authoritative because it is the one that runs.

`narrative check` passes. The `maintain` workflow is unchanged.
