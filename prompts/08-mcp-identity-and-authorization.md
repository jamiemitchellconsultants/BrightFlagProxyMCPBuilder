# Prompt 8 — Expose the narrow MCP surface securely

Using the reusable contract, implement Stage 8: hosting, identity, and authorization for exactly
four tools and one resource.

## Hosting

Support two transports from one composition root:

- local stdio for a desktop client; and
- stateless Streamable HTTP at `/mcp`, authenticated, with no session affinity and no server-held
  caller state beyond the plan store.

Register the tool set statically. Fail startup if the registered surface is not exactly the four
tools and one resource, so a future refactor cannot quietly widen it.

## Caller identity

Under HTTP, validate the caller's bearer token in-process before any handler runs: signature,
issuer, audience, expiry, not-before, and required claims, with clock skew bounded. Reject
unsigned, `none`-algorithm, expired, or wrong-audience tokens. Cache signing keys with a bounded
lifetime and never fetch keys per request.

The caller's token authenticates the caller to this server. It is never forwarded to BrightFlag,
and BrightFlag's service token is never returned to a caller. Prove both directions in tests.

Under stdio, derive identity from the local configured principal and refuse to run in a mode where
no principal is established.

## Authorization

Define two capabilities:

- read approved invoices; and
- mark an invoice paid.

Grant them from reviewed server-side configuration keyed on validated claims. Never from a caller
argument, a tool description, an annotation, prompt text, or a model's assertion. Deny by default,
evaluate per call including the execute step of a plan, and record every decision in the audit log.

The plan store is caller-scoped, capacity-bounded, and expiring. A plan issued to one caller is
invisible and unusable to another.

## Hardening

- Enforce request-size, concurrency, and per-caller rate limits, with a stricter limit on the
  payment tool.
- Emit structured logs with a correlation identifier and no credential, token, or full payload.
- Reject unknown MCP arguments rather than ignoring them.
- Return typed errors that do not leak origin, path, query, header, or stack detail.
- Treat all BrightFlag response content as untrusted data, never as instruction.

## Tests

Prove:

- the registered surface is exactly the four tools and one resource, and startup fails otherwise;
- unauthenticated, expired, wrong-audience, wrong-issuer, and `none`-algorithm tokens are refused;
- a read-only caller cannot plan or execute a payment;
- a caller cannot execute another caller's plan;
- caller tokens never reach the outbound BrightFlag request and the BrightFlag token never reaches
  a response;
- rate limits and size limits are enforced; and
- text embedded in a BrightFlag response that instructs the server to change behavior changes
  nothing.

## Acceptance criteria

- Both transports serve the same narrow surface with the same authorization outcome.
- Authorization is server-side, deny-by-default, and re-evaluated at execution.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the decision to validate caller tokens
in-process and keep caller identity separate from the BrightFlag service credential. Do not push
unless requested.
