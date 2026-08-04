# Prompt 22 — Select the BrightFlag upstream by class, and make payment an ordinary capability

Using the reusable contract and the artifacts produced by Prompts 1–11 and 17–20, replace the single
unnamed BrightFlag origin with a declared upstream class, and withdraw the special status earlier
stages gave the payment tool.

This stage is not contingent. It applies to every deployment, because every deployment has to say
which BrightFlag it is talking to, and until now only one of them could say it.

Apply this stage after Prompt 20. It does not depend on or authorise contingent Prompts 12, 13, or
21, though the `Fake` class defined here is the one Prompt 21's service answers.

## What this stage supersedes

Prompts 3, 8, 17, 19, and 21 remain historical and are not rewritten. When the sequence is played in
order, this stage replaces the following implemented decisions and nothing else:

| Was | Is |
|---|---|
| One BrightFlag origin, class unstated (Prompt 3) | One declared upstream class, and an origin validated against it |
| Plain HTTP permitted only for a loopback fake (Prompt 3) | Plain HTTP permitted only for the `Fake` class, on a private or container-network address |
| Two capabilities, read and mark-paid, granted separately (Prompt 8) | One capability governing the whole surface |
| A stricter rate limit on the payment tool (Prompt 8) | One limit set, applied uniformly across the surface |
| Payment disabled (Prompt 17) | Already superseded by Prompt 19; recorded here for completeness |
| Payment enabled by a named `MarkInvoicePaidRoles` value (Prompt 19) | Payment enabled by the same grant as every other tool |
| Payment against the integration-test tenant only (Prompt 19) | Payment against whichever upstream the declared class names |
| `BrightFlag__Upstream__Acknowledgement` (Prompt 21) | The declared class is itself the acknowledgement |
| The cross-repository name list, maintained by hand (Prompts 19 and 20) | A generated manifest, with the list derived from it |

Every other requirement of those prompts survives. Do not treat this table as licence to relax an
unlisted control.

## The upstream is declared, not inferred

Add a closed upstream class to the Prompt 3 options tree. It has exactly three values and no
default; startup fails when it is unset or unrecognised.

- `Production` — BrightFlag's production API.
- `IntegrationTest` — BrightFlag's integration-test API.
- `Fake` — a non-BrightFlag service standing in for it, such as the one Prompt 21 builds.

The origin remains a single configured value, as Prompt 3 established, and this stage does not put a
hostname in this repository. What changes is that the origin is now validated against the declared
class rather than against one rule for all deployments:

| Class | Scheme | Address | Secret provider | Deployment profile |
|---|---|---|---|---|
| `Production` | HTTPS only | publicly routable; loopback, private, and container-network addresses refused | the fake provider refused | production only |
| `IntegrationTest` | HTTPS only | publicly routable | the fake provider refused | any |
| `Fake` | HTTP or HTTPS | private, loopback, or container-network only; publicly routable addresses refused | any | non-production only |

Every Prompt 3 rejection still applies to every class without exception: caller-controlled origins,
paths, or query strings; URL user information, fragments, wildcards, traversal, or embedded
credentials; redirects to another origin; secrets in configuration files, URLs, or logs; unknown
configuration properties.

**The server cannot verify what is actually at the far end of an origin**, and Prompt 3 forbids it
from fetching the OpenAPI document at startup to find out. That is precisely why the class is
declared rather than detected. A `Fake` class pointed at BrightFlag, or a `Production` class pointed
at Prompt 21's fake, is a configuration error the server will not catch and the operator will meet
as a connection or authentication failure. Say so where the class is documented; do not imply the
server validates the upstream's identity.

## Marking follows the class

The class, not a separate setting, determines how a result is labelled. Under `IntegrationTest` and
under `Fake`, every audit record, every log scope, and **the body of every tool response** — a
successful `brightflag_mark_invoice_paid` result specifically included — carries an unmistakable
marker naming the class. Under `Production` there is no marker, because there is nothing to warn
about.

Deriving the marker from the class rather than configuring it separately is the point. A payment
confirmation is the one artifact in this system that must never read as production when it is not,
and a second setting that governs the label is a second thing that can be wrong.

## Payment is an ordinary capability

Withdraw the payment tool's special status, explicitly and completely.

An authenticated caller whose validated claims satisfy the server's one reviewed grant reaches the
entire surface: all four tools and the resource. There is no second grant, no payment-specific role,
no deployment posture that serves a partial surface, and no upstream class under which the payment
tool is registered but refused. Prompt 8's two capabilities collapse into one, and Prompt 19's
requirement that the payment grant be named separately in generated configuration is withdrawn — the
name is retired, not renamed.

Prompt 8's stricter rate limit on the payment tool goes with it; one limit set applies across the
surface. Nothing is lost by this: the plan-token mechanism already bounds payment far more tightly
than a rate limit ever did, because a caller cannot reach the write at all without first obtaining a
plan and can cause at most one POST per plan.

**What survives, in full, and is not touched by this stage.** A reader will assume this reversal is
broader than it is, so state the boundary:

