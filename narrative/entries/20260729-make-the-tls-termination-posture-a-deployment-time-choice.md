---
date: 2026-07-29
slug: make-the-tls-termination-posture-a-deployment-time-choice
title: "Make the TLS termination posture a deployment-time choice"
summary: "Stage 10 terminates TLS at a reverse proxy, settling a choice it had left open. Stage 8's HTTP transport gains an explicit deployment-time posture — direct termination or plain HTTP behind a named trusted proxy — with no silent default."
kind: architecture
status: accepted
sequence: 2026-07-29T20:21:48.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/21; merge commit 8af1f0b333fd18c8b0e1d4cfd883150e83bfc9ce"
---

## Context

Two related gaps, at opposite ends of the same request path.

Stage 10 offered the operator a choice of two TLS patterns — a reverse proxy on the same host, or a
trusted private overlay providing authenticated encryption — and documented both. Offering two patterns
means testing two, or claiming one untested. For a deployment guide whose value is that a learner can
follow it on a clean host and arrive somewhere known, an unresolved choice at the encryption boundary
is the wrong place to leave flexibility.

Stage 8, meanwhile, said nothing about where TLS terminates. It specified an authenticated Streamable
HTTP transport and left the transport-security posture implicit. That is survivable while every
deployment looks the same, and stops being survivable the moment two do not: a deployment fronted by a
load balancer or reverse proxy that has already terminated TLS has no supported way to say so, and the
server has no way to distinguish "plain HTTP because a trusted proxy handled encryption" from "plain
HTTP because someone misconfigured it".

The second reading is the dangerous one, because it is also the silent one. A server that accepts plain
HTTP without being told to is one configuration error away from serving an authenticated financial
capability unencrypted, and nothing in the stage would have refused.

## Decision

**Stage 10 terminates TLS at a reverse proxy on the same host.** The server container speaks plain HTTP
on the internal Compose network, only the proxy binds the LAN-facing TLS port, and the server's port is
reachable only from that proxy. The private-overlay pattern may be noted as an alternative for a reader
with different constraints, but it is not the documented, tested path.

**Stage 8's HTTP transport gains an explicit TLS posture, selected at deployment time**, because
different live deployments genuinely sit differently relative to termination:

- direct — the server terminates TLS itself; or
- fronted — the server serves plain HTTP on an interface reachable only from a configured proxy that
  has already terminated TLS.

Exactly one is selected explicitly and there is no third state. **Startup fails** if the server would
serve plain HTTP without the deployment explicitly declaring itself fronted. Plain HTTP is never an
accidental default; it is a thing a deployment has to assert, in the same spirit as the topology gate
in the preceding entries and Stage 3's refusal of a production profile with no authentication.

When fronted, forwarded-protocol information is honoured **only from the configured proxy hop**, never
from an arbitrary caller-supplied header. Trusting the header generally would hand any caller on plain
HTTP the ability to claim it had arrived over HTTPS, which converts the posture from a control into
decoration.

## Consequences

Stage 10's guide gets shorter and more testable: one pattern, with a proving test that the unencrypted
backend port is unreachable from another LAN machine. The alternative pattern survives as a note rather
than as a second thing to keep true.

Stage 8 gains a startup gate and a configuration surface it did not have, and the pairing of posture
with trusted hop means "fronted" cannot be enabled without naming what is trusted. The failure mode the
gate prevents — an authenticated financial surface served in the clear because nothing objected — is
the kind that is invisible until someone looks at a packet capture.

The fronted mode is what makes the later AWS deployment expressible at all: an ALB terminating TLS with
the task speaking plain HTTP inside the VPC is precisely this shape, and without the posture it would
have required either an unsupported configuration or a second code path.

This decision is confined to transport security. It does not touch caller identity, which is validated
in-process from a bearer token regardless of posture, and it grants no exemption anywhere: a fronted
deployment still authenticates every caller, and a LAN or VPC boundary is still not a substitute for
the deferred alignment with the deploying organisation's identity provider.

Two of the three commits carrying this work were stranded when the preceding pull request merged before
they arrived, and reached `main` only when recovered here — the reason this entry and the one before it
were both written after the fact rather than alongside their merges.
