---layout: post

title: "The Reply That Never Came"
date: 2026-07-24
author: mdp
type: diary
entry_type: note
subtype: diary
projects: [CaseHub Engine]
tags: [virtual-threads, reactive, event-bus, architecture, migration]
series: issue-381-virtual-thread-migration
status: draft
---

*Part of a series on [#770 — Complete virtual thread migration](https://github.com/casehubio/engine/issues/770). Previous: [Ripping Out Reactive](2026-07-23-mdp01-ripping-out-reactive.md).*

The previous session left me with a clean bill of health: production compile green, test-compile green, 255 files changed, 11,694 lines deleted. All that remained was a full test suite run.

The first module to fall over was runtime — a CDI wiring gap. `ChainedReactiveActionRiskClassifier` was deleted with the rest of the reactive stack, but nothing replaced it. `WorkflowExecutionCompletedHandler` injects `ActionRiskClassifier` with the `@Default` qualifier; the only beans available carry `@RiskClassifier`. A straightforward fix: `ChainedActionRiskClassifier` — same most-restrictive-wins chaining logic, no reactive wrappers.

Then the blackboard integration tests started timing out. Every one. Thirty failures, all identical: `(TIMEOUT,-1) Timed out after waiting 30000(ms) for a reply. address: __vertx.reply.1, repliedAddress: casehub.case.started`. The handler was running fine. The reply was simply never sent.

I went through the obvious fixes. Changed the handler return type from `void` to `String`. Still timed out. Switched from `@RunOnVirtualThread` to `@ConsumeEvent(blocking = true)`. Still timed out. Both together. Still timed out. Three different annotation combinations, same 30-second wall.

The problem wasn't which annotation to use — it was that `requestAndAwait` on the Vert.x event bus expects a reply, and Quarkus's `@ConsumeEvent` integration with `@RunOnVirtualThread` simply does not wire up the auto-reply mechanism. With `Uni<Void>` return, Quarkus subscribes to the Uni and sends the reply on completion. With `@RunOnVirtualThread void`, neither path activates. The handler executes; no reply is dispatched.

But the deeper question was why the event bus was involved at all. `CaseHubReactor` sends to `CASE_STARTED`; exactly one handler consumes it. Same for `SIGNAL_RECEIVED`, `BULK_SIGNAL_RECEIVED`, `TYPED_SIGNAL_RECEIVED` — four single-producer/single-consumer pairs. In the reactive world, `request().chain()` was the natural sequencing mechanism — it threaded asynchronous operations without blocking the event loop. In a blocking world, a direct method call does the same thing trivially.

The fix was architectural: inject the handlers directly and call them. No event bus request-reply. The handlers lose their `@ConsumeEvent` annotations — nobody else sends to those addresses. The fire-and-forget handlers (`CONTEXT_CHANGED`, `CASE_STATUS_CHANGED`) keep their annotations; `eventBus.publish()` doesn't need a reply.

The remaining failures were all incomplete migration — methods that still returned `Uni<Void>` but were called in a now-blocking context without subscription. `CaseResumptionService.resumeIfWaiting()` already did its work synchronously and wrapped the return in `Uni.createFrom().voidItem()`. `SchedulerService.registerScheduledTriggers()` built a list of `Uni<Void>` schedule operations that were never subscribed — lazy Uni, so the Quartz jobs were never created. Test helpers referenced deleted field names via reflection, verified old async method signatures, stubbed Vert.x locks that no longer existed. Six distinct root causes, twenty test failures, all traceable to "the reactive removal was incomplete in this spot."

The reactive-to-blocking migration has a shape: the obvious changes (delete interfaces, change return types, remove `Uni` wrappers) happen first and feel like the hard part. The subtle ones — a lazy Uni that was always subscribed by a reactive chain that no longer exists, a `requestAndAwait` that worked because the old handler returned `Uni<Void>`, a reflection-based test stub using a field name that changed — surface only when you run the full suite. The event bus reply gap is the most architectural of these: it's not a bug in the code, it's an incompatibility between two well-documented Quarkus features that simply don't compose.