- everything in `DESIGN-CALLS.md` §2 — the plan token, explicit confirmation, `paymentStatus` fixed
  to `PAID`, atomic plan consumption, at most one POST attempt, no automatic retry on timeout, 409,
  or 5xx, and an ambiguous outcome returned as ambiguous and requiring reconciliation;
- Prompt 6's plan-before-execute split and the permanently indeterminate plan;
- deny by default, evaluated per call including at the execute step, and every decision recorded in
  the audit log;
- the caller-scoped, capacity-bounded, expiring plan store, and the rule that one caller's plan is
  invisible and unusable to another; and
- Prompt 4's evidence rule, which the plan step re-verifies against a fresh read.

Those are integrity properties of the write. This stage changes who may call it, not what it does.

## What selecting `Production` does and does not authorise

Selecting `Production` makes the production API reachable and, because payment is now ordinary,
makes a production payment write reachable with it. State the inherited constraints plainly rather
than letting the class imply readiness:

- Prompt 8's single-instance topology still holds and still fails startup above one instance. The
  at-most-one-POST guarantee rests on an in-process plan store, so a production deployment under
  this stage is a single instance. Prompts 12 and 13 remain contingent and change nothing here.
- The production credential is a real one. The fake secret provider is refused under this class, and
  Prompt 3's rule that neither the bearer token nor the cursor-signing key appears in configuration
  files, URLs, or logs is unchanged.
- This stage prescribes no production deployment topology. It makes a production upstream
  selectable; it does not make one deployed, reviewed, or approved.

## The configuration contract is generated, not listed

Prompts 19 and 20 stated the cross-repository configuration contract as a hand-maintained list of
names. That list has been incomplete: it never named the BrightFlag origin, though Prompt 19
required that origin to be an integration-test one. A list that can be wrong is not a contract.

Emit a configuration manifest at startup, derived deterministically from the options tree, naming
every key a deployment must supply, each marked required or optional, each recording whether a
default exists, and **none carrying a value**. Two tests hold it honest: one fails when the manifest
and the options tree disagree, and one fails when a key the manifest marks required also carries a
default. The deployment generates configuration against the manifest rather than against prose.

A key present in the manifest and absent from any earlier hand-written list is not a new
requirement. It was always required and the list was wrong. The upstream class and the origin are
both in the manifest, which is what allows a deployment to name them at all.

Deployment ownership is unchanged: Prompt 19 made the LocalAI script the sole owner of the home-lab
deployment, and this stage neither edits that script nor reintroduces a deployment entry point here.
It documents what the script must now generate — the declared class and the origin for it — and
leaves the change to that repository, under its own instructions and review.

## Prove

- the upstream class admits exactly three values, has no default, and fails startup when unset or
  unrecognised;
- every row of the class table holds: an HTTP origin fails under `Production` and under
  `IntegrationTest`; a publicly routable origin fails under `Fake`; a private, loopback, or
  container-network origin fails under `Production`; the fake secret provider fails under
  `Production` and `IntegrationTest`; `Fake` fails under a deployment profile marked production; and
  `Production` fails under a profile not marked production;
- every Prompt 3 URL and configuration rejection still fails under all three classes;
- the marker derived from the class is present in audit records, log scopes, and tool response
  bodies under `IntegrationTest` and `Fake`, absent under `Production`, and asserted specifically on
  a successful `brightflag_mark_invoice_paid` result;
- one authenticated, granted caller reaches all four tools and the resource, and no configuration
  produces a partial surface — there is no value that registers the payment tool and refuses it;
- no separate payment role, payment grant, or payment-specific rate limit remains anywhere in
  configuration, code, generated files, or documentation, proven by a search that fails on the
  retired names;
- the payment write's integrity properties are unchanged under every class: one POST per atomically
  consumed plan, no automatic retry on timeout, 409, or 5xx, an ambiguous outcome returned as
  ambiguous, `paymentStatus` fixed to `PAID`, a plan unusable by another caller, and the Prompt 4
  evidence rule re-verified at the plan step;
- startup still fails above one declared instance, including under `Production`;
- the emitted manifest and the options tree agree, no key marked required carries a default, no
  manifest entry carries a value, and the manifest names the upstream class and the origin; and
- the registered surface is still exactly four tools and one resource.

## Acceptance criteria

- A deployment states which BrightFlag it is talking to, and the origin it supplies is validated
  against that statement rather than against one rule for every deployment.
- Production, integration-test, and fake upstreams are each selectable, and the two that are not
  production mark every result they produce.
- An authenticated, granted caller reaches the whole surface. Payment carries no separate grant, no
  separate limit, and no environment restriction.
- Every integrity property of the payment write survives unchanged, and the documentation says which
  properties were withdrawn and which were not.
- The cross-repository configuration contract is generated from the options tree, so it can no
  longer be silently incomplete.
- The limits are stated, not implied: the server cannot verify what is at an origin, a production
  upstream inherits the single-instance topology, and this stage deploys nothing.
- Every check that cannot run in this environment is a labelled manual gate with its exact command
  and expected result, and is never reported as passing.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the withdrawal of the payment tool's special
status and what was deliberately kept, the replacement of an unnamed origin with a declared upstream
class, the derivation of result marking from that class, and the replacement of the hand-maintained
cross-repository name list with a generated manifest. Do not push unless requested.
