---
layout: post
title: "The evaluation that became a compliance fix"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [event-hierarchy, lifecycle-protocol, groupstatus, multi-instance]
---

The question behind #279 was whether `WorkItemGroupLifecycleEvent` should integrate into the `WorkItemEvent` hierarchy — share a common base, implement the interface, or remain standalone. It was filed during the #278 API surface cleanup as a conscious deferral: "revisit if a concrete use case requires polymorphic observation."

I started by mapping every observer in the system. The answer came quickly: no observer handles both event types polymorphically. Every consumer that subscribes to both — WorkCloudEventAdapter, the engine's WorkItemLifecycleAdapter — uses separate methods with completely different logic. The field overlap is three infrastructure concerns (callerRef, tenancyId, occurredAt), not domain identity. WorkItemEvent centres on a WorkItemRef with 8 fields describing a single WorkItem's state. WorkItemGroupLifecycleEvent centres on aggregate counters — parentId, groupId, and four integers tracking M-of-N threshold progress. These are different domain concepts that happen to live in the same module.

The CDI observation semantics seal it. If WorkItemGroupLifecycleEvent implemented WorkItemEvent, every `@ObservesAsync WorkItemEvent` observer would receive group events. The engine's adapter — which just migrated to observing the WorkItemEvent interface in #585 — would get group events on its individual-WorkItem handler. That's not a theoretical concern; it's a concrete routing bug.

Standalone confirmed. But the evaluation surfaced something else.

GroupStatus has been in production since Chapter C23 — three values, clear terminal semantics — and it was never registered in the lifecycle coherence protocol. No `isTerminal()`, no `isActive()`, no LIFECYCLE.md entry. A compliance gap hiding in plain sight since April.

Worse, GroupStatus was being derived from raw fields in three separate locations: `WorkItemInstancesResource`, `JpaWorkItemStore.scanRoots()`, and `MultiInstanceGroupPolicy`. Each one reconstructed the status from `policyTriggered` plus count comparisons — the same ternary expression copy-pasted across the codebase. The right fix wasn't just adding the lifecycle methods. It was persisting `groupStatus` on `WorkItemSpawnGroup` and making `MultiInstanceGroupPolicy` the single authority that sets it.

The Flyway migration needed conditional backfill — not a blanket `DEFAULT 'IN_PROGRESS'`, which would corrupt every existing terminated group. The design review caught that, along with the need for a nullable column (non-multi-instance spawn groups have no lifecycle state machine).

Claude's final review caught the most interesting bug: the MongoDB store's `put()` method uses an explicit `findOneAndUpdate` field list for OCC updates. Every field in the `Updates.combine()` block is listed by name. Adding a field to the entity and the document mapper isn't sufficient — the update path silently drops it. The field persists on first insert and reverts to its initial value on every subsequent update. I submitted that to the garden as GE-20260701-5b2584; it's the kind of silent data loss that only shows up when the value should have changed but didn't.

GroupStatus is now registered in LIFECYCLE.md alongside WorkItemStatus, PlanItemStatus, and CommitmentState. The engine consumer migration — replacing `status != COMPLETED && status != REJECTED` with `!status.isTerminal()` — is tracked in engine#624.

The evaluation started as a question about type hierarchy. The answer was no. The value was in what it uncovered along the way.
