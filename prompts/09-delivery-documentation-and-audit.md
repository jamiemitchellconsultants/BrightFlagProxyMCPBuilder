# Prompt 9 — Package, document, and govern the server

Using the reusable contract, implement Stage 9: repeatable delivery for the completed surface.

## Build and packaging

- Add a container image built from a pinned, digest-referenced base, running as a non-root user
  with a read-only root filesystem and no build toolchain in the final layer.
- Expose only the HTTP transport port in the image.
- Produce a reproducible build and record the source revision.
- Emit a software bill of materials and fail the build on known-critical vulnerabilities.
- Add a health endpoint that reports readiness without contacting BrightFlag and without revealing
  the origin, tenant, or allow-list.

## Continuous integration

Add workflows that run restore in locked mode, format verification, build with warnings as errors,
tests, the ontology-schema drift check, dependency and secret scanning, and container build. Keep
live BrightFlag sandbox tests behind an explicit opt-in that is off in normal CI. Pin actions by
commit SHA and grant least-privilege permissions.

## Configuration and secrets

Document every configuration key, its default, its bound, and whether it is a decision-bearing
change. Ship an example configuration that uses placeholders only. Add a startup validation pass
that fails on an unknown key, a missing allow-list, a non-HTTPS non-loopback origin, or a payment
tolerance outside its permitted range.

## Documentation

Write documentation that states:

- the three capabilities and the five BrightFlag operations, with the excluded operations named;
- the definitions of approved for payment and paid, with their evidence requirements;
- the plan-and-confirm payment lifecycle and the meaning of an ambiguous outcome;
- what an operator must do when a payment result is unknown;
- how the ontology schema is consumed by a separate ontology service;
- the tenant-configuration inputs an integrator must supply; and
- the operational limits, including lookback, fan-out, page, and rate ceilings.

Add a security policy covering credential handling, the untrusted-content boundary, and reporting.
Add a threat model naming at least: a prompt-injected payment instruction, a replayed plan token, a
duplicated payment, a widened allow-list, a leaked service token, and an ontology schema carrying
live data.

## Runbook

Document how to rotate the BrightFlag token, refresh the OpenAPI snapshot and review its diff,
change the approved-status allow-list, respond to a duplicate payment, and roll back a release.
Each entry names who approves it and what evidence is retained.

## Acceptance criteria

- The container builds, starts, serves `/mcp`, and passes health checks with no BrightFlag call.
- CI runs every gate named above and is green.
- Documentation matches the implemented surface exactly, with no aspirational capability.
- The threat model and runbook are specific to this server, not generic advice.
- Formatting, build, and tests succeed.

Commit locally. Use `narrative-required` and record the delivery and governance decisions. Do not
push unless requested.
