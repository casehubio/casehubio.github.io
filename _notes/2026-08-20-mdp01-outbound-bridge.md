---
layout: post
title: "The outbound bridge — routing channel messages to external A2A agents"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-qhorus]
tags: [a2a, outbound, delivery-pump, channel-backend]
series: issue-396-a2a-interop-audit
---

Qhorus has had an inbound A2A path for a while — external agents send JSON-RPC requests, `A2AResource` dispatches them into channels, and the commitment lifecycle tracks the work. What was missing was the reverse: a channel message targeting an external agent gets forwarded via HTTP, and the response arrives back as a channel message.

We built this as `a2a-outbound/`, a new optional module that activates by classpath presence — same pattern as `slack-channel/` and `connector-backend/`. The module has three components and a clean separation between them.

**A2AInstanceResolver** does the target lookup. When a message has a `target` field, the resolver checks `ExternalAgentBindingStore` for a matching instanceId. If it finds one, the message is destined for an external agent. If not, it's internal and the backend ignores it. Null and blank targets are short-circuited — no store lookup, no overhead.

**A2AResponseHandler** maps A2A task states back to qhorus speech acts. COMPLETED becomes DONE, FAILED becomes FAILURE, CANCELED becomes DECLINE. Working and submitted both map to STATUS. The interesting case is INPUT_REQUIRED — it maps to STATUS with a `{"input_required": true}` payload, giving the internal agent enough signal to respond without inventing a new message type. Content extraction follows the inbound mapping in reverse: TextParts concatenate into `content`, DataParts become `payload`, and artifact parts merge into the terminal message so they're visible on the channel.

**A2AOutboundBackend** ties everything together as an AT_LEAST_ONCE `ChannelBackend`. The delivery pump calls `post()` for every message on registered channels. Most calls are no-ops — the selective interception checks the target, and only messages aimed at external agents trigger an HTTP call. A sender-based loop guard prevents responses from being re-forwarded: if the message sender is itself an external agent (i.e. a response we just dispatched), the backend returns immediately.

One design detail worth noting: the protocol module's `AuthConfig` originally resolved tokens via `System.getProperty`/`System.getenv`, which doesn't work with the platform's `CredentialResolver` SPI. Rather than create a parallel auth path in the outbound module, we extended `AuthConfig` with a `resolvedToken` field. The outbound backend resolves credentials through `CredentialResolver` at call time and passes the pre-resolved token directly. The two-arg constructor is preserved — existing code that uses `System.getProperty` resolution is unaffected.

The pump cycle depth is bounded at two: original message dispatched outbound, response dispatched back to the channel, pump picks up the response, sender guard fires, done. No unbounded recursion. The commitment state machine handles the rest — DONE fulfills the commitment, FAILURE fails it, DECLINE declines it. The outbound bridge is invisible to the normative layer above it.

What's left is the integration test and the CDI wiring that makes the module deployable in a real Quarkus app. That's next.
