---
layout: post
title: "When every agent call is an accountable act"
date: 2026-08-20
entry_type: article
subtype: diary
projects: [casehub-qhorus]
tags: [a2a, architecture, governance, channels, delegation]
series: issue-396-a2a-interop-audit
---

Continued from [The outbound bridge](2026-08-20-mdp01-outbound-bridge.md).

Most multi-agent frameworks treat external agent delegation as an HTTP call. An orchestrator sends a request, waits for a response, and continues. The problem isn't that it doesn't work — it does. The problem is that it's invisible. No audit trail, no obligation lifecycle, no governance. The delegation happened, but the system has no memory of it beyond whatever the orchestrator chose to log.

Qhorus takes a different position: every interaction between agents — internal or external — is a channel message. That message gets a ledger entry, a commitment, and the full normative lifecycle. The transport is irrelevant. What matters is that the act is recorded, the obligation is tracked, and the response is accountable.

This is an architectural claim, not a feature announcement. It shapes how we built the A2A outbound bridge, and it's worth understanding why.

### The three layers

The A2A interop work splits into three layers with strict dependency boundaries:

**Layer 0** is `casehub-a2a-protocol` — pure Java, no CDI, no Quarkus, no platform dependency. It contains the A2A model types (AgentCard, A2ATask, the sealed A2APart hierarchy), a JSON-RPC 2.0 HTTP client, and nothing else. Both the engine and qhorus depend on it. Neither depends on the other.

**Layer 1** is qhorus — the communication mesh. Two bridges: inbound (`A2AResource` + `A2AChannelBackend`, which already existed) and outbound (`A2AOutboundBackend`, new). The inbound bridge turns external JSON-RPC requests into channel messages. The outbound bridge turns channel messages into external JSON-RPC calls. Both directions flow through the same channel infrastructure — same ledger, same commitments, same observer pipeline.

**Layer 2** is the engine — the orchestration consumer. It doesn't know about A2A at all. When it delegates work to an external agent, it dispatches a COMMAND on a channel with a target. The target resolves to an `ExternalAgentBinding`, and the outbound bridge handles the rest. The engine doesn't make HTTP calls, doesn't parse A2A responses, and doesn't manage retry or health tracking. It's a pure channel consumer.

The layering matters because it determines what changes when the A2A spec evolves. Protocol changes (new task states, new part types) are isolated to Layer 0. Transport changes (authentication, streaming, push notifications) are isolated to Layer 1. The engine never touches either.

### What channels give you that HTTP doesn't

When an engine function handler calls an external agent via direct HTTP, five things are true:

1. The call is invisible to the ledger — no tamper-evident record of what was asked and what was answered
2. There is no commitment lifecycle — no way to track stale delegations, no expiry, no watchdog alerting
3. Observers don't see it — WebSocket, Kafka, and webhook observers have no idea the delegation happened
4. Retry and health tracking are the caller's problem — the engine has to implement circuit breaking, backoff, and failure handling for each integration
5. Credentials are managed inline — `System.getenv("AGENT_TOKEN")` rather than a platform-wide credential resolver

When the same delegation flows through a channel, all five problems disappear. The COMMAND creates a ledger entry. A commitment opens automatically. Every MessageObserver — including the ones the deployer added for their own purposes — sees the outbound request and the inbound response. The delivery pump handles retry with per-backend health tracking. Credentials resolve through `CredentialResolver`, which supports static tokens, environment variables, and vault-backed implementations.

The engine's `A2AWorkerFunctionHandler` is roughly 200 lines of HTTP client, error handling, retry logic, and response parsing. The channel-based equivalent is a COMMAND dispatch with a target field. The orchestration logic doesn't change. The ceremony disappears.

### External agents as first-class participants

The deeper consequence is that external agents become indistinguishable from internal ones at the channel level. An internal agent receiving a COMMAND and an external agent receiving one go through identical paths — the same type constraints, the same ACL checks, the same protocol enforcement. The only difference is the backend that handles delivery: `QhorusChannelBackend` (in-process) versus `A2AOutboundBackend` (HTTP).

This is not a convenience. It's an architectural invariant. If external delegations bypass the channel, then the system's normative guarantees have a hole in them exactly where they matter most — at the boundary where trust is lowest and auditability is most valuable.

Making the transport invisible and the governance visible is the whole point.
