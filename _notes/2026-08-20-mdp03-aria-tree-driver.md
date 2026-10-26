---
layout: post
title: "Closing the loop — ARIA dispatch for the scenario executor"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-pages, casehub-connectors]
tags: [aria, scenario-executor, push-wire, testing]
---

The scenario executor could dispatch GraphQL steps to backend services but ARIA steps were still a no-op — `ExecutionResult.ok(as.name(), Map.of())`. The browser-side handler was ready: `scenario-handler.ts` listens on `scenario:exec`, dispatches to the ARIA executor functions (click, fill, select, navigate), and sends back `CommandResult`. What was missing was the server-side dispatcher that sends commands and correlates responses.

The prior art sat right there in `AriaCommandBridge`. It uses `CompletableFuture` keyed by correlation ID, broadcasts via `EventBroadcaster`, waits for the browser's response. Clean pattern, works in tests. But it had a problem I hadn't noticed until Claude and I traced the references: `handleResult()` — the method that completes the future when a browser response arrives — had zero production callers. Every call site was in a test. The push wire sends commands out, the browser sends responses back over WebSocket, and the server-side WebSocket handler parses them correctly. It just doesn't route `command-result` ops to anything.

We needed to solve this before building the AriaDispatcher. The fix was a `CommandResultHandler` functional interface in the push module — same pattern as `SessionSender`, which already works this way. A `@DefaultBean` CDI implementation fires `Event<PushRequest.CommandResult>`. Any number of observers can listen. The hosting app's WebSocket handler injects `CommandResultHandler` and calls `handle()` when it parses a `command-result` message — one line of wiring.

The AriaDispatcher itself is straightforward once the routing exists. `@Observes PushRequest.CommandResult` completes the matching future. The interesting part was the navigate protocol: when the browser navigates to a new URL, the page tears down the push wire connection. The scenario handler is gone. The new page loads, creates a new connection, and resubscribes to `scenario:exec`. During that window, the AriaDispatcher needs to wait. It sends `{ action: "ready" }` probes at 500ms intervals — the existing `scenario-handler.ts` already handles `ready` as a no-op that returns `ok`. When a probe gets a response, the new page is live. When it doesn't, the browser is still loading. Simple polling, no browser changes needed.

Batching was the last piece. The ScenarioExecutor looks ahead for consecutive unnamed, non-navigate ARIA steps and groups them into a single `sendBatch()` call. Named steps break the batch because their `CommandResult.result` needs to feed into `VariableContext` for subsequent step interpolation. Navigate and wait steps break it because they have special execution semantics. The batching criteria (`isBatchable`) are four lines of code, but they encode the design decision about which ARIA steps are fire-and-forget versus data-producing.

One thing Claude's review caught: the original `executeAria` was passing the raw `AriaStep` to the dispatcher without resolving `${...}` variable references. An ARIA fill action referencing `${create-case.caseId}` as its value would have sent the literal template string to the browser. The fix creates a resolved copy of the step — walking `AriaTarget.name`, `value`, and `state` through `VariableContext.resolve()` before dispatch.

With this, the three delivery types in the scenario executor all work: `graphql` dispatches to backend services via HTTP POST, `aria` dispatches to the browser via push wire, and `simulated` remains a no-op placeholder for data pipeline injection. The branch covers the full arc from connectors MCP domain (#96) through scenario executor (#323) to ARIA dispatch (#324). What comes next is wiring it to real scenarios — the helpdesk example needs `@McpDomain` resolvers on its example app before the scenario executor can target it. That's examples#49, a separate branch.
