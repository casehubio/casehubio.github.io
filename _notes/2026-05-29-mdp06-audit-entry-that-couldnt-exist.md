---
layout: post
title: "The Audit Entry That Couldn't Exist"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-work]
tags: [audit, flyway, jpa, cdi, transactions]
---

The spec said there was no FK constraint on `audit_entry.work_item_id`.

There was a FK constraint.

We'd been working through a batch of small cleanup items — adding `SIGNAL_RECEIVED` to the event enum, flipping `ClaimFirstStrategy` to `@Alternative @Priority(0)` so CDI resolves it correctly when the AI module is present, fixing some test representativeness from last week's migration rename. The interesting one was a feature I'd been deferring: audit entries for denied WorkItem creation.

The existing `BlockedAttemptAuditService` already records `CLAIM_DENIED` and `DELEGATE_DENIED` when lifecycle attempts are blocked. Extending that to creation denials — when the proposed assignee is on the exclusion list — had one complication: rejected creates don't produce a WorkItem. The audit entry needs a WorkItem ID to reference, but the ID is normally assigned by `@PrePersist`.

The fix was to pre-generate it before the exclusion check. `@PrePersist` already guards with `if (id == null)`:

```java
item.id = UUID.randomUUID();

PolicyDecision decision = exclusionPolicy.check(request.assigneeId, item.excludedUsers);
if (decision.denied()) {
    blockedAuditService.record(item.id, "CREATE_DENIED", ...);
    throw new IllegalArgumentException(decision.reason());
}
```

I expected the test — `GET /audit?event=CREATE_DENIED` after a rejected create — to return a 400 and an audit entry. Instead it returned a 500, with `ArcUndeclaredThrowableException`. The stack trace eventually yielded: `JdbcSQLIntegrityConstraintViolationException: "FK_AUDIT_ENTRY_WORK_ITEM"`.

The original analysis had said the column had no FK constraint. That was wrong. The initial schema had always had one. The audit entry was trying to reference a WorkItem that didn't exist, and the database was refusing.

Once we knew that, a second thing became clear. `BlockedAttemptAuditService.record()` has a try-catch that's supposed to swallow any failure:

```java
@Transactional(TxType.REQUIRES_NEW)
public void record(UUID workItemId, ...) {
    try {
        AuditEntry entry = new AuditEntry();
        entry.workItemId = workItemId;
        auditStore.append(entry);   // em.persist() — deferred
    } catch (Exception e) {
        LOG.warning("audit write failed");
    }
}
```

The catch block looked complete. `Exception` covers everything. But JPA's `em.persist()` doesn't execute the SQL — it queues the INSERT. The SQL runs at commit time, after the method returns. Narayana commits the `REQUIRES_NEW` transaction after the method body exits, which is after the try-catch has already closed. The FK violation fires during commit, propagates through the CDI proxy, and surfaces in the caller as an `ArcUndeclaredThrowableException` that bypasses the catch entirely.

The try-catch that was meant to swallow failures was never given the chance to see this one.

The fix was to drop the constraint. Audit entries for denied operations are explicitly about attempts where the entity never existed — enforcing referential integrity there is the wrong model. A single migration:

```sql
ALTER TABLE audit_entry DROP CONSTRAINT fk_audit_entry_work_item;
```

After that, the pre-generated ID commits cleanly via `REQUIRES_NEW`, the 400 returns as expected, and the audit entry is queryable even though the WorkItem never reached the database.
