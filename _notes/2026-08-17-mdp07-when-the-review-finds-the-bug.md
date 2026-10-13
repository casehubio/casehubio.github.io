---
layout: post
title: "When the Review Finds the Bug in the Issue"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-fsitrading]
tags: [deliberation, websocket, push, cdi-events, design-review, architecture]
series: issue-26-deliberation-websocket-push
---

The C3 deliberation spec deferred WebSocket push to a follow-up issue. The implementation looked straightforward — wire lifecycle events (start, complete, fail) to `PushBroadcaster` topics, add per-round convergence updates. Two topics, four event types, mirror the existing market data push pattern.

The first design question was whether per-round convergence updates needed a cross-repo change. `ConversationOrchestrator` in casehub-blocks runs the debate loop, and it has no hook for observing state after each round. I filed blocks#125 to add a `RoundListener` SPI. Seemed clean — add the hook upstream, consume it downstream.

## The Wrong Execution Path

The decision review caught it. `FsiDeliberationStateObserver` — the class that bridges `MessageObserver` and `EventSource` for the `ChoreographedDriver` — is called on every channel message and already holds `ConversationState`. It's the natural per-round hook. But more importantly, fsitrading's deliberation doesn't use `ConversationOrchestrator.converse()` at all. It uses `Patterns.debate()` with `ExecutionBackend.choreographed()`, which builds an `ExecutionModel` consumed by a separate driver. `DebateBuilder` never references `ConversationOrchestrator`. The SPI I filed would have added a hook to a class that fsitrading's debate path doesn't touch.

The observer was already there. `CommonGroundAnalyser.analyse()` and `ConvergenceAnalyser.analyse()` are static methods — pure computation over `ConversationState`, no LLM calls, no I/O. Adding convergence analysis and broadcast to `onMessage()` gives message-level granularity with no cross-repo dependency. We re-scoped blocks#125 as a general improvement rather than closing it.

## The CDI Event Precedent

The second finding was about transport. I'd chosen direct calls — orchestrator injects push service, calls broadcast methods. Simple, testable. The review pointed out that C2's Market Pulse decisions (D4) explicitly chose CDI events for pipeline-to-arena decoupling. Dismissing CDI events for deliberation-to-push is inconsistent with an architectural choice made one branch earlier on the same project.

The "one consumer" argument didn't hold either. The spec already names `commitment-viz`, `blocks-timeline`, and `channel-activity` as consumers, and C4 will need deliberation failure events for escalation. CDI events are the established pattern. We switched.

The final design: `FsiDeliberationOrchestrator` fires CDI domain events (`DeliberationStartedEvent`, `DeliberationCompletedEvent`, `DeliberationFailedEvent`). A dedicated `FsiDeliberationPushListener` observes them and broadcasts to `deliberation:active` and `deliberation:{channelId}`. The observer handles per-round convergence directly — it computes common ground and convergence on every message and broadcasts a `ConvergenceUpdate` payload. All type-discriminated with a `type` field for client dispatch.

The review also caught the topic naming convention — market data uses colons (`market:ticks:AAPL`), the spec used slashes (`deliberation/{channelId}`). Colons for consistency.

The interesting thing about this branch isn't the implementation — it's four commits of straightforward wiring. The interesting thing is that the design review found a dependency that would have been wrong, and an inconsistency with a decision made a week earlier. Both would have shipped unnoticed. The cross-repo issue would have sat open, someone would eventually have implemented the SPI, and it wouldn't have helped.
