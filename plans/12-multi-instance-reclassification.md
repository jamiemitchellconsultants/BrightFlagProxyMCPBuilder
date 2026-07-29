# Stage 12 — Make the payment path safe under concurrent instances

Source: `BrightFlagProxyMCPBuilder/prompts/12-multi-instance-reclassification.md`

## Context

**Contingent stage. Not part of version 1.** It runs only after a corporate governance
reclassification moves this service off the low-use / low-criticality classification under which a
single instance is permitted. Nothing here is implemented speculatively.

Stage 8 declared single instance and Stage 6 built in-process stores on the strength of it. That was
never a scaling opinion — it was the *justification* for `IPlanStore` and `IPaymentRecordStore` being
in-process at all. Withdrawing the justification does not automatically break anything, which is
precisely the danger: the service will appear to work with two replicas, and the failure will surface
as a duplicate payment assertion in a finance system rather than as an error.

The whole stage reduces to one sentence. Stage 6's at-most-one-POST guarantee rested on
"atomically consumes the plan before attempting the write", and in one process that was a
compare-and-remove on a dictionary. Across instances it is a distributed decision, and the losing
instance must be told it lost by the store, not by a second read.

Stage 8's rejection note said a shared plan store "would add the one database dependency the contract
otherwise excludes". That exclusion is overridden here, by the reclassification and nothing else.
Record it as an override, not as a discovery that the exclusion was wrong.

## Preconditions

- The reclassification is recorded, with its date and the classification moved to. Cite it. If it
  cannot be cited, stop.
- Stages 1–9 committed and green.
- **Interface integrity, proven before anything is changed.** No caller of a concrete
  `InProcessPlanStore` / `InProcessPaymentRecordStore` type exists outside composition. Stage 6 put
  both behind interfaces specifically so this stage would be a substitution; if that has leaked,
  repairing it is work item 1 and is reported separately rather than folded into the store work.

## Scope in

Shared implementations of both stores; the atomic plan-consumption transition; duplicate-`paymentRef`
and already-paid checks as conditional writes; expiry without trusting an instance clock; the
inverted startup gate; per-caller rate limits across instances; instance identity in the audit
record; contract-version rejection across a rolling deploy; correcting every document that asserts
single instance.

## Scope explicitly out

Any MCP surface change. Any new BrightFlag operation. `PARTIALLY PAID`, `POSTED`, `VOID` — still out,
and a shared store is not a reason to revisit them. Deleting the in-process stores. AWS or any other
deployment topology (Stage 13). Autoscaling policy, which is a deployment concern and is not what
makes concurrency safe.

## Work items

### 1. Interface integrity preflight

Grep for direct construction and direct type references of both in-process stores. Report findings
before changing behaviour. A leak found here is a Stage 6 defect surfacing late, and naming it that
way keeps this stage's diff honest.

### 2. Store selection — the decision this stage owns

**Decided: DynamoDB**, with DynamoDB Local for tests. Evidenced against the four properties Prompt 12
requires:

- **atomic conditional write** — `PutItem` / `UpdateItem` with a `ConditionExpression`, and
  `TransactWriteItems` where two items must move together. Single-item conditional writes are
  linearizable;
- **strongly consistent reads** — available per-request, and therefore something the code must ask
  for explicitly. See work item 5;
- **expiry not dependent on an instance clock** — see work item 6, and note the TTL trap there;
- **durability across instance loss** — replicated within the region by default.

Rejected: Redis / ElastiCache — adequate primitives, but adds a cluster to operate and its persistence
story is weaker for a record that finance reconciles against. Rejected: RDS — a relational engine and
a connection-pool lifecycle for two key-value tables holding at most a handful of live rows at this
volume.

Chosen for correctness properties, not for throughput. At a few interactions an hour, throughput is
irrelevant and any of the three would serve; the conditional-write and consistency semantics are the
entire basis of the choice, and the plan says so rather than implying a performance rationale.

**No vendor SDK leaks past the store implementation.** Stage 3's "no vendor-specific vault" rule
holds by analogy: the AWS dependency lives in one adapter project behind `IPlanStore` /
`IPaymentRecordStore`, and `BrightFlagMcp.Core` gains no AWS reference.

### 3. Data model

Three item shapes, in one table or two — state which and why:

- **plan item**, keyed on the plan token's hash rather than the token itself, so the store never holds
  the capability in plaintext. Carries the Stage 6 binding (caller, tenant, invoice identity, amount,
  currency, `paymentRef`, contract version), an absolute UTC expiry, and a state of
  `issued` / `consumed` / `indeterminate` / `completed`;
- **payment record item**, keyed on invoice identity, backing the already-paid check;
- **`paymentRef` reservation item**, keyed on `paymentRef`, backing the duplicate check. A separate
  item so the check can be a conditional put rather than a scan.

