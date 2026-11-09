---
layout: post
title: "Extend Was Incomplete"
date: 2026-05-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [sla, breach-policy, cdi, events]
---

`BreachDecision.Extend` has been in the sealed hierarchy since we added `SlaBreachPolicy` — a record that says "give them more time." Until today, returning it from your policy did nothing. The breach executor didn't know what to do with it.

We added `WorkItemService.extend(id, newExpiresAt, actorId)` and wired it in. The validation is simple: non-terminal status (409 if the item is already cancelled or completed), and `newExpiresAt` strictly after the current deadline. Claude caught the null guard during review — `item.expiresAt` is always set at creation, but `!newExpiresAt.isAfter(item.expiresAt)` throws if it isn't. One condition fixes it: `item.expiresAt != null &&` before the comparison. That would have been a 500 on a WorkItem without a deadline.

Adding the `DEADLINE_EXTENDED` lifecycle event surfaced a trap that's now a protocol.

`WorkItemLifecycleEvent.eventType()` resolves the string via:

```java
try {
    return WorkEventType.valueOf(name.toUpperCase());
} catch (final IllegalArgumentException e) {
    return WorkEventType.CREATED;
}
```

Any string not in the enum falls through as `CREATED`. No compile error. No runtime warning. Every CDI observer that switches on `WorkEventType` handles the unknown event as an item creation — silently, consistently wrong. We'd have had `DEADLINE_EXTENDED` events routing through breach logic and notification hooks as if a new WorkItem had just been created.

The fix is one enum constant. The rule is now written: add the `WorkEventType` value in the same commit as the service method. We formalised it as a protocol because it's exactly the kind of thing that gets missed on a quick branch.

The other two fixes were smaller. One test class's `@BeforeEach` cleanup deleted templates without touching the work items and audit entries that reference them — a latent isolation failure when multiple test classes share a Quarkus session. The fix is FK-order deletion: audit entries first, then work items, then templates. And `casehub-work-core` was missing a `@DefaultBean NoOpRoutingCursorStore` — `RoundRobinStrategy` injects `RoutingCursorStore`, but the only implementation is in the runtime's JPA layer. Drop core onto a classpath without the runtime and Quarkus fails to boot. The no-op returns index 0 and yields to the JPA implementation when the runtime is present.

Four fixes. The extend path is closed. The event type fallback is written down.
