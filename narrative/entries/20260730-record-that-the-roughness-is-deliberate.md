---
date: 2026-07-30
slug: record-that-the-roughness-is-deliberate
title: "Record that the roughness is deliberate"
summary: "**Record the policy, not the list.** `DESIGN-CALLS.md` §4 states that deliberate roughness exists and that a discrepancy a reader has found may be one instance of it."
kind: product
status: accepted
sequence: 2026-07-30T18:28:32.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/33; merge commit 7428bfcb351ad2b134a1d44972ff61dfe1274dea"
---

## Context

This repository is a course, and its rough edges are load-bearing: where two documents answer the
same question differently, the exercise is for the agent to surface the tension and the learner to
resolve it. That intent was nowhere written down.

The consequence showed up in practice. An agent implementing Stage 10 in the implementation
repository found that the labelling criterion Prompt 2 tells a repository to record disagrees with
the pull-request template the same prompt installs, reported it as a defect, and proposed a fix to
the prompt sequence. The report was the mechanism working exactly as intended. The proposed fix would
have deleted the exercise, and only did not because the intent was supplied by hand, in conversation,
at the moment of asking.

Without a record, every future agent rediscovers the same tension and proposes the same tidy-up, and
one of them eventually gets a yes.

## Decision

**Record the policy, not the list.** `DESIGN-CALLS.md` §4 states that deliberate roughness exists and
that a discrepancy a reader has found may be one instance of it. It deliberately does **not**
enumerate which tensions are intentional: this file ships in the repository the learner clones, so an
enumeration would be a spoiler with a permanent audience.

**Tell agents what to do with a gap, in four rules.** Report it as an ordinary finding, with the
readings and their consequences. Do not resolve it unasked or silently write around it. Do not
announce that a tension is intentional or cite the section that says so — surfacing is the agent's
job and deciding is the learner's. And, drawn explicitly because the first three could be read as
licence for it: **if asked directly, answer truthfully.** Not volunteering a spoiler is not the same
as denying a fact, and an agent that tells a learner the sequence is consistent when it is not has
destroyed the thing the exercise runs on.

**Rejected: annotating each deliberate gap in place.** A `<!-- deliberate, do not fix -->` marker
hands over the answer at exactly the moment the learner should be working it out, and turns every
remaining unannotated inconsistency into an accident by implication.

**Rejected: reconciling the documents.** That produces a seamless specification nobody has to think
about, and the skill the sequence teaches — noticing that two authorities disagree, and choosing — is
the one that matters when the same person later reviews an agent's work on something that was never
a course.

## Consequences

An agent reading `CLAUDE.md` now learns before it starts that a gap may be intentional, which should
convert "I found a defect, here is the fix" into "I found a tension, here are the readings" without
suppressing the report itself.

Three costs, stated rather than qualified away. The instruction is guidance, not enforcement, and
pulls against an agent's default to be maximally helpful, so expect occasional leakage. The file is
in the learner's own repository, so a learner who reads `DESIGN-CALLS.md` early learns that the game
exists — they learn only that, not where the answers are. And deliberate roughness remains
indistinguishable from neglect to anyone who has not read §4, which is the cost of not enumerating.

§4 cites the labelling-criterion finding as evidence that the mechanism works, without naming which
document is wrong. That is the closest this file comes to a spoiler, and it is a deliberate trade for
having a concrete observed example rather than an assertion.

The cost to a learner who misses a gap is bounded and already documented: for the narrative mechanism
specifically, a missed entry is written by hand as a fragment afterwards — recoverable and tedious,
which is what makes it a usable lesson rather than a trap.