Retention stays a real limit and not a guarantee, exactly as Stage 6 said. A shared store can hold
records longer than a process lifetime, which makes the already-paid check *stronger* than it was —
state the new retention window explicitly and do not let "stronger" drift into "permanent".

### 4. Atomic plan consumption — the crux

One conditional state transition, `issued` → `consumed`, conditioned on the current state being
`issued`:

- the instance whose conditional write succeeds owns the right to POST;
- every other instance receives the condition failure and **rejects**. It does not retry, does not
  wait, does not re-read to "check whether the other instance finished", and does not fall back to any
  second decision path. A condition failure is a final answer;
- the transition is committed **before** the outbound POST, so the absence of a payment record can
  never be read as evidence that no attempt is in flight;
- after the POST, the plan moves to `completed` on a verified echo, or to `indeterminate` on timeout,
  transport failure, 409, or 5xx. `indeterminate` is terminal and visible to every instance, which is
  how Stage 6's "permanently indeterminate" rule stops being a per-process property.

The plan token stays an opaque random capability. It does not become signed or self-describing to
save a store round-trip — at this volume the round-trip is free, and the Stage 6 reasoning about
capabilities versus cursors is unchanged.

### 5. Duplicate `paymentRef` and already-paid, as writes not reads

Both were check-then-act, which is safe in one process and not across several. Express each as a
conditional write whose failure *is* the rejection:

- reserving a `paymentRef` is a conditional put that fails if the item exists for a different
  invoice;
- marking an invoice paid is a conditional put that fails if a payment record already exists.

Where the reservation and the plan consumption must both hold, use one `TransactWriteItems` rather
than two sequential conditional writes.

Every read that the payment path depends on sets `ConsistentRead = true`. **An eventually consistent
read anywhere in the payment path is a defect, not a performance trade** — and because eventual
consistency is DynamoDB's *default*, this is a thing that must be asserted in tests rather than
assumed from a code review.

### 6. Expiry without trusting a clock

Plan expiry is evaluated against the absolute UTC timestamp stored on the plan item, compared in the
conditional expression, so a consuming instance whose clock has drifted cannot extend or shorten a
plan beyond Stage 6's configured skew.

**DynamoDB TTL is cleanup only, never correctness.** TTL deletion is asynchronous and documented as
typically-within-48-hours, so an expired plan item may still be present and readable. Correctness
comes from the expiry condition; TTL exists so the table does not grow. Writing this the other way
round — trusting TTL to remove expired plans — silently widens every plan's lifetime to as much as
two days.

### 7. The startup gate, inverted

Same principle as Stage 8, opposite direction: a wrong topology is a refusal, not an intermittent
payment defect. Admit exactly two configurations and refuse every other:

| Declared instances | Store implementation | Result |
|---|---|---|
| 1 | in-process | start |
| >1 | shared, both stores | start |
| >1 | in-process, either store | **fail startup** |
| >1 | shared for one store only | **fail startup** |
| unset / neither shape | any | **fail startup** |

Order of configuration must not matter — the gate evaluates the final resolved configuration, not the
order it was bound in.

### 8. Per-caller rate limits

Stage 8's in-process limiter multiplies by instance count once replicas exist, making a documented
limit of N an enforced limit of N × instances. Stage 11's audit reads that as a false control.

**Decided: split by purpose.** The stricter payment-tool limit moves to a shared-store counter, since
it is the one that matters and at this volume a counter write costs nothing. Coarse request-rate and
concurrency limits move to the deployment edge, and the in-process limiter for those is **removed**
rather than left under-counting.

Do not leave both partially in place. A limiter that is present but wrong is worse than one that is
absent and documented as living at the edge.

### 9. Cross-instance invariants already required

Confirm, and add a test where none exists:

- the Stage 3 cursor-signing key is identical across instances and stable across restarts. This was
  already mandatory and already audited at Stage 11 point 12 — it moves from unused insurance to
  load-bearing, and a per-process key now breaks pagination on the very next request;
- caller-identity validation stays per-instance by design. Signing keys are cached per instance with a
  bounded lifetime; that needs no coordination and must not acquire any;
- audit records gain instance identity alongside the correlation identifier, so an attempted payment
  is attributable to the instance that attempted it.

### 10. Contract version across a rolling deploy

Two contract versions may serve concurrently during a rolling deploy — something a single-instance
deployment never faced. The plan record already carries a contract version from Stage 6. Enforce it
on consumption: a plan issued under a different version is **rejected with a typed re-plan error**,
never reinterpreted under the new version's rules.

### 11. Documentation to correct in the same change

