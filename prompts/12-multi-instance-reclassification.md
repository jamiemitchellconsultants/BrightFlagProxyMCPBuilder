# Prompt 12 — Make the payment path safe under concurrent instances

Using the reusable contract, implement Stage 12: replace the single-instance live topology declared
in Prompt 8 with a multi-instance one, without weakening the at-most-one-payment guarantee that
Prompt 6 established.

**This stage is contingent.** It is not part of the version 1 sequence and must not be implemented
speculatively. It exists so that a governance reclassification — from low-use / low-criticality,
where a single instance is permitted, to a classification requiring multiple instances — is a
prepared transformation rather than an improvisation. Do not begin unless that reclassification has
actually been recorded.

The single-instance decision was never a scaling opinion. It was the reason an in-process plan store
and payment record store were sufficient. Removing it removes that justification, so this stage is
about restoring the guarantee under concurrency, not about running more containers. Running more
containers is the easy part and is explicitly not the hard part.

## What must not change

The MCP surface stays exactly four tools and one resource. The four fixed BrightFlag operations, the
approved-for-payment evidence rule, the plan-before-execute split, the `PAID`-only payload, and the
no-automatic-retry rule are all unchanged. This stage adds no capability. If any acceptance criterion
below appears to require widening the surface, stop and report the conflict.

## Preconditions

State them explicitly and fail fast if unmet:

- the governance reclassification is recorded, with a date and the classification it moved to;
- Prompts 1–9 are implemented and green;
- `IPlanStore` and `IPaymentRecordStore` are still the only routes to plan and payment-record state,
  with no caller of a concrete in-process type outside composition. Prove this by inspection before
  changing anything — if the interfaces have leaked, repairing that is the first work item and is
  reported separately.

## Shared store implementations

Provide externally shared implementations of `IPlanStore` and `IPaymentRecordStore` alongside the
existing in-process ones. Both implementations remain selectable by configuration; neither is
deleted, because Prompt 10's homelab deployment keeps running a single dev instance with the
in-process pair.

The shared store must provide, and the implementation must be chosen for, these properties. State
which store was chosen and evidence each property against that store's documented guarantees:

- an **atomic conditional write** — create-if-absent and update-if-in-expected-state — as a single
  operation, not a read followed by a write;
- **strongly consistent reads** for every read the payment path depends on. An eventually consistent
  read anywhere in the plan-consumption path is a defect, not a performance trade;
- server-side expiry or an expiry check that does not depend on any single instance's clock;
- durability across the loss of any one instance.

Do not introduce a general-purpose database dependency beyond what these two stores require. Do not
route reads, listings, or ontology output through the shared store.

## The at-most-one-payment guarantee under concurrency

This is the substance of the stage. In a single process, "atomically consume the plan before
attempting the write" was a local operation. Across instances it is a distributed one, and the
failure mode is two instances both observing an unconsumed plan and both issuing a POST for the same
invoice.

Require:

- plan consumption is a **single atomic conditional state transition** in the shared store, from
  issued to consumed, conditioned on the plan being in the issued state. The instance whose
  transition succeeds may attempt the POST. Every other instance loses and must reject, not retry,
  not wait, and not fall back to a second read;
- the transition happens **before** the outbound POST, so a lost instance can never conclude from
  the absence of a payment record that no attempt is in flight;
- a plan whose POST may have been sent stays permanently indeterminate — that state is now stored
  where every instance sees it, and Prompt 6's rule that such a plan is never reusable is enforced
  across instances, not just within one;
- duplicate-`paymentRef` and already-paid checks read the shared store with a strongly consistent
  read and, where the check guards a write, are themselves expressed as a conditional write rather
  than a check-then-act;
- plan expiry is evaluated against an absolute stored timestamp in UTC, so instance clock drift can
  shorten or extend no plan's life beyond the configured skew already defined in Prompt 6;
- the plan token remains an opaque random capability. It does not become signed or self-describing
  to avoid a store round-trip. The binding still lives only in the store record.

## The startup gate, inverted

Prompt 8 fails startup when a configuration declares more than one instance. That gate does not get
deleted — it inverts, keeping the same principle that a wrong topology is a refusal rather than an
intermittent payment defect:

- fail startup when multi-instance is declared and a shared store implementation is **not**
  configured for both `IPlanStore` and `IPaymentRecordStore`;
- fail startup when the in-process implementation of either store is selected together with a
  multi-instance declaration, in either order of configuration;
- keep failing startup on any state that is neither an explicitly declared single instance with
  in-process stores nor an explicitly declared multi-instance with shared stores. There is no third
  state and no default.

## Per-caller rate limits

