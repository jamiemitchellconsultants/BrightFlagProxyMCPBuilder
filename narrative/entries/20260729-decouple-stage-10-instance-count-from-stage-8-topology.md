---
date: 2026-07-29
slug: decouple-stage-10-instance-count-from-stage-8-topology
title: "Decouple Stage 10's instance count from Stage 8's topology"
summary: "Stage 10's homelab deployment runs one instance as its own choice, not as an inheritance from Stage 8. The present agreement between the two is coincidental, and Stage 10 stays single-instance if Stage 8 is ever revised."
kind: correction
status: accepted
sequence: 2026-07-29T15:08:13.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/20; merge commit 359d19c8701973a239c85612f618b49faa34ca7c"
---

## Context

The preceding entry, [declare a single-instance live topology for Stage 8][entry-11], settled the live
plan-store topology and corrected `plans/10` in the same change. In doing so it recorded a specific
framing: that Stage 10's dev deployment "now happens to agree with the live topology, and the
documentation must say the agreement is a consequence of Stage 8's decision".

That framing is wrong in one direction, and the direction matters. It makes Stage 10's instance count
*derived* from Stage 8's — which reads correctly today, because both are one, but establishes a
dependency that does not exist and should not. Stage 10 runs a single instance because a homelab or
local-network Windows host is the wrong environment in which to validate horizontal scaling. That
reason is intrinsic to Stage 10 and survives any decision Stage 8 later takes.

The practical failure is easy to picture. If a governance reclassification moves Stage 8 to a
multi-instance live topology, a reader following the recorded reasoning would conclude that Stage 10's
homelab guide should follow it, and would set about running two containers on a single Windows host to
stay consistent with a decision that never governed Stage 10 in the first place.

## Decision

**Stage 10's single-instance choice belongs to Stage 10.** It is not derived from Stage 8's live
topology, does not narrow it, and does not track it.

The agreement between the two is stated as coincidental rather than load-bearing. If Stage 8 is later
revised to a multi-instance live topology, Stage 10's homelab deployment still runs exactly one
instance, and the documentation says so explicitly rather than leaving a reader to infer which way the
dependency runs.

The prior entry's core decision is untouched: the live deployment runs a single instance, enforced at
startup. Only its account of *why `plans/10` agrees* is corrected here. That entry stays `accepted`
rather than `superseded`, because the decision it records still holds and only its supporting
reasoning about Stage 10 is refined.

While the same files were open, a second scoping gap was closed. Stage 10's Windows 11 / Docker
Desktop / WSL 2 / PowerShell 7 baseline is the **deployment target**, and nothing in it constrains
Stage 9's CI pipeline or the project's general automated test environment. Stage 9's CI keeps whatever
runner it already uses and continues to build the `linux/amd64` image Stage 10 deploys; only the
inherently Windows-specific checks — Defender Firewall, NTFS ACLs, certificate store, Task Scheduler,
Docker Desktop's container mode — are Windows-only, and those were already written as labelled manual
gates.

## Consequences

The two stages are now independent in the direction that matters. A reclassification of Stage 8 is a
change to one stage rather than a change that propagates into a deployment guide it never governed,
and the contingent Stage 12 written later the same day depends on exactly this separation holding.

Stage 10's prompt and plan both state the coincidence explicitly, which is more words than a bare
"single instance" and is the point: the sentence exists to stop a future reader reconstructing the
dependency that was just removed.

The CI scoping statement forecloses a reading nobody had yet acted on but which the baseline invited —
that a Windows deployment target implied Windows CI runners. No workflow changed; the constraint was
never real, and is now written down as not real.

This entry is a correction to recorded reasoning rather than a new decision, and is filed as one.
Amending the prior entry in place was considered and rejected: an accepted narrative entry is a record
of what was decided and why at the time, and editing it to read as though the better framing had been
present from the start would remove the evidence that the framing ever needed correcting.

[entry-11]: #entry-declare-a-single-instance-live-topology-for-stage-8
