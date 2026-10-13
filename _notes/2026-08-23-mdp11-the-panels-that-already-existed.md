---
title: "The Panels That Already Existed"
date: 2026-08-23
author: Mark Proctor
series: casehub-fsitrading
entry_type: note
subtype: diary
tags: [blocks-ui, push-payloads, work-items, panels, platform-coherence]
---

# The Panels That Already Existed

I went into C4b thinking I needed to build eight UI panels from scratch. The issue says "wire blocks-ui panels to C4a backend" — but somehow the design process drifted toward building custom Lit components to replace three of them. The adversarial review caught it: work-item-inbox, work-item-detail, and approval-gate are production components in blocks-ui, used by three other CaseHub apps. Building custom replacements was a platform coherence violation hiding behind a reasonable-sounding rationale about "fewer components."

The corrected approach is simpler. Register the blocks-ui components via `hostPanel()`, point them at the C4a REST endpoints, and let them manage their own data subscriptions. The dock-workbench treats them like any other panel — `defaultOpen: false` in the right zone, available on demand.

## The real work was the push plumbing

C4a built the incident backend — case definition, SLA policy, risk classifier, thirteen response agents. But it broadcast raw CDI event records to the push topics. `FsiIncidentNotifier` was firing `IncidentCreatedEvent` objects directly through `EventBroadcaster`, while every other push listener in the app wraps events in typed sealed interfaces with a `type` discriminator.

The inconsistency matters on the client side. When three different event shapes arrive on the same `incidents/{caseId}` topic — created, SLA breach, resolved — the `topicSource` accumulator has no way to distinguish them. A type field solves client-side dispatch; `accumulate: false` solves the overwrite problem by treating the topic as an event stream rather than accumulated state.

So C4b adds `IncidentPushPayload` (three records) and `WorkItemPushPayload` (five records, including `GateOpened` which moved here from the incident domain — it creates a work item, so it belongs with work-item events). Both follow the `TradingPushPayload` pattern: sealed interface, type discriminator string, convenience constructor that auto-populates the discriminator.

## WorkItemLifecycleEvent has a confusing API

The platform fires `WorkItemLifecycleEvent` for every work item across every application. To filter for fsitrading's work items, you need `event.workItem().scope()` — there's no `scope()` on the event itself. And `workItem()` returns null for wire-deserialized events, so the null guard comes first.

Then there's `event.type()` vs `event.eventType()`. The first returns a CloudEvent URI string like `"io.casehub.work.workitem.created"`. The second parses it and returns the `WorkEventType` enum. A developer writing a switch statement would naturally reach for `type()` and get a string where they expected an enum. Each mistake compiles.

I captured this as a garden entry — it's the kind of thing that costs an hour the first time and zero the second.

## SLA deadlines belong server-side

The sla-indicator panel needs countdown timers. The CDI events didn't carry deadline timestamps — `IncidentCreatedEvent` had severity but not the actual `Instant` when the claim window expires. Computing deadlines client-side would duplicate the `IncidentSeverityDescriptor` logic in TypeScript and break the moment SLA windows change.

The fix: `FsiIncidentTrigger` now computes `claimDeadline` and `completionDeadline` at incident creation time (from the descriptor's `Duration` plus `Instant.now()`), stores them in `IncidentRecord`, carries them through the CDI event, and delivers them in the push payload. When `FsiSlaBreachPolicy` escalates and the work item's candidate group changes, `FsiWorkItemPushListener` broadcasts the updated deadlines so the panel can refresh its countdown.

The timestamps come from a single `Instant.now()` call in the trigger — the same instant used for the database record, the CDI event, and the push payload. The spec review caught this: computing `Instant.now()` independently in the notifier would create clock skew between the stored record and the pushed payload.

## Eight panels, zero custom components

The trading desk now has eighteen panels — ten from C1-C3 and eight from C4. The C4 panels sit in the right zone as `defaultOpen: false`, available when an operator opens them. C5b will move them to a dedicated Ops Centre page with proper zone assignments.

Every panel is either a pages-ui DSL composition or a blocks-ui web component. No custom Lit code in this branch. The platform's component library did the work — we just had to wire the plumbing underneath.
