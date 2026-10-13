---
layout: post
title: "Three Layers for the Outside World"
date: 2026-08-19
entry_type: article
subtype: diary
projects: [casehub-qhorus]
tags: [a2a, interoperability, architecture, protocol]
---

Qhorus has always been richer than A2A. Typed channels with declared semantics, a ten-type speech act taxonomy, commitment tracking with attestation, normative governance — it's architecturally deeper than anything the emerging interop standards define. The problem wasn't capability. The problem was isolation.

An embabel-agent can't talk to a qhorus agent. A LangChain pipeline can't delegate work to one. The A2A protocol is becoming the lingua franca for cross-framework agent communication, and qhorus had partial support — you could receive an A2A message and dispatch it into a channel — but the outbound path didn't exist. Worse, the engine's existing outbound A2A code (`A2AWorkerFunctionHandler`) bypassed the mesh entirely. It called HTTP directly, invisible to the ledger, invisible to commitment tracking, invisible to observers. A delegation to an external agent was a black box.

I wanted to fix this properly, not patch it.

## The decomposition that mattered

The key question wasn't "where does the bridge go?" — it was "what IS A2A, separated from what qhorus and the engine each do with it?"

A2A bundles two concerns: a wire protocol (JSON-RPC, SSE, agent cards) and a task lifecycle model (submitted → working → completed/failed/canceled). The wire protocol is transport — it belongs with the communication layer. The task lifecycle maps directly to qhorus commitments, which already track obligations from open through fulfilled/declined/failed/expired. So the lifecycle model isn't new machinery — it's a mapping.

That decomposition gave us three layers:

**Layer 0** is a pure Java protocol module — `casehub-a2a-protocol`. A2A data types (`A2APart` as a sealed interface with `TextPart`, `DataPart`, `FilePart`), a JSON-RPC 2.0 HTTP client, agent card model. Zero CDI, zero Quarkus, zero platform dependency. Jackson for JSON, `java.net.http.HttpClient` for transport. Any Java project can use it.

**Layer 1** is the qhorus bridge. The existing inbound path (`A2AResource` + `A2AChannelBackend`) gets refactored to use Layer 0 types — dropping the inline records that were crammed into the JAX-RS resource class. A new outbound module (`casehub-qhorus-a2a-outbound`) implements `ChannelBackend` with `AT_LEAST_ONCE` delivery via the existing delivery pump. When a channel message targets an external A2A agent, the backend intercepts it, forwards via the protocol client, and dispatches the response back as a channel message. The external delegation becomes a normal channel conversation — ledger recorded, commitment tracked, observers notified.

**Layer 2** is the engine, which eventually becomes a pure channel consumer. Instead of calling `A2AClient.send()` directly, it dispatches a COMMAND or HANDOFF on a qhorus channel. The outbound backend handles the rest. Six classes in the engine's `a2a` package become unnecessary. That migration is a follow-on issue — the architecture supports it, but we build the qhorus side first.

## External agents as first-class instances

External A2A agents are registered in the qhorus instance registry — same discovery, same capability tags, same addressing as internal agents. Their A2A-specific routing information (endpoint URL, auth config key, protocol version) lives in an `ExternalAgentBinding` entity, following the `ChannelConnectorBinding` pattern: a lightweight join entity that references Instance by ID without modifying the Instance entity itself. The instance stays clean of protocol concerns.

The decision to use a binding entity rather than instance metadata fields came out of the design review. The Instance entity has no metadata storage — adding it would have changed a core domain entity across four store implementations, all builders, all test utilities. Disproportionate to the need. The binding entity is the same scope as `ChannelConnectorBinding` — a well-tested pattern.

Capability-based routing works naturally. `target: "capability:code-review"` resolves through existing instance capability tags. If the resolved instance has an `ExternalAgentBinding`, the outbound backend routes it via A2A. Internal and external agents are indistinguishable at the targeting layer.

## Structured data earns its own field

A2A messages carry typed parts — text, structured data (JSON), files. Qhorus had `content` (a String) and `artefactRefs` (lifecycle-managed references to SharedData). Structured data had no proper home — it got crammed into `content` as a JSON string, and consumers guessed whether to parse it.

I added a `payload` field to Message. Three distinct concerns, three distinct fields: `content` is what the message says (text), `payload` is what it carries (structured data), `artefactRefs` is what it references (long-lived resources with claim/release lifecycle). The governance infrastructure — watchdogs, commitment tracking, ledger analysis — operates on `content` only. Payload is opaque to governance, which is intentional: Jaccard similarity on JSON objects produces noise, not signal.

The field is a nullable `String` with `@JdbcTypeCode(SqlTypes.JSON)` for dialect portability — JSONB on PostgreSQL, TEXT on H2. Keeping it as `String` rather than `JsonNode` means `qhorus-api` stays free of Jackson as a compile dependency. The A2A bridge layer, which already has Jackson, handles the conversion at the protocol boundary.

Propagating this through the codebase was the kind of work that looks simple in design and turns into a grind in practice — Message, MessageDispatch, OutboundMessage, MessageReceivedEvent, NormalisedMessage, MessageView, the entity layer, the mapper, the MCP tools, and several hundred test call sites that use the canonical record constructors.

## What the agent card becomes

The existing agent card at `/.well-known/agent-card.json` was static — four hardcoded skills describing infrastructure primitives like "Channel Messaging" and "Shared Data Store." An external orchestrator reading that card learns what the bus does, not what the passengers can do. It can't discover that a code reviewer or a fraud detection agent is available.

The new design has two tiers. A platform directory card at `/.well-known/agent.json` (path corrected per the A2A spec) describes qhorus as a multi-agent platform with an `agents` array listing per-agent card URLs. Per-agent cards at `/.well-known/agents/{instanceId}.json` describe specific registered agents and their capabilities. External A2A agents get proxied cards — fetched from their real endpoint, cached. An orchestrator discovers the platform, drills into specific agents, and delegates to the one with the right capabilities.

## The architecture that fell out

The three-layer decomposition wasn't the initial proposal. I started with "keep the split — inbound in qhorus, outbound in engine." That broke under a simple question: when would qhorus be used without the engine? In the CaseHub ecosystem — never. Duplicating an A2A client for a deployment mode that doesn't exist is waste.

But "move everything into qhorus" mixed orchestration concerns (timeout, retry, artifact accumulation) into the communication layer. The engine's `A2AWorkerFunctionHandler` has execution semantics that don't belong in a communication mesh.

The answer was to recognise that qhorus already has equivalents for every execution concern the engine provides — `Commitment.expiresAt` for timeout, the delivery pump for retry, `SharedData` for artifact management. The engine doesn't need its own A2A transport. It delegates via channels, and the mesh handles the rest. Every external delegation gets the same normative infrastructure as an internal one.

That's the architecture that matters: external agents aren't second-class citizens patched in through a side channel. They participate in channels with the same speech acts, the same commitment lifecycle, the same ledger, the same observers. The bridge is transparent — which is what a communication mesh should provide.
