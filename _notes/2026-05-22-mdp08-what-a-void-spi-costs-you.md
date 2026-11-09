---
layout: post
title: "What a Void SPI Costs You"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [quarkus, spi, design, cleanup]
---

`EscalationPolicy.escalate()` returned void. That's a small thing until you need to do something after the policy runs.

The runtime called `policy.escalate(event)` and stopped. Everything that happened next — status mutation, audit writes, CDI events — was the policy's problem. Which meant three concrete impls (`AutoReject`, `Reassign`, `Notify`) each doing their own version of those things, inconsistently. The runtime had no way to know whether the item was now EXPIRED or PENDING, whether to fire an escalation event, or whether to attempt reassignment.

`SlaBreachPolicy` inverted this. The policy returns a `BreachDecision`. The runtime executes it, owns all the side effects, fires `SlaBreachEvent`. The policy is pure decision logic.

We removed all three old impls this week, along with the producer, the qualifiers, and the config keys. Most of it was dead the moment `SlaBreachPolicy` shipped — the deletion just makes that explicit.

## What was worth keeping

Before deleting, I asked whether any of the old impls had logic worth preserving. Most didn't — `ReassignEscalationPolicy`'s candidate-group guard is already covered by `EscalateTo`'s factory validation, and `NotifyEscalationPolicy` fired a `WorkItemExpiredEvent` that had zero consumers.

`AutoRejectEscalationPolicy` had one thing: it set `entry.detail = "auto-rejected: expiry deadline exceeded"` on the audit record. The new `executeFail()` stored the reason in `item.resolution` but never surfaced it to the audit entry. One-line fix — `writeAudit()` takes a nullable detail now, and `executeFail()` passes `fail.reason()` through.

## The auto-assignment gap

The design inversion exposed a gap the void SPI had hidden. When `EscalateTo` changes the candidateGroups and sets status back to PENDING, nothing happens next — the item sits in the new pool waiting for a worker to claim manually, even under least-loaded routing.

We added `AssignmentTrigger.SLA_ESCALATED` and wired it into `executeEscalateTo()`, called before `put()` so a single write captures the final state:

```java
assignmentService.assign(item, AssignmentTrigger.SLA_ESCALATED);
workItemStore.put(item);  // ASSIGNED or PENDING depending on strategy
```

The elegant part is how `WorkerSelectionStrategy.triggers()` works:

```java
default Set<AssignmentTrigger> triggers() {
    return Set.of(AssignmentTrigger.values());
}
```

That's a live call to `values()`, not a snapshot. Every strategy using the default automatically picks up `SLA_ESCALATED`. `ClaimFirstStrategy` already returns `noChange()` for all inputs, so it stays claim-first with no code change — escalation just returns the item to the new pool. The routing strategies pre-assign.

## Claude going off-script

During cleanup, Claude went beyond scope. One review subagent noticed that `WorkItemStore` and `AuditEntryStore` still described the CDI priority ladder in prose that should link to a platform protocol instead. It filed an issue, implemented the Javadoc fix, and closed the issue — none of which I'd asked for.

I asked whether the work was valid. It was. The Javadoc now links to the protocol rather than restating the rule inline. I kept it.

## What the sweep missed

The cleanup removed the dead config keys from `runtime/src/main/resources/application.properties`. What it missed: `integration-tests/src/main/resources/application.properties` had the same keys as live properties, not comments. Claude caught them in the final code review — would have produced unknown-property warnings at startup.

When removing a `@ConfigMapping` method, grep across all modules. The integration-tests module is its own Maven artifact and carries its own `application.properties`. It is easy to forget.
