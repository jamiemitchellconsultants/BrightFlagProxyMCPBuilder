# Stage 22 — Declare the upstream by class, and withdraw payment's special status

Source: `BrightFlagProxyMCPBuilder/prompts/22-upstream-class-and-ordinary-payment.md`

## Context

Two problems meet in one stage, and they are the same problem seen from opposite ends.

The first is that the sequence never gave the BrightFlag origin a name in the cross-repository
contract. Prompt 19 lists eight environment-variable names, calls a rename on either side a breaking
change, and omits the origin — while the same prompt requires that origin to be an integration-test
one. The deployment must therefore be setting it under a name no document states. That gap is why
"point the server at a different BrightFlag" has no answer in the sequence as it stands.

The second is that the origin was never the only thing that varied. Prompt 3 wrote one rule for all
deployments — HTTPS everywhere, plain HTTP only for a loopback fake — and Prompt 19 bolted the
environment onto the *capability* instead, by enabling payment against the integration-test tenant
only. So "which BrightFlag" ended up half in the origin and half in the payment grant, and neither
half could express production.

Declaring a class fixes both. The class says which BrightFlag this is; the origin is then validated
against that statement; and the marking that tells a reader a payment confirmation is not real is
derived from it rather than configured beside it. Once the class carries the environment, the
payment grant no longer has to, which is what makes withdrawing payment's special status a
completion of this change rather than a separate loosening.

The reversal is deliberate and is the learner's call, already taken: an authenticated caller
reaching this server reaches all of it. What that does *not* touch is the write's integrity, and the
plan spends more effort on that boundary than on anything else, because it is the boundary a reader
will get wrong.

## Preconditions

- Stages 1–11 committed and green; Stages 17–20 applied.
- The decision to withdraw payment's special status is recorded, with its date. It reverses
  implemented behaviour in Prompts 8 and 19 and is not a defect repair; if it cannot be cited, stop.
- Stage 21 is applied **only if** the `Fake` class will be selected. The class is defined here
  regardless; the service it points at is Stage 21's.

## Scope in

The upstream class and its validation table; class-derived result marking; collapsing two
authorization capabilities into one; retiring the payment-specific role and rate limit; the
generated configuration manifest and its drift tests; correcting every document that states payment
is restricted or that the origin is integration-test by rule.

## Scope explicitly out

Any MCP surface change — still four tools and one resource. Any new BrightFlag operation. The
payment write's mechanics, all of which survive. Multi-instance topology, which stays Stage 12's and
stays contingent. Any production deployment topology, which is Stage 13's and also contingent.
Editing the LocalAI script, which Stage 19 made that repository's own.

## Work items

### 1. The class, and what it validates

A closed three-value option in the Stage 3 tree — `Production`, `IntegrationTest`, `Fake` — with no
default. Unset or unrecognised fails startup, on the same reasoning Stage 4 applied to the empty
allow-list.

The origin stays one configured value and no hostname enters this repository. What the class adds is
that the origin is checked against the row it belongs to, per the prompt's table: scheme, address
category, whether the fake secret provider is admissible, and which deployment profile may select
it. Every row is a test, and both directions of the profile rule matter — `Fake` refused under a
production profile, and `Production` refused under a profile not marked production.

Stage 3's blanket rejections are unchanged and must still be proven under all three classes. It
would be easy to reimplement URL validation around the class and lose one of them.

### 2. What the class cannot do

The server cannot verify what is at the far end of an origin, and Stage 3 forbids it from fetching
the OpenAPI document at startup to find out. So `Fake` pointed at BrightFlag, or `Production`
pointed at Stage 21's fake, passes startup and fails later as a connection or authentication error.

Write that down where the class is documented. A class that reads as validation of the upstream's
identity is worse than no class, because it invites trusting a label the server never checked.

### 3. Marking, derived not configured

Under `IntegrationTest` and `Fake`, the marker naming the class appears in every audit record, every
log scope, and the body of every tool response. Under `Production`, none.

The response body is the load-bearing one, asserted specifically on a successful
`brightflag_mark_invoice_paid` result. This is where Stage 21's
`BrightFlag__Upstream__Acknowledgement` is superseded: it was a second setting governing the same
label, and a second setting is a second thing that can be wrong. Retire the name; do not keep it as
an alias.

### 4. One capability

Stage 8 defined two capabilities and granted them separately. They collapse into one grant governing
the whole surface. Stage 19's `BrightFlag__Authorization__MarkInvoicePaidRoles__*` is **retired, not
renamed** — a search for the retired name is one of the tests.

Stage 8's stricter payment rate limit goes too; one limit set applies across the surface. Do not
leave it partially in place. Stage 12's plan already establishes the principle: a limiter that is
present but wrong is worse than one that is absent and documented.

The argument for dropping it is not that the payment tool is unimportant. It is that the plan-token
mechanism bounds the write far more tightly than a rate limit does — no plan, no reachable write,
and at most one POST per plan — so the rate limit was never what made payment safe. Say that rather
than implying the limit was redundant in general.

### 5. The boundary that survives

Every integrity property of the write is unchanged, and the diff must show that plainly:

