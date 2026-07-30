---
date: 2026-07-30
slug: add-binding-agent-instructions-and-teach-the-pattern-in-prompt-2
title: "Add binding agent instructions and teach the pattern in Prompt 2"
summary: "Adopt the pattern, adapted rather than copied. `CLAUDE.md` is binding regardless of which tool reads it and is the only place rules live; the six pointers say only that it is authoritative and that instruction changes belong there. `CLAUDE."
kind: product
status: accepted
sequence: 2026-07-30T04:53:59.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/26; merge commit bdbab092d5917372ec9d819bed5b204de5be2bd8"
---

## Context

This repository had no `CLAUDE.md` and no agent control files at all. Nothing on disk told an agent that a decision-bearing pull request needs **both** the `narrative-required` label **and** three `## Narrative …` headings in the pull-request body, or that `Narrative.md` is generated rather than hand-edited.

The cost was concrete and recent: PRs #20, #21, #22 and #23 all merged without narrative entries, and three had to be reconstructed by hand afterwards. The root cause was not a missing check — `.github/pull_request_template.md` documented the label rule the whole time. It was that creating pull requests with `gh pr create --body ...` replaces the template wholesale, so the instructions were never seen. An agent that had read a binding instruction file would not have made that mistake four times.

`BrightFlagCFS` already solves this with a `CLAUDE.md`-as-source-of-truth pattern and six thin pointer files. Two problems with copying it directly. Its narrative section predates this toolchain and describes a materially different mechanism — `seq`-numbered fragments, a singular `## Consequence` heading, and an explicit rule *never* to commit the compiled file. All three would fail this repository's validator, which requires plural `## Consequences`, date-prefixed filenames, and a committed `Narrative.md` kept in sync by `check`. Separately, its pointer files advertise `.amazonq/rules/` and `.kiro/steering/`, neither of which exists in that repository.

## Decision

Adopt the pattern, adapted rather than copied. `CLAUDE.md` is binding regardless of which tool reads it and is the only place rules live; the six pointers say only that it is authoritative and that instruction changes belong there.

`CLAUDE.md` §2 documents the mechanism as it actually behaves: the label and body sections are both required; a missing label exits the workflow silently while missing sections fail it visibly; **neither is repairable after merge**, because the workflow triggers on the merge event only. It states that supplying a PR body replaces the template, records the exact fragment schema, and forbids rewriting an accepted entry — a reversal is a new `correction` entry citing the original by slug.

§3 records the two git failures from this session as rules: do not stack a pull request on another PR's branch, and announce follow-up commits pushed to a branch with an open PR.

Pointers deliberately cite only locations that exist, and Prompt 2 and Plan 2 both state that constraint explicitly, naming the sibling repository's stale list as the failure mode being avoided.

## Consequences

An agent picking up any later stage now has the narrative contract in front of it, and the specific mistake made four times this session is written down as a rule rather than left as tacit knowledge.

Seven instruction files now exist where there were none, and the pointers are a maintenance surface — adding a tool means adding a pointer and updating the lists in the others. Keeping the pointers content-free is what bounds that cost.

Prompt 2's scope grows: implementation repositories now get an instruction file set as part of the unlabelled installation change. The two-pull-request lifecycle it already documented becomes enforceable by an agent rather than dependent on the learner remembering.

Deliberately not done: no CI check that a decision-bearing pull request carries the label. That was raised, is still open, and would catch what instructions can only discourage.