Prompt 8's per-caller rate limits, including the stricter limit on the payment tool, are enforced
in-process and therefore multiply by the instance count once replicas exist. Choose one and document
it:

- enforce limits against a shared store so the configured limit is the effective limit; or
- move enforcement to the deployment edge and **remove** the in-process limiter rather than leaving
  a limiter that under-counts.

Do not leave both partially in place. A limit that is documented as N and enforced as N × instances
is a false control, and Prompt 11's audit reads it as one.

## Cross-instance invariants already required

Confirm rather than re-derive, and add a test for each if one does not exist:

- the cursor-signing key from Prompt 3 is identical across instances. This was already required; it
  moves from unused insurance to load-bearing, and a per-instance key now produces cursors that fail
  on the next request rather than harmlessly;
- caller-identity validation is unchanged and per-instance by design — signing keys are cached per
  instance with a bounded lifetime, which is correct and needs no coordination;
- audit records carry the instance identity in addition to the correlation identifier, so a payment
  can be attributed to the instance that attempted it.

## Rolling deployment and version skew

Two instances at different contract versions may serve concurrently during a rolling deploy, which a
single-instance deployment never had to consider. Require that a plan issued by one version is
rejected — not misinterpreted — by an instance running a different contract version. The plan record
already carries a contract version per Prompt 6; enforce it on consumption and return a typed error
that tells the caller to re-plan.

## Documentation to correct

Every place that asserts single instance as a property of the running system must be corrected in the
same change, not left to drift:

- Prompt 9's runbook and product documentation, including the statement that an outstanding plan does
  not survive a restart. Under a shared store a plan can survive an instance restart, and the honest
  replacement statement is about the exactly-once payment guarantee, not about restart survival;
- Prompt 8's recorded topology decision and its stated consequences;
- the security policy and threat model, which gain a new trusted dependency;
- Prompt 10's homelab guide, which stays single instance and must now say so as its own choice rather
  than by reference to the live topology.

## Tests

Prove:

- two concurrent execute calls against the same plan token, from instances sharing one store, result
  in exactly one POST observed by the fake BrightFlag server and one typed rejection — run this
  repeatedly, not once, and assert on the observed call count rather than on a return value;
- the same, when the two calls arrive within the same millisecond and when one instance is paused
  between plan consumption and POST;
- an instance killed between plan consumption and POST leaves the plan permanently indeterminate,
  and a second instance refuses to reuse it;
- a duplicate `paymentRef` submitted concurrently to two instances is accepted at most once;
- an already-paid invoice is refused by an instance that never handled the original payment;
- a plan issued by one instance is consumable by another, and a plan issued to one caller is still
  invisible and unusable to every other caller across instances;
- plan expiry holds when the issuing and consuming instances disagree about wall-clock time by more
  than the configured skew;
- a plan carrying a different contract version is rejected with a typed re-plan error, not
  misinterpreted;
- startup fails for multi-instance with in-process stores, and for multi-instance with only one of
  the two stores shared;
- startup still fails for a declared instance count that matches neither supported topology;
- the configured per-caller rate limit is the effective limit across instances, or the in-process
  limiter is provably absent and the edge control is evidenced;
- cursors issued by one instance are accepted by another;
- the registered surface is still exactly four tools and one resource;
- every Prompt 6 rejection — amount mismatch, currency mismatch, stale or future date, non-approved
  invoice, expired, tampered, replayed, cross-caller token — still rejects identically under the
  shared store, as a parameterised test over both store implementations rather than a second copy of
  the suite.

The concurrency tests must exercise the real store implementation against a local instance of the
chosen store, not a mock. A mocked conditional write proves nothing about the guarantee this stage
exists to restore.

## Acceptance criteria

- Exactly one POST per plan token, proven under concurrent execution across instances, not inferred
  from the single-instance behaviour.
- Plan consumption is one atomic conditional transition; no check-then-act remains anywhere in the
  payment path.
- Both store implementations pass the same behavioural suite; the in-process pair remains available
  and remains what Prompt 10's homelab deployment uses.
- The startup gate admits exactly two topologies and refuses everything else, including the
  previously valid combination of multi-instance with in-process stores.
- Documented rate limits equal enforced rate limits.
- No new MCP tool, resource, argument, or BrightFlag operation.
- Formatting, build, and tests succeed, including the concurrency suite against a real store.

Commit locally. Use `narrative-required` and record the reclassification that triggered this stage,
the shared store chosen and the specific guarantee relied on, the inversion of the startup gate, the
rate-limiting decision, and the replacement of the restart-survival caveat with an exactly-once
statement. Do not push unless requested.
