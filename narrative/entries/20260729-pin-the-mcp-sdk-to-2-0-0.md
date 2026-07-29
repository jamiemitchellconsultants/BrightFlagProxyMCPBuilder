---
date: 2026-07-29
slug: pin-the-mcp-sdk-to-2-0-0
title: "Pin the MCP SDK to 2.0.0"
summary: "Pin 2.0.0 and say so in the contract rather than leaving \"the stable official SDK\" to be resolved per stage."
kind: product
status: accepted
sequence: 2026-07-29T04:45:41.000Z
evidence: "https://github.com/jamiemitchellconsultants/BrightFlagProxyMCPBuilder/pull/15; merge commit c9cc4bfa5cb98f2bab7b9659456757180c5c1f61"
---

## Context

The reusable contract requires "the stable official MCP C# SDK" and pinned NuGet versions verified
against current primary documentation. When the sequence was planned, nuget.org's stable line was
`ModelContextProtocol` 1.4.1 and 2.x existed only as `2.0.0-rc.2`. The plan pinned 1.4.1 and recorded
the condition explicitly: verify again before pinning, and if 2.x has gone stable by then, that is a
reviewed change rather than a silent one.

Stage 5 reached the pin and found 2.0.0 released and stable. Taking it inside the stage would have
been exactly the silent upgrade the plan warned against, so the stage shipped on 1.4.1 and reported
the discrepancy instead.

## Decision

Pin 2.0.0 and say so in the contract rather than leaving "the stable official SDK" to be resolved
per stage. The decision record keeps the history of why 1.4.x came first, so a reader can see that
the pin moved deliberately and on a stated condition rather than drifting.

The discipline is unchanged and restated: a newer stable line is another reviewed change, not a
default.

Rejected: leaving the contract version-free and letting each stage resolve "stable" at
implementation time, which is how two stages end up on different majors without anyone deciding
that; and silently bumping the pin inside Stage 5, which would have buried an architecture decision
in a feature diff.

## Consequences

Stage 5 shipped once on 1.4.1 and moved to 2.0.0 in a second, separate commit, so the reviewed change
is visible in the history rather than folded into the read capability.

The move cost no source change: every SDK API the server uses — the tool type and tool attributes
with their annotation properties, `AddMcpServer().WithTools`, `McpServer.Create`, the stream
transports on both sides, and `CallToolResult` — is unchanged across the major version. Later stages
that touch hosting inherit the 2.x line from the contract rather than rediscovering it.

Stage 8 introduces `ModelContextProtocol.AspNetCore`. It is named at the same version here so the
hosting stage does not have to make this decision again under time pressure.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