- Stage 9's runbook and product documentation. In particular, "an outstanding plan does not survive a
  restart" becomes false under a shared store. The honest replacement is a statement about the
  exactly-once payment guarantee and the reconciliation path for `indeterminate`, not a cheerful note
  that plans now survive restarts;
- Stage 8's topology decision and its recorded consequences;
- the security policy and threat model, which gain a trusted store dependency, its IAM boundary, and
  its retention window;
- Stage 6's note that keeping the stores behind interfaces makes this a substitution — now discharged,
  and worth recording as such;
- Stage 10's homelab guide, which stays single instance and in-process, already framed as its own
  choice rather than inherited.

## Tests

One-to-one with Prompt 12's "Prove" list:

- two concurrent execute calls on one plan token, across instances sharing a store, produce **exactly
  one** POST at the fake server and one typed rejection — asserted on the fake server's recorded call
  count, run repeatedly rather than once;
- the same with both calls inside one millisecond, and with one instance paused between consumption
  and POST;
- an instance killed between consumption and POST leaves the plan `indeterminate`, and a second
  instance refuses to reuse it;
- a duplicate `paymentRef` submitted concurrently to two instances is accepted at most once;
- an already-paid invoice is refused by an instance that never handled the original payment;
- a plan issued by one instance is consumable by another; a plan issued to one caller stays invisible
  and unusable to every other caller across instances;
- plan expiry holds when issuing and consuming instances disagree on wall-clock time by more than the
  configured skew;
- an expired plan whose TTL has not yet fired is still refused — the test that proves TTL is not
  load-bearing;
- a plan carrying a different contract version is rejected with a typed re-plan error;
- every payment-path read is a consistent read, asserted rather than reviewed;
- startup fails for each refusing row of work item 7's table, in both configuration orders;
- the configured payment rate limit is the effective limit across instances; the coarse in-process
  limiter is provably absent;
- cursors issued by one instance are accepted by another;
- the registered surface is still exactly four tools and one resource;
- every Stage 6 rejection — amount mismatch, currency mismatch, stale or future date, non-approved
  invoice, expired, tampered, replayed, cross-caller token — rejects identically under the shared
  store, as a **parameterised test over both implementations**, not a second copy of the suite.

The concurrency tests run against DynamoDB Local, not a mock. A mocked conditional write proves the
mock, and the guarantee this stage exists to restore is precisely the one a mock cannot exercise.

## Acceptance checks

- Exactly one POST per plan token, proven under genuine concurrent execution across instances.
- Plan consumption is one atomic conditional transition; no check-then-act remains in the payment
  path.
- Both implementations pass one shared behavioural suite; the in-process pair remains, and remains
  what Stage 10 uses.
- The startup gate admits two topologies and refuses everything else, including the previously legal
  multi-instance-with-in-process-stores combination.
- Documented rate limits equal enforced rate limits.
- No AWS type is visible outside the store adapter project.
- No new MCP tool, resource, argument, or BrightFlag operation.

```bash
docker compose -f test/dynamodb-local/compose.yaml up -d && \
  dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

The concurrency suite must be in the default `dotnet test` run, not behind an opt-in flag. A
correctness test that CI skips is a comment.

## Stage boundary

Commit locally. Suggested message: `Restore the one-payment guarantee under concurrent instances`.

`narrative-required` when published. Decisions to record: **the reclassification that authorised this
stage**, cited; **DynamoDB chosen for its conditional-write and consistency semantics** rather than
throughput; **the startup gate inverted rather than removed**; **the rate-limit split** between shared
store and edge; and **the replacement of the restart-survival caveat with an exactly-once statement**
plus its reconciliation path. Record also that the contract's no-database exclusion was overridden
deliberately and by whose authority.

Do not push unless requested. **Do not begin Stage 13.**

## Risks

- The concurrency tests are the ones most likely to be written weakly — Stage 6's plan already warned
  this, and the warning gets sharper once the race is genuinely distributed. Drive real parallel
  confirmations against a real store and assert an exact call count of one.
- `ConsistentRead` defaulting to false is the single most likely silent defect in the stage. It will
  pass every test that runs against an idle local store and fail intermittently in production, which
  is the worst possible failure signature for a payment path.
- TTL-as-correctness is the second. It also passes locally, because a local store with a handful of
  items behaves nothing like a TTL sweep under load.
- Inverting the startup gate means the previously-valid single-instance configuration must keep
  working unchanged, or Stage 10's homelab deployment breaks as collateral. Test both topologies, not
  just the new one.
- A shared store is a new availability dependency in front of a financial write. Its unavailability
  must fail closed — no plan issued, no payment attempted — and never degrade to an in-process
  fallback, which would reintroduce exactly the split-brain this stage removes.
