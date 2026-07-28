---
date: 2026-07-28
slug: store-the-reviewed-openapi-snapshot-canonically-not-as-served
title: "Store the reviewed OpenAPI snapshot canonically, not as served"
summary: "Canonicalize the fetched document before writing it, using the encoding the prompt already defines rather than a second one written for this purpose, and fingerprint the stored canonical bytes."
kind: product
status: accepted
sequence: 2026-07-28T18:02:00.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/12; merge commit a084a31c08d66ed28fc168806c7a091936290d99"
---

## Context

BrightFlag serves `/v3/api-docs/external` minified onto a single line: 171,651 bytes, zero newlines.

Prompt 1 requires the administrative refresh command to "print a review diff", and requires the
snapshot to be checked in and reviewed before contracts derive from it. Neither requirement is
achievable over those bytes. A line diff between two single-line documents reports that line 1
changed, which is no information at all. The printed diff is unreadable, and every later refresh
lands in a pull request as one unreviewable line — on a document whose whole purpose is to be the
authoritative, reviewed source the contracts are built from.

The prompt already defines a deterministic canonical encoding for generated artifacts: ordinal
property ordering, fixed indentation, normalized line endings. It simply never said to apply it
here, because the shape of the served document was not known when the prompt was written.

A second problem sat behind the first. Fingerprinting the served bytes means the fingerprint moves
whenever BrightFlag reorders the properties it emits, whether or not anything meaningful changed.
That fingerprint is not decorative: Stage 5 embeds it in the keyset cursor and rejects cursors that
disagree, and Stage 7 gates schema generation on it. Serializer churn upstream would invalidate
outstanding cursors and force schema regeneration for no reason.

## Decision

Canonicalize the fetched document before writing it, using the encoding the prompt already defines
rather than a second one written for this purpose, and fingerprint the stored canonical bytes. Pin
the checked-in snapshot to LF through `.gitattributes`, because the fingerprint is over exact stored
bytes and Stage 10 produces a Windows clone where a CRLF checkout would fail validation.

Require the stage to state plainly that the stored document is deliberately not byte-identical to
the served one, and add an acceptance criterion that the snapshot is reviewable as a line diff and
that its fingerprint is proven stable when the served document differs only in property order.

Rejected: keeping the served bytes and pretty-printing only for the diff view, which would mean
fingerprinting one form and reviewing another — two representations of the authoritative document,
and a standing invitation to review the wrong one.

## Consequences

The checked-in snapshot becomes 12,715 diffable lines, so a tenant release that moves an operation
or drops a field shows up as a reviewable diff rather than as a changed hash. The fingerprint now
tracks content rather than upstream serializer behaviour, so cursors and generated schemas are
invalidated by real change.

The cost is that the stored document is no longer byte-identical to what BrightFlag served, so it
cannot be diffed directly against a raw capture from the API. Anyone comparing the two must
canonicalize first. The prompt requires this to be stated rather than left for someone to discover.

Ordinal property ordering also reorders `paths` and `components.schemas` away from BrightFlag's
emission order. That is a deliberate consequence: the snapshot exists to be validated and reviewed,
not republished.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
