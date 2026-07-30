---
date: 2026-07-30
slug: make-fragments-only-a-binding-rule-in-claude-md
title: "Make fragments-only a binding rule in CLAUDE.md"
summary: "State it as a binding rule with the three cases named separately, because they fail in different ways: - **Adding an entry** — write a fragment. A hand-added index row is destroyed by the next compile, since the index is derived."
kind: product
status: accepted
sequence: 2026-07-30T05:30:51.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/29; merge commit a0c12e9fea980745430b92d3310c9a524352b2fc"
---

## Context

`CLAUDE.md` said `Narrative.md` was generated and should never be hand-edited, in one sentence. That was enough for the obvious case — appending an entry by hand — and not enough for the case that actually arose.

[PR #28](https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/28), the automation's proposal for #26, conflicted on `Narrative.md`. It had branched before #27 added a hand-written entry, so both sides recompiled the projection from different fragment sets. The fragments themselves merged cleanly — entries are separate files and the two sides added disjoint ones — and only the generated file collided.

Nothing in the instructions said what to do. Hand-reconciling the conflict markers would have looked entirely reasonable: the markers were small and the result would have rendered fine. It would also have been wrong, producing an index and entry numbering that the next compile silently discards, and there would have been no signal that anything was lost.

A related gap: the rule needed to distinguish *committing* generated output from *authoring* it. This repository commits the compiled file alongside the fragment because `validate-narrative.yml` runs `check`, which fails when the output is stale. A rule phrased as "never write `Narrative.md`" would read as forbidding that and would break CI on every narrative pull request.

## Decision

State it as a binding rule with the three cases named separately, because they fail in different ways:

- **Adding an entry** — write a fragment. A hand-added index row is destroyed by the next compile, since the index is derived.
- **Changing wording** — edit that entry's fragment, never the projection.
- **Resolving a conflict in `Narrative.md`** — discard both sides and recompile, however trivial the markers look. The correct resolution is the union of the fragments, which the compiler computes and a human should not.

Draw the line at authorship rather than at writes: running the compiler is not authoring the file, because compilation is deterministic and model-free, so the output is a function of the fragments and nothing else. "Compile it; never type it." The compiled file stays committed here, and the rule says so explicitly along with the note that other repositories in this family invert that convention.

Rejected: forbidding committed generated output outright, the `BrightFlagCFS` model where CI recompiles on `main` and a check rejects any branch touching the compiled file. That would eliminate this class of conflict entirely rather than prescribing a resolution, but it needs workflow changes here — `validate` would have to stop failing on staleness and something would have to recompile on `main`. Worth considering separately; asserting it in `CLAUDE.md` without the CI to match would just make the instructions wrong.

## Consequences

The conflict resolution that took a diagnosis this session is now a written rule, and the reasoning for it — fragments merge, projections collide — is recorded rather than left to be re-derived.

The rule as written still permits the conflict to occur; it only makes the resolution unambiguous and mechanical. Any narrative pull request merging while a proposal is open will still collide on the compiled file. Removing that requires the rejected alternative above.

The pointer files are unchanged. They restate none of `CLAUDE.md`'s rules by design, so a new rule in §2 needs no corresponding edit in six other places.
