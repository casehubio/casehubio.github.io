---
layout: post
title: "Two Protocols, One Socket"
date: 2026-08-18
entry_type: note
subtype: diary
projects: [casehub-fsitrading]
tags: [pages, websocket, push, dsl, trading-desk, wire-protocol]
---

The Trading Desk is a dock-workbench — nine panels composed entirely via the pages TypeScript DSL, no custom web components beyond the existing Market Pulse panel. Positions, P&L heatmap, strategies with a split order view, KPIs as a metric grid, trust scores, routing decisions, deliberations, commitments, and a regime badge in the status bar. The composition is a single TypeScript file calling `dockWorkbench()` with zone assignments and data bindings.

The backend push infrastructure followed the pattern from the deliberation work. Three CDI event records — `PositionUpdatedEvent`, `TrustScoreChangedEvent`, `RoutingDecisionEvent` — fired from `PositionService.applyFill()` and the arena execution pipeline. A single `FsiTradingPushListener` observes all three and broadcasts to push topics. Sealed `TradingPushPayload` interface with type discriminators, same shape as `DeliberationPushPayload`. Mechanical work — the pattern was proven, applying it was straightforward.

The interesting part was wiring the frontend to receive those push events.

The pages DSL's data binding model uses `DataSourceBinding` — you bind a `DataSource` to a named dataset ID, and components query that dataset via `lookup()`. For real-time data, the natural choice is `composite(fetchSource(url), wsSource(wsUrl))` — REST for the initial load, WebSocket for live updates.

I tried `wsSource` first. Connected to `/ws/push`, subscription appeared to succeed, but no events arrived. Zero errors, zero data. Claude dug into the pages-data source and found the disconnect: `wsSource` uses the `subscribe` wire protocol, sending `{op: "subscribe", dataset: "name"}`. But `EventBroadcaster.broadcast()` on the backend sends `{op: "event", topic: "pattern", payload: {...}}` — the `listen` wire protocol. Same WebSocket endpoint, two completely different message routing paths. `processWireMessage()` in pages-data handles `op: "event"` messages by dispatching to a DOM EventTarget, not to dataset subscriptions. They share a socket and nothing else.

The fix was a custom `topicSource` adapter — a `DataSource` implementation that connects to `/ws/push`, sends `{op: "listen", topics: [...]}`, accumulates payloads into a row map keyed by a configurable field, and emits full snapshots on each update. Wraps in `composite()` the same way: `composite(fetchSource("/api/positions"), topicSource(["position:*"], { keyField: "instrument" }))`.

The other discovery was less consequential but still ate time. Quarkus Arc silently ignores `@Observes` methods on `@ApplicationScoped` beans in the test classpath. The bean itself is discovered — `@Inject` works, `assertNotNull` passes — but the observer method never fires. Production-classpath observers for the same event type work fine. The distinction is augmentation scope: Arc registers observers at build time from the application classpath only. No error, no warning, just zero events captured. The workaround is to test observer logic directly (unit test with a mock broadcaster) rather than testing CDI event delivery end-to-end.

The pages-ui barrel export for the new DSL builders from pages#317 — `heatmapChart`, `eventTimeline`, `badge` — was also missing from the `dsl/index.ts` re-export. Deep import from `@casehubio/pages-ui/dist/dsl/builders.js` works as a workaround. Filed as casehub-pages#321.

The desk is fully composed and builds. Layout persistence via a local `LayoutResource` backed by `casehub-pages-layout-sqlite` — the pages-layout module's own endpoint requires JWT auth, so for pre-release we exclude it and provide one with hardcoded tenant defaults.

What this opens up: the Trading Desk is the composition target for C4 (Overnight Ops). Once #27 delivers the backend ops capabilities, their panels slot into the existing dock-workbench with additional zone assignments. The `topicSource` adapter is reusable — any pages consumer that needs topic-based push events rather than dataset subscriptions can use it. That feels like it should be a pages-data built-in, not a per-consumer adapter.
