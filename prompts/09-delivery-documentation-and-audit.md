# Prompt 9 — Package, document, and govern the server

Using the reusable contract, implement Stage 9: repeatable delivery for the completed surface.

## Build and packaging

- Add a container image built from a pinned, digest-referenced base, running as a non-root user
  with a read-only root filesystem and no build toolchain in the final layer.
- Expose only the HTTP transport port in the image.
- Exclude Prompt 8's dev token-issuing tool and local JWKS trust provider fixtures from this image;
  prove their absence rather than merely their inactivity.
- Produce a reproducible build and record the source revision.
- Emit a software bill of materials and fail the build on known-critical vulnerabilities.
- Add a health endpoint that reports readiness without contacting BrightFlag and without revealing
  the origin, tenant, or allow-list.

## Continuous integration

Add workflows that run restore in locked mode, format verification, build with warnings as errors,
tests, the ontology-schema drift check (the `schema check` command from Prompt 7), dependency and
secret scanning, and container build. Keep live BrightFlag sandbox tests behind an explicit opt-in
that is off in normal CI. Pin actions by commit SHA and grant least-privilege permissions.

## Configuration and secrets

Document every configuration key, its default, its bound, and whether it is a decision-bearing
change. Ship an example configuration that uses placeholders only. Add a startup validation pass
that fails on an unknown key, a missing allow-list, a non-HTTPS non-loopback origin, or a payment
tolerance outside its permitted range.

## Documentation

Write documentation that states:

- the three capabilities and the four BrightFlag operations, with the excluded operations named;
- the definitions of approved for payment and paid, with their evidence requirements;
- the plan-and-confirm payment lifecycle and the meaning of an ambiguous outcome;
- what an operator must do when a payment result is unknown;
- how the ontology schema is consumed by a separate ontology service;
- the tenant-configuration inputs an integrator must supply;
- the operational limits, including lookback, fan-out, page, and rate ceilings; and
- how to point the caller-identity trust provider at a real identity provider when moving from
  local development to a live deployment, and what happens under either profile if that step is
  skipped.

Add a security policy covering credential handling, the untrusted-content boundary, and reporting.
Add a threat model naming at least: a prompt-injected payment instruction, a replayed plan token, a
duplicated payment, a widened allow-list, a leaked service token, a compromised cursor-signing key,
unauthorized read access to the plan store, a local development identity-trust provider or
dev-token tool reachable in a production deployment, and an ontology schema carrying live data.

## Runbook

Document how to rotate the BrightFlag token, rotate the cursor-signing key, refresh the OpenAPI
snapshot and review its diff, change the approved-status allow-list, respond to a duplicate
payment, switch the caller-identity trust provider from local development to a live identity
provider, and roll back a release. Each entry names who approves it and what evidence is retained.

The cursor-signing key rotation entry must cover: generating a new key under a new key identifier;
deploying it so verification accepts both the new and the immediately prior key identifier, while
new cursors are only ever issued and signed under the new one; retiring the prior key identifier
from verification no sooner than the maximum cursor lifetime after cutover, so no cursor issued
under it is still outstanding; and the alternative scheduled procedure for a suspected-compromise
rotation, where the prior key identifier is retired from verification immediately instead, and
every caller with a cursor in flight simply receives a clean rejection and restarts pagination from
a fresh window, which is expected, tested behavior rather than an incident.

## Acceptance criteria

- The container builds, starts, serves `/mcp`, and passes health checks with no BrightFlag call.
- The dev token-issuing tool and local JWKS trust provider fixtures are verifiably absent from the
  built image.
- CI runs every gate named above and is green.
- Documentation matches the implemented surface exactly, with no aspirational capability.
- The threat model and runbook are specific to this server, not generic advice.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the delivery and governance decisions. Do not
push unless requested.