- `DESIGN-CALLS.md` §2 in full — plan token, explicit confirmation, `paymentStatus` fixed to `PAID`,
  atomic consumption, at most one POST attempt, no automatic retry on timeout, 409, or 5xx, and an
  ambiguous outcome returned as ambiguous;
- Stage 6's plan-before-execute split and the permanently indeterminate plan;
- deny by default, evaluated per call *including at the execute step*, every decision audited;
- the caller-scoped, capacity-bounded, expiring plan store, and cross-caller invisibility; and
- Stage 4's evidence rule, re-verified at the plan step against a fresh read.

`DESIGN-CALLS.md` §2 needs no change. Its title says the write is gated hard, and everything under
it is about the write's mechanics and its refusal to retry — not about who may call it. Access
control lives in Stage 8, and Stage 8 is what this stage reverses. Resist the urge to edit §2 to
match; it is already correct.

### 6. The manifest

Startup emits a configuration manifest derived deterministically from the options tree: every key a
deployment must supply, marked required or optional, recording whether a default exists, carrying no
values. One test fails on manifest-versus-tree drift; another fails when a required key also has a
default.

This is the part that actually closes the Stage 19 gap, and it closes it structurally. Completing
the hand-written list instead would fix today's omission and keep the mechanism that produced it —
and the list has already been wrong once, across two stages and a reviewed implementation.

The manifest names the class and the origin, which is what lets a deployment set them at all.

### 7. Documents to correct in the same change

- every statement that payment is enabled against the integration-test tenant only, or that payment
  is gated by a named role separate from the read grant;
- Stage 9's runbook and product documentation, wherever they describe a partial surface;
- the security policy and threat model, which now describe one grant rather than two, and which gain
  the upstream class as a stated control with an explicitly stated limit;
- Stage 11's audit expectations, where they check for the retired payment role; and
- the root `README.md` and `START-HERE.md`, wherever either describes payment as restricted.

## Tests

One-to-one with Prompt 22's "Prove" list:

- the class admits exactly three values, has no default, and fails startup unset or unrecognised;
- every row of the class table, both directions of the profile rule included;
- every Stage 3 URL and configuration rejection still fails under all three classes — one
  parameterised test over the classes, not three copies;
- the class-derived marker is present in audit records, log scopes, and tool response bodies under
  `IntegrationTest` and `Fake`, absent under `Production`, asserted on a successful
  `brightflag_mark_invoice_paid` result;
- one authenticated, granted caller reaches all four tools and the resource, and no configuration
  produces a partial surface;
- a search fails on the retired payment role name and the retired acknowledgement name, across
  configuration, code, generated files, and documentation;
- the write's integrity properties hold under every class: one POST per atomically consumed plan, no
  automatic retry on timeout, 409, or 5xx, ambiguous returned as ambiguous, `paymentStatus` fixed to
  `PAID`, cross-caller plan refusal, and the Stage 4 evidence rule re-verified at plan time;
- startup still fails above one declared instance, `Production` included;
- manifest and options tree agree; no required key carries a default; no entry carries a value; the
  class and the origin are both named; and
- the registered surface is still exactly four tools and one resource.

Stage 8's "a read-only caller cannot plan or execute a payment" test is the one test this stage
deletes rather than adapts. There is no read-only caller now, and rewriting it to assert something
adjacent would leave a test whose name claims a property the server no longer has.

## Acceptance checks

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

Selecting `Production` or `IntegrationTest` against a real BrightFlag origin is a **manual gate**.
Record the exact command, the class, and the expected result. Nothing in this environment reaches
either, and neither may be reported as passing.

## Stage boundary

Commit locally. Suggested message: `Declare the BrightFlag upstream by class, payment now ordinary`.

`narrative-required` when published. Decisions to record: **the withdrawal of payment's special
status**, naming both what was withdrawn and what was deliberately kept; **the declared upstream
class** replacing an unnamed origin, and the fact that the server cannot verify what is at the far
end; **marking derived from the class** rather than configured beside it; and **the generated
manifest** replacing a hand-maintained cross-repository name list that had been silently incomplete.

Do not push unless requested. **Do not begin Stage 12 or Stage 13.** Selecting `Production` here
does not discharge the reclassification those stages wait on, and a production upstream under this
stage is a single instance.

## Risks

- The reversal will be read as broader than it is. Someone will take "payment is an ordinary
  capability" as licence to add a retry, or to expose `paymentStatus`. Work item 5 is the mitigation
  and it is a documentation mitigation, which is the weak kind; state the boundary in the code's
  tests as well as in prose.
- `Production` becomes reachable in the same change that removes the payment grant. That is the
  intended outcome and it is also the largest single increase in blast radius in the sequence. The
  single-instance gate and the manual gates are what stand between it and a live write.
- Retiring a configuration name breaks the LocalAI script the moment this ships. Stage 19 made that
  repository the deployment owner, so the two changes have to land in a known order; a deployment
  generating the retired payment role against a server that rejects unknown properties fails
  startup, which is the loud failure, not the dangerous one.
- Class-conditioned URL validation is where a Stage 3 rejection quietly goes missing. Parameterise
  over the classes rather than reimplementing per class.
- The manifest is easy to write as documentation that happens to be generated. It has to be the
  thing the deployment reads, or it becomes a third place the contract is stated.
