# Stage 08 — Expose the narrow MCP surface securely

Source: `BrightFlagProxyMCPBuilder/prompts/08-mcp-identity-and-authorization.md`

## Context

Hosting, identity, and authorization for exactly four tools and one resource. Everything before this
stage assumed a trusted caller; this stage establishes who the caller is, what they may do, and how
the surface is prevented from widening by accident.

The topology declared here is the **live deployment's** topology. Stage 10's homelab deployment runs
one dev instance and does **not** narrow this choice.

## Preconditions

Stage 7 committed. Four tools and one resource exist and are settled.

## Scope in

Both transports from one composition root; static surface registration with a startup gate; caller
JWT validation; the swappable trust provider and its dev token tool; authorization; the plan-store
topology declaration; hardening.

## Scope explicitly out

Container packaging, CI, docs, threat model, runbook (Stage 9). Deployment artifacts (Stage 10).
Final alignment with the deploying organisation's MCP authorization profile — explicitly deferred,
see below.

## Work items

### 1. Hosting

Two transports from one composition root:

- local **stdio** for a desktop client;
- **stateless Streamable HTTP at `/mcp`**, authenticated, no session affinity, no server-held caller
  state beyond the plan store.

Register the tool set **statically**. Fail startup if the registered surface is not exactly the four
tools and one resource, so a future refactor cannot quietly widen it. This gate is audited at Stage
11 point 1 and re-verified at Stage 10.

The HTTP transport's TLS posture is a **deployment-time configuration choice**, not a hardcoded
assumption, because different live deployments sit differently relative to TLS termination:

- **Direct TLS** — the server terminates HTTPS itself; or
- **Fronted by a trusted proxy** — the server serves plain HTTP on an interface reachable only from a
  fronting reverse proxy that already terminated TLS (the shape Stage 10's homelab deployment uses).

Exactly one of the two must be selected explicitly; there is no silent third state. **Fail startup**
if the server would serve plain HTTP without the deployment explicitly marking itself as fronted by a
trusted proxy — plain HTTP is never an accidental default. When fronted, trust forwarded-protocol
information only from the configured proxy hop, never from an arbitrary caller-supplied header.

### 2. Caller identity

Under HTTP, validate the caller's bearer token **in-process before any handler runs**: signature,
issuer, audience, expiry, not-before, required claims, bounded clock skew. Reject unsigned,
`none`-algorithm, expired, and wrong-audience tokens. Cache signing keys with a bounded lifetime;
never fetch keys per request.

The caller's token authenticates the caller **to this server**. It is never forwarded to BrightFlag,
and BrightFlag's service token is never returned to a caller. Prove both directions.

Under stdio, derive identity from the local configured principal and **refuse to run** in a mode
where no principal is established.

### 3. Swappable trust provider

Make the JWT trust root — issuer, audience, signing-key source — entirely configuration-driven, so
the validation logic never changes between environments. Only *where it trusts keys and an issuer
from* changes.

- **Live provider** — fetches signing keys from a configured JWKS URL over HTTPS, matching the
  deploying organisation's identity provider (e.g. Entra ID tenant discovery).
- **Local provider** — loads signing keys from a configuration-supplied JWKS document, for local
  development and automated tests only.

Both feed the *same* validation. Neither weakens it; neither introduces a second code path through
it. **Fail startup — never merely warn** — when the local provider is selected under a profile not
explicitly marked non-production. That is the caller-identity analogue of Stage 3's rule rejecting a
production profile with no authentication.

### 4. Dev token-issuing tool

Built the same way as Stage 3's fake server. Given a requested caller identity (subject, tenant,
roles, groups, scope claims) and an optional expiry, wrong issuer, wrong audience, or missing claim,
it mints a token signed by the local provider's key, matching the claim shape live tokens carry — so
authorization logic under test never special-cases which provider issued the token.

It must refuse to run under a profile marked production, and must **not be present in the container
image** built in Stage 9. Place it outside the server project so absence is structural, not a build
flag — Stage 9 has to *prove* absence rather than inactivity.

### 5. Authorization

Two capabilities: **read approved invoices**, and **mark an invoice paid**.

Granted from reviewed server-side configuration keyed on validated claims. Never from a caller
argument, a tool description, an annotation, prompt text, or a model's assertion. Deny by default.
Evaluate **per call**, including at the execute step of a plan. Record every decision in the audit
log.

The plan store is caller-scoped, capacity-bounded, and expiring. A plan issued to one caller is
invisible and unusable to another.

### 6. Topology — the decision this stage owns

**Decided: single instance.** The live deployment runs exactly one server instance. The in-process
plan store and payment record store are therefore sufficient, and horizontal scaling of this service
is out of scope for version 1.

This must be **documented and enforced at startup**, not merely assumed: a configuration that
declares more than one instance fails, so the in-process store can never be running under a
deployment that believes it is scaled out. The two stores stay behind `IPlanStore` /
`IPaymentRecordStore`, which is what keeps a later multi-instance decision a substitution rather than
a rewrite of the payment path.

