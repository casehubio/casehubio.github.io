---
layout: post
title: "Three Abstractions Down to One"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [event-hierarchy, spi, refactoring, api-design]
---

The casehub-work event hierarchy had been bothering me since #275 extracted the WorkItemCreator SPI. That session introduced `WorkItemEvent` — a typed interface with `ref()` returning a `WorkItemRef` record — alongside the existing `WorkLifecycleEvent` abstract class with its `source(): Object`. Two different philosophies coexisting in the same module, and `WorkItemLifecycleEvent` implementing both.

The `Object` return was the real problem. Every caller — five of them — immediately downcasted to `WorkItem`. All five lived in modules that already depended on runtime. The type erasure bought nothing; it just hid information the compiler could have checked.

I decided to promote `WorkItemEvent` to be the single event contract and delete `WorkLifecycleEvent` entirely. No deprecation, no adapter — just remove it. The interface got four new abstract methods (`eventType()`, `occurredAt()`, `actor()`, `detail()`) and `WorkItemLifecycleEvent` already had matching concrete methods, so it compiled against the enriched interface without changes to its own method bodies.

The more interesting move was typing `FilterAction.apply()` from `Object` to `WorkItem`. Every implementation — `ApplyLabelAction`, `SetPriorityAction`, `OverrideCandidateGroupsAction` — opened with `final WorkItem workItem = (WorkItem) workUnit;`. Three unchecked casts doing nothing but converting a generic parameter into what everyone knew it was. The `Object` parameter was stranded generality from an era when the filter engine might have supported multiple work-unit types. It never did.

The SPI migration was mechanical but satisfying: fourteen consumer-facing interfaces moved from `io.casehub.work.api` flat to `io.casehub.work.api.spi`. This brings the api module in line with the `consumer-spi-placement` protocol that had been established platform-wide but never applied retroactively here. After the move, `api/spi/` has sixteen interfaces (the fourteen plus the two from #275), and `api/` has only value types, events, and utilities.

Template consolidation closed the loop on #275's `createFromTemplate()`. Three `instantiate()` overloads with positional arguments and three corresponding `toCreateRequest()` statics — all deleted. Every caller now builds a `WorkItemCreateRequest` and passes it to `createFromTemplate()`, which owns the template lookup, merge semantics, and label application. The REST endpoint and the schedule service both simplified: no template lookup on their side, no positional override juggling.

The one catch nobody's tests anticipated: `createFromTemplate()` fetches the template from the store, while the old `instantiate()` accepted the in-memory object directly. Five integration tests set template properties *after* `persist()` and relied on the in-memory modifications being visible. With the new path, those modifications never reached the database. The fix was straightforward — a `persistWith()` helper that applies a `Consumer<WorkItemTemplate>` before persist — but the silent failure mode was instructive. The assertions failed with null values, not exceptions, and nothing in the error pointed at the entity lifecycle as the cause.

The branch breaks the engine's `WorkItemLifecycleAdapter`, which observes `WorkLifecycleEvent` — a class that no longer exists. That's engine#585, and it's intentional: the migration forces the engine to observe `WorkItemEvent` instead, which gives it typed access to every field it was previously extracting through downcasts.
