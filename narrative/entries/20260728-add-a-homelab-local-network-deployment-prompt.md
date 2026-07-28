---
date: 2026-07-28
slug: add-a-homelab-local-network-deployment-prompt
title: "Add a homelab and local-network deployment prompt"
summary: "Add a dedicated stage that turns the packaged server into a reproducible, encrypted, single-host LAN deployment with explicit identity, secret, network, validation, upgrade, and rollback procedures."
kind: product
status: accepted
sequence: 2026-07-28T12:00:00.000Z
---

## Context

Prompts 8 and 9 made local deployment possible: they added Streamable HTTP, a guarded local caller
identity trust provider, a hardened container, and an operational runbook. They did not require a
coding agent to assemble those pieces into instructions that a homelab operator could follow from a
clean host. Important choices such as the supported container runtime, LAN binding, TLS termination,
firewall boundary, identity bootstrap, secret-file permissions, persistence, client verification,
upgrade, and rollback were therefore left to the learner.

That gap is especially risky for this server because a private LAN is not automatically a trusted
transport or authorization boundary, and the only write capability changes invoice payment state.
A superficially convenient deployment could expose unencrypted HTTP, put credentials in Compose,
publish the service through a router, or treat the local development identity provider as suitable
for a live organisational deployment.

## Decision

Insert Prompt 10 after packaging and before the independent audit. Require it to generate and
validate an operator-ready, single-instance Windows 11, Docker Desktop, WSL 2 Linux-container, and
PowerShell baseline for explicitly allowed LAN clients. The deployment uses immutable image digests,
container hardening, HTTPS through a local reverse proxy or trusted private overlay, narrow Windows
interface and Defender Firewall rules, NTFS-protected external secret files, and the same JWT
validation path established by Prompt 8.

Keep private local-identity signing material and the dev token issuer outside the delivered server
image. Make the local identity path explicitly non-production, bootstrap read-only access first,
and require a separate deliberate step before payment authorization. Require exact procedures for
preflight, startup, network isolation, authentication failures, MCP-surface verification,
redacted diagnostics, upgrade, rollback, and removal. Automate every safe check without contacting
a live BrightFlag tenant, and label infrastructure checks that remain manual instead of treating
them as passed.

Move the independent reconstruction audit to Prompt 11 and add an explicit deployment verification
point so the new stage is independently checked rather than trusted because documentation exists.

## Consequences

A learner now gets a bounded deployment recipe instead of having to invent the security-sensitive
glue between the packaged image and a LAN MCP client. The primary supported target is intentionally
narrow: one Windows 11 host, Docker Desktop with its WSL 2 Linux-container backend and Compose, one
server instance, and named local clients. Windows Server, Windows containers, Podman, NAS appliances,
orchestration platforms, multiple instances, and public-internet exposure remain unsupported unless
a later decision adds and tests them. Docker Desktop's interactive startup and reboot behavior must
be proven or recorded as a manual availability gate rather than hidden behind a container restart
policy.

The guide can support functional investigation with local JWT issuance, but it cannot turn that
development trust root into production identity merely by adding TLS or a firewall. Moving to a
live organisational deployment still requires Prompt 9's identity-provider migration and corporate
alignment work.
