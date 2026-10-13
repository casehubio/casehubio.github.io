---
layout: post
title: "Closing the push gap"
date: 2026-08-13
entry_type: note
subtype: diary
projects: [casehub-fsitrading]
tags: [websocket, pages-push, quinoa, vertical-slice]
series: issue-19-market-pulse
---

The previous session built the Market Pulse pipeline — five summarisation levels, synthetic tick generation, LLM-powered regime assessment, REST endpoints, arena integration. All of it working. But the push path — the thing that makes it a live trading panel instead of a polling API — was hollow. `FsiMarketPushService` existed as a standalone POJO with a test double for the broadcaster. No WebSocket endpoint. No CDI wiring. No browser could connect.

The plan called the final task "Minimal UI Panel" and scoped it as Quinoa setup plus a static web component. Reading the decision record (D1: full vertical slice), that didn't match. D1 says "prove the end-to-end push path works." A panel that renders HTML but never connects to a WebSocket proves the build tooling, not the architecture.

## What was actually missing

The pages-push library is deliberately transport-agnostic. `EventBroadcaster` distributes events; `TopicRegistry` manages subscriptions via a trie with colon-separated segments and wildcard matching (`market:ticks:*`); `SessionSender` is the escape hatch where the application plugs in its transport. The library doesn't include a WebSocket endpoint — that's the application's job.

So the real work was three pieces the plan didn't account for:

`FsiPushConnectionRegistry` implements `SessionSender` — a `ConcurrentHashMap` of WebSocket connections, keyed by connection ID. When `EventBroadcaster` calls `send(connectionId, message)`, the registry finds the right connection and pushes. Simple, but without it, broadcast events have nowhere to go.

`FsiPushWebSocket` is the Quarkus WebSocket Next endpoint at `/ws/push`. It handles the pages-push wire protocol: clients send `{"op":"listen","id":"1","topics":["market:ticks:*"]}`, the endpoint registers them in the `TopicRegistry`, sends back an ack. On close, it cleans up both registries. The protocol supports wildcard subscriptions — a client listening on `market:regime:*` gets regime assessments for every instrument without knowing which instruments exist.

The CDI bridge: `MarketPulseConfiguration` produces `FsiMarketPushService` backed by `EventBroadcaster`, and `MarketPulseScheduler.onStart()` subscribes it to all five level buses. Ticks flow from the scheduler through the pipeline, through the push service, through the broadcaster, through the registry, to the browser.

## The ObjectMapper surprise

The first build with push wiring produced background stack traces in the Quarkus test output — tests still passed, but `EventBroadcaster` was silently failing to serialize `PriceTick` records. The `casehub-pages-push-runtime` module provides a `@DefaultBean` `JsonWriter` backed by `new ObjectMapper()` — a bare instance with no `JavaTimeModule`. Any record with an `Instant` field (which is every domain type in the pipeline) throws `InvalidDefinitionException` on serialization.

The fix: override the `@DefaultBean` with a producer that injects the Quarkus-configured `ObjectMapper`, which already has all the right modules registered. One line of CDI wiring, but the failure mode was genuinely hard to trace — the exception was thrown inside a bus subscriber callback during scheduled tick generation, with no obvious connection to the push infrastructure.

## The panel

With the transport layer working, the panel itself is straightforward TypeScript. A web component that opens a WebSocket, sends a listen for `market:ticks:*`, `market:bars:*`, and `market:regime:*`, then updates a table with live prices, regime badges, and SVG sparklines as events arrive. Quinoa handles the esbuild bundling inside the Maven build — `npm install` and `npm run build` happen automatically during `quarkus:build`.

The whole push path — from synthetic tick generation through five summarisation levels to a browser table cell — runs in a single process. `EventStreamBus` publishes; bus subscribers fan out to the observation cache, the push service, the channel bridge, and downstream runners. The push service calls `EventBroadcaster.broadcast()`; the broadcaster serializes to JSON, walks the topic trie for matching connections, and calls `SessionSender.send()` on each. The data reaches the browser as `{"op":"event","topic":"market:ticks:AAPL","payload":{...},"seq":42}`.

This is D1 delivered. The minimal panel will get replaced by C5's dock-workbench composition, but the transport layer, the CDI wiring, and the wire protocol are the real infrastructure — and those carry forward.