Rejected: multi-instance with an externally shared plan store. It would add the one database
dependency the contract otherwise excludes, and Stage 10's homelab deployment runs a single dev
instance — so the shared store would ship unproven, with the dev deployment implying an assurance the
live one had not earned. The transport is stateless and session-affinity-free regardless, and Stage 3
already requires the cursor-signing key to be identical across instances, so nothing about this
choice is hard to revisit.

Do not ship an in-process-only store silently alongside multi-instance guidance elsewhere: the
single-instance limit is a documented, enforced property of version 1, and Stage 9's documentation
must say so plainly.

### 7. Deferred corporate alignment

This stage establishes the narrow surface, transport authentication, and server-side authorization
boundary. It does **not** claim final alignment with the deploying organisation's MCP authorization
profile, identity-provider discovery, gateway controls, or production trust policy. Record those as
an explicit deployment gap. Do not weaken anything here while that work is deferred.

### 8. Hardening

Request-size, concurrency, and per-caller rate limits, **stricter on the payment tool**. Structured
logs with a correlation identifier and no credential, token, or full payload. Unknown MCP arguments
rejected, not ignored. Typed errors that leak no origin, path, query, header, or stack detail. All
BrightFlag response content treated as untrusted data, never as instruction.

## Tests

- registered surface is exactly four tools and one resource, and startup fails otherwise;
- unauthenticated, expired, wrong-audience, wrong-issuer, and `none`-algorithm tokens refused;
- a live-provider token and a local-provider token with **identical claims** produce **identical**
  authorization outcomes;
- startup fails when the local provider is selected under a production profile;
- the dev token tool refuses to run under a production profile and is absent from the image;
- a dev-minted token with wrong issuer, wrong audience, missing claim, or expired lifetime is
  rejected exactly as the equivalent live token would be;
- a read-only caller cannot plan or execute a payment;
- a caller cannot execute another caller's plan;
- caller tokens never reach the outbound BrightFlag request; the BrightFlag token never reaches a
  response;
- rate limits and size limits enforced;
- **the declared topology holds** — single-instance operation is documented and enforced, and a
  configuration declaring more than one instance fails startup;
- startup fails if plain HTTP is served without the deployment explicitly marked as fronted by a
  trusted TLS-terminating proxy;
- forwarded-protocol information is honoured only from the configured proxy hop, not from an
  arbitrary caller-supplied header;
- text embedded in a BrightFlag response instructing the server to change behaviour changes nothing.

## Acceptance checks

- Both transports serve the same narrow surface with the same authorization outcome.
- Authorization is server-side, deny-by-default, re-evaluated at execution.
- Single-instance topology is stated and enforced, not left implicit.
- The HTTP transport's TLS posture — direct or fronted-by-proxy — is an explicit, deployment-time
  configuration choice with no silent plain-HTTP default.
- The trust root is swappable by configuration alone; no auth code path differs between providers.
- A production profile cannot select the local provider, and the dev token tool cannot run under one
  or ship in the image.

```bash
dotnet format --verify-no-changes && dotnet build --no-restore && dotnet test --no-build
```

Plus a manual smoke: stdio tool listing shows exactly four tools and one resource; the same over
authenticated `POST /mcp`.

## Stage boundary

Commit locally. Suggested message: `Serve the narrow MCP surface with swappable caller identity`.

`narrative-required` when published. Decisions to record: **validating caller tokens in-process**;
**keeping caller identity separate from the BrightFlag service credential**; **making the trust root
swappable by configuration rather than by code**; **declaring a single-instance live topology**, with
its consequence for the plan store — an outstanding plan does not survive a restart, and horizontal
scaling is out of scope for version 1 rather than merely untested; and **making the HTTP transport's
TLS posture a deployment-time configuration choice** — direct TLS termination or serving plain HTTP
behind a trusted fronting proxy — rather than assuming every live deployment looks like Stage 10's
homelab shape.

Do not push unless requested. **Do not begin Stage 9.**

## Risks

- The `none`-algorithm and unsigned-token tests must exercise the real validation path, not a
  hand-rolled parser in the test.
- "Identical claims produce identical outcomes" is the test that proves the two providers share one
  code path. Write it as a parameterised test over both providers, not two separate tests.
- Single instance is only safe while it is enforced. The failure mode is a deployment that scales to
  two replicas because the transport is stateless and nothing stopped it, at which point a plan
  issued by one instance is invisible to the other and the already-paid check silently narrows to
  whichever instance took the call. The startup gate is what makes that a refusal rather than an
  intermittent payment defect.
- An outstanding plan does not survive a restart. That is acceptable for a five-minute capability —
  the caller re-plans and re-planning re-runs the fresh approval check — but Stage 9's runbook has to
  say it rather than leaving an operator to discover it during a deployment.
- The fronted-by-proxy mode is only safe while the trust boundary is exact. If forwarded-protocol
  headers were honoured from any caller rather than only the configured proxy hop, a caller on plain
  HTTP could claim to already be on HTTPS and bypass the intended posture. Configuration must name the
  trusted hop, not merely enable the mode.
